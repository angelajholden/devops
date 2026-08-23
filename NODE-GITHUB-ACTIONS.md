# Deploy a Node App to DigitalOcean With GitHub Actions

This guide creates a production deployment workflow for a Node application hosted on a DigitalOcean Droplet. The worked example deploys an Express API to `/var/www/html`, installs production dependencies, and reloads the app with PM2 whenever changes reach the `main` branch.

The same process can deploy other Node services. Change the application directory, entry point, process name, health-check URL, and build commands to match the project.

## How the Deployment Workflow Works

```text
Local integrations branch
        ↓
GitHub integrations branch
        ↓
Development testing
        ↓
Pull request into main
        ↓
Push created by the merge
        ↓
GitHub Actions production deployment
        ↓
DigitalOcean /var/www/html
        ↓
npm ci --omit=dev
        ↓
PM2 reload
        ↓
Local health check
```

A pull request does not directly trigger this production workflow. Merging the pull request creates a push to `main`, and that push triggers the deployment.

## Prerequisites

- An existing DigitalOcean Node.js Droplet
- Node.js, npm, Nginx, and PM2 installed on the Droplet
- A Node application already working through Nginx
- A non-root administrative SSH user with sudo access
- A project hosted in a GitHub repository
- GitHub Actions enabled for the repository
- A committed `package-lock.json`
- A known application directory such as `/var/www/html`

Complete and audit the server setup first:

- [How to Deploy a Node.js App to DigitalOcean 💧 Nginx + PM2 + Subdomain DNS](NODE.md)
- [Node Server Audit Checklist](NODE-AUDIT.md)

## Choose the Process Owner

PM2 keeps a separate process list for every Linux user. The user that runs the production PM2 process must remain consistent across manual and automated deployments.

This guide uses a dedicated non-sudo user named `deploy` to:

- Own the application files
- Receive files through SSH
- Install application dependencies
- Run the production PM2 process
- Save the PM2 process list

The workflow does not receive sudo access and cannot modify Nginx, UFW, users, or other system configuration.

If the app currently runs under another user such as `angela`, migrate it deliberately. Do not leave the same app running in two separate PM2 environments.

## Create the Deployment User

SSH into the Droplet as the existing sudo user:

```zsh
sudo adduser deploy
```

Do not add `deploy` to the `sudo` group.

Make it the owner of the application directory:

```zsh
sudo chown -R deploy:deploy /var/www/html
sudo find /var/www/html -path /var/www/html/node_modules -prune -o \
  -type d -exec chmod 755 {} \;
sudo find /var/www/html -path /var/www/html/node_modules -prune -o \
  -type f ! -name '.env' -exec chmod 644 {} \;
sudo chmod 600 /var/www/html/.env
```

If the application does not use a `.env` file, skip the last command. Do not apply blanket permissions inside `node_modules`; installed packages can contain executable files and symlinks with different modes.

## Migrate the PM2 Process

First inspect every relevant PM2 environment:

```zsh
pm2 list
sudo -u nodejs pm2 list
sudo -u deploy pm2 list
sudo ss -ltnp | grep ':3000'
```

Stop the existing production process under its old user. For example, if it currently runs under `angela`:

```zsh
pm2 delete express-server
pm2 save
```

Also remove the DigitalOcean demo process if it remains:

```zsh
sudo -u nodejs pm2 delete hello
sudo -u nodejs pm2 save
```

Do not continue until port 3000 is free:

```zsh
sudo ss -ltnp | grep ':3000'
```

No output means nothing is listening on that port.

## Add a PM2 Ecosystem File

Create `ecosystem.config.cjs` in the Node project and commit it to Git:

```js
module.exports = {
	apps: [
		{
			name: "express-server",
			script: "./index.js",
			cwd: "/var/www/html",
			env: {
				NODE_ENV: "production",
			},
		},
	],
};
```

The `.cjs` extension lets PM2 load this CommonJS configuration even when the application uses `"type": "module"`.

Adapt these values for another Node app:

| Setting  | Purpose                                  |
| -------- | ---------------------------------------- |
| `name`   | Stable PM2 process name                  |
| `script` | Application entry point                  |
| `cwd`    | Production application directory         |
| `env`    | Non-secret production environment values |

Do not put API keys, tokens, passwords, or other secrets in this committed file.

## Start PM2 Under the Deployment User

Install dependencies and start the app as `deploy`:

```zsh
sudo -iu deploy
cd /var/www/html
npm ci --omit=dev
pm2 start ecosystem.config.cjs --env production
pm2 save
curl http://127.0.0.1:3000/instagram/images
exit
```

Replace the health-check path with a stable endpoint provided by the application. A dedicated endpoint such as `/health` is preferable to one that calls an external API.

Configure PM2 to restore this user's process list after a reboot:

```zsh
sudo -iu deploy pm2 startup
```

PM2 prints a command beginning with `sudo env`. Run that exact command from the administrative account, then save the deployment user's process list again:

```zsh
sudo -iu deploy pm2 save
sudo systemctl status pm2-deploy
```

## Create a Dedicated SSH Key

Create a key specifically for this deployment on your local computer:

```zsh
ssh-keygen -m PEM -t rsa -b 4096 \
  -f ~/.ssh/express-server-deploy
```

Leave the passphrase empty because the rsync action cannot enter one interactively.

The command creates:

```text
express-server-deploy       Private key
express-server-deploy.pub   Public key
```

- Use a dedicated deployment key, not an administrator key.
- Add the public key to the server.
- Add the private key to GitHub Actions secrets.
- Never display or commit the private key.

## Authorize the Public Key

On the Droplet:

```zsh
sudo -iu deploy
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Paste the complete contents of `express-server-deploy.pub`. Do not paste the private key.

```zsh
chmod 600 ~/.ssh/authorized_keys
exit
sudo chown -R deploy:deploy /home/deploy/.ssh
```

## Test the Deployment Account

From your local computer:

```zsh
ssh -i ~/.ssh/express-server-deploy \
  deploy@api.practicelayouts.com
```

Test file access, dependencies, PM2, and the local application:

```zsh
touch /var/www/html/deploy-test.txt
rm /var/www/html/deploy-test.txt
cd /var/www/html
npm --version
pm2 list
curl http://127.0.0.1:3000/instagram/images
```

Do not create the action until these checks work without sudo.

## Protect Production Secrets

Keep the production `.env` file on the Droplet:

```zsh
sudo chown deploy:deploy /var/www/html/.env
sudo chmod 600 /var/www/html/.env
```

The workflow must exclude `.env` so it neither uploads a local file nor replaces the production file. Never commit `.env` to Git.

If the application uses a managed secret service instead of `.env`, adapt the deployment without printing secret values in workflow logs.

## Create GitHub Actions Secrets

Navigate to:

```text
Repository
→ Settings
→ Secrets and variables
→ Actions
→ New repository secret
```

Create these repository secrets:

| Secret               | Value                                       |
| -------------------- | ------------------------------------------- |
| `DO_SSH_PRIVATE_KEY` | Complete private deployment key             |
| `DO_HOST`            | `api.practicelayouts.com` or the Droplet IP |
| `DO_USER`            | `deploy`                                    |
| `DO_TARGET_DIR`      | `/var/www/html`                             |

Do not put secret values directly in the workflow file.

For additional isolation, store production secrets in a GitHub `production` environment and restrict which branches can deploy to it. Feature availability depends on repository visibility and GitHub plan.

## Create the Workflow File

Create:

```text
.github/workflows/deploy-production.yml
```

## Production Workflow

This workflow verifies the dependency lockfile on the GitHub runner, deploys the application files, installs production dependencies on the Droplet, reloads PM2, saves the process list, and runs a local health check.

```yaml
name: Deploy Node App to DigitalOcean

on:
    push:
        branches:
            - main

permissions:
    contents: read

concurrency:
    group: node-production-deployment
    cancel-in-progress: false

jobs:
    deploy:
        runs-on: ubuntu-latest

        steps:
            - name: Checkout repository
              uses: actions/checkout@v4

            - name: Set up Node.js
              uses: actions/setup-node@v4
              with:
                  node-version: 22
                  cache: npm

            - name: Verify dependencies
              run: npm ci

            - name: Run tests when available
              run: npm test --if-present

            - name: Deploy and reload application
              uses: easingthemes/ssh-deploy@v6.0.3
              with:
                  SSH_PRIVATE_KEY: ${{ secrets.DO_SSH_PRIVATE_KEY }}
                  REMOTE_HOST: ${{ secrets.DO_HOST }}
                  REMOTE_USER: ${{ secrets.DO_USER }}
                  TARGET: ${{ secrets.DO_TARGET_DIR }}
                  SOURCE: ./
                  EXCLUDE: ".git/, .gitignore, .github/, node_modules/, .env, .DS_Store, README.md, LICENSE.md, TODO.md"
                  SCRIPT_AFTER: |
                      set -e
                      cd /var/www/html
                      npm ci --omit=dev
                      chmod 600 .env
                      pm2 startOrReload ecosystem.config.cjs --env production
                      pm2 save
                      curl --fail --silent --show-error http://127.0.0.1:3000/instagram/images > /dev/null
                  SCRIPT_AFTER_REQUIRED: true
```

Change these project-specific values for another Node app:

- Node major version
- Target and working directory
- PM2 ecosystem filename
- Exclusion list
- Build or migration commands
- Health-check URL

The GitHub runner's Node version should match the production server's supported major version. Confirm the server version with:

```zsh
node --version
```

The current `express-server` project has no test script, so `npm test --if-present` exits successfully without running tests. Add meaningful automated tests when the project has them.

## What the Workflow Does

- `push` restricts production deployment to changes reaching `main`.
- `permissions: contents: read` limits the workflow token.
- `concurrency` prevents overlapping production deployments.
- `npm ci` verifies that the committed lockfile installs successfully.
- `SOURCE` sends application source, not local dependencies or secrets.
- `npm ci --omit=dev` installs a clean production dependency tree on the Droplet.
- `pm2 startOrReload` starts the process on the first run and reloads it on later deployments.
- `pm2 save` updates the process list restored after reboot.
- The final `curl` fails the deployment if the local app does not respond successfully.
- `SCRIPT_AFTER_REQUIRED: true` makes remote-script failures fail the workflow.

## Review the Source and Exclusions

Do not copy the exclusion list blindly. Confirm that each excluded path is unnecessary in production.

Always exclude:

- `.git`
- `.github`
- Local `node_modules`
- `.env`
- Development-only documentation and design files

The server should generate `node_modules` using the committed lockfile. Production secrets should already exist on the Droplet.

For a TypeScript or compiled application, build on the GitHub runner and deploy only the reviewed build output plus the production package files. Adapt `SOURCE`, the install step, and the PM2 entry point accordingly.

## Decide Whether to Delete Remote Files

This workflow does not enable rsync's `--delete` option. Without it, files removed from Git can remain on the server.

Adding `--delete` makes production mirror the deployment source, but it can remove valid server files if the source or exclusions are wrong. The production `.env` must remain protected by an exclusion.

Test the equivalent rsync command manually with `--dry-run` before enabling deletion. Review [RSYNC.md](RSYNC.md) for the safe preview workflow.

## Optional: Verify the Server Host Key

The `ssh-deploy` action documents a default that disables strict host-key checking. For a hardened deployment, store a previously verified `known_hosts` entry in a protected `DO_KNOWN_HOSTS` secret.

Add this step before deployment:

```yaml
- name: Configure known hosts
  shell: bash
  run: |
      install -m 700 -d ~/.ssh
      printf '%s\n' "${{ secrets.DO_KNOWN_HOSTS }}" > ~/.ssh/known_hosts
      chmod 600 ~/.ssh/known_hosts
```

Then add this input to the reviewed action version:

```yaml
SSH_CMD_ARGS: "-o StrictHostKeyChecking=yes"
```

Verify the host key through an existing trusted administrator connection or the DigitalOcean console. `ssh-keyscan` discovers a key but does not prove the server's identity by itself.

## First Deployment

1. Confirm the app works under the `deploy` user's PM2 environment.
2. Confirm `pm2-deploy` starts after reboot.
3. Confirm the dedicated SSH key works.
4. Confirm `deploy` can write to `/var/www/html` without sudo.
5. Confirm `.env` exists and is `600`.
6. Add the GitHub secrets.
7. Commit the ecosystem and workflow files.
8. Push the workflow to `main` for the initial test.
9. Open the repository's **Actions** tab.
10. Confirm checkout, dependency verification, deployment, PM2 reload, and the health check succeed.
11. Verify the public HTTPS endpoint.

Server checks:

```zsh
sudo ss -ltnp | grep ':3000'
sudo -u deploy pm2 list
sudo systemctl status pm2-deploy
curl http://127.0.0.1:3000/instagram/images
curl -IL https://api.practicelayouts.com/instagram/images
```

## Normal Branch Workflow

Create the development branch before beginning the normal cycle:

```zsh
git switch -c integrations
git push -u origin integrations
```

Normal deployments follow this process:

1. Work locally on `integrations`.
2. Push and test the development branch.
3. Open a pull request from `integrations` into `main`.
4. Review and merge the pull request.
5. Watch the production action.
6. Verify PM2 and the public endpoint.

## Roll Back a Failed Deployment

The workflow modifies the live application directory before reloading PM2. If a deployment fails after files are transferred:

1. Revert the problematic commit or restore a known-good commit.
2. Push or merge the correction into `main`.
3. Watch the replacement deployment.
4. Confirm PM2 is online and the health check passes.

For higher-risk applications, use versioned release directories and an atomic symlink switch instead of deploying directly into the live directory.

## Clean Up and Rotate Keys

GitHub does not reveal a secret after it is saved. Store the deployment key in an approved secure credential backup, or document how to rotate it.

To rotate the deployment key:

1. Generate a new dedicated key pair.
2. Add the new public key to `/home/deploy/.ssh/authorized_keys`.
3. Replace `DO_SSH_PRIVATE_KEY` in GitHub.
4. Run and verify a deployment.
5. Remove the old public key.
6. Remove temporary local copies according to your credential policy.

Never remove the old key until the new key has completed a successful deployment.

## Troubleshooting

### Permission denied (publickey)

Check the private-key secret, public key, deployment user, and SSH permissions:

```zsh
sudo ls -la /home/deploy/.ssh
```

Expected:

```text
/home/deploy/.ssh                  deploy:deploy 700
/home/deploy/.ssh/authorized_keys  deploy:deploy 600
```

### rsync reports permission denied

```zsh
sudo chown -R deploy:deploy /var/www/html
sudo -u deploy touch /var/www/html/deploy-test.txt
sudo -u deploy rm /var/www/html/deploy-test.txt
```

### npm ci fails

Confirm `package-lock.json` is committed and matches `package.json`. Check that the production Node version satisfies the project's requirements.

### PM2 cannot find the process or starts a second process

Confirm every PM2 command runs as `deploy`:

```zsh
sudo -u deploy pm2 list
sudo -u nodejs pm2 list
sudo ss -ltnp | grep ':3000'
```

Do not run deployment PM2 commands as root, `nodejs`, or the administrator account.

### The health check fails

Test each layer separately:

```zsh
sudo -u deploy pm2 logs express-server --lines 100
curl http://127.0.0.1:3000/instagram/images
sudo nginx -t
curl -IL https://api.practicelayouts.com/instagram/images
```

Troubleshoot Node and PM2 before changing Nginx if the localhost request fails.

### Removed files remain on the server

This is expected without rsync's `--delete`. Review the deletion section before changing that behavior.

## Final Checklist

- [ ] A dedicated non-sudo `deploy` user owns the application.
- [ ] The production PM2 process runs only under `deploy`.
- [ ] The DigitalOcean demo PM2 process is removed.
- [ ] Exactly one intended process owns port 3000.
- [ ] The PM2 ecosystem file is committed and contains no secrets.
- [ ] PM2 restores the `deploy` process list after reboot.
- [ ] A dedicated deployment SSH key works.
- [ ] The public key is installed with correct permissions.
- [ ] `.env` remains only on the server and is `600`.
- [ ] The four GitHub secrets are configured.
- [ ] The workflow is restricted to pushes to `main`.
- [ ] The workflow token has read-only repository permissions.
- [ ] Concurrent production deployments are prevented.
- [ ] Local `node_modules` and secrets are excluded.
- [ ] Production dependencies install from the lockfile.
- [ ] PM2 reload and save succeed.
- [ ] The local health check passes.
- [ ] The public HTTPS endpoint works.
- [ ] Key rotation and rollback procedures are understood.
