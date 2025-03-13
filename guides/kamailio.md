---
title: Install and Configure Kamailio
layout: default
parent: Community Guides
---

# Manual Setup Guide for Kamailio with PostgreSQL for Kazoo/2600Hz

This guide will help you manually set up a Kamailio Session Border Controller (SBC) with PostgreSQL for Kazoo/2600Hz. Since the 2600hz provided database for kamailio (Kazoo_DB) is not open source, we have to replace it with PostgreSQL.

This is an improvement anyway, as KazooDB has issues with presence/BLF at larger scales without putting it on a ramdisk.

## Overview

The Kamailio SBC acts as the entry point for SIP traffic in a Kazoo platform. It handles tasks like:
- SIP signaling management
- Load balancing
- Security and authentication
- Registration handling

## Prerequisites

- Debian/Ubuntu system (Debian 11 Bullseye recommended)
- Basic understanding of SIP, PostgreSQL, and Kazoo

## 1. PostgreSQL Installation

```bash
# Add PostgreSQL repository
apt-key add https://www.postgresql.org/media/keys/ACCC4CF8.asc
echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/postgresql.list

# Update and install PostgreSQL 13
apt update
apt install -y postgresql-13 postgresql-client-13 postgresql-contrib-13 \
    python3-psycopg2 postgresql-server-dev-13
```

## 2. PostgreSQL Configuration

Edit PostgreSQL configuration files:

```bash
# Configure PostgreSQL access
nano /etc/postgresql/13/main/pg_hba.conf
```

Add these lines:

```
# IPv4 local connections:
host    all             all             127.0.0.1/32            password
```

Configure main PostgreSQL settings:

```bash
nano /etc/postgresql/13/main/postgresql.conf
```

Key settings to update:

```conf
# Basic settings
listen_addresses = '127.0.0.1'
port = 5432

# Memory
shared_buffers = 512MB
max_connections = 400
superuser_reserved_connections = 10

# For Kamailio performance
datestyle = 'iso, dmy'
timezone = 'UTC'  # Or your timezone
```

Restart PostgreSQL:

```bash
systemctl restart postgresql@13-main
```

## 3. Kamailio Installation

### Add Kamailio Repository

```bash
apt-key add https://deb.kamailio.org/kamailiodebkey.gpg
echo "deb http://deb.kamailio.org/kamailio55 bullseye main" > /etc/apt/sources.list.d/kamailio.list
echo "deb-src http://deb.kamailio.org/kamailio55 bullseye main" >> /etc/apt/sources.list.d/kamailio.list
apt update
```

### Install Kamailio Packages

```bash
apt install -y kamailio kamailio-postgres-modules kamailio-kazoo-modules \
    kamailio-outbound-modules kamailio-presence-modules kamailio-tls-modules \
    kamailio-utils-modules kamailio-websocket-modules kamailio-extra-modules \
    kamailio-xmpp-modules
```

## 4. Database Setup for Kamailio

Create kamailio database and user: 
```bash
sudo -u postgres psql -c "CREATE DATABASE kamailio;"
sudo -u postgres psql -c "CREATE USER kamailio WITH PASSWORD 'your_secure_password';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE kamailio TO kamailio;"
```

Create the initial table structure:
```
psql -U kamailio -d postgres://kamailio:kamailio@127.0.0.1/kamailio -f /etc/kazoo/kamailio/db_scripts/kamailio_initdb_postgres.sql
```

## 5. Kazoo Configuration

### Clone Kamailio Configuration Repository

```bash
mkdir -p /etc/kazoo
git clone https://github.com/kazoo-classic/kazoo-configs-kamailio.git /etc/kazoo
```

### Initialize Kamailio Database

```bash
# Create necessary directories
mkdir -p /etc/kazoo/kamailio/db
chown kamailio:kamailio /etc/kazoo/kamailio -R

# Initialize Kamailio database
sudo -u postgres psql -U kamailio -d postgres://kamailio:your_secure_password@127.0.0.1/kamailio \
  -f /etc/kazoo/kamailio/db_scripts/kamailio_initdb_postgres.sql
```

## 6. Configure Kamailio

### System Configuration

Create/edit the Kamailio system config:

```bash
nano /etc/default/kamailio
```

Add the recommended values:

```conf
# Kamailio startup options
SHM_MEMORY=64
PKG_MEMORY=8
DUMP_CORE=no
CFGFILE=/etc/kazoo/kamailio/kamailio.cfg
```

### Create Local Configuration

Edit the local configuration file for Kamailio:

```bash
nano /etc/kazoo/kamailio/local.cfg
```

Set your server information:

```conf
#!substdef "!MY_HOSTNAME!sbc.z1.your-domain.com!g"
#!substdef "!MY_IP_ADDRESS!YOUR_SBC_IP_ADDRESS!g"
```

Point AMQP at your RabbitMQ Server. You can add more zones and secondary/terniary urls:

```conf
#!substdef "!MY_AMQP_ZONE!z1!g"
#!substdef "!MY_AMQP_URL!zone=z1;amqp://guest:guest@rabbitmq.your-domain.com:5672!g"
```

Configure your postgres connection
```conf
#!trydef KZ_DB_MODULE postgres
#!substdef "!KAMAILIO_DBMS!postgres!g"
#!substdef "!KAZOO_DB_URL!postgres://kamailio:your_secure_password@127.0.0.1/kamailio!g"
```

Set your SIP bindings:
```conf
#!substdef "!UDP_SIP!udp:MY_IP_ADDRESS:5060!g"
#!substdef "!TCP_SIP!tcp:MY_IP_ADDRESS:5060!g"
#!substdef "!UDP_ALG_SIP!udp:MY_IP_ADDRESS:7000!g"
#!substdef "!TCP_ALG_SIP!tcp:MY_IP_ADDRESS:7000!g"

listen=UDP_SIP
listen=TCP_SIP
listen=UDP_ALG_SIP
listen=TCP_ALG_SIP
```

## 7. Configure Logging

### Configure rsyslog

```bash
mkdir -p /var/log/kamailio
chown kamailio:kamailio /var/log/kamailio

# Create rsyslog configuration
cat > /etc/rsyslog.d/10-kamailio.conf << 'EOF'
if $programname == 'kamailio' then /var/log/kamailio/kamailio.log
& ~
EOF

# Create logrotate configuration
cat > /etc/logrotate.d/kamailio << 'EOF'
/var/log/kamailio/kamailio.log {
    daily
    size 500M
    nodateext
    missingok
    notifempty
    rotate 31
    maxage 5
    create
    compress
    delaycompress
    sharedscripts
    postrotate
        /bin/kill -HUP `cat /var/run/syslogd.pid 2> /dev/null` 2> /dev/null || true
        /bin/kill -HUP `cat /var/run/rsyslogd.pid 2> /dev/null` 2> /dev/null || true
    endscript
}
EOF

# Restart rsyslog
systemctl restart rsyslog
```

## 8. Final Configuration and Start

### Start and Enable Services

```bash
# Start PostgreSQL
systemctl enable postgresql
systemctl start postgresql

# Start Kamailio
systemctl enable kamailio
systemctl start kamailio
```

### Verify Configuration

```bash
# Check Kamailio status
systemctl status kamailio

# Verify logs
tail -f /var/log/kamailio/kamailio.log

# Check SIP ports are open
netstat -tulpn | grep kamailio
```
