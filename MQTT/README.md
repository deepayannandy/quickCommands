# MQTT Broker (Mosquitto) with Docker Compose

A ready-to-run Eclipse Mosquitto 2.0 MQTT broker with password login, credentials from a `.env` file, data stored in a local folder, daily log rotation, and automatic deletion of logs older than 30 days.

## Key benefits

- **One-command setup**: `docker compose up -d` gives you a secured broker. The login user is created from `.env` automatically, with no need to run `mosquitto_passwd` by hand.
- **No anonymous access**: every client must log in, on both MQTT and WebSockets.
- **MQTT and WebSockets**: port `1883` for devices and backends, port `9001` for browsers.
- **Data stays on your disk**: retained messages, sessions, logs and the password file are stored in `./mosquitto-data/`, so they survive container restarts, rebuilds and `docker compose down`.
- **Automatic log rotation and cleanup**: a small cleaner container rotates `mosquitto.log` once a day and deletes rotated files older than 30 days (configurable), so logs never fill the disk.
- **Capped Docker logs**: the containers' own logs are size-limited with log rotation.
- **Credentials kept out of the code**: username, password and ports live in `.env`, which is git-ignored.
- **No permission headaches**: a one-time init container sets the correct ownership on the data folders.
- **Health check built in**: the broker is marked healthy only when a logged-in client can actually subscribe.
- **Reproducible**: the pinned image version means every server runs the same setup.

## Folder structure

```
.
├── docker-compose.yml
├── mosquitto.conf        # broker config, committed to git
├── sample.env            # template, committed to git
├── .env                  # your real settings, NOT committed
├── README.md
└── mosquitto-data/       # created on first start, NOT committed
    ├── data/             # retained messages and sessions
    ├── log/              # mosquitto.log + daily rotated logs (auto-cleaned)
    └── auth/             # password file
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
cd quickCommands/MQTT
```

Or download only the three files you need:

```bash
mkdir -p ~/mqtt && cd ~/mqtt
BASE=https://raw.githubusercontent.com/deepayannandy/quickCommands/main/MQTT
curl -fsSL -o docker-compose.yml $BASE/docker-compose.yml
curl -fsSL -o mosquitto.conf     $BASE/mosquitto.conf
curl -fsSL -o sample.env         $BASE/sample.env
```

### 2. Create your `.env` file

```bash
cp sample.env .env
```

Edit `.env` and set a strong password:

```env
MQTT_USERNAME=admin
MQTT_PASSWORD=change_me_strong_password
MQTT_PORT=1883
MQTT_WS_PORT=9001
LOG_RETENTION_DAYS=30
DATA_DIR=./mosquitto-data
```

| Variable             | Description                               | Default            |
| -------------------- | ----------------------------------------- | ------------------ |
| `MQTT_USERNAME`      | Login user created on start               | `admin`            |
| `MQTT_PASSWORD`      | Password for that user                    | (must be set)      |
| `MQTT_PORT`          | MQTT port on the host machine             | `1883`             |
| `MQTT_WS_PORT`       | WebSocket port on the host machine        | `9001`             |
| `LOG_RETENTION_DAYS` | Delete rotated logs older than this       | `30`               |
| `DATA_DIR`           | Folder where data, logs and passwords go  | `./mosquitto-data` |

> **Note:** avoid `$` in the password, because Docker Compose treats it as a variable. Unlike PostgreSQL and MongoDB, changing the password in `.env` and running `docker compose up -d` again **does** update it.

### 3. Start the broker

```bash
docker compose up -d
```

### 4. Check that it's running

```bash
docker compose ps
```

After about 10 seconds, `mosquitto` should show **healthy**. `mqtt-init` will show as exited, which is expected because it only runs once per start.

### 5. Test it

Open two terminals on the server. In the first, subscribe:

```bash
docker exec -it mosquitto mosquitto_sub -u admin -P 'your_password' -t 'test/#' -v
```

In the second, publish:

```bash
docker exec -it mosquitto mosquitto_pub -u admin -P 'your_password' -t 'test/hello' -m 'it works'
```

The first terminal should print `test/hello it works`.

### 6. Connect

| Client                           | URL                                      |
| -------------------------------- | ---------------------------------------- |
| Devices, Node.js, Python, MQTTX  | `mqtt://<server_ip>:1883`                |
| Browsers (MQTT.js over WebSocket)| `ws://<server_ip>:9001`                  |

Use the username and password from `.env`. In Node.js with [MQTT.js](https://github.com/mqttjs/MQTT.js):

```js
const mqtt = require("mqtt");
const client = mqtt.connect("mqtt://localhost:1883", {
  username: process.env.MQTT_USERNAME,
  password: process.env.MQTT_PASSWORD,
});
```

## Managing users

The user from `.env` is managed for you. Add more users (for example one per device or service):

```bash
docker exec -it mosquitto mosquitto_passwd /mosquitto/auth/passwd device01
docker compose restart mosquitto
```

Remove a user:

```bash
docker exec -it mosquitto mosquitto_passwd -D /mosquitto/auth/passwd device01
docker compose restart mosquitto
```

Extra users are kept when you restart or run `docker compose up -d` again.

## Everyday commands

```bash
docker compose ps                   # status
docker compose logs -f mosquitto    # follow broker output (Ctrl+C to exit)
docker compose stop                 # stop
docker compose start                # start again
docker compose restart mosquitto    # restart (after editing mosquitto.conf)
docker compose down                 # stop and remove containers (data is kept)
```

## Logs

- The current broker log is `mosquitto-data/log/mosquitto.log`. It records client connections, disconnections, warnings and errors.
- Once a day the cleaner rotates it to `mosquitto.log.YYYY-MM-DDTHH-MM-SS` and empties the current file. The first rotation happens when the stack starts, then every 24 hours after that.
- Rotated files older than `LOG_RETENTION_DAYS` are deleted automatically.
- Follow the current log:

```bash
sudo tail -f mosquitto-data/log/mosquitto.log
```

- See what the cleaner rotated and removed:

```bash
docker compose logs log-cleaner
```

## Reset everything

**Warning: this deletes all retained messages, sessions, logs and extra users.**

```bash
docker compose down
sudo rm -rf mosquitto-data
docker compose up -d
```

## Security notes

- Never commit `.env`. Only `sample.env` belongs in git.
- Use a strong password, and preferably a separate user per device or service so you can remove one without touching the others.
- Plain `mqtt://` and `ws://` aren't encrypted. If clients connect over the internet, use TLS: put the WebSocket port behind Nginx with HTTPS (see [NginxConfigs](../NginxConfigs)) so browsers connect with `wss://`, or add a TLS listener on port 8883 in `mosquitto.conf`.
- If only apps on the same server use the broker, expose the ports only on localhost by changing the port lines in `docker-compose.yml` to:

```yaml
ports:
  - "127.0.0.1:${MQTT_PORT:-1883}:1883"
  - "127.0.0.1:${MQTT_WS_PORT:-9001}:9001"
```

## Troubleshooting

| Problem                                     | Fix                                                                                                         |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `docker: 'compose' is not a docker command` | You have Compose v1. Use `docker-compose up -d` or install Compose v2.                                      |
| `port is already allocated`                 | Another broker is using 1883 or 9001. Change `MQTT_PORT` / `MQTT_WS_PORT` in `.env`.                        |
| `Connection Refused: not authorised`        | Wrong username or password. Check `.env`, then run `docker compose up -d` to re-apply it.                   |
| `mqtt-init` exits with an error             | `MQTT_USERNAME` or `MQTT_PASSWORD` is empty in `.env`. Check with `docker compose logs mqtt-init`.          |
| `mosquitto` stays **unhealthy**             | Check `docker compose logs mosquitto`. A typo in `mosquitto.conf` stops the broker from starting.           |
| Broker won't start after editing the config | Mosquitto doesn't allow comments after a value (`listener 1883 # mqtt` breaks it). Put comments on their own line. |
| Permission denied when reading `mosquitto-data` | Files are owned by the container's mosquitto user (uid 1883). Use `sudo`.                               |
