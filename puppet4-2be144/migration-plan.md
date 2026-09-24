# MIGRATION FROM PUPPET TO ANSIBLE

## Executive Summary

This repository contains a Puppet control repository with a roles and profiles pattern implementation for managing a multi-tier application stack. The migration to Ansible will involve converting 5 Puppet modules, their dependencies, and the hierarchical data structure to Ansible roles, collections, and variable precedence.

**Estimated Timeline:**
- Planning & Setup: 2 weeks
- Core Module Migration: 6-8 weeks
- Testing & Validation: 2-4 weeks
- Total: 10-14 weeks

**Complexity Assessment:** Medium-High
- Well-structured Puppet code with clear separation of concerns
- Hierarchical data management with Hiera
- Multiple environment support
- Custom facts and functions that will need Ansible equivalents

## Module Migration Plan

This repository contains Puppet modules that need individual migration planning:

### MODULE INVENTORY

- **base_utils**:
    - Description: Base utility module providing common helpers, defined types, functions, and Bolt tasks
    - Path: site-modules/base_utils
    - Technology: Puppet
    - Key Features: MOTD management, utility package installation, custom facts, Puppet functions, Bolt tasks

- **profile_app_stack**:
    - Description: Full application stack profile for Python application with PostgreSQL database and systemd service
    - Path: site-modules/profile_app_stack
    - Technology: Puppet
    - Key Features: Python app deployment, PostgreSQL integration, systemd service management, application monitoring

- **profile_haproxy**:
    - Description: HAProxy load balancer profile with multi-backend support
    - Path: site-modules/profile_haproxy
    - Technology: Puppet
    - Key Features: HAProxy configuration, SSL termination, backend discovery, firewall management

- **profile_postgresql**:
    - Description: PostgreSQL installation with PGDG repository and version pinning
    - Path: site-modules/profile_postgresql
    - Technology: Puppet
    - Key Features: PostgreSQL repository management, package installation, service configuration

- **profile_redis_cluster**:
    - Description: Redis cluster profile using puppet-redis with PuppetDB node discovery
    - Path: site-modules/profile_redis_cluster
    - Technology: Puppet
    - Key Features: Redis configuration, cluster setup, PuppetDB integration for node discovery

- **profile**:
    - Description: Thin wrapper profiles that delegate to the implementation modules
    - Path: site-modules/profile
    - Technology: Puppet
    - Key Features: Base OS configuration, application stack, HAProxy, and Redis integration

- **role**:
    - Description: Role classes that compose profiles for specific node types
    - Path: site-modules/role
    - Technology: Puppet
    - Key Features: App server, app stack, HAProxy, and Redis cluster role definitions

### Infrastructure Files

- `Puppetfile`: External module dependencies including stdlib, concat, firewall, vcsrepo, redis, systemd, inifile, and apt
- `hiera.yaml`: Hierarchical data configuration with node, OS family, environment, and common levels
- `environment.conf`: Module path configuration
- `data/`: Hierarchical data files for common and environment-specific settings
- `manifests/site.pp`: Main site manifest with node classification and test repository setup
- `Vagrantfile`: Local development environment configuration

### Target Details

Based on the source configuration files:

- **Operating System**: Multiple OS support including RedHat 8/9, Debian 11/12, and Ubuntu 22.04/24.04
- **Virtual Machine Technology**: Vagrant for development (based on Vagrantfile)
- **Cloud Platform**: Not specified in the examined files

## Migration Approach

### Key Dependencies to Address

- **puppetlabs-stdlib (9.7.0)**: Replace with Ansible built-in filters and modules
- **puppetlabs-concat (9.0.2)**: Replace with Ansible template module and blockinfile/lineinfile
- **puppetlabs-firewall (8.1.3)**: Replace with Ansible firewalld or iptables modules
- **puppetlabs-vcsrepo (6.1.0)**: Replace with Ansible git module
- **puppet-redis (11.0.0)**: Replace with Ansible Redis role or collection
- **puppet-systemd (7.1.0)**: Replace with Ansible systemd module
- **puppetlabs-inifile (6.1.1)**: Replace with Ansible ini_file module
- **puppetlabs-apt (9.4.0)**: Replace with Ansible apt modules

### Security Considerations

- **HAProxy SSL Configuration**: Migration must preserve SSL certificate paths, ciphers, and minimum TLS version
- **Redis Password**: Secure handling of Redis password in profile_redis_cluster
- **Database Credentials**: PostgreSQL database credentials in profile_app_stack
- **Application Secret Key**: Secret key for the Python application in profile_app_stack
- **HAProxy Stats Authentication**: Username and password for HAProxy statistics page
- **Vault/secrets management**:
  - Hiera-based secrets management in the Puppet code
  - Hardcoded credentials in some modules (Redis password set to 'CHANGEME')
  - SSL/TLS certificate references in HAProxy profile
  - Database credentials in app_stack profile

### Technical Challenges

- **Hierarchical Data**: Puppet's Hiera has 21 levels of hierarchy in some modules; Ansible variable precedence is different and will need careful mapping
- **PuppetDB Queries**: The Redis cluster uses PuppetDB for node discovery; will need to replace with Ansible inventory or dynamic inventory
- **Custom Facts**: Several modules use custom facts that will need to be replaced with Ansible facts or variables
- **Custom Functions**: Custom Puppet functions will need to be reimplemented as Ansible filters or lookup plugins
- **Strict Dependency Chains**: Several modules use strict dependency ordering that will need to be implemented with Ansible handlers and notify mechanisms

### Migration Order

1. **base_utils** (low risk, foundation for other modules)
2. **profile_postgresql** (moderate complexity, dependency for app_stack)
3. **profile_app_stack** (high complexity, depends on PostgreSQL)
4. **profile_haproxy** (moderate complexity, independent)
5. **profile_redis_cluster** (moderate complexity, PuppetDB dependency)
6. **profile and role modules** (low risk, thin wrappers)

### Assumptions

1. The current Puppet implementation follows the roles and profiles pattern, which can be mapped to Ansible roles and collections
2. The application is a Python web application with PostgreSQL database and Redis caching
3. HAProxy is used as a load balancer with SSL termination
4. The infrastructure supports multiple environments (production, staging)
5. PuppetDB is used for node discovery in the Redis cluster
6. The repository contains a test environment with Vagrant
7. The migration will preserve the current functionality and configuration options
8. The migration will maintain support for the same operating systems
9. The current implementation uses a hierarchical data structure that will need to be mapped to Ansible variable precedence
10. Some hardcoded credentials will need to be replaced with Ansible Vault or another secrets management solution