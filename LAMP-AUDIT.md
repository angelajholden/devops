# LAMP Server Audit Checklist

Run this separately on each LAMP server.

## 1. Confirm the Deployment User

- [ ] Identify the user you use for SSH and deployments.
- [ ] Confirm the user belongs to the expected groups.

```zsh
whoami
groups
```

For example:

```text
angela sudo www-data
```

or:

```text
heidi sudo www-data
```

## 2. Check the Apache File Owner and Group

- [ ] The deployment user should own the site files.
- [ ] The group should be `www-data`.
- [ ] Look for files unexpectedly owned by `root` or another project's user.

Main site:

```zsh
sudo find /var/www/html -maxdepth 2 -printf '%M %u:%g %p\n'
```

Subdomain:

```zsh
sudo find /var/www/colorado.practicelayouts.com -maxdepth 3 \
  -printf '%M %u:%g %p\n'
```

Expected ownership examples:

```text
angela:www-data
heidi:www-data
```

Repair patterns:

```zsh
sudo chown -R angela:www-data /var/www/html
```

```zsh
sudo chown -R heidi:www-data \
  /var/www/colorado.practicelayouts.com
```

## 3. Check Directory and File Permissions

- [ ] Directories should normally be `2755`.
- [ ] Regular files should normally be `644`.
- [ ] Files should not all be executable.
- [ ] The entire website should not be broadly writable with `775`.

```zsh
sudo find /var/www/html -type d ! -perm 2755 -ls
sudo find /var/www/html -type f ! -perm 0644 -ls
```

Repair:

```zsh
sudo find /var/www/html -type d -exec chmod 2755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

Subdomain equivalent:

```zsh
sudo find /var/www/colorado.practicelayouts.com \
  -type d -exec chmod 2755 {} \;

sudo find /var/www/colorado.practicelayouts.com \
  -type f -exec chmod 644 {} \;
```

Application-specific secrets and writable directories may need different permissions.

## 4. Confirm Apache's Virtual-Host Mapping

- [ ] Every domain should map to the intended configuration file.
- [ ] Every document root should point to the intended project.
- [ ] No hostname should unexpectedly use the default site.

```zsh
sudo apache2ctl -S
```

For the Colorado site, expect to see:

```text
colorado.practicelayouts.com
```

associated with:

```text
colorado.practicelayouts.com.conf
colorado.practicelayouts.com-le-ssl.conf
```

## 5. Check Enabled and Available Sites

- [ ] Confirm each required port 80 site is enabled.
- [ ] Confirm each Certbot SSL site is enabled.
- [ ] Look for obsolete or incorrectly named sites.

```zsh
ls -la /etc/apache2/sites-available
ls -la /etc/apache2/sites-enabled
sudo a2query -s
```

The Colorado server should not rely on:

```text
000-default-le-ssl.conf
```

for the Colorado subdomain. It should use the domain-specific SSL configuration.

## 6. Search for Copied Configuration Mistakes

- [ ] Confirm each file contains only the correct domain.
- [ ] Look for `thelemonstack.com`, `fiberandkraft.com`, or another unrelated project inside the wrong server configuration.

```zsh
sudo grep -R \
  "ServerName\|ServerAlias\|DocumentRoot\|Redirect\|RewriteRule" \
  /etc/apache2/sites-available/
```

For a single subdomain file:

```zsh
sudo grep -n \
  "ServerName\|ServerAlias\|DocumentRoot\|Redirect\|RewriteRule" \
  /etc/apache2/sites-available/colorado.practicelayouts.com.conf
```

## 7. Check Directory-Listing Settings

- [ ] Look for `Options Indexes`.
- [ ] Replace it with `Options -Indexes +FollowSymLinks` where appropriate.

```zsh
sudo grep -R "Options.*Indexes" /etc/apache2/
```

Preferred:

```apache
Options -Indexes +FollowSymLinks
```

## 8. Validate Apache Before Changing Anything

- [ ] The current Apache configuration should pass its syntax test.

```zsh
sudo apache2ctl configtest
```

Expected:

```text
Syntax OK
```

After making any correction:

```zsh
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Do not reload Apache if the test fails.

## 9. Check Certbot's Domains and Files

- [ ] Each certificate should contain the expected domain names.
- [ ] Root sites using `www` should include both names.
- [ ] A standalone subdomain needs only that subdomain.
- [ ] Certificate configuration names should match the correct site.

```zsh
sudo certbot certificates
```

Examples:

```text
Domains: fiberandkraft.com www.fiberandkraft.com
```

```text
Domains: thelemonstack.com www.thelemonstack.com
```

```text
Domains: colorado.practicelayouts.com
```

## 10. Test the Actual Site Mapping

- [ ] HTTP should redirect to HTTPS.
- [ ] HTTPS should serve the correct project.
- [ ] `www` should redirect to the preferred root domain where configured.
- [ ] No URL should enter a redirect loop.

```zsh
curl -IL http://example.com
curl -IL https://example.com
curl -IL http://www.example.com
curl -IL https://www.example.com
```

For Colorado:

```zsh
curl -IL http://colorado.practicelayouts.com
curl -IL https://colorado.practicelayouts.com
```

Start with the deployment user, owner/group, permissions, virtual-host mapping, enabled sites, copied domain mistakes, directory listings, Apache syntax, certificates, and redirects.
