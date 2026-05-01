thinking/actions.md
# Deploying Docker Container to DigitalOcean with GitHub Actions

This guide explains how to set up automatic deployment of a Docker container to a DigitalOcean droplet using GitHub Actions.

## Overview

When you push to the `main` branch, the GitHub Action will:
1. Build a Docker image from the code
2. Push the image to Docker Hub
3. SSH into your DigitalOcean droplet
4. Pull the new image and restart the container

## Prerequisites

### DigitalOcean Droplet
- A Linux droplet with Docker installed
- SSH access configured
- Port 80 open in the firewall

Install Docker on your droplet:
```bash
curl -fsSL https://get.docker.com | sh
systemctl start docker
systemctl enable docker
```

### Docker Hub Account
- A Docker Hub account (free is fine)
- Create an access token: Docker Hub → Account Settings → Security → New Access Token

## Required Secrets

Go to your GitHub repository **Settings → Secrets and variables → Actions** and add these secrets:

| Secret | Description | Example |
|--------|-------------|---------|
| `DOCKER_USERNAME` | Your Docker Hub username | `johndoe` |
| `DOCKER_PASSWORD` | Your Docker Hub access token | `dckr_pat_xxx` |
| `DROPLET_HOST` | Your droplet's IP address | `164.92.188.1` |
| `DROPLET_USER` | SSH user (usually `root`) | `root` |
| `DROPLET_SSH_KEY` | Private SSH key | `-----BEGIN RSA...` |

### Generating SSH Key for Droplet

1. On your local machine, generate a new SSH key:
```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

2. Add the public key to your droplet:
```bash
cat ~/.ssh/id_rsa.pub | ssh root@YOUR_DROPLET_IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

3. Add the private key (the content of `~/.ssh/id_rsa`) as the `DROPLET_SSH_KEY` secret in GitHub.

## Project Structure

```
thinking/
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions workflow
├── html/
│   └── index.html              # Static site content
└── Dockerfile                  # Nginx Docker image definition
```

## How It Works

### Workflow File (.github/workflows/deploy.yml)

The workflow triggers on every push to `main` and:

1. **Checkout** - Downloads your code
2. **Set up Docker Buildx** - Enables Docker build features
3. **Login to Docker Hub** - Authenticates using the secrets
4. **Extract metadata** - Creates tags for the image
5. **Build and push** - Builds the Docker image and pushes to Docker Hub
6. **Deploy to droplet** - SSH in and restart the container

### Deployment Script

The SSH action runs these commands on your droplet:
```bash
# Login to Docker Hub
echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

# Pull the latest image
docker pull username/nginx-static-site:COMMIT_SHA

# Stop and remove old container
docker stop nginx-static-site || true
docker rm nginx-static-site || true

# Run new container
docker run -d --name nginx-static-site -p 80:80 --restart always username/nginx-static-site:COMMIT_SHA

# Cleanup
docker image prune -f
```

## Debugging

### Check Workflow Status
1. Go to your repository on GitHub
2. Click on the **Actions** tab
3. Click on the most recent workflow run
4. Check each step for errors (highlighted in red)

### Common Issues

**"Invalid workflow file" YAML syntax error**
- Check for extra characters or formatting issues in the YAML file
- Ensure no backticks or markdown syntax is included in the file

**Docker Hub login failed**
- Verify `DOCKER_USERNAME` and `DOCKER_PASSWORD` are correct
- Make sure the access token has the correct permissions

**SSH connection failed**
- Verify `DROPLET_HOST`, `DROPLET_USER`, and `DROPLET_SSH_KEY` are correct
- Ensure the SSH key is authorized on your droplet

**Container not running**
- SSH into your droplet manually and check: `docker ps -a`
- Check logs: `docker logs nginx-static-site`

## Using GitHub Container Registry (Alternative)

If you prefer GitHub's container registry instead of Docker Hub:

1. Change the login step:
```yaml
- name: Login to GitHub Container Registry
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.repository_owner }}
    password: ${{ secrets.GITHUB_TOKEN }}
```

2. Update the image name to use `ghcr.io/your-username/nginx-static-site`.

The `GITHUB_TOKEN` secret is automatically available, so you don't need to add it manually.

## Manual Deployment

To manually trigger a deployment:
```bash
git push origin main
```

Or to test the workflow without pushing:
1. Go to **Actions** tab
2. Click on "Deploy to DigitalOcean"
3. Click "Run workflow" → "Run workflow"