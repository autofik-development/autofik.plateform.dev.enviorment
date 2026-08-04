# Kibana + Elasticsearch Setup

## Stack

| Service | Image | Port |
|---------|-------|------|
| Elasticsearch | `elasticsearch:8.19.0` | `9200` |
| Kibana | `kibana:8.19.0` | `5601` |

---

## Files

```
kibana/
├── docker-compose.yml   — 3 services: elasticsearch, setup, kibana
├── kibana.yml           — Kibana config mounted into the container
└── README.md
```

---

## Before Running

### 1. Check Docker is running

```bash
docker info
```

If Docker is not running, start Docker Desktop first.

### 2. Check ports are free

Make sure nothing is already using ports `9200` and `5601`.

```bash
# Windows
netstat -ano | findstr :9200
netstat -ano | findstr :5601
```

If a port is in use, stop the process using it or change the port in `docker-compose.yml`.

### 3. Confirm both config files exist side by side

```
kibana/
├── docker-compose.yml   ✓ must exist
├── kibana.yml           ✓ must exist  ← if missing, Kibana will not start correctly
```

`kibana.yml` is mounted into the Kibana container. Without it the container starts
but encryption keys are missing and Actions/Alerting APIs are disabled.

### 4. kibana.yml — what each line does

```yaml
server.host: "0.0.0.0"                          # allow connections from outside the container
                                                 # without this you get ERR_EMPTY_RESPONSE in browser

elasticsearch.hosts: ["http://elasticsearch:9200"]  # internal Docker network hostname
elasticsearch.username: "kibana_system"          # internal service account (not for login)
elasticsearch.password: "KibanaSys12345"         # must match setup service in docker-compose.yml

xpack.security.encryptionKey: "..."              # fixes random key WARN on restart
xpack.encryptedSavedObjects.encryptionKey: "..."  # required for Actions/Alerting APIs
xpack.reporting.encryptionKey: "..."             # required for PDF/CSV reporting
```

### 5. Password rules — both files must match

| File | Setting | Default value |
|------|---------|---------------|
| `docker-compose.yml` | `ELASTIC_PASSWORD` | `Elastic12345` |
| `docker-compose.yml` | setup `-d` password | `KibanaSys12345` |
| `kibana.yml` | `elasticsearch.password` | `KibanaSys12345` |

> The two `KibanaSys12345` values **must be identical**.
> If they differ, Kibana cannot connect to Elasticsearch.

### 6. Encryption key rules

Each of the three keys in `kibana.yml` must be **at least 32 characters**.
They can be any string — just keep them fixed so sessions survive restarts.

Generate a random key:
```bash
openssl rand -hex 32
```

> If you change a key after Kibana has saved data, existing saved objects
> (connectors, rules, reports) will become unreadable. Keep keys stable.

---

## Running

```bash
docker compose up -d
```

### Startup sequence

```
Step 1 — elasticsearch starts
         └── healthcheck polls until cluster is ready (up to 5 min)

Step 2 — setup runs (one-time only, exits when done)
         └── calls Elasticsearch API to set kibana_system password
         └── returns {} on success

Step 3 — kibana starts
         └── reads kibana.yml from mounted volume
         └── connects to Elasticsearch as kibana_system
         └── logs "Kibana is now available"
         └── ready at http://localhost:5601
```

Kibana takes ~60 seconds to fully initialize after the container starts.

---

## Login

Open in browser: **http://localhost:5601**

| Field | Value |
|-------|-------|
| Username | `elastic` |
| Password | `Elastic12345` |

> Do NOT log in as `kibana_system` — it is an internal service account used only
> by Kibana to talk to Elasticsearch. It has no UI access.

---

## Verify everything is working

```bash
# 1. All containers running
docker compose ps

# 2. Setup succeeded (should show {} and exit code 0)
docker logs elasticsearch-setup

# 3. Kibana reachable
curl -s -o /dev/null -w "%{http_code}" http://localhost:5601/api/status
# expected: 200

# 4. Elasticsearch reachable
curl -s -u elastic:Elastic12345 http://localhost:9200/_cluster/health
# expected: "status":"green" or "status":"yellow"
```

---

## Stopping

```bash
# Stop containers but keep all data
docker compose down

# Stop and delete all data (full clean slate)
docker compose down -v
```

---

## Troubleshooting

### `ERR_EMPTY_RESPONSE` in browser

`server.host: "0.0.0.0"` is missing from `kibana.yml`.
Without it, Kibana only listens on `127.0.0.1` inside the container
and is unreachable from the browser.

Check the log — it will say one of:
- `http server running at http://localhost:5601` → missing server.host (broken)
- `http server running at http://0.0.0.0:5601`  → correct (working)

Fix: add `server.host: "0.0.0.0"` as the first line of `kibana.yml`, then restart.

---

### `security_exception: unable to authenticate user [kibana_system]`

The setup service did not run or failed. This happens when:
- The Docker volume already exists from a previous run with different passwords
- The setup container was skipped

Fix — full clean restart:
```bash
docker compose down -v
docker compose up -d
```

Verify setup ran:
```bash
docker logs elasticsearch-setup
# expected output: {}
```

---

### `APIs are disabled: Encrypted Saved Objects plugin is missing encryption key`

The `kibana.yml` is not mounted or encryption keys are missing.

Check:
1. `kibana.yml` exists in the same folder as `docker-compose.yml`
2. The volume line in `docker-compose.yml` is correct:
   ```yaml
   volumes:
     - ./kibana.yml:/usr/share/kibana/config/kibana.yml
   ```
3. All three `xpack.*encryptionKey` lines are present in `kibana.yml`

Fix — recreate the Kibana container:
```bash
docker compose up -d kibana --force-recreate
```

---

### `value of "elastic" is forbidden`

You used `elastic` as `ELASTICSEARCH_USERNAME` in `docker-compose.yml`.
Kibana 8.x blocks the superuser from being used as the connector.

Fix: use `kibana_system` as the username (already set correctly in `kibana.yml`).

---

### Kibana keeps crashing or restarting

Check Elasticsearch is healthy first — Kibana depends on it:
```bash
docker logs elasticsearch --tail 30
docker compose ps
```

If Elasticsearch is not healthy, give it more time or increase memory:
```yaml
ES_JAVA_OPTS: -Xms1g -Xmx1g   # in docker-compose.yml
```

---

## Why 3 services?

Elasticsearch starts with `kibana_system` locked (no password).
Kibana 8.x requires `kibana_system` — it blocks the `elastic` superuser.
The `setup` service runs once to unlock `kibana_system` via the ES API.

```
elastic user      →  auto-created by ELASTIC_PASSWORD          ✓
kibana_system     →  exists but locked → setup service sets it  ✓
```

There is no other way to do this. Elasticsearch does not support setting
`kibana_system` password via environment variable on startup.

---

## Why kibana.yml instead of environment variables?

Kibana config uses camelCase keys (e.g. `encryptionKey`).
Environment variables are uppercase and case-insensitive, so:

```
XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY
  resolves to → xpack.encryptedsavedobjects.encryptionkey   (wrong case)
  expected    → xpack.encryptedSavedObjects.encryptionKey   (camelCase)
```

Kibana does not match them. Mounting `kibana.yml` sets the exact key
names Kibana expects and always works.
