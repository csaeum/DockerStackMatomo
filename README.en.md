# Matomo Docker Stack

[🇩🇪 Deutsch](README.md) | 🇬🇧 **English** | [🇫🇷 Français](README.fr.md)

Optimized Docker Stack for Matomo Analytics with Traefik Reverse Proxy, Redis Caching, and automatic backups.

**Optimized for:** 10-15 websites, ~500 users/day per site, ~10,000 page views/day

---

## ✨ Features

- ✅ **PHP 8.3** with OPcache + JIT compiler
- ✅ **MariaDB 11.4** with performance-optimized configuration
- ✅ **Redis 7** for caching and sessions
- ✅ **Traefik** integration with security middlewares
- ✅ **Automatic SSL/TLS** via Let's Encrypt
- ✅ **Automatic database backups** every 6 hours
- ✅ **Ofelia CronJobs** for archiving and maintenance
- ✅ **Apache security** configuration (blocks /tmp/, /config/, etc.)
- ✅ **tmpfs** for improved I/O performance

---

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start](#quick-start)
3. [Matomo Configuration](#matomo-configuration)
4. [Maintenance](#maintenance)
5. [Troubleshooting](#troubleshooting)
6. [Backup & Restore](#backup--restore)
7. [Scaling](#scaling)

---

## 🔧 Prerequisites

### Required External Services

1. **Traefik Reverse Proxy** must be running
   - Network: `traefik_proxy_network`
   - Required Middlewares: `security-headers@file`, `compression@file`, `rate-limit@file`, `redirect-to-https@file`
   - TLS Options: `modern@file`

2. **Ofelia CronJob Scheduler** must be running
   - Repository: https://github.com/csaeum/DockerStackOfelia

### Server Requirements

- **RAM:** At least 6GB available
- **CPU:** 2+ cores recommended
- **Storage:** SSD with min. 50GB free space
- **Docker:** Version 24.0+
- **Docker Compose:** Version 2.20+

---

## 🚀 Quick Start

### 1. Clone Repository

```bash
git clone https://github.com/csaeum/DockerStackMatomo.git
cd DockerStackMatomo
```

### 2. Create Volumes

```bash
mkdir -p volumes/html volumes/sql
```

### 3. Configure Environment

```bash
cp .env.example .env
nano .env
```

**Important settings in `.env`:**

```bash
COMPOSE_PROJECT_NAME=matomo
BACKUPVOLUME=/path/to/your/backup
HOSTRULE=Host(`matomo.your-domain.com`)

# Generate secure passwords:
# openssl rand -base64 32
REDIS_PASSWORD=YOUR_GENERATED_PASSWORD
SQL_PASSWORD=YOUR_GENERATED_PASSWORD
SQL_ROOT_PASSWORD=YOUR_GENERATED_PASSWORD
```

### 4. Start Stack

```bash
docker compose up -d
```

### 5. Complete Matomo Installation

1. Open browser: `https://your-domain.com`
2. Follow Matomo setup wizard:
   - **Database Host:** `sql`
   - **Database Name:** `MatomoDB`
   - **Database User:** `MatomoUser`
   - **Database Password:** Your `SQL_PASSWORD` from `.env`

---

## ⚙️ Matomo Configuration

### 1. Enable Redis Cache (IMPORTANT!)

Edit `volumes/html/config/config.ini.php`:

```bash
docker exec -it matomo-php-apache nano /var/www/html/config/config.ini.php
```

Add at the end:

```ini
[Cache]
backend = redis

[RedisCache]
host = "redis"
port = 6379
password = "YOUR_REDIS_PASSWORD"
database = 0
timeout = 0.0

[QueuedTracking]
backend = "redis"
host = "redis"
port = 6379
password = "YOUR_REDIS_PASSWORD"
database = 1
```

### 2. Disable Browser Archiving

In the same `config.ini.php`:

```ini
[General]
enable_browser_archiving_triggering = 0
browser_archiving_disabled_enforce = 1
```

### 3. Restart Container

```bash
docker compose restart php-apache
```

---

## ⏰ Automatic CronJobs

Configured via Ofelia (in `docker-compose.yaml`):

1. **Archiving:** Hourly (generates reports for all sites)
2. **Log Deletion:** Daily at 2 AM (removes logs older than 180 days)
3. **Temp Cleanup:** Daily at 3 AM (purges old archive data)

### Check CronJob Logs

```bash
docker logs -f ofelia

# Manual test
docker exec matomo-php-apache php /var/www/html/console core:archive
```

---

## 🔧 Maintenance

### View Logs

```bash
docker compose logs -f
docker compose logs -f php-apache
docker compose logs -f sql
```

### Restart Containers

```bash
docker compose restart
docker compose restart php-apache
```

### Update Images

```bash
docker compose pull
docker compose up -d
docker image prune -f
```

---

## 🐛 Troubleshooting

### Database Connection Failed

```bash
docker compose ps sql
docker compose logs sql
docker exec matomo-sql mariadb-admin ping -uroot -p"${SQL_ROOT_PASSWORD}"
```

### Reports Not Generated

```bash
docker logs ofelia
docker exec matomo-php-apache php /var/www/html/console core:archive
```

### Redis Connection Failed

```bash
docker exec matomo-redis redis-cli -a "${REDIS_PASSWORD}" ping
docker compose logs redis
```

---

## 💾 Backup & Restore

### Automatic Backups

SQL backups are created automatically by `sqlbackup` container:

- **Frequency:** Every 6 hours
- **Location:** `${BACKUPVOLUME}/matomo/`
- **Retention:** Last 12 backups
- **Compression:** GZIP Level 9

### Manual Backup

```bash
docker exec matomo-sql mariadb-dump \
    -uroot -p"${SQL_ROOT_PASSWORD}" \
    --single-transaction \
    MatomoDB > backup_$(date +%Y%m%d_%H%M%S).sql
```

### Restore Backup

```bash
docker compose down
docker compose up -d sql
docker exec -i matomo-sql mariadb -uroot -p"${SQL_ROOT_PASSWORD}" MatomoDB < backup.sql
docker compose up -d
```

---

## 📈 Scaling

### When Traffic Increases (>50,000 Page Views/Day)

Install **QueuedTracking Plugin**:

1. In Matomo: **Administration → Marketplace**
2. Search for "QueuedTracking" and install
3. Add to `docker-compose.yaml`:

```yaml
- ofelia.job-exec.${COMPOSE_PROJECT_NAME}-queue-worker.schedule=* * * * *
- ofelia.job-exec.${COMPOSE_PROJECT_NAME}-queue-worker.command=php /var/www/html/console queuedtracking:process
```

### Increase Resources

```yaml
# In docker-compose.yaml
sql:
  deploy:
    resources:
      limits:
        memory: "8G"  # from 4G to 8G

redis:
  command:
    - --maxmemory
    - 2gb  # from 1gb to 2gb
```

Update `configs/sql/custom.cnf`:

```ini
innodb_buffer_pool_size = 6G  # 75% of 8G
```

---

## 📚 Resources

- **Matomo Documentation:** https://matomo.org/guides/
- **Traefik Labels:** https://github.com/csaeum/DockerStackTraefik/blob/main/LABELS-CHECKLIST.md
- **Ofelia CronJobs:** https://github.com/csaeum/DockerStackOfelia
- **Performance FAQ:** https://matomo.org/faq/on-premise/how-to-configure-matomo-for-speed/

---

## 📝 Changelog

### Version 1.0 (2025-12-28)
- ✅ Initial Release
- ✅ Optimized for 10-15 sites, ~500 users/day
- ✅ Redis caching integration
- ✅ Traefik security middlewares
- ✅ Apache security config
- ✅ Automatic backups
- ✅ Ofelia CronJobs
- ✅ Performance-optimized MariaDB
- ✅ PHP 8.3 with OPcache + JIT

---

## 📄 License

This project is licensed under the [GPL-3.0-or-later](LICENSE) License.

---

## 🤝 Support & Contribution

**Made with ❤️ by WSC - Web SEO Consulting**

This project is free and open source. If it helped you, I appreciate your support:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

### Questions or Issues?

- 🐛 **Bug Reports:** [GitHub Issues](https://github.com/csaeum/DockerStackMatomo/issues)
- 💬 **Discussions:** [GitHub Discussions](https://github.com/csaeum/DockerStackMatomo/discussions)
- 📧 **Contact:** [info@web-seo-consulting.eu](mailto:info@web-seo-consulting.eu)
