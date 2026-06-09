# Bulletproof Android Automation: Encrypted Backups with Restic, Termux, Tasker, and Backblaze B2

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Standard cloud backup apps for Android are bloated, eat system resources, and track your data. This repository contains an enterprise-grade, end-to-end encrypted, fully automated backup pipeline for Android power users. 

By leveraging **Restic** for snapshot deduplication, **Termux** for local runtime execution, and **Tasker** for scheduling, your device will securely push backups directly to **Backblaze B2** while you sleep—running natively on your phone without requiring root.

---

## 🛠️ Prerequisites

Before starting, ensure your Android device has the following applications installed (prefer F-Droid builds over Google Play for Termux extensions):

1. **Termux** (The terminal emulator environment)
2. **Termux:Tasker** plugin (Allows Tasker to invoke scripts inside Termux)
3. **Termux:API** app + package (Required for background native notifications)
4. **Tasker** (The automation engine)
5. A free **Backblaze B2 Account**

---

## 🏗️ Step-by-Step Architecture

### Step 1: Provision the Backblaze Vault
1. Log into your Backblaze account, go to **B2 Cloud Storage > Buckets**, and click **Create a Bucket**.
2. Give it a globally unique name (e.g., `yourname-phone-backup`). Mark it **Private**. Keep Default Encryption *disabled* (Restic handles state-of-the-art encryption locally before upload).
3. Go to **Application Keys** and click **Add a New Application Key**. 
4. Restrict its scope to your backup bucket with **Read and Write** access.
5. ⚠️ **CRITICAL WARNING:** Copy both the `keyID` and `applicationKey` immediately. The application key will self-destruct from the dashboard view the second you reload the page.

### Step 2: Establish Termux File Hygiene
Launch Termux and run the following sequential commands to update your architecture, unlock system storage access, and build an isolated configuration hierarchy:

```bash
# Update core packages and install Restic + Termux API
pkg update && pkg upgrade -y
pkg install restic termux-api nano -y

# Grant Termux permission to read your Android internal storage
termux-setup-storage

# Construct a clean config, script, and logging ecosystem
mkdir -p ~/.config/backup/scripts
mkdir -p ~/.config/backup/logs
```

### Step 3: Secure Your Environmental Credentials
To keep your infrastructure secure, we separate the raw authentication keys from the execution scripts. If you ever accidentally show off your backup script, your cloud credentials remain uncompromised.

Create your secret vault file:
```bash
nano ~/.config/backup/b2-credentials.conf
```

Paste the following variables, inserting your Backblaze strings:
```bash
# Backblaze B2 Authentication Variables
export B2_ACCOUNT_ID='your_alphanumeric_key_id'
export B2_ACCOUNT_KEY='your_massive_random_application_key'
```
> ⚠️ **THE BASH DOUBLE-QUOTE TRAP:** You **must** wrap your keys in **single quotes (`'`)**, not double quotes. Backblaze application keys often contain special characters like `$`. If you use double quotes, Bash will interpret the string as an empty variable extension, silently mutilating your key and throwing a frustrating `401 Unauthorized` error.

Protect your encryption password file (replace `your_super_secure_backup_password` with a real password):
```bash
echo "your_super_secure_backup_password" > ~/.config/backup/restic-password
chmod 600 ~/.config/backup/restic-password
```

### Step 4: Deploy the Production Script
Save this script as `~/.config/backup/scripts/phone-backup.sh`. It features extreme error safety handles (`set -Eeuo pipefail`), handles Android wake locks to prevent CPU throttling, and triggers native device notifications on status changes.

```bash
#!/data/data/com.termux/files/usr/bin/bash
# Hardened Shell Flags: Fail fast on errors, unassigned variables, or broken pipelines
set -Eeuo pipefail

# ==============================================================================
# ⚙️ USER CONFIGURATION BLOCK
# Change these values to match your specific setup
# ==============================================================================
BUCKET_NAME="your-b2-bucket-name"          # Your Backblaze B2 bucket name
DEVICE_TAG="my-phone"                      # Unique tag for this device (e.g., pixel7, s24)
BACKUP_TAG="downloads"                     # Identifier for this backup dataset
TARGET_DIR="$HOME/storage/shared/Download"   # Full path to the directory you want to backup
# ==============================================================================

# 1. Pull in the cloud authorization layer
source "$HOME/.config/backup/b2-credentials.conf"

# 2. Dynamic Target Repository Configurations
export RESTIC_REPOSITORY="b2:${BUCKET_NAME}:/phone/${DEVICE_TAG}"
export RESTIC_PASSWORD_FILE="$HOME/.config/backup/restic-password"

# 3. Dynamic Log Management
LOG_DIR="$HOME/.config/backup/logs"
mkdir -p "$LOG_DIR"
LOG="${LOG_DIR}/${BACKUP_TAG}-backup.log"
exec >>"$LOG" 2>&1

echo "════════════════════════════════════"
echo "📥 BACKUP STARTED [${BACKUP_TAG^^}]: $(date)"
echo "════════════════════════════════════"

# Acquire wake lock to stop Android OS from putting Termux to sleep mid-run
termux-wake-lock
sleep 2

# Verify target directory exists before running cryptographic actions
if [ ! -d "$TARGET_DIR" ]; then
  echo "❌ Targeted directory missing or inaccessible: $TARGET_DIR"
  termux-notification \
    --title "❌ Backup Failed [${DEVICE_TAG}]" \
    --content "Target directory missing: $TARGET_DIR"
  termux-wake-unlock
  exit 1
fi

echo "✅ Backing up target directory: $TARGET_DIR"

# Fire the client engine with auto-deduplication and compression
restic backup \
  --verbose \
  --exclude-caches \
  "$TARGET_DIR"

RC=$?

# 4. Automation Notification Router
if [ "$RC" -eq 0 ]; then
  restic snapshots --latest 1 | tail -5
  termux-notification \
    --title "✅ Backup Successful [${DEVICE_TAG}]" \
    --content "${BACKUP_TAG^^} directory successfully snapshot to Backblaze."
else
  restic unlock || true
  termux-notification \
    --title "❌ Backup Failed [${DEVICE_TAG}]" \
    --content "Restic encountered an error backing up ${BACKUP_TAG}. Check local logs."
fi

# Release CPU block and allow Android to enter normal sleep states
termux-wake-unlock
echo "🏁 Execution Finished: $(date)"
exit "$RC"
```

Make the script executable:
```bash
chmod +x ~/.config/backup/scripts/phone-backup.sh
```

### Step 5: Implement the Automation Safety Net
If Android’s aggressive Out-Of-Memory (OOM) manager kills your script in the middle of a massive file transfer, the system might leave a `termux-wake-lock` active, draining your battery to 0% by morning. 

To prevent this, create a safety cleanup script at `~/.config/backup/scripts/wake-unlock.sh`:

```bash
#!/data/data/com.termux/files/usr/bin/bash
set -e

# Fail-safe: release wakelock under any circumstance
if command -v termux-wake-unlock >/dev/null 2>&1; then
    termux-wake-unlock >/dev/null 2>&1 || true
fi

exit 0
```
Make it executable:
```bash
chmod +x ~/.config/backup/scripts/wake-unlock.sh
```

### Step 6: Initialize the Repository
Before automating, you must initialize the cloud directory manually once:
```bash
source ~/.config/backup/b2-credentials.conf
export RESTIC_REPOSITORY="b2:your-b2-bucket-name:/phone/your-device-tag"
export RESTIC_PASSWORD_FILE="$HOME/.config/backup/restic-password"

restic init
```

---

## 🤖 Tasker Automation Logic

To run this completely hands-free, configure Tasker to orchestrate the pipeline without causing performance overhead while you use your device:

*(Note: Depending on your Termux version, you may need to symlink these scripts into `~/.termux/tasker/` or enable `allow-external-apps=true` in `~/.termux/termux.properties` for the plugin to see them).*

1. **Profile Conditions:** * **Time:** `3:00 AM` to `3:05 AM`
   * **State:** `Power [Source: Any]` (Ensures the phone is charging)
   * **State:** `Wifi Connected [SSID: Your Home Wi-Fi]` (Bypasses mobile data drain)
2. **Entry Task (Execute Backup):**
   * Action: **Plugin > Termux:Tasker > Configuration**
   * Executable: `phone-backup.sh`
   * Target Directory: `/data/data/com.termux/files/home/.config/backup/scripts/`
3. **Exit Task (Safety Valve):**
   * Action: **Plugin > Termux:Tasker > Configuration**
   * Executable: `wake-unlock.sh`
   * Target Directory: `/data/data/com.termux/files/home/.config/backup/scripts/`
   * *Why:* If Tasker times out or the primary script crashes, this ensures the system wake lock is systematically cleared, protecting your hardware battery lifespan.

---

## 🛠️ Troubleshooting & Hidden Gotchas

* **The Silenced Console:** Because the script uses standard stream redirection (`exec >> log`), running it manually will appear to freeze your terminal screen. This is working as intended. Check its heartbeats live from another session using:
  ```bash
  tail -f "$(ls -t ~/.config/backup/logs/*.log | head -n1)"
  ```
* **Error 401 Unauthorized:** Your `b2-credentials.conf` file either has trailing white spaces copied from your mobile clipboard or used double quotes that broke special syntax keys. Shift to single quotes (`'`) and verify clean formatting.

---
