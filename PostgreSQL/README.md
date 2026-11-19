# PostgreSQL with Docker Compose

A ready-to-run PostgreSQL 17 setup with credentials from a `.env` file, data stored in a local folder, and automatic deletion of logs older than 30 days.

## Key benefits

- **One-command setup**: `docker compose up -d` gives you a configured PostgreSQL instance. There's no manual install and nothing to change on the host system.
- **Data stays on your disk**: database files and logs are stored in `./postgres-data/` next to the compose file, so they survive container restarts, rebuilds and `docker compose down`. They're also easy to back up, move or inspect.
- **Automatic log cleanup**: PostgreSQL writes one log file per day, and a small cleaner container deletes files older than 30 days (configurable), so logs never fill the disk.
- **Capped Docker logs**: the containers' own logs are size-limited with log rotation.
- **Credentials kept out of the code**: username, password, database name and port live in `.env`, which is git-ignored.
- **No permission headaches**: a one-time init container sets the correct ownership on the log folder automatically.
- **Health check built in**: `pg_isready` reports when the database is actually ready to accept connections.
- **Reproducible**: the pinned image version means everyone on the team, and every server, runs the same setup.
- **Portable**: works the same on Linux, macOS and Windows (Docker Desktop or WSL2).

## Folder structure

```
.
├── docker-compose.yml
├── .env.example          # template, committed to git
├── .env                  # your real settings, NOT committed
├── .gitignore
├── README.md
└── postgres-data/        # created on first start, NOT committed
    ├── db/               # database files
    └── logs/             # daily PostgreSQL logs (auto-cleaned)
```

## Requirements

- [Docker](https://docs.docker.com/get-docker/) with Docker Compose v2 (`docker compose version`)

## Getting started

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
```

### 2. Create your `.env` file

```bash
cp .env.example .env
```

Edit `.env` and set a strong password:

```env
POSTGRES_USER=admin
POSTGRES_PASSWORD=change_me_strong_password
POSTGRES_DB=appdb
POSTGRES_PORT=5432
LOG_RETENTION_DAYS=30
TZ=Asia/Kolkata
DATA_DIR=./postgres-data
```

| Variable             | Description                                | Default           |
| -------------------- | ------------------------------------------ | ----------------- |
| `POSTGRES_USER`      | Database superuser name                    | `admin`           |
| `POSTGRES_PASSWORD`  | Superuser password                         | (must be set)     |
| `POSTGRES_DB`        | Database created on first start            | `appdb`           |
| `POSTGRES_PORT`      | Port on the host machine                   | `5432`            |
| `LOG_RETENTION_DAYS` | Delete log files older than this           | `30`              |
| `TZ`                 | Timezone for the server and log timestamps | `Asia/Kolkata`    |
| `DATA_DIR`           | Folder where DB files and logs are stored  | `./postgres-data` |

> **Note:** the user, password and database are only applied the **first time**, when `postgres-data/db` is empty. To change them later, use SQL (`ALTER USER ...`) or delete the data folder and start over.

### 3. Start PostgreSQL

```bash
docker compose up -d
```

### 4. Check that it's running

```bash
docker compose ps
```

After about 10 seconds, `postgres` should show **healthy**. `log-init` will show as exited, which is expected because it only runs once.

### 5. Connect

From the terminal:

```bash
docker exec -it postgres psql -U admin -d appdb
```

From an app or GUI tool (DBeaver, pgAdmin, TablePlus):

| Setting  | Value                            |
| -------- | -------------------------------- |
| Host     | `localhost`                      |
| Port     | `5432` (or your `POSTGRES_PORT`) |
| User     | value of `POSTGRES_USER`         |
| Password | value of `POSTGRES_PASSWORD`     |
| Database | value of `POSTGRES_DB`           |

Connection string:

```
postgresql://admin:your_password@localhost:5432/appdb
```

## Everyday commands

```bash
docker compose ps                  # status
docker compose logs -f postgres    # follow container output (Ctrl+C to exit)
docker compose stop                # stop
docker compose start               # start again
docker compose restart postgres    # restart
docker compose down                # stop and remove containers (data is kept)
```

## Logs

- PostgreSQL logs are written to `postgres-data/logs/postgresql-YYYY-MM-DD.log`, one file per day.
- The `log-cleaner` container checks once a day and deletes files older than `LOG_RETENTION_DAYS`.
- View today's log:

```bash
sudo tail -f postgres-data/logs/postgresql-$(date +%F).log
```

- See what the cleaner removed:

```bash
docker compose logs log-cleaner
```

## Backup and restore

Backup:

```bash
docker exec -t postgres pg_dump -U admin -d appdb > backup_$(date +%F).sql
```

Restore:

```bash
cat backup_2026-11-19.sql | docker exec -i postgres psql -U admin -d appdb
```

## Reset everything

**Warning: this deletes all data.**

```bash
docker compose down
sudo rm -rf postgres-data
docker compose up -d
```

## Security notes

- Never commit `.env`. It's already listed in `.gitignore`.
- Use a strong password.
- If the machine is reachable from the internet, expose the port only on localhost by changing the port line in `docker-compose.yml` to:

```yaml
ports:
  - "127.0.0.1:${POSTGRES_PORT:-5432}:5432"
```

## Troubleshooting

| Problem                                        | Fix                                                                                                            |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `docker: 'compose' is not a docker command`    | You have Compose v1. Use `docker-compose up -d` or install Compose v2.                                         |
| `port is already allocated`                    | Another PostgreSQL is using 5432. Change `POSTGRES_PORT` in `.env`.                                            |
| `password authentication failed`               | The credentials were set on the first run. Use the original ones or reset the data folder.                     |
| Permission denied when reading `postgres-data` | Files are owned by the container's postgres user (uid 999). Use `sudo`.                                        |
| Upgrading to PostgreSQL 18+                    | Change the volume to `.../db:/var/lib/postgresql`, remove `PGDATA`, and migrate data with `pg_dump` / restore. |
