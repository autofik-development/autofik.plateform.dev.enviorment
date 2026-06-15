# Database Setup - PostgreSQL (PostGIS) & PgAdmin

This setup creates a PostgreSQL + PostGIS database and PgAdmin management interface using Docker Compose. PostGIS adds spatial/geographic data support (geometry, geography types, spatial queries).

## Prerequisites

- Docker and Docker Compose installed
- Port 5432 available (PostgreSQL)
- Port 8080 available (PgAdmin)

## Quick Start

### 1. Create the Docker Network for both pgadmin and pgdb

```powershell
cd database/pgdb
docker network create postgres-network
```
```powershell
cd database/pgadmin
docker network create postgres-network
```

If linux or mac no need to run this

### 2. Start PostgreSQL

From the `database/pgdb` directory:

```powershell
cd database/pgdb
docker-compose up -d
```
If linux or mac run

```
cd database/pgdb/linux
docker-compose up -d
```

### 3. Start PgAdmin

From the `database/pgadmin` directory:

```powershell
cd database/pgadmin
docker-compose up -d
```

## Access Services

- **PostgreSQL:** `localhost:5432`
- **PgAdmin:** `http://localhost:8080`

## PgAdmin Login

- **Email:** `admin@example.com`
- **Password:** `root123`

## Connect PostgreSQL to PgAdmin

1. Open PgAdmin at `http://localhost:8080`
2. Click "Add New Server"
3. Fill in:
   - **Name:** postgres
   - **Host:** postgres
   - **Port:** 5432
   - **Username:** postgres
   - **Password:** root123
   - **Database:** postgres
4. Click Save

## Database Credentials

- **User:** postgres
- **Password:** root123
- **Database:** postgres
- **Port:** 5432

## Stop Services

```powershell
# Stop PostgreSQL
docker-compose -f database/pgdb/docker-compose.yml down

# Stop PgAdmin
docker-compose -f database/pgadmin/docker-compose.yml down
```

## Cleanup

To remove everything including data:

```powershell
docker-compose -f database/pgdb/docker-compose.yml down -v
docker-compose -f database/pgadmin/docker-compose.yml down
docker network rm postgres-network
``` 

## PostGIS

PostGIS is automatically enabled on the database when the container first starts. To verify:

```sql
SELECT PostGIS_Version();
```

If you switched from `postgres:16` to `postgis/postgis:16-3.4` on an existing volume, run this manually in your database:

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

## Troubleshooting

If PgAdmin shows `FATAL: password authentication failed for user "postgres"`, the Postgres volume was likely created with a different password before `root123` was set in the compose file.

Reset the database volume and start it again:

```powershell
docker-compose -f database/pgdb/docker-compose.yml down -v
docker-compose -f database/pgdb/docker-compose.yml up -d
```

Then connect with:

- **Host:** `postgres`
- **Port:** `5432`
- **Username:** `postgres`
- **Password:** `root123`
- **Database:** `postgres`