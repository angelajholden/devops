# How to Deploy Node.js Apps to DigitalOcean 💧 Nginx + PM2 + Subdomain DNS

## Sign Up!

These are affiliate links. If you sign up using my link, I get a small commission. It's totally optional, but it helps support the channel.

- Create a DigitalOcean hosting account: [DigitalOcean](https://www.digitalocean.com/?refcode=510e633915b2&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)
- Use hover.com to register a domain: [Hover](https://hover.com/SjMp9blQ)

## What This Guide Covers

This guide starts with DigitalOcean's Node.js 1-Click image and supports two deployment layouts:

1. **API only:** Nginx proxies the entire subdomain to a Node/Express process.
2. **Full stack:** Nginx serves a compiled frontend and proxies `/api/` requests to Node/Express.

It also supports hosting multiple applications on one Droplet. Give each application its own directory, private Node port, PM2 process, subdomain, and Nginx server block.

Example shared-server layout:

| Application      | Directory                   | Private port | PM2 process            |
| ---------------- | --------------------------- | -----------: | ---------------------- |
| Fiber & Kraft    | `/var/www/fiberandkraft`    |         3000 | `fiberandkraft-api`    |
| YouTube Database | `/var/www/youtube-database` |         3001 | `youtube-database-api` |

Only SSH, HTTP, and HTTPS should be publicly accessible. Node and database ports remain private.

## Before a Livestream

> [!CAUTION]
> **Stop sharing your terminal before the first SSH login.**
>
> The DigitalOcean Node.js 1-Click image may print generated credentials in its login banner. The same banner can appear again whenever a new SSH session opens—not only during the root login.
>
> Do not resume screen sharing until the generated account password has been disabled and any sensitive banner or credentials file has been removed.

Use the same precaution before:

- Creating or editing `.env`
- Entering an account or database password
- Displaying environment variables or application configuration
- Opening PM2 or application logs that may contain secrets
- Pasting tokens, private keys, connection strings, or recovery codes

> [!WARNING]
> If a password, token, connection string, or private key appears on a livestream, treat it as compromised. Disable or rotate it immediately. Editing or blurring the recording afterward is useful, but it does not replace rotating the credential.

## How To Use the Node.js 1-Click Install on DigitalOcean

- [Node.js 1-Click App](https://docs.digitalocean.com/products/marketplace/catalog/nodejs/)

The Node.js 1-Click image includes Ubuntu, Node.js, Nginx, and PM2. It may also include a sample app at `/var/www/html/hello.js` that runs on port 3000 as the `nodejs` user.

This guide removes the sample application and deploys each production app to `/var/www/<application-name>`.

## Create the Droplet

1. In the DigitalOcean control panel, click **Create > Droplets**.
2. Choose a data center near your users.
3. Under **Choose an Image**, open the Marketplace tab and select the Node.js 1-Click App.
4. Choose a Droplet size appropriate for the workload. Allow additional memory when running a database on the same server.
5. Select an existing SSH key or add a new one. Do not use password authentication for SSH.
6. Enable monitoring and backups as needed.
7. Name the Droplet and click **Create Droplet**.
8. Copy its public IPv4 address.

## DNS: Point the Subdomain

Create the record wherever the domain's authoritative DNS is managed. In this example, the domain uses Hover's name servers, so the record belongs in Hover—not DigitalOcean.

1. Open the domain's DNS settings in Hover.
2. Create an **A** record.
3. Set **Hostname** to the desired subdomain, such as `api` or `fiberandkraft`.
4. Set **IP Address** to the Droplet's public IPv4 address.
5. Save the record.

You do not need to add the domain to DigitalOcean Networking unless you have delegated DNS to DigitalOcean's name servers.

## Droplet Setup

### SSH in as root

> [!CAUTION]
> **Livestream checkpoint: hide the terminal now.** The first login may display the generated `nodejs` account password. Keep the terminal private until that credential has been disabled and removed from the login message.

On your local computer:

```zsh
ssh root@<IP>
```

Answer `yes` if prompted to add the host to `known_hosts`.

### Secure the generated `nodejs` account

If the login banner displayed a generated password, assume that anyone watching could use it. Keep the current root session open and lock the account password immediately:

```zsh
passwd --lock nodejs
```

Do not show or print the generated password while investigating the banner. If the image stored it in `/root/.digitalocean_passwords`, remove that file after the password is locked:

```zsh
ls -l /root/.digitalocean_passwords
rm /root/.digitalocean_passwords
```

If the file does not exist, inspect the image's login-message configuration without printing secret values. Do not resume screen sharing until a new test login no longer reveals a credential.

### Update and upgrade

```zsh
apt update
apt upgrade -y
```

Reboot if kernel updates were installed:

```zsh
reboot
```

> [!CAUTION]
> Keep the terminal hidden when reconnecting if you have not yet confirmed that the login banner is safe.

```zsh
ssh root@<IP>
```

### Check what's installed

```zsh
node --version
npm --version
nginx -v
pm2 --version
systemctl status nginx
```

### Nano text editor

[Nano Shortcuts](https://www.nano-editor.org/dist/latest/cheatsheet.html)

To save and exit Nano:

```text
Ctrl + X
Y
Enter
```

### Check the firewall

The Node.js 1-Click image should allow SSH, HTTP, and HTTPS. Do not expose application ports such as 3000 or 3001, or database ports such as 27017. Nginx and the application communicate locally.

```zsh
ufw status
```

Expected rules:

| To           | Action | From          |
| ------------ | ------ | ------------- |
| 22/tcp       | LIMIT  | Anywhere      |
| 80/tcp       | ALLOW  | Anywhere      |
| 443/tcp      | ALLOW  | Anywhere      |
| 22/tcp (v6)  | LIMIT  | Anywhere (v6) |
| 80/tcp (v6)  | ALLOW  | Anywhere (v6) |
| 443/tcp (v6) | ALLOW  | Anywhere (v6) |

If UFW is inactive, configure it before enabling it:

```zsh
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw enable
ufw status
```

## Create Separate Admin and Deployment Users

Use two accounts with distinct responsibilities:

| User     | Responsibility                                                    |
| -------- | ----------------------------------------------------------------- |
| `angela` | `sudo`, packages, systemd, Nginx, MongoDB, and system permissions |
| `deploy` | Application files, SFTP, builds, Node, and PM2                    |

The `deploy` account should not have `sudo` access.

### Create the admin user

> [!CAUTION]
> **Livestream checkpoint:** hide the terminal while creating or entering the admin password. Password prompts normally do not echo characters, but the password is still sensitive.

As `root`:

```zsh
adduser angela
usermod -aG sudo angela
mkdir -p /home/angela/.ssh
cp /root/.ssh/authorized_keys /home/angela/.ssh/authorized_keys
chown -R angela:angela /home/angela/.ssh
chmod 700 /home/angela/.ssh
chmod 600 /home/angela/.ssh/authorized_keys
```

### Create the key-only deployment user

Create this account without a usable password:

```zsh
adduser --disabled-password --gecos "" deploy
mkdir -p /home/deploy/.ssh
touch /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```

For routine deployments, create a dedicated deploy key instead of sharing an administrative key. On your local computer:

```zsh
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_digitalocean_deploy -C "digitalocean-deploy"
```

Copy only the contents of the new `.pub` file into `/home/deploy/.ssh/authorized_keys`. A public key is safe to copy; never upload or paste the private key.

### Test both users before disabling root login

Keep the original root session open. Open separate terminal tabs on your local computer and test both accounts:

```zsh
ssh angela@<IP>
sudo nginx -t
```

```zsh
ssh -i ~/.ssh/id_ed25519_digitalocean_deploy deploy@<IP>
```

Continue only after:

- `angela` can sign in and use `sudo`.
- `deploy` can sign in with its dedicated key without a password prompt.
- The login banner no longer displays credentials.

Do not add `deploy` to the `sudo` group.

### Disable root and password login with SSH

Return to the root session:

```zsh
nano /etc/ssh/sshd_config
```

Set or confirm:

```text
PermitRootLogin no
PasswordAuthentication no
```

Validate the configuration before reloading SSH:

```zsh
sshd -t
systemctl reload ssh
```

Keep the existing root session open while testing one more new admin login. Close it only after the new login succeeds.

Use the admin account for system work from this point forward:

```zsh
ssh angela@<subdomain.example.com>
```

## Remove the DigitalOcean Sample App

Run this section as `angela`.

PM2 keeps a separate process list for each Linux user. Deleting `/var/www/html/hello.js` does not stop the sample process because it may already be running under the `nodejs` user's PM2 instance.

Inspect and remove the sample process:

```zsh
sudo -u nodejs pm2 list
sudo -u nodejs pm2 delete hello
sudo -u nodejs pm2 save
```

If PM2 reports that no process exists, continue. Confirm that the first application port is available:

```zsh
sudo ss -ltnp | grep ':3000'
```

No output means the port is available. If a process appears, identify its owner before stopping it.

Inspect the default web root and remove only identified sample files:

```zsh
ls -la /var/www/html
sudo rm /var/www/html/hello.js
```

Leave `/var/www/html` itself in place. The default Nginx configuration may still refer to it while the new application is being prepared.

## Prepare an Application Directory

Run this section as `angela`. Replace `fiberandkraft` with the application's directory name:

```zsh
sudo mkdir -p /var/www/fiberandkraft
sudo chown -R deploy:deploy /var/www/fiberandkraft
sudo chmod 755 /var/www/fiberandkraft
```

Application files and the PM2 process should both belong to `deploy`.

## Deploy via SFTP Using FileZilla

- [Download FileZilla Client](https://filezilla-project.org/)
- Open **File > Site Manager**.
- Click **New Site** and enter a name.
- Protocol: SFTP (SSH File Transfer Protocol)
- Host: the application's subdomain or the Droplet's IP address
- Port: 22
- Logon Type: Key file
- User: `deploy`
- Key file: the dedicated private deploy key
- Remote directory: `/var/www/<application-name>`

Upload the source files the production application needs, such as:

- `package.json` and `package-lock.json`
- `.nvmrc`
- Backend directories and entry files
- Frontend source and build configuration when building on the server
- Static assets

Do not deploy:

- `.env`
- `.git`
- `.github`
- `node_modules`
- `.angular`
- Existing `dist` output when the server will build the app
- `.DS_Store`
- Editor settings, local logs, test artifacts, or private documentation

After uploading, verify ownership as `angela`:

```zsh
sudo chown -R deploy:deploy /var/www/<application-name>
sudo find /var/www/<application-name> -type d -exec chmod 755 {} \;
sudo find /var/www/<application-name> -type f ! -name '.env' -exec chmod 644 {} \;
```

The `.env` exclusion prevents a later upload from accidentally changing an existing secrets file to world-readable permissions.

## Verify the Node Version

Run application commands as `deploy`:

```zsh
ssh -i ~/.ssh/id_ed25519_digitalocean_deploy deploy@<subdomain.example.com>
cd /var/www/<application-name>
node --version
```

If the project includes `.nvmrc`, compare it with the active version:

```zsh
cat .nvmrc
node --version
```

Install or activate the required Node version before installing dependencies. The interactive shell and PM2 startup environment must use the same Node installation.

## Create Production Secrets

Create production secrets directly on the server. Do not commit `.env`, upload it with the general application files, paste it into a livestream chat, or print it with `cat`.

> [!CAUTION]
> **Livestream checkpoint: stop sharing now.** The next step opens the production `.env` file. Keep the terminal private until the editor has closed and the screen no longer contains secret values.

As `deploy`:

```zsh
cd /var/www/<application-name>
touch .env
chmod 600 .env
nano .env
```

Example variable names—not production values—might include:

```text
NODE_ENV=production
HOST=127.0.0.1
PORT=3000
MONGO_URI=mongodb://127.0.0.1:27017/<database-name>
JWT_SECRET=<generate-a-unique-secret>
```

Verify permissions without displaying the file's contents:

```zsh
ls -l .env
```

The file must be owned and readable by the user running PM2. A missing or unreadable `.env` often appears in Node as an undefined environment variable.

## Install Dependencies and Build

Run this section as `deploy` from the application directory.

### API-only application

If the application runs directly from source and does not need a production build:

```zsh
npm ci --omit=dev
```

Use `npm install --omit=dev` only when the project does not have a `package-lock.json` file.

### Application built on the server

Angular and many other frontend builds require packages listed in `devDependencies`:

```zsh
npm ci
npm run build
```

Do not run `ng serve` as the production web server. Nginx should serve the compiled output.

Locate the generated entry file rather than guessing the build path:

```zsh
find dist -name index.html -print
```

Use the directory containing that `index.html` as the Nginx `root`.

If Angular stops on a component stylesheet budget, review the unexpectedly large stylesheet first. When the size is intentional, update the relevant `maximumWarning` and `maximumError` values in `angular.json`, then rebuild.

## Optional: Local MongoDB

Skip this section when the application does not use MongoDB or uses a managed database.

Install MongoDB using its current instructions for the Droplet's Ubuntu release. Manage it as `angela`, not `deploy`:

```zsh
sudo systemctl enable --now mongod
sudo systemctl status mongod
```

Keep MongoDB bound to localhost and do not add a public UFW rule for port 27017.

If restoring a backup, run `mongorestore` as a user who can access the backup directory. A backup stored beneath `/home/deploy` may be inaccessible to `angela` even when its individual files appear readable.

Example current namespace syntax:

```zsh
mongorestore --nsInclude='<database-name>.*' /path/to/backup
```

Verify the database and collections before starting Node:

```zsh
mongosh
```

> [!CAUTION]
> Do not show database passwords, connection strings, user records, or private application data on a livestream. A database being reachable only from localhost does not make an exposed password safe to keep.

## Test the Application Before PM2

Run the backend manually as `deploy` before introducing PM2, Nginx, DNS, or HTTPS:

```zsh
cd /var/www/<application-name>
node backend/server.js
```

Use the correct entry file for the application. In another `deploy` SSH session, test a real endpoint:

```zsh
curl http://127.0.0.1:3000/api/products
```

Stop the manual process with `Ctrl + C` after the test succeeds.

This sequence isolates application, environment, and database problems before adding the process manager and reverse proxy.

## Run the App With PM2

PM2 process lists are user-specific. Always run application PM2 commands as `deploy`.

### First deployment

```zsh
cd /var/www/<application-name>
pm2 start backend/server.js --name <application-name>-api
pm2 list
pm2 save
```

A first deployment uses `pm2 start`. `pm2 restart` cannot find a process that has not yet been created under the current user.

### Later backend or environment update

```zsh
cd /var/www/<application-name>
pm2 restart <application-name>-api --update-env
pm2 save
```

### Test PM2 locally

```zsh
curl http://127.0.0.1:<application-port>/api/products
```

If the app repeatedly restarts, check its status and confirm the port is not already occupied:

```zsh
pm2 list
ss -ltnp | grep ':<application-port>'
```

If the port belongs to another user and its process details are hidden, ask `angela` to run the same `ss` command with `sudo`.

> [!CAUTION]
> **Livestream checkpoint:** application logs can accidentally include environment values, database connection strings, request data, or tokens. Review the application's logging behavior before displaying `pm2 logs` on stream.

When it is safe to inspect them:

```zsh
pm2 logs <application-name>-api --lines 100
```

### Start PM2 after a reboot

As `deploy`:

```zsh
pm2 startup
```

PM2 prints a `sudo env ...` command. Copy that exact command and run it as `angela`, then return to the `deploy` session and save the process list:

```zsh
pm2 save
```

## Prepare Frontend API URLs

A browser cannot reach the server's Express process through `http://localhost:3000`. In a visitor's browser, `localhost` means the visitor's own computer.

Use relative production URLs:

```ts
private apiUrl = '/api/products';
```

For Angular development, route relative API requests to the local Express server with `proxy.conf.json`:

```json
{
	"/api": {
		"target": "http://127.0.0.1:3000",
		"secure": false,
		"changeOrigin": true
	}
}
```

Example `package.json` script:

```json
"start": "ng serve --proxy-config proxy.conf.json"
```

The proxy file is local development configuration, not a secret. Production Nginx does not use it.

When the frontend and API share the same production origin through Nginx, those browser requests do not require cross-origin access. CORS may still be useful for local development or intentionally separate origins. Allow only the origins the application actually uses.

## Configure Nginx

Run Nginx commands as `angela`. Give each application its own server block instead of replacing the default configuration.

### API-only server block

Create `/etc/nginx/sites-available/<application-name>`:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Full-stack server block

Use the directory that actually contains the compiled `index.html` as `root`:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name fiberandkraft.example.com;

    root /var/www/fiberandkraft/dist/fiberandkraft;
    index index.html;

    location /api/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

The `try_files` fallback allows client-side routes to load directly. The `proxy_pass` above intentionally has no trailing path slash so `/api/...` reaches Express unchanged.

Enable the site, test the complete Nginx configuration, and reload it:

```zsh
sudo ln -s /etc/nginx/sites-available/<application-name> /etc/nginx/sites-enabled/<application-name>
sudo nginx -t
sudo systemctl reload nginx
```

Do not remove the default site until the new server block has passed `nginx -t` and the intended domain is working.

### Verify DNS and HTTP

```zsh
dig +short <subdomain.example.com>
curl -I http://<subdomain.example.com>
curl http://<subdomain.example.com>/api/products
```

DNS should return the Droplet's public IP. The HTTP requests should reach the frontend and API as appropriate.

## Let's Encrypt

Certbot's Nginx plugin can request the certificate, update Nginx, and configure the HTTP-to-HTTPS redirect. DNS must resolve correctly and the site must already be reachable on port 80.

Run as `angela`:

```zsh
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
certbot --version
sudo nginx -t
sudo certbot --nginx -d <subdomain.example.com>
```

Enter an email address, agree to the terms, and choose the redirect option when prompted.

Verify HTTPS and certificate renewal:

```zsh
curl -I http://<subdomain.example.com>
curl -I https://<subdomain.example.com>
sudo systemctl list-timers | grep certbot
sudo certbot renew --dry-run
sudo certbot certificates
```

## Repeatable Update Workflow

Run application updates as `deploy`.

### Frontend-only change

```zsh
cd /var/www/<application-name>
npm run build
```

Nginx serves the new compiled output immediately. Re-enabling the site and restarting PM2 are unnecessary when only frontend files changed.

### Backend-only change

```zsh
cd /var/www/<application-name>
pm2 restart <application-name>-api --update-env
pm2 save
```

### Frontend and backend changes

```zsh
cd /var/www/<application-name>
npm run build
pm2 restart <application-name>-api --update-env
pm2 save
```

### Nginx change

Run as `angela`:

```zsh
sudo nginx -t
sudo systemctl reload nginx
```

## Troubleshooting

| Symptom                                                  | Likely cause or next check                                                                                                    |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| An environment variable such as `MONGO_URI` is undefined | `.env` is missing, unreadable by `deploy`, loaded from the wrong working directory, or not loaded by the app                  |
| PM2 says the process does not exist                      | The first `pm2 start` has not happened, the process name differs, or PM2 is running as the wrong Linux user                   |
| Angular stops on a stylesheet budget                     | A component stylesheet exceeds the budget in `angular.json`; review the CSS and adjust the budget deliberately if appropriate |
| The production browser requests `localhost:3000`         | The compiled frontend still contains a hard-coded development URL                                                             |
| An uploaded frontend change is missing                   | The frontend source was uploaded but `npm run build` was not run afterward                                                    |
| `mongorestore` reports permission denied                 | The current user cannot traverse the directory containing the backup                                                          |
| A direct frontend route returns 404                      | The Nginx server block is missing the SPA `try_files` fallback                                                                |
| Nginx returns 502 for the API                            | Express is stopped, listening on a different address or port, or running under another user's PM2 instance                    |
| Port 3000 is already in use                              | The DigitalOcean sample or another application is still running on that port                                                  |
| Nginx serves its default page                            | The new site is not enabled, `server_name` does not match, or the default server block is winning the request                 |

## Final Checks

Run the system checks as `angela`:

```zsh
sudo nginx -t
sudo systemctl status nginx
sudo ufw status
sudo ss -ltnp
```

Run application checks as `deploy`:

```zsh
cd /var/www/<application-name>
node --version
pm2 list
curl http://127.0.0.1:<application-port>/api/products
```

Run public checks from your local computer:

```zsh
dig +short <subdomain.example.com>
curl -I http://<subdomain.example.com>
curl -I https://<subdomain.example.com>
curl https://<subdomain.example.com>/api/products
```

Confirm that:

- The domain resolves to the correct Droplet.
- HTTP redirects to HTTPS.
- The frontend and API respond correctly.
- PM2 lists the application under `deploy`.
- Node and database ports are not open in UFW.
- Production secrets are not present in Git, SFTP upload lists, terminal output, screenshots, recordings, or logs.
