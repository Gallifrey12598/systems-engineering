# RustDesk Self-Host Deployment on Ubuntu 24.04 (Docker)

## Overview
This document outlines the steps required to deploy **RustDesk Self-Host** on an **Ubuntu 24.04 Minimal** host using **Docker** as the deployment method.

Official Docker installation documentation can be found here:
https://docs.docker.com/engine/install/ubuntu/

---

## Prerequisites
- Ubuntu 24.04 Minimal installed
- Root or sudo access
- Internet connectivity
- Pre-auth key for the RustDesk Headscale account

---

## Step 1: Remove Existing Docker Installations

Before installing Docker, remove any existing or conflicting Docker-related packages:

```bash
sudo apt remove $(dpkg --get-selections docker.io docker-compose docker-compose-v2 docker-doc podman-docker containerd runc | cut -f1)
```

---

## Step 2: Set Up the Docker Repository

### Add Docker’s Official GPG Key

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### Add the Docker APT Repository

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

```bash
sudo apt update
```

---

## Step 3: Install Docker

Install the latest available Docker packages:

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Verify Docker Status

```bash
sudo systemctl status docker
```

If Docker is not running, start it manually:

```bash
sudo systemctl start docker
```

---

## Step 4: Install and Configure Tailscale (Headscale)

Because the Headscale backbone is deployed in **Google Cloud Platform (GCP)** and not on-premises, the automated `install-tailscale.sh` script cannot be used. Tailscale must be installed and configured manually.

### Install Tailscale Client

Run the official Tailscale installation script:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Additional Tailscale documentation:
https://tailscale.com/kb/1017/install

---

## Step 5: Join the Headscale Network

Bring the Tailscale client online and authenticate using a **pre-auth key**.

> **Note:** You must have a valid pre-auth key for the RustDesk Headscale account. Contact an engineer if you do not have one.

```bash
sudo tailscale up \
  --login-server https://headscale.223cos.net \
  --authkey {PRE_AUTH_KEY} \
  --accept-routes
```

---

## Step 6: Retrieve Tailscale IP Address

Once connected, retrieve and record the Tailscale IP address:

```bash
sudo tailscale ip
```

This IP will be required for downstream RustDesk configuration.

---

## Completion
At this point:
- Docker is installed and running
- Tailscale is installed and connected to Headscale
- The host is ready for RustDesk Self-Host deployment

Proceed with RustDesk container deployment as required.
