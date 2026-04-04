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

## 3. NGINX Configs (`config/Nextcloud/nginx/`)
* **Up-to-Date:** `ssl.conf`, `nginx.conf`, and `site-confs/default.conf` were successfully upgraded to the pristine 2025 standards from their `.sample` files. No manual overrides are currently active or needed.