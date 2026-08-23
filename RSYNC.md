# How to Deploy a Website to DigitalOcean 💧 LAMP + Rsync (MacOS/Linux) + DNS Setup

## Sign Up!

These are affiliate links. If you sign up using my link, I get a small commission. It's totally optional, but it helps support the channel.

- Create a DigitalOcean hosting account: [DigitalOcean](https://www.digitalocean.com/?refcode=510e633915b2&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)
- Use hover.com to register a domain: [Hover](https://hover.com/SjMp9blQ)

## How To Use the LAMP 1-Click Install on DigitalOcean

- [LAMP 1-Click App](https://docs.digitalocean.com/products/marketplace/catalog/lamp/)

## DNS: Point Your Domain

Create these records wherever your domain's authoritative DNS is managed. If your domain uses Hover's name servers, create the records in Hover. You do not need to add the domain to DigitalOcean Networking unless you have delegated DNS to DigitalOcean's name servers.

1. Create an **A** record for `fiberandkraft.com` that points to the Droplet's public IPv4 address.
2. Create a **CNAME** record for `www` with `fiberandkraft.com` as its target.

## Droplet Setup

### SSH in as root

```zsh
# Login as root
ssh root@<IP>

# answer 'yes' to confirm known_hosts
yes
```

### Update + upgrade first

```zsh
apt update && apt upgrade -y

# (Optional) reboot if kernel updates were installed:
reboot

# reconnect:
ssh root@<IP>
```

### Check what’s installed

```zsh
apache2 -v
mysql --version
php -v
systemctl status apache2
```

### Nano text editor

[Nano Shortcuts](https://www.nano-editor.org/dist/latest/cheatsheet.html)

```zsh
# To exit the nano editor and save:
Ctrl + X
Y enter
enter (again)
```

### MySQL root password

If you’re deploying a WordPress or database-driven site, your MySQL root password is stored in `/root/.digitalocean_password` on your Droplet. I skipped this part since this demo was a static site deployment.

```zsh
nano /root/.digitalocean_password
```

### Check the Firewall

This is the expected output:

| To               | Action | From          |
| ---------------- | ------ | ------------- |
| 22/tcp           | LIMIT  | Anywhere      |
| Apache Full      | ALLOW  | Anywhere      |
| 22/tcp (v6)      | LIMIT  | Anywhere (v6) |
| Apache Full (v6) | ALLOW  | Anywhere (v6) |

```zsh
ufw status

# if inactive
ufw allow OpenSSH
ufw allow 80
ufw allow 443
ufw enable
```

### Create your non-root sudo user

You can leave the questions blank when creating a sudo user:

- Full Name []:
- Room Number []:
- Work Phone []:
- Home Phone []:
- Other []:
- Is this information correct? [Y/n]

```zsh
adduser angela
usermod -aG sudo angela
```

### Copy root’s authorized key to the new user so you can SSH without passwords

```zsh
# make the .ssh
mkdir -p /home/angela/.ssh

# copy the keys from root
cp -a /root/.ssh/authorized_keys /home/angela/.ssh/

# change the ownership + read/write/execute permissions
chown -R angela:angela /home/angela/.ssh
chmod 700 /home/angela/.ssh
chmod 600 /home/angela/.ssh/authorized_keys
```

### Test the new user before disabling root login

Keep the root session open. In a second terminal on your local computer, confirm that the new account can use both SSH and sudo:

```zsh
ssh angela@<IP>
sudo apache2ctl configtest
```

Continue only after the SSH login succeeds and Apache reports `Syntax OK`.

### Disable root login with SSH

```zsh
nano /etc/ssh/sshd_config
# Set/confirm:
#   PermitRootLogin no
#   PasswordAuthentication no

# Validate the SSH configuration before reloading it
sshd -t
systemctl reload ssh
```

### Reconnect as the new user

```zsh
# logout + log back in as sudo user
exit
ssh angela@fiberandkraft.com
```

### Possible Warning

If you've intentionally moved this domain to a new server or IP, you might get this warning. First compare the new Droplet's SSH host-key fingerprint with the fingerprint shown in the warning. Remove the old entry only after you have confirmed that the server change is legitimate.

```zsh
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ED25519 key sent by the remote host is
SHA256:abc123abc123abc123abc123abc123abc123abc123abc123.
Please contact your system administrator.
Add correct host key in /Users/angelajholden/.ssh/known_hosts to get rid of this message.
Offending ED25519 key in /Users/angelajholden/.ssh/known_hosts:29
Host key for fiberandkraft.com has changed and you have requested strict checking.
Host key verification failed.

# remove the domain from your local known_hosts file
ssh-keygen -R fiberandkraft.com

# login again
ssh angela@fiberandkraft.com
```

### Add sudo user to the www-data group

```zsh
sudo usermod -aG www-data angela

# logout + log back in for the group change to take effect
exit
ssh angela@fiberandkraft.com

# confirm it worked
groups

# you should see
angela sudo www-data
```

### Fix the permissions

Make the deployment user the owner and Apache's `www-data` group the group. Apache can read the site, while `angela` can create, update, and delete deployment files.

```zsh
sudo chown -R angela:www-data /var/www/html
sudo find /var/www/html -type d -exec chmod 2755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

The leading `2` on directory permissions sets the setgid bit so new files and directories inherit the `www-data` group. If an application needs to write uploads, cache files, or other runtime data, grant write access only to those specific directories.

### Deploy via rsync from your local machine

#### Just remember that the trailing slash in your local directory path matters:

- `/Users/angelajholden/Projects/coming-soon/` (with `/`) syncs contents of that folder into /var/www/html/.
- Without `/`, it would nest the folder inside (`/var/www/html/coming-soon/`).

```zsh
# Preview the deployment first. --delete removes remote files that are not present locally.
rsync -avz --no-owner --no-group --progress --delete --dry-run --exclude '.git' --exclude '.gitignore' --exclude '.github' --exclude '.DS_Store' --exclude '*.md' --exclude 'LICENSE*' /Users/angelajholden/Projects/coming-soon/ angela@fiberandkraft.com:/var/www/html/

# Review the dry-run output carefully, then remove --dry-run to deploy.
rsync -avz --no-owner --no-group --progress --delete --exclude '.git' --exclude '.gitignore' --exclude '.github' --exclude '.DS_Store' --exclude '*.md' --exclude 'LICENSE*' /Users/angelajholden/Projects/coming-soon/ angela@fiberandkraft.com:/var/www/html/
```

### Reset the remote permissions

SSH back into the Droplet before running these commands. The `/var/www/html` path is on the server, not your local computer.

```zsh
ssh angela@fiberandkraft.com
sudo chown -R angela:www-data /var/www/html
sudo find /var/www/html -type d -exec chmod 2755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

## Configure Apache

Configure the port 80 virtual host before requesting the SSL certificate. Certbot can then identify the correct domain and add the HTTPS configuration for you.

```zsh
sudo nano /etc/apache2/sites-available/000-default.conf
```

The virtual host should include the domain names, document root, and directory rules:

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName fiberandkraft.com
    ServerAlias www.fiberandkraft.com
    DocumentRoot /var/www/html

    <Directory /var/www/html/>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

`-Indexes` prevents Apache from displaying a directory listing when a directory has no index file. Keep `AllowOverride All` only if the site uses `.htaccess`; otherwise, use `AllowOverride None`.

### Test and reload Apache

Always test Apache's configuration before reloading it:

```zsh
sudo apache2ctl configtest
sudo systemctl reload apache2
sudo apache2ctl -S
```

`configtest` should report `Syntax OK`, and the virtual-host listing should associate both domain names with `000-default.conf`.

## Let's Encrypt

### Make sure DNS and HTTP are ready

```zsh
dig +short fiberandkraft.com
dig +short www.fiberandkraft.com
curl -I http://fiberandkraft.com
curl -I http://www.fiberandkraft.com
```

Both DNS commands should resolve to the Droplet. Both HTTP requests should reach this Apache site before you continue.

### Check that Certbot is installed

The current LAMP 1-Click image includes Certbot. If it is missing, install Certbot and its Apache plugin:

```zsh
certbot --version

# Run this only if Certbot is not installed
sudo apt update
sudo apt install certbot python3-certbot-apache -y
```

### Request and install the certificate

Include both names in the certificate so HTTPS works before the `www` request redirects to the root domain.

```zsh
sudo certbot --apache -d fiberandkraft.com -d www.fiberandkraft.com
```

Certbot will ask for an email address, ask you to agree to the terms, update Apache, and reload it. Choose the option to redirect HTTP traffic to HTTPS when prompted.

Do not manually replace Certbot's generated SSL virtual host. Let Certbot manage its certificate paths and renewal configuration.

### Check automatic renewal

```zsh
sudo systemctl list-timers | grep certbot
sudo certbot renew --dry-run
sudo certbot certificates
```

The dry run should complete successfully, and the certificate listing should include both `fiberandkraft.com` and `www.fiberandkraft.com`.

### Redirect www to the root domain

Certbot can redirect HTTP to HTTPS, but it does not necessarily make the root domain canonical. Use `sudo apache2ctl -S` to identify the active port 443 virtual-host file, then open that file. On this one-site setup, Certbot normally creates:

```zsh
sudo nano /etc/apache2/sites-available/000-default-le-ssl.conf
```

Add these rules inside the `<VirtualHost *:443>` block:

```apache
RewriteEngine On
RewriteCond %{HTTP_HOST} ^www\.fiberandkraft\.com$ [NC]
RewriteRule ^ https://fiberandkraft.com%{REQUEST_URI} [R=301,L]
```

Enable the rewrite module, test the complete configuration, and reload Apache:

```zsh
sudo a2enmod rewrite
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### Test HTTPS and redirects

```zsh
curl -I http://fiberandkraft.com
curl -I http://www.fiberandkraft.com
curl -I https://fiberandkraft.com
curl -I https://www.fiberandkraft.com
```

The HTTP requests should redirect to HTTPS. If you also want `www` to redirect to the root domain, confirm that the final URL is `https://fiberandkraft.com/`.

If you encounter a redirect loop, inspect the enabled virtual hosts and redirect rules instead of repeatedly enabling modules:

```zsh
sudo apache2ctl configtest
sudo apache2ctl -S
sudo grep -R "Redirect\|RewriteRule" /etc/apache2/sites-enabled/
```

### Test the result

🔒 Open `https://fiberandkraft.com` in a browser. You should see the lock icon and your site.
