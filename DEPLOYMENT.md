# Build, Run, and Deploy Guide

This guide covers the database, backend, frontend, Docker workflow, and deployment steps.

## Requirements

- Node.js `24.12.0` recommended, from `.nvmrc`
- npm
- Docker and Docker Compose, for containerized local/prod-like runs
- PostgreSQL, if running the database outside Docker

## Project Parts

- Frontend: repository root, Vite app
- Backend: `backend/`, Node + Express + Prisma
- Database: PostgreSQL, configured with `DATABASE_URL`

## Environment Files

### Backend

Create `backend/.env`:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/me?schema=public"
PORT=4000
JWT_SECRET="replace_with_a_long_random_secret"
NODE_ENV=development
```

For production, use a real PostgreSQL host and a strong `JWT_SECRET`:

```env
DATABASE_URL="postgresql://USER:PASSWORD@HOST:5432/DB_NAME?schema=public"
PORT=4000
JWT_SECRET="very_long_random_secret"
NODE_ENV=production
```

### Frontend

The root `.env.example` contains:

```env
VITE_API_URL=https://api.dabirgress.runflare.run
```

## Database Setup Without Docker

Start PostgreSQL locally, then create a database named `me`.

Example with `psql`:

```bash
createdb me
```

Set the backend connection string in `backend/.env`:

```env
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/me?schema=public"
```

Install dependencies from the repo root:

```bash
npm install
```
note: if you have error when installing cypress, it's a mismatch for your node version. then you use --force to install or change node version to v20 or lower:

```bash
npm install --force
```

Generate Prisma client:

```bash
npm run prisma:generate
```

Create/update database tables from the Prisma schema:

```bash
npm run prisma:db:push
```

Optional seed data:

```bash
cd backend
npm run seed
```

## Backend Run Without Docker

From repo root:

```bash
npm install
npm run prisma:generate
npm run prisma:db:push
npm run dev:backend
```

Or from `backend/`:

```bash
cd backend
npm install
npm run prisma:generate
npm run prisma:db:push
npm run dev
```

Production-style start:

```bash
cd backend
NODE_ENV=production npm start
```

Backend listens on the `PORT` value, usually:

```text
http://localhost:4000
```

## Frontend Run Without Docker

From repo root:

```bash
npm install
npm run dev
```

Frontend dev server runs at:

```text
http://127.0.0.1:5173
```

The Vite dev proxy in `vite.config.js` points `/api` to:

```text
http://127.0.0.1:4001
```

If you use the proxy, make sure the backend port matches the proxy target, or update the proxy/backend port to match.

## Frontend Build

From repo root:

```bash
npm install
npm run build
```

Build output is created in:

```text
dist/
```

Preview the production build locally:

```bash
npm run preview
```

Preview server runs at:

```text
http://localhost:5173
```

## Docker Run: Database + Backend

A Docker setup is included for backend + PostgreSQL.

From repo root:

```bash
docker compose up --build
```

This starts:

- `db`: PostgreSQL container
- `backend`: Node backend container

The backend container runs database setup before starting:

```bash
npm run prisma:db:push && npm start
```

Backend is exposed at:

```text
http://localhost:4000
```

Stop containers:

```bash
docker compose down
```

Stop containers and delete database volume:

```bash
docker compose down -v
```

## Docker Build Backend Only

From repo root:

```bash
docker build -f backend/Dockerfile -t me-backend .
```

Run the image manually:

```bash
docker run --rm -p 4000:4000 \
  -e DATABASE_URL="postgresql://postgres:postgres@host.docker.internal:5432/me?schema=public" \
  -e JWT_SECRET="replace_with_a_long_random_secret" \
  -e PORT=4000 \
  me-backend
```

Run database migration/schema push inside a one-off container:

```bash
docker run --rm \
  -e DATABASE_URL="postgresql://postgres:postgres@host.docker.internal:5432/me?schema=public" \
  me-backend \
  npm run prisma:db:push
```

## Production Deployment Checklist

### Database

1. Create a managed PostgreSQL database.
2. Copy the production database URL.
3. Set `DATABASE_URL` in the backend hosting environment.
4. Run Prisma schema setup:

```bash
npm run prisma:db:push
```

For stricter production workflows, use Prisma migrations instead of `db push`.

### Backend

1. Build the Docker image:

```bash
docker build -f backend/Dockerfile -t me-backend .
```

2. Deploy it to your host/container platform.
3. Set environment variables:

```env
DATABASE_URL="postgresql://USER:PASSWORD@HOST:5432/DB_NAME?schema=public"
PORT=4000
JWT_SECRET="very_long_random_secret"
NODE_ENV=production
```

4. Expose the backend publicly behind HTTPS.
5. Confirm the backend domain works with `/api` routes.

### Frontend

1. Make sure frontend API URLs point to the deployed backend domain.
2. Build the frontend:

```bash
npm run build
```

3. Deploy the `dist/` folder to static hosting, for example Netlify, Vercel, Cloudflare Pages, Nginx, or S3/CloudFront.
4. Configure SPA fallback to serve `index.html` for client-side routes.

## Common Local Commands

Install everything:

```bash
npm install
```

Run backend:

```bash
npm run dev:backend
```

Run frontend:

```bash
npm run dev
```

Build frontend:

```bash
npm run build
```

Generate Prisma client:

```bash
npm run prisma:generate
```

Push Prisma schema to DB:

```bash
npm run prisma:db:push
```

Run Docker backend + DB:

```bash
docker compose up --build
```

## Troubleshooting

### Backend cannot connect to database

Check `DATABASE_URL`, database host, username, password, database name, and whether PostgreSQL is running.

### Prisma client error

Regenerate the Prisma client:

```bash
npm run prisma:generate
```

### Tables do not exist

Push the Prisma schema:

```bash
npm run prisma:db:push
```