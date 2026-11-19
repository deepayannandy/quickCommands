# Nginx Configs

Ready-to-use Nginx server blocks for Ubuntu (22.04 / 24.04), plus free HTTPS with Certbot.

| File                                               | Use it for                                                                                       |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| [nodejs-api.conf](nodejs-api.conf)                 | Reverse proxy to a Node.js app running on a local port, with WebSocket support                   |
| [websocket-upgrade.conf](websocket-upgrade.conf)   | Small helper that `nodejs-api.conf` needs. Install it once per server                            |
| [react-site.conf](react-site.conf)                 | Serving a built React / Vite / single-page app, with client-side routing and asset caching       |

## Before you start

- A server with a public IP and Ubuntu.
- A domain (or subdomain) with a DNS **A record** pointing to the server's IP. Check it with `ping <your_domain>`. Certbot fails until DNS points to the server.
- Ports **80** and **443** open in your cloud firewall / security group (AWS, etc.).

## Step 1: Install Nginx

```bash
sudo apt update
sudo apt install -y nginx
```

If UFW is enabled, allow HTTP and HTTPS:

```bash
sudo ufw allow 'Nginx Full'
```

Open `http://<server_ip>` in a browser. You should see the "Welcome to nginx" page.

## Step 2: Add a site

Each site gets its own file in `/etc/nginx/sites-available/`, named after its domain, and a link in `/etc/nginx/sites-enabled/` to turn it on.

### Option A: Node.js API

1. Make sure your app is running and reachable on the server, for example `curl http://127.0.0.1:3000`. To keep it running after reboots, see [SystemD setup for NodeJs App](../SystemD%20setup%20for%20NodeJs%20App).

2. Install the WebSocket helper (only once per server):

   ```bash
   sudo curl -fsSL -o /etc/nginx/conf.d/websocket-upgrade.conf \
     https://raw.githubusercontent.com/deepayannandy/quickCommands/main/NginxConfigs/websocket-upgrade.conf
   ```

3. Download the site config:

   ```bash
   sudo curl -fsSL -o /etc/nginx/sites-available/api.example.com \
     https://raw.githubusercontent.com/deepayannandy/quickCommands/main/NginxConfigs/nodejs-api.conf
   ```

4. Edit it and replace `<your_domain>` and `<port>`:

   ```bash
   sudo nano /etc/nginx/sites-available/api.example.com
   ```

5. If you use Express, add `app.set('trust proxy', 1)` so `req.ip` and `req.protocol` show the real client IP and `https`.

### Option B: React / static site

1. Build the app on your machine or the server:

   ```bash
   npm run build      # output goes to dist/ (Vite) or build/ (Create React App)
   ```

2. Copy the build output to the web folder:

   ```bash
   sudo mkdir -p /var/www/my-site
   sudo cp -r dist/* /var/www/my-site/
   ```

   To redeploy later, clear the folder first so old files don't pile up: `sudo rm -rf /var/www/my-site/*`.

3. Download the site config:

   ```bash
   sudo curl -fsSL -o /etc/nginx/sites-available/example.com \
     https://raw.githubusercontent.com/deepayannandy/quickCommands/main/NginxConfigs/react-site.conf
   ```

4. Edit it and replace `<folder_name>` (for example `my-site`) and the domains (for example `example.com www.example.com`):

   ```bash
   sudo nano /etc/nginx/sites-available/example.com
   ```

## Step 3: Enable the site and reload

```bash
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
sudo nginx -t                    # always test before reloading
sudo systemctl reload nginx
```

`nginx -t` must say **syntax is ok** and **test is successful**. If it doesn't, fix the line it names and test again. A failed test never affects the sites that are already running.

Optional: remove the default welcome site so unknown domains don't show it:

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

Open `http://<your_domain>` and check that your site or API responds.

## Step 4: Add HTTPS with Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
```

- Pass every domain in `server_name` with its own `-d`.
- When asked, enter your email and accept the terms. Certbot gets a free Let's Encrypt certificate, adds the `listen 443 ssl` and certificate lines to your config, and sets up the redirect from HTTP to HTTPS.
- Certificates last 90 days and renew automatically. Check that renewal works:

  ```bash
  sudo certbot renew --dry-run
  ```

- See all certificates and their expiry dates: `sudo certbot certificates`.

## Everyday commands

```bash
sudo nginx -t                        # test the config
sudo systemctl reload nginx          # apply config changes without downtime
sudo systemctl restart nginx         # full restart
sudo systemctl status nginx          # is it running?
sudo tail -f /var/log/nginx/error.log    # follow errors
sudo tail -f /var/log/nginx/access.log   # follow requests
```

Turn a site off without deleting its config:

```bash
sudo rm /etc/nginx/sites-enabled/example.com
sudo nginx -t && sudo systemctl reload nginx
```

## Troubleshooting

| Problem                                          | Fix                                                                                                                                         |
| ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `502 Bad Gateway`                                | The Node app isn't running or is on a different port. Check `curl http://127.0.0.1:<port>` on the server and `sudo tail /var/log/nginx/error.log`. |
| `unknown "connection_upgrade" variable`          | `websocket-upgrade.conf` isn't in `/etc/nginx/conf.d/`. Do Step 2A.2.                                                                        |
| `duplicate "map" ... connection_upgrade`         | The `map` block is defined twice. Keep it only in `/etc/nginx/conf.d/websocket-upgrade.conf`.                                                 |
| `413 Request Entity Too Large`                   | Upload is bigger than `client_max_body_size`. Raise it in the site config and reload.                                                       |
| React page shows 404 on refresh                  | The `try_files ... /index.html` line is missing, or `root` points to the wrong folder.                                                      |
| `403 Forbidden` on the React site                | `index.html` isn't in the `root` folder (you may have copied the `dist` folder itself instead of its contents), or files aren't readable. |
| Old version still shows after redeploy           | Hard refresh the browser (Ctrl+Shift+R). `index.html` isn't cached, so new builds show up after that.                                       |
| Certbot: `Timeout during connect` / `unauthorized` | DNS doesn't point to this server yet, or port 80 is blocked by the firewall / security group.                                             |
| Site shows the "Welcome to nginx" page           | The site isn't linked in `sites-enabled`, or `server_name` doesn't match the domain. Remove the `default` site.                            |
