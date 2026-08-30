---
source-path: site-modules/profile_haproxy
---

# Migration Plan: profile_haproxy

**TLDR**: This module manages HAProxy load balancer installation, configuration, and service management. It supports SSL termination, statistics page, multiple backend configurations, and firewall management. The module also includes service discovery capabilities using PuppetDB.

## Service Type and Instances

**Service Type**: Load Balancer (HAProxy)

**Configured Instances**:
- **HAProxy**: HTTP/HTTPS load balancer
  - Location/Path: /etc/haproxy
  - Port/Socket: 80, 443 (when SSL enabled), 9000 (stats)
  - Key Config: Round-robin and least-connection balancing for web and API backends

## File Structure

- **Manifests**:
  - `site-modules/profile_haproxy/manifests/init.pp`
  - `site-modules/profile_haproxy/manifests/install.pp`
  - `site-modules/profile_haproxy/manifests/config.pp`
  - `site-modules/profile_haproxy/manifests/service.pp`
  - `site-modules/profile_haproxy/manifests/firewall.pp`
  - `site-modules/profile_haproxy/manifests/discover.pp`
- **Templates**:
  - `site-modules/profile_haproxy/templates/haproxy.cfg.erb`
  - `site-modules/profile_haproxy/templates/backend.conf.epp`
- **Data**:
  - `site-modules/profile_haproxy/data/common.yaml`
  - `site-modules/profile_haproxy/data/environment/production.yaml`
  - `site-modules/profile_haproxy/data/os/RedHat.yaml`
  - `site-modules/profile_haproxy/data/os/Debian.yaml`
- **Custom Facts**:
  - `site-modules/profile_haproxy/lib/facter/haproxy_version.rb`

## Module Explanation

The module performs operations in this order:

1. **role::haproxy** (`site-modules/role/manifests/haproxy.pp`):
   - Executes Linux-specific commands if $facts['kernel'].downcase == 'linux'
   - Includes base profile (not analyzed)
   - Includes HAProxy profile
   - Ensures base is applied before HAProxy

2. **profile::loadbalancer::haproxy** (`site-modules/profile/manifests/loadbalancer/haproxy.pp`):
   - Includes main HAProxy module

3. **profile_haproxy** (`site-modules/profile_haproxy/manifests/init.pp`):
   - Sets class parameters with Hiera lookups
   - Contains profile_haproxy::install
   - Contains profile_haproxy::config
   - Contains profile_haproxy::service
   - Contains profile_haproxy::firewall
   - Contains discovery class if enabled
   - Sets ordering: install -> config ~> service
   - With discovery: install -> config -> discover ~> service

4. **profile_haproxy::install** (`site-modules/profile_haproxy/manifests/install.pp`):
   - Installs haproxy package
   - Installs extra packages if defined
     - For RedHat: haproxy-systemd-wrapper, policycoreutils-python-utils
     - For Debian: hatop
   - Creates haproxy group
   - Creates haproxy user
   - Creates directories: /etc/haproxy, /etc/haproxy/conf.d, /var/lib/haproxy
   - Runs SELinux commands if enabled (setsebool -P haproxy_connect_any=1)

5. **profile_haproxy::config** (`site-modules/profile_haproxy/manifests/config.pp`):
   - Creates /etc/haproxy/haproxy.cfg from template haproxy.cfg.erb
     - Passes: log_server=127.0.0.1, log_facility=local0, log_level=warning (production), global_maxconn=16384 (production), user=haproxy, group=haproxy, ssl_enabled=true (production), ssl_ciphers="ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256", connect_timeout=5s, client_timeout=60s (production), server_timeout=60s (production), retries=3, stats_enabled=false (production), stats_port=9000, stats_uri=/haproxy-stats, stats_user=admin, stats_password=[encrypted]
   - Creates backend configuration files:
     - Creates /etc/haproxy/conf.d/webservers.cfg from template backend.conf.epp
       - Passes: backend_name=webservers, balance=roundrobin, port=8080, health_check=httpchk GET /health, health_interval=5s, servers=[{name: web1, address: 10.0.1.10, weight: 100}, {name: web2, address: 10.0.1.11, weight: 100}, {name: web3, address: 10.0.1.12, weight: 100}], ssl_enabled=true (production)
     - Creates /etc/haproxy/conf.d/api.cfg from template backend.conf.epp
       - Passes: backend_name=api, balance=leastconn, port=3000, health_check=httpchk GET /api/health, health_interval=10s, servers=[{name: api1, address: 10.0.2.10, weight: 100}, {name: api2, address: 10.0.2.11, weight: 100}], ssl_enabled=true (production)
   - Creates error pages:
     - Creates /etc/haproxy/errors/503.http
     - Creates /etc/haproxy/errors/408.http
   - Creates /etc/haproxy/errors directory
   - Creates stick table config if enabled (in production):
     - Creates /etc/haproxy/conf.d/stick-tables.cfg with stick_table_size=200k, stick_table_expire=30m

6. **profile_haproxy::service** (`site-modules/profile_haproxy/manifests/service.pp`):
   - Creates /etc/systemd/system/haproxy.service.d directory
   - Creates /etc/systemd/system/haproxy.service.d/override.conf
   - Runs systemctl daemon-reload when needed
   - Ensures haproxy service is running and enabled
   - Runs haproxy config check when needed
   - Creates /etc/logrotate.d/haproxy

7. **profile_haproxy::firewall** (`site-modules/profile_haproxy/manifests/firewall.pp`):
   - Handles firewall configuration based on provider:
     - For firewalld (RedHat): Reloads firewalld when needed
     - For ufw (Debian): Installs ufw, allows http/https, enables ufw
     - For none: No action (firewall management disabled)
     - For unknown providers: Sends notification

8. **profile_haproxy::discover** (`site-modules/profile_haproxy/manifests/discover.pp`):
   - Exports current node as balancermember for webservers backend
   - Collects exported balancermembers for webservers backend
   - Creates balancermembers for each app server found via PuppetDB query

## Variables

**Variable Flow Summary**: 30+ variables across 4 Hiera levels (common, environment, OS, node)

### Variable Definitions

**common.yaml (defaults)** → Migration note: Base defaults for all nodes
- `profile_haproxy::package_name`: `haproxy` (type: string)
- `profile_haproxy::config_dir`: `/etc/haproxy` (type: string)
- `profile_haproxy::config_file`: `/etc/haproxy/haproxy.cfg` (type: string)
- `profile_haproxy::service_name`: `haproxy` (type: string)
- `profile_haproxy::user`: `haproxy` (type: string)
- `profile_haproxy::group`: `haproxy` (type: string)
- `profile_haproxy::stats_enabled`: `true` (type: boolean)
- `profile_haproxy::stats_port`: `9000` (type: integer)
- `profile_haproxy::stats_uri`: `/haproxy-stats` (type: string)
- `profile_haproxy::stats_user`: `admin` (type: string)
- `profile_haproxy::stats_password`: `[encrypted]` (type: string)
- `profile_haproxy::global_maxconn`: `4096` (type: integer)
- `profile_haproxy::client_timeout`: `30s` (type: string)
- `profile_haproxy::server_timeout`: `30s` (type: string)
- `profile_haproxy::connect_timeout`: `5s` (type: string)
- `profile_haproxy::retries`: `3` (type: integer)
- `profile_haproxy::ssl_enabled`: `false` (type: boolean)
- `profile_haproxy::ssl_cert_path`: `/etc/ssl/certs` (type: string)
- `profile_haproxy::ssl_key_path`: `/etc/ssl/private` (type: string)
- `profile_haproxy::ssl_ciphers`: `ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256` (type: string)
- `profile_haproxy::ssl_min_version`: `TLSv1.2` (type: string)
- `profile_haproxy::log_server`: `127.0.0.1` (type: string)
- `profile_haproxy::log_facility`: `local0` (type: string)
- `profile_haproxy::log_level`: `info` (type: string)
- `profile_haproxy::backends`: (type: hash) with webservers and api backends

**environment/production.yaml** → Migration note: Production environment overrides
- `profile_haproxy::global_maxconn`: `16384` (type: integer)
- `profile_haproxy::ssl_enabled`: `true` (type: boolean)
- `profile_haproxy::log_level`: `warning` (type: string)
- `profile_haproxy::client_timeout`: `60s` (type: string)
- `profile_haproxy::server_timeout`: `60s` (type: string)
- `profile_haproxy::stats_enabled`: `false` (type: boolean)
- `profile_haproxy::stick_table_enabled`: `true` (type: boolean)
- `profile_haproxy::stick_table_size`: `200k` (type: string)
- `profile_haproxy::stick_table_expire`: `30m` (type: string)

**os/RedHat.yaml** → Migration note: RedHat-specific variables
- `profile_haproxy::package_name`: `haproxy` (type: string)
- `profile_haproxy::config_dir`: `/etc/haproxy` (type: string)
- `profile_haproxy::firewall_provider`: `firewalld` (type: string)
- `profile_haproxy::firewall_zone`: `public` (type: string)
- `profile_haproxy::extra_packages`: `['haproxy-systemd-wrapper', 'policycoreutils-python-utils']` (type: array)
- `profile_haproxy::selinux_enabled`: `true` (type: boolean)

**os/Debian.yaml** → Migration note: Debian-specific variables
- `profile_haproxy::package_name`: `haproxy` (type: string)
- `profile_haproxy::config_dir`: `/etc/haproxy` (type: string)
- `profile_haproxy::firewall_provider`: `ufw` (type: string)
- `profile_haproxy::extra_packages`: `['hatop']` (type: array)
- `profile_haproxy::selinux_enabled`: `false` (type: boolean)

### Variable Migration Summary

- **Common defaults**: 24 variables from common.yaml (base configuration for all nodes)
- **OS-specific variables**: 6 variables that vary by operating system family
- **Environment-specific variables**: 9 variables that vary by deployment environment
- **Encrypted variables**: 1 variable that is encrypted (stats_password) and needs secure storage

### Cross-Level Overrides

Variables defined at multiple Hiera levels:
- **profile_haproxy::global_maxconn**: defined at common and environment levels, merge strategy: first
- **profile_haproxy::ssl_enabled**: defined at common and environment levels, merge strategy: first
- **profile_haproxy::log_level**: defined at common and environment levels, merge strategy: first
- **profile_haproxy::client_timeout**: defined at common and environment levels, merge strategy: first
- **profile_haproxy::server_timeout**: defined at common and environment levels, merge strategy: first
- **profile_haproxy::stats_enabled**: defined at common and environment levels, merge strategy: first
- **profile_haproxy::package_name**: defined at common and OS levels, merge strategy: first
- **profile_haproxy::config_dir**: defined at common and OS levels, merge strategy: first
- **profile_haproxy::backends**: defined at common level, merge strategy: deep

### Merge Strategy Notes

- Variables using `hash` merge - Hash values from multiple levels are merged (shallow merge)
- Variables using `deep` merge - Hash values are recursively merged (deep merge)
- Variables using `first` (default) - First value found wins, no merging

## Custom Types and Providers

- **fact: haproxy_version**
  - File: `site-modules/profile_haproxy/lib/facter/haproxy_version.rb`
  - Description: Detects the installed HAProxy version on Linux systems by executing 'haproxy -v' and extracting the version number using regex.

## Dependencies

**External module dependencies**:
- puppetlabs-stdlib (version: 9.7.0)
- puppetlabs-concat (version: 9.0.2)
- puppetlabs-firewall (version: 8.1.3)

**System package dependencies**:
- haproxy
- OS-specific packages:
  - RedHat: haproxy-systemd-wrapper, policycoreutils-python-utils
  - Debian: hatop

**Service dependencies**:
- systemd (for service management)
- firewall service (firewalld on RedHat, ufw on Debian)

## Puppet Facts Used

- `$facts['kernel']`: Determines if running on Linux
- `$facts['networking']['fqdn']`: Used for server identification in discovery
- `$facts['networking']['ip']`: Used for server IP in discovery
- `$facts['puppet_environment']`: Used for environment-specific discovery

## Template Conversion Notes

**haproxy.cfg.erb**:
- Variables: log_server, log_facility, log_level, global_maxconn, user, group, ssl_enabled, ssl_ciphers, connect_timeout, client_timeout, server_timeout, retries, stats_enabled, stats_port, stats_uri, stats_user, stats_password, ssl_enabled, ssl_cert_path, backends
- Logic blocks: SSL configuration (conditional), stats page configuration (conditional), backend inclusion

**backend.conf.epp**:
- Variables: backend_name, balance, port, servers, health_check, health_interval, ssl_enabled
- Logic blocks: Health check configuration (conditional), server iteration with weight and SSL settings

## PuppetDB Dependencies

**Exported Resources** (`@@`):
- `@@haproxy::balancermember[$facts['networking']['fqdn']]`: Exports the current node as a balancermember for the webservers backend. Migration note: This requires cross-node data sharing for service discovery.

**Resource Collectors** (`<<| |>>`):
- `Haproxy::Balancermember <| listening_service == 'webservers' |>`: Collects all exported balancermembers for the webservers backend. Migration note: This requires node discovery capabilities.

**PuppetDB Queries**:
- Query to find app servers in the same environment: Used to dynamically discover and configure backend servers. Migration note: This requires infrastructure data access patterns.

## Checks for the Migration

**Files to verify**:
- `/etc/haproxy/haproxy.cfg`
- `/etc/haproxy/conf.d/webservers.cfg`
- `/etc/haproxy/conf.d/api.cfg`
- `/etc/haproxy/conf.d/stick-tables.cfg` (production only)
- `/etc/haproxy/errors/503.http`
- `/etc/haproxy/errors/408.http`
- `/etc/systemd/system/haproxy.service.d/override.conf`
- `/etc/logrotate.d/haproxy`

**Service endpoints to check**:
- HTTP: port 80
- HTTPS: port 443 (when SSL enabled)
- Stats page: port 9000 (when stats enabled)

**Templates rendered**:
- haproxy.cfg.erb (1 instance)
- backend.conf.epp (2 instances: webservers, api)

## Pre-flight checks:
```bash
# Service status command
systemctl status haproxy

# Configuration validation
haproxy -c -f /etc/haproxy/haproxy.cfg

# HTTP/HTTPS connectivity
curl -k https://localhost/
curl -k http://localhost:80/

# Stats page check (when enabled)
curl -k http://localhost:9000/haproxy-stats

# Backend health checks
curl -k http://localhost:8080/health
curl -k http://localhost:3000/api/health
```