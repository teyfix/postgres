# teyfix/postgres

Daily builds of PostgreSQL-based Docker images with pre-compiled extensions.

## Available Images

### timescaledb-pgrx

TimescaleDB with PGRX extensions including Supabase Wrappers (ClickHouse FDW).

```bash
docker pull teyfix/timescaledb-pgrx:latest
```

See [timescaledb-pgrx/README.md](timescaledb-pgrx/README.md) for detailed usage instructions.

## Build Schedule

All images are built daily at 2:00 AM UTC via GitHub Actions to ensure they include the latest security updates and dependencies.

## Tags

Each image is tagged with:

- `latest`: Most recent build
- `YYYYMMDD`: Daily build date (e.g., `20250622`)
- Git commit SHA for specific builds

## Docker Hub

All images are published to Docker Hub under the `teyfix` user:

- [teyfix](https://hub.docker.com/u/teyfix)
