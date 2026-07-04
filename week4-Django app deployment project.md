# AWS Solutions Architect — Week 3: Production Deployment of a Django Application

A detailed, step-by-step runbook covering the full journey of the Week 3 capstone project: deploying a Django application on a single Ubuntu server with Gunicorn and Nginx, securing it locally with a self-managed SSL certificate, then evolving that setup into a production-grade, horizontally scalable AWS architecture using an Application Load Balancer, AWS Certificate Manager, an Auto Scaling Group, a Launch Template, and a custom domain managed through Hostinger + Route 53.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Project Overview & Architecture](#2-project-overview--architecture)
3. [Application Setup — Django on Ubuntu 22.04](#3-application-setup--django-on-ubuntu-2204)
4. [Gunicorn — Application Server](#4-gunicorn--application-server)
5. [Nginx — Reverse Proxy](#5-nginx--reverse-proxy)
6. [Local SSL with Certbot](#6-local-ssl-with-certbot)
7. [Building a Launch Template (AMI-Based)](#7-building-a-launch-template-ami-based)
8. [Auto Scaling Group](#8-auto-scaling-group)
9. [Application Load Balancer & ACM SSL](#9-application-load-balancer--acm-ssl)
10. [DNS Management via Route 53](#10-dns-management-via-route-53-domain-purchased-at-hostinger)
11. [Security & Secrets Management](#11-security--secrets-management)
12. [Cost Estimate & Teardown](#12-cost-estimate--teardown)
13. [End-to-End Validation Checklist](#13-end-to-end-validation-checklist)
14. [Limitations & Next Steps](#14-limitations--next-steps)

---

## 1. Prerequisites

Before starting, make sure you have:

**Access & tooling**
- An AWS account with an IAM user/role granted permissions for: EC2 (instances, AMIs, Launch Templates, Security Groups), Auto Scaling, Elastic Load Balancing (ALB, Target Groups), Certificate Manager (ACM), and Route 53.
- AWS CLI installed and configured (`aws configure`) if you intend to run any step from the terminal rather than the console.
- SSH access configured (a key pair) to reach EC2 instances for setup and troubleshooting.

**Networking**
- An existing VPC with at least **two public subnets in two different Availability Zones** — required by both the ALB and the Auto Scaling Group for fault tolerance.

**Domain**
- A registered domain. This project uses one purchased via Hostinger, but the process is identical for any registrar — DNS is delegated to Route 53 regardless of where the domain was bought (see Section 10).

**Application**
- A Django project with a `requirements.txt` file, ready to run under Gunicorn (this runbook uses [sunilkumar0633/social_media](https://github.com/sunilkumar0633/social_media) as the example app).

**Knowledge**
- Basic comfort with Ubuntu 22.04, `systemd` units, Nginx configuration syntax, and the AWS EC2/ALB/Route 53 consoles.

---

## 2. Project Overview & Architecture

This project took a single-server Django deployment and evolved it, in stages, into a fault-tolerant, auto-scaling production architecture.

**Stage 1 (single instance):** Client → Nginx (full reverse proxy: static/media serving + `proxy_pass` to Gunicorn's Unix socket, port 80/443) → Gunicorn (Unix socket) → Django, with SSL terminated locally on the instance via Certbot/Let's Encrypt.

**Nginx's role across stages:** the same Nginx reverse-proxy config from Stage 1 (listening on port 80, proxying over the Unix socket to Gunicorn) carries unchanged into the AMI used by the Launch Template — it is **not** stripped down or replaced. The only thing that changes for production is *where SSL terminates*: instead of Certbot's local certificate on the instance, SSL now terminates at the ALB using an ACM-issued certificate, and the ALB forwards plain HTTP traffic to Nginx on port 80 on each instance.

**Stage 2 (production architecture):**

```
User Browser
      │
      ▼
Route 53 (hosted zone for the Hostinger-purchased domain, Alias record → ALB)
      │
      ▼
Application Load Balancer (HTTPS:443, cert issued/managed by ACM)
      │
      ▼
Auto Scaling Group (EC2 instances launched from a Launch Template, registered in the ALB's target group)
      │
      ▼
Nginx (reverse proxy on port 80, same config as Stage 1 — no SSL block here anymore)
      │
      ▼
Gunicorn (WSGI server, Unix socket)
      │
      ▼
Django Application
```

*A rendered version of this diagram (draw.io or the AWS icon set) plus deployment screenshots — healthy target group, issued ACM certificate, ASG instance list — belong in a `/docs/screenshots` folder alongside this file; see Section 14.*

**Key architectural shift:** in Stage 1, SSL terminates on the instance itself via Certbot. In Stage 2, SSL terminates at the **ALB** using an **ACM-issued certificate**; Nginx keeps running exactly as before but only ever receives plain HTTP on port 80 from the ALB — standard practice, since ACM certificates can't be installed directly on EC2 instances, and centralizing SSL termination at the load balancer removes per-instance certificate management entirely.

---

## 3. Application Setup — Django on Ubuntu 22.04

### 3.1 Install required packages

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv nginx git
```

### 3.2 Clone the repository and set up a clean virtual environment

```bash
cd /dj/social_media
git clone https://github.com/sunilkumar0633/social_media.git

# If a corrupted or stale venv already exists, remove it first:
rm -rf venv

python3 -m venv venv
source venv/bin/activate
```

Activating the venv isolates this project's Python packages from the system-wide Python installation, so dependency versions don't conflict with other projects or the OS itself.

### 3.3 Install Python dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

### 3.4 Run migrations and collect static files

```bash
python manage.py migrate
python manage.py collectstatic --noinput
```

### 3.5 Configure and test the application

```bash
vim social/settings.py   # add server IP/domain to ALLOWED_HOSTS
```

**Why `ALLOWED_HOSTS` matters — this is request validation, not boilerplate.** Every incoming HTTP request carries a `Host` header, and Django checks it against `ALLOWED_HOSTS` before doing anything else with the request. If the header isn't on the list, Django rejects it outright with a `400 Bad Request`. Leaving this empty (in debug mode) or set to `'*'` removes that check entirely, which matters because a forged `Host` header can be used to poison password-reset emails, cache keys, or any URL the app builds from `request.get_host()` — the attacker's chosen hostname ends up embedded in output the app trusts as its own. Only add the exact hostnames you intend to serve (e.g. `example.com`, `www.example.com`, and the instance's IP only if you're testing directly against it).

> ⚠️ **Secrets note:** this is also where `SECRET_KEY` and database credentials typically live in `settings.py`. Editing them directly here is fine for a local sanity check, but **do not commit real values to git or bake real secrets into a public AMI.** See Section 11 for the recommended approach.

> ⚠️ **Proxy/SSL note (needed once the ALB is in front — Section 9):** the ALB terminates HTTPS and forwards plain HTTP to Nginx, so Django has no way to know a request originally arrived over HTTPS unless told. Without this, `request.is_secure()` returns `False`, which can silently break CSRF checks and secure-cookie behavior. Add to `settings.py`:
> ```python
> SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
> SESSION_COOKIE_SECURE = True
> CSRF_COOKIE_SECURE = True
> ```
> This is safe to add even before the ALB exists — it only takes effect when the `X-Forwarded-Proto` header is actually present, which Nginx/the ALB set automatically.

```bash
python manage.py runserver 0.0.0.0:8000
```

Verify in a browser at `http://your-server-ip:8000`, then stop the dev server (`Ctrl + C`) — it's single-threaded and not hardened for production traffic, which is exactly why Gunicorn is introduced next.

---

## 4. Gunicorn — Application Server

### 4.1 Install and manually test Gunicorn

```bash
pip install gunicorn
gunicorn --bind 0.0.0.0:8000 social.wsgi
```

Confirm the app responds the same way it did under `runserver`, then stop it (`Ctrl + C`).

### 4.2 Create a `systemd` service for Gunicorn

```bash
sudo vim /etc/systemd/system/gunicorn.service
```

```ini
[Unit]
Description=Gunicorn daemon for Django app
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/dj/social_media
ExecStart=/dj/social_media/venv/bin/gunicorn --workers 3 --bind unix:/dj/social_media/social.sock social.wsgi:application

[Install]
WantedBy=multi-user.target
```

- `User`/`Group=www-data` runs Gunicorn as the standard low-privilege web server user rather than root.
- Binding to a **Unix socket** instead of a TCP port is faster for local-only communication and avoids exposing Gunicorn on a network port — only Nginx (on the same machine) can reach it.
- `--workers 3` follows the common starting formula `(2 × CPU cores) + 1`.

```bash
sudo systemctl daemon-reload
sudo systemctl enable gunicorn
sudo systemctl start gunicorn
sudo systemctl status gunicorn
ls -l /dj/social_media/social.sock   # confirm the socket was created
```

---

## 5. Nginx — Reverse Proxy

### 5.1 Create the site configuration

**Permissions note before you start:** every command in this section needs root (`sudo`), because binding to port 80/443 requires elevated privilege — ports below 1024 are "privileged ports" on Linux, and only root (or a process explicitly granted `CAP_NET_BIND_SERVICE`) can bind to them. This is exactly why Nginx's architecture is split in two: its **master process** starts as root purely to grab port 80/443, then immediately spawns **worker processes** running as the unprivileged `www-data` user to actually handle requests. Gunicorn, by contrast, never touches a privileged port at all — it only binds to a local Unix socket (Section 4.2) — which is why the `gunicorn.service` file runs entirely as `www-data` with no root involvement whatsoever.

```bash
sudo vim /etc/nginx/sites-available/social_media
```

```nginx
server {
    listen 80;
    server_name www.example.com;  # replace with your domain

    location / {
        include proxy_params;
        proxy_pass http://unix:/dj/social_media/social.sock;
    }

    location /static/ {
        root /dj/social_media;
    }

    location /media/ {
        root /dj/social_media;
    }
}
```

- `proxy_pass` forwards dynamic requests over the Unix socket to Gunicorn.
- The `/static/` and `/media/` blocks let Nginx serve those files **directly from disk**, bypassing Gunicorn/Django — far more efficient than serving static assets through the Python app server.

### 5.2 Enable the site and validate

```bash
sudo ln -s /etc/nginx/sites-available/social_media /etc/nginx/sites-enabled/
sudo nginx -t                     # check syntax before reloading
sudo systemctl restart nginx
sudo systemctl enable nginx
```

Confirm the app is reachable over plain HTTP at `http://your-server-ip` (port 80, not the dev port 8000).

### 5.3 Note: this same Nginx config carries into the production (AMI/ALB) setup

The config above is used in **both** stages, unchanged. When the project moved to building an AMI for the Auto Scaling Group (Section 7), Nginx and this exact site config were included as-is — there was no separate "production" config. The only difference in production is that Nginx only ever receives plain HTTP on port 80 from the ALB, since the ALB (not Certbot) now handles SSL termination (Section 9).

---

## 6. Local SSL with Certbot

This step secures the single instance directly, as the starting point before production moved to load-balancer-terminated SSL.

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d your_domain
```

Certbot prompts for an email (renewal/expiry notices) and asks whether to redirect all HTTP → HTTPS — choose "redirect" to enforce SSL for every visitor.

```bash
sudo systemctl restart gunicorn
sudo systemctl restart nginx
sudo certbot certificates          # confirm the cert is active
sudo systemctl status certbot.timer  # confirm auto-renewal is scheduled (certs expire every 90 days)
```

This instance-level certificate was later replaced by **ACM + ALB** (Section 9) so SSL termination and renewal are managed centrally rather than per-instance — essential once multiple instances exist behind an Auto Scaling Group, since you don't want a separate Certbot renewal running on every one of them.

---

## 7. Building a Launch Template (AMI-Based)

To scale beyond a single hand-configured server, the working instance was converted into a reusable template the Auto Scaling Group launches identical copies from.

### 7.1 Create the AMI

With the fully configured instance running (Django + venv + Gunicorn + Nginx all working):
- EC2 Console → select the instance → **Actions ▸ Image and templates ▸ Create image**
- Name it (e.g., `social-media-app-v1`), add a description
- Click **Create image** and wait for it to reach `available`

> ⚠️ Make sure no real secrets are baked into this AMI (see Section 11) — an AMI can be shared or copied, and anything in `settings.py` at the time of capture goes with it.

### 7.2 Create the Launch Template

- **EC2 ▸ Launch Templates ▸ Create launch template**, name it (e.g., `social-media-lt`)
- Under **Application and OS Images**, select the custom AMI from 7.1
- Choose an **Instance type** (e.g., `t3.micro`)
- Attach the security group(s) defined below
- Under **Advanced details**, add a user data script so services start on boot even if not already enabled in the AMI:

```bash
#!/bin/bash
systemctl start gunicorn
systemctl start nginx
```

> This script only **starts existing services** — it does not pull the latest application code or run migrations on boot. Deploying an app update currently means: update code on an instance → rebuild the AMI → update the Launch Template → refresh the ASG instances. See Section 14.

### 7.3 Security group definitions

| Security Group | Direction | Port | Source/Destination | Purpose |
|---|---|---|---|---|
| **ALB-SG** | Inbound | 80 | `0.0.0.0/0` | Accept HTTP from the internet (redirected to HTTPS) |
| **ALB-SG** | Inbound | 443 | `0.0.0.0/0` | Accept HTTPS from the internet |
| **EC2-SG** | Inbound | 80 | **ALB-SG** (by reference, not `0.0.0.0/0`) | Accept app traffic only from the load balancer |
| **EC2-SG** | Inbound | 22 | Your IP `/32` | SSH for management — never left open to `0.0.0.0/0` |
| **EC2-SG** | Outbound | All | `0.0.0.0/0` | Allow updates, package installs, outbound calls |

The EC2 instances should **never** accept inbound traffic directly from the internet on port 80/443 — all traffic must pass through the ALB first. Referencing the ALB's security group ID as the source (rather than a CIDR range) is what enforces this at the network level.

Save the Launch Template — it is now the single source of truth the Auto Scaling Group uses to launch every instance, guaranteeing the whole fleet is configured identically.

---

## 8. Auto Scaling Group

1. **EC2 ▸ Auto Scaling Groups ▸ Create Auto Scaling group.**
2. Name it (e.g., `social-media-asg`), select the Launch Template from Section 7.
3. Choose the VPC and select subnets across **at least two Availability Zones** — this is what gives the architecture fault tolerance.
4. Under **Load balancing**, attach the target group created for the ALB (Section 9) so the ASG registers/deregisters instances automatically as it scales.
5. Enable **ELB health checks** in addition to default EC2 health checks, so the ASG replaces instances that are running but failing to serve traffic (e.g., Nginx crashed), not just instances that are fully down.
6. Configure group size — e.g.:
   - Desired: `2`
   - Minimum: `2`
   - Maximum: `4`
7. Add a scaling policy — e.g., target tracking on **Average CPU utilization**, target `60%`. This launches additional instances when load increases and terminates extras when load drops, within the min/max bounds above.
8. Create the group and confirm in the console that the desired number of instances launches and each registers as **healthy** in the target group.

---

## 9. Application Load Balancer & ACM SSL

### 9.1 Request a certificate in ACM

1. **Certificate Manager ▸ Request a certificate ▸ Request a public certificate.**
2. Enter the domain(s) to secure (e.g., `example.com` and `www.example.com`).
3. Choose **DNS validation** (enables automatic renewal, unlike email validation).
4. ACM generates a CNAME record (name + value) that must be added at the DNS provider to prove ownership — added in Route 53 (Section 10.2).
5. Once the CNAME propagates, ACM automatically moves the certificate to **Issued**.

### 9.2 Create the Application Load Balancer

1. **EC2 ▸ Load Balancers ▸ Create Load Balancer ▸ Application Load Balancer.**
2. Name it (e.g., `social-media-alb`), scheme **Internet-facing**, same VPC and ≥2 subnets used by the ASG.
3. Listeners:
   - **HTTP:80** — redirects to HTTPS (step 5)
   - **HTTPS:443** — select the ACM certificate from 9.1
4. Create/select a **Target Group** (type: Instances, protocol HTTP, port **80** — matching Nginx). This is the same target group the ASG registers into.
5. Set the HTTP:80 listener's default action to **redirect** to HTTPS:443.
6. Configure the target group's health check path (e.g., `/`).

   > ⚠️ **Recommended improvement:** point this at a dedicated `/healthz/` view instead of `/`, if your app's root view renders a full template or queries the database. A minimal health check should do neither — it should return a bare `200 OK` as fast as possible, so the ALB's health assessment reflects whether the instance can serve traffic, not whether a specific page happens to render correctly at that moment. A one-line Django view is enough:
   > ```python
   > from django.http import HttpResponse
   > def healthz(request):
   >     return HttpResponse("OK")
   > ```
7. Create the load balancer and wait for **Active** state.
8. Note the ALB's DNS name (e.g., `social-media-alb-1234567890.us-east-1.elb.amazonaws.com`) — used for the domain's DNS record in Section 10.

---

## 10. DNS Management via Route 53 (Domain Purchased at Hostinger)

The domain was purchased through **Hostinger**, but DNS records are managed centrally in **Route 53** rather than Hostinger's own panel — the standard pattern when a domain is bought from a third-party registrar but the infrastructure lives in AWS, since Route 53 supports Alias records pointing at an ALB that most registrar DNS panels don't.

### 10.1 Delegate the domain to Route 53

1. **Route 53 ▸ Hosted zones ▸ Create hosted zone.** Enter the domain, type **Public hosted zone**.
2. Route 53 generates four **NS records**. Copy them.
3. In Hostinger's domain panel, find **Nameservers** and replace the defaults with the four Route 53 nameservers.
4. Save. Confirm propagation (minutes to 24–48 hours):

```bash
dig NS example.com
```

### 10.2 Add the ACM validation record

1. Back in **Certificate Manager**, for a domain whose hosted zone already exists in Route 53, click **"Create records in Route 53"** to insert the CNAME validation record automatically.
2. Confirm it appears under **Route 53 ▸ Hosted zones ▸ example.com** (a CNAME like `_xxxxxxxx.example.com`).
3. Wait for ACM to detect it and move the certificate to **Issued** (typically a few minutes via Route 53).

### 10.3 Point the domain at the ALB using Alias records

1. **Create record** for the apex domain:
   - Name: blank · Type: **A** · Alias: **Yes** · Route to: Application Load Balancer → select `social-media-alb`
   - (Alias records solve the classic limitation that a plain CNAME can't be used on an apex domain.)
2. Repeat for `www`, same target.
3. Verify:

```bash
dig example.com
dig www.example.com
```

Both should resolve to the ALB's IP addresses.

---

## 11. Security & Secrets Management

A few security-hygiene points that apply across the whole project, consolidated here:

- **Never commit real secrets.** `SECRET_KEY`, database credentials, and any API keys should not be hardcoded into `settings.py` in a committed file or baked into a shared AMI. In production, source them from:
  - Environment variables (e.g., via `django-environ` or `python-decouple`), populated at boot by the user data script, **or**
  - **AWS Systems Manager Parameter Store** (SecureString) or **AWS Secrets Manager**, pulled at instance startup.
- **Least-privilege security groups.** As shown in Section 7.3, EC2 instances should only accept traffic from the ALB's security group, never `0.0.0.0/0` on ports 80/443. SSH should be scoped to a specific IP, not open to the internet.
- **Least-privilege IAM.** The IAM user/role used to build this (Section 1) should be scoped to only the services actually needed (EC2, ASG, ELB, ACM, Route 53) rather than `AdministratorAccess`, if this is anything beyond a personal training account.
- **Certificate renewal.** ACM certificates with DNS validation auto-renew — no action needed as long as the validation CNAME remains in the hosted zone. This is a deliberate advantage of ACM+ALB over the instance-level Certbot approach from Section 6, which requires the `certbot.timer` to keep running on every instance.

---

## 12. Cost Estimate & Teardown

This architecture runs continuously once deployed and incurs ongoing charges — an ALB, 2–4 EC2 instances, and a Route 53 hosted zone. Use the [AWS Pricing Calculator](https://calculator.aws) for exact figures in your region; rough order of magnitude:

| Resource | Approx. cost |
|---|---|
| Application Load Balancer | ~$16/month base + usage (LCU-hours) |
| EC2 instances (e.g., 2× `t3.micro`) | Varies by region/hours run |
| Route 53 hosted zone | ~$0.50/month + query volume |
| ACM certificate | Free when used with an ALB |

**Teardown order matters** — some resources block deletion of others:

1. Delete the **Auto Scaling Group** (terminates its EC2 instances).
2. Delete the **Application Load Balancer**.
3. Delete the associated **Target Group**.
4. Delete/deregister the **ACM certificate** (only after the ALB using it is gone).
5. Delete the **Route 53 records**, then the **hosted zone** last (it carries its own small monthly charge if left behind).
6. Deregister the custom **AMI** and delete its backing **EBS snapshot**, if created for this project.

---

## 13. End-to-End Validation Checklist

- [ ] `dig NS example.com` returns the four Route 53 nameservers.
- [ ] ACM certificate status shows **Issued**.
- [ ] `dig example.com` and `dig www.example.com` resolve to the ALB.
- [ ] All targets in the target group show **healthy**.
- [ ] `curl -I https://example.com` returns a `200`/`301` with a valid TLS handshake.
- [ ] Browser shows a valid padlock for the domain, with the full chain working: **Route 53 → ALB (SSL termination) → Auto Scaling Group → Nginx → Gunicorn → Django**.
- [ ] Manually terminate one instance in the EC2 console and confirm the ASG launches a replacement automatically, with the site remaining continuously reachable throughout — this is the fault-tolerance test the whole architecture was built for.

---

## 14. Limitations & Next Steps

This project was built manually through the AWS Console as a learning exercise — a deliberate choice at this stage, since clicking through each service builds a clearer mental model than a single `terraform apply` would. Known limitations and the path to close each:

1. **Not reproducible from code** — every resource was created via console clicks; re-running this by hand won't produce an identical environment.
2. **AMI is a point-in-time snapshot** — new instances run whatever code was baked in at AMI creation; there's no boot-time pull of the latest app code (see Section 7.2).
3. **No CI/CD** — deployments and AMI rebuilds are manual.
4. ~~No diagram image or screenshots committed yet~~ — resolved, see `docs/architecture.svg` and `screenshots/`.

Recommended next steps, in priority order: move secrets out of `settings.py` into Parameter Store/Secrets Manager (Section 11) → add a rendered architecture diagram and deployment screenshots → convert Sections 7–10 into Terraform modules (`networking`, `compute`, `load-balancer`, `dns`) → automate AMI builds with Packer → add a GitHub Actions pipeline running `terraform plan` on PRs.

---

*This document serves as a structured technical reference and portfolio summary of the Week 3 capstone project: taking a Django application from a single manually configured server to a production-grade, auto-scaling AWS architecture with managed SSL and DNS.*
