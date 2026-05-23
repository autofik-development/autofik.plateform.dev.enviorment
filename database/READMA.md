# Database Setup - PostgreSQL & PgAdmin

This setup creates a PostgreSQL database and PgAdmin management interface using Docker Compose.

## Prerequisites

- Docker and Docker Compose installed
- Port 5432 available (PostgreSQL)
- Port 8080 available (PgAdmin)

## Quick Start

### 1. Create the Docker Network

```powershell
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