# RabbitMQ Setup

This folder runs a local RabbitMQ broker (with the management UI plugin) using
Docker Compose. Any API can connect to it to publish/consume messages.

## What this gives you

The `docker-compose.yml` file in this folder starts:

- **RabbitMQ** (`rabbitmq:3-management-alpine`) — AMQP broker on port `5672`
- **Management UI** on port `15672`
- A login: user `admin`, password `root123`
- A storage volume `rabbitmq_data` so queues/messages survive a restart

## Prerequisites

- Docker and Docker Compose installed
- Port `5672` available (AMQP)
- Port `15672` available (management UI)

## 1. Start RabbitMQ

```powershell
cd rabbitmq
docker compose up -d
```

Check that it is running:

```powershell
docker ps --filter "name=rabbitmq"
```

You should see the `rabbitmq` container with ports `5672` and `15672` mapped.

## 2. Connect your API

```env
# RabbitMQ
RABBITMQ_URI=amqp://admin:root123@localhost:5672
RABBITMQ_EXCHANGE=<your_exchange_name>
```

## 3. Run your API

Start your API the normal way for your project, for example:

```powershell
npm install
npm run start:dev
```

When it starts, it should connect to RabbitMQ using the values from `.env`.

## 4. Check that it works

- **Management UI:** open `http://localhost:15672` and log in with `admin` /
  `root123`. Check the **Exchanges** and **Queues** tabs to see what your API
  has declared, and **publish/deliver** counters to confirm messages are
  flowing.
- **CLI:** `docker exec -it rabbitmq rabbitmqctl list_exchanges` or
  `docker exec -it rabbitmq rabbitmqctl list_queues`.

## Access Services

- **AMQP (from host or other services):** `localhost:5672`
- **Management UI:** `http://localhost:15672`

## Start, stop, and remove the broker

```powershell
# Stop the broker (your data stays)
docker compose stop

# Stop and remove the container (your data stays in the volume)
docker compose down

# Remove the container AND delete all data
docker compose down -v
```

## Troubleshooting

- **`ACCESS_REFUSED` / login fails** — Your connection string must include the
  credentials: `amqp://admin:root123@localhost:5672`. Without them, the client
  falls back to `guest`/`guest`, which RabbitMQ only allows from inside the
  container by default.
- **Port `5672` or `15672` already in use** — Another RabbitMQ (or something
  else) is already running. Stop it, or change the host-side port in
  `docker-compose.yml` (for example `"5673:5672"`) and update `RABBITMQ_URI`
  to match.
- **API can't connect** — Make sure the container is running (`docker ps`) and
  that `RABBITMQ_URI` points to `localhost:5672` with the correct credentials.
