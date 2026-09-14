# Banrimkwae.com — WordPress on OpenLiteSpeed

This repository hosts the **production** WordPress stack for `banrimkwae.com` using
**OpenLiteSpeed** (OLS) + **LSPHP** and **MariaDB**, all wired together with
`docker compose`.

This configuration replaces the previous Nginx + PHP-FPM stack that lives in
`/home/zongbao/www.banrimkwae.com/`.

---

## Why OpenLiteSpeed?

OpenLiteSpeed bundles **LSPHP (LiteSpeed PHP, a.k.a. LSAPI)** into the same
container as the web server — there is no separate PHP-FPM container needed.
Compared to the old Nginx + PHP-FPM setup this means:

- **Fewer containers** (2 instead of 3)
- **Faster PHP** via in-process LSAPI (no FPM socket hop)
- **`.htaccess`-like** per-directory rewrites (WordPress permalinks) work natively

---

## Architecture

| Service      | Image                            | Role                                          |
| ------------ | -------------------------------- | --------------------------------------------- |
| `database`   | `mariadb:10.11`                  | MariaDB — re-uses existing `brk_data` volume  |
| `wordpress`  | `litespeedtech/openlitespeed`    | OLS web server **+** LSPHP runtime            |

Both services share the user-defined bridge network `brk-network`.

---

## File layout

```
.
├── docker-compose.yml          # Main stack definition (2 services)
├── .env.example                # Template for environment variables
├── .gitignore
├── .dockerignore
├── README.md                   # This file
├── database/.gitkeep           # empty (DB is read from existing volume)
├── ols-conf/                   # OpenLiteSpeed configuration directory
│   ├── httpd_config.conf       # Main OLS config (listeners, virtual hosts, ...)
│   ├── php.ini                 # PHP runtime settings used by LSPHP
│   └── vhosts/
│       └── banrimkwae/
│           └── vhconf.conf     # Per-vhost config (rewrite, security, cache, ...)
├── ols-admin-conf/             # OLS WebAdmin config (admin password, listener)
│   └── admin_config.conf
└── www/                        # WordPress installation (bind-mounted into container)
```

---

## Prerequisites

- Docker Engine 20.10+
- Docker Compose v2 (`docker compose` CLI)
- The existing Docker volume referenced by `DB_VOLUME_NAME` in `.env`
  (default: `wwwbanrimkwaecom_brk_data`) is already on the host. It was
  created by the previous Nginx stack and contains the production database
  — **do not delete it**.

Verify it exists:

```bash
docker volume inspect $(grep DB_VOLUME_NAME .env | cut -d= -f2)
```

If it is missing, see the **Recovery** section below.

---

## First-time setup

1. Copy the example env file and edit values:

   ```bash
   cp .env.example .env
   nano .env
   ```

   At minimum set:

   - `DATABASENAME` / `DATABASEUSER` / `DATABASEPASS` — must match the
     values in `www/wp-config.php`.
   - `DB_VOLUME_NAME` — the existing volume name on the host
     (default already points at the production volume).
   - `MYSQL_ROOT_PASSWORD` — must match what was used with the old stack
     if you re-use a previously initialized data directory.

2. Make sure `www/` contains the WordPress installation (it should already —
   it is the same folder previously served by Nginx).

3. OLS configs under `ols-conf/` are mounted into `/usr/local/lsws/conf`.
   `ols-admin-conf/` is mounted into `/usr/local/lsws/admin/conf` so you
   can change the WebAdmin password without losing it across `up`.

4. Bring the stack up:

   ```bash
   docker compose up -d
   ```

5. Verify:

   ```bash
   docker compose ps
   docker compose logs -f wordpress
   curl -I http://localhost
   ```

---

## Migrating from the old Nginx stack

The old stack lives at `/home/zongbao/www.banrimkwae.com/`.
The MariaDB **volume is shared** between the two stacks via the external
volume referenced by `DB_VOLUME_NAME`, so database data is preserved.

Migration steps:

```bash
# 1. Bring just the database container up first (re-attaches to existing volume)
cd /home/zongbao/banrimkwae.com
docker compose up -d database

# 2. Stop the old stack
cd /home/zongbao/www.banrimkwae.com
docker compose down

# 3. Bring up the full new stack
cd /home/zongbao/banrimkwae.com
docker compose up -d
```

> ℹ️  `docker compose down` on the old stack (without `-v`) keeps the named
>  volume intact. **Never** run `docker compose down -v` — that would drop
>  the MariaDB data volume.

---

## Day-to-day commands

```bash
# Start / stop
docker compose up -d
docker compose down
docker compose restart wordpress

# Logs
docker compose logs -f wordpress
docker compose logs -f database

# Shell into a service
docker compose exec wordpress bash
docker compose exec database bash

# Backup database
docker compose exec database mysqldump \
    -u root -p"${MYSQL_ROOT_PASSWORD:-rootpassword}" \
    ${DATABASENAME} > backup-$(date +%F).sql

# OpenLiteSpeed WebAdmin UI
# https://<server-ip>:7080
```

---

## Key OpenLiteSpeed notes

- **Cloudflare real-IP**: OLS inherits Cloudflare's `CF-Connecting-IP` so
  PHP sees the real visitor IP. Make sure the cluster is behind Cloudflare
  when using `vhconf.conf`.
- **`/context/` path**: protected with the same hard-coded token used in
  the old Nginx config.
- **Upload limit**: 64 MB, controlled by both OLS and PHP (`upload_max_filesize`,
  `post_max_size` in `ols-conf/php.ini`).
- **HTTPS**: port 443 is exposed; drop Cloudflare Origin TLS certs into
  `ols-conf/vhosts/banrimkwae/` and reference them in `vhconf.conf`.
- **WebAdmin port 7080** is exposed; treat the admin password in
  `ols-admin-conf/admin_config.conf` like a secret.

---

## Troubleshooting

### `docker compose up` fails because the DB volume is missing

```bash
docker volume create wwwbanrimkwaecom_brk_data
# Then restore from a backup (see "Recovery")
```

### WordPress cannot reach DB

- Confirm `DATABASENAME` / `DATABASEUSER` / `DATABASEPASS` in `.env` match
  the values in `www/wp-config.php`.
- From inside the OLS container:

  ```bash
  docker compose exec wordpress bash -c \
      "mysql -h database -u ${DATABASEUSER} -p${DATABASEPASS} -e 'SHOW DATABASES;'"
  ```

### Permission issues on `www/`

LSPHP runs as `nobody:nogroup` inside the OLS image. WordPress still works
because the document root is bind-mounted and writable.

---

## Recovery (if the production volume is lost)

If the DB volume no longer exists and you have a SQL dump:

```bash
docker volume create $(grep DB_VOLUME_NAME .env | cut -d= -f2)
docker compose up -d database
# Or restore manually:
docker compose exec -T database mysql -u root -p"${MYSQL_ROOT_PASSWORD}" \
    ${DATABASENAME} < /path/to/backup.sql
docker compose up -d
```