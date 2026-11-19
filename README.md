# Quick Commands

A repository where I keep ready-to-use setups and commands for deploying backend services on Ubuntu servers, all in one place.

## What's inside

| Folder                                                          | What it sets up                                                                                         |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| [PostgreSQL](PostgreSQL)                                        | PostgreSQL 17 in Docker Compose: `.env` credentials, local data folder, daily logs with auto-cleanup    |
| [MongoDB](MongoDB)                                              | MongoDB 8.0 in Docker Compose: `.env` credentials, local data folder, daily log rotation and cleanup    |
| [EMQX](EMQX)                                                    | EMQX MQTT broker in Docker Compose, with the web dashboard                                              |
| [MQTT](MQTT)                                                    | Eclipse Mosquitto MQTT broker in Docker Compose: password login from `.env`, WebSockets, daily log rotation and cleanup |
| [NginxConfigs](NginxConfigs)                                    | Nginx reverse proxy for Node.js APIs, static hosting for React apps, and free HTTPS with Certbot        |
| [SystemD setup for NodeJs App](SystemD%20setup%20for%20NodeJs%20App) | Run a Node.js app as a systemd service: starts on boot, restarts on crash, logs in journald         |

Each folder has its own README with step-by-step instructions.

## A typical server setup

1. Install Docker: `curl -sSL https://get.docker.com | sh`
2. Start the databases or brokers you need ([PostgreSQL](PostgreSQL), [MongoDB](MongoDB), [EMQX](EMQX) or [MQTT](MQTT)).
3. Run your Node.js backend as a [systemd service](SystemD%20setup%20for%20NodeJs%20App).
4. Put [Nginx](NginxConfigs) in front of it and add HTTPS with Certbot.

## Requirements

- Ubuntu 22.04 / 24.04 (most steps also work on other Linux distributions)
- Docker with Docker Compose v2 for the database and broker setups
- `sudo` access on the server

## Tech Stack

**Tools:** Docker, Docker Compose, Nginx, Certbot, systemd, PostgreSQL, MongoDB, MQTT (Mosquitto, EMQX)

**Deployed on:** AWS, Proxmox, Bare metal server

## Authors

- [@Deepayan Nandy](https://www.github.com/deepayannandy)
