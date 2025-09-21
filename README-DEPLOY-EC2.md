## Deploy to EC2 - Staging (GitHub Actions)

This document shows how to prepare an EC2 instance and configure GitHub Actions to automatically deploy the Node.js app to a staging EC2 instance on push to `main`.

Prerequisites
- AWS account with permissions to create EC2 instances and key pairs
- GitHub repository admin access (to add secrets)

High-level flow
1. Prepare an EC2 instance (Ubuntu 22.04 LTS recommended).
2. Create an SSH key pair or use an existing one; add public key to `ec2-user` (or custom user).
3. Add required GitHub secrets (see list below).
4. Push to `main` and check Actions -> workflow run logs.

Required GitHub secrets
- `EC2_HOST` - public IP or DNS of EC2 (e.g. ec2-3-21-45-123.compute-1.amazonaws.com)
- `EC2_USER` - SSH username (e.g. ubuntu)
- `EC2_SSH_KEY` - **private** SSH key (PEM) contents (newline preserved)
- `EC2_SSH_PORT` - SSH port (usually `22`)
- `EC2_APP_PATH` - absolute path on the EC2 instance where app will be deployed (e.g. `/home/ubuntu/app`)
- `EC2_PM2_APP_NAME` - the pm2 process name to run the app (e.g. `aws-study-staging`)

EC2 setup (Ubuntu)
1. Launch an EC2 instance (Ubuntu 22.04) with a security group that allows inbound TCP 22 (SSH) and inbound TCP 3000 (or whichever port your app uses).
2. SSH into the instance using your key:

```powershell
ssh -i path\to\key.pem ubuntu@<EC2_HOST>
```

3. Create deploy user (optional):

```bash
sudo adduser deploy
sudo usermod -aG sudo deploy
mkdir -p /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
# paste your public key into /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```

4. Install Node.js and PM2 (as the deploy user):

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs build-essential
sudo npm install -g pm2
```

5. Create the app directory and give ownership to deploy user:

```bash
sudo mkdir -p /home/deploy/app
sudo chown -R deploy:deploy /home/deploy/app
```

6. (Optional) Set up a process manager systemd service for PM2 startup:

```bash
pm2 startup systemd
sudo env PATH=$PATH:/usr/bin pm2 startup systemd -u deploy --hp /home/deploy
pm2 save
```

How the GitHub Actions workflow works
- On push to `main`, workflow installs dependencies, runs tests, zips the repo, scp the zip to `$EC2_APP_PATH/app.zip`, SSH into EC2, unzips to `release/`, runs `npm ci --production` (or `npm install`), and restarts the app using `pm2` with the configured name.

Triggering and verifying
1. Push commits to `main`.
2. Go to GitHub -> Actions -> `Build, Test, and Deploy to AWS Staging` (or the updated workflow) and watch the run.
3. On the EC2 instance, verify process:

```bash
pm2 status
curl http://localhost:3000/   # or the port your app listens on
```

Notes & Troubleshooting
- If deployment fails due to SSH or permissions, verify your `EC2_SSH_KEY` secret is the exact private key content and that the public key is in `authorized_keys` for the target user.
- If pm2 isn't starting, check `~/.pm2/logs` for error logs.

Security
- Keep your private SSH key secret. Use GitHub Secrets to store it.
- Consider deploying via a bastion host or using AWS CodeDeploy for more advanced deployment needs.

If you want, I can:
- Update the workflow to use `rsync` instead of `scp` to copy only changed files.
- Add a systemd unit file template instead of using pm2.
- Configure nginx as a reverse proxy and SSL with Let's Encrypt.
