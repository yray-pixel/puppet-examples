# MIGRATION FROM PUPPET TO ANSIBLE

## Executive Summary

This repository contains a Puppet-based infrastructure configuration with a roles and profiles pattern. The codebase manages application stacks, HAProxy load balancers, and Redis clusters. The migration to Ansible will require converting Puppet modules, classes, and Hiera data to Ansible roles, playbooks, and variable structures.

**Estimated Timeline:**
- Analysis and Planning: 2 weeks
- Core Module Migration: 4-6 weeks
- Testing and Validation: 2-3 weeks
- Documentation and Knowledge Transfer: 1 week
- Total: 9-12 weeks

**Complexity Assessment:** Medium to High
- Well-structured Puppet code with clear separation of concerns
- Hierarchical data management with Hiera
- Multiple interdependent modules with strict dependency chains
- Secrets management with Hiera eyaml

## Module Migration Plan

This repository contains Puppet modules that need individual migration planning:

### MODULE INVENTORY

- **profile_app_stack**:
    - Description: Full application stack profile: Python app with PostgreSQL and systemd service
    - Path: modules/profile_app_stack
    - Technology: Puppet
    - Key Features: Python application deployment, PostgreSQL database configuration, systemd service management, monitoring

- **profile_haproxy**:
    - Description: HAProxy load balancer profile with multi-backend support
    - Path: modules/profile_haproxy
    - Technology: Puppet
    - Key Features: HAProxy configuration, SSL termination, stats interface, backend management, firewall rules

- **profile_redis_cluster**:
    - Description: Redis profile using puppet-redis with PuppetDB node discovery
    - Path: modules/profile_redis_cluster
    - Technology: Puppet
    - Key Features: Redis cluster configuration, memory management, PuppetDB integration

- **puppetdb_query_stub**:
    - Description: Stub function for PuppetDB queries when no PuppetDB is available
    - Path: modules/puppetdb_query_stub
    - Technology: Puppet
    - Key Features: Function stub for testing environments

- **profile::base::base**:
    - Description: Base OS profile included by all roles
    - Path: site/profile/manifests/base/base.pp
    - Technology: Puppet
    - Key Features: NTP configuration, syslog management, utility packages

- **role::app_stack**:
    - Description: Application stack node role (Python app + PostgreSQL)
    - Path: site/role/manifests/app_stack.pp
    - Technology: Puppet
    - Key Features: Composition of base and app stack profiles

- **role::haproxy**:
    - Description: HAProxy load balancer node role
    - Path: site/role/manifests/haproxy.pp
    - Technology: Puppet
    - Key Features: Composition of base and HAProxy profiles

- **role::redis_cluster**:
    - Description: Redis cluster node role
    - Path: site/role/manifests/redis_cluster.pp
    - Technology: Puppet
    - Key Features: Composition of base and Redis profiles

### Infrastructure Files

- `Puppetfile`: Defines external module dependencies and versions
- `environment.conf`: Configures the Puppet environment modulepath
- `hiera.yaml`: Defines the Hiera hierarchy for data lookup
- `data/common.yaml`: Common default values for all environments
- `data/environment/production.yaml`: Production-specific overrides
- `data/environment/staging.yaml`: Staging-specific overrides
- `Vagrantfile`: Development environment configuration
- `vagrant-provision.sh`: Provisioning script for Vagrant

### Target Details

Based on the source configuration files:

- **Operating System**: Ubuntu 24.04 (based on metadata.json files)
- **Virtual Machine Technology**: VirtualBox (inferred from Vagrantfile presence)
- **Cloud Platform**: Not specified (no cloud-specific configurations found)

## Migration Approach

### Key Dependencies to Address

- **puppetlabs-stdlib (9.7.0)**: Replace with Ansible built-in filters and modules
- **puppetlabs-concat (9.0.2)**: Replace with Ansible template module and blockinfile
- **puppetlabs-firewall (8.1.3)**: Replace with Ansible firewalld or iptables modules
- **puppetlabs-vcsrepo (6.1.0)**: Replace with Ansible git module
- **puppet-redis (11.0.0)**: Replace with Ansible Redis role (community.general.redis or custom role)
- **puppetlabs-apt (9.4.0)**: Replace with Ansible apt module

### Security Considerations

- **Hiera eyaml**: Migration from Hiera eyaml to Ansible Vault
  - Encrypted node-specific data in `data/nodes/%{trusted.certname}.yaml`
  - Need to extract secrets and convert to Ansible Vault format

- **Sensitive Data**:
  - Database credentials in profile_app_stack (db_user, db_password)
  - Redis password in profile_redis_cluster
  - HAProxy stats credentials in profile_haproxy
  - Application secret key in profile_app_stack

- **SSL/TLS Configuration**:
  - HAProxy SSL certificate and key paths need to be managed securely
  - SSL cipher configuration needs to be preserved

### Technical Challenges

- **PuppetDB Queries**: The profile_redis_cluster module uses PuppetDB for node discovery
  - Mitigation: Replace with Ansible inventory or dynamic inventory plugins

- **Strict Dependency Chains**: Several modules use strict ordering with contain/-> syntax
  - Mitigation: Use Ansible handlers, meta tasks, or explicit dependencies

- **Hiera Data Hierarchy**: Complex data lookup with multiple levels
  - Mitigation: Convert to Ansible group_vars and host_vars with proper precedence

- **Custom Functions**: Custom Puppet functions need to be reimplemented
  - Mitigation: Convert to Ansible filters or lookup plugins

### Migration Order

1. **Base Profile** (profile::base::base)
   - Low complexity, foundational for other roles
   - Handles basic OS configuration

2. **HAProxy Profile** (profile_haproxy)
   - Medium complexity
   - Standalone service with minimal dependencies

3. **Redis Cluster Profile** (profile_redis_cluster)
   - Medium complexity
   - Requires addressing PuppetDB query replacement

4. **Application Stack Profile** (profile_app_stack)
   - High complexity
   - Multiple components with strict dependency chain
   - Database integration and secrets management

### Assumptions

1. The target environment will continue to be Ubuntu 24.04 as specified in the metadata.json files.
2. The current roles and profiles pattern will be maintained in Ansible, with equivalent separation of concerns.
3. Secrets currently managed with Hiera eyaml will be migrated to Ansible Vault.
4. The Vagrant development environment will be preserved or replaced with an equivalent.
5. No changes to the application architecture are planned during migration.
6. The PuppetDB query functionality will need an alternative implementation in Ansible.
7. The strict dependency chains in Puppet will need to be preserved in Ansible.
8. The migration will be done module by module, with testing at each stage.