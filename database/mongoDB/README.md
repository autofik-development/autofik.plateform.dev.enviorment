# MongoDB Setup with Docker

This folder runs a local MongoDB database using Docker. You can connect any API
to it. Just follow the steps below to start the database and connect your API.

> Replace anything in `<...>` with your own values (for example your API folder
> path and your database name).

## What this gives you

The `docker-compose.yml` file in this folder starts:

- **MongoDB** (`mongo:latest`) on port `27017`
- A login: user `admin`, password `root123`
- A storage volume `mongodb_data` so your data is kept even after a restart

## Before you start

- Install [Docker Desktop](https://www.docker.com/products/docker-desktop/) and make sure it is running
- Have your API code ready in its own folder

## 1. Start MongoDB

Open this folder (`database/mongoDB`) and run:

```powershell
cd database
cd mongoDB
docker compose up -d
```

Check that it is running:

```powershell
docker ps --filter "name=mongodb"
```

You should see the `mongodb` container with port `0.0.0.0:27017->27017/tcp`.

## 2. Connect your API

```env
# MongoDB
MONGODB_URI=mongodb://admin:root123@localhost:27017/<your_database_name>?authSource=admin
MONGODB_DATABASE=<your_database_name>
```

## 3. Run your API

Start your API the normal way for your project, for example:

```powershell
npm install
npm run start:dev
```

When it starts, it should connect to MongoDB using the values from `.env`.

## 4. Check that it works

Open the database with `mongosh` to see your data:

```powershell
docker exec -it mongodb mongosh -u admin -p root123 --authenticationDatabase admin
```

```javascript
use <your_database_name>
db.getCollectionNames()
```

If your API has already saved some data, you will see your collections listed.

## 5. See your data in a GUI (optional)

[MongoDB Compass](https://www.mongodb.com/products/tools/compass) is a free app
that lets you view and edit your data with a mouse instead of commands.

1. Download and install Compass from the link above.
2. Make a new connection with this link (same login the API uses):

   ```text
   mongodb://admin:root123@localhost:27017/?authSource=admin
   ```

3. Click **Connect**, then open your database to see your collections and
   documents.

## Start, stop, and remove the database

```powershell
# Stop the database (your data stays)
docker compose stop

# Stop and remove the container (your data stays in the volume)
docker compose down

# Remove the container AND delete all data
docker compose down -v
```

## If something goes wrong

- **`Authentication failed`** — Your link must have the user, password, and
  `?authSource=admin`. Without `authSource`, MongoDB looks for the user in the
  wrong place and the login fails.
- **Port `27017` already in use** — Another MongoDB is already running. Stop it,
  or change the port in `docker-compose.yml` (for example `"27018:27017"`) and
  update `MONGODB_URI` to match.
- **API can't connect** — Make sure the container is running (`docker ps`) and
  that `MONGODB_URI` points to `localhost:27017`.
