# TimescaleDB with PGRX Extensions

## Table of Contents

- [TimescaleDB with PGRX Extensions](#timescaledb-with-pgrx-extensions)
  - [Table of Contents](#table-of-contents)
  - [Usage](#usage)
    - [Quick Start](#quick-start)
    - [Connect to Database](#connect-to-database)
  - [Included Extensions](#included-extensions)
    - [TimescaleDB](#timescaledb)
    - [Supabase Wrappers](#supabase-wrappers)
      - [ClickHouse FDW](#clickhouse-fdw)
      - [Setting up the extension](#setting-up-the-extension)
      - [Connecting PostgreSQL to ClickHouse](#connecting-postgresql-to-clickhouse)
  - [Configuration](#configuration)
  - [Persistence](#persistence)
  - [Image Details](#image-details)
  - [Available Tags](#available-tags)
  - [Logs](#logs)
  - [Health Check](#health-check)
  - [Docker Compose](#docker-compose)

Pre-built [TimescaleDB](https://github.com/timescale/timescaledb) Docker image with [PGRX](https://github.com/pgcentralfoundation/pgrx) extensions, specifically including
[Supabase Wrappers](https://fdw.dev/) with [ClickHouse FDW](https://fdw.dev/catalog/clickhouse/) support.

## Usage

### Quick Start

```bash
docker run -d \
  --name timescaledb-pgrx \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD=your_password \
  teyfix/timescaledb-pgrx:latest
```

### Connect to Database

```bash
psql -h localhost -U postgres -d postgres
```

## Included Extensions

### TimescaleDB

Full [TimescaleDB](https://github.com/timescale/timescaledb) functionality for time-series data.

### Supabase Wrappers

Checkout [fdw.dev](https://fdw.dev/catalog/clickhouse/) for more details about
`wrappers` extension.

#### ClickHouse FDW

Foreign data wrapper for connecting to ClickHouse databases.

> [!NOTE]
> The ClickHouse Foreign Data Wrapper (FDW) does **not support** certain
> ClickHouse-specific data types such as `LowCardinality(String)`, `Array(T)`,
> `JSON`, and other complex types directly. To ensure correct data retrieval,
> you **must explicitly cast** these unsupported columns within a ClickHouse
> view **before** querying them through the FDW. Failing to cast will cause
> `column data type 'LowCardinality(String)' is not supported` error where
> `LowCardinality(String)` is a ClickHouse data type.

```sql
-- Example: Create a ClickHouse view normalizing complex types for FDW access
CREATE VIEW "sample_fdw" AS
SELECT
  "id",
  -- Convert LowCardinality(String) to String explicitly
  CAST(materialize("low_cardinality_str") AS String) AS "low_cardinality_str",
  -- Convert JSON or Array columns to JSON string representation
  CAST(toJSONString("json_column") AS String) AS "json_column"
FROM "sample";
```

Checkout [ClickHouse FDW Documentation](https://fdw.dev/catalog/clickhouse/) for
more details and
[supported data types](https://fdw.dev/catalog/clickhouse/#supported-data-types).

#### Setting up the extension

Clean up existing wrappers extension and all dependent objects:

```sql
DROP EXTENSION IF EXISTS wrappers CASCADE;
```

Create dedicated schema for the wrappers extension:

```sql
CREATE SCHEMA IF NOT EXISTS wrappers;
```

Install the wrappers extension into the wrappers schema (keeps wrapper functions
organized and avoids namespace conflicts):

```sql
CREATE EXTENSION IF NOT EXISTS wrappers
WITH
  SCHEMA wrappers;
```

Create the ClickHouse foreign data wrapper using Supabase wrappers handlers
(manages connection and data type conversion between PostgreSQL and ClickHouse):

```sql
CREATE FOREIGN DATA WRAPPER clickhouse_fdw
  HANDLER wrappers.click_house_fdw_handler
  VALIDATOR wrappers.click_house_fdw_validator;
```

#### Connecting PostgreSQL to ClickHouse

Remove existing server connection if it exists:

```sql
DROP SERVER IF EXISTS clickhouse_server CASCADE;
```

Create server connection to ClickHouse instance (conn_string format:
`tcp://username:password@host:port/database`):

> [!WARNING]
> This image does not contain the [Vault](https://supabase.com/docs/guides/database/vault) extension, therefore you can not use `vault.create_secret` which is documented in [ClickHouse FDW Documentation](https://fdw.dev/catalog/clickhouse/#store-your-credentials-optional).

```sql
CREATE SERVER clickhouse_server FOREIGN DATA WRAPPER clickhouse_fdw OPTIONS (
  conn_string 'tcp://default:secret@clickhouse:9000/default' -- Example with local clickhouse container
);
```

Create schema to hold foreign tables that mirror ClickHouse tables:

```sql
CREATE SCHEMA IF NOT EXISTS clickhouse;
```

Import all table definitions from ClickHouse "default" database _(this doesn't
seem to work at this time - 2025/06/22):_

```sql
IMPORT FOREIGN SCHEMA "default"
FROM
  SERVER clickhouse_server INTO clickhouse;
```

Clean up existing foreign table if it exists:

```sql
DROP FOREIGN TABLE IF EXISTS clickhouse.users;
```

Create custom foreign table with specific column mapping:

> [!CAUTION]
> Incorrect column type definitions will cause silent data corruption.
> For example, a value like 86 stored as a Float64 might be misinterpreted as 4.6370818327779e-310 when queried, leading to confusing or invalid data results.

```sql
CREATE FOREIGN TABLE clickhouse.users (
  id UUID,
  email TEXT,
  password TEXT,
  username TEXT,
  full_name TEXT,
  created_at TIMESTAMP(3)
) SERVER clickhouse_server OPTIONS (TABLE 'users');
```

## Configuration

Standard PostgreSQL and TimescaleDB environment variables are supported:

- `POSTGRES_PASSWORD`: Required - PostgreSQL superuser password
- `POSTGRES_USER`: PostgreSQL superuser name (default: postgres)
- `POSTGRES_DB`: Default database name (default: postgres)

## Persistence

Mount a volume for data persistence:

```bash
docker run -d \
  --name timescaledb-pgrx \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=your_password \
  teyfix/timescaledb-pgrx:latest
```

## Image Details

- **Base**: timescale/timescaledb-ha:pg17
- **PostgreSQL**: Version 17
- **Architecture**: Multi-stage build for optimized size
- **Updates**: Built daily with latest dependencies

## Available Tags

- `latest`: Most recent build
- `YYYYMMDD`: Daily builds (e.g., `20250622`)
- Git commit SHA for specific versions

## Logs

View container logs:

```bash
docker logs timescaledb-pgrx
```

## Health Check

The image includes [`pg_isready`](https://www.postgresql.org/docs/current/app-pg-isready.html) built-in health monitoring. Check container
status:

```bash
docker exec timescaledb-pgrx pg_isready -d postgres -U postgres
```

## Docker Compose

How to use with [Docker Compose](https://docs.docker.com/compose/):

```yaml
services:
  postgres:
    image: teyfix/timescaledb-pgrx:20250622
    restart: unless-stopped
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -d ${POSTGRES_DB:?} -U ${POSTGRES_USER:?}
      interval: 5s
      timeout: 5s
      retries: 15
      start_period: 5s
    environment:
      POSTGRES_DB: ${POSTGRES_DB:?}
      POSTGRES_USER: ${POSTGRES_USER:?}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?}
```
