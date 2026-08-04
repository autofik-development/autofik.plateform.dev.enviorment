# Debezium Setup

Debezium captures every INSERT / UPDATE / DELETE from PostgreSQL
and streams them as events into Kafka topics in real time.

## Architecture

```
PostgreSQL (WAL)
      │
      ├── db_service_a  →  connector-service-a  →  Kafka topics: service-a.*
      ├── db_service_b  →  connector-service-b  →  Kafka topics: service-b.*
      ├── db_service_c  →  connector-service-c  →  Kafka topics: service-c.*
      └── ...

Debezium (Kafka Connect) — REST API on port 8083
```

One Debezium instance handles all connectors.
Each connector watches one database independently.

---

## Files

```
debezium/
├── docker-compose.yml        — 2 services: debezium + debezium-setup
├── connectors/               — one JSON file per service database
│   ├── service-a.json
│   ├── service-b.json
│   └── ...                   ← add more here, one per service
└── README.md
```

---

## Multiple Databases — One Connector Per Service

**Debezium can only watch one database per connector.**
This is a PostgreSQL limitation (each replication slot is per-database).

For 6 services with 6 databases → create 6 connector JSON files in `connectors/`.

`debezium-setup` automatically registers **all files** in the `connectors/` folder on startup.

### What changes per connector file

Only these 5 fields must be unique per connector:

| Field | Rule | Example |
|-------|------|---------|
| `name` | unique connector name | `connector-service-a` |
| `database.dbname` | your service's database | `db_service_a` |
| `topic.prefix` | Kafka topic prefix | `service-a` |
| `slot.name` | unique replication slot | `debezium_slot_service_a` |
| `publication.name` | unique publication | `debezium_pub_service_a` |

Everything else (`hostname`, `port`, `user`, `password`, `plugin.name`) stays the same.

### Connector file template

Copy this for each service and fill in the 5 unique fields:

```json
{
  "name": "connector-<service-name>",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres",
    "database.port": "5432",
    "database.user": "postgres1",
    "database.password": "root123",
    "database.dbname": "<your-database-name>",
    "topic.prefix": "<service-name>",
    "plugin.name": "pgoutput",
    "slot.name": "debezium_slot_<service_name>",
    "publication.name": "debezium_pub_<service_name>",
    "heartbeat.interval.ms": "5000",
    "snapshot.mode": "initial"
  }
}
```

### Example — 6 services

| Service | Database | topic.prefix | slot.name |
|---------|----------|-------------|-----------|
| service-a | db_service_a | service-a | debezium_slot_service_a |
| service-b | db_service_b | service-b | debezium_slot_service_b |
| service-c | db_service_c | service-c | debezium_slot_service_c |
| service-d | db_service_d | service-d | debezium_slot_service_d |
| service-e | db_service_e | service-e | debezium_slot_service_e |
| service-f | db_service_f | service-f | debezium_slot_service_f |

---

## Dependencies

Debezium depends on both Kafka and PostgreSQL already running.
Start these first:

| Stack | Folder | Must be running |
|-------|--------|-----------------|
| Kafka | `kafka/` | `kafka` container on `kafka-network` |
| PostgreSQL | `database/pgdb/` | `postgres` container on `postgres-network` |

---

## Before Running

### 1. Enable logical replication on PostgreSQL

Already added to `database/pgdb/docker-compose.yml`:
```yaml
command: ["postgres", "-c", "wal_level=logical"]
```

**Restart PostgreSQL once** for this to take effect:
```bash
cd ../database/pgdb
docker compose down && docker compose up -d
```

Verify:
```bash
docker exec postgres psql -U postgres1 -c "SHOW wal_level;"
# expected: logical
```

### 2. Confirm Docker networks exist

```bash
docker network ls | grep kafka-network
docker network ls | grep postgres-network
```

If missing:
```bash
docker network create kafka-network
docker network create postgres-network
```

### 3. Create connector files for your services

Add one JSON file per service into `connectors/`.
Use the template above. Name the file `<service-name>.json`.

`debezium-setup` registers every file in `connectors/` automatically on startup.

### 4. Make sure each database exists in PostgreSQL

Debezium can only connect to a database that already exists.
Create missing databases before starting:

```bash
docker exec postgres psql -U postgres1 -c "CREATE DATABASE db_service_a;"
docker exec postgres psql -U postgres1 -c "CREATE DATABASE db_service_b;"
# repeat for each service
```

---

## Running

```bash
docker compose up -d
```

### Startup sequence

```
Step 1 — debezium starts (Kafka Connect)
         └── connects to Kafka at kafka:29092
         └── creates 3 internal Kafka topics
         └── healthcheck polls until REST API is ready

Step 2 — debezium-setup runs (one-time, exits when done)
         └── loops through every file in connectors/
         └── POSTs each connector to Debezium REST API
         └── Debezium creates a replication slot per database
         └── takes initial snapshot of existing data
         └── starts streaming WAL changes to Kafka
```

---

## Kafka Topics

Each connector creates topics automatically:

```
<topic.prefix>.<schema>.<table>
```

Example for `service-a` with a table `orders` in schema `public`:
```
service-a.public.orders
```

Internal Debezium topics (shared across all connectors):
```
debezium_connect_configs
debezium_connect_offsets
debezium_connect_statuses
```

---

## Verify it is working

```bash
# 1. Debezium REST API is up
curl -s http://localhost:8083/ | grep version

# 2. List all registered connectors
curl -s http://localhost:8083/connectors

# 3. Check status of one connector
curl -s http://localhost:8083/connectors/connector-service-a/status

# 4. List Kafka topics
docker exec kafka /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 --list
```

Expected connector status:
```json
{
  "connector": { "state": "RUNNING" },
  "tasks": [{ "state": "RUNNING" }]
}
```

---

## Adding a new connector after startup

No need to restart Debezium. Register it directly via REST API:

```bash
curl -X POST -H "Content-Type: application/json" \
  http://localhost:8083/connectors \
  -d @connectors/service-c.json
```

---

## Stopping

```bash
# Stop Debezium (connectors stay registered in Kafka)
docker compose down

# Remove a specific connector
curl -X DELETE http://localhost:8083/connectors/connector-service-a

# Remove all connectors
curl -s http://localhost:8083/connectors | \
  tr -d '[]"' | tr ',' '\n' | \
  xargs -I{} curl -X DELETE http://localhost:8083/connectors/{}
```

---

## Troubleshooting

### `replication slot "debezium_slot_service_a" already exists`

A previous run left the slot in PostgreSQL. Drop it:
```bash
docker exec postgres psql -U postgres1 -d db_service_a -c \
  "SELECT pg_drop_replication_slot('debezium_slot_service_a');"
```

Then re-register:
```bash
curl -X DELETE http://localhost:8083/connectors/connector-service-a
curl -X POST -H "Content-Type: application/json" \
  http://localhost:8083/connectors -d @connectors/service-a.json
```

### `connector already exists` on startup

The connector was registered in a previous run and is stored in Kafka.
It will persist even after `docker compose down` (data is in `debezium_connect_configs` Kafka topic).

To force re-registration, delete first:
```bash
curl -X DELETE http://localhost:8083/connectors/connector-service-a
```

### `database "db_service_a" does not exist`

Create the database before registering the connector:
```bash
docker exec postgres psql -U postgres1 -c "CREATE DATABASE db_service_a;"
```

### `wal_level is not logical`

PostgreSQL was not restarted after adding the command flag:
```bash
cd ../database/pgdb
docker compose down && docker compose up -d
```

### Connector status is `FAILED`

Check the error message:
```bash
curl -s http://localhost:8083/connectors/connector-service-a/status
```

Common causes:
- Wrong `database.dbname` — database does not exist
- Wrong password in connector JSON
- `wal_level` not set to `logical`
- PostgreSQL container not on `postgres-network`
