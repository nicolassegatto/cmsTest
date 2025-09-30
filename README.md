# Directus CMS Stack (Postgres + MinIO + Custom Extensions)

A containerized Directus environment using Postgres for the database, MinIO as an S3-compatible object store, and a mount for developing custom Directus extensions (endpoints) locally.

---
## Contents
- Overview
- Stack Architecture
- Prerequisites
- Quick Start
- Environment Variables (.env example)
- Managing the Stack
- Creating a Custom Endpoint Extension (MANDATORY NAMING RULE)
- Example Endpoint Code
- Rebuilding / Reloading Extensions
- Accessing MinIO Console & Bucket
- Data Persistence
- Troubleshooting & Tips
- Next Ideas

---
## Overview
This project spins up:
- **Postgres** – Directus primary database
- **MinIO** – S3-compatible storage backend (public bucket setup)
- **Directus** – Headless CMS/API running in a container with hot-loaded extensions from `./data/directus/extensions`

You can create custom **endpoint extensions** without rebuilding the Directus image: they're mounted as a volume.

---
## Stack Architecture
```
./docker-compose.yml
./data/
  postgres/                  # Persistent Postgres data volume
  minio/                     # Persistent MinIO data & uploaded objects
  directus/
    extensions/              # Local development space for Directus extensions
      directus-extension-endpoint-example/  # (example custom endpoint)
```
Services (from `docker-compose.yml`):
- `postgres` (port: env POSTGRES_PORT -> container 5432)
- `minio` API (port: env MINIO_API_PORT -> container 9000)
- `minio` Console (port: env MINIO_CONSOLE_PORT -> container 9001)
- `directus` (port: env DIRECTUS_PORT -> container 8055)

---
## Prerequisites
- Docker & Docker Compose (Compose V2 recommended)
- Node.js ≥ 18 (for extension development & build)
- `npm` (or `pnpm`/`yarn` if you adapt commands)

---
## Quick Start
1. Create a `.env` file (see template below)
2. Start services:
   ```bash
   docker compose up -d
   ```
3. Wait until Directus is healthy (first boot will create the admin user).
4. Open Directus: `http://localhost:<DIRECTUS_PORT>`
5. Log in with the admin credentials you set in `.env`.

---
## Environment Variables (.env Example)
Create a `.env` at the project root:
```env
# Images
POSTGRES_IMAGE=postgres:16-alpine
MINIO_IMAGE=minio/minio:latest
DIRECTUS_IMAGE=directus/directus:latest

# Ports (host:container)
POSTGRES_PORT=5433
MINIO_API_PORT=9000
MINIO_CONSOLE_PORT=9001
DIRECTUS_PORT=8055

# Postgres
POSTGRES_DB=directus
POSTGRES_USER=directus
POSTGRES_PASSWORD=directus_password

# MinIO / S3 storage (used by Directus)
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin
MINIO_BUCKET=directus
STORAGE_DRIVER=s3
STORAGE_S3_BUCKET=directus
STORAGE_S3_KEY=minioadmin
STORAGE_S3_SECRET=minioadmin
STORAGE_S3_REGION=us-east-1
STORAGE_S3_ENDPOINT=http://minio:9000
STORAGE_S3_FORCE_PATH_STYLE=true

# Directus Core
PUBLIC_URL=http://localhost:8055
KEY=replace_with_random_32_chars
SECRET=replace_with_random_64_chars
ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD=ChangeMe123!
WEBSOCKETS_ENABLED=true
CORS_ENABLED=true
CORS_ORIGIN=*
```
Generate secure KEY/SECRET quickly:
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))" # for KEY
node -e "console.log(require('crypto').randomBytes(48).toString('hex'))" # for SECRET
```

---
## Managing the Stack
Common lifecycle commands:
```bash
# Start (detached)
docker compose up -d

# View logs (follow one or all)
docker compose logs -f directus

# Restart a single service (useful after building an extension)
docker compose restart directus

# Stop services
docker compose down

# Stop & remove volumes/data (DANGEROUS – full reset)
docker compose down -v
```

---
## Creating a Custom Endpoint Extension
All endpoint extensions MUST follow this naming convention:

Prefix: `directus-extension-endpoint-`

For example: `directus-extension-endpoint-healthcheck` or `directus-extension-endpoint-reports`.

Why? Directus auto-detects extensions by folder name; the prefix keeps things consistent and future-safe.

Steps:
```bash
# 1. Move into the extensions directory
cd data/directus/extensions

# 2. Run the Directus extension generator
npx create-directus-extension@latest

# 3. When prompted:
#    ? What type of extension?  -> endpoint
#    ? What is the name?        -> directus-extension-endpoint-yourfeature
#    (Accept defaults for the rest unless you need changes)

# 4. Enter the new folder
cd directus-extension-endpoint-yourfeature

# 5. Install dependencies (if the generator didn't already)
npm install

# 6. Build the extension
npm run build

# 7. Return to project root and restart Directus so it picks it up
cd ../../../..
docker compose restart directus
```
After restart, the endpoint becomes available at the route you define within the code (see next section). No image rebuild needed because the volume is mounted.

---
## Example Endpoint Code
Inside the generated extension you'll have something like `src/index.ts`.
A minimal example:
```ts
import type { EndpointConfig } from '@directus/extensions-sdk';

// The exported default object registers your routes.
export default {
  id: 'directus-extension-endpoint-yourfeature',
  handler: (router, { services, database, getSchema, logger }) => {
    router.get('/ping', (_req, res) => {
      res.json({ status: 'ok', time: new Date().toISOString() });
    });

    router.get('/db-stats', async (_req, res, next) => {
      try {
        const schema = await getSchema();
        res.json({ collections: Object.keys(schema.collections).length });
      } catch (err) {
        logger.error(err);
        next(err);
      }
    });
  },
} satisfies EndpointConfig;
```
If the folder is `directus-extension-endpoint-yourfeature`, and you used `/ping`, the public route becomes:
```
GET /extensions/yourfeature/ping
```
Test it:
```bash
curl http://localhost:8055/extensions/yourfeature/ping
```
Expected JSON:
```json
{"status":"ok","time":"2025-09-30T..."}
```

---
## Rebuilding / Reloading Extensions
When you change TypeScript/ESM source:
```bash
cd data/directus/extensions/directus-extension-endpoint-yourfeature
npm run build
cd ../../../..
docker compose restart directus
```
For faster iteration you can optionally run `npm run dev` (if scaffolded) and only restart Directus when adding new files or changing manifest-level metadata.

---
## Accessing MinIO
- API (S3-compatible): `http://localhost:<MINIO_API_PORT>` (default 9000)
- Console UI: `http://localhost:<MINIO_CONSOLE_PORT>` (default 9001)
- Credentials: `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`
- Public bucket automatically created: `${MINIO_BUCKET}`

You can point external S3 clients (e.g. `aws cli`) by exporting:
```bash
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=minioadmin
export AWS_EC2_METADATA_DISABLED=true
aws --endpoint-url http://localhost:9000 s3 ls
```

---
## Data Persistence
Mounted volumes ensure data survives container restarts:
- Postgres data: `./data/postgres`
- MinIO objects: `./data/minio`
- Directus custom code: `./data/directus/extensions`

Remove with caution using `docker compose down -v` (destroys volumes).

---
## Troubleshooting & Tips
| Issue | Cause | Fix |
|-------|-------|-----|
| Directus container restarts | Bad env vars or DB not ready | Check `docker compose logs -f directus` |
| 404 on custom endpoint | Forgot build or restart | Re-run `npm run build` + restart Directus |
| MinIO bucket missing | Init race (rare) | Restart `minio-mc-init` or recreate stack |
| Uploads failing | Wrong S3 endpoint vars | Confirm `STORAGE_S3_ENDPOINT=http://minio:9000` and force path style |
| SSL errors to MinIO | Using https URL locally | Use `http://` in local dev |
| Permissions error in extension build | Node version mismatch | Ensure Node ≥ 18 locally |

Additional tips:
- Use unique KEY/SECRET in non-dev environments.
- Keep admin credentials out of version control.
- Consider adding a `Makefile` or NPM scripts for repetitive tasks.

---
## Next Ideas
- Add a `Makefile` for `make up`, `make restart`.
- Add automated tests for endpoint logic (Jest + supertest hitting a running Directus).
- Implement additional extension types (interfaces, hooks, operations).
- Configure HTTPS + reverse proxy (Traefik / Caddy / Nginx) for production.

---
## License
Add your license details here (MIT, Apache-2.0, proprietary, etc.).

---
## Summary
You now have a portable Directus stack with Postgres + MinIO and a clear workflow for creating endpoint extensions:
1. Scaffold (name must start with `directus-extension-endpoint-`)
2. Implement logic in `src/index.ts`
3. `npm run build`
4. `docker compose restart directus`
5. Use at `/extensions/<short-name>/...`

Happy building!
