---
title: Install and Configure Kazoo-Applications
layout: default
parent: Community Guides
---

# Kazoo Application Server Manual Deployment Guide

This guide provides manual instructions for deploying and configuring a Kazoo application server without using Ansible automation.

## Overview

Kazoo is a cloud-based telecommunications platform that provides carrier-grade VoIP services. The application server component is responsible for handling the core business logic, APIs, and service orchestration.

## Prerequisites

- RHEL 8 or CentOS 8 server
- Minimum 4GB RAM
- 20GB+ free disk space
- Network connectivity to all dependent services
- Already configured database server (CouchDB)
- Already configured SBC server (Kamailio)
- Already configured Media server (Freeswitch)

## 1. System Preparation

### Update the System

```bash
sudo yum update -y
sudo yum install -y epel-release
sudo yum install -y wget git nano net-tools curl gnupg dnf-utils
```

### Set Timezone

It's recommended to set all your servers to UTC for better log tracing/event correlation.

```bash
sudo timedatectl set-timezone UTC
```

### Configure Hostname

```bash
# Replace app.example.com with your actual hostname
sudo hostnamectl set-hostname app.example.com

# Verify hostname
hostname -f
```

### Update /etc/hosts

You don't want your cluster/infrastructure going down just because of a DNS failure. It's recommended to manually add all your hosts to /etc/hosts to remove the reliance on DNS resolution servers.

Add entries for your Kazoo infrastructure:

```bash
sudo nano /etc/hosts
```

Add lines for all servers in your infrastructure:

```
# Example (replace with your actual IPs and hostnames)
127.0.0.1 localhost
x.x.x.x app.example.com app1
x.x.x.x db.example.com db1
x.x.x.x sbc.example.com sbc1
x.x.x.x media.example.com media1
```

## 2. Installing Required Packages

### Download Kazoo Packages

It is recommended to build from source or download the latest release from https://github.com/kazoo-classic/kazoo/releases

```bash
# Create a directory for the RPM packages
mkdir -p /opt/rpm_installs

# Download the packages from GitHub
wget -P /opt/rpm_installs https://github.com/kazoo-classic/kazoo/releases/download/v4.3-1-alpha/erlang-otp-19.3-1.el8.x86_64.rpm
wget -P /opt/rpm_installs https://github.com/kazoo-classic/kazoo/releases/download/v4.3-1-alpha/rebar-2.6.4-1.el8.x86_64.rpm
wget -P /opt/rpm_installs https://github.com/kazoo-classic/kazoo/releases/download/v4.3-1-alpha/elixir-1.5.3-1.el8.x86_64.rpm
wget -P /opt/rpm_installs https://github.com/kazoo-classic/kazoo/releases/download/v4.3-1-alpha/kazoo-classic-4.3-1.el8.x86_64.rpm
```

### Install Packages in Order

This example uses --nodeps to bypass a mistake made during the build of this release. It won't be needed in newer releases.

```bash
# Install the packages in the correct order
sudo rpm -i /opt/rpm_installs/erlang-otp-19.3-1.el8.x86_64.rpm --nodeps
sudo rpm -i /opt/rpm_installs/rebar-2.6.4-1.el8.x86_64.rpm --nodeps
sudo rpm -i /opt/rpm_installs/elixir-1.5.3-1.el8.x86_64.rpm --nodeps
sudo rpm -i /opt/rpm_installs/kazoo-classic-4.3-1.el8.x86_64.rpm --nodeps
```

## 3. Setting Up Container Runtime (for RabbitMQ)

Kazoo uses an AMQP message bus via RabbitMQ. To avoid dependancy conflicts, it's recommended to run this via a container (docker/podman).

### Install Podman for Container Management

```bash
sudo yum install -y podman podman-docker podman-compose
```

### Configure Container Settings

```bash
# Create necessary directories
sudo mkdir -p /etc/containers/registries.conf.d

# Configure default registry
cat << EOF | sudo tee /etc/containers/registries.conf.d/000-docker.conf
unqualified-search-registries = ["docker.io"]
EOF

```

## 4. Setting Up RabbitMQ

### Create RabbitMQ Configuration Directories

```bash
sudo mkdir -p /etc/kazoo/rabbitmq
sudo mkdir -p /opt/docker/kazoo-rabbitmq
sudo mkdir -p /opt/docker/kazoo-rabbitmq/.docker-conf/rabbitmq/data
sudo mkdir -p /opt/docker/kazoo-rabbitmq/.docker-conf/rabbitmq/log

# Create rabbitmq group/user
sudo groupadd -g 1999 rabbitmq
sudo useradd -u 1999 -g rabbitmq -s /sbin/nologin -d /nonexistent rabbitmq

# Set permissions
sudo chown -R rabbitmq:rabbitmq /etc/kazoo/rabbitmq
sudo chown -R rabbitmq:rabbitmq /opt/docker/kazoo-rabbitmq
```

### Download RabbitMQ Configuration

```bash
# Clone Kazoo RabbitMQ config repository
git clone https://github.com/2600hz/kazoo-configs-rabbitmq.git -b 4.3 /tmp/kazoo-configs-rabbitmq

# Copy files to the correct location
sudo cp -r /tmp/kazoo-configs-rabbitmq/rabbitmq/* /etc/kazoo/rabbitmq/
```

### Create Erlang Cookie

```bash
# Generate a secure cookie or use a predetermined one for clustering
# This MUST be the same across all Kazoo nodes
ERLANG_COOKIE="GENERATE_A_SECURE_RANDOM_STRING_HERE"

# Save the cookie
echo "$ERLANG_COOKIE" | sudo tee /opt/docker/kazoo-rabbitmq/.docker-conf/rabbitmq/data/.erlang.cookie
sudo chmod 400 /opt/docker/kazoo-rabbitmq/.docker-conf/rabbitmq/data/.erlang.cookie
sudo chown rabbitmq:rabbitmq /opt/docker/kazoo-rabbitmq/.docker-conf/rabbitmq/data/.erlang.cookie
```

### Create RabbitMQ Container Service

```bash
cat << EOF | sudo tee /etc/systemd/system/rabbitmq-container.service
[Unit]
Description=RabbitMQ Container
Requires=network-online.target
After=network-online.target

[Service]
Type=simple
Environment=PODMAN_SYSTEMD_UNIT=%n
Restart=always
RestartSec=30
TimeoutStartSec=900

ExecStartPre=-/usr/bin/podman rm -f rabbitmq
ExecStart=/usr/bin/podman run \\
    --name rabbitmq \\
    --uidmap 0:100000:999 \\
    --uidmap 999:1999:1 \\
    --gidmap 0:100000:999 \\
    --gidmap 999:1999:1 \\
    --network rabbitmq_go_net \\
    -v /opt/docker/kazoo-rabbitmq/.docker-conf/rabbitmq/data:/var/lib/rabbitmq:Z \\
    -v /opt/docker/kazoo-rabbitmq/.docker-conf/rabbitmq/log:/var/log/rabbitmq:Z \\
    -v /etc/kazoo/rabbitmq:/etc/rabbitmq:Z \\
    -p 5672:5672 \\
    -p 15672:15672 \\
    --ulimit nofile=64000:64000 \\
    docker.io/rabbitmq:3.13-management

ExecStop=/usr/bin/podman stop -t 10 rabbitmq
ExecStopPost=/usr/bin/podman rm -f rabbitmq

[Install]
WantedBy=multi-user.target
EOF

# Create the network
sudo podman network create rabbitmq_go_net

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable rabbitmq-container
sudo systemctl start rabbitmq-container

# Wait for RabbitMQ to start
echo "Waiting for RabbitMQ to start..."
until sudo podman exec rabbitmq rabbitmqctl status &>/dev/null; do
  sleep 5
done
echo "RabbitMQ is running"
```

## 5. Configuring Kazoo

### Create Kazoo Configuration Directory Structure

```bash
sudo mkdir -p /etc/kazoo/core
```

### Configure Kazoo Core Settings

Create the main configuration file:

```bash
cat << EOF | sudo tee /etc/kazoo/core/config.ini
; Core Kazoo configuration

[zone]
name = "v1"
amqp_uri = "amqp://guest:guest@app1.example.com:5672"

[bigcouch]
compact_automatically = true
cookie = "YOUR_COUCHDB_COOKIE"
ip = "YOUR_COUCHDB_IP"
port = 15984
username = "YOUR_COUCHDB_USER"
password = "YOUR_COUCHDB_PASSWORD"
admin_port = 15986

[kazoo_apps]
cookie = YOUR_ERLANG_COOKIE
zone = "v1"
host = "app1.example.com"

[ecallmgr]
cookie = YOUR_ERLANG_COOKIE
zone = "v1"
host = "app1.example.com"

[log]
syslog = info
console = notice
file = error
EOF
```

> **Note**: Replace the placeholders with your actual values. Make sure the Erlang cookie here matches the one set for RabbitMQ.

### Configure Kazoo Service

Create a systemd service file for Kazoo:

```bash
cat << EOF | sudo tee /etc/default/kazoo-applications
SHM_MEMORY=64
PKG_MEMORY=8
DUMP_CORE=no
CFGFILE=/etc/kazoo/core/config.ini
EOF

sudo systemctl daemon-reload
sudo systemctl enable kazoo-applications
sudo systemctl start kazoo-applications
```

### Wait for Kazoo to Start

```bash
# Wait for Kazoo to start fully (may take a minute or two)
echo "Waiting for Kazoo to start..."
until sup kz_nodes status 2>/dev/null | grep -q "kazoo_apps@"; do
  sleep 10
  echo "Still waiting..."
done
echo "Kazoo is running"
```

## 6. Configuring System Settings

### Setup System Configuration

Once Kazoo is running, you can configure various subsystems:

```bash
# Set up Crossbar (API Server) settings
sup kapps_config set crossbar auth_tokeninfo_enabled true
# Enable modules to autoload
sup kapps_config set crossbar autoload_modules '["cb_about", "cb_accounts", "cb_api_auth", "cb_basic_auth", "cb_callflows", "cb_devices_v1", "cb_devices_v2", "cb_directories", "cb_faxboxes", "cb_faxes", "cb_phone_numbers_v1", "cb_phone_numbers_v2", "cb_users_v1", "cb_users_v2", "cb_vmboxes", "cb_whitelabel"]'

# Configure ecallmgr (Call Control)
# Use an australian ringtone
sup kapps_config set ecallmgr default_ringback "%(400,200,400,425);%(400,2000,400,425)"
# Enable authz and deny if not authed
sup kapps_config set ecallmgr authz_enabled true
sup kapps_config set ecallmgr authz_default_action deny

# Configure Email notifications
sup kapps_config set smtp_client relay "YOUR_SMTP_SERVER"
sup kapps_config set smtp_client port "2525"
sup kapps_config set notify default_from "no_reply@example.com"
```

### Start Kazoo Applications

```bash
# Optionally manually start core applications if needed
sup kapps_controller start_app blackhole
sup kapps_controller start_app callflow
sup kapps_controller start_app cdr
sup kapps_controller start_app conference
sup kapps_controller start_app crossbar
sup kapps_controller start_app fax
sup kapps_controller start_app hangups
sup kapps_controller start_app media_mgr
sup kapps_controller start_app milliwatt
sup kapps_controller start_app omnipresence
sup kapps_controller start_app pivot
sup kapps_controller start_app registrar
sup kapps_controller start_app reorder
sup kapps_controller start_app stepswitch
sup kapps_controller start_app teletype
sup kapps_controller start_app webhook

# Check running apps
sup kapps_controller running_apps
```

## 7. Setting Up Nginx and SSL Certificates

### Install Nginx

```bash
# For RHEL/CentOS
sudo yum install -y nginx

# For Debian/Ubuntu
# sudo apt-get update
# sudo apt-get install -y nginx
```

### Install Certbot for SSL Certificates

```bash
# For RHEL/CentOS 8
sudo yum install -y epel-release
sudo yum install -y certbot python3-certbot-nginx

# For Debian/Ubuntu
# sudo apt-get install -y certbot python3-certbot-nginx
```

### Obtain SSL Certificate

```bash
# Replace portal.example.com with your actual domain
# This is an example, you'll need to configure certbot for however it suits you
sudo certbot certonly --nginx -d portal.example.com --email admin@example.com --agree-tos --non-interactive
```

### Configure Nginx for Monster UI

Create the Nginx configuration file:

```bash
cat << 'EOF' | sudo tee /etc/nginx/conf.d/monster-ui.conf
server {
    listen 80;
    server_name portal.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name portal.example.com;

    # SSL configuration
    ssl_certificate /etc/letsencrypt/live/portal.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/portal.example.com/privkey.pem;

    # Modern SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_stapling on;
    ssl_stapling_verify on;

    # HSTS (comment out if you're not sure)
    # add_header Strict-Transport-Security "max-age=63072000" always;

    root /var/www/monster-ui;
    index index.html;

    # Logging
    access_log /var/log/nginx/monster-ui_access.log combined buffer=512k flush=1m;
    error_log /var/log/nginx/monster-ui_error.log warn;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 7d;
        add_header Cache-Control "public, no-transform";
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-secure" always;
}
EOF
```

> **Note**: Replace `portal.example.com` with your actual Monster UI domain.

### Set Up Auto-Renewal for SSL Certificates

```bash
# Test auto-renewal
sudo certbot renew --dry-run

# Set up cron job for auto-renewal (runs twice daily)
echo "0 0,12 * * * root python -c 'import random; import time; time.sleep(random.random() * 3600)' && certbot renew -q" | sudo tee -a /etc/crontab > /dev/null
```

### Start and Enable Nginx

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

## 8. Setting Up Monster UI (Web Interface)

### Create Directory for Monster UI

```bash
sudo mkdir -p /var/www/monster-ui
```

### Download and Extract Monster UI

```bash
# Download the latest release from kazoo-classic repository
wget -O /tmp/monster-ui-4.3.c1-allapps.tar.gz https://github.com/kazoo-classic/monster-ui/releases/download/v4.3.1-allapps/monster-ui-4.3.c1-allapps.tar.gz

# Extract the archive
sudo tar xzvf /tmp/monster-ui-4.3.c1-allapps.tar.gz -C /var/www/monster-ui --strip-components=1

# Set proper ownership
sudo chown -R nginx:nginx /var/www/monster-ui
```

> **Note**: Always check the [Kazoo Classic Monster UI releases page](https://github.com/kazoo-classic/monster-ui/releases) for the latest version and update the URL accordingly.

### Configure Monster UI

The example here is using port 8443, assuming you've put your API behind a HAProxy server that's running the api via SSL

```bash
cat << EOF | sudo tee /var/www/monster-ui/js/config.js
define({
    api: {
        'default': 'https://api.example.com:8443/v2/',
    },
    advancedView: true,
    resellerId: 'YOUR_RESELLER_ACCOUNT_ID',
    disableBraintree: true,
    whitelabel: {
        additionalCss: ['branding/custom.css'],
        companyName: 'Your Company',
        applicationTitle: 'Your Voice Platform',
        callReportEmail: 'support@example.com',
        nav: {
            help: 'https://www.example.com'
        },
        port: {
            loa: 'https://www.example.com/loa.pdf',
            resporg: 'https://www.example.com/resporg.pdf'
        }
    }
});
EOF
```

### Create Custom CSS (Optional)

If you want to customize the look and feel of the interface:

```bash
# Create directory for custom CSS
sudo mkdir -p /var/www/monster-ui/css/branding

# Create a custom CSS file
cat << EOF | sudo tee /var/www/monster-ui/css/branding/custom.css
/* Custom branding styles */
.company-logo {
    background-image: url(../../img/logo.png);
    background-repeat: no-repeat;
    background-size: contain;
    width: 250px;
    height: 80px;
}

.login-content .left-part {
    background-color: #3D8CBA;
}

/* Additional customizations can be added here */
EOF

# Set proper permissions
sudo chown nginx:nginx /var/www/monster-ui/css/branding/custom.css
```

### Custom Logo Setup (Optional)

```bash
# If you have a custom logo, place it in the right directory
sudo cp your-company-logo.png /var/www/monster-ui/img/logo.png
sudo chown nginx:nginx /var/www/monster-ui/img/logo.png
```

### OPTION 1: Initialize Monster UI Apps via SUP (easy, if frontend is on the same server)

```bash
sup crossbar_maintenance init_apps '/var/www/monster-ui/apps' 'http://your.api.{{SERVER}}:8000/v2'
```

### OPTION 2: Initialize Monster UI Apps via the API

The Monster UI tarball you downloaded already includes all the apps, but they need to be registered with the Kazoo platform:

```bash
# Make sure jq is installed (for JSON processing)
sudo yum install -y jq

# md5sum the password as per Basic Auth docs
PASSWORD=`echo -n YOUR_USERNAME:YOUR_PASSWORD | md5sum | awk '{print $1}'`

# Get the authentication token
AUTH_TOKEN=$(curl -s -X PUT https://api.example.com:8443/v2/api_auth \
  -d '{"data":{"credentials":"YOUR_USERNAME:PASSWORD"}}' | jq -r '.auth_token')

# Check if auth token was obtained successfully
if [ -z "$AUTH_TOKEN" ] || [ "$AUTH_TOKEN" == "null" ]; then
  echo "Failed to obtain auth token. Check your credentials and API endpoint."
  exit 1
fi

# Define the apps to initialize
APPS=("voip" "accounts" "numbers" "callflows" "fax" "pbxs" "voicemails" "webhooks")
API_URL="https://api.example.com:8443/v2"
ACCOUNT_ID="YOUR_ACCOUNT_ID"

# Initialize each app
for APP in "${APPS[@]}"; do
  echo "Initializing $APP app..."
  curl -s -X PUT "https://api.example.com:8443/v2/accounts/$ACCOUNT_ID/apps_store" \
    -H "X-Auth-Token: $AUTH_TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"data\": {\"name\": \"$APP\", \"api_url\": \"$API_URL\"}}"
  echo ""
done

echo "App initialization complete!"
```

> **Note**: Replace `YOUR_USERNAME:YOUR_PASSWORD`, `YOUR_ACCOUNT_ID`, and other placeholders with your actual values. `YOUR_ACCOUNT_ID` should be your root reseller account id.

### Test Monster UI Access

Once everything is set up:

1. Navigate to your Monster UI domain in a browser: `https://portal.example.com:8443`
2. You should see the login screen with your custom branding (if configured)
3. Log in with your credentials to verify the setup is working correctly

## 9. Configuring Carriers (Optional Example)

To configure SIP trunking carriers:

```bash
# Create a carrier
curl -X PUT https://api.example.com:8443/v2/resources \
  -H "X-Auth-Token: $AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "name": "YourCarrier",
      "emergency": false,
      "enabled": true,
      "flags": ["fax"],
      "grace_period": 6,
      "weight_cost": 10,
      "formatters": {
        "caller_id_number": {
          "regex": "^\\\\+?(\\\\d+)$"
        },
        "outbound_caller_id_number": {
          "regex": "^\\\\+?(\\\\d+)$"
        }
      },
      "gateways": [
        {
          "server": "sbc.carrier.com",
          "realm": "carrier.com",
          "username": "YOUR_USERNAME",
          "password": "YOUR_PASSWORD",
          "enabled": true,
          "channel_selection": "ascending",
          "endpoint_type": "sip",
          "invite_format": "route",
          "port": 5060
        }
      ],
      "media": {
        "audio": {
          "codecs": [
            "PCMA",
            "PCMU",
            "G729",
            "G722_16"
          ]
        },
        "video": {
          "codecs": []
        },
        "fax_option": true
      },
      "rules": ["^\\\\+1(\\\\d{10})$"]
    }
  }'
```

### Test Carrier Configuration

After setting up a carrier, you can test if it's properly configured:

```bash
# Get the list of configured carriers
curl -s https://api.example.com:8443/v2/resources \
  -H "X-Auth-Token: $AUTH_TOKEN" | jq '.data'

# Test carrier matching for a specific number
curl -s -X POST https://api.example.com:8443/v2/accounts/YOUR_ACCOUNT_ID/resources/resource_templates/classifiers \
  -H "X-Auth-Token: $AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data":{"numbers":["+15551234567"]}}' | jq '.'
```

## 10. Verification and Troubleshooting

### Verify Kazoo is Running

```bash
# Check overall status
sup kz_nodes status

# Check API access
curl -v http://localhost:8000/v2/accounts
```

### Check Logs

```bash
# View Kazoo logs
journalctl -u kazoo-applications

# Check RabbitMQ logs
sudo podman logs rabbitmq
```

### Troubleshooting Steps

If Kazoo fails to start:

1. Check RabbitMQ is running:
   ```bash
   sudo systemctl status rabbitmq-container
   ```

2. Verify CouchDB connectivity:
   ```bash
   curl -u username:password http://your-couchdb-server:5984
   ```

3. Check Erlang cookies are consistent:
   ```bash
   # Compare the cookies
   cat /opt/docker/kazoo-rabbitmq/.docker-conf/rabbitmq/data/.erlang.cookie
   grep cookie /etc/kazoo/core/config.ini
   ```

4. Restart services:
   ```bash
   sudo systemctl restart rabbitmq-container
   sudo systemctl restart kazoo-applications
   ```

## 11. Maintenance

### SSL Certificate Renewal

Certbot should automatically renew certificates, but you can manually trigger renewal. You may need to set nginx to reload upon deployment.

```bash
sudo certbot renew
```

## Additional Configurations

### Firewall Setup (Optional)

If you're using firewalld:

```bash
# Open necessary ports
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-port=5672/tcp  # RabbitMQ
sudo firewall-cmd --permanent --add-port=8000/tcp  # Kazoo API
sudo firewall-cmd --permanent --add-port=15672/tcp # RabbitMQ admin
sudo firewall-cmd --reload
```

### System Tuning for High Performance (Optional)

Adjust system limits and kernel parameters:

```bash
# Create a sysctl configuration file for Kazoo
cat << EOF | sudo tee /etc/sysctl.d/30-kazoo.conf
# Increase file descriptor limits
fs.file-max = 300000

# Increase TCP performance
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_tw_reuse = 1
EOF

# Apply sysctl settings
sudo sysctl -p /etc/sysctl.d/30-kazoo.conf

# Create a limits file for Kazoo
cat << EOF | sudo tee /etc/security/limits.d/kazoo.conf
kazoo soft nofile 65536
kazoo hard nofile 65536
kazoo soft nproc 65536
kazoo hard nproc 65536
EOF
```

## Conclusion

You have successfully deployed and configured a Kazoo application server manually. This server is now ready to handle API requests, call processing logic, and integrate with the other components of your voice platform. The deployment includes:

- Core Kazoo application server
- RabbitMQ message broker
- Nginx web server with SSL
- Monster UI web interface

For more detailed information on specific Kazoo components and features, refer to the [Kazoo documentation](https://docs.2600hz.com/). Always check the [Kazoo Classic GitHub](https://github.com/kazoo-classic/kazoo/releases) for the latest releases and updates.