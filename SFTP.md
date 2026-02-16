# How to Deploy a Website to DigitalOcean 💧 LAMP + SFTP (Windows) + DNS Setup

## Sign Up!

These are affiliate links. If you sign up using my link, I get a small commission. It's totally optional, but it helps support the channel.

- Create a DigitalOcean hosting account: [DigitalOcean](https://www.digitalocean.com/?refcode=510e633915b2&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)
- Use hover.com to register a domain: [Hover](https://hover.com/SjMp9blQ)

## How To Use the LAMP 1-Click Install on DigitalOcean

https://www.digitalocean.com/community/tutorials/how-to-use-the-lamp-1-click-install-on-digitalocean

## DNS: Point Your Domain

1. Do this with the domain registrar.
2. Create an A Record to the IP address.
3. Create a CNAME for 'www' with a value or target of 'thelemonstack.com'.
4. Add the domain name to DigitalOcean.

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

### Disable root Login with SSH

```zsh
nano /etc/ssh/sshd_config
# Set/confirm:
#   PermitRootLogin no
#   PasswordAuthentication no
systemctl reload ssh
```

### Reconnect as the new user

```zsh
# logout + log back in as sudo user
exit
ssh angela@thelemonstack.com
```

### Possible Warning

If you've used this domain on another server or IP, you might get this warning. You just need to delete the domain from your local `known_hosts` file.

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
Host key for thelemonstack.com has changed and you have requested strict checking.
Host key verification failed.

# remove the domain from your local known_hosts file
ssh-keygen -R thelemonstack.com

# login again
ssh angela@thelemonstack.com
```

### Add sudo user to the www-data group

```zsh
sudo usermod -aG www-data angela

# logout + log back in for the group change to take effect
exit
ssh angela@thelemonstack.com

# confirm it worked
groups

# you should see
angela sudo www-data
```

### Fix the permissions

That 775 gives group write access, so angela (as part of the www-data group) can update files without breaking Apache ownership.

```zsh
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 775 /var/www/html

# If you want to make sure all future files keep that group:
sudo chmod g+s /var/www/html
# That sets the setgid bit, so any new files/folders inherit the www-data group automatically.
```

### Deploy via SFTP using FileZilla

- [Download FileZilla Client](https://filezilla-project.org/)
- File > Site Manager
- Click "New Site" > Type in a name
- Protocol > SFTP (SSH File Transfer Protocol)
- Host: thelemonstack.com (can also be the IP address)
- Port: 22 (but you can leave this blank)
- Logon Type: Key file
- User: angela (sudo user)
- Key file: /User/angelajholden/.ssh/id_abc1234
- Click "Connect"
- Navigate to the local site on the left: `/Users/angelajholden/Projects/the-lemon-stack/`
- Navigate to the remote site on the right: `/var/www/html`
- Drag the files you want to deploy from the left window to the right window.

Don't deploy any file or directory that isn't required for site functionality. You should always exclude the following files:

- .git
- .gitignore
- Any file with .md
- .DS_Store (on a Mac)
- Design files or directories

```zsh
# Just in case you need to reset ownership/permissions after rsync:
sudo chown -R www-data:www-data /var/www/html
sudo find /var/www/html -type d -exec chmod 755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

## Let's Encrypt

### Make sure DNS is ready

```zsh
ping thelemonstack.com
# crtl + c to quit
```

### Check that Certbot is installed

```zsh
# The 1-Click LAMP image should already have it
certbot --version

# If you get a version number, you’re set.
# If not (rare), install it manually:
sudo apt install certbot python3-certbot-apache -y
```

### Request and install the certificate

We have to install an SSL certificate on both the root and www, even if we're just doing a redirect to the root.

Browsers are checking the domain on port 443 first, and if the SSL/TLS handshake fails (no certificate) then it can't complete the redirect. Make sure you install the SSL certificate on both versions of the domain. It's free!

```zsh
# Run Certbot’s Apache plugin:
sudo certbot --apache -d thelemonstack.com -d www.thelemonstack.com
```

#### Certbot will:

1. Ask for your email (for renewal notices)
2. Ask to agree to the terms
3. Automatically edit Apache to use HTTPS
4. Reload Apache

```zsh
# When it’s done, you’ll see something like:
Congratulations! Your certificate and chain have been saved at:
/etc/letsencrypt/live/thelemonstack.com/fullchain.pem
```

### Enable SSL site if needed

```zsh
sudo a2ensite 000-default-le-ssl.conf
sudo systemctl reload apache2
```

### Run this if in a redirect loop

```zsh
sudo a2enmod ssl
sudo systemctl reload apache2
```

### Auto-renewal check

```zsh
# Certbot installs a renewal timer automatically.
# It checks twice per day, ~12 hours
# If it's within 30 days of renewal, it renews automatically
# Verify it
sudo systemctl list-timers | grep certbot

# Manual dry-run
sudo certbot renew --dry-run

# Output should end with
Congratulations, all renewals succeeded.
```

### Verify which domains are active

```zsh
sudo certbot certificates

# You’ll see a list like:
Certificate Name: thelemonstack.com
Domains: thelemonstack.com www.thelemonstack.com
Expiry Date: 2026-01-20
```

## Edit the Apache Virtual Host Files

### Port 80 VHost

```zsh
sudo nano /etc/apache2/sites-available/000-default.conf
```

Add this INSIDE `<VirtualHost *:80> </VirtualHost>`, at the top of the page.

```zsh
ServerName thelemonstack.com
ServerAlias www.thelemonstack.com
Redirect 301 / https://thelemonstack.com/
```

Comment out these four rewrite lines at the bottom of the file:

```zsh
# RewriteEngine on
# RewriteCond %{SERVER_NAME} =www.thelemonstack.com [OR]
# RewriteCond %{SERVER_NAME} =thelemonstack.com
# RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
```

### Port 443 VHost

```zsh
sudo nano /etc/apache2/sites-available/000-default-le-ssl.conf
```

The port 443 vhost file should look like this:

```zsh
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ServerName thelemonstack.com
    ServerAlias www.thelemonstack.com

    RewriteEngine On
    RewriteCond %{HTTP_HOST} ^www\.thelemonstack\.com$ [NC]
    RewriteRule ^ https://thelemonstack.com%{REQUEST_URI} [R=301,L]

    <Directory /var/www/html/>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    <IfModule mod_dir.c>
        DirectoryIndex index.php index.pl index.cgi index.html index.xhtml index.htm
    </IfModule>

    Include /etc/letsencrypt/options-ssl-apache.conf
    SSLCertificateFile /etc/letsencrypt/live/thelemonstack.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/thelemonstack.com/privkey.pem
</VirtualHost>
</IfModule>
```

### Make sure the rewite module is enabled and reload Apache

```zsh
sudo a2enmod rewrite
sudo systemctl reload apache2

# If you want to be extra sure the SSL vhost is active:
sudo a2ensite 000-default-le-ssl.conf
sudo systemctl reload apache2
```

### Test the redirects

```zsh
# Then you can confirm the redirect behavior later in your browser or by running:
curl -I http://thelemonstack.com
curl -I http://www.thelemonstack.com
curl -I https://thelemonstack.com
curl -I https://www.thelemonstack.com

# You should see:
HTTP/1.1 301 Moved Permanently
Location: https://thelemonstack.com/
```

### Test the result

🔒 You should see the lock icon and your site.

```zsh
# Open your site in the browser
http://thelemonstack.com
http://www.thelemonstack.com
https://thelemonstack.com
https://www.thelemonstack.com
```
