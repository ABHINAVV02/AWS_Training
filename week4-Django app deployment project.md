# AWS Solutions Architect — Week 3: Production Deployment of a Django Application

A detailed, step-by-step runbook covering the full journey of the Week 3 capstone project: deploying a Django application on a single Ubuntu server with Gunicorn and Nginx, securing it locally with a self-managed SSL certificate, then evolving that setup into a production-grade, horizontally scalable AWS architecture using an Application Load Balancer, AWS Certificate Manager, an Auto Scaling Group, a Launch Template, and a custom domain managed through Hostinger.

---

## Table of Contents

1. [Project Overview & Architecture](#1-project-overview--architecture)
2. [Application Setup — Django on Ubuntu 22.04](#2-application-setup--django-on-ubuntu-2204)
3. [Gunicorn — Application Server](#3-gunicorn--application-server)
4. [Nginx — Reverse Proxy](#4-nginx--reverse-proxy)
   - [4.3 Note on Nginx in Production](#43-note-this-same-nginx-config-carries-into-the-production-amialb-setup)
5. [Local SSL with Certbot](#5-local-ssl-with-certbot)
6. [Building a Launch Template (AMI-Based)](#6-building-a-launch-template-ami-based)
7. [Auto Scaling Group](#7-auto-scaling-group)
8. [Application Load Balancer & ACM SSL](#8-application-load-balancer--acm-ssl)
9. [DNS Management via Route 53](#9-dns-management-via-route-53-domain-purchased-at-hostinger)

---

## 1. Project Overview & Architecture

This project took a single-server Django deployment and evolved it, in stages, into a fault-tolerant, auto-scaling production architecture.

**Stage 1 (single instance):** Client → Nginx (full reverse proxy: static/media serving + `proxy_pass` to Gunicorn's Unix socket, port 80/443) → Gunicorn (Unix socket) → Django, with SSL terminated locally on the instance via Certbot/Let's Encrypt.

**Nginx's role across stages:** the same Nginx reverse-proxy config from Stage 1 (listening on port 80, proxying over the Unix socket to Gunicorn) was carried unchanged into the AMI used by the Launch Template — it was **not** stripped down or replaced. The only thing that changed for production is where SSL terminates: instead of Certbot's local certificate on the instance, SSL now terminates at the ALB using an ACM-issued certificate, and the ALB forwards plain HTTP traffic to Nginx on port 80 on each instance. This is also why the local Certbot certificate itself didn't need to carry over into the AMI — the Nginx *config* did, but the SSL-specific block Certbot added was not needed once the ALB took over certificate handling (see Section 8).

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

**Key architectural shift:** in Stage 1, SSL is terminated on the instance itself via Certbot, with Nginx proxying to Gunicorn's Unix socket on port 80/443. In Stage 2, SSL terminates at the **ALB** using an **ACM-issued certificate**; Nginx keeps running exactly as before but only ever receives plain HTTP on port 80 from the ALB now — this is standard practice, since ACM certificates can't be installed directly on EC2 instances, and terminating SSL at the load balancer removes that overhead from every instance individually.

---

## 2. Application Setup — Django on Ubuntu 22.04

### 2.1 Install required packages

**Steps:**
1. Update the system and install core dependencies:
   ```
   sudo apt update && sudo apt upgrade -y
   sudo apt install -y python3 python3-pip python3-venv nginx git
   ```

### 2.2 Clone the repository and set up a clean virtual environment

**Steps:**
1. Navigate to (or create) the project directory:
   ```
   cd /dj/social_media
   ```
2. Clone the application repository:
   ```
   git clone https://github.com/sunilkumar0633/social_media.git
   ```
3. If a corrupted or stale virtual environment already exists, remove it before recreating:
   ```
   rm -rf venv
   ```
4. Create a fresh virtual environment and activate it:
   ```
   python3 -m venv venv
   source venv/bin/activate
   ```
   Activating the venv isolates this project's Python packages from the system-wide Python installation, so dependency versions don't conflict with other projects or the OS itself.

### 2.3 Install Python dependencies

**Steps:**
1. Upgrade `pip` inside the venv, then install the project's requirements:
   ```
   pip install --upgrade pip
   pip install -r requirements.txt
   ```

### 2.4 Run migrations and collect static files

**Steps:**
1. Apply database migrations to build/update the schema:
   ```
   python manage.py migrate
   ```
2. Collect all static assets (CSS/JS/images) into a single directory that Nginx will serve directly:
   ```
   python manage.py collectstatic --noinput
   ```

### 2.5 Configure and test the application

**Steps:**
1. Edit `settings.py` to add the server's IP or domain to `ALLOWED_HOSTS` (Django rejects requests for unrecognized hosts by default, as a security measure):
   ```
   vim social/settings.py
   ```
2. Run Django's built-in development server to sanity-check the app before wiring up Gunicorn/Nginx:
   ```
   python manage.py runserver 0.0.0.0:8000
   ```
3. Verify it's reachable in a browser:
   ```
   http://your-server-ip:8000
   ```
4. Stop the dev server (`Ctrl + C`) once confirmed — it is **not** suitable for production; it's single-threaded and not hardened for real traffic, which is exactly why Gunicorn is introduced next.

---

## 3. Gunicorn — Application Server

### 3.1 Install and manually test Gunicorn

**Steps:**
1. Install Gunicorn inside the venv:
   ```
   pip install gunicorn
   ```
2. Do a manual test run, bound directly to a TCP port:
   ```
   gunicorn --bind 0.0.0.0:8000 social.wsgi
   ```
3. Confirm the app responds the same way it did under `runserver`, then stop it (`Ctrl + C`) — the next step wires Gunicorn into `systemd` so it runs persistently in the background instead of tying up a terminal session.

### 3.2 Create a `systemd` service for Gunicorn

**Steps:**
1. Create the unit file:
   ```
   sudo vim /etc/systemd/system/gunicorn.service
   ```
2. Paste in the service definition:
   ```
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
   - Binding to a **Unix socket** (`unix:/dj/social_media/social.sock`) instead of a TCP port is faster than TCP for local-only communication and avoids exposing Gunicorn on a network port at all — only Nginx (on the same machine) can reach it.
   - `--workers 3` runs three worker processes to handle concurrent requests; a common starting formula is `(2 × CPU cores) + 1`.
3. Reload `systemd` so it picks up the new unit file, then enable and start the service:
   ```
   sudo systemctl daemon-reload
   sudo systemctl enable gunicorn
   sudo systemctl start gunicorn
   sudo systemctl status gunicorn
   ```
4. Confirm the socket file was created:
   ```
   ls -l /dj/social_media/social.sock
   ```

---

## 4. Nginx — Reverse Proxy

### 4.1 Create the site configuration

**Steps:**
1. Create a new Nginx site config:
   ```
   sudo vim /etc/nginx/sites-available/social_media
   ```
2. Add the reverse proxy configuration:
   ```
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
   - `proxy_pass` forwards dynamic requests over the Unix socket straight to Gunicorn.
   - The `/static/` and `/media/` blocks let Nginx serve those files **directly from disk**, bypassing Gunicorn/Django entirely — static file serving is far more efficient handled by Nginx than by the Python application server.
3. Save and exit `vim` (`Esc`, then `:wq`).

### 4.2 Enable the site and validate

**Steps:**
1. Create a symbolic link from `sites-available` into `sites-enabled` (Nginx only loads configs that are linked into `sites-enabled`):
   ```
   sudo ln -s /etc/nginx/sites-available/social_media /etc/nginx/sites-enabled/
   ```
2. Test the configuration for syntax errors **before** reloading Nginx — this prevents a bad config from taking the web server down:
   ```
   sudo nginx -t
   ```
3. Restart Nginx to apply the new site, and enable it to start on boot:
   ```
   sudo systemctl restart nginx
   sudo systemctl enable nginx
   ```
4. Confirm the app is now reachable over plain HTTP through Nginx (port 80) rather than the dev port 8000:
   ```
   http://your-server-ip
   ```

### 4.3 Note: this same Nginx config carries into the production (AMI/ALB) setup

The config in 4.1–4.2 is used in **both** stages, unchanged. When the project moved to building an AMI for the Auto Scaling Group (Section 6), Nginx and this exact site config were included as-is in the image — there was no separate "production" Nginx config. The only difference in production is that Nginx now only ever receives plain HTTP on port 80 from the ALB, since the ALB (not Certbot) handles SSL termination — see Section 8.

---

## 5. Local SSL with Certbot

This stage secures the single instance directly, before the project moved to load-balancer-terminated SSL in production.

**Steps:**
1. Install Certbot and its Nginx plugin:
   ```
   sudo apt install certbot python3-certbot-nginx
   ```
2. Request and auto-install a certificate for the domain, letting Certbot edit the Nginx config automatically to add the `listen 443 ssl` block and redirect HTTP → HTTPS:
   ```
   sudo certbot --nginx -d your_domain
   ```
3. Certbot will prompt for an email (for renewal/expiry notices) and ask whether to redirect all HTTP traffic to HTTPS — choosing "redirect" enforces SSL for every visitor.
4. Restart the services to make sure everything picked up the new configuration cleanly:
   ```
   sudo systemctl restart gunicorn
   sudo systemctl restart nginx
   ```
5. Confirm the certificate is active by visiting the site over `https://` and checking the padlock, or via:
   ```
   sudo certbot certificates
   ```
6. Certbot installs a renewal timer automatically; confirm it's active (Let's Encrypt certs expire every 90 days):
   ```
   sudo systemctl status certbot.timer
   ```

This local-Certbot setup was the starting point. In production, this instance-level certificate was replaced by **ACM + ALB** (Section 8) so that SSL termination and renewal are managed centrally rather than per-instance — critical once multiple instances exist behind an Auto Scaling Group, since you don't want to manage a separate Certbot renewal on every instance.

---

## 6. Building a Launch Template (AMI-Based)

To scale beyond a single hand-configured server, the working instance was converted into a reusable template that the Auto Scaling Group could launch identical copies from.

**Steps:**
1. With the fully configured instance running (Django + venv + Gunicorn + Nginx all working), create an AMI (Amazon Machine Image) from it:
   - In the EC2 Console, select the instance → **Actions > Image and templates > Create image**
   - Give it a name (e.g., `social-media-app-v1`) and description
   - Click **Create image** and wait for it to reach the `available` state
2. Create a Launch Template based on that AMI:
   - Go to **EC2 > Launch Templates > Create launch template**
   - Name it (e.g., `social-media-lt`)
   - Under **Application and OS Images**, select the custom AMI created in step 1
   - Choose an **Instance type** (e.g., `t3.micro`)
   - Attach the appropriate **Security Group** (allowing inbound HTTP on port `80` from the ALB's security group only, and SSH from your IP for management)
   - Under **Advanced details**, add any needed **User data** script — for example, to ensure Gunicorn/Nginx are started on boot in case the AMI's services aren't already enabled:
     ```
     #!/bin/bash
     systemctl start gunicorn
     systemctl start nginx
     ```
3. Save the launch template. This template is now the single source of truth the Auto Scaling Group will use to launch every new instance, guaranteeing every instance in the fleet is configured identically.

---

## 7. Auto Scaling Group

**Steps:**
1. Go to **EC2 > Auto Scaling Groups > Create Auto Scaling group**.
2. Name the group (e.g., `social-media-asg`) and select the Launch Template created in Section 6.
3. Choose the VPC and select multiple **subnets across at least two Availability Zones** — this is what gives the architecture fault tolerance; if one AZ has an outage, the ASG can still serve traffic from instances in the other AZ.
4. Under **Load balancing**, attach the group to the **Target Group** that will be created for the Application Load Balancer (see Section 8) — this lets the ASG register/deregister instances with the load balancer automatically as they scale.
5. Enable **ELB health checks** in addition to the default EC2 health checks, so the ASG replaces instances that are running but failing to serve traffic correctly (e.g., Nginx crashed), not just instances that are fully down.
6. Configure group size:
   - **Desired capacity:** e.g., `2`
   - **Minimum capacity:** e.g., `2`
   - **Maximum capacity:** e.g., `4`
7. Add a **scaling policy** — for example, a target tracking policy on average CPU utilization:
   - Metric type: **Average CPU utilization**
   - Target value: e.g., `60%`
   This automatically launches additional instances from the Launch Template when load increases, and terminates extras when load drops, always keeping within the min/max bounds set above.
8. Review and create the Auto Scaling Group. Confirm in the console that the desired number of instances launches successfully and each one registers as **healthy** in the target group.

---

## 8. Application Load Balancer & ACM SSL

### 8.1 Request a certificate in AWS Certificate Manager (ACM)

**Steps:**
1. Go to **Certificate Manager > Request a certificate**.
2. Choose **Request a public certificate**.
3. Enter the domain name(s) to secure (e.g., `example.com` and `www.example.com`).
4. Choose **DNS validation** (recommended over email validation — it also enables automatic renewal with no manual steps later).
5. ACM generates a **CNAME record** (name + value) that must be added at the DNS provider to prove domain ownership — this record is added at Hostinger (see Section 9).
6. Once the CNAME is added and propagates, ACM automatically detects it and moves the certificate to **Issued** status.

### 8.2 Create the Application Load Balancer

**Steps:**
1. Go to **EC2 > Load Balancers > Create Load Balancer**, and choose **Application Load Balancer**.
2. Name it (e.g., `social-media-alb`), set scheme to **Internet-facing**, and select the same VPC and at least two subnets (across two AZs) used by the Auto Scaling Group.
3. Under **Listeners**, add:
   - **HTTP : 80** — used only to redirect to HTTPS (configured in step 5 below)
   - **HTTPS : 443** — select the ACM certificate issued in Section 8.1 from the certificate dropdown
4. Create (or select) a **Target Group** of type "Instances," protocol HTTP, **port 80** — matching Nginx's config (Section 4), which listens on 80 and proxies over the Unix socket to Gunicorn. This is the same target group the Auto Scaling Group registers its instances into.
5. Set the HTTP:80 listener's default action to **redirect** to HTTPS:443, so any plain HTTP request is automatically upgraded — this replaces the instance-level redirect that Certbot used to handle in Section 5.
6. Configure the target group's **health check path** (e.g., `/`) so the ALB can detect and route around unhealthy instances.
7. Create the load balancer and wait for its state to become **Active**.
8. Note the ALB's DNS name (e.g., `social-media-alb-1234567890.us-east-1.elb.amazonaws.com`) — this is the target used for the domain's DNS record in Section 9.

---

## 9. DNS Management via Route 53 (Domain Purchased at Hostinger)

The domain itself was purchased through **Hostinger**, but DNS records were managed centrally in **Amazon Route 53** rather than in Hostinger's own DNS panel. This is the standard pattern when a domain is bought from a third-party registrar but the infrastructure lives in AWS — it gives native support for AWS-specific record types (like Alias records pointing at an ALB) that most registrars' DNS panels don't offer.

### 9.1 Delegate the domain to Route 53

**Steps:**
1. In the Route 53 console, go to **Hosted zones > Create hosted zone**.
2. Enter the domain name (e.g., `example.com`) and set the type to **Public hosted zone**.
3. Create the hosted zone. Route 53 automatically generates an **NS (Name Server) record** listing four AWS name servers for this domain.
4. Copy those four name server values.
5. Log in to **Hostinger**, open the domain's management panel, and find the **Nameservers** section.
6. Replace Hostinger's default nameservers with the four Route 53 nameservers copied in step 4.
7. Save the change. This delegates authority for the domain's DNS resolution from Hostinger to Route 53 — from this point forward, all DNS records for the domain are managed in Route 53, not in Hostinger.
8. Confirm delegation has propagated (this can take anywhere from a few minutes up to 24–48 hours):
   ```
   dig NS example.com
   ```
   The output should list the same four AWS name servers configured in step 6.

### 9.2 Add the ACM validation record in Route 53

**Steps:**
1. Return to the **Certificate Manager** console for the certificate requested in Section 8.1.
2. For a certificate requested through ACM for a domain whose hosted zone already exists in Route 53, ACM offers a **"Create records in Route 53"** button directly in the console — click it to have ACM insert the required CNAME validation record automatically, rather than copying values manually.
3. Confirm the record was created by checking the hosted zone in Route 53:
   ```
   Route 53 > Hosted zones > example.com
   ```
   A new CNAME record with a name like `_xxxxxxxx.example.com` should now be listed.
4. Wait for ACM to detect the record and move the certificate to **Issued** status (this is typically fast — a few minutes — since Route 53 validation avoids third-party DNS propagation delays).

### 9.3 Point the domain at the ALB using an Alias record

**Steps:**
1. In the same Route 53 hosted zone, click **Create record**.
2. For the apex/root domain (`example.com`):
   - Record name: leave blank (represents the apex)
   - Record type: **A**
   - Toggle **Alias** to "Yes"
   - Route traffic to: **Alias to Application and Classic Load Balancer**
   - Select the correct Region, then select the ALB created in Section 8.2 (`social-media-alb`) from the dropdown
   - Route 53 **Alias records** solve the classic limitation that a plain CNAME can't be used on an apex domain — this is exactly why Route 53 was used instead of relying on Hostinger's own DNS panel, which may not support ALIAS/ANAME records for apex domains.
3. Repeat for the `www` subdomain:
   - Record name: `www`
   - Record type: **A**
   - Alias: **Yes**, pointing to the same ALB
4. Save both records.
5. Verify resolution:
   ```
   dig example.com
   dig www.example.com
   ```
   Both should resolve to the ALB's IP addresses (Alias records resolve at the DNS layer directly to the load balancer, unlike a CNAME which would require an extra lookup hop).
6. Test the full path end-to-end in a browser:
   ```
   https://example.com
   ```
   Confirm the padlock shows a valid ACM-issued certificate, and that the page loads through the full chain: **Route 53 → ALB (SSL termination) → Auto Scaling Group → Nginx → Gunicorn → Django**.
7. As a final validation step, terminate one instance manually from the EC2 console and confirm the Auto Scaling Group launches a replacement automatically from the Launch Template, and that the site remains continuously reachable throughout — confirming the fault tolerance the whole architecture was built for.

---

*This document serves as a structured technical reference and portfolio summary of the Week 3 capstone project: taking a Django application from a single manually configured server to a production-grade, auto-scaling AWS architecture with managed SSL and DNS.*
