---
source-path: modules/profile_haproxy
---

# Migration Plan: profile_haproxy

**TLDR**: This module configures HAProxy load balancers with dynamic backend discovery via PuppetDB, multi-level Hiera configuration (common → OS → datacenter → environment → cluster → node), SSL/TLS termination, statistics interface, session persistence via stick-tables, and firewall integration (firewalld/ufw). It exports balancermember resources for automatic service discovery and collects them to build backend server pools. The module supports 2 backends in common.yaml (webservers, api), adds 1 in cluster config (internal_monitoring), and overrides server lists at node level. Production Frankfurt cluster (lb01.fra.example.com) runs with 32768 maxconn, TLSv1.3, stats on port 9001, and weighted API backend routing.

## Service Type and Instances

**Service Type**: Load Balancer (HAProxy)

**Configured Instances**:
- **haproxy service**: Main load balancer daemon
  - Location/Path: /etc/haproxy/haproxy.cfg
  - Port/Socket: 80 (HTTP), 443 (HTTPS when SSL enabled), 9001 (stats on lb01.fra.example.com)
  - Key Config: 32768 maxconn, TLSv1.3, 60s timeouts, 3 retries

**Backend Pools** (after deep merge across hierarchy levels):
- **webservers**: roundrobin, port 8080, 3 servers (web1-3 at 10.0.1.10-12)
- **api**: leastconn, port 3000, 2 servers on lb01.fra.example.com (api1-fra weight 200, api2-fra weight 100)
- **internal_monitoring**: roundrobin, port 9090, 1 server (prom1-fra at 10.100.3.10)

## File Structure

```
site/
├── role/
│   └── manifests/
│       └── haproxy.pp
├── profile/
│   └── manifests/
│       ├── base/
│       │   └── base.pp
│       └── loadbalancer/
│           └── haproxy.pp
└── modules/
    ├── common/
    │   └── base_utils/
    │       └── manifests/
    │           └── init.pp
    └── linux/
        └── profile_haproxy/
            ├── manifests/
            │   ├── init.pp
            │   ├── install.pp
            │   ├── config.pp
            │   ├── discover.pp
            │   ├── service.pp
            │   └── firewall.pp
            ├── templates/
            │   ├── haproxy.cfg.erb
            │   ├── backend.conf.epp
            │   ├── 503.http.erb
            │   ├── 408.http.erb
            │   └── stick-tables.cfg.erb
            ├── data/
            │   ├── common.yaml
            │   ├── os/
            │   │   └── Debian.yaml
            │   ├── datacenter/
            │   │   └── dc1_fra.yaml
            │   ├── environment/
            │   │   └── production.yaml
            │   ├── cluster/
            │   │   └── haproxy_prod_fra.yaml
            │   └── nodes/
            │       └── lb01.fra.example.com.yaml
            └── lib/
                └── facter/
                    └── haproxy_version.rb
```

## Module Explanation

The module performs operations in this order:

### 1. role::haproxy (role/manifests/haproxy.pp)

Top-level role class that orchestrates the HAProxy configuration:
- Conditional execution based on `$facts['kernel'] == 'Linux'`
- `include profile::base::base` - Base system configuration
- `include profile::loadbalancer::haproxy` - HAProxy-specific profile

### 2. profile::base::base (profile/manifests/base/base.pp)

Base system configuration applied to all nodes:
- `include base_utils` - Common utility packages and MOTD
- Manages system-wide settings (NTP, logging, etc.)
- Provides foundation for service-specific profiles

### 3. base_utils (site/modules/common/base_utils/manifests/init.pp)

Common utilities and message of the day:
- **Parameters**: `$manage_motd` (boolean), `$utility_packages` (array)
- `file '/etc/motd'` → ensure: `file`, content: template, owner: `root`, group: `root`, mode: `0644`
- Iterates `$utility_packages.each` for package installation
- Conditional execution based on `$manage_motd` parameter

### 4. profile::loadbalancer::haproxy (profile/manifests/loadbalancer/haproxy.pp)

Wrapper profile class for HAProxy configuration:
- Uses `fact('environment')` for environment-specific logic
- `include profile_haproxy` - Delegates to main HAProxy module
- Provides abstraction layer between role and implementation

### 5. profile_haproxy (site/modules/linux/profile_haproxy/manifests/init.pp)

Main orchestration class with 24 parameters resolved via Hiera lookup with deep merge for backends hash:
- Sets package_name: `haproxy`
- Sets config_dir: `/etc/haproxy`
- Sets config_file: `/etc/haproxy/haproxy.cfg`
- Sets service_name: `haproxy`
- Sets user: `haproxy`, group: `haproxy`
- Sets stats_enabled: `true` (overridden from production's false by node-level), stats_port: `9001`, stats_uri: `/haproxy-stats`, stats_user: `admin`, stats_password: `Sensitive[ENCRYPTED]`
- Sets global_maxconn: `32768` (cluster override)
- Sets client_timeout: `60s`, server_timeout: `60s`, connect_timeout: `5s`, retries: `3`
- Sets ssl_enabled: `true`, ssl_cert_path: `/etc/ssl/certs`, ssl_key_path: `/etc/ssl/private`, ssl_ciphers: `ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384`, ssl_min_version: `TLSv1.3`
- Sets log_server: `10.100.0.5` (datacenter override), log_facility: `local0`, log_level: `warning`
- Sets backends: deep-merged hash with 3 backends (webservers, api, internal_monitoring)
- `contain profile_haproxy::install`
- `contain profile_haproxy::config`
- `contain profile_haproxy::discover`
- `contain profile_haproxy::service`
- `contain profile_haproxy::firewall`
- Sets ordering: `Class['profile_haproxy::install'] -> Class['profile_haproxy::config'] -> Class['profile_haproxy::discover'] ~> Class['profile_haproxy::service']`

### 6. profile_haproxy::install (site/modules/linux/profile_haproxy/manifests/install.pp)

Package and directory installation:
- `package 'haproxy'` → ensure: `present`
- `package 'hatop'` → ensure: `present` (Debian only, from extra_packages)
- `group 'haproxy'` → ensure: `present`, system: `true`
- `user 'haproxy'` → ensure: `present`, gid: `haproxy`, home: `/var/lib/haproxy`, shell: `/usr/sbin/nologin`, system: `true`
- `file '/etc/haproxy'` → ensure: `directory`, owner: `root`, group: `root`, mode: `0755`
- `file '/etc/haproxy/conf.d'` → ensure: `directory`, owner: `root`, group: `root`, mode: `0755`
- `file '/var/lib/haproxy'` → ensure: `directory`, owner: `haproxy`, group: `haproxy`, mode: `0750`

### 7. profile_haproxy::config (site/modules/linux/profile_haproxy/manifests/config.pp)

Configuration file management:
- `file '/etc/haproxy/haproxy.cfg'` (template `haproxy.cfg.erb`) → owner: `root`, group: `root`, mode: `0644`
  - Passes 19 variables including log_server, global_maxconn, ssl settings, stats configuration, backends hash
  - Template contains conditional SSL config block, conditional stats listen block, backend iteration for includes
- Iterates `$backends.each` — 3 iterations for **webservers**, **api**, **internal_monitoring**:
  - **webservers**: `file '/etc/haproxy/conf.d/webservers.cfg'` (template `backend.conf.epp`) with 3 servers (web1, web2, web3 at 10.0.1.10-12:8080)
  - **api**: `file '/etc/haproxy/conf.d/api.cfg'` (template `backend.conf.epp`) with 2 servers (api1-fra weight 200, api2-fra weight 100 at 10.100.2.10-11:3000)
  - **internal_monitoring**: `file '/etc/haproxy/conf.d/internal_monitoring.cfg'` (template `backend.conf.epp`) with 1 server (prom1-fra at 10.100.3.10:9090)
- `file '/etc/haproxy/errors'` → ensure: `directory`, owner: `root`, group: `root`, mode: `0755`
- Iterates error codes `['503', '408'].each` — 2 iterations:
  - **503**: `file '/etc/haproxy/errors/503.http'` (template `503.http.erb`) → owner: `root`, group: `root`, mode: `0644`
  - **408**: `file '/etc/haproxy/errors/408.http'` (template `408.http.erb`) → owner: `root`, group: `root`, mode: `0644`
- Conditional when `stick_table_enabled == true` (production environment):
  - `file '/etc/haproxy/conf.d/stick-tables.cfg'` (template `stick-tables.cfg.erb`) → owner: `root`, group: `root`, mode: `0644`
- Notifies: All config file changes trigger `~> service[haproxy]` restart

### 8. profile_haproxy::discover (site/modules/linux/profile_haproxy/manifests/discover.pp)

PuppetDB-based service discovery:
- Exports `@@haproxy::balancermember[$facts['networking']['fqdn']]` → listening_service: `webservers`, server_names: `$facts['networking']['fqdn']`, ipaddresses: `$facts['networking']['ip']`, ports: `8080`, options: `check`
- Collects `Haproxy::Balancermember <<| listening_service == 'webservers' |>>` from PuppetDB
- Queries PuppetDB for nodes with `Class['Profile::App_server']` in same `$facts['puppet_environment']`
- Iterates query results to create `haproxy::balancermember` resources dynamically
- Uses facts: `$facts['networking']['fqdn']`, `$facts['networking']['ip']`, `$facts['puppet_environment']`, `fact('environment')`

### 9. profile_haproxy::service (site/modules/linux/profile_haproxy/manifests/service.pp)

Service management with validation:
- `exec 'haproxy_config_check'` → command: `/usr/sbin/haproxy -c -f /etc/haproxy/haproxy.cfg`, refreshonly: `true`, logoutput: `on_failure`
- `service 'haproxy'` → ensure: `running`, enable: `true`, hasstatus: `true`, hasrestart: `true`
  - Subscribes to all config files (restart on changes)
  - Requires `exec[haproxy_config_check]` (validation before restart)
- `file '/etc/logrotate.d/haproxy'` (template) → owner: `root`, group: `root`, mode: `0644`

### 10. profile_haproxy::firewall (site/modules/linux/profile_haproxy/manifests/firewall.pp)

Firewall rule management:
- Case statement on `$firewall_provider`:
  - **ufw branch** (Debian):
    - `package 'ufw'` → ensure: `present`
    - `exec 'ufw_allow_http'` → command: `/usr/sbin/ufw allow 80/tcp`, unless: check if rule exists
    - `exec 'ufw_allow_https'` → command: `/usr/sbin/ufw allow 443/tcp`, unless: check if rule exists
    - `exec 'ufw_enable'` → command: `/usr/sbin/ufw --force enable`, unless: check if active
  - **firewalld branch** (RedHat): Not executed on Debian systems
  - **default branch**: `notify 'Unknown firewall provider: ${firewall_provider}'` for unsupported providers

## Variables

**Variable Flow Summary**: 30+ variables across 6 Hiera levels (common → OS → datacenter → environment → cluster → node)

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
- `profile_haproxy::stats_password`: `ENC[PKCS7,...]` (type: string, encrypted)
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
- `profile_haproxy::backends`: (type: hash, merge strategy: deep)
  - `webservers`: balance=roundrobin, port=8080, health_check=`httpchk GET /health`, health_interval=5s, servers=[web1@10.0.1.10, web2@10.0.1.11, web3@10.0.1.12]
  - `api`: balance=leastconn, port=3000, health_check=`httpchk GET /api/health`, health_interval=10s, servers=[api1@10.0.2.10, api2@10.0.2.11]
- `base_utils::manage_motd`: `true` (type: boolean)
- `base_utils::utility_packages`: `[]` (type: array)

**os/Debian.yaml (OS-specific)** → Migration note: OS-specific variables, loaded conditionally based on OS family
- `profile_haproxy::package_name`: `haproxy` (type: string)
- `profile_haproxy::config_dir`: `/etc/haproxy` (type: string)
- `profile_haproxy::firewall_provider`: `ufw` (type: string)
- `profile_haproxy::extra_packages`: `['hatop']` (type: array)
- `profile_haproxy::selinux_enabled`: `false` (type: boolean)

**datacenter/dc1_fra.yaml (datacenter-specific)** → Migration note: Datacenter-specific overrides for Frankfurt location
- `profile_haproxy::log_server`: `10.100.0.5` (type: string, overrides common)
- `profile_haproxy::ntp_servers`: `['10.100.0.1', '10.100.0.2']` (type: array)
- `profile_haproxy::backends`: (type: hash, deep-merged)
  - Updates server addresses to datacenter-local IPs

**environment/production.yaml (environment-specific)** → Migration note: Production environment settings
- `profile_haproxy::global_maxconn`: `16384` (type: integer, overrides common)
- `profile_haproxy::ssl_enabled`: `true` (type: boolean, overrides common)
- `profile_haproxy::log_level`: `warning` (type: string, overrides common)
- `profile_haproxy::client_timeout`: `60s` (type: string, overrides common)
- `profile_haproxy::server_timeout`: `60s` (type: string, overrides common)
- `profile_haproxy::stats_enabled`: `false` (type: boolean, overrides common)
- `profile_haproxy::stick_table_enabled`: `true` (type: boolean)
- `profile_haproxy::stick_table_size`: `200k` (type: string)
- `profile_haproxy::stick_table_expire`: `30m` (type: string)

**cluster/haproxy_prod_fra.yaml (cluster-specific)** → Migration note: Frankfurt production cluster settings
- `profile_haproxy::global_maxconn`: `32768` (type: integer, overrides production)
- `profile_haproxy::ssl_ciphers`: `ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384` (type: string, overrides common)
- `profile_haproxy::ssl_min_version`: `TLSv1.3` (type: string, overrides common)
- `profile_haproxy::backends`: (type: hash, deep-merged)
  - Adds `internal_monitoring` backend: balance=roundrobin, port=9090, health_check=`httpchk GET /-/healthy`, health_interval=15s, servers=[prom1-fra@10.100.3.10]

**nodes/lb01.fra.example.com.yaml (node-specific)** → Migration note: Host-specific overrides for lb01.fra.example.com
- `profile_haproxy::stats_enabled`: `true` (type: boolean, overrides production's false)
- `profile_haproxy::stats_port`: `9001` (type: integer, overrides common)
- `profile_haproxy::backends`: (type: hash, deep-merged)
  - Overrides `api` backend servers: [api1-fra@10.100.2.10 weight 200, api2-fra@10.100.2.11 weight 100]

### Variable Migration Summary

- **Common defaults**: 25 variables from common.yaml (base configuration for all nodes)
- **OS-specific variables**: 5 variables that vary by operating system family (Debian/RedHat)
- **Datacenter-specific variables**: 3 variables for Frankfurt datacenter location
- **Environment-specific variables**: 9 variables for production environment
- **Cluster-specific variables**: 4 variables for haproxy_prod_fra cluster
- **Host-specific variables**: 3 variables for individual host lb01.fra.example.com
- **Encrypted variables**: 1 variable (stats_password) requiring ansible-vault migration

### Cross-Level Overrides

Variables defined at multiple Hiera levels with final resolved values for lb01.fra.example.com:

- **profile_haproxy::global_maxconn**: defined at common (4096), production (16384), cluster (32768) → merge strategy: first → final: `32768`
- **profile_haproxy::client_timeout**: defined at common (30s), production (60s) → merge strategy: first → final: `60s`
- **profile_haproxy::server_timeout**: defined at common (30s), production (60s) → merge strategy: first → final: `60s`
- **profile_haproxy::ssl_enabled**: defined at common (false), production (true) → merge strategy: first → final: `true`
- **profile_haproxy::ssl_ciphers**: defined at common (AES128), cluster (AES256) → merge strategy: first → final: `ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384`
- **profile_haproxy::ssl_min_version**: defined at common (TLSv1.2), cluster (TLSv1.3) → merge strategy: first → final: `TLSv1.3`
- **profile_haproxy::log_server**: defined at common (127.0.0.1), datacenter (10.100.0.5) → merge strategy: first → final: `10.100.0.5`
- **profile_haproxy::log_level**: defined at common (info), production (warning) → merge strategy: first → final: `warning`
- **profile_haproxy::stats_enabled**: defined at common (true), production (false), node (true) → merge strategy: first → final: `true`
- **profile_haproxy::stats_port**: defined at common (9000), node (9001) → merge strategy: first → final: `9001`
- **profile_haproxy::backends**: defined at common (2 backends), datacenter (server address updates), cluster (adds internal_monitoring), node (api server weight override) → merge strategy: deep → final: 3 backends with node-specific server lists

### Merge Strategy Notes

- Variables using `deep` merge: `profile_haproxy::backends` - Hash values are recursively merged across all hierarchy levels
- Variables using `first` (default): All other variables - First value found in hierarchy wins, no merging

## Custom Types and Providers

**Custom Fact: haproxy_version**
- **File**: site/modules/linux/profile_haproxy/lib/facter/haproxy_version.rb
- **Purpose**: Extracts HAProxy version by executing `haproxy -v` and parsing output with regex `/version\s+(\d+\.\d+\.\d+)/`
- **Confine**: Only runs on Linux systems (`Facter.value(:kernel) == 'Linux'`)
- **Ansible Migration**: Use `ansible.builtin.package_facts` module to query installed package version, or create custom fact in `/etc/ansible/facts.d/haproxy.fact` (JSON/INI format), or use `ansible.builtin.command` with `haproxy -v` and register output, then parse with regex_search filter

Example Ansible equivalent:
```yaml
- name: Get HAProxy version
  ansible.builtin.command: haproxy -v
  register: haproxy_version_output
  changed_when: false

- name: Parse HAProxy version
  ansible.builtin.set_fact:
    haproxy_version: "{{ haproxy_version_output.stdout | regex_search('version\\s+(\\d+\\.\\d+\\.\\d+)', '\\1') | first }}"
```

## Dependencies

**External module dependencies**:
- puppetlabs-stdlib (9.7.0): Used for standard library functions (validate types, string manipulation)
- puppetlabs-concat (9.0.2): Available for fragment-based file assembly (not directly used)
- puppetlabs-firewall (8.1.3): Available but not used (module implements firewall via exec resources)
- puppetlabs-vcsrepo (6.1.0): Not used in this module
- puppet-redis (11.0.0): Not used in this module
- puppetlabs-apt (9.4.0): Not used in this module

**System package dependencies**:
- haproxy: Main load balancer package
- hatop: HAProxy monitoring tool (Debian only)
- ufw: Uncomplicated Firewall (Debian firewall provider)
- chrony: NTP client (from profile::base::base)
- rsyslog: System logging (from profile::base::base)

**Service dependencies**:
- Ordering: install → config → discover → service
- Notification chain: config changes ~> service restart (notify relationship triggers handler)
- Config validation: exec[haproxy_config_check] runs before service restart
- Firewall rules must be applied before service starts accepting connections

## Puppet Facts Used

- `$facts['kernel']`: OS kernel type (Linux), used for conditional execution in role::haproxy and custom fact confine
- `$facts['networking']['fqdn']`: Fully qualified domain name, used in discover.pp for exported resource title and server_names
- `$facts['networking']['ip']`: Primary IP address, used in discover.pp for backend member ipaddresses
- `$facts['puppet_environment']`: Puppet environment name (production/staging/dev), used in PuppetDB query to filter app servers by environment
- `fact('environment')`: Legacy fact function call in profile::loadbalancer::haproxy for environment-specific logic (differs from `$facts['puppet_environment']` in access method)

## Template Conversion Notes

**haproxy.cfg.erb** (19 variables, 3 logic blocks):
- **Variables**: log_server, log_facility, log_level, global_maxconn, user, group, ssl_enabled, ssl_ciphers, ssl_cert_path, connect_timeout, client_timeout, server_timeout, retries, stats_enabled, stats_port, stats_uri, stats_user, stats_password, backends
- **Ruby logic**:
  - Conditional SSL configuration block: `<% if @ssl_enabled -%>` renders ssl-default-bind-ciphers and tune.ssl.default-dh-param
  - Conditional stats listen block: `<% if @stats_enabled -%>` renders entire stats interface configuration with authentication
  - Backend iteration: `<% @backends.each do |name, config| -%>` generates include comments for conf.d files
- **Sensitive data**: stats_password must be unwrapped from Sensitive type in Puppet, use ansible-vault in Ansible
- **Ansible equivalent**: Use Jinja2 template with `{% if ssl_enabled %}`, `{% if stats_enabled %}`, `{% for name, config in backends.items() %}`

**backend.conf.epp** (7 variables, 4 logic blocks):
- **Variables**: backend_name, balance, port, servers (array of hashes), health_check (optional), health_interval (optional), ssl_enabled
- **EPP logic**:
  - Conditional health check: `<%- if $health_check { -%>` renders option line
  - Conditional health interval: `<%- if $health_interval { -%>` renders default-server inter line
  - Server iteration: `<%- $servers.each |$server| { -%>` generates server lines with name, address, port, weight
  - Nested conditional SSL: `<% if $ssl_enabled { %>` appends `ssl verify none` to server line
- **Ansible equivalent**: Use Jinja2 template with `{% if health_check is defined %}`, `{% for server in servers %}`, `{% if ssl_enabled %}`
- **Render count**: 3 times (webservers, api, internal_monitoring backends)

**503.http.erb and 408.http.erb** (error page templates):
- Standard HTTP error response files
- Minimal variables (error code and message)
- Static content with basic template variable substitution
- **Ansible equivalent**: Static files or simple Jinja2 templates

**stick-tables.cfg.erb** (2 variables, conditional rendering):
- **Variables**: stick_table_size, stick_table_expire
- Renders stick-table configuration for session persistence
- Only rendered when stick_table_enabled=true (production environment)
- **Ansible equivalent**: Conditional template inclusion with `when: stick_table_enabled`

## PuppetDB Dependencies

**Context**: PuppetDB provides centralized data store for cross-node resource sharing, node facts, and infrastructure queries. This module heavily relies on PuppetDB for dynamic backend discovery and service registration.

**Exported Resources**:
- `@@haproxy::balancermember[$facts['networking']['fqdn']]` in discover.pp
  - Exports this node as a backend member to PuppetDB
  - Parameters: listening_service=`webservers`, server_names=`$facts['networking']['fqdn']`, ipaddresses=`$facts['networking']['ip']`, ports=`8080`, options=`check`
  - **Migration note**: Replace with service discovery mechanism (Consul, etcd, AWS service discovery) or inventory-based dynamic groups. Exported resources enable automatic backend pool population without manual configuration - this pattern requires architectural redesign in Ansible.

**Resource Collectors**:
- `Haproxy::Balancermember <<| listening_service == 'webservers' |>>` in discover.pp (spaceship operator)
  - Collects all exported balancermember resources with listening_service=webservers from PuppetDB
  - Realizes them as local resources to dynamically populate backend server pools
  - **Migration note**: Use dynamic inventory plugin or query service registry, generate backend configuration from inventory groups. The spaceship operator (`<<| |>>`) is a PuppetDB-specific pattern for realizing exported resources - requires complete redesign in Ansible using inventory groups or external service discovery.

**PuppetDB Queries**:
- Query in discover.pp for app servers: Finds all nodes with `Class['Profile::App_server']` in same environment (`$facts['puppet_environment']`)
  - Returns certname and parameters for dynamic backend member creation
  - **Migration note**: Use inventory groups (e.g., `groups['app_servers']`), filter by environment variable, or query external CMDB/service registry. PuppetDB queries enable infrastructure-wide node discovery - Ansible equivalent requires inventory-based grouping or external data source integration.

**Migration Strategy**:
- **Option 1 - Inventory-based**: Define backend servers in Ansible inventory groups, use group_vars for environment-specific servers, template backend configs from inventory
- **Option 2 - Service Discovery**: Integrate with Consul/etcd, use consul_kv lookup plugin or API queries to discover backend members dynamically
- **Option 3 - Hybrid**: Use Ansible Tower/AWX dynamic inventory from external source (AWS EC2, VMware, etc.) combined with inventory grouping

## Checks for the Migration

**Files to verify**:
- /etc/haproxy/haproxy.cfg (main configuration)
- /etc/haproxy/conf.d/webservers.cfg (webservers backend)
- /etc/haproxy/conf.d/api.cfg (api backend)
- /etc/haproxy/conf.d/internal_monitoring.cfg (monitoring backend)
- /etc/haproxy/conf.d/stick-tables.cfg (session persistence, production only)
- /etc/haproxy/errors/503.http (service unavailable error page)
- /etc/haproxy/errors/408.http (request timeout error page)
- /etc/logrotate.d/haproxy (log rotation configuration)
- /etc/motd (message of the day from base_utils)

**Service endpoints to check**:
- Port 80 (HTTP) - haproxy service
- Port 443 (HTTPS) - hapr