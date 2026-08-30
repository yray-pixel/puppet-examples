# MIGRATION FROM PUPPET TO ANSIBLE

This repository contains a Puppet control repository implementing a three-tier web application stack with HAProxy load balancing, Redis caching, and a Python application backend. The migration involves converting 4 Puppet modules, 3 roles, 4 wrapper profiles, and a complex 21-level Hiera hierarchy to Ansible roles, playbooks, and inventory-based variable management. Estimated timeline: 6-8 weeks for a team of 2-3 engineers, including testing and validation.

## Module Migration Plan

This repository contains Puppet modules that need individual migration planning:

### MODULE INVENTORY

**CRITICAL PATH VERIFICATION:**
All paths have been verified using `list_directory` and `file_search` tools.

- **profile_app_stack**:
    - Description: Python application stack with gunicorn WSGI server, PostgreSQL database integration, Celery worker management, systemd service hardening, log rotation, automated backups, and health check monitoring with Prometheus metrics push support
    - Path: modules/profile_app_stack
    - Technology: Puppet
    - Key Features: Git repository cloning, Python virtualenv with pip dependencies, PostgreSQL URL construction with password encoding, environment-specific configuration (DEBUG mode, CORS settings), systemd unit with security hardening (NoNewPrivileges, ProtectSystem, PrivateTmp), daily log rotation with compression, PostgreSQL backup script with 30-day retention, health check script with Prometheus pushgateway integration

- **profile_haproxy**:
    - Description: HAProxy load balancer with SSL/TLS termination, backend health checks, stick-table session persistence, UFW firewall rules, custom error pages, and stats interface with authentication
    - Path: modules/profile_haproxy
    - Technology: Puppet
    - Key Features: Dynamic backend configuration via Hiera deep merge, SSL/TLS with configurable cipher suites and protocols (TLSv1.2/1.3), HTTP/HTTPS frontend with X-Forwarded-For headers, backend health checks with rise/fall thresholds, stick-table for session persistence, UFW firewall rules for ports 80/443/stats, custom 503 error page, stats interface with authentication, environment-specific overrides (production: maxconn 16384, SSL enabled; Frankfurt cluster: maxconn 32768, TLSv1.3 only)

- **profile_redis_cluster**:
    - Description: Redis in-memory cache with authentication, persistence configuration, memory eviction policies, and systemd service management
    - Path: modules/profile_redis_cluster
    - Technology: Puppet
    - Key Features: Redis installation via puppet-redis module, bind address configuration, requirepass authentication, RDB/AOF persistence options, maxmemory and eviction policy (allkeys-lru), systemd service management, template-driven redis.conf generation

- **base_utils**:
    - Description: Base system utilities module providing MOTD management, OS-specific utility package installation, custom facts for platform information, Bolt tasks for service checking, and Bolt plans for rolling restarts
    - Path: site/modules/common/base_utils
    - Technology: Puppet
    - Key Features: Dynamic MOTD with hostname, OS, kernel, uptime facts; OS-specific utility packages (Debian: dnsutils, RedHat: bind-utils); custom fact base_utils_info checking package installation and uptime; Bolt task check_service for service status validation; Bolt plan rolling_restart for orchestrated service restarts with health checks; custom type Base_utils::Port for port validation; custom function normalize_port for type conversion; external fact platform_info.json for static metadata

### Infrastructure Files

- **Puppetfile**: Declares external module dependencies (puppetlabs-stdlib 9.6.0, puppet-redis 10.1.0, puppetlabs-firewall 8.1.3, puppet-chrony 3.0.0, puppetlabs-rsyslog 5.0.0, saz-ssh 12.0.0, puppetlabs-vcsrepo 6.1.0, puppetlabs-postgresql 10.3.0) with version pinning for reproducibility
- **environment.conf**: Defines Puppet environment configuration with modulepath prioritizing site-specific modules over shared modules (site/modules/linux:site/modules/common:modules:$basemodulepath)
- **hiera.yaml**: Root Hiera configuration with 3-level hierarchy (environment → node → common) using YAML backend
- **data/environment/production.yaml**: Production environment Hiera data with encrypted passwords (eyaml PKCS7), production-specific settings
- **Vagrantfile**: Local testing environment using Ubuntu 24.04 with libvirt provider, 2GB RAM, 2 CPUs, port forwarding (80→8080, 443→8443)
- **vagrant-provision.sh**: Vagrant provisioning script installing Puppet 8 from apt.puppet.com, installing Forge modules, copying test manifests, applying Puppet configuration
- **test/site.pp**: Test manifest creating git repository fixture and applying all three profiles for integration testing
- **test/hiera.yaml**: Test-specific Hiera configuration with plaintext password overrides for development
- **test/data/common.yaml**: Test data with plaintext passwords, disabled firewall, reduced security for local development

### Wrapper Profiles and Roles

- **site/profile/manifests/base/base.pp**: Base profile composing chrony (NTP), rsyslog (logging), and base_utils (utilities) for all nodes
- **site/profile/manifests/app/stack.pp**: Thin wrapper delegating to profile_app_stack module
- **site/profile/manifests/loadbalancer/haproxy.pp**: Thin wrapper delegating to profile_haproxy module
- **site/profile/manifests/cache/redis.pp**: Thin wrapper delegating to profile_redis_cluster module
- **site/role/manifests/app_stack.pp**: Role composing base profile + app stack profile with dependency ordering
- **site/role/manifests/haproxy.pp**: Role composing base profile + haproxy profile with dependency ordering
- **site/role/manifests/redis_cluster.pp**: Role composing base profile + redis profile with dependency ordering

### Target Details

- **Operating System**: Ubuntu 24.04 LTS (Noble Numbat) - explicitly specified in Vagrantfile and test infrastructure. The codebase uses Debian-specific package names (dnsutils) and systemd service management, indicating Ubuntu/Debian target environment.
- **Virtual Machine Technology**: Libvirt/KVM - specified in Vagrantfile with libvirt provider configuration (2GB RAM, 2 CPUs, nested virtualization disabled)
- **Cloud Platform**: Not specified - no cloud-specific configurations detected. The infrastructure appears designed for on-premises or cloud-agnostic deployment with generic systemd and networking configurations.

## Migration Approach

### Key Dependencies to Address

**External Puppet Modules (from Puppetfile):**
- **puppetlabs-stdlib (9.6.0)**: Replace with Ansible built-in filters and modules (no direct equivalent needed, functionality built into Ansible core)
- **puppet-redis (10.1.0)**: Replace with `ansible.builtin.package` + `ansible.builtin.template` for redis.conf + `ansible.builtin.systemd` service management, or use `community.general.redis` module
- **puppetlabs-firewall (8.1.3)**: Replace with `community.general.ufw` module for UFW firewall rules (ports 80, 443, stats port)
- **puppet-chrony (3.0.0)**: Replace with `ansible.builtin.package` + `ansible.builtin.template` for chrony.conf + `ansible.builtin.systemd` service management
- **puppetlabs-rsyslog (5.0.0)**: Replace with `ansible.builtin.package` + `ansible.builtin.template` for rsyslog.conf + `ansible.builtin.systemd` service management
- **saz-ssh (12.0.0)**: Replace with `ansible.builtin.template` for sshd_config + `ansible.builtin.systemd` service management
- **puppetlabs-vcsrepo (6.1.0)**: Replace with `ansible.builtin.git` module for repository cloning and management
- **puppetlabs-postgresql (10.3.0)**: Replace with `community.postgresql.postgresql_db`, `community.postgresql.postgresql_user`, and related modules from the PostgreSQL collection

**Python Dependencies (from profile_app_stack):**
- **pip packages**: Managed via requirements.txt in git repository, install using `ansible.builtin.pip` module with virtualenv support
- **gunicorn**: WSGI server installed via pip, managed as systemd service
- **psycopg2-binary**: PostgreSQL adapter for Python, installed via pip

**System Dependencies:**
- **HAProxy**: Install via `ansible.builtin.apt` module, configure via `ansible.builtin.template` for haproxy.cfg
- **Redis**: Install via `ansible.builtin.apt` module, configure via `ansible.builtin.template` for redis.conf
- **PostgreSQL client**: Install via `ansible.builtin.apt` module for backup script functionality
- **UFW**: Install via `ansible.builtin.apt` module, manage rules via `community.general.ufw` module

### Security Considerations

**Secrets Management:**
- **profile_app_stack (4 credentials detected)**:
  - Database password in `app_db_password` (encrypted with eyaml in production, plaintext in test)
  - Redis password in `app_redis_password` (encrypted with eyaml in production, plaintext in test)
  - SECRET_KEY environment variable for Django/Flask application (encrypted with eyaml)
  - Prometheus pushgateway authentication token (if configured)
  - **Migration approach**: Convert eyaml-encrypted values to ansible-vault encrypted variables, store in group_vars/host_vars with vault encryption

- **profile_haproxy (2 credentials detected)**:
  - Stats interface password in `haproxy_stats_password` (encrypted with eyaml in production)
  - SSL/TLS certificate private key (referenced in `haproxy_ssl_cert_path`, file content not in repository)
  - **Migration approach**: Use ansible-vault for stats password, manage SSL certificates via `ansible.builtin.copy` with vault-encrypted content or external certificate management system

- **profile_redis_cluster (1 credential detected)**:
  - Redis requirepass authentication password (encrypted with eyaml in production)
  - **Migration approach**: Convert to ansible-vault encrypted variable in group_vars/redis_cluster

- **base_utils (0 credentials detected)**:
  - No sensitive credentials managed by this module

**SSL/TLS Certificate Management:**
- Puppet code references SSL certificate paths but does not manage certificate content
- Certificates appear to be deployed via external process or manual installation
- **Migration approach**: Implement certificate deployment via `ansible.builtin.copy` with vault-encrypted content, or integrate with Let's Encrypt via `community.crypto.acme_certificate` module

**Systemd Service Hardening:**
- profile_app_stack implements security hardening in systemd unit file:
  - `NoNewPrivileges=true`: Prevents privilege escalation
  - `ProtectSystem=strict`: Read-only /usr, /boot, /efi
  - `ProtectHome=true`: Makes /home, /root, /run/user inaccessible
  - `PrivateTmp=true`: Private /tmp and /var/tmp
  - `ReadWritePaths=/var/log/myapp`: Explicit write access to log directory
- **Migration approach**: Preserve all security hardening directives in Ansible-generated systemd unit files using `ansible.builtin.template` module

**Firewall Configuration:**
- UFW firewall rules managed by profile_haproxy for ports 80, 443, and stats port
- Test environment disables firewall (`profile_haproxy::manage_firewall: false`)
- **Migration approach**: Use `community.general.ufw` module with conditional execution based on environment (enabled in production, disabled in test)

**Backup Security:**
- PostgreSQL backup script stores backups in `/var/backups/postgresql/` with 30-day retention
- No encryption of backup files detected
- **Migration approach**: Consider adding GPG encryption to backup files in Ansible implementation, restrict file permissions to root:root 0600

### Technical Challenges

**Challenge 1: 21-Level Hiera Hierarchy Complexity**
- **Description**: profile_haproxy uses an extremely complex 21-level Hiera hierarchy: per-node → role → cluster → app_tier → application → team → business_unit → lifecycle → environment → network_zone → datacenter → region → country → architecture → OS release → OS family/major → OS name → OS family → kernel → container → virtual → common. This hierarchy enables fine-grained configuration overrides at multiple organizational and infrastructure levels.
- **Mitigation strategy**: 
  - Flatten hierarchy to Ansible's inventory-based variable precedence: all → group_vars (by environment, datacenter, cluster) → host_vars
  - Map Puppet facts to Ansible inventory groups: `cluster: haproxy_prod_fra` → group `haproxy_prod_fra` in inventory
  - Use Ansible's `group_by` module to dynamically create groups based on facts (OS family, architecture, virtual/container)
  - Implement variable merging using `combine` filter for deep merge of `haproxy_backends` hash
  - Document variable precedence clearly in inventory structure and README
  - Consider using Ansible inventory plugins (e.g., `constructed` plugin) to automatically create groups based on host variables

**Challenge 2: Deep Merge of Hiera Data Structures**
- **Description**: profile_haproxy uses Hiera's `lookup_options` to enable deep merge of the `haproxy_backends` hash across hierarchy levels. Production environment adds SSL configuration and stick-table settings, Frankfurt cluster adds monitoring backend, all merged with common defaults.
- **Mitigation strategy**:
  - Use Ansible's `combine(recursive=True)` filter in role defaults to merge backend configurations
  - Structure variables as: `haproxy_backends_common` (role defaults) + `haproxy_backends_environment` (group_vars/production) + `haproxy_backends_cluster` (group_vars/haproxy_prod_fra)
  - Implement merge in role tasks: `haproxy_backends: "{{ haproxy_backends_common | combine(haproxy_backends_environment | default({}), haproxy_backends_cluster | default({}), recursive=True) }}"`
  - Test merge behavior thoroughly to ensure identical results to Puppet's deep merge
  - Document merge strategy in role README with examples

**Challenge 3: Custom Puppet Functions**
- **Description**: Repository contains custom Puppet functions that require Ansible equivalents:
  - `profile_app_stack::app_db_url`: Constructs PostgreSQL URL with password URL-encoding
  - `base_utils::normalize_port`: Converts String/Integer to validated port number
  - `puppetdb_query` stub: Returns empty array when PuppetDB unavailable
- **Mitigation strategy**:
  - `app_db_url`: Replace with Jinja2 template filter using `urlencode` filter: `postgresql://{{ db_user }}:{{ db_password | urlencode }}@{{ db_host }}:{{ db_port }}/{{ db_name }}`
  - `normalize_port`: Replace with Jinja2 `int` filter with validation: `{{ port | int }}` with assertion `assert port | int >= 1 and port | int <= 65535`
  - `puppetdb_query`: Not needed in Ansible; use inventory and facts directly, or implement dynamic inventory script if PuppetDB integration required
  - Create custom Jinja2 filter plugins in `filter_plugins/` directory if complex logic required
  - Document all custom filters in role README

**Challenge 4: Custom Facts and External Facts**
- **Description**: Repository uses custom Ruby facts (`base_utils_info.rb`) and external JSON facts (`platform_info.json`) to provide additional system information
- **Mitigation strategy**:
  - Replace custom facts with Ansible's `ansible.builtin.setup` module (gathers comprehensive facts automatically)
  - For custom logic in `base_utils_info.rb` (checking package installation, uptime), use Ansible tasks with `register` to capture state
  - Replace external JSON facts with Ansible's `set_fact` module or inventory variables
  - Use `ansible.builtin.package_facts` module to gather installed package information
  - Document fact mapping in migration guide: `$facts['base_utils_info']['uptime_days']` → `ansible_uptime_seconds / 86400`

**Challenge 5: Bolt Tasks and Plans for Orchestration**
- **Description**: Repository includes Bolt tasks (`check_service.sh`) and plans (`rolling_restart.pp`) for orchestration workflows like rolling service restarts with health checks
- **Mitigation strategy**:
  - Replace Bolt tasks with Ansible ad-hoc commands or playbooks: `check_service.sh` → `ansible all -m systemd -a "name=myapp state=started"`
  - Replace Bolt plans with Ansible playbooks using `serial` directive for rolling updates: `rolling_restart.pp` → playbook with `serial: 1` and `wait_for` health checks
  - Implement health check validation using `ansible.builtin.uri` module to check HTTP endpoints
  - Use `ansible.builtin.wait_for` module to implement delay between restarts
  - Use `ansible.builtin.fail` module with `when` condition to abort on health check failure
  - Document orchestration playbooks in separate `playbooks/orchestration/` directory

**Challenge 6: Template Syntax Conversion (ERB and EPP)**
- **Description**: Repository uses both ERB (Embedded Ruby) and EPP (Embedded Puppet) template formats with conditional logic, loops, and fact references
- **Mitigation strategy**:
  - Convert ERB templates to Jinja2: `<%= @variable %>` → `{{ variable }}`, `<% if @condition %>` → `{% if condition %}`
  - Convert EPP templates to Jinja2: `<%= $variable %>` → `{{ variable }}`, `<% if $condition { %>` → `{% if condition %}`
  - Map Puppet facts to Ansible facts: `$facts['environment']` → `ansible_env.ENVIRONMENT` or inventory variable
  - Test all templates with sample data to ensure identical output
  - Pay special attention to conditional logic in haproxy.cfg.erb (SSL enabled, stats enabled) and app.env.erb (DEBUG mode, CORS settings)
  - Document template variable requirements in role README

**Challenge 7: Module Duplication and Modulepath Priority**
- **Description**: Modules exist in both `modules/` and `site/modules/linux/` directories with identical content. Modulepath prioritizes `site/modules/linux` first, which can cause confusion during migration.
- **Mitigation strategy**:
  - Consolidate modules into single Ansible roles directory structure: `roles/profile_app_stack/`, `roles/profile_haproxy/`, etc.
  - Eliminate duplication by choosing canonical location for each role
  - Use Ansible's `roles_path` in ansible.cfg to define role search paths if multiple locations needed
  - Document role organization in README and migration guide
  - Clean up duplicate modules after successful migration and testing

**Challenge 8: Roles/Profiles Pattern Translation**
- **Description**: Puppet uses roles/profiles pattern with thin wrapper profiles delegating to implementation modules, and roles composing multiple profiles with dependency ordering
- **Mitigation strategy**:
  - Translate Puppet roles to Ansible playbooks: `role::app_stack` → `playbooks/app_stack.yml`
  - Translate Puppet profiles to Ansible roles: `profile::app::stack` → `roles/app_stack/`
  - Use `dependencies` in role `meta/main.yml` to declare role dependencies: `profile::base::base` → `dependencies: [base]`
  - Use playbook `roles` section to compose multiple roles: `roles: [base, app_stack]`
  - Maintain dependency ordering using `pre_tasks`, `roles`, `tasks`, `post_tasks` sections in playbooks
  - Document roles/profiles mapping in migration guide

### Migration Order

The migration should proceed in the following order based on dependencies and complexity:

1. **base_utils** (Priority 1: Low risk, foundational, no external dependencies)
   - Simplest module with utility package installation and MOTD management
   - No external module dependencies (only stdlib)
   - Provides foundation for all other roles
   - Estimated effort: 3-5 days
   - Deliverables: `roles/base_utils/` with tasks, templates, defaults, custom facts as variables

2. **profile_redis_cluster** (Priority 2: Moderate complexity, single external dependency)
   - Single external dependency (puppet-redis module)
   - Straightforward configuration with redis.conf template
   - No complex orchestration or multi-service coordination
   - Estimated effort: 5-7 days
   - Deliverables: `roles/redis_cluster/` with Redis installation, configuration, service management

3. **profile_app_stack** (Priority 3: High complexity, multiple dependencies)
   - Multiple external dependencies (vcsrepo, postgresql client)
   - Complex systemd service configuration with security hardening
   - Custom function for database URL construction
   - Backup and health check scripts
   - Environment-specific configuration (DEBUG, CORS)
   - Estimated effort: 10-14 days
   - Deliverables: `roles/app_stack/` with git clone, virtualenv, systemd service, backup/health check scripts

4. **profile_haproxy** (Priority 4: Highest complexity, deep merge, firewall integration)
   - Most complex Hiera hierarchy (21 levels) with deep merge
   - Multiple configuration files (haproxy.cfg, backend.conf, error pages)
   - Firewall integration (UFW rules)
   - SSL/TLS certificate management
   - Environment and cluster-specific overrides
   - Estimated effort: 12-16 days
   - Deliverables: `roles/haproxy/` with HAProxy installation, configuration, firewall rules, SSL setup

5. **Wrapper profiles and roles** (Priority 5: Integration and orchestration)
   - Translate thin wrapper profiles to role dependencies
   - Create playbooks for each role (app_stack, haproxy, redis_cluster)
   - Implement base profile composition in all playbooks
   - Test end-to-end integration
   - Estimated effort: 5-7 days
   - Deliverables: `playbooks/app_stack.yml`, `playbooks/haproxy.yml`, `playbooks/redis_cluster.yml`

6. **Orchestration and testing** (Priority 6: Validation and rollout)
   - Convert Bolt tasks and plans to Ansible playbooks
   - Implement rolling restart playbook with health checks
   - Create test inventory and variable files
   - Validate against Vagrant test environment
   - Document migration and operational procedures
   - Estimated effort: 7-10 days
   - Deliverables: `playbooks/orchestration/rolling_restart.yml`, test inventory, documentation

**Total estimated effort: 42-59 days (6-8 weeks) for a team of 2-3 engineers**

### Assumptions

1. **Hiera eyaml encryption keys**: Assumes access to eyaml private key (`keys/private_key.pkcs7.pem`) to decrypt production passwords for migration to ansible-vault. If keys are unavailable, passwords must be reset or retrieved from secure storage.

2. **SSL/TLS certificates**: Puppet code references SSL certificate paths (`/etc/ssl/certs/haproxy.pem`) but does not manage certificate content. Assumes certificates are deployed via external process (e.g., Let's Encrypt, manual installation, certificate management system). Migration plan assumes certificates will be managed via Ansible `copy` module or external integration.

3. **PuppetDB integration**: Repository includes `puppetdb_query` stub function returning empty array. Assumes PuppetDB is not actively used for exported resources or cross-node queries. If PuppetDB integration is required, dynamic inventory script or Ansible Tower/AWX integration may be needed.

4. **Git repository for application code**: profile_app_stack clones application code from git repository. Assumes git repository URL, branch, and authentication credentials are available and will be migrated to Ansible variables. Test environment uses local git repository fixture.

5. **PostgreSQL database server**: profile_app_stack connects to PostgreSQL database but does not manage database server installation. Assumes PostgreSQL server is managed separately or will be added to migration scope. Backup script assumes PostgreSQL client tools are installed.

6. **Network configuration**: HAProxy backend server lists are defined in Hiera data. Assumes backend server IP addresses and ports are accurate and will be migrated to Ansible inventory or group_vars. Dynamic backend discovery via PuppetDB is not implemented.

7. **Monitoring integration**: Health check script includes optional Prometheus pushgateway integration. Assumes Prometheus infrastructure exists if `PROMETHEUS_PUSHGATEWAY` environment variable is set. Migration plan includes this feature but marks it as optional.

8. **Operating system consistency**: Vagrantfile specifies Ubuntu 24.04, but Hiera hierarchy includes OS-specific overrides (Debian, RedHat). Assumes target environment is Ubuntu/Debian-based unless RedHat-specific configurations are actively used. Migration plan prioritizes Ubuntu/Debian support.

9. **Firewall management**: profile_haproxy manages UFW firewall rules. Assumes UFW is the desired firewall solution for target environment. If iptables or firewalld is preferred, firewall module selection must be adjusted.

10. **Service restart coordination**: Bolt rolling_restart plan implements orchestrated service restarts with health checks and delays. Assumes this orchestration pattern is required in Ansible implementation. If simpler restart mechanism is acceptable, orchestration playbook can be simplified.

11. **Module duplication resolution**: Modules exist in both `modules/` and `site/modules/linux/` with identical content. Assumes duplication is unintentional and can be consolidated during migration. If duplication serves a purpose (e.g., environment-specific overrides), migration plan must be adjusted.

12. **Hiera hierarchy simplification**: 21-level Hiera hierarchy in profile_haproxy appears over-engineered for current usage (only 3 levels actively used: common, environment, cluster). Assumes hierarchy can be simplified to Ansible's inventory-based precedence without loss of functionality. If all 21 levels are actively used in production, more complex inventory structure may be required.

13. **Test environment parity**: Test environment uses plaintext passwords and disabled firewall for convenience. Assumes production environment uses encrypted passwords and enabled firewall. Migration plan includes both configurations with environment-based conditionals.

14. **Backup retention policy**: Backup script implements 30-day retention policy. Assumes this retention period is compliant with organizational backup policies. If different retention is required, backup role must be adjusted.

15. **Systemd service management**: All services (app, HAProxy, Redis, chrony, rsyslog) are managed via systemd. Assumes target environment uses systemd (Ubuntu 24.04 does). If SysV init or other init system is required, service management tasks must be adjusted.

16. **Python application framework**: profile_app_stack configures environment variables for Django/Flask (SECRET_KEY, DEBUG, DATABASE_URL). Assumes application is Django or Flask-based. If different framework is used, environment variable configuration may need adjustment.

17. **Redis cluster vs standalone**: Module is named `profile_redis_cluster` but configuration appears to be standalone Redis (no cluster mode, sentinel, or replication). Assumes standalone Redis is sufficient. If true Redis cluster is required, additional configuration and roles are needed.

18. **HAProxy backend health checks**: HAProxy configuration includes HTTP health checks with rise/fall thresholds. Assumes backend servers implement health check endpoints (e.g., `/health`, `/api/health`). If health check endpoints are not available, health check configuration must be adjusted.

19. **Log aggregation**: profile_base includes rsyslog configuration. Assumes centralized log aggregation is desired. If logs should remain local only, rsyslog configuration can be simplified or removed.

20. **Time synchronization**: profile_base includes chrony for NTP. Assumes accurate time synchronization is required (critical for distributed systems, SSL/TLS, logging). If time sync is managed externally, chrony role can be removed.

21. **SSH configuration**: Puppetfile includes saz-ssh module but no SSH configuration is visible in repository. Assumes SSH configuration is managed elsewhere or will be added to migration scope. Migration plan does not include SSH role unless configuration is discovered.

22. **Vagrant test environment**: Vagrantfile uses libvirt provider with Ubuntu 24.04. Assumes libvirt/KVM is available for local testing. If VirtualBox or other provider is preferred, Vagrantfile must be adjusted. Ansible testing can use Molecule with Docker or Vagrant.

23. **Ansible version**: Migration plan assumes Ansible 2.15+ with support for modern modules (community.general.ufw, community.postgresql.*). If older Ansible version is required, module compatibility must be verified.

24. **Ansible collections**: Migration plan uses modules from ansible.builtin, community.general, and community.postgresql collections. Assumes these collections are available via ansible-galaxy. If air-gapped environment, collections must be bundled.

25. **Variable naming conventions**: Puppet uses `profile_app_stack::app_name` naming convention. Ansible migration will use `app_stack_app_name` convention (role prefix + variable name). Assumes this naming convention is acceptable. If different convention is preferred, variable names must be adjusted throughout roles.

