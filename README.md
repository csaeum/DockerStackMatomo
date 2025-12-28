# Matomo Docker Stack

Optimierter Docker Stack für Matomo Analytics mit Traefik Reverse Proxy, Redis Caching und automatischen Backups.

**Optimiert für:** 10-15 Websites, ~500 User/Tag pro Site, ~10.000 Page Views/Tag

---

## 📋 Inhaltsverzeichnis

1. [Voraussetzungen](#voraussetzungen)
2. [Installation](#installation)
3. [Matomo Konfiguration](#matomo-konfiguration)
4. [Ofelia CronJobs](#ofelia-cronjobs)
5. [Wartung](#wartung)
6. [Troubleshooting](#troubleshooting)
7. [Performance Monitoring](#performance-monitoring)
8. [Backup & Restore](#backup--restore)

---

## 🔧 Voraussetzungen

### Erforderliche externe Services

1. **Traefik Reverse Proxy** muss laufen
   - Netzwerk: `traefik_proxy_network`
   - Benötigte Middlewares (in Traefik `dynamic.yml`):
     - `security-headers@file`
     - `compression@file`
     - `rate-limit@file`
     - `redirect-to-https@file`
   - TLS Options: `modern@file`

2. **Ofelia CronJob Scheduler** muss laufen
   - Docker Socket muss gemountet sein
   - Repository: https://github.com/csaeum/DockerStackOfelia

### Server-Anforderungen

- **RAM:** Mindestens 6GB verfügbar
- **CPU:** 2+ Cores empfohlen
- **Speicher:** SSD mit min. 50GB freiem Platz
- **Docker:** Version 24.0+
- **Docker Compose:** Version 2.20+

---

## 🚀 Installation

### 1. Repository klonen

```bash
git clone https://github.com/IhrUsername/DockerStackMatomo.git
cd DockerStackMatomo
```

### 2. Volumes erstellen

```bash
mkdir -p volumes/html
mkdir -p volumes/sql
```

### 3. Umgebungsvariablen konfigurieren

```bash
cp .env.example .env
nano .env
```

**Wichtige Anpassungen in `.env`:**

```bash
# Projekt-Name (wird für Container-Namen verwendet)
COMPOSE_PROJECT_NAME=matomo

# Backup-Verzeichnis (anpassen!)
BACKUPVOLUME=/pfad/zu/ihrem/backup

# Ihre Domains (anpassen!)
HOSTRULE=Host(`matomo.ihre-domain.de`)

# Sichere Passwörter generieren:
# openssl rand -base64 32

REDIS_PASSWORD=HIER_GENERIERTES_PASSWORT
SQL_PASSWORD=HIER_GENERIERTES_PASSWORT
SQL_ROOT_PASSWORD=HIER_GENERIERTES_PASSWORT
```

### 4. Stack starten

```bash
docker compose up -d
```

### 5. Matomo Installation

1. Browser öffnen: `https://ihre-domain.de`
2. Matomo Setup-Wizard durchlaufen:
   - **Datenbank-Host:** `sql`
   - **Datenbank-Name:** `MatomoDB`
   - **Datenbank-User:** `MatomoUser`
   - **Datenbank-Passwort:** Ihr `SQL_PASSWORD` aus `.env`
   - **Tabellen-Präfix:** `matomo_` (Standard)

3. Admin-User anlegen
4. Erste Website hinzufügen

### 6. Sicherheitsprüfung

Nach der Installation sollten Sie die Matomo System-Diagnose prüfen:

**Administration → System → Systemprüfung**

Alle Punkte sollten grün sein, insbesondere:
- ✅ **Empfohlene private Verzeichnisse** - `/tmp/` und `/config/` sollten nicht erreichbar sein
- ✅ **Datei-Integrität**
- ✅ **PHP-Konfiguration**

**Testen Sie den Schutz:**
```bash
# Diese URLs sollten 403 Forbidden zurückgeben:
curl -I https://ihre-domain.de/tmp/
curl -I https://ihre-domain.de/config/
curl -I https://ihre-domain.de/config/config.ini.php
```

---

## ⚙️ Matomo Konfiguration

Nach der Installation müssen Sie die Matomo-Konfiguration anpassen für optimale Performance.

### 1. Redis Cache aktivieren

**Wichtig:** Dies ist der wichtigste Performance-Boost!

Bearbeiten Sie `volumes/html/config/config.ini.php`:

```bash
docker exec -it matomo-php-apache nano /var/www/html/config/config.ini.php
```

Fügen Sie am Ende der Datei hinzu:

```ini
[Cache]
backend = redis

[RedisCache]
host = "redis"
port = 6379
password = "IHR_REDIS_PASSWORD_AUS_ENV"
database = 0
timeout = 0.0

[QueuedTracking]
; Redis für Session-Storage verwenden
backend = "redis"
host = "redis"
port = 6379
password = "IHR_REDIS_PASSWORD_AUS_ENV"
database = 1
```

**Ersetzen Sie `IHR_REDIS_PASSWORD_AUS_ENV` mit dem Wert aus Ihrer `.env` Datei!**

### 2. Browser-Archivierung deaktivieren

**Wichtig:** Verhindert Performance-Probleme durch gleichzeitige Report-Generierung.

In derselben `config.ini.php` Datei:

```ini
[General]
enable_browser_archiving_triggering = 0
browser_archiving_disabled_enforce = 1
```

### 3. Trusted Hosts konfigurieren

Fügen Sie alle Ihre Domains als Trusted Hosts hinzu:

```ini
[General]
trusted_hosts[] = "matomo.ihre-domain.de"
trusted_hosts[] = "matomo.andere-domain.com"
```

### 4. Session Storage auf Redis umstellen

```ini
[General]
session_save_handler = "dbtable"

[RedisSession]
host = "redis"
port = 6379
password = "IHR_REDIS_PASSWORD_AUS_ENV"
database = 2
```

### 5. Datenschutz-Einstellungen (optional aber empfohlen)

```ini
[Tracker]
; IP-Anonymisierung aktivieren (DSGVO-konform)
ip_address_mask_length = 2

[PrivacyManager]
; IPs anonymisieren
anonymizeIpEnabled = 1
; Referrer anonymisieren
anonymizeReferrerEnabled = 1
; User-ID anonymisieren
anonymizeUserIdEnabled = 1
```

### 6. Performance-Tuning

```ini
[General]
; Archivierungs-Timeout erhöhen für große Installationen
archiving_query_max_execution_time = 7200

; Browser-Archivierung komplett deaktivieren
enable_browser_archiving_triggering = 0

[Tracker]
; Tracking-Requests optimieren
bulk_requests_max_requests_to_process = 100
```

### Konfiguration neu laden

Nach allen Änderungen:

```bash
docker compose restart php-apache
```

---

## ⏰ Ofelia CronJobs

Die folgenden CronJobs werden automatisch von Ofelia ausgeführt (konfiguriert in `docker-compose.yaml`):

### 1. Archivierung (Stündlich)

```yaml
Schedule: 0 * * * *  # Jede Stunde zur vollen Stunde
Command:  php /var/www/html/console core:archive
```

Generiert Reports für **alle konfigurierten Websites automatisch**. Läuft im Hintergrund ohne UI-Belastung.

**Hinweis:** Ohne `--url` Parameter werden alle in Matomo angelegten Sites archiviert. Optional können Sie eine spezifische Site archivieren mit `--url=https://ihre-domain.de`.

### 2. Alte Logs löschen (Täglich um 2 Uhr)

```yaml
Schedule: 0 2 * * *  # Täglich um 2:00 Uhr
Command:  php /var/www/html/console core:delete-logs-data --dates=180
```

Löscht Tracking-Logs älter als 180 Tage (6 Monate). Spart Speicherplatz.

### 3. Temporäre Daten bereinigen (Täglich um 3 Uhr)

```yaml
Schedule: 0 3 * * *  # Täglich um 3:00 Uhr
Command:  php /var/www/html/console core:purge-old-archive-data
```

Bereinigt alte temporäre Archive und Cache-Dateien.

### CronJob-Logs überprüfen

```bash
# Ofelia-Logs anzeigen
docker logs -f ofelia

# Manuell Archivierung testen (alle Sites)
docker exec matomo-php-apache php /var/www/html/console core:archive

# Oder nur eine spezifische Site archivieren
docker exec matomo-php-apache php /var/www/html/console core:archive --url=https://ihre-domain.de
```

---

## 🔧 Wartung

### Logs anzeigen

```bash
# Alle Services
docker compose logs -f

# Nur PHP/Apache
docker compose logs -f php-apache

# Nur MariaDB
docker compose logs -f sql

# Nur Redis
docker compose logs -f redis
```

### Container neu starten

```bash
# Alle Container
docker compose restart

# Nur PHP/Apache
docker compose restart php-apache

# Nur Datenbank
docker compose restart sql
```

### Updates durchführen

```bash
# Images aktualisieren
docker compose pull

# Stack neu starten mit neuen Images
docker compose up -d

# Alte Images aufräumen
docker image prune -f
```

### Matomo Updates

Matomo kann über das Web-Interface aktualisiert werden:

1. Login als Admin
2. **Administration → System → Updates**
3. Automatisches Update durchführen
4. Nach Update Container neu starten:
   ```bash
   docker compose restart php-apache
   ```

---

## 🐛 Troubleshooting

### Problem: "Database connection failed"

**Lösung:**
```bash
# Prüfen ob MariaDB läuft
docker compose ps sql

# MariaDB Logs prüfen
docker compose logs sql

# Healthcheck testen
docker exec matomo-sql mariadb-admin ping -uroot -p"${SQL_ROOT_PASSWORD}"
```

### Problem: Reports werden nicht generiert

**Lösung:**
```bash
# Ofelia-Logs prüfen
docker logs ofelia

# Manuell Archivierung ausführen
docker exec matomo-php-apache php /var/www/html/console core:archive

# Browser-Archivierung in config.ini.php deaktivieren (siehe oben)
```

### Problem: "Redis connection failed"

**Lösung:**
```bash
# Redis-Status prüfen
docker exec matomo-redis redis-cli -a "${REDIS_PASSWORD}" ping

# Redis-Logs prüfen
docker compose logs redis

# Passwort in config.ini.php überprüfen
```

### Problem: Langsame Performance

**Schritte zur Diagnose:**

1. **Container-Ressourcen prüfen:**
   ```bash
   docker stats
   ```

2. **MariaDB Buffer Pool Status:**
   ```bash
   docker exec matomo-sql mariadb -uroot -p"${SQL_ROOT_PASSWORD}" -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size';"
   ```

3. **Redis Memory Usage:**
   ```bash
   docker exec matomo-redis redis-cli -a "${REDIS_PASSWORD}" INFO memory
   ```

4. **PHP OPcache Status:**
   - Browser: `https://ihre-domain.de/index.php?module=Installation&action=systemCheck`
   - Prüfen: "OPcache" sollte grün sein

### Problem: Zu viel Speicherplatz belegt

**Lösung:**
```bash
# Alte Logs löschen (manuell)
docker exec matomo-php-apache php /var/www/html/console core:delete-logs-data --dates=90

# Temporäre Dateien bereinigen
docker exec matomo-php-apache php /var/www/html/console core:purge-old-archive-data

# Docker Volumes Größe prüfen
du -sh volumes/*
```

---

## 📊 Performance Monitoring

### Matomo System Check

Browser: `https://ihre-domain.de/index.php?module=Installation&action=systemCheck`

**Grüne Häkchen sollten sein bei:**
- ✅ PHP Version 8.3
- ✅ MySQL/MariaDB
- ✅ Required PHP Extensions
- ✅ File Integrity
- ✅ OPcache aktiviert

### Redis Monitoring

```bash
# Redis Stats
docker exec matomo-redis redis-cli -a "${REDIS_PASSWORD}" INFO stats

# Memory Usage
docker exec matomo-redis redis-cli -a "${REDIS_PASSWORD}" INFO memory

# Connected Clients
docker exec matomo-redis redis-cli -a "${REDIS_PASSWORD}" CLIENT LIST
```

### MariaDB Monitoring

```bash
# Database Size
docker exec matomo-sql mariadb -uroot -p"${SQL_ROOT_PASSWORD}" -e "
SELECT
    table_schema 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) 'Size (MB)'
FROM information_schema.tables
WHERE table_schema = 'MatomoDB'
GROUP BY table_schema;"

# InnoDB Status
docker exec matomo-sql mariadb -uroot -p"${SQL_ROOT_PASSWORD}" -e "SHOW ENGINE INNODB STATUS\G"
```

### Archivierungs-Performance messen

```bash
# Archivierung mit Timer
time docker exec matomo-php-apache php /var/www/html/console core:archive

# Detaillierte Statistiken
docker exec matomo-php-apache php /var/www/html/console core:archive --concurrent-requests-per-website=3
```

---

## 💾 Backup & Restore

### Automatische Backups

SQL-Backups werden automatisch erstellt durch den `sqlbackup` Container:

- **Frequenz:** Alle 6 Stunden (3:00, 9:00, 15:00, 21:00)
- **Speicherort:** `${BACKUPVOLUME}/matomo/`
- **Aufbewahrung:** Letzte 12 Backups
- **Kompression:** GZIP Level 9

### Manuelles Backup erstellen

```bash
# Datenbank Backup
docker exec matomo-sql mariadb-dump \
    -uroot -p"${SQL_ROOT_PASSWORD}" \
    --single-transaction \
    --quick \
    --lock-tables=false \
    MatomoDB > backup_$(date +%Y%m%d_%H%M%S).sql

# Matomo-Dateien sichern (config, plugins, etc.)
tar -czf matomo_files_$(date +%Y%m%d_%H%M%S).tar.gz volumes/html/config volumes/html/plugins
```

### Backup wiederherstellen

```bash
# 1. Stack stoppen
docker compose down

# 2. Datenbank wiederherstellen
docker compose up -d sql
docker exec -i matomo-sql mariadb -uroot -p"${SQL_ROOT_PASSWORD}" MatomoDB < backup_20250101_120000.sql

# 3. Dateien wiederherstellen (optional)
tar -xzf matomo_files_20250101_120000.tar.gz

# 4. Stack vollständig starten
docker compose up -d
```

### Disaster Recovery

**Komplette Neuinstallation mit Backup:**

```bash
# 1. Repository klonen
git clone https://github.com/IhrUsername/DockerStackMatomo.git
cd DockerStackMatomo

# 2. .env konfigurieren
cp .env.example .env
nano .env  # Passwörter MÜSSEN identisch sein!

# 3. Volumes vorbereiten
mkdir -p volumes/html volumes/sql

# 4. Nur Datenbank starten
docker compose up -d sql

# 5. Backup einspielen
docker exec -i matomo-sql mariadb -uroot -p"${SQL_ROOT_PASSWORD}" MatomoDB < backup.sql

# 6. Matomo-Dateien wiederherstellen
tar -xzf matomo_files.tar.gz -C volumes/html/

# 7. Stack vollständig starten
docker compose up -d

# 8. Permissions korrigieren
docker exec matomo-php-apache chown -R www-data:www-data /var/www/html/config
docker exec matomo-php-apache chown -R www-data:www-data /var/www/html/tmp
```

---

## 🔐 Sicherheit

### Private Verzeichnisse schützen

Der Stack schützt automatisch folgende Verzeichnisse vor öffentlichem Zugriff:

- `/tmp/` - Temporäre Dateien und Cache
- `/config/` - Konfigurationsdateien (inkl. `config.ini.php`)
- `/lang/` - Sprachdateien
- `/vendor/` - Composer Dependencies
- `.git/` - Git-Repository (falls vorhanden)
- Backup-Dateien (`.bak`, `.backup`, `.sql`, etc.)

**Konfiguration:** `configs/apache/matomo-security.conf`

Diese wird automatisch von Apache geladen. Bei Zugriff auf diese Verzeichnisse erhalten Besucher einen **403 Forbidden** Fehler.

### Passwörter rotieren

```bash
# 1. Neue Passwörter in .env eintragen
nano .env

# 2. Redis Passwort ändern
docker compose restart redis

# 3. MariaDB Passwörter ändern
docker exec matomo-sql mariadb -uroot -p"${OLD_PASSWORD}" -e "
ALTER USER 'root'@'%' IDENTIFIED BY '${NEW_ROOT_PASSWORD}';
ALTER USER 'MatomoUser'@'%' IDENTIFIED BY '${NEW_USER_PASSWORD}';
FLUSH PRIVILEGES;"

# 4. config.ini.php aktualisieren (Redis-Passwort)
nano volumes/html/config/config.ini.php

# 5. Stack neu starten
docker compose restart
```

### SSL/TLS Zertifikate

Werden automatisch von Traefik mit Let's Encrypt verwaltet.

**Prüfen:**
```bash
# SSL Labs Test
https://www.ssllabs.com/ssltest/analyze.html?d=ihre-domain.de
```

### Security Headers prüfen

```bash
curl -I https://ihre-domain.de
```

Sollte enthalten:
- `Strict-Transport-Security`
- `X-Frame-Options`
- `X-Content-Type-Options`
- `Content-Security-Policy`

---

## 📈 Skalierung

### Wenn Traffic steigt (>50.000 Page Views/Tag)

**QueuedTracking Plugin installieren:**

1. In Matomo: **Administration → Marketplace**
2. "QueuedTracking" suchen und installieren
3. In `docker-compose.yaml` zusätzlichen CronJob hinzufügen:

```yaml
# Ofelia Label hinzufügen:
- ofelia.job-exec.${COMPOSE_PROJECT_NAME}-queue-worker.schedule=* * * * *
- ofelia.job-exec.${COMPOSE_PROJECT_NAME}-queue-worker.command=php /var/www/html/console queuedtracking:process
```

4. Stack neu starten: `docker compose up -d`

### Resource Limits anpassen

In `docker-compose.yaml`:

```yaml
# MariaDB RAM erhöhen
sql:
  deploy:
    resources:
      limits:
        memory: "8G"  # von 4G auf 8G

# Redis RAM erhöhen
redis:
  command:
    - --maxmemory
    - 2gb  # von 1gb auf 2gb
```

Dann `configs/sql/custom.cnf` anpassen:

```ini
innodb_buffer_pool_size = 6G  # 75% von 8G
```

---

## 📚 Weitere Ressourcen

- **Matomo Dokumentation:** https://matomo.org/guides/
- **Traefik Labels:** https://github.com/csaeum/DockerStackTraefik/blob/main/LABELS-CHECKLIST.md
- **Ofelia CronJobs:** https://github.com/csaeum/DockerStackOfelia
- **Matomo Forums:** https://forum.matomo.org/
- **Performance FAQ:** https://matomo.org/faq/on-premise/how-to-configure-matomo-for-speed/

---

## 📝 Changelog

### Version 1.0 (2025-12-28)
- ✅ Initial Release
- ✅ Optimiert für 10-15 Sites, ~500 User/Tag
- ✅ Redis Caching Integration
- ✅ Traefik Security Middlewares
- ✅ Apache Security Config (blockiert /tmp/, /config/, etc.)
- ✅ Automatische Backups
- ✅ Ofelia CronJobs für Archivierung
- ✅ Performance-optimierte MariaDB Config
- ✅ PHP 8.3 mit OPcache + JIT

---

## 📄 Lizenz

Dieses Projekt ist unter der [GPL-3.0-or-later](LICENSE) Lizenz lizenziert.

---

## 🤝 Support & Unterstützung

**Made with ❤️ by WSC - Web SEO Consulting**

Dieses Projekt ist kostenlos und Open Source. Wenn es dir geholfen hat, freue ich mich über deine Unterstützung:

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

### Fragen oder Probleme?

- 🐛 **Bug Reports:** [GitHub Issues](https://github.com/csaeum/DockerStackMatomo/issues)
- 💬 **Diskussionen:** [GitHub Discussions](https://github.com/csaeum/DockerStackMatomo/discussions)
- 📧 **Kontakt:** [info@web-seo-consulting.eu](mailto:info@web-seo-consulting.eu)

---

**Weitere Sprachen:**
- [🇬🇧 English](README.en.md)
- [🇫🇷 Français](README.fr.md)
