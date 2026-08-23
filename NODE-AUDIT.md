# Node Server Audit Checklist

Run these checks on the Node.js Droplet.

## 1. Confirm the Deployment User

- [ ] Confirm you are logged in as the intended deployment user.
- [ ] Confirm the user can run `sudo`.

```zsh
whoami
groups
sudo -v
```

Expected deployment user:

```text
angela
```

## 2. Check Application Ownership

- [ ] Application files should be owned by `angela`.
- [ ] Look for files unexpectedly owned by `root` or `nodejs`.

```zsh
sudo find /var/www/html -maxdepth 3 \
  -printf '%M %u:%g %p\n'
```

Repair ownership if necessary:

```zsh
sudo chown -R angela:angela /var/www/html
```

## 3. Check Application Permissions

- [ ] Directories should normally be `755`.
- [ ] Regular application files should normally be `644`.
- [ ] `.env` should be `600`.
- [ ] Nothing containing secrets should be readable by other users.

```zsh
sudo find /var/www/html -path /var/www/html/node_modules -prune -o \
  -type d ! -perm 0755 -ls
sudo find /var/www/html -path /var/www/html/node_modules -prune -o \
  -type f ! -perm 0644 ! -name '.env' -ls
stat /var/www/html/.env
```

Repair:

```zsh
sudo find /var/www/html -path /var/www/html/node_modules -prune -o \
  -type d -exec chmod 755 {} \;
sudo find /var/www/html -path /var/www/html/node_modules -prune -o \
  -type f ! -name '.env' -exec chmod 644 {} \;
chmod 600 /var/www/html/.env
```

Run the `.env` command after the general file-permission command. Do not apply blanket file permissions inside `node_modules`; installed packages can contain executable files and symlinks with different modes.

## 4. Check Deployed Files

- [ ] `.git` is not on the production server.
- [ ] `.github`, documentation, and design files were not uploaded.
- [ ] `node_modules` was installed on the server rather than uploaded from your computer.
- [ ] `package.json` and `package-lock.json` are present.

```zsh
ls -la /var/www/html
sudo find /var/www/html -maxdepth 2 \
  \( -name '.git' -o -name '.github' -o -name '*.md' \) -print
```

Expected production files include:

```text
index.js
package.json
package-lock.json
node_modules
middleware
routes
services
.env
```

## 5. Check Both PM2 Environments

This was the biggest deployment-specific issue.

- [ ] Check PM2 under `angela`.
- [ ] Check PM2 under DigitalOcean's `nodejs` user.
- [ ] Confirm the old `hello` demo process is gone.
- [ ] Confirm the production app is managed only once.

```zsh
pm2 list
sudo -u nodejs pm2 list
```

Under `angela`, expect something like:

```text
express-server    online
```

Under `nodejs`, the process list should be empty if you are no longer using that account.

If `hello` still exists:

```zsh
sudo -u nodejs pm2 delete hello
sudo -u nodejs pm2 save
```

## 6. Check Which Process Owns Port 3000

- [ ] Exactly one intended Node process should listen on port 3000.
- [ ] The DigitalOcean demo process should not own the port.
- [ ] The application should not be repeatedly fighting for the port.

```zsh
sudo ss -ltnp | grep ':3000'
```

If the result is unclear:

```zsh
sudo lsof -iTCP:3000 -sTCP:LISTEN -n -P
```

## 7. Check the Production PM2 Process

- [ ] The status should be `online`.
- [ ] The restart count should not keep increasing.
- [ ] Logs should not show `EADDRINUSE`, missing environment variables, or repeated crashes.

```zsh
pm2 list
pm2 describe express-server
pm2 logs express-server --lines 100
```

Run `pm2 list` twice a few minutes apart if you suspect a restart loop.

## 8. Test Express Directly

- [ ] Test Node without Nginx, DNS, or HTTPS.
- [ ] Confirm the expected API endpoint returns the production data.

```zsh
curl http://127.0.0.1:3000/instagram/images
```

If this fails, troubleshoot Node and PM2 before changing Nginx.

## 9. Check PM2 Reboot Persistence

- [ ] Confirm the PM2 startup service exists for `angela`.
- [ ] Confirm the production process list has been saved.
- [ ] Make sure persistence was not configured only for `nodejs`.

```zsh
systemctl status pm2-angela
pm2 save
```

You can inspect the service with:

```zsh
sudo systemctl cat pm2-angela
```

If `pm2-angela` does not exist:

```zsh
pm2 startup
```

Run the exact `sudo env ...` command PM2 prints, then:

```zsh
pm2 save
```

## 10. Check the Nginx Server Block

- [ ] `server_name` should be `api.practicelayouts.com`.
- [ ] `proxy_pass` should point to `127.0.0.1:3000`.
- [ ] Remove the one-click demo hostname such as `hello_node`.
- [ ] Remove unused demo static-file configuration.
- [ ] Confirm proxy headers are present.

```zsh
sudo nginx -T | grep -n \
  "server_name\|proxy_pass\|root\|hello"
```

Expected values:

```nginx
server_name api.practicelayouts.com;
proxy_pass http://127.0.0.1:3000;
```

## 11. Validate Nginx

- [ ] The current Nginx configuration should pass.
- [ ] Nginx should be active.

```zsh
sudo nginx -t
sudo systemctl status nginx
```

After any correction:

```zsh
sudo nginx -t
sudo systemctl reload nginx
```

Do not reload Nginx if the test fails.

## 12. Confirm Port 3000 Is Not Public

- [ ] UFW should allow 22, 80, and 443.
- [ ] UFW should not expose port 3000.

```zsh
sudo ufw status
```

If an unnecessary port 3000 rule exists, inspect its numbered rule before removing it:

```zsh
sudo ufw status numbered
```

## 13. Check the Certificate

- [ ] The certificate should include `api.practicelayouts.com`.
- [ ] Certbot's renewal test should pass.
- [ ] Nginx should reference the correct certificate.

```zsh
sudo certbot certificates
sudo certbot renew --dry-run
sudo nginx -T | grep -n \
  "ssl_certificate\|api.practicelayouts.com"
```

## 14. Test the Complete Request Path

Test each layer in order:

```zsh
# Express directly
curl http://127.0.0.1:3000/instagram/images

# Nginx over public HTTP
curl -IL http://api.practicelayouts.com/instagram/images

# Nginx over public HTTPS
curl -IL https://api.practicelayouts.com/instagram/images
```

Confirm:

- [ ] Localhost returns the API response.
- [ ] HTTP redirects to HTTPS.
- [ ] HTTPS returns the API response.
- [ ] There is no redirect loop.
- [ ] The response comes from your app, not DigitalOcean's demo.

The highest-priority checks are the two separate PM2 environments, ownership of port 3000, PM2 reboot persistence, `.env` permissions, the Nginx `server_name` and `proxy_pass`, and the full HTTP-to-HTTPS request path.
