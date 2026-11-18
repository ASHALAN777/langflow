# Deploy the Langflow UI to Vercel

This is a **static frontend deployment**. It does not package the Langflow
Python backend as a Vercel Function. Langflow's backend runs startup/lifespan
work, background services, database migrations, and optional persistent file
storage. The official Docker build compiles the frontend and copies its assets
into the Python package.

## Architecture

- **Vercel:** builds and serves the React visual editor.
- **Container host:** runs the complete Langflow backend using
  `docker/build_and_push.Dockerfile` (default `full` target).
- **Database/storage:** use managed PostgreSQL and persistent storage in
  production. Do not rely on a serverless function's local filesystem.

The frontend talks directly to the backend origin compiled at build time.

## Vercel project settings

1. Import this repository into Vercel.
2. Set **Root Directory** to the repository root, not `src/frontend`.
3. Leave build/install commands to the root `vercel.json`.
4. Set `LANGFLOW_BACKEND_URL` for Production, Preview, and Development. Use
   the public HTTPS origin of the running backend, e.g.
   `https://your-langflow-api.example.com`, with no trailing slash or API path.
5. Deploy.

The build intentionally fails if `LANGFLOW_BACKEND_URL` is missing. This
prevents compiling the default localhost backend address into the production
bundle.

## Backend host requirements

Create a Docker web service from this repository:

- Dockerfile: `docker/build_and_push.Dockerfile`
- Docker build context: repository root
- Image target: the Dockerfile's default `full` target
- Start command: keep the Dockerfile default, `langflow run`
- Public HTTPS URL: required for browser access from Vercel

Configure:

- `LANGFLOW_HOST=0.0.0.0`
- `LANGFLOW_AUTO_LOGIN=false`
- `LANGFLOW_DATABASE_URL`: managed PostgreSQL connection string
- `LANGFLOW_CORS_ORIGINS`: exact Vercel production origin and any custom
  frontend domain (comma-separated origins are supported)

Use persistent storage for any local data/files Langflow must retain, and
configure database and file backups. Do not use wildcard CORS origins with
credentialed browser access; restrict allowed origins to domains you control.

## Validation checklist

1. Open the Vercel deployment and confirm the visual editor loads.
2. Verify API requests target the configured backend and have no CORS errors.
3. Verify the backend `/health` endpoint responds.
4. Create, save, and reopen a flow to verify persistence.
5. Run a flow and verify streaming/realtime and required WebSocket connections.
6. Restart/redeploy the backend and confirm saved flows remain available.

## Important limitation

This is a split deployment: Vercel hosts the UI; a container host hosts the
backend. It avoids mapping the CLI module `langflow.__main__` as a FastAPI
entrypoint. This configuration does not claim that the entire Langflow runtime
can run as a single Vercel deployment.
