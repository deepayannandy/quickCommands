# MongoDB with Docker Compose

A ready-to-run MongoDB 8.0 setup with credentials from a `.env` file, data stored in a local folder, daily log rotation, and automatic deletion of logs older than 30 days.

## Key benefits

- **One-command setup**: `docker compose up -d` gives you a configured MongoDB instance with authentication enabled. There's no manual install and nothing to change on the host system.
- **Data stays on your disk**: database files and logs are stored in `./mongodb-data/` next to the compose file, so they survive container restarts, rebuilds and `docker compose down`. They're also easy to back up, move or inspect.
- **Automatic log rotation and cleanup**: a small cleaner container rotates `mongod.log` once a day and deletes rotated files older than 30 days (configurable), so logs never fill the disk.
- **Capped Docker logs**: the containers' own logs are size-limited with log rotation.
- **Credentials kept out of the code**: username, password and port live in `.env`, which is git-ignored.
- **No permission headaches**: a one-time init container sets the correct ownership on the log folder automatically.
- **Health check built in**: a `mongosh` ping reports when the database is actually ready to accept connections.
- **Reproducible**: the pinned image version means everyone on the team, and every server, runs the same setup.
- **Portable**: works the same on Linux, macOS and Windows (Docker Desktop or WSL2).

## Folder structure

```
.
├── docker-compose.yml
├── sample.env            # template, committed to git
├── .env                  # your real settings, NOT committed
├── README.md
└── mongodb-data/         # created on first start, NOT committed
    ├── db/               # database files
    └── logs/             # mongod.log + daily rotated logs (auto-cleaned)
```

## Requirements

- [Docker](https://docs.docker.com/get-docker/) with Docker Compose v2 (`docker compose version`)

On a fresh Ubuntu server you can install Docker with:

```bash
sudo apt-get update && sudo apt-get upgrade -y
curl -sSL https://get.docker.com | sh
```

## Getting started

### 1. Get the files

Clone the repository:

```bash
git clone https://github.com/deepayannandy/quickCommands.git
cd quickCommands/MongoDB
```

Or download only the two files you need:

```bash
mkdir -p ~/mongodb && cd ~/mongodb
curl -fsSL -o docker-compose.yml https://raw.githubusercontent.com/deepayannandy/quickCommands/main/MongoDB/docker-compose.yml
curl -fsSL -o sample.env https://raw.githubusercontent.com/deepayannandy/quickCommands/main/MongoDB/sample.env
```

### 2. Create your `.env` file

```bash
cp sample.env .env
```

Edit `.env` and set a strong password:

```env
MONGO_INITDB_ROOT_USERNAME=admin
MONGO_INITDB_ROOT_PASSWORD=change_me_strong_password
MONGO_PORT=27017
LOG_RETENTION_DAYS=30
TZ=Asia/Kolkata
DATA_DIR=./mongodb-data
```

| Variable                     | Description                                | Default          |
| ---------------------------- | ------------------------------------------ | ---------------- |
| `MONGO_INITDB_ROOT_USERNAME` | Root (admin) user name                     | `admin`          |
| `MONGO_INITDB_ROOT_PASSWORD` | Root user password                         | (must be set)    |
| `MONGO_PORT`                 | Port on the host machine                   | `27017`          |
| `LOG_RETENTION_DAYS`         | Delete rotated logs older than this        | `30`             |
| `TZ`                         | Timezone for the server and log timestamps | `Asia/Kolkata`   |
| `DATA_DIR`                   | Folder where DB files and logs are stored  | `./mongodb-data` |

> **Note:** the root user and password are only applied the **first time**, when `mongodb-data/db` is empty. To change them later, use `mongosh` (`db.changeUserPassword(...)`) or delete the data folder and start over. The log cleaner reads the same values from `.env`, so keep them in sync.

### 3. Start MongoDB

```bash
docker compose up -d
```

### 4. Check that it's running

```bash
docker compose ps
```

After about 20 seconds, `mongodb` should show **healthy**. `log-init` will show as exited, which is expected because it only runs once.

### 5. Connect

From the terminal:

```bash
docker exec -it mongodb mongosh -u admin -p --authenticationDatabase admin
```

From a GUI tool (MongoDB Compass, Studio 3T):

```
mongodb://admin:your_password@localhost:27017/
```

From an app (Node.js, Python, etc.), add the database name and `authSource=admin`, because the root user lives in the `admin` database:

```
mongodb://admin:your_password@localhost:27017/appdb?authSource=admin
```

> MongoDB creates a database the first time you write to it, so `appdb` doesn't need to exist beforehand.

## Everyday commands

```bash
docker compose ps                  # status
docker compose logs -f mongodb     # follow container output (Ctrl+C to exit)
docker compose stop                # stop
docker compose start               # start again
docker compose restart mongodb     # restart
docker compose down                # stop and remove containers (data is kept)
```

## Logs

- The current MongoDB log is `mongodb-data/logs/mongod.log`.
- Once a day the cleaner rotates it to `mongod.log.YYYY-MM-DDTHH-MM-SS` and starts a fresh file. The first rotation happens when the stack starts, then every 24 hours after that.
- Rotated files older than `LOG_RETENTION_DAYS` are deleted automatically.
- Follow the current log (MongoDB logs are JSON, one entry per line):

```bash
sudo tail -f mongodb-data/logs/mongod.log
```

- See what the cleaner rotated and removed:

```bash
docker compose logs log-cleaner
```

## Backup and restore

Backup all databases to a single compressed file:

```bash
docker exec mongodb mongodump -u admin -p your_password --authenticationDatabase admin \
  --archive --gzip > backup_$(date +%F).archive.gz
```

Restore:

```bash
docker exec -i mongodb mongorestore -u admin -p your_password --authenticationDatabase admin \
  --archive --gzip < backup_2026-11-19.archive.gz
```

Add `--db appdb` to `mongodump` to back up a single database, or `--drop` to `mongorestore` to replace existing collections.

## Reset everything

**Warning: this deletes all data.**

```bash
docker compose down
sudo rm -rf mongodb-data
docker compose up -d
```

## Security notes

- Never commit `.env`. Only `sample.env` belongs in git.
- Use a strong password. Authentication is enabled automatically because a root user is set.
- If the machine is reachable from the internet, expose the port only on localhost by changing the port line in `docker-compose.yml` to:

```yaml
ports:
  - "127.0.0.1:${MONGO_PORT:-27017}:27017"
```

## Troubleshooting

| Problem                                       | Fix                                                                                                                 |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `docker: 'compose' is not a docker command`   | You have Compose v1. Use `docker-compose up -d` or install Compose v2.                                              |
| `port is already allocated`                   | Another MongoDB is using 27017. Change `MONGO_PORT` in `.env`.                                                      |
| `Authentication failed`                       | The credentials were set on the first run. Use the original ones or reset the data folder. Also check `authSource=admin`. |
| `log-cleaner` shows authentication errors     | The root password was changed in MongoDB but not in `.env`. Update `.env` and run `docker compose up -d`.           |
| Permission denied when reading `mongodb-data` | Files are owned by the container's mongodb user (uid 999). Use `sudo`.                                              |
| MongoDB won't start on an old CPU             | MongoDB 5.0+ needs a CPU with AVX. On older hardware or some VMs, enable AVX passthrough or use `mongo:4.4`.         |
