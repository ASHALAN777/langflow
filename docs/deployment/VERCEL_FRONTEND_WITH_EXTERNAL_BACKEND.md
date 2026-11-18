# Deploy the Langflow UI to Vercel

This repository builds Langflow's frontend from `src/frontend`. The Vercel
configuration at the repository root is intentionally a **static frontend
deployment**. It does not try to package Langflow's Python backend as a Vercel
Function.

Langflow's backend is a long-running FastAPI application with startup/lifespan
work, background services, a database, and optional persistent file storage.
The official Docker build compiles the frontend and copies the resulting assets
into the Python package. A Vercel static deployment cannot replace that
container runtime or provide durable backend storage.

## Architecture

- **Vercel:** builds and serves the React visual editor.
- **Container host:** runs the complete Langflow backend using
  `docker/build_and_push.Dockerfile` (the default `full` target).
- **Database/storage:** use managed PostgreSQL and persistent storage for
  production. Do not rely on a serverless function's local filesystem for
  Langflow's database or user files.

The frontend talks directly to the backend origin configured at build time.
This avoids pretending that a static Vercel deployment also hosts the Python
API or its WebSocket connections.

## Vercel project settings

1. Import this repository into Vercel.
2. Set the project **Root Directory** to the repository root (not
   `src/frontend`).
3. Leave the build and install commands to `vercel.json`.
4. Add the environment variable `LANGFLOW_BACKEND_URL` for Production,
   Preview, and Development. Its value must be the public HTTPS origin of the
   running Langflow backend, for example `https://your-langflow-api.example.com`.
   Do not add a trailing slash or an API path.
5. Deploy.

The build intentionally fails when `LANGFLOW_BACKEND_URL` is missing. This
prevents a successful-looking frontend build from silently compiling the
default `http://localhost:7860` backend address into the production bundle.

## Backend host requirements

Create a Docker web service from this repository with:

- Dockerfile: `docker/build_and_push.Dockerfile`
- Docker build context: repository root
- Image target: the Dockerfile's default `full` target
- Start command: keep the Dockerfile default, `langflow run`
- Public HTTPS URL: required for browser access from the Vercel site

Configure the backend host with at least:

- `LANGFLOW_HOST=0.0.0.0`
- `LANGFLOW_AUTO_LOGIN=false`
- `LANGFLOW_DATABASE_URL`: a managed PostgreSQL connection string
- `LANGFLOW_CORS_ORIGINS`: the exact Vercel production origin, plus any
  custom frontend domain (comma-separated origins are supported by Langflow's
  settings)

Use a persistent disk for any local data/files Langflow is configured to keep.
The container's writable layer is not durable across deployments. Configure
backups for the database and persistent files.

Do not use `LANGFLOW_CORS_ORIGINS=*` with credentialed browser access. Keep
the allowed origins restricted to the exact frontend domains you control.

## Validation checklist

After both services are deployed:

1. Open the Vercel deployment and confirm the visual editor loads.
2. In browser developer tools, verify API requests go to the configured
   backend origin and do not fail with CORS errors.
3. Verify the backend `/health` endpoint responds successfully.
4. Create, save, and reopen a flow to verify database persistence.
5. Run a flow and verify streaming/realtime behavior and any required
   WebSocket connections work through the backend host.
6. Restart/redeploy the backend and confirm saved flows remain available.

## Important limitation

This is a split deployment: Vercel hosts the UI, while a container host hosts
the backend. It is the supported direction for retaining the complete
Langflow experience without incorrectly mapping the CLI module
`langflow.__main__` to a FastAPI entrypoint. A one-service, all-on-Vercel
deployment is not provided by this configuration.
