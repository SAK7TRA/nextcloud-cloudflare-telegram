# Sec Horizon Cloud — Automated Email Notifications for Cloudflare Quick Tunnel

## Overview

Sec Horizon Cloud is a self-hosted Nextcloud service running on a Debian Linux home server (`sakserv`). It is exposed through a Cloudflare Quick Tunnel. Because Quick Tunnel URLs can change when a new tunnel is created, users need a way to receive the latest access link.

The existing Bash-based tunnel manager already waited for Nextcloud readiness, started `cloudflared`, extracted the generated `trycloudflare.com` URL, updated Nextcloud's `trusted_domains`, and sent Telegram notifications. This improvement adds **email notifications** as an additional delivery channel, using Gmail SMTP and `msmtp`, while retaining the existing Telegram workflow.

> **Status:** Email notification delivery was verified with a real Gmail recipient. Telegram remained configured but its requests timed out during the recorded test. Email and Telegram still execute sequentially in the tunnel script; parallel notification execution is a planned improvement, not a completed one.

## Problem statement

- Cloudflare Quick Tunnel creates an access URL that may change after a restart.
- Telegram was the only automated way to distribute the updated link.
- Telegram requests timed out during testing, delaying downstream notifications.
- The system needed a second channel, multiple recipients, and a way to diagnose delivery failures without disrupting Nextcloud.

## Environment and components

| Component | Role |
| --- | --- |
| Debian 13 (`sakserv`) | Self-hosted Linux server |
| Nextcloud (Docker Compose) | Cloud application, available locally at `http://127.0.0.1:8081` |
| `cloudflared` | Generates a Cloudflare Quick Tunnel URL |
| systemd | Runs `nextcloud-quick-tunnel.service` |
| Bash | Tunnel lifecycle and notification automation |
| `msmtp` | SMTP client for outbound email |
| Gmail SMTP | Authenticated TLS email relay, `smtp.gmail.com:465` |
| Telegram Bot API | Existing notification channel |

## Architecture and event flow

```text
Nextcloud container (127.0.0.1:8081)
         |
         v
nextcloud-quick-tunnel.service
         |
         +--> Check Nextcloud readiness
         +--> Start cloudflared Quick Tunnel
         +--> Extract https://<generated>.trycloudflare.com
         +--> Update Nextcloud trusted_domains
         |
         +--> Telegram Bot API (existing notification loop)
         |
         +--> sechorizon-send-email (new helper)
                  |
                  +--> Read email-recipients.txt
                  +--> Build a separate email per recipient
                  +--> msmtp --> Gmail SMTP (TLS) --> recipient mailbox
```

**Important:** In the implementation tested here, the Telegram loop completes before the email helper is started in the background. Five Telegram timeouts caused approximately 75 seconds of delay. Running the two notification channels independently is a recommended next change.

## Files and responsibilities

| Path | Purpose |
| --- | --- |
| `/usr/local/sbin/nextcloud-quick-tunnel` | Main tunnel management script |
| `/usr/local/sbin/sechorizon-send-email` | Validates a Quick Tunnel URL and emails recipients |
| `/etc/cloudflared/email-recipients.txt` | One recipient address per line |
| `/etc/msmtprc` | SMTP account configuration |
| `/etc/cloudflared/gmail-app-password` | Restricted file holding the Gmail App Password |
| `/etc/cloudflared/telegram.env` | Existing Telegram notification configuration |
| `/etc/systemd/system/nextcloud-quick-tunnel.service` | systemd unit for the tunnel |

**Security:** Do not commit real email addresses, app passwords, bot tokens, environment files, or unredacted logs to GitHub. The commands and snippets below deliberately omit secrets.

## 1. SMTP configuration

`msmtp` was configured to use Gmail over implicit TLS on port 465. The authenticated sender was a dedicated Gmail account. The example below uses a placeholder sender for public documentation:

```ini
# /etc/msmtprc — sanitized example
defaults
auth on
tls on
tls_starttls off
tls_trust_file /etc/ssl/certs/ca-certificates.crt

account sechorizon
host smtp.gmail.com
port 465
from SENDER@gmail.com
user SENDER@gmail.com
passwordeval "cat /etc/cloudflared/gmail-app-password"

account default : sechorizon
```

The Gmail App Password is stored separately with root-only access. The SMTP configuration is also restricted. No password is embedded in the Bash scripts.

## 2. Multi-recipient configuration

The helper reads `/etc/cloudflared/email-recipients.txt` at runtime. Each line contains one email address:

```text
recipient-one@example.com
recipient-two@example.com
```

The script skips blank lines and comments, validates the address format, and sends a separate message to each recipient so recipients do not see one another's addresses.

Example file permissions:

```bash
sudo chown root:root /etc/cloudflared/email-recipients.txt
sudo chmod 600 /etc/cloudflared/email-recipients.txt
```

Adding a recipient requires editing the file, not restarting the tunnel. New recipients receive subsequent notifications; a manual helper test can send the current URL immediately.

## 3. Email notification helper

The helper is located at `/usr/local/sbin/sechorizon-send-email`. Its responsibilities are:

1. Require exactly one URL matching `https://<subdomain>.trycloudflare.com`.
2. Verify that the recipients file and SMTP configuration are readable.
3. Read and validate recipient entries.
4. Generate standard email headers, including `From`, `To`, `Subject`, `Date`, `MIME-Version`, and `Content-Type`.
5. Send via `msmtp --file=/etc/msmtprc --account=sechorizon`, with a timeout per recipient.
6. Report submitted and failed messages.

A representative invocation is:

```bash
sudo /usr/local/sbin/sechorizon-send-email \
  'https://example.trycloudflare.com'
```

`example.trycloudflare.com` is a **test placeholder**, not an actual Sec Horizon access URL.

The email subject used during testing was:

```text
Sec Horizon - Your Cloud Is Ready
```

The message includes the newly generated tunnel URL and instructs recipients to use their existing Nextcloud credentials.

## 4. Integration with the tunnel manager

The main script calls the email helper only after Nextcloud readiness and the trusted-domain update succeed. The existing Telegram loop is preserved.

The following excerpt illustrates the **tested sequential ordering** (Telegram first, email afterward):

```bash
if [[ "$ready" -eq 1 ]]; then
    # Existing Telegram notification loop executes here.

    if [[ -x /usr/local/sbin/sechorizon-send-email ]]; then
        (
            /usr/bin/timeout 90 \
                /usr/local/sbin/sechorizon-send-email "$url" \
                || echo "Sec Horizon email notification failed."
        ) &
    fi
fi
```

The email helper is launched in the background after Telegram finishes, preventing its own SMTP processing from blocking the main log-reading loop at that point. This does **not** eliminate the preceding Telegram timeout delay.

## 5. Troubleshooting and validation

### A. Script execution failure: `203/EXEC`

After editing the tunnel script, systemd reported an `Exec format error`. Byte inspection revealed a newline **before** the `#!/bin/bash` shebang. Linux requires the shebang at the beginning of an executable script.

```bash
sudo head -c 12 /usr/local/sbin/nextcloud-quick-tunnel | od -An -tx1
sudo bash -n /usr/local/sbin/nextcloud-quick-tunnel
```

After correcting the leading newline, systemd successfully started the service and `cloudflared` generated a new URL. The same leading-newline problem was also corrected in the email helper.

### B. Telegram timeouts

Five Telegram notification attempts failed, each after approximately 15 seconds. The email helper then started and successfully submitted the notification. This established that the email integration worked but was delayed by the sequential notification flow.

### C. SMTP accepted, but no message in Inbox

`msmtp` reported successful submission, including an SMTP `250 2.0.0 OK` response. The recipient initially could not find the notification in the Inbox. A search across Gmail folders revealed that the messages were being classified as **Spam**.

The recipient marked the message **Not Spam** and configured a Gmail filter for the trusted sender with **Never send it to Spam** and **Mark as important**. A subsequent manual test was reported successful.

An SMTP `250` response confirms acceptance for processing, **not guaranteed Inbox placement**.

### D. SMTP log file permission error

An optional `--logfile=/var/log/sechorizon-email.log` parameter produced a permission error even though the SMTP transaction succeeded. The optional logfile argument was removed, leaving notification status available through command output and the systemd journal.

### E. Useful verification commands

```bash
# Bash syntax
sudo bash -n /usr/local/sbin/nextcloud-quick-tunnel
sudo bash -n /usr/local/sbin/sechorizon-send-email

# Service health
sudo systemctl status nextcloud-quick-tunnel.service --no-pager

# Notification events (avoid publishing unredacted logs)
sudo journalctl -u nextcloud-quick-tunnel.service \
  --since '10 minutes ago' --no-pager \
  | grep -E 'Email notification|Telegram notification|trusted domain'
```

**Operational caution:** Process listings and unredacted systemd logs can expose secrets if a command includes a bot token in its URL. Rotate exposed tokens and redact logs before publishing.

## 6. Results

- The Cloudflare Quick Tunnel service recovered from the shebang-related execution failure.
- Nextcloud readiness and trusted-domain updating remained functional.
- The existing Telegram notification configuration was preserved; delivery attempts timed out during testing.
- The email helper submitted messages using the configured Gmail SMTP sender.
- The recipient located the automatic notifications in Spam and confirmed the actual tunnel URL was present.
- A Gmail filter was configured to reduce future spam classification.
- Additional recipients can be managed through a simple text file.

## 7. Future improvements

- **Independent notification dispatch:** Start Telegram and email concurrently so Telegram timeouts cannot delay email submission.
- **Safer credential handling:** Avoid embedding the Telegram bot token in command-line arguments visible in process listings; rotate any exposed token.
- **Structured observability:** Track per-channel notification outcomes without recording credentials.
- **Retry/backoff:** Add bounded retries for temporary SMTP or API failures, avoiding duplicate notifications.
- **Long-term tunnel stability:** Consider a named Cloudflare Tunnel if a stable URL and stronger uptime expectations become necessary; Quick Tunnels do not provide an uptime guarantee.

## Lessons learned

This change was more than adding a mail command. The practical work involved shell execution semantics, systemd diagnostics, SMTP authentication and TLS, per-recipient configuration, message delivery verification, and spam filtering. Most importantly, **successful SMTP submission and successful Inbox delivery are different states**, and notification reliability depends on both.

---

*Project: Sec Horizon Cloud · Platform: Debian Linux, Nextcloud, Docker, Cloudflare Quick Tunnel, Bash, systemd, msmtp, Gmail SMTP.*
