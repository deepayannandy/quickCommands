# Run a Node.js App with systemd

Keep a Node.js backend running 24/7 on an Ubuntu server: it starts on boot, restarts after a crash, and its logs are kept and rotated automatically.

## Key benefits

- **Automatic restarts**: if the app crashes (unhandled exception, out of memory), systemd restarts it after 5 seconds.
- **Starts on boot**: the app comes back by itself after a reboot or power loss.
- **Logs handled for you**: everything from `console.log` and `console.error` goes to journald, which rotates and size-limits logs automatically.
- **Resource limits**: optionally cap memory and CPU so the app can't starve the rest of the server.
- **Startup order**: waits for the network, and optionally for Docker (for a database container on the same server), before starting.
- **Nothing extra to install**: systemd is already part of Ubuntu, so there's no PM2 or other process manager to keep updated.

## Files

| File                             | What it is                                       |
| -------------------------------- | ------------------------------------------------ |
| [nodejs.service](nodejs.service) | Service file template. Replace every `<...>` value |

## Setup

The examples use an app called `my-app` in `/home/ubuntu/my-app`, run by the `ubuntu` user. Change these to match your server.

### 1. Check the app runs by hand

```bash
cd /home/ubuntu/my-app
npm ci --omit=dev
node index.js
```

If it starts without errors, stop it with Ctrl+C.

### 2. Find the values you need

```bash
whoami          # user        -> ubuntu
pwd             # app folder  -> /home/ubuntu/my-app
which node      # node path   -> /usr/bin/node
```

> **Using nvm?** `which node` gives a path like `/home/ubuntu/.nvm/versions/node/v22.11.0/bin/node`. Use that full path. When you upgrade Node with nvm, update `ExecStart` to the new path.

### 3. Create the service file

The file name becomes the service name. Download the template:

```bash
sudo curl -fsSL -o /etc/systemd/system/my-app.service \
  "https://raw.githubusercontent.com/deepayannandy/quickCommands/main/SystemD%20setup%20for%20NodeJs%20App/nodejs.service"
sudo nano /etc/systemd/system/my-app.service
```

Replace every `<...>` value. Filled in, the main part looks like this:

```ini
[Service]
User=ubuntu
Group=ubuntu
WorkingDirectory=/home/ubuntu/my-app
Environment=NODE_ENV=production
EnvironmentFile=/home/ubuntu/my-app/.env
ExecStart=/usr/bin/node index.js
SyslogIdentifier=my-app
```

- No `.env` file? Delete the `EnvironmentFile=` line, or the service won't start.
- Started with `npm start`? Use the file it runs (for example `ExecStart=/usr/bin/node dist/server.js`) instead of running npm. That way signals go straight to node, and stopping the service is clean.
- Write comments only on their own line. `User=ubuntu # comment` breaks the service.

### 4. Start it and enable it on boot

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now my-app
```

### 5. Check it's running

```bash
systemctl status my-app
```

It should show **active (running)**. If not, read the logs (see below).

## Everyday commands

```bash
sudo systemctl start my-app        # start
sudo systemctl stop my-app         # stop
sudo systemctl restart my-app      # restart (after deploying new code)
systemctl status my-app            # status and the last few log lines
sudo systemctl enable my-app       # start on boot
sudo systemctl disable my-app      # don't start on boot
```

After editing the `.service` file, always run:

```bash
sudo systemctl daemon-reload
sudo systemctl restart my-app
```

A change to the `.env` file only needs `sudo systemctl restart my-app`.

## Logs

```bash
journalctl -u my-app -f                    # follow live (Ctrl+C to exit)
journalctl -u my-app -n 100                # last 100 lines
journalctl -u my-app --since "1 hour ago"
journalctl -u my-app --since today -p err  # errors only (console.error)
```

Check how much disk the logs use, and cap them:

```bash
journalctl --disk-usage
sudo journalctl --vacuum-time=30d          # delete logs older than 30 days
```

To set a permanent limit, set `SystemMaxUse=1G` in `/etc/systemd/journald.conf`, then run `sudo systemctl restart systemd-journald`.

## Deploying new code

```bash
cd /home/ubuntu/my-app
git pull
npm ci --omit=dev
npm run build        # only if your app has a build step
sudo systemctl restart my-app
```

## Resource limits

Uncomment these lines in the service file to cap the app. With a memory limit, systemd restarts the app if it goes over:

```ini
MemoryMax=1G
CPUQuota=80%
```

Then reload and restart (see "Everyday commands").

## Remove a service

```bash
sudo systemctl disable --now my-app
sudo rm /etc/systemd/system/my-app.service
sudo systemctl daemon-reload
```

## Troubleshooting

| Problem                                           | Fix                                                                                                                       |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `status=203/EXEC`                                 | `ExecStart` path is wrong or not absolute. Check with `which node` (as the same user).                                    |
| `status=200/CHDIR`                                | `WorkingDirectory` doesn't exist or the user can't access it.                                                             |
| `status=217/USER`                                 | The `User=` doesn't exist, or there's a comment after the value on the same line.                                         |
| `Failed to load environment files`                | The `EnvironmentFile=` path is wrong. Fix it or delete the line.                                                          |
| Works with `node index.js` but not as a service   | Usually a relative path in the code (use `path.join(__dirname, ...)`) or a variable that was only set in your shell. Add it to `.env`. |
| `EADDRINUSE`                                      | Another copy of the app is still running (for example a manual `node index.js` or PM2). Stop it.                         |
| `EACCES` on port 80 or 443                        | Don't run Node on low ports. Run it on, say, 3000 and put Nginx in front (see [NginxConfigs](../NginxConfigs)).           |
| `Start request repeated too quickly`              | The app crashes right away. Check `journalctl -u my-app -n 50`, fix the error, then `sudo systemctl reset-failed my-app`. |
| `EROFS: read-only file system`                    | `ProtectSystem=full` blocks writes to `/usr` and `/etc`. Write files inside the app folder or `/var/lib/my-app` instead.  |
