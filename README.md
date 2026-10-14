# Self-Hosted Nextcloud with Cloudflare Tunnel & Telegram Notifications

A self-hosted cloud storage solution built with **Nextcloud, Docker, Cloudflare Quick Tunnel, and a Telegram bot**. Access files remotely through an HTTPS link, store data on a dedicated HDD, and automatically notify multiple Telegram users when the tunnel becomes available.

## Overview

This project demonstrates how to deploy a personal cloud storage server on Debian Linux and make it accessible remotely without configuring inbound router port forwarding.

The setup includes persistent storage, separate read-only and writable HDD locations, automated tunnel startup, and Telegram notifications.

### Features

- **Self-hosted cloud storage:** Run Nextcloud on your own Debian server.
- **Docker deployment:** Run Nextcloud and MariaDB in containers.
- **Remote access:** Expose Nextcloud through a Cloudflare Quick Tunnel.
- **Automatic startup:** Use systemd to start and supervise the tunnel service.
- **Telegram notifications:** Automatically send the public URL to multiple Telegram chat IDs.
- **External HDD storage:** Access existing files through a read-only mount.
- **Writable shared storage:** Give Nextcloud users a dedicated directory for uploading, downloading, and creating files.
- **Persistent data:** Store Nextcloud user files on the server's HDD.

## Architecture

```text
                  Remote Users
                       |
                       v
              Telegram Notification
                       ^
                       |
              Telegram Bot API
                       ^
                       |
Debian Server  <--- Cloudflare Quick Tunnel
                       |
                       v
              Nextcloud :8081
                       |
                       v
                  MariaDB
                       |
                       v
              Persistent HDD Storage
                 /           \
                /             \
        Server HDD          Shared Files
        Read-only           Read / Write
        Admin access        Nextcloud users
```

The Cloudflare tunnel forwards HTTPS requests to the local Nextcloud service at `http://127.0.0.1:8081`.

The tunnel URL is generated dynamically, so a new Quick Tunnel URL may be issued when the service restarts.

## Technology Stack

| Technology | Purpose |
|---|---|
| Debian Linux | Server operating system |
| Docker & Docker Compose | Container deployment |
| Nextcloud | Personal cloud storage |
| MariaDB | Database backend |
| Cloudflare Tunnel | Remote HTTPS access |
| `cloudflared` | Tunnel client |
| systemd | Service management and automatic startup |
| Telegram Bot API | Automated notifications |
| Bash & Python | Automation and configuration |

## 1. Deploy Nextcloud with Docker

Install Debian Linux, Docker Engine, and the Docker Compose plugin.

Create a project directory and configure the Nextcloud and MariaDB services using a Compose file.

```bash
sudo mkdir -p /opt/nextcloud
cd /opt/nextcloud
sudo nano compose.yaml
```

Configure the services with persistent database storage, Nextcloud storage, and the required environment variables. Use strong credentials stored securely rather than committing them to Git.

Start the deployment:

```bash
sudo docker compose -f /opt/nextcloud/compose.yaml up -d
```

Check the containers:

```bash
sudo docker ps
```

In this setup, Nextcloud is available locally through port `8081`.

## 2. Configure HDD Storage

The server uses a dedicated HDD mounted at:

```text
/srv/storage
```

Nextcloud's primary user-data directory is:

```text
/srv/storage/nextcloud-data
```

Existing HDD files are exposed through a read-only external storage mount, while a separate directory is writable.

Create the dedicated shared directory:

```bash
sudo mkdir -p /srv/storage/Shared-Files
sudo chown 33:33 /srv/storage/Shared-Files
sudo chmod 2770 /srv/storage/Shared-Files
```

Here, UID/GID `33:33` corresponds to `www-data` in the configured container. The setgid permission helps new entries inherit the directory's group.

Add the following bind mounts to the Nextcloud service in `compose.yaml`:

```yaml
volumes:
  - /srv/storage/nextcloud-data:/var/www/html/data
  - /srv/storage:/mnt/server-hdd:ro
  - /srv/storage/Shared-Files:/mnt/shared-files
```

**Important:** Preserve existing data and permissions. Do not recursively change ownership or permissions across the entire HDD.

Apply the Compose changes:

```bash
sudo docker compose -f /opt/nextcloud/compose.yaml up -d nextcloud
```

## 3. Configure External Storage in Nextcloud

Enable the External Storage Support app:

```bash
sudo docker exec -u www-data nextcloud-nextcloud-1 \
  php occ app:enable files_external
```

Create the read-only HDD mount:

```bash
sudo docker exec -u www-data nextcloud-nextcloud-1 \
  php occ files_external:create \
  "Server HDD" local null::null \
  -c datadir=/mnt/server-hdd
```

Create the writable shared mount:

```bash
sudo docker exec -u www-data nextcloud-nextcloud-1 \
  php occ files_external:create \
  "Shared Files" local null::null \
  -c datadir=/mnt/shared-files
```

Check the configured mounts:

```bash
sudo docker exec -u www-data nextcloud-nextcloud-1 \
  php occ files_external:list
```

Verify the storage configuration:

```bash
sudo docker exec -u www-data nextcloud-nextcloud-1 \
  php occ files_external:verify 1

sudo docker exec -u www-data nextcloud-nextcloud-1 \
  php occ files_external:verify 2
```

The mount IDs shown above correspond to the original deployment; verify the actual IDs before running commands on another installation.

Restrict the read-only `Server HDD` mount to the administrator and keep `Shared Files` available to the intended Nextcloud users.

The host directory's permissions and the container's bind-mount settings provide the underlying access boundaries. Nextcloud mount visibility alone is not a substitute for operating-system permissions.

## 4. Configure the Cloudflare Quick Tunnel

Install `cloudflared` on the server, then use a Quick Tunnel to expose the local Nextcloud service:

```bash
cloudflared tunnel --url http://127.0.0.1:8081
```

Cloudflare generates a temporary HTTPS URL ending in `trycloudflare.com`.

Quick Tunnels are useful for testing and personal projects, but they do not guarantee uptime or a permanent URL. Restarting the tunnel can generate a different address.

For a stable production deployment, consider a named Cloudflare Tunnel and a domain you control.

## 5. Automate Tunnel Startup with systemd

The project uses a custom script:

```text
/usr/local/sbin/nextcloud-quick-tunnel
```

and a systemd service:

```text
/etc/systemd/system/nextcloud-quick-tunnel.service
```

The script:

1. Waits for Nextcloud to become ready.
2. Starts `cloudflared`.
3. Detects the generated public URL.
4. Updates Nextcloud's trusted-domain configuration.
5. Sends the URL to the configured Telegram recipients.
6. Retries after the tunnel exits.

Enable the service at boot:

```bash
sudo systemctl enable nextcloud-quick-tunnel.service
sudo systemctl start nextcloud-quick-tunnel.service
```

Check the status and logs:

```bash
sudo systemctl status nextcloud-quick-tunnel.service --no-pager

sudo journalctl -u nextcloud-quick-tunnel.service \
  -n 60 --no-pager
```

Restart it when a fresh tunnel URL is needed:

```bash
sudo systemctl restart nextcloud-quick-tunnel.service
```

## 6. Configure Telegram Notifications

Create a Telegram bot using [@BotFather](https://t.me/BotFather), then obtain the chat IDs of the intended recipients.

Each recipient must open a private conversation with the bot and press **Start** before the bot can send them private messages.

The configuration is stored outside the Git repository:

```text
/etc/cloudflared/telegram.env
```

Example structure — use your own values:

```bash
TELEGRAM_BOT_TOKEN="YOUR_BOT_TOKEN"
TELEGRAM_CHAT_ID="YOUR_PRIMARY_CHAT_ID"
```

Protect the configuration file:

```bash
sudo chown root:root /etc/cloudflared/telegram.env
sudo chmod 600 /etc/cloudflared/telegram.env
```

The notification script loops through the primary chat ID and any additional recipients. For example:

```bash
for chat_id in "$TELEGRAM_CHAT_ID" "YOUR_FRIEND_CHAT_ID"; do
    /usr/bin/curl -fsS --max-time 15 \
        -X POST \
        "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
        --data-urlencode "chat_id=${chat_id}" \
        --data-urlencode "text=${message}" \
        >/dev/null || echo "Telegram notification failed."
done
```

Replace `YOUR_FRIEND_CHAT_ID` with the intended recipient's numeric chat ID. Keep the bot token private.

When Nextcloud is ready and the tunnel URL is detected, the script sends a message similar to:

```text
Your Nextcloud tunnel is ready:
https://YOUR-TEMPORARY-TUNNEL.trycloudflare.com
```

## 7. Verify the Writable Shared Directory

Test write access from inside the Nextcloud container:

```bash
sudo docker exec -u www-data nextcloud-nextcloud-1 \
  touch /mnt/shared-files/server-write-test.txt
```

Confirm that the file exists on the host:

```bash
sudo ls -la /srv/storage/Shared-Files
```

Then log in to Nextcloud with a normal user account and verify that you can create a folder and upload a small file under **Files → Shared Files**.

Remove the test file afterward:

```bash
sudo rm /srv/storage/Shared-Files/server-write-test.txt
```

Only remove this specific test file after confirming it exists.

## 8. Reboot Test

A reboot test confirmed that the systemd service starts again, Nextcloud becomes ready, a new Quick Tunnel is established, and a notification is delivered through Telegram.

The final workflow is:

```text
Server starts
     |
     v
Nextcloud becomes ready
     |
     v
Cloudflare Quick Tunnel starts
     |
     v
Public HTTPS URL is detected
     |
     v
Nextcloud trusted domain is updated
     |
     v
Telegram sends the link to recipients
```

## Security Considerations

- Do not commit Telegram bot tokens, database passwords, or secret environment files.
- Do not expose Docker, MariaDB, or SSH to the public internet unnecessarily.
- Keep existing HDD data read-only inside the Nextcloud container where appropriate.
- Grant normal users access only to the storage locations they need.
- Use strong, unique passwords and enable multi-factor authentication where available.
- Keep Debian, Docker images, Nextcloud, and its apps updated.
- Maintain backups of both the Nextcloud database and user data.
- Treat Cloudflare Quick Tunnels as a convenient testing and personal-use option, not a guaranteed production hosting service.
- Remember that a public URL is reachable by anyone who has it unless access is otherwise restricted by authentication or additional controls.

## Project Structure

A suggested repository layout:

```text
nextcloud-cloudflare-telegram/
├── README.md
├── LICENSE
├── .gitignore
└── docs/
    └── images/
        ├── architecture.png
        ├── nextcloud-files.png
        └── telegram-notification.png
```

Add screenshots of the actual Nextcloud interface, terminal verification, systemd status, and Telegram notification. Remove or blur personal information, private URLs, IP addresses, tokens, and other sensitive data before publishing.

## Troubleshooting

| Problem | Checks |
|---|---|
| Nextcloud does not load | `sudo docker ps` and `sudo docker logs --tail 100 nextcloud-nextcloud-1` |
| Tunnel does not start | Check `systemctl status` and the service journal |
| Telegram message is missing | Confirm the recipient started the bot and verify the chat ID |
| Shared Files cannot be written | Check host permissions, bind mounts, and external-storage verification |
| Public URL changed | Use the newest URL delivered by Telegram |

## What I Learned

- Deploying a multi-container application on Debian with Docker Compose.
- Integrating external HDD storage into Nextcloud.
- Separating read-only data from writable shared storage.
- Automating service startup and recovery with systemd.
- Using Cloudflare Tunnel for remote HTTPS access.
- Integrating the Telegram Bot API with a Bash automation workflow.
- Verifying file permissions and validating the deployment after a reboot.

## Conclusion

This project combines self-hosted cloud storage, persistent HDD storage, remote access, and automated notifications into one practical home-server workflow.

It demonstrates how Linux administration, containerization, storage permissions, networking, and scripting can work together to build a useful personal cloud.

**Built on Debian Linux. Powered by Nextcloud, Docker, Cloudflare Tunnel, and Telegram.**

---

**Author:** sak7tra

**Topics:** `nextcloud` `self-hosting` `cloudflare-tunnel` `telegram-bot` `docker` `debian` `linux` `homelab` `automation` `privacy`
