# MIGRATION FROM PUPPET TO ANSIBLE

## Executive Summary

This repository contains a Puppet control repository with multiple site-modules (profiles) that manage application infrastructure components including HAProxy load balancers, PostgreSQL databases, Redis clusters, and Python application deployments. The migration scope is moderate to complex, with approximately 7 custom modules to migrate, plus dependencies on 8 external Puppet Forge modules.

Estimated timeline: 4-6 weeks for a complete migration, with the following phases:
- Week 1-2: Infrastructure setup, inventory creation, and base module migration
- Week 3-4: Core application stack and database modules migration
- Week 5-6: Testing, validation, and documentation

## Module Migration Plan

This repository contains Puppet modules that need individual migration planning:

### MODULE INVENTORY

- **base_utils**:
    - Description: Base utility module providing common helpers, defined types, functions, and tasks
    - Path: site-modules/base_utils
    - Technology: Puppet
    - Key Features: MOTD management, utility package installation, helper functions

- **profile::base::base**:
    - Description: Base OS profile included by all roles, handles common OS-level configuration
    - Path: site-modules/profile/manifests/base
    - Technology: Puppet
    - Key Features: NTP management, syslog configuration, utility inclusion

- **profile_app_stack**:
    - Description: Full application stack profile for Python applications with PostgreSQL backend and systemd service
    - Path: site-modules/profile_app_stack
    - Technology: Puppet
    - Key Features: Python app deployment, database connection, service management, monitoring

- **profile_haproxy**:
    - Description: HAProxy load balancer profile with multi-backend support
    - Path: site-modules/profile_haproxy
    - Technology: Puppet
    - Key Features: HAProxy configuration, SSL termination, stats page, backend discovery

- **profile_postgresql**:
    - Description: PostgreSQL installation with PGDG repo and version pinning
    - Path: site-modules/profile_postgresql
    - Technology: Puppet
    - Key Features: Repository management, package installation, service configuration

- **profile_redis_cluster**:
    - Description: Redis profile using puppet-redis with PuppetDB node discovery
    - Path: site-modules/profile_redis_cluster
    - Technology: Puppet
    - Key Features: Redis cluster configuration, memory management, password protection

- **role::app_server**:
    - Description: Role class that combines profiles for application servers
    - Path: site-modules/role/manifests/app_server.pp
    - Technology: Puppet
    - Key Features: Profile inclusion for base, HAProxy, app stack, and Redis

### Infrastructure Files

- `Puppetfile`: Defines external module dependencies from Puppet Forge
- `hiera.yaml`: Defines the Hiera hierarchy for parameter lookups
- `data/common.yaml`: Contains default parameter values for all profiles
- `data/environment/`: Contains environment-specific parameter values
- `environment.conf`: Defines the module path for Puppet
- `manifests/site.pp`: Contains node definitions and test repository setup
- `Vagrantfile`: Defines development environment for testing

### Target Details

Based on the source configuration files:

- **Operating System**: Multiple OS support including RedHat 8/9, Debian 11/12, and Ubuntu 22.04/24.04 (based on metadata.json files)
- **Virtual Machine Technology**: Vagrant (based on Vagrantfile presence)
- **Cloud Platform**: Not specified, appears to be infrastructure-agnostic

## Migration Approach

### Key Dependencies to Address

- **puppetlabs-stdlib (9.7.0)**: Replace with Ansible built-in filters and modules
- **puppetlabs-concat (9.0.2)**: Replace with Ansible template module and blockinfile
- **puppetlabs-firewall (8.1.3)**: Replace with Ansible firewalld or iptables modules
- **puppetlabs-vcsrepo (6.1.0)**: Replace with Ansible git module
- **puppet-redis (11.0.0)**: Replace with Ansible redis role
- **puppet-systemd (7.1.0)**: Replace with Ansible systemd module
- **puppetlabs-inifile (6.1.1)**: Replace with Ansible ini_file module
- **puppetlabs-apt (9.4.0)**: Replace with Ansible apt module

### Security Considerations

- **Credential Management**: Multiple credentials found in Hiera data:
  - HAProxy stats password in `data/common.yaml`
  - Database password in `data/common.yaml`
  - Redis password in `data/common.yaml`
  - Application secret key in `data/common.yaml`
  - Migration approach: Move all credentials to Ansible Vault

- **SSL Configuration**: HAProxy SSL configuration parameters present but disabled by default
  - Migration approach: Implement with Ansible using ssl_certificate module

- **Firewall Rules**: HAProxy module includes firewall management
  - Migration approach: Use Ansible firewalld or iptables modules with appropriate rules

### Technical Challenges

- **PuppetDB Queries**: The Redis cluster module uses PuppetDB queries for node discovery
  - Mitigation: Replace with Ansible inventory groups or dynamic inventory

- **Custom Functions**: Several modules use custom Puppet functions
  - Mitigation: Reimplement as Ansible filters or lookup plugins

- **Hiera Data Hierarchy**: Complex parameter lookup system with multiple levels
  - Mitigation: Use Ansible group_vars and host_vars with variable precedence

- **Strict Dependency Ordering**: Several modules use strict class ordering
  - Mitigation: Use Ansible handlers, meta dependencies, and proper task ordering

### Migration Order

1. **base_utils** (low risk, foundation for other modules)
2. **profile::base::base** (low complexity, required by all roles)
3. **profile_postgresql** (moderate complexity, database foundation)
4. **profile_app_stack** (high complexity, core application functionality)
5. **profile_redis_cluster** (moderate complexity, caching layer)
6. **profile_haproxy** (moderate complexity, load balancing)
7. **role::app_server** (low complexity, combines all profiles)

### Assumptions

1. The current Puppet implementation is functional and represents the desired state
2. All modules listed in the Puppetfile are required and actively used
3. The Vagrant environment is used for development and testing only
4. The application is a Python web application using PostgreSQL for persistence
5. HAProxy is used as the front-end load balancer
6. Redis is used for caching or session storage
7. No custom facts or external integrations beyond what's visible in the code
8. The migration will maintain the same functionality and configuration parameters
9. The target environment supports systemd (based on service management in the modules)
10. No complex orchestration beyond what's defined in the dependency chains