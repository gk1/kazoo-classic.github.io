---
title: Install and Configure HAProxy
layout: default
parent: Community Guides
---

# HAProxy Setup Guide for Kazoo Voice Platform

This guide provides step-by-step instructions for manually setting up HAProxy as a load balancer and SSL terminator for a Kazoo voice platform. HAProxy will handle traffic for CouchDB, RabbitMQ, Kazoo API, and other optional related services.

## Prerequisites

- A Debian/Ubuntu-based server
- Already installed and running CouchDB, RabbitMQ, Kazoo-Applications

## 1. Installation

### 1.1 Install Required Packages

First, install HAProxy and required SSL/TLS tools:

```bash
apt-get update
apt-get install -y haproxy python3 python3-pip certbot

# If you're using cloudflare, do your cert renewal via cloudflare DNS validation
apt-get install -y python3-certbot-dns-cloudflare
```

## 2. SSL Certificate Setup

### 2.1 Create Required Directories If They Don't Exist

```bash
mkdir -p /etc/letsencrypt/renewal-hooks/deploy
mkdir -p /etc/letsencrypt/renewal-hooks/post
mkdir -p /etc/haproxy/ssl
mkdir -p /root/.secrets/certbot
```

### 2.2 Configure Cloudflare Credentials (Optional)

Create a configuration file for Cloudflare DNS authentication:

```bash
cat > /root/.secrets/certbot/cloudflare.ini << EOF
# Cloudflare API token for certificate management
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN
EOF

# Secure the credentials file
chmod 600 /root/.secrets/certbot/cloudflare.ini
```

### 2.3 Create Certificate Deployment Script (Optional)

To make my life easy, I made a dynamic python script to assist with post-renewal deployment of renewed certificates:

```bash
cat > /etc/letsencrypt/renewal-hooks/deploy/01-deploy-cert.py << EOF
#!/usr/bin/env python3
import os
import re
import sys
import logging
from datetime import datetime

# Set up logging
LOG_FILE = '/var/log/letsencrypt-deploy.log'
logging.basicConfig(
    filename=LOG_FILE,
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

def ensure_directory_exists(path):
    """Create directory and any necessary parent directories if they don't exist."""
    try:
        os.makedirs(os.path.dirname(path), exist_ok=True)
        return True
    except Exception as e:
        logging.error(f"Error creating directory {os.path.dirname(path)}: {e}")
        return False

def get_base_hostname(domain):
    """Extract the first part of the hostname (before the first dot)."""
    return domain.split('.')[0]

def main():
    try:
        # Log start of execution
        logging.info("Starting certificate deployment")

        # Certbot sets an environment variable RENEWED_LINEAGE
        lineage = os.environ.get('RENEWED_LINEAGE')

        # If nothing renewed, exit
        if not lineage:
            logging.info("No certificates renewed, exiting")
            return

        # Extract domain name from path
        result = re.match(r'.*/live/(.+)$', lineage)

        # Exit if path not recognized
        if not result:
            logging.error(f"Could not parse domain from lineage path: {lineage}")
            sys.exit(1)

        # Extract domain name and get base hostname
        domain = result.group(1)
        base_hostname = get_base_hostname(domain)
        logging.info(f"Processing domain: {domain} (base hostname: {base_hostname})")

        # Define HAProxy cert path
        deploy_path = f"/etc/haproxy/ssl/{base_hostname}.pem"

        # Ensure destination directory exists
        if not ensure_directory_exists(deploy_path):
            sys.exit(1)

        # Source certificate files
        source_key = lineage + "/privkey.pem"
        source_chain = lineage + "/fullchain.pem"

        # Verify source files exist
        if not os.path.isfile(source_key):
            logging.error(f"Source key file not found: {source_key}")
            sys.exit(1)
        if not os.path.isfile(source_chain):
            logging.error(f"Source chain file not found: {source_chain}")
            sys.exit(1)

        # Combine key and chain for HAProxy
        with open(deploy_path, "w") as deploy, \\
                open(source_key, "r") as key, \\
                open(source_chain, "r") as chain:
            deploy.write(key.read())
            deploy.write(chain.read())

        logging.info(f"Successfully deployed certificate for {domain} to {deploy_path}")

    except Exception as e:
        logging.error(f"Unexpected error during certificate deployment: {str(e)}")
        sys.exit(1)

if __name__ == "__main__":
    # Ensure log directory exists
    log_dir = os.path.dirname(LOG_FILE)
    try:
        os.makedirs(log_dir, exist_ok=True)
    except Exception as e:
        print(f"Error creating log directory {log_dir}: {e}", file=sys.stderr)
        sys.exit(1)

    main()
EOF

chmod 755 /etc/letsencrypt/renewal-hooks/deploy/01-deploy-cert.py
```

### 2.4 Obtain SSL Certificates (via Cloudflare)

Run for each domain you need to secure:

```bash
certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/certbot/cloudflare.ini \
  -d api.example.com \
  --non-interactive \
  --agree-tos \
  --email your-email@example.com \
  --dns-cloudflare-propagation-seconds 20
```

Repeat for other domains like:
- db.example.com
- mq.example.com

### 2.5 Combine Certificates for HAProxy

For each domain, combine the certificate and private key. This is also done by the post-deploy script above, but you can do it manually for the first run if you want:

```bash
cat /etc/letsencrypt/live/api.example.com/fullchain.pem \
    /etc/letsencrypt/live/api.example.com/privkey.pem \
    > /etc/haproxy/ssl/api.pem

chmod 600 /etc/haproxy/ssl/api.pem
```

Repeat for all other domains.

### 2.6 Configure Automatic Certificate Renewal

You should be able to do this as an environment varible through sysconfig. You can optionally create a renewal script and place it in the appropriate location(s).

```bash
echo 'DEPLOY_HOOK="systemctl reload haproxy"' >> /etc/sysconfig/certbot
```

## 3. HAProxy Configuration

### 3.1 Create Configuration Directory

```bash
mkdir -p /etc/haproxy/configs
```

### 3.2 Create Base Global Configuration

```bash
cat > /etc/haproxy/configs/_globals.cfg << EOF
global
        log /dev/log    local0 info
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin
        stats timeout 30s
        user haproxy
        group haproxy
        daemon
        tune.ssl.default-dh-param 2048
        stats socket /var/lib/haproxy/stats
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private
        ssl-default-bind-ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
        ssl-default-bind-ciphersuites TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
        ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        option  log-health-checks
        option  httpchk
        option  allbackups
        option  http-server-close
        option  redispatch
        http-check send meth GET uri / ver HTTP/1.1 hdr Host localhost
        timeout connect          12000ms
        timeout client           12000ms
        timeout server           12000ms
        timeout http-request     12000ms
        timeout queue            24000ms
        timeout http-keep-alive  10s
        timeout check           10s
        retries 3
        maxconn 2000

listen haproxy-stats
  bind 0.0.0.0:22002
  mode http
  stats uri /
  stats refresh 10s
EOF
```

### 3.3 Configure API Frontend/Backend

```bash
cat > /etc/haproxy/configs/api.cfg << EOF
# API HTTPS Frontend
frontend www-api-https
  bind 0.0.0.0:8443 ssl crt /etc/haproxy/ssl/api.pem
  mode http
  timeout client          60000ms
  redirect scheme https if !{ ssl_fc }
  default_backend www-api-http

# API HTTP Backend
backend www-api-http
  balance roundrobin
  option forwardfor
  option httpchk
  http-check send meth GET uri / ver HTTP/1.1 hdr Host localhost
  timeout http-request    60000ms
  timeout queue           64000ms
  timeout http-keep-alive 20s
  timeout check           20s
  timeout connect         60000ms
  timeout server          60000ms
  http-request set-header X-Forwarded-Port %[dst_port]
  http-request add-header X-Forwarded-Proto https if { ssl_fc }
  server app1.node1.example.com 10.0.0.1:8000 check
EOF
```

### 3.4 Configure CouchDB Frontend/Backend (optional)

```bash
cat > /etc/haproxy/configs/couchdb.cfg << EOF
# CouchDB Data Listener
listen couchdb-data
  bind 0.0.0.0:15984
  http-check disable-on-404
  option httpchk
  http-check send meth GET uri /_up ver HTTP/1.1 hdr Host localhost
  option forwardfor
  balance roundrobin
  server db1.node1.example.com 127.0.0.1:5984 check inter 5s fall 6 rise 3

# CouchDB NODE Listener - used to make kazoo v4 work with couchdb v3.
listen couchdb-data-node
  bind 0.0.0.0:15986
  http-check disable-on-404
  option httpchk
  http-check send meth GET uri / ver HTTP/1.1 hdr Host localhost
  http-request set-path /_node/_local
  option forwardfor
  balance roundrobin
  server db1.node1.example.com 127.0.0.1:5984 check inter 5s fall 6 rise 3

# CouchDB HTTPS Frontend
frontend www-couchdb-data-https
  bind 0.0.0.0:25984 ssl crt /etc/haproxy/ssl/db.pem
  mode http
  redirect scheme https if !{ ssl_fc }
  default_backend couchdb-data

# CouchDB HTTPS Node Frontend
frontend www-couchdb-node-https
  bind 0.0.0.0:25986 ssl crt /etc/haproxy/ssl/db.pem
  mode http
  redirect scheme https if !{ ssl_fc }
  default_backend couchdb-data-node
EOF
```

### 3.5 Configure N8N Frontend/Backend (optional)

If using N8N, you need specific ssl versions and ciphers for this to work, otherwise pivots in callflows will fail:

```bash
cat > /etc/haproxy/configs/n8n.cfg << EOF
# FRONTEND n8n
frontend www-n8n-https
  bind 0.0.0.0:443 ssl crt /etc/haproxy/ssl/pivot.pem ssl-min-ver TLSv1.2 ssl-max-ver TLSv1.3 ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:@SECLEVEL=0
  log global
  option httplog
  log-format "%ci:%cp [%tr] %ft %b/%s %TR/%Tw/%Tc/%Tr/%Ta %ST %B %CC %CS %tsc %ac/%fc/%bc/%sc/%rc %sq/%bq %sslv/%sslc %{+Q}r"
  capture request header User-Agent len 128
  mode http
  timeout client          60000ms
  redirect scheme https if !{ ssl_fc }
  default_backend www-n8n-http

frontend www-n8n-http
  bind 0.0.0.0:8765
  mode http
  default_backend www-n8n-http

# BACKEND n8n
backend www-n8n-http
  balance roundrobin
  option forwardfor
  option httpchk
  http-check send meth GET uri / ver HTTP/1.1 hdr Host localhost
  timeout http-request    60000ms
  timeout queue           64000ms
  timeout http-keep-alive 20s
  timeout check           20s
  timeout connect         60000ms
  timeout server          60000ms
  http-request set-header X-Forwarded-Port %[dst_port]
  http-request add-header X-Forwarded-Proto https if { ssl_fc }
  server extra1.node1.example.com 10.0.0.4:5678 check
EOF
```

### 3.6 Configure RabbitMQ Frontend/Backend (optional)

```bash
cat > /etc/haproxy/configs/rabbitmq.cfg << EOF
# RabbitMQ Management Interface
listen kazoo-rabbitmq
  bind 0.0.0.0:15672
  http-check disable-on-404
  option httpchk
  http-check send meth GET uri / ver HTTP/1.1 hdr Host localhost
  option forwardfor
  balance roundrobin
  server app1.node1.example.com 10.0.0.1:15672 check
EOF
```

### 3.7 SMTP to Fax Relay (optional)

Outside the scope of this guide, it's recommended to use HAProxy to relay SMTP traffic to the SMTP to FAX port on your Apps node. If you would like to submit a PR with this config, it can be merged with this guide.

For security, I'd recommend running a postfix service instead as your relay to filter submissions to the fax service.

## 4. HAProxy Service Configuration

### 4.1 Configure Systemd Service (Custom optional version)

This particular service file sets HAProxy to load a series of config files from the conf.d folder, so it doesn't have to have everything in one conf file

```bash
cat > /etc/systemd/system/haproxy.service << EOF
[Unit]
Description=HAProxy Load Balancer
Documentation=man:haproxy(1)
Documentation=file:/usr/share/doc/haproxy/configuration.txt.gz
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
Environment="CONFIG=/etc/haproxy/configs"
EnvironmentFile=-/etc/default/haproxy
ExecStartPre=/usr/sbin/haproxy -f $CONFIG -c
ExecStart=/usr/sbin/haproxy -Ws -f $CONFIG -p /run/haproxy.pid
ExecReload=/usr/sbin/haproxy -f $CONFIG -c
ExecReload=/bin/kill -USR2 $MAINPID
Restart=on-failure
RuntimeDirectory=haproxy
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target
EOF
```

### 4.3 Reload Systemd and Enable Service

```bash
systemctl daemon-reload
systemctl enable haproxy
```

## 5. Starting and Testing HAProxy

### 5.1 Start HAProxy Service

```bash
systemctl start haproxy
```

### 5.2 Check Service Status

```bash
systemctl status haproxy
```

### 5.3 Test Configuration

```bash
haproxy -c -f /etc/haproxy/configs
```

### 5.4 Test SSL Connectivity

```bash
openssl s_client -connect portal.example.com:8443
```

## 6. Monitoring and Troubleshooting

### 6.1 Access HAProxy Stats Dashboard

Open a web browser and navigate to:

```
http://your-server-ip:22002/
```

### 6.2 Check Logs

```bash
journalctl -u haproxy
```

### 6.3 Check Certificate Renewal Logs (If using the optional post-deploy script)

```bash
cat /var/log/letsencrypt-deploy.log
```

## 7. Maintenance

### 7.1 Reloading Configuration

After making changes to configuration files:

```bash
systemctl reload haproxy
```

### 7.2 Testing Configuration Before Reload

```bash
haproxy -c -f /etc/haproxy/configs
```

### 7.3 Manually Testing Certificate Renewal

```bash
certbot renew --dry-run
```

## Conclusion

Your HAProxy installation is now configured to provide load balancing and SSL termination for your Kazoo voice platform services. It will handle CouchDB, RabbitMQ, API, and other services' traffic, with automatic certificate renewal through Let's Encrypt.