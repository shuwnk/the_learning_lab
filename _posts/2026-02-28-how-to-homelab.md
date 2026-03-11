---
layout: post
title: How To - How I Ditched Google Drive and Built My Own Cloud Server
author: MHD
---

# How I Ditched Google Drive and Built My Own Cloud Server

Hey! So today we're going to do something fun — we're going to take an old PC, install a few things on it, and turn it into your very own personal cloud server. Think Google Drive, but it's sitting in your house, you own all the data, and nobody can snoop on your files or charge you monthly fees.

By the end of this guide, you'll have:

- A **Nextcloud** server (basically self-hosted Google Drive with file sync, calendar, contacts, and even collaborative document editing)
- **HTTPS encryption** so your connection is secure
- **Remote access via Tailscale** so you can reach your files from anywhere — your phone, your laptop at a coffee shop, wherever
- **Automated backups** so you don't lose everything if a drive dies
- Desktop and mobile apps syncing your files automatically

And the best part? You don't need a domain name. You don't need to mess with port forwarding. Tailscale handles all of that for free.

Let's get into it.

---

## What You'll Need

Nothing fancy. I'm using an **HP ProDesk 405 G4 Desktop Mini** that was collecting dust, but any old PC or laptop works. Here's the minimum:

- A PC with Ubuntu installed (I'm on Ubuntu 24.04)
- At least 4GB of RAM (8GB+ is better
- A hard drive with enough space for your files
- An internet connection

That's it. No special hardware. No cloud subscription.

---

## Step 1: Update Everything and Set Up SSH

First things first — open a terminal on your PC (`Ctrl+Alt+T`) and make sure the system is up to date:

```bash
sudo apt update && sudo apt upgrade -y
```

Now let's install SSH. This lets you :manage the server from another computer without having to walk over and sit in front of it every time. Trust me, you'll want this.

```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh
sudo systemctl start ssh
```

To find your server's IP address (you'll need this a lot):

```bash
ip a | grep "inet " | grep -v 127.0.0.1
```

In my case, my server's IP turned out to be `10.0.0.12`. Yours will be different — write it down. From now on, whenever I say `10.0.0.12`, substitute your own IP.

You can now SSH into your server from another machine:

```bash
ssh your-username@10.0.0.12
```

---

## Step 2: Give Your Server a Static IP

Right now your router is assigning your server an IP address via DHCP, which means it could change at any time. That's no good for a server — we need a fixed address.

Find your netplan config file:

```bash
ls /etc/netplan/
```

Edit it:

```bash
sudo nano /etc/netplan/$(ls /etc/netplan/)
```

Replace whatever's in there with something like this (adjust for your network):

```yaml
network:
  version: 2
  ethernets:
    eno1:   # Your interface name — check with: ip link
      dhcp4: no
      addresses:
        - 10.0.0.12/24
      routes:
        - to: default
          via: 10.0.0.1      # Your router's IP
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Apply it:

```bash
sudo netplan apply
```

---

## Step 3: Set Up the Firewall

Ubuntu comes with `ufw` (Uncomplicated Firewall), and we should turn it on. We'll open only the ports we actually need:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 443/tcp
sudo ufw allow 8080/tcp
sudo ufw enable
```

Type `y` to confirm. Now your server is only accepting connections on the ports we explicitly allowed. Everything else is blocked.

---

## Step 4: Prepare a Data Drive

If you have a separate hard drive for storing your files (recommended), let's set it up. If you're just using your OS drive, skip ahead — just run `sudo mkdir -p /mnt/data`.

First, see what drives you have:

```bash
lsblk
```

Find the drive you want to use (NOT your OS drive), then format and mount it:

```bash
sudo mkfs.ext4 /dev/sdb1         # Replace with your actual drive
sudo mkdir -p /mnt/data
sudo mount /dev/sdb1 /mnt/data
```

We need this to survive reboots. Get the drive's UUID:

```bash
sudo blkid /dev/sdb1
```

Then add it to `/etc/fstab` so it mounts automatically:

```bash
echo 'UUID=YOUR-UUID-HERE /mnt/data ext4 defaults 0 2' | sudo tee -a /etc/fstab
```

And set the permissions:

```bash
sudo chown -R $USER:$USER /mnt/data
```

---

## Step 5: Install Docker

We're going to run everything in Docker containers. Why? Because it keeps things clean and isolated. Nextcloud, the database, the cache, the reverse proxy — they each get their own little sandbox. And if something breaks, you just recreate the container. No reinstalling packages, no dependency hell.

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
```

**Important:** Log out and log back in after this. The group change won't take effect until you do.

Then verify:

```bash
docker --version
docker compose version
```

Create a directory for our project:

```bash
mkdir -p ~/homelab && cd ~/homelab
```

This is where all our config files will live.

---

## Step 6: Deploy Nextcloud

Alright, this is the big one. We're going to create a single file — `docker-compose.yml` — that defines our entire stack: a MariaDB database, a Redis cache (for speed), Nextcloud itself, and a Caddy reverse proxy for HTTPS.

But first, we need to create the Caddyfile (we'll fill in the HTTPS certs later — for now, just create the file so Docker doesn't complain):

```bash
touch ~/homelab/Caddyfile
```

Now create the compose file:

```bash
nano ~/homelab/docker-compose.yml
```

Paste this in. **Change the passwords** — don't use the defaults:

```yaml
services:
  # --- Database ---
  # MariaDB stores all of Nextcloud's metadata: users, shares,
  # file references, calendar entries, contacts, etc.
  db:
    image: mariadb:11
    container_name: nextcloud-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ChangeThisRootPassword123!
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: ChangeThisDBPassword456!
    volumes:
      - ./db-data:/var/lib/mysql
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW

  # --- Redis ---
  # An in-memory cache that makes Nextcloud noticeably faster.
  # File listings, session data, and locks all go through here.
  redis:
    image: redis:7-alpine
    container_name: nextcloud-redis
    restart: unless-stopped

  # --- Nextcloud ---
  # The main application. This is your Google Drive replacement.
  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud-app
    restart: unless-stopped
    depends_on:
      - db
      - redis
    ports:
      - "8080:80"
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: ChangeThisDBPassword456!
      REDIS_HOST: redis
      NEXTCLOUD_TRUSTED_DOMAINS: "10.0.0.12 mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net"
      OVERWRITEPROTOCOL: https
      PHP_MEMORY_LIMIT: 1G
      PHP_UPLOAD_LIMIT: 16G
    volumes:
      - ./nextcloud-app:/var/www/html
      - /mnt/data/nextcloud:/var/www/html/data

  # --- Caddy ---
  # A lightweight reverse proxy that handles HTTPS for us.
  # It sits in front of Nextcloud and encrypts all traffic.
  caddy:
    image: caddy:2-alpine
    container_name: caddy
    restart: unless-stopped
    ports:
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - /etc/caddy/certs:/etc/caddy/certs:ro
      - ./caddy-data:/data
```

Save with `Ctrl+O`, Enter, `Ctrl+X`.

Now start it up:

```bash
cd ~/homelab && docker compose up -d
```

Give it about 60 seconds to initialize, then check that everything is running:

```bash
docker compose ps
```

You should see all four containers with status `Up`.

Open a browser and go to `http://10.0.0.12:8080`. You'll see the Nextcloud setup wizard. Create an admin account with a strong password, and click Install. That's it — Nextcloud is running.

### A Quick Performance Tweak

By default, Nextcloud uses AJAX for background tasks, which is slow. Let's switch it to cron:

```bash
docker exec -u www-data nextcloud-app php occ background:cron

(crontab -l 2>/dev/null; echo "*/5 * * * * docker exec -u www-data nextcloud-app php -f /var/www/html/cron.php") | crontab -
```

This runs background jobs every 5 minutes, which keeps things snappy.

---

## Step 7: Set Up Tailscale for Remote Access

Okay so right now your Nextcloud only works on your local network. Step outside your house and you can't reach it. We're going to fix that with Tailscale.

Tailscale is a mesh VPN. It connects your devices together in a private, encrypted network — no matter where they are. Your server at home, your phone at work, your laptop at a coffee shop — they can all talk to each other securely. And unlike traditional VPNs or port forwarding, you don't need to open any ports on your router.

The killer feature for us? **Tailscale gives you free HTTPS certificates.** No domain name needed. No Let's Encrypt setup. Just a clean, trusted HTTPS URL.

### Install Tailscale on the Server

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

It'll print a URL — open it in your browser and log in to authorize the machine.

Check your machine name:

```bash
tailscale status
```

Mine showed up as `mhd-hp-prodesk-405-g4-desktop-mini` on the tailnet `tail381ffc.ts.net`. That means my full hostname is:

```
mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net
```

### Get an HTTPS Certificate

First, enable HTTPS in the Tailscale admin console:

1. Go to **login.tailscale.com/admin/dns**
2. Click **Enable HTTPS** at the bottom

Then generate your certificate:

```bash
sudo tailscale cert mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net
```

This creates two files — a `.crt` and a `.key`. Move them somewhere permanent:

```bash
sudo mkdir -p /etc/caddy/certs
sudo mv mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net.crt /etc/caddy/certs/
sudo mv mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net.key /etc/caddy/certs/
```

### Configure Caddy

Now let's tell Caddy to use these certificates. Edit the Caddyfile we created earlier:

```bash
nano ~/homelab/Caddyfile
```

Replace the contents with:

```
mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net {
    tls /etc/caddy/certs/mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net.crt /etc/caddy/certs/mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net.key

    reverse_proxy nextcloud:80

    header {
        Strict-Transport-Security "max-age=31536000;"
    }

    redir /.well-known/carddav /remote.php/dav 301
    redir /.well-known/caldav /remote.php/dav 301
}
```

What this does: any request coming in for our Tailscale hostname gets encrypted with our TLS certificate, then forwarded to the Nextcloud container on port 80. The `carddav` and `caldav` redirects are so calendar and contact sync works properly.

Restart everything:

```bash
cd ~/homelab && docker compose up -d
```

### Tell Nextcloud About HTTPS

Nextcloud needs to know it's behind a reverse proxy and that it should use HTTPS:

```bash
docker exec -u www-data nextcloud-app php occ config:system:set trusted_domains 1 --value="mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net"

docker exec -u www-data nextcloud-app php occ config:system:set overwriteprotocol --value="https"

docker exec -u www-data nextcloud-app php occ config:system:set overwrite.cli.url --value="https://mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net"

docker exec -u www-data nextcloud-app php occ config:system:set trusted_proxies 0 --value="172.16.0.0/12"
```

### Install Tailscale on Your Other Devices

Install the Tailscale app on your phone (App Store or Play Store) and on any computers you want to access your cloud from. Log in with the same account.

Now try it. Open your browser and go to:

```
https://mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net
```

You should see Nextcloud with a nice green padlock. Secure, encrypted, accessible from anywhere. That's your cloud.

### Auto-Renew the Certificate

Tailscale certs expire after a while, so let's set up automatic renewal:

```bash
(crontab -l 2>/dev/null; echo "0 0 1 * * sudo tailscale cert mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net && sudo cp mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net.* /etc/caddy/certs/ && docker restart caddy") | crontab -
```

This runs on the first of every month, grabs a fresh cert, copies it over, and restarts Caddy. Set it and forget it.

---

## Step 8: Install the Sync Clients

What good is a cloud if your files don't sync automatically? Nextcloud has native apps for everything.

### Desktop

Download the Nextcloud desktop client from **nextcloud.com/install/#install-clients**. When it asks for the server address, enter:

```
https://mhd-hp-prodesk-405-g4-desktop-mini.tail381ffc.ts.net
```

Log in, pick which folders to sync, and you're done. It works just like Google Drive or Dropbox — a folder on your computer that stays in sync with the server.

### Mobile

Install the **Nextcloud** app from the App Store or Play Store. Enter the same server address, log in, and go to **Settings → Auto Upload** to have your photos and videos backed up automatically.

---

## Step 9: Adding More Users

Nextcloud is multi-user out of the box. Want to give family members or roommates their own accounts? Each person gets their own private space.

From the command line:

```bash
docker exec -u www-data nextcloud-app php occ user:add --display-name="John" john
```

Or just go to the web UI: click your avatar in the top right → **Users** → **New user**.

You can set storage quotas per user too:

```bash
docker exec -u www-data nextcloud-app php occ user:setting john files quota "50 GB"
```

---

## Step 10: Backups (Don't Skip This)

I know backups aren't exciting, but listen — your hard drive will fail eventually. It's not a matter of if, it's when. Let's make sure you don't lose everything when it happens.

Create a backup script:

```bash
nano ~/homelab/backup.sh
```

```bash
#!/bin/bash
BACKUP_DIR="/mnt/data/backups/nextcloud"
DATE=$(date +%Y-%m-%d)

mkdir -p "$BACKUP_DIR"

echo "=== Backing up Nextcloud ==="

# Put Nextcloud in maintenance mode so nothing changes mid-backup
docker exec -u www-data nextcloud-app php occ maintenance:mode --on

# Dump the database
docker exec nextcloud-db mariadb-dump \
  -u nextcloud -pChangeThisDBPassword456! nextcloud \
  > "$BACKUP_DIR/db-$DATE.sql"

# Archive the app config
tar czf "$BACKUP_DIR/app-config-$DATE.tar.gz" \
  -C ~/homelab nextcloud-app/

# Turn off maintenance mode
docker exec -u www-data nextcloud-app php occ maintenance:mode --off

# Clean up — keep only the last 7 days
ls -t "$BACKUP_DIR"/db-*.sql | tail -n +8 | xargs -r rm
ls -t "$BACKUP_DIR"/app-config-*.tar.gz | tail -n +8 | xargs -r rm

echo "=== Backup complete ==="
```

Make it executable and schedule it to run every night at 3 AM:

```bash
chmod +x ~/homelab/backup.sh

(crontab -l 2>/dev/null; echo "0 3 * * * /home/$USER/homelab/backup.sh >> /home/$USER/homelab/backup.log 2>&1") | crontab -
```

### Offsite Backups (Highly Recommended)

Local backups protect against software failures, but not against things like fires or theft. For true safety, sync your backups to the cloud with rclone. Backblaze B2 costs about $6 per terabyte per month.

```bash
sudo apt install rclone -y
rclone config
```

Follow the interactive setup, then add this line to the end of your backup script:

```bash
rclone sync /mnt/data/backups/nextcloud remote:your-bucket/nextcloud-backups
```

---

## Bonus: Extra Apps Worth Adding

Now that you have the infrastructure running, adding more services is trivially easy. Here are my favorites:

### Immich — Replace Google Photos

A beautiful self-hosted photo manager with AI-powered facial recognition, timeline view, and mobile auto-upload. Check out **immich.app** for the docker-compose setup.

### Portainer — Manage Docker Visually

If you'd rather click buttons than type terminal commands to manage your containers:

```bash
docker run -d \
  --name portainer \
  --restart unless-stopped \
  -p 9443:9443 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Then go to `https://10.0.0.12:9443`.

### Collabora — Google Docs-like Editing

Want to edit documents right inside Nextcloud? Add this to your `docker-compose.yml`:

```yaml
  collabora:
    image: collabora/code
    container_name: collabora
    restart: unless-stopped
    ports:
      - "9980:9980"
    environment:
      - domain=10\\.0\\.0\\.12
      - extra_params=--o:ssl.enable=false
    cap_add:
      - MKNOD
```

Then in Nextcloud, install the **Nextcloud Office** app and point it to `http://10.0.0.12:9980`.

---

## Quick Reference

Here are the commands you'll use most often:

| What | Command |
|------|---------|
| Start everything | `cd ~/homelab && docker compose up -d` |
| Stop everything | `cd ~/homelab && docker compose down` |
| Check status | `docker compose ps` |
| View logs | `docker compose logs -f nextcloud` |
| Update containers | `docker compose pull && docker compose up -d` |
| Nextcloud CLI | `docker exec -u www-data nextcloud-app php occ` |
| Add a user | `docker exec -u www-data nextcloud-app php occ user:add username` |
| Reset a password | `docker exec -it -u www-data nextcloud-app php occ user:resetpassword username` |
| Check disk space | `df -h /mnt/data` |
| Tailscale status | `tailscale status` |
| Run a backup | `~/homelab/backup.sh` |

---

## Troubleshooting

**Can't access Nextcloud in the browser?**

```bash
docker compose ps              # Are the containers actually running?
docker compose logs nextcloud  # Check for errors
sudo ufw status                # Is the port open?
```

**"Access through untrusted domain" error?**

You're accessing Nextcloud from an IP or hostname it doesn't recognize. Add it:

```bash
docker exec -u www-data nextcloud-app php occ config:system:set trusted_domains 2 --value="your-new-ip"
```

**Stuck in maintenance mode?**

```bash
docker exec -u www-data nextcloud-app php occ maintenance:mode --off
```

If you get permission errors, fix the data directory ownership:

```bash
sudo chown -R www-data:www-data /mnt/data/nextcloud
```

**Slow performance?**

Make sure Redis caching is wired up:

```bash
docker exec -u www-data nextcloud-app php occ config:system:set memcache.local --value="\\OC\\Memcache\\APCu"
docker exec -u www-data nextcloud-app php occ config:system:set memcache.locking --value="\\OC\\Memcache\\Redis"
docker exec -u www-data nextcloud-app php occ config:system:set redis host --value="redis"
docker exec -u www-data nextcloud-app php occ config:system:set redis port --value="6379" --type=integer
```

---

## Wrapping Up

That's it. You now have your own personal cloud server sitting in your house. Your files are encrypted in transit, synced across all your devices, backed up automatically, and accessible from anywhere in the world through Tailscale.

No monthly fees. No storage limits (besides your hard drive). No company scanning your files. Just your data, on your hardware, under your control.

If you want to take it further, look into adding Immich for photos, Paperless-ngx for document management, or even Vaultwarden for a self-hosted password manager. Once you have the homelab running, the possibilities are endless.

Happy self-hosting!
