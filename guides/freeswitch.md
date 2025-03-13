---
title: Install and Configure FreeSWITCH
layout: default
parent: Community Guides
---

# Manual Setup Guide for FreeSWITCH for Kazoo/2600Hz

This guide will help you manually set up FreeSWITCH as a media server for Kazoo/2600Hz. FreeSWITCH handles all the audio/video media processing and conferencing in a Kazoo platform.

## Overview

FreeSWITCH is an open-source telephony platform that serves as the media server in a Kazoo deployment. It handles:
- Media processing (audio/video)
- Conference bridging
- Transcoding
- Voice applications
- Integration with Kazoo via mod_kazoo

## Prerequisites

- Debian/Ubuntu system (Debian 11 Bullseye recommended)
- SignalWire token (required for FreeSWITCH repository access)
- Basic understanding of VoIP and Kazoo

## 1. System Preparation

Begin by updating your system and installing required packages:

```bash
apt update -y && apt upgrade -y
apt install sudo wget git net-tools lsb-release gnupg2 nano -y
```

## 2. Optional: Secure SSH (if using key-based authentication)

If you've already configured authorized_keys for SSH access, you can secure your SSH daemon:

```bash
cat <<EOF > /etc/ssh/sshd_config.d/root_security.conf
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no
PermitRootLogin no
EOF

# Restart SSH service
systemctl restart sshd
```

## 3. Adding FreeSWITCH Repository

FreeSWITCH packages are available from SignalWire's repository. You'll need to obtain a token from SignalWire.

- [How to create a Signalwire Personal Access Token](https://developer.signalwire.com/freeswitch/FreeSWITCH-Explained/Installation/how-to-create-a-personal-access-token/how-to-create-a-personal-access-token/)

```bash
# Replace YOURTOKENHERE with your actual SignalWire token
TOKEN=YOURTOKENHERE

# Download and install the repository GPG key
wget --http-user=signalwire --http-password=$TOKEN -O /usr/share/keyrings/signalwire-freeswitch-repo.gpg https://freeswitch.signalwire.com/repo/deb/debian-release/signalwire-freeswitch-repo.gpg

# Configure authentication for the repository
echo "machine freeswitch.signalwire.com login signalwire password $TOKEN" > /etc/apt/auth.conf
chmod 600 /etc/apt/auth.conf

# Add the repository to your sources
echo "deb [signed-by=/usr/share/keyrings/signalwire-freeswitch-repo.gpg] https://freeswitch.signalwire.com/repo/deb/debian-release/ `lsb_release -sc` main" > /etc/apt/sources.list.d/freeswitch.list
echo "deb-src [signed-by=/usr/share/keyrings/signalwire-freeswitch-repo.gpg] https://freeswitch.signalwire.com/repo/deb/debian-release/ `lsb_release -sc` main" >> /etc/apt/sources.list.d/freeswitch.list

# Update package lists
apt-get update
```

## 4. Installing FreeSWITCH 1.10.9

For Kazoo compatibility, we'll install FreeSWITCH 1.10.9 instead of newer versions which may have compatibility issues with mod_kazoo or experience crashes when communicating with kazoo-apps via Erlang.

```bash
# Set the specific version for FreeSWITCH 1.10.9
apt-get install -y freeswitch-meta-vanilla=1.10.9* freeswitch-mod-commands=1.10.9* freeswitch-mod-conference=1.10.9* freeswitch-mod-dptools=1.10.9* freeswitch-mod-event-socket=1.10.9* freeswitch-mod-loopback=1.10.9* freeswitch-mod-sofia=1.10.9* freeswitch-mod-dialplan-xml=1.10.9* freeswitch-mod-g723-1=1.10.9* freeswitch-mod-g729=1.10.9* freeswitch-mod-amr=1.10.9* freeswitch-mod-amrwb=1.10.9* freeswitch-mod-ilbc=1.10.9* freeswitch-mod-speex=1.10.9* freeswitch-mod-opus=1.10.9* freeswitch-mod-sndfile=1.10.9* freeswitch-mod-native-file=1.10.9* freeswitch-mod-shout=1.10.9* freeswitch-mod-spandsp=1.10.9* freeswitch-mod-uuid=1.10.9* freeswitch-mod-tone-stream=1.10.9* freeswitch-mod-console=1.10.9* freeswitch-mod-say-en=1.10.9* freeswitch-mod-local-stream=1.10.9* freeswitch-mod-flite=1.10.9* freeswitch-mod-av=1.10.9* freeswitch-mod-h26x=1.10.9* freeswitch-mod-expr=1.10.9* freeswitch-mod-http-cache=1.10.9* freeswitch-mod-logfile=1.10.9* freeswitch-mod-soundtouch=1.10.9* freeswitch-mod-video-filter=1.10.9*

# Install the critical mod_kazoo module
apt-get install -y freeswitch-mod-kazoo=1.10.9*

# Hold the packages to prevent accidental upgrades
apt-mark hold freeswitch libfreeswitch1 freeswitch-mod-kazoo
```

Alternatively, you can set the version via preferences;

```bash
# Create apt preferences file for FreeSWITCH
cat > /etc/apt/preferences.d/freeswitch << EOF
Package: freeswitch*
Pin: version 1.10.9*
Pin-Priority: 1001
EOF
```

Then it's pinned already and you can just install with:

```bash
apt-get install -y freeswitch-meta-vanilla freeswitch-mod-commands freeswitch-mod-conference freeswitch-mod-dptools freeswitch-mod-event-socket freeswitch-mod-loopback freeswitch-mod-sofia freeswitch-mod-dialplan-xml freeswitch-mod-g723-1 freeswitch-mod-g729 freeswitch-mod-amr freeswitch-mod-amrwb freeswitch-mod-ilbc freeswitch-mod-speex freeswitch-mod-opus freeswitch-mod-sndfile freeswitch-mod-native-file freeswitch-mod-shout freeswitch-mod-spandsp freeswitch-mod-uuid freeswitch-mod-tone-stream freeswitch-mod-console freeswitch-mod-say-en freeswitch-mod-local-stream freeswitch-mod-flite freeswitch-mod-av freeswitch-mod-h26x freeswitch-mod-expr freeswitch-mod-http-cache freeswitch-mod-logfile freeswitch-mod-soundtouch freeswitch-mod-video-filter freeswitch-mod-kazoo

# Hold the required libfreeswitch1 package to prevent accidental upgrades
apt-mark hold libfreeswitch1
```
## 5. Setting up the System Environment

Set the timezone to UTC for consistent operation:

```bash
timedatectl set-timezone UTC
```

## 6. Installing Kazoo Configurations and Sounds

Clone the necessary repositories and set up the configuration files:

```bash
# Clone sound and configuration repositories
git clone -b 4.3 https://github.com/2600hz/kazoo-sounds.git
git clone -b 4.3 https://github.com/2600hz/kazoo-configs-freeswitch.git

# Create Kazoo directory structure
mkdir -p /etc/kazoo/freeswitch
mkdir -p /usr/share/kazoo-freeswitch/sounds

# Copy configuration files
cp -R ./kazoo-configs-freeswitch/freeswitch/* /etc/kazoo/freeswitch/
chown -R freeswitch:freeswitch /etc/kazoo/freeswitch/

# Copy sound files
cp -R ./kazoo-sounds/freeswitch/music /usr/share/kazoo-freeswitch/sounds/
cp -R ./kazoo-sounds/freeswitch/en /usr/share/kazoo-freeswitch/sounds/
rm -rf /usr/share/kazoo-freeswitch/sounds/en/gb
chown -R freeswitch:freeswitch /usr/share/kazoo-freeswitch/sounds/
```

## 7. System Configuration

Copy system scripts and configuration files:

```bash
# Copy system scripts and configurations
cp ./kazoo-configs-freeswitch/system/sbin/* /sbin/
cp ./kazoo-configs-freeswitch/system/security/limits.d/freeswitch.limits.conf /etc/security/limits.d/
cp ./kazoo-configs-freeswitch/system/logrotate.d/freeswitch.conf /etc/logrotate.d/

# Update logrotate configuration to run as freeswitch user
# Add 'su freeswitch freeswitch' to line 2 of the logrotate configuration
sed -i '2i su freeswitch freeswitch' /etc/logrotate.d/freeswitch.conf
systemctl restart logrotate

# Install systemd service files
cp ./kazoo-configs-freeswitch/system/systemd/kazoo-freeswitch* /etc/systemd/system/
systemctl daemon-reload
systemctl enable kazoo-freeswitch
```

## 8. Configure Erlang Cookie

Update the Erlang cookie in the kazoo.conf.xml file. This must match the cookie used by your Kazoo application servers:

```bash
# Edit the kazoo.conf.xml file
nano /etc/kazoo/freeswitch/autoload_configs/kazoo.conf.xml
```

Find the following section and update the "erlang-cookie" value:

```xml
<param name="erlang-cookie" value="change_me"/>
```

Replace "change_me" with the same cookie value used by your Kazoo cluster.

## 9. Starting FreeSWITCH

Start the FreeSWITCH service:

```bash
systemctl start kazoo-freeswitch
```

## 10. Verification

Verify that FreeSWITCH is running correctly:

```bash
# Check service status
systemctl status kazoo-freeswitch

# Check logs for any errors
tail -f /var/log/freeswitch/freeswitch.log

# Verify FreeSWITCH is listening on the event socket
netstat -tulpn | grep 8021
```

## Troubleshooting

### Common Issues:

1. **Service fails to start**: Check permissions on the log folders
   ```bash
   chown -R freeswitch:freeswitch /var/log/freeswitch
   ```

2. **Connection issues with Kazoo**: Verify the Erlang cookie matches between FreeSWITCH and Kazoo

3. **Audio problems**: Check that the correct codecs are loaded
   ```bash
   fs_cli -x "show modules" | grep mod_
   ```

### Logs:

The main FreeSWITCH logs are located at:
```
/var/log/freeswitch/freeswitch.log
```

## Security Considerations

- FreeSWITCH should be placed behind a firewall, with only necessary ports exposed
- Default ports:
  - 8021 (Event Socket for Kazoo communication)
  - 16384-32768 UDP (RTP media)

## Maintenance

To check the health of your FreeSWITCH server:

```bash
# Connect to the FreeSWITCH CLI
fs_cli

# Show current calls
show calls

# Show detailed status
status

# Check memory usage
memstats

# Exit the CLI
/exit
```