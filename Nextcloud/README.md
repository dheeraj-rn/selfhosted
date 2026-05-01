# Nextcloud Configuration Manual Changes
This document tracks the specific manual customizations applied to the Nextcloud container configurations.

## 1. Core Config (`config/Nextcloud/www/nextcloud/config/config.php`)
```php
// Trust Traefik docker network proxies
'trusted_proxies' => 
array (
  0 => '172.18.0.0/16',
),

// Local network bypass removed. Trusted domains restricted to web proxy:
'trusted_domains' =>
array (
  0 => 'nextcloud.exampledomain.com',
),

// Reverse proxy & external domain configuration
'overwrite.cli.url' => 'https://nextcloud.exampledomain.com',
'overwritehost' => 'nextcloud.exampledomain.com',
'overwriteprotocol' => 'https',

// Regional and Maintenance adjustments
'default_phone_region' => 'IN',
'maintenance_window_start' => 21,

// Memory Caching (Redis for locking and distributed, APCu for local)
'memcache.local' => '\\OC\\Memcache\\APCu',
'memcache.distributed' => '\\OC\\Memcache\\Redis',
'memcache.locking' => '\\OC\\Memcache\\Redis',
'redis' => 
array (
  'host' => 'redis',
  'password' => '...',
  'port' => 6379,
),
```

## 2. Docker Compose (`Nextcloud/docker-compose.yml`)
* Traefik `redirectregex` middleware labels were added for `/.well-known/carddav` and `/.well-known/caldav` to securely handle Contact and Calendar sync routing defaults.
* **Memory Cgroups (Fix for `docker stats` 0B error)**:
    Append the following to the end of the line in `/boot/firmware/cmdline.txt`:
    `cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1`

## 3. NGINX Configs (`config/Nextcloud/nginx/`)
* **Up-to-Date:** `ssl.conf`, `nginx.conf`, and `site-confs/default.conf` were successfully upgraded to the pristine 2025 standards from their `.sample` files. No manual overrides are currently active or needed.

## 4. PHP-FPM Performance (`config/Nextcloud/php/www2.conf`)
Applied to resolve high CPU usage and "max_children" errors during heavy syncing.
```ini
[www]
pm = ondemand
pm.max_children = 20
pm.process_idle_timeout = 60s
pm.max_requests = 500
```
* **pm = ondemand**: Reduces RAM overhead by killing idle PHP processes.
* **pm.max_children = 20**: Increased from the default (5) to prevent 504 Gateway Timeouts during photo uploads or indexing.

## 5. OS-Level Optimizations (Host System)
Adjustments made to the Raspberry Pi OS/OMV environment to handle multi-container workloads.

### Swap Expansion (`/etc/dphys-swapfile`)
* **Setting**: `CONF_SWAPSIZE=2048`
* **Purpose**: Expands the swap safety net to 2GB to prevent OOM-Killer crashes during heavy processing.

### Kernel Swappiness (`/etc/sysctl.conf`)
* **Setting**: `vm.swappiness=10`
* **Purpose**: Ensures the 8GB of RAM is prioritized for active containers while offloading idle data to the SSD swap.

## 6. Maintenance & Performance Monitoring

### Background Jobs
* **Configuration**: Ensure **Cron** is selected in *Administration settings > Basic settings*. Avoid using **AJAX**, as it causes CPU spikes during user interaction.

### Real-Time Monitoring
* **HTOP Customization**: Enable the `M_SWAP` column (*F2 > Screens > Main > Active Columns*) to track which containers are offloading to the SSD.
* **Nextcloud Apps**: Use the **Memories** app for high-performance gallery browsing and **Recognize** (AI) for automated tagging.
* **Log Location**: Check `/home/omv/Docker/config/Nextcloud/log/php/error.log` for `pm.max_children` warnings.
* **Redis Caching**: Monitor real-time Nextcloud cache hits and file locks by running `docker exec -it redis redis-cli -a 'password' monitor` (press `Ctrl+C` to stop).

## 7. Recognize (AI) App Configuration & Hardware Fixes
Applied to enable high-performance Face and Object recognition on Raspberry Pi 5 (ARM64) while bypassing Alpine Linux (musl) compatibility issues.

### Binary Environment (`Nextcloud/docker-compose.yml`)
To resolve "Node.js binary not found" and "GLIBC" errors common in the LSIO Alpine image, the following Docker Mods were implemented to install native ARM64 binaries:
```yaml
environment:
  - DOCKER_MODS=linuxserver/mods:universal-package-install
  - INSTALL_PACKAGES=nodejs|ffmpeg|coreutils
```
* **coreutils**: Required to provide the full GNU `nice` binary (default Alpine `nice` often fails with specific Nextcloud app flags).

### PHP Memory Override (`config/Nextcloud/php/php-local.ini`)
AI processing is RAM-intensive. While `www2.conf` handles process management, a local INI override is required to give the PHP engine enough "desk space" for image analysis.
```ini
; Path: config/Nextcloud/php/php-local.ini
memory_limit = 2G
upload_max_filesize = 10G
post_max_size = 10G
```
* **Note**: In the LSIO image, setting `PHP_MEMORY_LIMIT` in Docker environment variables is often ignored; the `.ini` file is the source of truth.

### Recognize App Settings (Web UI)
The following absolute paths must be set in *Administration Settings > Recognize* to utilize the binaries installed via Docker Mods:
* **Node.js binary path**: `/usr/bin/node`
* **FFmpeg binary path**: `/usr/bin/ffmpeg`
* **Nice binary path**: `/bin/nice` (Verify via `which nice` if warning persists)
* **Execution Mode**: **Node.js** (Enabled only after Docker Mod installation; significantly faster than WASM on Pi 5).

### Essential Maintenance & AI Commands
| Command | Purpose |
| :--- | :--- |
| `docker exec -it nextcloud /usr/bin/occ maintenance:repair` | **First Aid:** Checks for broken background jobs, repairs database schema mismatches, and clears "stuck" app states. Run this after unexpected power outages. |
| `docker exec -it nextcloud /usr/bin/occ db:add-missing-indices` | **Optimization:** Fixes database performance to handle thousands of AI tags. |
| `docker exec -it nextcloud /usr/bin/occ recognize:recrawl` | **Reset Queue:** Forces the app to re-examine the library and queue files that were skipped or failed. |
| `docker exec -it nextcloud /usr/bin/occ recognize:classify` | **Detection:** Scans files for objects/scenes (The "Librarian"). |
| `docker exec -it nextcloud /usr/bin/occ recognize:cluster-faces` | **Grouping:** Groups identified faces into specific people (The "Organizer"). |

### Known Constraints
* **WASM Mode**: Used as a fallback if Node.js crashes. It is restricted to a **single CPU core** on the Pi 5.
* **AVX Instructions**: The Pi 5 lacks AVX instructions found in X86 CPUs. Performance is expected to be slower (approx. 1-2 seconds per image).
* **Power Resilience**: Sudden reboots (power outages) wipe `tmux` sessions. Always check `recognize:stats` after an unclean shutdown to verify the queue status.

### Monitoring & Status
* **Queue Status**: Since there is no CLI `stats` command, the real-time queue count must be monitored via **Settings > Administration > Recognize**.
* **Power Resilience**: If the Pi reboots during a scan, run `maintenance:repair` followed by `recognize:classify` to pick up where it left off.

## 8. Memories App & Database Performance
Applied to enable high-speed timeline scrubbing and background metadata processing for large libraries (16k+ files).

### Database Trigger Fix (Error 1419)
The Memories app requires database triggers for performance. By default, MariaDB/MySQL restricts trigger creation if binary logging is enabled without "Super" privileges.

**Step 1: Grant Trust on Database Host**
Run this on the database container (not Nextcloud) to allow trigger creation:
```bash
docker exec -it mariadb mariadb -u root -p -e "SET GLOBAL log_bin_trust_function_creators = 1;"
```

**Step 2: Initialize Triggers & Index**
Nextcloud 33+ handles trigger setup via the repair command.
```bash
docker exec -it nextcloud occ maintenance:repair
docker exec -it nextcloud occ memories:index
```

### EXIF & Reverse Geocoding
* **EXIF Extraction**: Enabled **System Perl** in *Settings > Memories* to handle metadata extraction efficiently using the `perl` package installed via Docker Mods.
* **Reverse Geocoding**: Initialized the coordinate tables and downloaded the offline planet database.
    ```bash
    docker exec -it nextcloud occ memories:places-setup
    ```

---

## 9. Hardware Acceleration (VA-API) for Pi 5
Configured to offload video transcoding and UI rendering to the VideoCore VII GPU.

### Device Mapping & Permissions (`docker-compose.yml`)
The Pi 5 separates display and rendering. Both groups must be mapped to the `abc` user.
```yaml
services:
  nextcloud:
    devices:
      - /dev/dri:/dev/dri
    group_add:
      - "44"  # video
      - "105" # render (Verify ID via 'getent group render' on host)
    environment:
      - INSTALL_PACKAGES=...|perl|libva-utils|mesa-va-gallium|mesa-dri-gallium
      - LIBVA_DRIVER_NAME=v3d
      - MESA_LOADER_DRIVER_OVERRIDE=v3d
      - LIBVA_DRIVERS_PATH=/usr/lib/dri
```

### Manual Driver Symlink (Alpine/Musl Fix)
The LSIO Alpine image may fail to create the specific `v3d` symlink for VA-API. This must be manually linked to the Gallium driver:
```bash
docker exec -it nextcloud ln -s /usr/lib/libgallium-25.1.9.so /usr/lib/dri/v3d_drv_video.so
```
*Note: If `vainfo` fails after an image update, verify the version number of `libgallium` inside `/usr/lib/`.*

---

## 10. Gallery Maintenance & Preview Generation
Essential for preventing Raspberry Pi 5 CPU exhaustion when browsing thousands of high-resolution photos.

### Preview Generation (The "Snappy" Factor)
Nextcloud generates thumbnails on-the-fly by default, which is too heavy for the Pi. The **Preview Generator** app pre-builds these in the background.

**Initial Generation (Run once):**
```bash
docker exec -it nextcloud occ preview:generate-all -vvv
```

**Automated Background Cron (Host Crontab):**
Add this via `crontab -e` on the OMV host to process new uploads every 15 minutes:
```bash
*/15 * * * * docker exec nextcloud occ preview:pre-generate
```

### Maintenance Commands Summary
| Command | Purpose |
| :--- | :--- |
| `occ memories:index` | Scans filesystem for new media and updates the Memories timeline. |
| `occ memories:places-setup` | Initializes/Updates the reverse geocoding (maps) database. |
| `vainfo --display drm --device /dev/dri/renderD128` | **Diagnostic:** Verifies the container can communicate with the Pi 5 GPU. |
| `occ preview:generate-all` | Pre-builds all thumbnails (Run this after adding large bulk libraries). |

---

### Known Constraints (Pi 5 Specific)
* **H.264 Hardware Encoding**: The Pi 5 lacks a dedicated H.264 hardware encoder. While VA-API helps with *decoding* and UI acceleration, live *encoding* (transcoding) for streaming will fallback to the CPU. The Pi 5 CPU is sufficiently powerful to handle 1-2 concurrent software streams.
* **Memory Limit**: Ensure the PHP memory limit remains at **2G** to accommodate the simultaneous demands of `Memories` and `Recognize`.
