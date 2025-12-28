# Matomo Docker Stack

[🇩🇪 Deutsch](README.md) | [🇬🇧 English](README.en.md) | 🇫🇷 **Français**

Stack Docker optimisé pour Matomo Analytics avec Traefik Reverse Proxy, mise en cache Redis et sauvegardes automatiques.

**Optimisé pour :** 10-15 sites web, ~500 utilisateurs/jour par site, ~10 000 pages vues/jour

---

## ✨ Fonctionnalités

- ✅ **PHP 8.3** avec OPcache + compilateur JIT
- ✅ **MariaDB 11.4** avec configuration optimisée
- ✅ **Redis 7** pour le cache et les sessions
- ✅ Intégration **Traefik** avec middlewares de sécurité
- ✅ **SSL/TLS automatique** via Let's Encrypt
- ✅ **Sauvegardes automatiques** de la base de données toutes les 6 heures
- ✅ **CronJobs Ofelia** pour l'archivage et la maintenance
- ✅ Configuration de **sécurité Apache** (bloque /tmp/, /config/, etc.)
- ✅ **tmpfs** pour améliorer les performances I/O

---

## 📋 Table des matières

1. [Prérequis](#prérequis)
2. [Démarrage rapide](#démarrage-rapide)
3. [Configuration Matomo](#configuration-matomo)
4. [Maintenance](#maintenance)
5. [Dépannage](#dépannage)
6. [Sauvegarde & Restauration](#sauvegarde--restauration)
7. [Mise à l'échelle](#mise-à-léchelle)

---

## 🔧 Prérequis

### Services externes requis

1. **Traefik Reverse Proxy** doit être en cours d'exécution
   - Réseau : `traefik_proxy_network`
   - Middlewares requis : `security-headers@file`, `compression@file`, `rate-limit@file`, `redirect-to-https@file`
   - Options TLS : `modern@file`

2. **Ofelia CronJob Scheduler** doit être en cours d'exécution
   - Dépôt : https://github.com/csaeum/DockerStackOfelia

### Configuration requise du serveur

- **RAM :** Au moins 6 Go disponibles
- **CPU :** 2+ cœurs recommandés
- **Stockage :** SSD avec min. 50 Go d'espace libre
- **Docker :** Version 24.0+
- **Docker Compose :** Version 2.20+

---

## 🚀 Démarrage rapide

### 1. Cloner le dépôt

```bash
git clone https://github.com/csaeum/DockerStackMatomo.git
cd DockerStackMatomo
```

### 2. Créer les volumes

```bash
mkdir -p volumes/html volumes/sql
```

### 3. Configurer l'environnement

```bash
cp .env.example .env
nano .env
```

**Paramètres importants dans `.env` :**

```bash
COMPOSE_PROJECT_NAME=matomo
BACKUPVOLUME=/chemin/vers/votre/sauvegarde
HOSTRULE=Host(`matomo.votre-domaine.fr`)

# Générer des mots de passe sécurisés :
# openssl rand -base64 32
REDIS_PASSWORD=VOTRE_MOT_DE_PASSE_GENERE
SQL_PASSWORD=VOTRE_MOT_DE_PASSE_GENERE
SQL_ROOT_PASSWORD=VOTRE_MOT_DE_PASSE_GENERE
```

### 4. Démarrer le stack

```bash
docker compose up -d
```

### 5. Terminer l'installation de Matomo

1. Ouvrir le navigateur : `https://votre-domaine.fr`
2. Suivre l'assistant de configuration Matomo :
   - **Hôte de base de données :** `sql`
   - **Nom de la base de données :** `MatomoDB`
   - **Utilisateur :** `MatomoUser`
   - **Mot de passe :** Votre `SQL_PASSWORD` du fichier `.env`

---

## ⚙️ Configuration Matomo

### 1. Activer le cache Redis (IMPORTANT !)

Éditer `volumes/html/config/config.ini.php` :

```bash
docker exec -it matomo-php-apache nano /var/www/html/config/config.ini.php
```

Ajouter à la fin :

```ini
[Cache]
backend = redis

[RedisCache]
host = "redis"
port = 6379
password = "VOTRE_MOT_DE_PASSE_REDIS"
database = 0
timeout = 0.0

[QueuedTracking]
backend = "redis"
host = "redis"
port = 6379
password = "VOTRE_MOT_DE_PASSE_REDIS"
database = 1
```

### 2. Désactiver l'archivage du navigateur

Dans le même fichier `config.ini.php` :

```ini
[General]
enable_browser_archiving_triggering = 0
browser_archiving_disabled_enforce = 1
```

### 3. Redémarrer le conteneur

```bash
docker compose restart php-apache
```

---

## ⏰ CronJobs automatiques

Configurés via Ofelia (dans `docker-compose.yaml`) :

1. **Archivage :** Toutes les heures (génère des rapports pour tous les sites)
2. **Suppression des logs :** Tous les jours à 2h (supprime les logs de plus de 180 jours)
3. **Nettoyage temporaire :** Tous les jours à 3h (purge les anciennes données d'archive)

### Vérifier les logs des CronJobs

```bash
docker logs -f ofelia

# Test manuel
docker exec matomo-php-apache php /var/www/html/console core:archive
```

---

## 🔧 Maintenance

### Afficher les logs

```bash
docker compose logs -f
docker compose logs -f php-apache
docker compose logs -f sql
```

### Redémarrer les conteneurs

```bash
docker compose restart
docker compose restart php-apache
```

### Mettre à jour les images

```bash
docker compose pull
docker compose up -d
docker image prune -f
```

---

## 🐛 Dépannage

### Échec de connexion à la base de données

```bash
docker compose ps sql
docker compose logs sql
docker exec matomo-sql mariadb-admin ping -uroot -p"${SQL_ROOT_PASSWORD}"
```

### Les rapports ne sont pas générés

```bash
docker logs ofelia
docker exec matomo-php-apache php /var/www/html/console core:archive
```

### Échec de connexion Redis

```bash
docker exec matomo-redis redis-cli -a "${REDIS_PASSWORD}" ping
docker compose logs redis
```

---

## 💾 Sauvegarde & Restauration

### Sauvegardes automatiques

Les sauvegardes SQL sont créées automatiquement par le conteneur `sqlbackup` :

- **Fréquence :** Toutes les 6 heures
- **Emplacement :** `${BACKUPVOLUME}/matomo/`
- **Rétention :** 12 dernières sauvegardes
- **Compression :** GZIP Niveau 9

### Sauvegarde manuelle

```bash
docker exec matomo-sql mariadb-dump \
    -uroot -p"${SQL_ROOT_PASSWORD}" \
    --single-transaction \
    MatomoDB > backup_$(date +%Y%m%d_%H%M%S).sql
```

### Restaurer une sauvegarde

```bash
docker compose down
docker compose up -d sql
docker exec -i matomo-sql mariadb -uroot -p"${SQL_ROOT_PASSWORD}" MatomoDB < backup.sql
docker compose up -d
```

---

## 📈 Mise à l'échelle

### Lorsque le trafic augmente (>50 000 pages vues/jour)

Installer le **plugin QueuedTracking** :

1. Dans Matomo : **Administration → Marketplace**
2. Rechercher "QueuedTracking" et installer
3. Ajouter dans `docker-compose.yaml` :

```yaml
- ofelia.job-exec.${COMPOSE_PROJECT_NAME}-queue-worker.schedule=* * * * *
- ofelia.job-exec.${COMPOSE_PROJECT_NAME}-queue-worker.command=php /var/www/html/console queuedtracking:process
```

### Augmenter les ressources

```yaml
# Dans docker-compose.yaml
sql:
  deploy:
    resources:
      limits:
        memory: "8G"  # de 4G à 8G

redis:
  command:
    - --maxmemory
    - 2gb  # de 1gb à 2gb
```

Mettre à jour `configs/sql/custom.cnf` :

```ini
innodb_buffer_pool_size = 6G  # 75% de 8G
```

---

## 📚 Ressources

- **Documentation Matomo :** https://matomo.org/guides/
- **Labels Traefik :** https://github.com/csaeum/DockerStackTraefik/blob/main/LABELS-CHECKLIST.md
- **CronJobs Ofelia :** https://github.com/csaeum/DockerStackOfelia
- **FAQ Performance :** https://matomo.org/faq/on-premise/how-to-configure-matomo-for-speed/

---

## 📝 Journal des modifications

### Version 1.0 (2025-12-28)
- ✅ Version initiale
- ✅ Optimisé pour 10-15 sites, ~500 utilisateurs/jour
- ✅ Intégration du cache Redis
- ✅ Middlewares de sécurité Traefik
- ✅ Configuration de sécurité Apache
- ✅ Sauvegardes automatiques
- ✅ CronJobs Ofelia
- ✅ MariaDB optimisé pour les performances
- ✅ PHP 8.3 avec OPcache + JIT

---

## 📄 Licence

Ce projet est sous licence [GPL-3.0-or-later](LICENSE).

---

## 🤝 Support & Contribution

**Fait avec ❤️ par WSC - Web SEO Consulting**

Ce projet est gratuit et open source. S'il vous a aidé, j'apprécie votre soutien :

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/csaeum)
[![GitHub Sponsors](https://img.shields.io/badge/GitHub%20Sponsors-ea4aaa?style=for-the-badge&logo=github-sponsors&logoColor=white)](https://github.com/sponsors/csaeum)
[![PayPal](https://img.shields.io/badge/PayPal-00457C?style=for-the-badge&logo=paypal&logoColor=white)](https://paypal.me/csaeum)

### Questions ou problèmes ?

- 🐛 **Rapports de bugs :** [GitHub Issues](https://github.com/csaeum/DockerStackMatomo/issues)
- 💬 **Discussions :** [GitHub Discussions](https://github.com/csaeum/DockerStackMatomo/discussions)
- 📧 **Contact :** [info@web-seo-consulting.eu](mailto:info@web-seo-consulting.eu)
