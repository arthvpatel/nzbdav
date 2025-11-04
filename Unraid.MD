Here's the **complete Unraid Guide** section you can add to your GitHub repository (e.g., in `README.md` under a new heading). It includes all your requested details in a clean, professional, and user-friendly format:

---

```markdown
## Unraid Guide

This guide walks you through setting up **nzbdav** on **Unraid** with full integration into your media stack (Sonarr, Radarr, Plex, etc.) using **rclone** and a startup script.

---

### Required Plugins & Containers

1. **Plugins (install via Community Applications):**
   - `rclone`
   - `User Scripts`

2. **Docker Container:**
   - `nzbdav` (available in the Unraid App Store / Community Apps)

---

### Suggested Directory Structure

Create the following directories via Unraid **Shares** or manually in Terminal:

```
/mnt/user/nzbdav
/mnt/user/nzbdav/mount
/mnt/user/nzbdav/media
/mnt/user/nzbdav/media/movies
/mnt/user/nzbdav/media/tv
```

> These paths will be used for mounting and media access.

---

### Step 1: Configure rclone (WebDAV Remote)

1. Go to **Settings > rclone** in the Unraid GUI.
2. Add a new remote config:

```ini
[nzb-dav]
type = webdav
url = https://your-nzbdav-domain.com  # Replace with your actual endpoint
vendor = other
user = your-webdav-username
pass = ***OBSCURED***  # Use `rclone obscure yourpassword` to generate
```

> **Generate obscured password:**
> ```bash
> rclone obscure your_actual_password
> ```
> Copy the output and paste it into the `pass` field.

---

### Step 2: Update Docker Container Paths

For **all** containers (including `nzbdav`, `sonarrnzb`, `radarrnzb`, `plex`, etc.), **change the host path mapping** from:

```
Container Path: /mnt/user/nzbdav
Host Path:     /mnt/user/nzbdav
```

**TO:**

```
Container Path: /mnt/user/nzbdav
Host Path:     /mnt/user/nzbdav/mount
```

> This ensures all apps access files **through the rclone mount**, not directly on disk.

#### Specific Media Paths in Sonarr/Radarr:
- **Sonarr:** Root Folder → `/mnt/user/nzbdav/media/tv`
- **Radarr:** Root Folder → `/mnt/user/nzbdav/media/movies`

---

### Step 3: Startup Script (Run at Array Start)

Create a **User Script** in Unraid:

1. Go to **Settings > User Scripts**
2. Click **Add New Script**
3. Name: `Start nzbdav + Mount + Media Stack`
4. Schedule: **At Startup of Array**
5. Paste the script below:

```bash
#!/bin/bash
# === SETTINGS ===
RCLONE_REMOTE_PATH="nzb-dav:" # rclone remote path
MOUNT_POINT="/mnt/user/nzbdav/mount" # local mount point
CACHE_PATH="/dev/shm" # rclone VFS cache directory (fast RAM)
REMOTE_NAME="NzbDAV" # Docker container name (case-sensitive!)
WAIT_TIME=10 # wait time for container to initialize
DOCKER_CONTAINERS=( "sonarrnzb" "radarrnzb" "plex" ) # Add your containers here

echo "Sleeping 10 seconds for system boot..."
sleep 10

# === STEP 1: Start nzbdav Docker Container ===
echo "🚀 Starting nzbdav container..."
docker start "$REMOTE_NAME"

# === STEP 2: Confirm nzbdav is running ===
echo "🔍 Checking if $REMOTE_NAME is running..."
if docker ps --filter "name=$REMOTE_NAME" --filter "status=running" | grep -q "$REMOTE_NAME"; then
    echo "✅ $REMOTE_NAME is running."
else
    echo "❌ $REMOTE_NAME failed to start. Exiting."
    exit 1
fi

# === STEP 3: Wait for WebDAV server to initialize ===
echo "⏳ Waiting $WAIT_TIME seconds for $REMOTE_NAME to be ready..."
sleep "$WAIT_TIME"

# === STEP 4: Mount WebDAV with rclone ===
echo "📂 Mounting WebDAV to $MOUNT_POINT..."
mkdir -p "$MOUNT_POINT"

# Unmount if already mounted
if mountpoint -q "$MOUNT_POINT"; then
    echo "🔄 Unmounting existing mount..."
    fusermount -u "$MOUNT_POINT" || umount -f "$MOUNT_POINT" || true
fi

# Mount with rclone
rclone mount "$RCLONE_REMOTE_PATH" "$MOUNT_POINT" \
  --vfs-read-ahead=256M \
  --vfs-read-chunk-size=64M \
  --vfs-read-chunk-size-limit=4G \
  --vfs-cache-mode=full \
  --vfs-cache-max-age=3h \
  --vfs-cache-max-size=25G \
  --buffer-size=64M \
  --dir-cache-time=1s \
  --log-file=/var/log/rclone-nzbdav.log \
  --cache-dir="$CACHE_PATH" \
  --allow-other \
  --links \
  --use-cookies \
  --uid=1000 \
  --gid=1000 &

sleep 2

# Confirm mount
if mountpoint -q "$MOUNT_POINT"; then
    echo "✅ WebDAV mounted successfully at $MOUNT_POINT"
else
    echo "❌ Failed to mount WebDAV."
    exit 1
fi

# === STEP 5: Start Additional Docker Containers ===
echo "📦 Starting media stack containers..."
for container in "${DOCKER_CONTAINERS[@]}"; do
    if docker ps -a --filter "name=^/${container}$" --format '{{.Names}}' | grep -q "^${container}$"; then
        echo "▶️ Starting $container..."
        docker start "$container"
    else
        echo "⚠️ Container $container not found. Skipping."
    fi
done

echo "🎉 All tasks completed. Media stack is ready!"
```

> **Make script executable:**
> ```bash
> chmod +x /boot/config/plugins/user.scripts/scripts/Start_nzbdav_Mount_Media_Stack/script.sh
> ```

---

### Final Notes

- **Logs:** Check `/var/log/rclone-nzbdav.log` for mount issues.
- **Cache:** Uses `/dev/shm` (RAM) — ensure you have **at least 32GB RAM** for 25G cache.
- **Performance Tips:**
  - Increase `--vfs-cache-max-size` if you have more RAM.
  - Use SSD cache pool for `/mnt/user/nzbdav` if possible.

---

**Enjoy seamless cloud media access on Unraid with zero local storage bloat!**
```

---

