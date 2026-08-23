# Deploy a Static LAMP Site to DigitalOcean With GitHub Actions

This guide creates a production deployment workflow for a static site hosted on a DigitalOcean LAMP Droplet. GitHub Actions uses rsync over SSH to deploy files whenever changes reach the `main` branch.

A dedicated, non-sudo `deploy` user limits the workflow to the website directory. The workflow does not receive an administrator account or permission to modify server configuration.

## How the Deployment Workflow Works

```text
Local integrations branch
        ↓
GitHub integrations branch
        ↓
GitHub Pages development site
        ↓
Pull request into main
        ↓
Push created by the merge
        ↓
GitHub Actions production deployment
        ↓
DigitalOcean /var/www/html
```

A pull request does not directly trigger this production workflow. Merging the pull request creates a push to `main`, and that push triggers the deployment.

## Prerequisites

- An existing DigitalOcean LAMP Droplet
- A static site already working through Apache
- A non-root administrative SSH user with sudo access
- A project hosted in a GitHub repository
- GitHub Actions enabled for the repository
- `rsync` installed on the Droplet
- A known production target such as `/var/www/html`

Complete the applicable server setup before creating the action:

- [How to Deploy a Website to DigitalOcean 💧 LAMP + Rsync (MacOS/Linux) + DNS Setup](RSYNC.md)
- [How to Deploy a Website to DigitalOcean 💧 LAMP + SFTP (Windows) + DNS Setup](SFTP.md)
- [LAMP Server Audit Checklist](LAMP-AUDIT.md)

## Create a Limited Deployment User

SSH into the Droplet as your existing sudo user, then create a dedicated deployment account:

```zsh
sudo adduser deploy
```

Do not add `deploy` to the `sudo` group. A static-site deployment only needs SSH access and permission to modify one document root. It should not be able to install software, restart services, create users, or edit Apache configuration.

Make `deploy` the owner of the production site while keeping Apache's `www-data` group:

```zsh
sudo chown -R deploy:www-data /var/www/html
sudo find /var/www/html -type d -exec chmod 2755 {} \;
sudo find /var/www/html -type f -exec chmod 644 {} \;
```

This intentionally changes the production deployment owner from your manual deployment user to `deploy`. Apache can continue reading the files through the `www-data` group and normal read permissions.

If an application needs writable runtime directories, configure those directories separately. Do not make the entire document root writable by Apache.

## Create a Dedicated SSH Key

Create a key specifically for this GitHub Actions deployment on your local computer:

```zsh
ssh-keygen -m PEM -t rsa -b 4096 \
  -f ~/.ssh/practice-layouts-deploy
```

When prompted for a passphrase, leave it empty. The rsync action used in this guide cannot enter a passphrase interactively.

The command creates two files:

```text
practice-layouts-deploy       Private key
practice-layouts-deploy.pub   Public key
```

- Use a dedicated deployment key, not your personal administrator key.
- Add the public key to the `deploy` account on the server.
- Add the private key to GitHub Actions secrets.
- Never display the private key, secret value, or a command that prints the private key during a stream, screenshot, or recording.

## Authorize the Public Key

On the Droplet, switch to the deployment user and prepare its SSH directory:

```zsh
sudo -iu deploy
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the complete contents of `practice-layouts-deploy.pub` into `authorized_keys`. Do not paste the private key.

Save the file, then set its permissions:

```zsh
chmod 600 ~/.ssh/authorized_keys
exit
```

Confirm the deployment user's SSH ownership and permissions:

```zsh
sudo chown -R deploy:deploy /home/deploy/.ssh
sudo chmod 700 /home/deploy/.ssh
sudo chmod 600 /home/deploy/.ssh/authorized_keys
```

## Test the Deployment Account

Test the dedicated key from your local computer:

```zsh
ssh -i ~/.ssh/practice-layouts-deploy \
  deploy@practicelayouts.com
```

Confirm that `deploy` can create and delete a file in the exact production directory without sudo:

```zsh
touch /var/www/html/deploy-test.txt
rm /var/www/html/deploy-test.txt
```

Do not continue until the dedicated key can log in and the `deploy` user can create and delete the test file.

If the test fails, return to the sudo account and check the document-root ownership:

```zsh
sudo chown -R deploy:www-data /var/www/html
```

## Create GitHub Actions Secrets

Open the GitHub repository and navigate to:

```text
Settings
→ Secrets and variables
→ Actions
→ New repository secret
```

Create these four repository secrets:

| Secret               | Value                                             |
| -------------------- | ------------------------------------------------- |
| `DO_SSH_PRIVATE_KEY` | Complete contents of the private deployment key   |
| `DO_HOST`            | `practicelayouts.com` or the Droplet's IP address |
| `DO_USER`            | `deploy`                                          |
| `DO_TARGET_DIR`      | `/var/www/html`                                   |

For `DO_SSH_PRIVATE_KEY`, copy the entire private key, including its beginning and ending lines. Do not use the `.pub` file.

Do not put secret values directly in the workflow file. GitHub secrets are referenced using expressions such as:

```yaml
${{ secrets.DO_SSH_PRIVATE_KEY }}
```

For additional isolation, you can store production secrets in a GitHub `production` environment and restrict which branches can deploy to it. Environment features and protection-rule availability depend on the repository visibility and GitHub plan.

## Create the Workflow File

Create this file in the project:

```text
.github/workflows/deploy-production.yml
```

`.github` and `workflows` are directories. `deploy-production.yml` is the file.

## Production Workflow

This version follows the workflow created for Practice Layouts and adds explicit read-only GitHub token permissions and deployment concurrency:

```yaml
name: Deploy to DigitalOcean

on:
    push:
        branches:
            - main

permissions:
    contents: read

concurrency:
    group: production-deployment
    cancel-in-progress: false

jobs:
    deploy:
        runs-on: ubuntu-latest

        steps:
            - name: Checkout repository
              uses: actions/checkout@v4

            - name: Deploy files
              uses: easingthemes/ssh-deploy@v6.0.3
              with:
                  SSH_PRIVATE_KEY: ${{ secrets.DO_SSH_PRIVATE_KEY }}
                  REMOTE_HOST: ${{ secrets.DO_HOST }}
                  REMOTE_USER: ${{ secrets.DO_USER }}
                  TARGET: ${{ secrets.DO_TARGET_DIR }}
                  SOURCE: ./
                  EXCLUDE: ".git/, .gitignore, .github/, svg/, snippets.html, .DS_Store, README.md, LICENSE.md, QA.md"
```

The original Practice Layouts workflow used `easingthemes/ssh-deploy@v5.1.0`. The example uses the current `v6.0.3` release. Review release notes before upgrading an existing workflow.

For stronger supply-chain protection, GitHub recommends pinning third-party actions to a reviewed full commit SHA instead of a movable version tag. A SHA-pinned line looks like:

```yaml
uses: easingthemes/ssh-deploy@<full-reviewed-commit-sha> # v6.0.3
```

Do not substitute an abbreviated SHA. Resolve and review the full commit for the release you intend to use.

### What the Workflow Does

- `push` limits the production trigger to changes reaching `main`.
- `permissions: contents: read` gives the workflow's GitHub token only the repository access needed for checkout.
- `concurrency` prevents two production deployments from running simultaneously.
- `ubuntu-latest` describes the temporary GitHub-hosted runner, not the DigitalOcean Droplet.
- `actions/checkout` places the repository on the runner.
- `ssh-deploy` runs rsync over SSH.
- `SOURCE` is relative to the checked-out repository.
- `TARGET` is the directory on the DigitalOcean server.
- `EXCLUDE` is a comma-separated list for this action.

## Review the Source and Exclusions

The Practice Layouts project is a vanilla HTML, CSS, and JavaScript site without a build step, so it deploys the repository root and excludes development-only files.

Do not copy the exclusion list blindly. Confirm that every excluded path is unnecessary in production. For example, exclude `svg/` only if it contains design references rather than images required by the site.

The checked-out repository includes `.git`, so it must be excluded along with `.github` and other development-only content.

For future projects, a dedicated deployable directory such as `dist/` or `public/` is safer than deploying the repository root and maintaining a growing exclusion list. A build-based project would usually use something like:

```yaml
SOURCE: dist/
```

## Decide Whether to Delete Remote Files

The Practice Layouts workflow does not use rsync's `--delete` option. Without `--delete`, a file removed from Git can remain on the production server.

Adding `--delete` makes production mirror the deployment source, but an incorrect source path or exclusion can remove valid production files. Treat it as an intentional deployment decision, not a default.

Before adding it to the action, test the equivalent rsync command manually with `--dry-run` as described in [RSYNC.md](RSYNC.md).

If you decide production should mirror the repository, add reviewed rsync arguments supported by the selected action version. Do not enable `--delete` until the dry-run output is correct.

## Optional: Verify the Server Host Key

The `ssh-deploy` action documents a default SSH option that disables strict host-key checking. That is convenient for initial setup, but it does not verify that the runner connected to the intended server.

For a hardened deployment:

1. Obtain the Droplet's SSH host key fingerprint through a trusted channel, such as an existing verified administrator connection or the DigitalOcean console.
2. Compare it with the server's public host key.
3. Store the verified `known_hosts` entry as a protected GitHub secret such as `DO_KNOWN_HOSTS`.
4. Populate the runner's `known_hosts` file before deployment.
5. Set strict host-key checking to `yes` for the deployment action.

Do not trust an unverified `ssh-keyscan` result by itself; it discovers a key but does not prove the server's identity.

Example hardening step:

```yaml
- name: Configure known hosts
  shell: bash
  run: |
      install -m 700 -d ~/.ssh
      printf '%s\n' "${{ secrets.DO_KNOWN_HOSTS }}" > ~/.ssh/known_hosts
      chmod 600 ~/.ssh/known_hosts
```

Then add this input to the deployment step if it is supported by the reviewed action version:

```yaml
SSH_CMD_ARGS: "-o StrictHostKeyChecking=yes"
```

## Test the First Deployment

For the initial production test:

1. Confirm the dedicated deployment key works manually.
2. Confirm `deploy` can write to the target directory.
3. Add the four GitHub secrets.
4. Commit `.github/workflows/deploy-production.yml`.
5. Push the workflow to `main`.
6. Open the repository's **Actions** tab.
7. Open the **Deploy to DigitalOcean** workflow run.
8. Confirm both checkout and deployment steps succeed.
9. Open the production site.
10. Confirm excluded files were not uploaded.

Useful server checks:

```zsh
ls -la /var/www/html
sudo find /var/www/html -maxdepth 2 -printf '%M %u:%g %p\n'
```

## Create the Integrations Branch

Create the development branch before beginning the normal deployment cycle:

```zsh
git switch -c integrations
git push -u origin integrations
```

Configure the project's GitHub Pages settings to deploy the `integrations` branch to the development site.

The normal workflow is:

1. Work locally on `integrations`.
2. Push `integrations` to GitHub.
3. Review the GitHub Pages development site.
4. Open a pull request from `integrations` into `main`.
5. Review and merge the pull request.
6. Watch the production GitHub Actions workflow.
7. Verify the production site.

## Clean Up and Rotate Keys

GitHub does not reveal a secret after it has been saved. Store the deployment key in an approved secure credential backup, or document how to rotate it.

If the local private key was created only as a temporary transfer copy, delete it after confirming that the GitHub secret and deployment work. Remember that losing every recoverable copy means you must rotate the key before updating the secret.

To rotate the deployment key:

1. Generate a new dedicated key pair.
2. Add the new public key to `/home/deploy/.ssh/authorized_keys`.
3. Replace `DO_SSH_PRIVATE_KEY` in GitHub.
4. Run and verify a deployment.
5. Remove the old public key from `authorized_keys`.
6. Remove any temporary local copies according to your credential policy.

Never remove the old public key until the new key has completed a successful deployment.

## Troubleshooting

### Permission denied (publickey)

Check:

- `DO_SSH_PRIVATE_KEY` contains the private key, not the `.pub` file.
- `DO_USER` is `deploy`.
- The matching public key is in `/home/deploy/.ssh/authorized_keys`.
- `/home/deploy/.ssh` is `700`.
- `authorized_keys` is `600`.
- The files are owned by `deploy:deploy`.

```zsh
sudo ls -la /home/deploy/.ssh
```

### rsync reports permission denied

Confirm the target and ownership:

```zsh
sudo find /var/www/html -maxdepth 2 -printf '%M %u:%g %p\n'
sudo chown -R deploy:www-data /var/www/html
```

Then test as the deployment user:

```zsh
sudo -u deploy touch /var/www/html/deploy-test.txt
sudo -u deploy rm /var/www/html/deploy-test.txt
```

### The workflow did not run after a pull request

Opening or updating a pull request does not trigger this workflow. The workflow runs when the pull request is merged and GitHub creates a push to `main`.

Confirm that the merge reached `main` and inspect the repository's **Actions** tab.

### Removed files remain on the server

This is expected when the workflow does not use `--delete`. Review the deletion section before changing rsync behavior.

### Unexpected files were deployed

Review `SOURCE` and the comma-separated `EXCLUDE` value. Consider moving production files into a dedicated deployment directory instead of expanding the exclusion list indefinitely.

### GitHub Pages shows the wrong branch

Open the repository's Pages settings and confirm the development site uses `integrations`, not `main`.

## Final Checklist

- [ ] The `deploy` user exists and is not a sudo user.
- [ ] `deploy` owns only the intended website directory.
- [ ] A dedicated deployment SSH key was created.
- [ ] The public key is in the deployment user's `authorized_keys` file.
- [ ] SSH directory and key permissions are correct.
- [ ] Manual SSH and document-root write tests succeed.
- [ ] The four repository secrets are configured.
- [ ] The workflow is in `.github/workflows/deploy-production.yml`.
- [ ] Production deployment is restricted to pushes to `main`.
- [ ] The workflow token has read-only repository permissions.
- [ ] Concurrent production deployments are prevented.
- [ ] `SOURCE` and every exclusion were reviewed for this project.
- [ ] The first workflow run succeeds.
- [ ] The production site contains only required files.
- [ ] The `integrations` branch powers the development site.
- [ ] The key is securely stored or cleaned up according to a documented rotation process.
