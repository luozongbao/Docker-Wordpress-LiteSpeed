# Banrimkwae.com — WordPress on OpenLiteSpeed

This repository hosts the **production** WordPress stack for `banrimkwae.com` using
**OpenLiteSpeed** (OLS), **PHP-FPM 8.3**, and **MariaDB**, all wired together with
`docker compose`.

This configuration replaces the previous Nginx-based stack that lives in
`/home/zongbao/www.banrimkwae.com/`.

---

## Architecture

| Service        | Image                                | Role                                  |
| -------------- | ------------------------------------ | ------------------------------------- |
| `litespeed`    | `litespeedtech/openlitespeed:latest` | Web server (HTTP/80, HTTPS/443)       |
| `php`          | custom `php:8.3-fpm` (built locally) | PHP-FPM backend for WordPress         |
| `db`           | `mariadb:latest`                      | MariaDB — uses the **existing** data volume `brk_data` |

All three services share the user-defined bridge network `brk-network`.

---

## File layout

```
.
├── docker-compose.yml          # Main stack definition
├── dockerfile                  # Custom PHP 8.3-FPM image
├── .env.example                # Template for environment variables
├── .gitignore
├── .dockerignore
├── README.md                   # This file
├── ols-conf/                   # OpenLiteSpeed configuration
│   ├── httpd_config.conf       # Main OLS config (listeners, virtual hosts, ...)
│   ├── php.ini                 # PHP settings used by OLS LSAPI
│   ├── php-fpm.conf            # PHP-FPM pool config
│   └── vhosts/
│       └── banrimkwae/
│           └── vhconf.conf     # Per-vhost config (rewrite, security, gzip, ...)
└── www/                        # WordPress installation (read/write by containers)
```

---

## Prerequisites

- Docker Engine 20.10+
- Docker Compose v2 (`docker compose` CLI)
- The existing Docker volume `wwwbanrimkwaecom_brk_data` is already present
  on the host (it was created by the previous Nginx stack and contains the
  production database — **do not delete it**).

Verify the volume exists before bringing the stack up:

```bash
docker volume inspect wwwbanrimkwaecom_brk_data
```

If it is missing, see the **Recovery** section below.

---

## First-time setup

1. Copy the example env file and edit values:

   ```bash
   cp .env.example .env
   nano .env
   ```

   At minimum set `DATABASENAME`, `DATABASEUSER`, `DATABASEPASS` to match
   the existing WordPress `wp-config.php`.

2. Make sure `www/` contains the WordPress installation (it should already —
   it is the same folder previously served by Nginx).

3. Verify OLS configs under `ols-conf/` (domain name, Cloudflare IP ranges,
   security headers, etc.) — they are mounted read-only into the container.

4. Bring the stack up:

   ```bash
   docker compose up -d
   ```

5. Verify:

   ```bash
   docker compose ps
   docker compose logs -f litespeed
   curl -I http://localhost
   ```

---

## Migrating from the old Nginx stack

The old stack lives at `/home/zongbao/www.banrimkwae.com/`.
The MariaDB **volume is shared** between the two stacks via the external volume
`wwwbanrimkwaecom_brk_data`, so database data is preserved.

Migration steps:

```bash
# 1. Bring the new stack up first (DB container re-attaches to existing volume)
cd /home/zongbao/banrimkwae.com
docker compose up -d db

# 2. Stop the old stack
cd /home/zongbao/www.banrimkwae.com
docker compose down

# 3. Bring the rest of the new stack up
cd /home/zongbao/banrimkwae.com
docker compose up -d
```

---

## Day-to-day commands

```bash
# Start / stop
docker compose up -d
docker compose down
docker compose restart litespeed

# Logs
docker compose logs -f litespeed
docker compose logs -f php
docker compose logs -f db

# Shell into a service
docker compose exec litespeed bash
docker compose exec php bash
docker compose exec db bash

# Backup database
docker compose exec db mysqldump \
    -u root -p"${MYSQL_ROOT_PASSWORD:-rootpassword}" \
    ${DATABASENAME} > backup-$(date +%F).sql
```

---

## Key OpenLiteSpeed notes

- **Cloudflare real-IP**: `ols-conf/vhosts/banrimkwae/vhconf.conf` declares
  the Cloudflare IP ranges as trusted. Requests originating from CF will
  have their real client IP forwarded to PHP.
- **AI/context path** (`/context/`): protected with the same token used in
  the old Nginx config (see `$CONTEXT_TOKEN`).
- **Upload limit**: 64 MB, controlled by both OLS and PHP (`upload_max_filesize`,
  `post_max_size` in `ols-conf/php.ini`).
- **HTTPS**: port 443 is exposed; you can drop Cloudflare Origin TLS certs
  into `ols-conf/vhosts/banrimkwae/` and reference them in `vhconf.conf`.

---

## Troubleshooting

### `docker compose up` fails because volume is missing

```bash
docker volume create wwwbanrimkwaecom_brk_data
# Then restore from a backup (see "Recovery")
```

### WordPress cannot reach DB

- Confirm `DATABASENAME` / `DATABASEUSER` / `DATABASEPASS` in `.env` match
  the values in `www/wp-config.php`.
- From inside the PHP container:

  ```bash
  docker compose exec php bash -c \
      "mysql -h db -u ${DATABASEUSER} -p${DATABASEPASS} -e 'SHOW DATABASES;'"
  ```

### Permission issues on `www/`

The PHP container runs `chown -R www-data:www-data /var/www/html` on start.
OLS writes files as `nobody:nogroup` — this is expected and WordPress will
still work because PHP owns the document root.

---

## Recovery (if the production volume is lost)

If `wwwbanrimkwaecom_brk_data` does not exist and you have a SQL dump:

```bash
docker volume create wwwbanrimkwaecom_brk_data
docker compose up -d db        # will run init scripts from /docker-entrypoint-initdb.d if mounted
# Or restore manually:
docker compose exec -T db mysql -u root -p"${MYSQL_ROOT_PASSWORD}" \
    ${DATABASENAME} < /path/to/backup.sql
```