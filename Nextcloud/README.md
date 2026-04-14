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

// Memory Caching (Note: Redis is recommended over APCu for locking)
'memcache.local' => '\\OC\\Memcache\\APCu',
'memcache.locking' => '\\OC\\Memcache\\APCu',
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