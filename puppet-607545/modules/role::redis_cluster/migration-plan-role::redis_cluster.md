---
source-path: site/role/manifests/redis_cluster.pp
---

# Migration Plan: role::redis_cluster

**TLDR**: This module sets up a Redis cluster node with a basic Redis server installation. It configures Redis with password authentication, memory limits, and append-only persistence.

## Service Type and Instances

**Service Type**: Cache (Redis)

**Configured Instances**:
- **default**: Redis server instance
  - Location/Path: /var/lib/redis
  - Port/Socket: 6379
  - Key Config: bind=0.0.0.0, password authentication, append-only persistence

## File Structure

- **Role Manifests**:
  - `site/role/manifests/redis_cluster.pp`
- **Profile Manifests**:
  - `site/profile/manifests/base/base.pp`
  - `site/profile/manifests/cache/redis.pp`
- **Module Manifests**:
  - `modules/profile_redis_cluster/manifests/init.pp`
  - `modules/profile_redis_cluster/manifests/install.pp`
- **Utility Manifests**:
  - `site/modules/common/base_utils/manifests/init.pp`
- **Dependency Manifests**:
  - `migration-dependencies/redis/manifests/init.pp`
  - `migration-dependencies/redis/manifests/preinstall.pp`
  - `migration-dependencies/redis/manifests/install.pp`
  - `migration-dependencies/redis/manifests/config.pp`
  - `migration-dependencies/redis/manifests/service.pp`
  - `migration-dependencies/redis/manifests/instance.pp`
  - `migration-dependencies/redis/manifests/ulimit.pp`
  - `migration-dependencies/redis/manifests/dnfmodule.pp`

## Module Explanation

The module performs operations in this order:

1. **role::redis_cluster** (`site/role/manifests/redis_cluster.pp`):
   - Checks if kernel is Linux and sets exec path to `/usr/bin:/bin:/usr/sbin:/sbin`
   - Includes `::profile::base::base`
   - Contains `::profile::cache::redis`
   - Sets ordering: `Class['::profile::base::base'] -> Class['::profile::cache::redis']`

2. **profile::base::base** (`site/profile/manifests/base/base.pp`):
   - If $manage_utils (true), includes base_utils
     - If $manage_motd (true), creates file '/etc/motd' with content from template
   - If $manage_ntp and Linux kernel (true), installs chrony package and ensures chronyd service is running
   - If $manage_syslog and Linux kernel (true), installs rsyslog package and ensures rsyslog service is running

3. **profile::cache::redis** (`site/profile/manifests/cache/redis.pp`):
   - Includes `profile_redis_cluster` class

4. **profile_redis_cluster** (`modules/profile_redis_cluster/manifests/init.pp`):
   - Sets class parameters: redis_port=6379, redis_password='CHANGEME', maxmemory_mb=2048, maxmemory_policy='allkeys-lru'
   - Contains `profile_redis_cluster::install`

5. **profile_redis_cluster::install** (`modules/profile_redis_cluster/manifests/install.pp`):
   - Includes `redis` class with parameters: bind='0.0.0.0', port=6379, requirepass='CHANGEME', maxmemory='2048mb', appendonly=true, appendfsync='everysec', manage_package=true

6. **redis** (`migration-dependencies/redis/manifests/init.pp`):
   - Contains `redis::preinstall`
   - Contains `redis::install`
   - Contains `redis::config`
   - Contains `redis::service`
   - Sets ordering: `redis::preinstall -> redis::install -> redis::config ~> redis::service`

7. **redis::preinstall** (`migration-dependencies/redis/manifests/preinstall.pp`):
   - If $redis::manage_repo (false), no repository management occurs

8. **redis::install** (`migration-dependencies/redis/manifests/install.pp`):
   - If $redis::manage_package (true), installs 'redis' package
   - If $redis::dnf_module_stream (undef), no DNF module management occurs

9. **redis::config** (`migration-dependencies/redis/manifests/config.pp`):
   - Creates directory '/etc/redis' with owner 'redis', group 'redis', mode '0750'
   - Creates directory '/var/log/redis' with owner 'redis', group 'redis', mode '0750'
   - Creates directory '/var/lib/redis' with owner 'redis', group 'redis', mode '0750'
   - If $redis::default_install (true), creates redis::instance 'default' with parameters:
     - bind: '0.0.0.0'
     - port: 6379
     - requirepass: 'CHANGEME'
     - maxmemory: '2048mb'
     - maxmemory_policy: 'allkeys-lru'
     - appendonly: true
     - appendfsync: 'everysec'
   - Creates file '/etc/redis/redis.conf.puppet' with Redis configuration
   - Executes copy of '/etc/redis/redis.conf.puppet' to '/etc/redis/redis.conf'
   - If $redis::ulimit_managed (true), includes 'redis::ulimit' class
     - Creates file '/etc/systemd/system/redis-server.service.d/limit.conf' with ulimit configuration

10. **redis::service** (`migration-dependencies/redis/manifests/service.pp`):
    - If $redis::service_manage (true), ensures 'redis-server' service is running and enabled

## Variables

**Variable Flow Summary**: 4 variables in profile_redis_cluster module, plus additional variables in supporting modules

### Variable Definitions

**profile_redis_cluster parameters** → Migration note: Core Redis configuration
- `profile_redis_cluster::redis_port`: `6379` (type: integer)
- `profile_redis_cluster::redis_password`: `CHANGEME` (type: string)
- `profile_redis_cluster::maxmemory_mb`: `2048` (type: integer)
- `profile_redis_cluster::maxmemory_policy`: `allkeys-lru` (type: string)

**profile::base::base parameters** → Migration note: Base system configuration
- `profile::base::manage_ntp`: `true` (type: boolean)
- `profile::base::manage_syslog`: `true` (type: boolean)
- `profile::base::manage_utils`: `true` (type: boolean)

**base_utils parameters** → Migration note: Utility configuration
- `base_utils::manage_motd`: `true` (type: boolean)
- `base_utils::motd_template`: `base_utils/motd.erb` (type: string)
- `base_utils::utility_packages`: `[]` (type: array)

### Variable Migration Summary

- **Common defaults**: 4 variables from profile_redis_cluster module (base configuration for all nodes)
- **OS-specific variables**: 0 variables
- **Environment-specific variables**: 0 variables
- **Host-specific variables**: 0 variables
- **Encrypted variables**: 1 variable needing secure storage (redis_password)

### Cross-Level Overrides

No cross-level overrides detected for the Redis cluster module.

### Merge Strategy Notes

No merge strategies identified in this module.

## Dependencies

**External module dependencies**:
- puppet-redis (forge, version: 11.0.0)

**System package dependencies**:
- redis
- chrony
- rsyslog

**Service dependencies**:
- chronyd
- rsyslog
- redis-server

## Puppet Facts Used

- `$facts['kernel']`: Determines if the system is Linux
- `$facts['os']['family']`: Used by Redis module for OS-specific configurations
- `$facts['os']['name']`: Used by Redis module for OS-specific configurations
- `fact('environment')`: Used to determine environment name

## Template Conversion Notes

**base_utils/motd.erb**:
- Used for generating the message of the day file
- Simple template with minimal logic

## Checks for the Migration

**Files to verify**:
- `/etc/redis/redis.conf`
- `/etc/systemd/system/redis-server.service.d/limit.conf`
- `/etc/motd`
- `/var/lib/redis`
- `/var/log/redis`

**Service endpoints to check**:
- Redis server on port 6379

**Templates rendered**:
- Redis configuration file (1 instance)
- MOTD file (1 instance)

## Pre-flight checks:
```bash
# Service status commands
systemctl status redis-server
systemctl status chronyd
systemctl status rsyslog

# Redis instance checks
redis-cli ping
redis-cli -a CHANGEME info
redis-cli -a CHANGEME CONFIG GET maxmemory
redis-cli -a CHANGEME CONFIG GET appendonly

# Configuration validation
grep -i "bind 0.0.0.0" /etc/redis/redis.conf
grep -i "port 6379" /etc/redis/redis.conf
grep -i "requirepass" /etc/redis/redis.conf
grep -i "maxmemory 2048mb" /etc/redis/redis.conf
grep -i "appendonly yes" /etc/redis/redis.conf
```