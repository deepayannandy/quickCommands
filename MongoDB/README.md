# Setup MongoDB in Docker

Please follow the step to install MongoDB in Ubuntu 24 using docker compose.

#### Key Highlight

- Store all the DB in your Persistent storage.
- Keep a limit and retention period for the logs.

### Step 1

Make your system ready for the installation.

```
sudo apt-get update && sudo apt-get upgrade

```

### Step 2

Install Docker.

```
curl -sSL https://get.docker.com | sh

```

### Step 3

Create a directory to store all container related data in one place

```
sudo mkdir -p /db/mongodb

cd /db/mongodb

```

### Step 4

Download the mongodb-docker-compose.yml file

```
curl https://raw.githubusercontent.com/deepayannandy/quickCommands/refs/heads/main/MongoDB/mongo-docker-compose.yml | cat >> mongo-docker-compose.yml

```

### Step 6

Make a directory to store all the logs and Database files

```
mkdir ./mongodb_data

```

### Step 7

Let's start your MongoDB using the compose file

```
docker compose up -d

```

## Additional points

Use this connection string to connect your Data base using Compass

```
mongodb://<username>:<password>@<HostName>:<port>/

```

Use this connection string to connect your Data base using JS

```
mongodb://<username>:<password>@<HostName>:<port>/<collectionName>?authSource=admin

```
