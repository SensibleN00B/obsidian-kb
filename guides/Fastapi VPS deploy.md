# <img src="https://img.icons8.com/?id=13441&format=png&size=48" width="48" height="48" alt="Python" style="vertical-align: middle;"/> Deploying a FastAPI App on a VPS with Nginx and SSL

This guide provides a comprehensive, step-by-step process for deploying a production-ready **FastAPI** application on a Linux VPS (Ubuntu/Debian).

It covers:
- Setting up **Gunicorn** as a process manager.
- Configuring **Nginx** as a reverse proxy for web traffic and static files.
- Securing the application with a free **SSL certificate** from Let's Encrypt.

---

> [!CHECK] Prerequisites
> 1. **VPS**: A server running Ubuntu or Debian with root or sudo access.
> 2. **Domain Name**: A registered domain pointing to your VPS's public IP (both `@` and `www` records).
> 3. **FastAPI App**: Your application code ready for deployment (preferably in a Git repo).
>    - Should have a main file (e.g., `main.py`) with an app instance (`app = FastAPI()`).
> 4. **Static Files**: If serving static assets (CSS, JS, images), ensure they are in a single directory (e.g., `static/`).

---

## Step 1: Initial Server Setup

First, connect to your server to perform basic setup and security hardening.

```bash
# Connect to your server as root
ssh root@YOUR_SERVER_IP
```

### Create a New User
Avoid using `root` for daily tasks. Create a new user with sudo privileges.

```bash
# Create a new user (follow prompts for password)
adduser your_user

# Grant sudo (administrative) privileges
usermod -aG sudo your_user
```

### Configure Firewall
Set up a basic firewall to allow only essential traffic.

```bash
# Allow OpenSSH
sudo ufw allow OpenSSH

# Allow Nginx traffic (HTTP and HTTPS)
sudo ufw allow 'Nginx Full'

# Enable the firewall
sudo ufw enable
```

> [!IMPORTANT]
> Log out of `root` and log back in with your new user before proceeding.
> ```bash
> ssh your_user@YOUR_SERVER_IP
> ```

---

## Step 2: Install Python & Virtual Environment

Install Python, `pip`, and the `venv` module.

```bash
sudo apt update
sudo apt install python3-pip python3-dev python3-venv
```

### Option A: Standard Setup (venv)

Create a directory for your project and set up the virtual environment using the standard library.

```bash
# Create project directory
mkdir ~/myproject
cd ~/myproject

# Create virtual environment
python3 -m venv venv

# Activate virtual environment
source venv/bin/activate
```

### Option B: Faster Setup with `uv` (Recommended) ⚡

[uv](https://github.com/astral-sh/uv) is an extremely fast Python package installer and resolver, written in Rust.

**1. Install `uv`:**
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.cargo/env
```

**2. Initialize project:**
```bash
# Create project directory
mkdir ~/myproject
cd ~/myproject

# Create virtual environment (instantly)
uv venv
```

**3. Activate it:**
```bash
source .venv/bin/activate
```
*Note: `uv` creates a hidden `.venv` directory by default.*

---

## Step 3: Get Your Application Code

The best way to deploy code is by cloning from a Git repository.

```bash
# Install Git
sudo apt install git

# Clone your repository (current directory '.')
git clone https://github.com/your_username/your_project.git .
```

Install dependencies (ensure you have a `requirements.txt`).

**Standard:**
```bash
pip install -r requirements.txt
```

**With `uv`:**
```bash
uv pip install -r requirements.txt
```

---

## Step 4: Install & Configure Gunicorn

We will use **Gunicorn** as a process manager to run multiple worker processes of **Uvicorn**.

**Standard:**
```bash
pip install gunicorn uvicorn
```

**With `uv`:**
```bash
uv pip install gunicorn uvicorn
```

Test running the app with Gunicorn from your project directory:

```bash
gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app -b 0.0.0.0:8000
```

**Breakdown of flags:**
- `-w 4`: Specifies 4 worker processes (Rule of thumb: `2 * num_cores + 1`).
- `-k uvicorn.workers.UvicornWorker`: Tells Gunicorn to use Uvicorn workers.
- `main:app`: `main.py` file and `app` instance.
- `-b 0.0.0.0:8000`: Binds to all interfaces on port 8000.

> [!TIP]
> You can verify it's working by visiting `http://YOUR_SERVER_IP:8000`.
> Press `Ctrl+C` to stop it when done testing.

---

## Step 5: Create a Systemd Service

To ensure your app runs in the background and restarts automatically, create a systemd service file.

```bash
sudo nano /etc/systemd/system/fastapi.service
```

Paste the following configuration (update `User`, `Group`, and paths):

```ini
[Unit]
Description=Gunicorn instance to serve FastAPI application
After=network.target

[Service]
User=your_user
Group=your_user
WorkingDirectory=/home/your_user/myproject
ExecStart=/home/your_user/myproject/venv/bin/gunicorn -w 4 -k uvicorn.workers.UvicornWorker main:app -b 0.0.0.0:8000
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

**Start and enable the service:**

```bash
# Start the service
sudo systemctl start fastapi

# Enable on boot
sudo systemctl enable fastapi

# Check status
sudo systemctl status fastapi
```

---

## Step 6: Configure Nginx as Reverse Proxy

Nginx will handle internet requests, forward them to Gunicorn, and serve static files efficiently.

```bash
sudo apt install nginx
```

Create a server block for your domain:

```bash
sudo nano /etc/nginx/sites-available/your_domain
```

Paste the configuration below (update `server_name` and `alias` path):

```nginx
server {
    listen 80;
    server_name your_domain www.your_domain;

    # Serve static files directly
    location /static/ {
        alias /home/your_user/myproject/static/;
    }

    # Reverse proxy to Gunicorn
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site and restart Nginx:

```bash
# Create symbolic link
sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled

# Test configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

---

## Step 7: Secure with SSL (Let's Encrypt)

Secure your site with HTTPS using **Certbot**.

```bash
# Install Certbot and Nginx plugin
sudo apt install certbot python3-certbot-nginx

# Obtain certificate (follow prompts)
sudo certbot --nginx -d your_domain -d www.your_domain
```

*Select "Redirect" when asked to redirect HTTP traffic to HTTPS.*

**Test Auto-Renewal:**
```bash
sudo certbot renew --dry-run
```

---

## Step 8: Verification

> [!SUCCESS] Done!
> Visit `https://your_domain` in your browser.
> 
> You should see your FastAPI application served securely over HTTPS 🔒.

---

## Step 9: Managing Environment Variables

For production, never hardcode secrets (database URLs, API keys) in your code. Use a `.env` file.

1. **Create a `.env` file** in your project directory:
   ```bash
   nano ~/myproject/.env
   ```
   
   Add your variables:
   ```ini
   DATABASE_URL=postgresql://user:password@localhost/dbname
   SECRET_KEY=your_super_secret_key
   DEBUG=False
   ```

2. **Update the Systemd Service** to load this file.
   ```bash
   sudo nano /etc/systemd/system/fastapi.service
   ```

   Add `EnvironmentFile` under `[Service]`:
   ```ini
   [Service]
   ...
   EnvironmentFile=/home/your_user/myproject/.env
   ExecStart=...
   ```

3. **Reload systemd and restart the service:**
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart fastapi
   ```

---

## Step 10: Maintenance & Monitoring

### 🔄 Updating Your App
To deploy new code, follow these steps:

```bash
cd ~/myproject

# 1. Pull changes
git pull origin main

# 2. Install new dependencies (if any)
# Standard:
source venv/bin/activate && pip install -r requirements.txt
# OR with uv:
uv pip install -r requirements.txt

# 3. Restart the service to apply changes
sudo systemctl restart fastapi
```

### 📊 Viewing Logs
If something goes wrong, check the logs.

**Application Logs (Gunicorn/FastAPI):**
```bash
# View last 50 lines and follow live updates
sudo journalctl -u fastapi -n 50 -f
```

**Nginx Logs (Access & Errors):**
```bash
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

---

## Troubleshooting

- **502 Bad Gateway:** Usually means Gunicorn isn't running. Check status:
  ```bash
  sudo systemctl status fastapi
  ```
- **Permission Denied:** Ensure `your_user` owns the files:
  ```bash
  sudo chown -R your_user:your_user ~/myproject
  ```
- **Static Files 404:** Check the path in Nginx config matches the actual path on disk.