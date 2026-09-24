---
source-path: site-modules/profile
---

# Migration Plan: profile

**TLDR**: The profile module implements a layered configuration approach with base OS settings and HAProxy load balancer functionality. It follows the roles and profiles pattern where the role::haproxy class composes the profile::base::base and profile::loadbalancer::haproxy classes with proper ordering.

## Service Type and Instances

**Service Type**: Load Balancer (HAProxy)

**Configured Instances**:
- **HAProxy**: Load balancer service
  - Location/Path: System default
  - Port/Socket: Not explicitly defined in the examined code
  - Key Config: Stats authentication enabled with password from Hiera

## File Structure

- **Manifests**:
  - site-modules/profile/manifests/base/base.pp
  - site-modules/profile/manifests/loadbalancer/haproxy.pp
  - site-modules/role/manifests/haproxy.pp

## Module Explanation

The module performs operations in this order:

1. **role::haproxy** (`site-modules/role/manifests/haproxy.pp`):
   - If $facts['kernel'].downcase == 'linux', sets path: '/usr/bin:/bin:/usr/sbin:/sbin' (default for all Exec resources)
   - Includes ::profile::base::base
   - Contains ::profile::loadbalancer::haproxy
   - Sets ordering: Class['::profile::base::base'] -> Class['::profile::loadbalancer::haproxy']

2. **profile::base::base** (`site-modules/profile/manifests/base/base.pp`):
   - Sets class parameters: manage_ntp=true, manage_syslog=true, manage_utils=true
   - If $manage_utils, includes base_utils class
   - If $manage_ntp and $facts['kernel'] == 'Linux':
     - Installs 'chrony' package
     - Ensures 'chronyd' service is running and enabled
   - If $manage_syslog and $facts['kernel'] == 'Linux':
     - Installs 'rsyslog' package
     - Ensures 'rsyslog' service is running and enabled

3. **profile::loadbalancer::haproxy** (`site-modules/profile/manifests/loadbalancer/haproxy.pp`):
   - Sets class parameters: environment_name=fact('environment')
   - Includes 'profile_haproxy' class which configures:
     - Package: from profile_haproxy::package_name
     - Stats authentication: using profile_haproxy::stats_password
     - Firewall: using profile_haproxy::firewall_provider
     - Additional packages: from profile_haproxy::extra_packages array
     - SSL: enabled/disabled based on profile_haproxy::ssl_enabled
     - Stick tables: enabled/disabled based on profile_haproxy::stick_table_enabled

## Variables

**Variable Flow Summary**: 14 variables across multiple Hiera levels

### Variable Definitions

**common.yaml (defaults)** → Migration note: Base defaults for all nodes
- `profile_haproxy::package_name`: `haproxy` (type: string)
- `profile_haproxy::stats_password`: `[password value]` (type: string)
- `profile_haproxy::firewall_provider`: `iptables` (type: string)
- `profile_haproxy::extra_packages`: `[]` (type: array)
- `profile_haproxy::ssl_enabled`: `true` (type: boolean)
- `profile_haproxy::stick_table_enabled`: `false` (type: boolean)
- `ntp::servers`: `[array of NTP servers]` (type: array)
- `syslog::server`: `central-log.example.com` (type: string)
- `syslog::facility`: `local0` (type: string)
- `profile::base::manage_ntp`: `true` (type: boolean)
- `profile::base::manage_syslog`: `true` (type: boolean)
- `profile::base::manage_utils`: `true` (type: boolean)

**environment/production.yaml** → Migration note: Production environment overrides
- `ntp::servers`: `[production NTP servers]` (type: array)
- `syslog::server`: `prod-log.example.com` (type: string)
- `syslog::facility`: `local1` (type: string)

**environment/staging.yaml** → Migration note: Staging environment overrides
- `syslog::server`: `stage-log.example.com` (type: string)

### Variable Migration Summary

- **Common defaults**: 12 variables from common.yaml (base configuration for all nodes)
- **Environment-specific variables**: 4 variables that vary by deployment environment (production, staging)
- **Host-specific variables**: 0 variables for individual host overrides
- **Encrypted variables**: 1 variable that is encrypted (stats_password) and needs secure storage

### Cross-Level Overrides

Variables defined at multiple Hiera levels:
- **ntp::servers**: defined at common and production levels, merge strategy: first
- **syslog::server**: defined at common, production, and staging levels, merge strategy: first
- **syslog::facility**: defined at common and production levels, merge strategy: first

### Merge Strategy Notes

- Variables using `first` (default) - First value found wins, no merging

## Dependencies

**External module dependencies**:
- puppetlabs-stdlib (forge, version: 9.7.0)
- puppetlabs-concat (forge, version: 9.0.2)
- puppetlabs-firewall (forge, version: 8.1.3)
- puppetlabs-vcsrepo (forge, version: 6.1.0)
- puppet-redis (forge, version: 11.0.0)
- puppet-systemd (forge, version: 7.1.0)
- puppetlabs-inifile (forge, version: 6.1.1)
- puppetlabs-apt (forge, version: 9.4.0)

**System package dependencies**:
- chrony
- rsyslog
- haproxy (implied from profile_haproxy::package_name)

**Service dependencies**:
- profile::base::base must be applied before profile::loadbalancer::haproxy

## Puppet Facts Used

- `$facts['kernel']`: Determines if the system is Linux
- `fact('environment')`: Gets the Puppet environment name

## Checks for the Migration

**Files to verify**:
- System configuration files for chrony
- System configuration files for rsyslog
- HAProxy configuration files (likely /etc/haproxy/haproxy.cfg)

**Service endpoints to check**:
- HAProxy service status
- HAProxy stats page (URL not specified in the code)
- Chronyd service status
- Rsyslog service status

**Templates rendered**: None directly referenced in the examined code

## Pre-flight checks:
```bash
# Service status commands
systemctl status chronyd
systemctl status rsyslog
systemctl status haproxy

# Configuration validation commands
haproxy -c -f /etc/haproxy/haproxy.cfg
```