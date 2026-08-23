# How to Add a Subdomain on DigitalOcean 💧 Apache Virtual Host + DNS

## Sign Up!

These are affiliate links. If you sign up using my link, I get a small commission. It's totally optional, but it helps support the channel.

- Create a DigitalOcean hosting account: [DigitalOcean](https://www.digitalocean.com/?refcode=510e633915b2&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)
- Use hover.com to register a domain: [Hover](https://hover.com/SjMp9blQ)

## How To Use the LAMP 1-Click Install on DigitalOcean

- [LAMP 1-Click App](https://docs.digitalocean.com/products/marketplace/catalog/lamp/)

## Droplet Setup

Complete one of the primary deployment guides before adding a subdomain:

- [How to Deploy a Website to DigitalOcean 💧 LAMP + SFTP (Windows) + DNS Setup](SFTP.md)
- [How to Deploy a Website to DigitalOcean 💧 LAMP + Rsync (MacOS/Linux) + DNS Setup](RSYNC.md)

This guide assumes the Droplet already has Apache, a non-root sudo user named `heidi`, SSH-key authentication, and UFW rules for ports 22, 80, and 443.

## DNS: Point the Subdomain

Create the record wherever your domain's authoritative DNS is managed. If `practicelayouts.com` uses Hover's name servers, create the record in Hover. You do not need to add the domain to DigitalOcean Networking unless you have delegated DNS to DigitalOcean's name servers.

1. Open the DNS settings for `practicelayouts.com`.
2. Create an **A** record.
3. Set **Hostname** to `colorado`.
4. Set **IP Address** to the existing Droplet's public IPv4 address.
5. Save the record.

## Prepare the Subdomain Directory

### SSH in as the sudo user

Use the Droplet's IP address or an existing hostname that already points to it:

```zsh
ssh heidi@<IP>
```

### Update + upgrade as needed

```zsh
sudo apt update
sudo apt upgrade -y

# Reboot if kernel updates were installed
sudo reboot

# Reconnect after the Droplet comes back online
ssh heidi@<IP>
```

### Nano text editor

[Nano Shortcuts](https://www.nano-editor.org/dist/latest/cheatsheet.html)

```zsh
# To exit nano and save
Ctrl + X
Y
Enter
```

### Create the document root

```zsh
sudo mkdir -p /var/www/colorado.practicelayouts.com/public_html
```

### Set the ownership and permissions

Make the deployment user the owner and Apache's `www-data` group the group. Apache can read the site, while `heidi` can create, update, and delete deployment files.

```zsh
sudo chown -R heidi:www-data /var/www/colorado.practicelayouts.com
sudo find /var/www/colorado.practicelayouts.com -type d -exec chmod 2755 {} \;
sudo find /var/www/colorado.practicelayouts.com -type f -exec chmod 644 {} \;
```

The leading `2` on directory permissions sets the setgid bit so new files and directories inherit the `www-data` group. If an application needs writable runtime directories, grant write access only to those specific directories.

## Deploy via SFTP Using FileZilla

- [Download FileZilla Client](https://filezilla-project.org/)
- Open **File > Site Manager**.
- Click **New Site** and enter a name.
- Protocol: SFTP (SSH File Transfer Protocol)
- Host: the Droplet's IP address or an existing hostname that points to it
- Port: 22
- Logon Type: Key file
- User: `heidi`
- Key file: the private SSH key used for this Droplet
- Local directory: the `colorado` project on your computer
- Remote directory: `/var/www/colorado.practicelayouts.com/public_html`

Upload only the files the production site needs. Do not deploy:

- `.git`
- `.gitignore`
- `.github`
- `.DS_Store`
- Markdown or license files
- Design, test, or other development-only files

### Reset the remote permissions

SSH into the Droplet before running these commands:

```zsh
ssh heidi@<IP>
sudo chown -R heidi:www-data /var/www/colorado.practicelayouts.com
sudo find /var/www/colorado.practicelayouts.com -type d -exec chmod 2755 {} \;
sudo find /var/www/colorado.practicelayouts.com -type f -exec chmod 644 {} \;
```

## Create the Port 80 Virtual Host

Create a dedicated configuration file instead of copying another site's configuration. This avoids carrying unrelated domain names or redirect rules into the new virtual host.

```zsh
sudo nano /etc/apache2/sites-available/colorado.practicelayouts.com.conf
```

Add this configuration:

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName colorado.practicelayouts.com
    DocumentRoot /var/www/colorado.practicelayouts.com/public_html

    <Directory /var/www/colorado.practicelayouts.com/public_html/>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/colorado.practicelayouts.com-error.log
    CustomLog ${APACHE_LOG_DIR}/colorado.practicelayouts.com-access.log combined
</VirtualHost>
```

`-Indexes` prevents Apache from displaying a directory listing when a directory has no index file. Keep `AllowOverride All` only if the site uses `.htaccess`; otherwise, use `AllowOverride None`.

### Enable and test the virtual host

Enable the port 80 site before running Certbot. Certbot must be able to find an active virtual host with the correct `ServerName`.

```zsh
sudo a2ensite colorado.practicelayouts.com.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
sudo apache2ctl -S
```

Continue only if `configtest` reports `Syntax OK` and the virtual-host listing associates `colorado.practicelayouts.com` with its new configuration file.

### Verify DNS and HTTP

```zsh
dig +short colorado.practicelayouts.com
curl -I http://colorado.practicelayouts.com
```

The DNS command should return the Droplet's public IP address. The HTTP request should reach the new virtual host before you request the certificate.

If Apache displays another site, check the active virtual-host mapping and confirm that `ServerName` is correct:

```zsh
sudo apache2ctl -S
```

## Let's Encrypt

### Check that Certbot is installed

The current LAMP 1-Click image includes Certbot. If it is missing, install Certbot and its Apache plugin:

```zsh
certbot --version

# Run this only if Certbot is not installed
sudo apt update
sudo apt install certbot python3-certbot-apache -y
```

### Request and install the certificate

This subdomain needs only one certificate name. It does not require a `www` certificate unless you also create and serve `www.colorado.practicelayouts.com` (which we would never do).

```zsh
sudo certbot --apache -d colorado.practicelayouts.com
```

Certbot will ask for an email address, ask you to agree to the terms, create and enable the port 443 virtual host, and reload Apache. Choose the option to redirect HTTP traffic to HTTPS when prompted.

Do not manually replace Certbot's generated SSL virtual host or hard-code its certificate paths. Let Certbot manage the certificate configuration and renewal.

### Verify the enabled virtual hosts

```zsh
sudo apache2ctl configtest
sudo apache2ctl -S
```

The output should show the subdomain on ports 80 and 443. Certbot normally creates and enables `colorado.practicelayouts.com-le-ssl.conf` automatically.

### Check automatic renewal

```zsh
sudo systemctl list-timers | grep certbot
sudo certbot renew --dry-run
sudo certbot certificates
```

The dry run should complete successfully, and the certificate listing should include `colorado.practicelayouts.com`.

## Test HTTPS and the Redirect

```zsh
curl -I http://colorado.practicelayouts.com
curl -I https://colorado.practicelayouts.com
```

The HTTP request should redirect to HTTPS. The HTTPS request should return a successful response from the new site.

If you encounter a redirect loop, inspect the enabled virtual hosts and redirect rules:

```zsh
sudo apache2ctl configtest
sudo apache2ctl -S
sudo grep -R "Redirect\|RewriteRule" /etc/apache2/sites-enabled/
```

## Test the Result

🔒 Open `https://colorado.practicelayouts.com` in a browser. You should see the lock icon and the subdomain site.
