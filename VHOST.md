# How to Add a Subdomain on DigitalOcean 💧 Apache Virtual Host + DNS

## Sign Up!

These are affiliate links. If you sign up using my link, I get a small commission. It's totally optional, but it helps support the channel.

- Create a DigitalOcean hosting account: [DigitalOcean](https://www.digitalocean.com/?refcode=510e633915b2&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)
- Use hover.com to register a domain: [Hover](https://hover.com/SjMp9blQ)

## How To Use the LAMP 1-Click Install on DigitalOcean

- [How To Use the LAMP 1-Click Install on DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-use-the-lamp-1-click-install-on-digitalocean)

## Droplet Setup

- [How to Deploy a Website to DigitalOcean 💧 LAMP + SFTP (Windows) + DNS Setup](SFTP.md)
- [How to Deploy a Website to DigitalOcean 💧 LAMP + Rsync (MacOS/Linux) + DNS Setup](RSYNC.md)

## DNS: Point Your Domain

1. Do this with the domain registrar.
2. Create an A Record to the IP address.
3. Add the domain name to DigitalOcean.

### SSH in as sudo user

```zsh
# Login as sudo user
ssh heidi@practicelayouts.com
```

### Update + upgrade as needed

```zsh
sudo apt update
sudo apt upgrade -y

# (Optional) reboot if kernel updates were installed:
sudo reboot

# reconnect:
ssh heidi@practicelayouts.com
```

### Nano text editor

[Nano Shortcuts](https://www.nano-editor.org/dist/latest/cheatsheet.html)

```zsh
# To exit the nano editor and save:
Ctrl + X
Y enter
enter (again)
```

### Create the sub domain + public_html directories

```zsh
sudo mkdir -p /var/www/colorado.practicelayouts.com
sudo cd /var/www/colorado.practicelayouts.com
sudo mkdir -p public_html
```

### Fix the permissions

That 775 gives group write access, so 'heidi' (as part of the www-data group) can update files without breaking Apache ownership.

```zsh
sudo chown -R www-data:www-data /var/www/colorado.practicelayouts.com
sudo chmod -R 775 /var/www/colorado.practicelayouts.com

# If you want to make sure all future files keep that group:
sudo chmod g+s /var/www/colorado.practicelayouts.com
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
- Navigate to the local site on the left: `/Users/angelajholden/Projects/colorado/`
- Navigate to the remote site on the right: `/var/www/colorado.practicelayouts.com/public_html`
- Drag the files you want to deploy from the left window to the right window.

Don't deploy any file or directory that isn't required for site functionality. You should always exclude the following files:

- .git
- .gitignore
- Any file with .md
- .DS_Store (on a Mac)
- Design files or directories

```zsh
# Just in case you need to reset ownership/permissions after rsync:
sudo chown -R www-data:www-data /var/www/colorado.practicelayouts.com
sudo find /var/www/colorado.practicelayouts.com -type d -exec chmod 755 {} \;
sudo find /var/www/colorado.practicelayouts.com -type f -exec chmod 644 {} \;
```

## Create + Edit the Port 80 Virtual Host File

Create (copy) the VHost file

```zsh
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/colorado.practicelayouts.com.conf
```

Open the VHost file to edit

```zsh
sudo nano /etc/apache2/sites-available/colorado.practicelayouts.com.conf
```

Add this INSIDE `<VirtualHost *:80> </VirtualHost>`, at the top of the page.

```zsh
ServerName colorado.practicelayouts.com
Redirect 301 / https://colorado.practicelayouts.com/
```

Comment out these four rewrite lines at the bottom of the file:

```zsh
# RewriteEngine on
# RewriteCond %{SERVER_NAME} =www.thelemonstack.com [OR]
# RewriteCond %{SERVER_NAME} =thelemonstack.com
# RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
```

## Let's Encrypt

### Make sure DNS is ready

```zsh
ping colorado.practicelayouts.com
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
sudo certbot --apache -d colorado.practicelayouts.com
```

#### Certbot will:

1. Ask for your email (for renewal notices)
2. Ask to agree to the terms
3. Automatically edit Apache to use HTTPS
4. Reload Apache

```zsh
# When it’s done, you’ll see something like:
Congratulations! Your certificate and chain have been saved at:
/etc/letsencrypt/live/colorado.practicelayouts.com/fullchain.pem
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
Certificate Name: colorado.practicelayouts.com
Domains: colorado.practicelayouts.com
Expiry Date: 2026-05-17
```

## Edit the Port 443 Virtual Host File

Open the port 443 vhost file

```zsh
sudo nano /etc/apache2/sites-available/colorado.practicelayouts.com-le-ssl.conf
```

The port 443 vhost file should look like this:

```zsh
<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/colorado.practicelayouts.com/public_html
    ServerName colorado.practicelayouts.com

    <Directory /var/www/colorado.practicelayouts.com/public_html/>
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
    SSLCertificateFile /etc/letsencrypt/live/colorado.practicelayouts.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/colorado.practicelayouts.com/privkey.pem
</VirtualHost>
</IfModule>
```

### Make sure the rewite module is enabled and reload Apache

```zsh
sudo a2enmod rewrite
sudo systemctl reload apache2

# If you want to be extra sure the SSL vhost is active:
sudo a2ensite colorado.practicelayouts.com-le-ssl.conf
sudo systemctl reload apache2
```

### Test the redirects

```zsh
# Then you can confirm the redirect behavior later in your browser or by running:
curl -I http://colorado.practicelayouts.com
curl -I https://colorado.practicelayouts.com

# You should see:
HTTP/1.1 301 Moved Permanently
Location: https://colorado.practicelayouts.com/
```

### Test the result

🔒 You should see the lock icon and your site.

```zsh
# Open your site in the browser
http://colorado.practicelayouts.com
https://colorado.practicelayouts.com
```
