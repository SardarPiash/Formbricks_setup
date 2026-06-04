# Formbricks Setup for Rokomari

This folder runs a self-hosted Formbricks instance for collecting website survey responses from `rokomari120`.

## Local Setup

### 1. Start Formbricks

From this folder:

```bash
cd /home/gh/Projects/ddd/formbricks-quickstart
docker compose up -d
```

Open Formbricks:

```text
http://localhost:3000
```

Check containers:

```bash
docker compose ps
```

Expected healthy services:

- `postgres`
- `redis`
- `hub`
- `cube`
- `formbricks`

### 2. Local Environment

Local values are loaded from:

```text
/home/gh/Projects/ddd/formbricks-quickstart/.env
```

Required local values:

```env
HUB_API_KEY=<random-hex-secret>
CUBEJS_API_SECRET=<random-hex-secret>
CUBEJS_JWT_ISSUER=formbricks-web
CUBEJS_JWT_AUDIENCE=formbricks-cube
```

The compose file defaults these values for local URLs:

```env
WEBAPP_URL=http://localhost:3000
NEXTAUTH_URL=http://localhost:3000
DATABASE_URL=postgresql://postgres:postgres@postgres:5432/formbricks_app?schema=public
HUB_DATABASE_URL=postgresql://postgres:postgres@postgres:5432/formbricks_hub?sslmode=disable
HUB_API_URL=http://hub:8080
CUBEJS_API_URL=http://cube:4000
```

### 3. Rokomari Local Website Integration

The Formbricks script is currently added here:

```text
/home/gh/Projects/rokomari120/src/main/webapp/WEB-INF/tags/200/tracking.tag
```

This tag is included by the newer 200 desktop/mobile wrappers:

```text
/home/gh/Projects/rokomari120/src/main/webapp/WEB-INF/tags/200/desktop/basicHtml.tag
/home/gh/Projects/rokomari120/src/main/webapp/WEB-INF/tags/200/mobile/basicHtml.tag
```

Local script values:

```js
var appUrl = "http://localhost:3000";
var environmentId = "<your-formbricks-environment-id>";
```

Test route for Rokomari:

```text
http://localhost:8080/
```

In browser DevTools console, verify:

```js
window.formbricks
```

If it returns an object/function, the integration script is loaded.

## Create a Dummy Survey

1. Open `http://localhost:3000`.
2. Go to **Surveys**.
3. Click **Start from scratch**.
4. Add a simple question, for example:

```text
How was your Rokomari homepage experience?
```

5. Go to **Settings**.
6. Under **Survey Trigger**, add action:
   - Type: `No code`
   - User action: `Page View`
   - Page filter: `On all pages` for first test
7. Keep **Target Audience** as all visitors.
8. Publish the survey.
9. Open `http://localhost:8080/` in an incognito window.

If the survey does not show, clear Formbricks browser state:

```js
Object.keys(localStorage)
  .filter((key) => key.toLowerCase().includes("formbricks"))
  .forEach((key) => localStorage.removeItem(key));

Object.keys(sessionStorage)
  .filter((key) => key.toLowerCase().includes("formbricks"))
  .forEach((key) => sessionStorage.removeItem(key));
```

Then hard refresh the Rokomari page.

## Production Setup

### 1. Use Production URLs

In production, set these values in the deployment environment or production `.env`:

```env
WEBAPP_URL=https://formbricks.your-domain.com
NEXTAUTH_URL=https://formbricks.your-domain.com
PUBLIC_URL=https://formbricks.your-domain.com
```

The Rokomari website script must also use the same production URL:

```js
var appUrl = "https://formbricks.your-domain.com";
var environmentId = "<production-environment-id>";
```

Do not keep `http://localhost:3000` in production.

### 2. Generate Production Secrets

Generate new production-only secrets:

```bash
openssl rand -hex 32
```

Use unique values for:

```env
NEXTAUTH_SECRET=<random-hex-secret>
ENCRYPTION_KEY=<random-hex-secret>
CRON_SECRET=<random-hex-secret>
HUB_API_KEY=<random-hex-secret>
CUBEJS_API_SECRET=<random-hex-secret>
```

Do not reuse local development secrets in production.

### 3. Use Persistent Databases

Recommended production setup:

- PostgreSQL hosted outside the app container
- Redis/Valkey hosted outside the app container
- Regular database backups
- Separate databases for Formbricks app and Hub

Example:

```env
DATABASE_URL=postgresql://formbricks_user:<password>@<postgres-host>:5432/formbricks_app?schema=public
HUB_DATABASE_URL=postgresql://formbricks_user:<password>@<postgres-host>:5432/formbricks_hub?sslmode=require
REDIS_URL=redis://:<password>@<redis-host>:6379
```

### 4. Put Formbricks Behind HTTPS

Use a reverse proxy/load balancer such as Nginx, Traefik, or cloud ingress.

Minimum requirements:

- HTTPS enabled
- Formbricks public domain points to the `formbricks` service on port `3000`
- WebSocket/proxy headers supported
- Request body size large enough if file upload questions will be used

### 5. Email Setup

For production invites, password reset, and verification emails, configure SMTP:

```env
MAIL_FROM=noreply@your-domain.com
MAIL_FROM_NAME=Rokomari
SMTP_HOST=<smtp-host>
SMTP_PORT=587
SMTP_USER=<smtp-user>
SMTP_PASSWORD=<smtp-password>
SMTP_AUTHENTICATED=1
```

For local testing, email can stay disabled:

```env
EMAIL_VERIFICATION_DISABLED=1
PASSWORD_RESET_DISABLED=1
```

### 6. File Upload Warning

If Formbricks shows:

```text
File storage not set up, uploads will likely fail
```

That is not a blocker for normal text/rating surveys.

For production surveys with file upload questions, configure S3-compatible storage:

```env
S3_ACCESS_KEY=<access-key>
S3_SECRET_KEY=<secret-key>
S3_REGION=<region>
S3_BUCKET_NAME=<bucket>
S3_ENDPOINT_URL=<optional-s3-compatible-endpoint>
S3_FORCE_PATH_STYLE=0
```

## Useful Commands

Start:

```bash
docker compose up -d
```

Stop:

```bash
docker compose down
```

View logs:

```bash
docker compose logs -f formbricks
docker compose logs -f cube
docker compose logs -f hub
```

Pull latest images:

```bash
docker compose pull
docker compose up -d
```

Reset local data only when you intentionally want a clean Formbricks install:

```bash
docker compose down -v
docker compose up -d
```

## Troubleshooting

### Formbricks does not start

Check:

```bash
docker compose ps
docker compose logs --tail=200 formbricks
```

### Cube stays unhealthy

This setup uses a Node-based Cube healthcheck because the Cube image may not include `wget`.

Check:

```bash
docker compose logs --tail=200 cube
```

### Survey script loads but modal does not show

1. Confirm the survey is published.
2. Use `No code` + `Page View` + `On all pages` for the first test.
3. Test in an incognito browser window.
4. Verify `window.formbricks` exists on `http://localhost:8080/`.
5. Clear Formbricks keys from local/session storage and hard refresh.

### Browser warning about preloaded surveys script

This warning is usually harmless:

```text
The resource http://localhost:3000/js/surveys.umd.cjs was preloaded but not used
```

It means the Formbricks script was reached. If the modal does not appear, check survey trigger, audience, recontact, and publish status.
