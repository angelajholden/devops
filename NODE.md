# How to Deploy a Node.js App to DigitalOcean 💧 Nginx + PM2 + Subdomain DNS

## Sign Up!

These are affiliate links. If you sign up using my link, I get a small commission. It's totally optional, but it helps support the channel.

- Create a DigitalOcean hosting account: [DigitalOcean](https://www.digitalocean.com/?refcode=510e633915b2&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)
- Use hover.com to register a domain: [Hover](https://hover.com/SjMp9blQ)

## How To Use the Node.js 1-Click Install on DigitalOcean

- [Node.js 1-Click App](https://docs.digitalocean.com/products/marketplace/catalog/nodejs/)

The Node.js 1-Click image includes Ubuntu, Node.js, Nginx, and PM2. It also includes a sample app at `/var/www/html/hello.js` that runs on port 3000 as the `nodejs` user.

This guide replaces that sample with an Express app served from `/var/www/html` at `api.practicelayouts.com`.

## Create the Droplet

1. In the DigitalOcean control panel, click **Create > Droplets**.
2. Choose a data center near your users.
3. Under **Choose an Image**, open the Marketplace tab and select the Node.js 1-Click App.
4. Choose a Droplet size. The smallest basic plan is enough for a small Express API.
5. Select an existing SSH key or add a new one. Do not use password authentication.
6. Enable monitoring and backups as needed.
7. Name the Droplet and click **Create Droplet**.
8. Copy its public IPv4 address.

## DNS: Point the Subdomain

Create the record wherever your domain's authoritative DNS is managed. In this example, the domain uses Hover's name servers, so the record belongs in Hover—not DigitalOcean.

1. Open the domain's DNS settings in Hover.
2. Create an **A** record.
3. Set **Hostname** to `api`.
4. Set **IP Address** to the Droplet's public IPv4 address.
5. Save the record.

You do not need to add the domain to DigitalOcean Networking unless you have delegated DNS to DigitalOcean's name servers.

## Droplet Setup

### SSH in as root

```zsh
# Login using the Droplet's IP address
ssh root@<IP>

# Answer yes when prompted to add the host to known_hosts
yes
```

### Update + upgrade first

```zsh
apt update
apt upgrade -y

# Reboot if kernel updates were installed
reboot

# Reconnect after the Droplet comes back online
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

```zsh
# To exit nano and save
Ctrl + X
Y
Enter
```

### Check the firewall

The Node.js 1-Click image should allow SSH, HTTP, and HTTPS. Port 3000 should not be exposed publicly because Nginx will proxy requests to it locally.

```zsh
ufw status
```

This is the expected output:

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
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
ufw status
```

### Create your non-root sudo user

You can leave the optional profile questions blank when creating the user.

```zsh
adduser angela
usermod -aG sudo angela
```

### Copy root's authorized key to the new user

```zsh
mkdir -p /home/angela/.ssh
cp /root/.ssh/authorized_keys /home/angela/.ssh/authorized_keys
chown -R angela:angela /home/angela/.ssh
chmod 700 /home/angela/.ssh
chmod 600 /home/angela/.ssh/authorized_keys
```

### Test the new user before disabling root login

Keep the root session open. In a second terminal on your local computer, confirm that the new account can use both SSH and sudo:

```zsh
ssh angela@<IP>
sudo nginx -t
```

Continue only after the SSH login succeeds and Nginx reports that its configuration test is successful.

### Disable root and password login with SSH

Return to the root session:

```zsh
nano /etc/ssh/sshd_config
```

Set or confirm these values:

```text
PermitRootLogin no
PasswordAuthentication no
```

Validate the SSH configuration before reloading it:

```zsh
sshd -t
systemctl reload ssh
```

Use the sudo account for the rest of the guide:

```zsh
exit
ssh angela@api.practicelayouts.com
```

## Remove the DigitalOcean Sample App

PM2 keeps a separate process list for each Linux user. Deleting `/var/www/html/hello.js` does not stop the sample process because it is already running under the `nodejs` user's PM2 instance.

Stop and remove the sample app before starting another app on port 3000:

```zsh
sudo -u nodejs pm2 list
sudo -u nodejs pm2 delete hello
sudo -u nodejs pm2 save
```

Confirm that nothing is listening on port 3000:

```zsh
sudo ss -ltnp | grep ':3000'
```

No output means the port is available. If a process still appears, identify and stop it before continuing.

## Prepare the Application Directory

The one-click image is configured around `/var/www/html`, so this guide keeps that path.

```zsh
# Inspect the sample files before removing them
ls -la /var/www/html

# Remove the one-click sample application
sudo rm /var/www/html/hello.js
sudo chown -R angela:angela /var/www/html
sudo chmod 755 /var/www/html
```

Remove any other clearly identified sample assets individually. Do not delete an entire directory unless you have confirmed that it contains no application data you need.

## Deploy via SFTP Using FileZilla

- [Download FileZilla Client](https://filezilla-project.org/)
- Open **File > Site Manager**.
- Click **New Site** and enter a name.
- Protocol: SFTP (SSH File Transfer Protocol)
- Host: `api.practicelayouts.com` or the Droplet's IP address
- Port: 22
- Logon Type: Key file
- User: `angela`
- Key file: the local private SSH key selected when creating the Droplet
- Remote directory: `/var/www/html`

Upload only the files the production application needs, such as:

- `index.js`
- `package.json`
- `package-lock.json`
- Application directories such as `middleware`, `routes`, and `services`
- `.env`, when the application uses local environment variables

Do not deploy:

- `.git`
- `.gitignore`
- `.github`
- `node_modules`
- `.DS_Store`
- Markdown, design, test, or other development-only files

### Secure the deployed files

```zsh
sudo chown -R angela:angela /var/www/html
find /var/www/html -type d -exec chmod 755 {} \;
find /var/www/html -type f -exec chmod 644 {} \;

# Restrict secrets to the application owner
chmod 600 /var/www/html/.env
```

If the app does not use a `.env` file, skip the last command. Never commit or print production secrets.

## Install Production Dependencies

```zsh
cd /var/www/html
npm ci --omit=dev
```

Use `npm install --omit=dev` instead if the project does not have a `package-lock.json` file.

## Run the App With PM2

The one-click image includes PM2. Run all commands for this app as the same non-root user so that you do not create another hidden PM2 process list.

```zsh
cd /var/www/html
pm2 start index.js --name express-server
pm2 list
```

### Test the app directly

Test Node before involving Nginx, DNS, or HTTPS. Replace the path with an endpoint that exists in your app.

```zsh
curl http://127.0.0.1:3000/instagram/images
```

The response should come from your app. If PM2 repeatedly restarts it, inspect the logs and check for a port conflict:

```zsh
pm2 logs express-server --lines 100
sudo ss -ltnp | grep ':3000'
```

### Start PM2 after a reboot

```zsh
pm2 startup
```

PM2 will print a `sudo env ...` command. Copy and run that exact command, then save the current process list:

```zsh
pm2 save
```

## Configure Nginx

Open the configuration included with the one-click image:

```zsh
sudo nano /etc/nginx/sites-available/default
```

Replace the sample server block with:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name api.practicelayouts.com;

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

Test the configuration before reloading Nginx:

```zsh
sudo nginx -t
sudo systemctl reload nginx
```

### Verify DNS and HTTP

```zsh
dig +short api.practicelayouts.com
curl http://api.practicelayouts.com/instagram/images
```

The DNS command should return the Droplet's public IP, and the HTTP request should return a response from the Express app.

## Let's Encrypt

Certbot's Nginx plugin can request the certificate, update Nginx, and configure the HTTP-to-HTTPS redirect. DNS must resolve correctly and the app must already be reachable on port 80.

### Install Certbot and its Nginx plugin

```zsh
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
certbot --version
```

### Request and install the certificate

```zsh
sudo nginx -t
sudo certbot --nginx -d api.practicelayouts.com
```

Enter an email address, agree to the terms, and choose the redirect option when prompted. Certbot will update and reload Nginx.

### Test HTTPS and the redirect

```zsh
curl -I http://api.practicelayouts.com/instagram/images
curl -I https://api.practicelayouts.com/instagram/images
```

The HTTP request should redirect to HTTPS. The HTTPS request should return a successful response from the application.

### Check automatic renewal

```zsh
sudo systemctl list-timers | grep certbot
sudo certbot renew --dry-run
sudo certbot certificates
```

## Final Checks

```zsh
# Nginx configuration is valid
sudo nginx -t

# Nginx is running
sudo systemctl status nginx

# The app is online under the correct user
pm2 list

# The app responds directly on the Droplet
curl http://127.0.0.1:3000/instagram/images

# The public endpoint redirects to HTTPS and responds securely
curl -I http://api.practicelayouts.com/instagram/images
curl -I https://api.practicelayouts.com/instagram/images
```

🔒 Open `https://api.practicelayouts.com/instagram/images` in a browser to confirm that the certificate and API response work.
