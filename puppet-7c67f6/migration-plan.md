# MIGRATION FROM PUPPET TO ANSIBLE

## Executive Summary

This repository contains a Puppet-based infrastructure configuration for a multi-tier application stack consisting of HAProxy load balancers, Python application servers with PostgreSQL databases, and Redis cache clusters. The migration to Ansible will involve converting 3 primary Puppet modules, their associated roles and profiles, and the Hiera-based configuration hierarchy.

Estimated timeline: 3-4 weeks with 1-2 dedicated engineers, depending on testing requirements and environment complexity.

## Module Migration Plan

This repository contains Puppet modules that need individual migration planning:

### MODULE INVENTORY

- **profile_app_stack**:
    - Description: Python application stack with PostgreSQL database, including app deployment from Git, database configuration, service management, and monitoring
    - Path: modules/profile_app_stack
    - Technology: Puppet
    - Key Features: Python virtual environment setup, PostgreSQL database integration, systemd service configuration, application deployment from Git repository

- **profile_haproxy**:
    - Description: HAProxy load balancer configuration with multi-backend support, SSL termination, and statistics monitoring
    - Path: modules/profile_haproxy
    - Technology: Puppet
    - Key Features: Backend server configuration, SSL certificate management, statistics page, firewall rules

- **profile_redis_cluster**:
    - Description: Redis cluster configuration with PuppetDB node discovery
    - Path: modules/profile_redis_cluster
    - Technology: Puppet
    - Key Features: Redis server configuration, memory management, cluster node discovery

- **puppetdb_query_stub**:
    - Description: Stub function for PuppetDB queries in development environments
    - Path: modules/puppetdb_query_stub
    - Technology: Puppet
    - Key Features: Function stub for puppetdb_query that returns empty arrays

- **profile::base::base**:
    - Description: Base OS configuration included by all roles
    - Path: site/profile/manifests/base/base.pp
    - Technology: Puppet
    - Key Features: NTP (chrony), syslog (rsyslog), and utility package management

- **role::app_stack**:
    - Description: Role for application stack nodes combining base profile with app stack profile
    - Path: site/role/manifests/app_stack.pp
    - Technology: Puppet
    - Key Features: Role composition with dependency ordering

- **role::haproxy**:
    - Description: Role for HAProxy load balancer nodes
    - Path: site/role/manifests/haproxy.pp
    - Technology: Puppet
    - Key Features: Role composition with dependency ordering

- **role::redis_cluster**:
    - Description: Role for Redis cluster nodes
    - Path: site/role/manifests/redis_cluster.pp
    - Technology: Puppet
    - Key Features: Role composition with dependency ordering

### Infrastructure Files

- `Puppetfile`: Defines external module dependencies (puppetlabs-stdlib, puppetlabs-concat, puppetlabs-firewall, puppetlabs-vcsrepo, puppet-redis, puppetlabs-apt)
- `hiera.yaml`: Defines the Hiera hierarchy for configuration data with EYAML encryption for sensitive data
- `environment.conf`: Defines the module path for Puppet environments
- `Vagrantfile`: Development environment configuration using Ubuntu 24.04
- `vagrant-provision.sh`: Provisioning script for Vagrant that installs Puppet and required modules
- `test/site.pp`: Test manifest that creates a stub Git repository and includes all profiles for testing

### Target Details

Based on the source configuration files:

- **Operating System**: Ubuntu 24.04 (identified from Vagrantfile and module metadata.json files)
- **Virtual Machine Technology**: Vagrant with libvirt provider (identified from Vagrantfile)
- **Cloud Platform**: Not specified in the repository

## Migration Approach

### Key Dependencies to Address

- **puppetlabs-stdlib (9.7.0)**: Replace with Ansible built-in filters and modules
- **puppetlabs-concat (9.0.2)**: Replace with Ansible template module and blockinfile/lineinfile modules
- **puppetlabs-firewall (8.1.3)**: Replace with Ansible firewalld or iptables modules
- **puppetlabs-vcsrepo (6.1.0)**: Replace with Ansible git module
- **puppet-redis (11.0.0)**: Replace with Ansible redis role from Ansible Galaxy
- **puppetlabs-apt (9.4.0)**: Replace with Ansible apt module

### Security Considerations

- **Hiera EYAML Encryption**: The repository uses EYAML for encrypting sensitive data in Hiera. Migration will require:
  - Extracting encrypted values from Hiera
  - Converting to Ansible Vault for secrets management
  - Updating all references to use Ansible Vault variables

- **Database Credentials**: The profile_app_stack module contains database credentials that need to be securely managed:
  - `db_user` and `db_password` parameters in profile_app_stack
  - These should be migrated to Ansible Vault

- **Redis Password**: The profile_redis_cluster module contains a Redis password that needs to be securely managed:
  - `redis_password` parameter in profile_redis_cluster
  - This should be migrated to Ansible Vault

- **HAProxy Stats Authentication**: The profile_haproxy module contains credentials for the HAProxy stats page:
  - `stats_user` and `stats_password` parameters in profile_haproxy
  - These should be migrated to Ansible Vault

- **SSL Certificates**: The profile_haproxy module manages SSL certificates:
  - `ssl_cert_path` and `ssl_key_path` parameters in profile_haproxy
  - Certificate handling should be migrated to Ansible's certificate management approach

### Technical Challenges

- **PuppetDB Query Replacement**: The profile_redis_cluster module uses PuppetDB queries to discover cluster nodes:
  - This functionality needs to be replaced with Ansible inventory or dynamic inventory
  - Consider using Ansible facts gathering or a custom inventory plugin

- **Hiera Hierarchy**: The repository uses a complex Hiera hierarchy for configuration:
  - Need to map Hiera hierarchy to Ansible group_vars and host_vars
  - Consider using Ansible's variable precedence rules to replicate Hiera behavior

- **Module Dependencies**: The Puppet modules have strict dependency chains:
  - Need to map these to Ansible role dependencies or playbook ordering
  - Consider using Ansible's pre_tasks, tasks, and post_tasks to maintain ordering

- **Custom Functions**: The repository includes custom Puppet functions:
  - Need to reimplement these as Ansible filters or lookup plugins
  - Example: profile_app_stack::app_db_url function

### Migration Order

1. **profile::base::base** (Priority 1, low risk)
   - Basic OS configuration is a good starting point
   - Simple dependencies and functionality

2. **profile_app_stack** (Priority 2, high value)
   - Core application functionality
   - Complex but critical for testing the migration

3. **profile_haproxy** (Priority 3, moderate complexity)
   - Load balancer configuration
   - Depends on application servers being configured

4. **profile_redis_cluster** (Priority 4, moderate complexity)
   - Cache layer
   - Requires replacing PuppetDB query functionality

5. **Role Structure** (Priority 5, low risk)
   - Convert roles to Ansible playbooks or role dependencies
   - Implement after individual profiles are migrated

### Assumptions

1. The target environment will continue to use Ubuntu 24.04 as the operating system.
2. The application stack will remain a Python application with PostgreSQL database.
3. HAProxy will continue to be used as the load balancer.
4. Redis will continue to be used as the caching layer.
5. The development environment will be migrated from Vagrant to an equivalent Ansible-based approach.
6. The current node classification based on roles will be replaced with Ansible inventory groups.
7. Hiera data will be migrated to Ansible group_vars and host_vars.
8. EYAML encrypted values will be migrated to Ansible Vault.
9. The strict dependency ordering in Puppet will be maintained in Ansible using role dependencies or playbook ordering.
10. The test environment will be recreated using Ansible Molecule or equivalent testing framework.