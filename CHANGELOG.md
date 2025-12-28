# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2025-12-28

### Added
- Initial release of Matomo Docker Stack
- PHP 8.3 with OPcache and JIT compiler
- MariaDB 11.4 with performance-optimized configuration
- Redis 7 for caching and session management
- Traefik integration with security middlewares
  - `security-headers@file` for HSTS, X-Frame-Options, CSP
  - `compression@file` for Gzip/Brotli compression
  - `rate-limit@file` for DoS protection
  - `redirect-to-https@file` for HTTP to HTTPS redirection
- Automatic SSL/TLS certificates via Let's Encrypt
- Automatic database backups every 6 hours
- Ofelia CronJobs for maintenance
  - Hourly archiving of all sites
  - Daily log deletion (180 days retention)
  - Daily temporary data cleanup
- Apache security configuration
  - Blocks public access to `/tmp/`, `/config/`, `/vendor/`, etc.
  - Protects sensitive files (config.ini.php, backups)
- tmpfs mount for improved I/O performance
- Multi-language documentation (DE, EN, FR)
- GitHub Actions CI/CD workflows
  - Automated testing on push/PR
  - YAML linting
  - Security scanning with Gitleaks
  - Container build tests
  - Automated releases on tag push
- Comprehensive README with installation, configuration, and troubleshooting guides

### Configuration
- MariaDB optimized for 10-15 sites, ~500 users/day
  - `innodb_buffer_pool_size = 3G`
  - `innodb_flush_log_at_trx_commit = 2` for better performance
  - `max_connections = 200`
- Redis with 1GB memory limit and volatile-lru eviction policy
- PHP with 768M memory limit and 120s execution time
- Session cookies configured as secure and httponly

### Security
- All sensitive data moved to `.env.example` with placeholders
- `.env` excluded from Git repository via `.gitignore`
- Apache security headers configured
- Private directories properly protected
- No hardcoded passwords or domains in repository

### Documentation
- Detailed installation guide
- Matomo configuration instructions (Redis, browser archiving)
- CronJob documentation
- Troubleshooting section
- Performance monitoring guidelines
- Backup and restore procedures
- Scaling recommendations for high-traffic scenarios
- Available in German, English, and French

---

## How to Upgrade

When upgrading between versions, always:

1. **Backup first:**
   ```bash
   docker exec matomo-sql mariadb-dump -uroot -p"${SQL_ROOT_PASSWORD}" MatomoDB > backup.sql
   ```

2. **Pull latest changes:**
   ```bash
   git pull
   ```

3. **Update containers:**
   ```bash
   docker compose pull
   docker compose up -d
   ```

4. **Check logs:**
   ```bash
   docker compose logs -f
   ```

---

## Support

For questions, issues, or feature requests:
- 🐛 [GitHub Issues](https://github.com/csaeum/DockerStackMatomo/issues)
- 💬 [GitHub Discussions](https://github.com/csaeum/DockerStackMatomo/discussions)
