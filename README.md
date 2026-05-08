# AWS Private EC2 Architecture — SSM, Jenkins, Load Balancer, EFS

> A production-style reference for deploying a Django application on a **private EC2 instance** with **AWS Systems Manager (SSM)** for access, **Jenkins** for CI/CD, an **Application Load Balancer** for ingress, **NAT Gateway** for egress, **Amazon EFS** for shared media storage, and **Nginx + Gunicorn** as the application runtime.

![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20SSM%20%7C%20ELB%20%7C%20EFS-FF9900?logo=amazon-aws&logoColor=white)
![Jenkins](https://img.shields.io/badge/CI%2FCD-Jenkins-D24939?logo=jenkins&logoColor=white)
![Nginx](https://img.shields.io/badge/Reverse%20Proxy-Nginx-009639?logo=nginx&logoColor=white)
![Gunicorn](https://img.shields.io/badge/WSGI-Gunicorn-499848?logo=gunicorn&logoColor=white)
![Django](https://img.shields.io/badge/Framework-Django-092E20?logo=django&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.10.8-3776AB?logo=python&logoColor=white)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu-E95420?logo=ubuntu&logoColor=white)

---

## Overview

This repository documents an end-to-end, security-first deployment pattern for a backend EC2 instance that:

- Has **no public IP** and **no SSH port exposed**
- Is reached by engineers and CI/CD through **AWS SSM Session Manager**
- Sits behind an **Application Load Balancer** for public ingress
- Reaches the internet only through a **NAT Gateway**
- Runs a **Django application via Gunicorn**, fronted by **Nginx**
- Persists shared, user-uploaded media on **Amazon EFS**
- Is deployed by **Jenkins** through **SSM Run Command**, eliminating long-lived SSH credentials

It is intended both as a reference architecture and as a step-by-step runbook for bootstrapping a new private EC2 host.

---

## Architecture

```text
                         ┌──────────────────────┐
       Public Internet ─▶│  Application Load    │
                         │  Balancer (public)   │
                         └──────────┬───────────┘
                                    │  HTTP/HTTPS
                                    ▼
  ┌──────────────────────────────────────────────────────────┐
  │                      PRIVATE SUBNET                      │
  │                                                          │
  │   ┌──────────┐      ┌──────────┐      ┌──────────────┐   │
  │   │  Nginx   │ ───▶ │ Gunicorn │ ───▶ │    Django    │   │
  │   │ (proxy)  │      │  (WSGI)  │      │ application  │   │
  │   └──────────┘      └──────────┘      └──────────────┘   │
  │        ▲                                                 │
  │        │  /var/www/media ── symlink ──┐                  │
  │        │                              ▼                  │
  │        │                    ┌─────────────────────┐      │
  │        │                    │  EFS mount point    │      │
  │        │                    │  /mnt/storage       │      │
  │        │                    └─────────────────────┘      │
  │                                                          │
  │  Outbound  ──▶ NAT Gateway ──▶ Internet Gateway          │
  │  Mgmt I/O  ◀── AWS SSM Session Manager / Run Command     │
  └──────────────────────────────────────────────────────────┘

  CI/CD:  Jenkins ──▶ AWS SSM Run Command ──▶ /usr/local/bin/deploy.sh
```

---

## Tech Stack

| Layer          | Component                              |
| -------------- | -------------------------------------- |
| Compute        | Amazon EC2 (private subnet)            |
| Access         | AWS Systems Manager Session Manager    |
| CI/CD          | Jenkins + AWS SSM Run Command          |
| Ingress        | Application Load Balancer              |
| Egress         | NAT Gateway + Internet Gateway         |
| Web server     | Nginx                                  |
| App server     | Gunicorn (systemd-managed)             |
| Application    | Django (Python 3.10.8)                 |
| Shared storage | Amazon EFS (NFSv4.1)                   |
| OS             | Ubuntu                                 |

---

## Table of Contents

1. [Networking foundation](#1-networking-foundation)
2. [Why SSM replaces SSH](#2-why-ssm-replaces-ssh)
3. [Base OS preparation](#3-base-os-preparation)
4. [Application system user](#4-application-system-user)
5. [Application directory layout](#5-application-directory-layout)
6. [Installing Python 3.10.8 from source](#6-installing-python-3108-from-source)
7. [Python virtual environment](#7-python-virtual-environment)
8. [Deployment script](#8-deployment-script)
9. [Nginx reverse proxy](#9-nginx-reverse-proxy)
10. [Gunicorn systemd service](#10-gunicorn-systemd-service)
11. [Shared environment, keys, cache, temp folders](#11-shared-environment-keys-cache-temp-folders)
12. [AWS CLI on EC2](#12-aws-cli-on-ec2)
13. [EFS mount setup](#13-efs-mount-setup)
14. [Media folder on EFS](#14-media-folder-on-efs)
15. [Auto-mount EFS after reboot](#15-auto-mount-efs-after-reboot)
16. [Migrating existing media](#16-migrating-existing-media)
17. [Media permissions for Nginx](#17-media-permissions-for-nginx)
18. [Troubleshooting commands](#18-troubleshooting-commands)
19. [Bootstrap checklist for a new EC2](#19-bootstrap-checklist-for-a-new-ec2)
20. [Production rules](#20-production-rules)

---
## 1. Networking foundation

The networking layout is what makes private EC2 + SSM + Load Balancer work safely.

### Internet Gateway (IGW)

- An IGW provides internet access to resources in a **public subnet**.
- A public subnet's route table must contain:

```text
0.0.0.0/0 → Internet Gateway
```

### NAT Gateway

- A NAT Gateway lets **private EC2 instances** initiate outbound internet traffic.
- It must live in a **public subnet** and uses the IGW for egress.

```text
Private EC2 → NAT Gateway → Internet Gateway → Internet
```

### Private EC2 instance

- The private EC2 has **no public IP**.
- This is the secure home for backend services, workers, and Jenkins-managed application servers.
- The private subnet's route table must contain:

```text
0.0.0.0/0 → NAT Gateway
```

### SSM requirements

For AWS Systems Manager Session Manager to work, the EC2 instance needs:

- An IAM role attached with the policy `AmazonSSMManagedInstanceCore`
- Outbound HTTPS (port `443`) reachability
- Either NAT Gateway access **or** VPC endpoints for SSM (`ssm`, `ssmmessages`, `ec2messages`)

### Common reasons SSM fails

- NAT Gateway placed in the wrong subnet
- Private subnet route table not pointing to the NAT Gateway
- Security group or NACL blocking outbound `443`
- DNS support disabled on the VPC
- Missing or incorrect IAM role
- No internet path through NAT or VPC endpoints

### Subnet layout

```text
PUBLIC SUBNET
- Internet Gateway route
- NAT Gateway
- Application Load Balancer

PRIVATE SUBNET
- EC2 application servers
- Django backend services
- SQS consumers and workers
- Jenkins-managed deployment targets
```

### Quick-reference rule

```text
Public subnet  → IGW
Private subnet → NAT Gateway
SSM needs      → IAM role + outbound 443 + (NAT or VPC endpoints)
```

---

## 2. Why SSM replaces SSH

For this architecture, SSH is intentionally avoided. The EC2 instance is private and must not expose port 22.

All interactive and CI/CD access goes through **AWS Systems Manager Session Manager**, which provides shell-like access from the AWS Console or AWS CLI without opening any inbound ports.

### Benefits over SSH

- No public IP needed
- No inbound port 22
- No SSH key distribution
- Access controlled through IAM
- Session activity logged in CloudTrail / CloudWatch

### Required configuration

Attach this managed policy to the EC2 instance role:

```text
AmazonSSMManagedInstanceCore
```

Confirm outbound HTTPS (`443`) reachability through either:

- NAT Gateway, or
- VPC endpoints for SSM

---

## 3. Base OS preparation

Update the system:

```bash
sudo apt-get update
```

Install build dependencies needed for Python and common Python libraries:

```bash
sudo apt-get install -y \
  build-essential \
  wget \
  curl \
  ca-certificates \
  libssl-dev \
  zlib1g-dev \
  libbz2-dev \
  libreadline-dev \
  libsqlite3-dev \
  libffi-dev \
  libncursesw5-dev \
  xz-utils \
  tk-dev \
  libxml2-dev \
  libxmlsec1-dev \
  liblzma-dev
```

---

## 4. Application system user

Create a dedicated, non-login system user for the application:

```bash
sudo useradd --system --shell /usr/sbin/nologin portal
id portal
```

Why a dedicated user:

- The application never runs as `root`
- Permission boundaries are explicit and minimal
- Jenkins / SSM deployments operate as `portal`
- Gunicorn runs under `portal`

---

## 5. Application directory layout

Create the directory tree:

```bash
sudo mkdir -p /opt/example/web-api/releases
sudo mkdir -p /opt/example/web-api/shared
sudo mkdir -p /opt/python-src /opt/python
```

Set ownership and permissions:

```bash
sudo chown root:root /opt/example
sudo chmod 755 /opt/example

sudo chown -R portal:portal /opt/example/web-api
sudo chmod -R 750 /opt/example/web-api
```

Folder roles:

```text
/opt/example/web-api/releases   → individual deployment releases
/opt/example/web-api/current    → symlink to the active release
/opt/example/web-api/shared     → shared .env, keys, cache, tmp
/opt/example/web-api/venv       → Python virtual environment
```

---

## 6. Installing Python 3.10.8 from source

Download the source:

```bash
cd /opt/python-src
sudo wget https://www.python.org/ftp/python/3.10.8/Python-3.10.8.tgz
```

Extract:

```bash
sudo rm -rf Python-3.10.8
sudo tar -xzf Python-3.10.8.tgz
cd Python-3.10.8
```

Build and install into an isolated prefix:

```bash
sudo ./configure --prefix=/opt/python/3.10.8 --enable-optimizations
sudo make -j"$(nproc)"
sudo make install
```

Verify:

```bash
/opt/python/3.10.8/bin/python3 --version
/opt/python/3.10.8/bin/pip3 --version
```

Lock down the runtime tree:

```bash
sudo chown -R root:root /opt/python
sudo chmod -R 755 /opt/python
```

---

## 7. Python virtual environment

Create the venv as the `portal` user:

```bash
sudo -u portal /opt/python/3.10.8/bin/python3 -m venv /opt/example/web-api/venv
```

Verify:

```bash
sudo -u portal -H bash -lc '
source /opt/example/web-api/venv/bin/activate
which python
python --version
pip --version
readlink -f $(which python)
'
```

Upgrade packaging tools:

```bash
sudo -u portal -H bash -lc '
source /opt/example/web-api/venv/bin/activate
pip install --upgrade pip setuptools wheel
'
```

Install application packages from the deployment script or a `requirements.txt`. Example:

```bash
sudo -u portal /opt/example/web-api/venv/bin/pip install psycopg2==2.9.10
```

---

## 8. Deployment script

Create the deployment entrypoint that Jenkins will invoke through SSM:

```bash
sudo vim /usr/local/bin/deploy.sh
sudo chmod +x /usr/local/bin/deploy.sh
bash -n /usr/local/bin/deploy.sh
```

Responsibilities of `deploy.sh`:

- Pull source from Git into a new release directory
- Install Python dependencies into the venv
- Link the shared `.env` and key material into the release
- Run `collectstatic` and database migrations as needed
- Update the `current` symlink to the new release
- Restart the Gunicorn service

Jenkins triggers this script through **SSM Run Command** rather than over SSH.

---

## 9. Nginx reverse proxy

Install Nginx:

```bash
sudo apt-get update
sudo apt-get install -y nginx

server {
    listen 80 default_server;
    server_name _;

    client_max_body_size 50M;

    # Logs
    access_log /var/log/nginx/example.access.log;
    error_log  /var/log/nginx/example.error.log;

    # -----------------------------
    # ALB health check (IMPORTANT)
    # -----------------------------
    location = /health {
        access_log off;
        return 200 "OK";
        add_header Content-Type text/plain;
    }

    # ---------------------------------------------------
    # If ALB sends /api (without slash), normalize it
    # ---------------------------------------------------
    location = /api {
        return 301 /api/;
    }

    # -----------------------------
    # API reverse proxy
    # -----------------------------
    location /api/ {

        # 🔥 Your backend app (maybe Gunicorn)
        proxy_pass http://127.0.0.1:8000;

        # Timeouts
        proxy_connect_timeout 60s;
        proxy_send_timeout 300s;
        proxy_read_timeout 300s;

        # Keep the host & client IP details
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;

        # ALB terminates HTTPS → this tells Django it was HTTPS
        proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;

        # Optional: if your app expects original URL info
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  $server_port;

        # Avoid proxy buffering issues on APIs / streaming responses
        proxy_buffering off;
    }

    # -----------------------------
    # Default fallback (optional)
    # -----------------------------
    location / {
        return 404;
    }
}

```


Enable and start it:

```bash
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx
```

Remove the default site and create the application config:

```bash
sudo rm -f /etc/nginx/sites-enabled/default
sudo vim /etc/nginx/sites-available/web-api
sudo ln -s /etc/nginx/sites-available/web-api /etc/nginx/sites-enabled/web-api
```

Validate and reload:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Local health check:

```bash
curl -i http://127.0.0.1/health
```

---

## 10. Gunicorn systemd service

Create the service file:

```bash
sudo vim /etc/systemd/system/web-api-gunicorn.service
```

Core `ExecStart` line:

```text
[Unit]
Description=Portal API (Gunicorn)
After=network-online.target
Wants=network-online.target

# Prevent infinite restart loops (recommended for production)
StartLimitIntervalSec=60
StartLimitBurst=5

[Service]
Type=simple
User=adms
Group=adms
UMask=0027

# Your deploy flips this symlink atomically
WorkingDirectory=/opt/example/adms/current

# Load secrets/config (must exist)
EnvironmentFile=/opt/example/adms/shared/.env.prod
Environment="DJANGO_SETTINGS_MODULE=settings"
Environment="HOME=/opt/example/adms/shared/home"
Environment="TMPDIR=/opt/example/adms/shared/tmp"
Environment="XDG_CACHE_HOME=/opt/example/adms/shared/.cache"

# Creates /run/adms and keeps it owned correctly
RuntimeDirectory=adms
RuntimeDirectoryMode=0750


# Gunicorn command (TCP bind example)
ExecStart=/opt/example/adms/venv/bin/gunicorn \
  v1.wsgi:application \
  --name adms \
  --bind 127.0.0.1:8000 \
  --workers 3 \
  --timeout 120 \
  --access-logfile - \
  --error-logfile -

# Explicitly no PID file (Type=simple)
PIDFile=

# Restart behavior
Restart=always
RestartSec=3

# Shutdown / safety
KillSignal=SIGQUIT
TimeoutStopSec=30

# Hardening
PrivateTmp=true
NoNewPrivileges=true

# Limits
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Reload systemd, then start and enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl start web-api-gunicorn.service
sudo systemctl enable web-api-gunicorn.service
sudo systemctl status web-api-gunicorn.service -l
```

Restart after a deployment:

```bash
sudo systemctl restart web-api-gunicorn.service
```

Tail logs:

```bash
sudo journalctl -u web-api-gunicorn.service -n 100 --no-pager
sudo journalctl -u web-api-gunicorn.service -f
```

---

## 11. Shared environment, keys, cache, temp folders

Create the shared key tree:

```bash
sudo mkdir -p /opt/example/web-api/shared/keys/prod
sudo chown -R portal:portal /opt/example/web-api/shared/keys
sudo chmod 750 /opt/example/web-api/shared/keys
sudo chmod 750 /opt/example/web-api/shared/keys/prod
```

Create runtime directories used by the application:

```bash
sudo mkdir -p /opt/example/web-api/shared/home
sudo mkdir -p /opt/example/web-api/shared/tmp
sudo mkdir -p /opt/example/web-api/shared/.cache

sudo chown -R portal:portal \
  /opt/example/web-api/shared/home \
  /opt/example/web-api/shared/tmp \
  /opt/example/web-api/shared/.cache

sudo chmod 700 \
  /opt/example/web-api/shared/home \
  /opt/example/web-api/shared/tmp \
  /opt/example/web-api/shared/.cache
```

---

## 12. AWS CLI on EC2

Install AWS CLI v2:

```bash
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt-get install -y unzip
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

Inspect the result of an SSM Run Command invocation:

```bash
aws ssm get-command-invocation \
  --region us-east-2 \
  --command-id COMMAND_ID \
  --instance-id INSTANCE_ID \
  --output json
```

---

## 13. EFS mount setup

Install the EFS / NFS client tools:

```bash
sudo apt update
sudo apt install -y amazon-efs-utils nfs-common
```

Create the mount point:

```bash
sudo mkdir -p /mnt/storage
```

Mount over NFSv4.1 (replace the file system DNS name with your own):

```bash
sudo mount -t nfs4 -o nfsvers=4.1 \
  fs-03117d0337a9c6d15.efs.us-east-2.amazonaws.com:/ /mnt/storage
```

Verify:

```bash
df -h
df -h | grep mnt
```

Confirm the EC2 region from instance metadata (IMDSv2):

```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/region
```

DNS sanity check:

```bash
nslookup fs-03117d0337a9c6d15.efs.us-east-2.amazonaws.com
```

---

## 14. Media folder on EFS

Create and own the media directory on EFS:

```bash
sudo mkdir -p /mnt/storage/media
sudo chown -R portal:portal /mnt/storage
sudo chown -R portal:portal /mnt/storage/media
sudo chmod 775 /mnt/storage/media
```

Expose it under `/var/www/media` via a symlink so application and Nginx configs stay path-stable:

```bash
sudo rm -rf /var/www/media
sudo ln -s /mnt/storage/media /var/www/media
```

Verify:

```bash
ls -ld /mnt/storage/media
ls -ld /var/www/media
sudo -u portal touch /var/www/media/test.txt
ls /mnt/storage/media
```

---

## 15. Auto-mount EFS after reboot

Add an entry to `/etc/fstab`:

```bash
sudo vim /etc/fstab
```

Test before rebooting:

```bash
sudo mount -a
df -h | grep mnt
```

> Do **not** reboot the instance until `mount -a` succeeds — a broken fstab entry can leave the host unbootable.

---

## 16. Migrating existing media

Sync media from a previous host (example):

```bash
sudo -u portal rsync -avz -e ssh \
  ecologix@192.168.193.13:/var/www/media/ /var/www/media/
```

Inspect what landed:

```bash
du -sh /mnt/storage/media
ls -al /mnt/storage/media
```

---

## 17. Media permissions for Nginx

Directories must be traversable; files must be readable:

```bash
sudo find /mnt/storage/media -type d -exec chmod 755 {} \;
sudo find /mnt/storage/media -type f -exec chmod 644 {} \;
```

Trace permissions along a full path when Nginx returns `403`:

```bash
namei -l /var/www/media/path/to/file.png
```

---

## 18. Troubleshooting commands

Disk and mounts:

```bash
df -h
df -h | grep mnt
du -sh /mnt/storage/media
```

Nginx:

```bash
sudo nginx -t
sudo service nginx status
sudo service nginx restart
```

Gunicorn:

```bash
sudo systemctl status web-api-gunicorn.service -l
sudo systemctl restart web-api-gunicorn.service
sudo journalctl -u web-api-gunicorn.service -n 100 --no-pager
sudo journalctl -u web-api-gunicorn.service -f
```

Inspect the unit file and reset failed state:

```bash
sudo systemctl cat web-api-gunicorn.service
sudo systemctl daemon-reload
sudo systemctl reset-failed web-api-gunicorn.service
sudo systemctl restart web-api-gunicorn.service
```

Quick local probes:

```bash
curl http://127.0.0.1:8000
curl -i http://127.0.0.1/health
```

---

## 19. Bootstrap checklist for a new EC2

```text
 1. Create private EC2 without a public IP
 2. Attach IAM role with AmazonSSMManagedInstanceCore
 3. Confirm private subnet route → NAT Gateway
 4. Confirm outbound HTTPS 443 is allowed
 5. Connect via SSM (not SSH)
 6. Install OS packages
 7. Create the portal system user
 8. Create /opt/example/web-api directory tree
 9. Install Python 3.10.8
10. Create the virtual environment
11. Add deploy.sh
12. Install and configure Nginx
13. Create the Gunicorn systemd service
14. Configure shared .env, keys, tmp, cache folders
15. Mount EFS at /mnt/storage
16. Symlink /var/www/media → /mnt/storage/media
17. Apply media permissions
18. Smoke-test Nginx, Gunicorn, and /health
19. Register the EC2 in the Load Balancer target group
20. Drive deployments through Jenkins + SSM Run Command
```

---

## 20. Production rules

```text
SSH is not required for private EC2.

Use:
  SSM              for access
  Jenkins          for deployment
  Load Balancer    for public traffic
  NAT Gateway      for outbound internet
  EFS              for shared media
  Nginx + Gunicorn for serving Django
```

Additional guidance:

- Keep only the NAT Gateway and Load Balancer in public subnets.
- Keep all backend EC2 in private subnets, with no public IPs.
- Prefer SSM Session Manager to SSH for everything interactive.
- Allow outbound `443` from private subnets so SSM and package mirrors work.
- Attach the correct IAM role **before** the first SSM connection attempt.
- Drive every deployment through Jenkins + SSM Run Command — never through long-lived SSH credentials.

---

## Author

**Arun Prasher** — Backend & DevOps Engineer

Designed and operated this pattern in production for Django services on AWS.
If you find this useful, a ⭐ on the repository is appreciated.
