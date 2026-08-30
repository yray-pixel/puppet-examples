# Ansible Role: profile_haproxy

Configure HAProxy load balancer with SSL/TLS termination, multiple backend pools, statistics interface, and firewall integration.

## Description

This role installs and configures HAProxy load balancer with support for:
- Multiple backend server pools with configurable balance algorithms
- SSL/TLS termination with customizable cipher suites
- Statistics interface with authentication
- Health checks for backend servers
- Session persistence via stick-tables
- Firewall rule management (UFW)
- Syslog integration
- Configuration validation before service restart

## Requirements

- Ansible 2.9 or higher
- Target systems: Debian/Ubuntu or RHEL/CentOS
- AAP credential types for sensitive data (stats credentials, SSL certificates)

## Role Variables

### Package and Service Configuration

- `profile_haproxy_package_name`: HAProxy package name (default: `haproxy`)
- `profile_haproxy_service_name`: Systemd service name (default: `haproxy`)
- `profile_haproxy_config_dir`: Configuration directory (default: `/etc/haproxy`)
- `profile_haproxy_config_file`: Main config file path (default: `/etc/haproxy/haproxy.cfg`)
- `profile_haproxy_user`: System user (default: `haproxy`)
- `profile_haproxy_group`: System group (default: `haproxy`)

### Global HAProxy Settings

- `profile_haproxy_global_maxconn`: Maximum concurrent connections (default: `4096`)
- `profile_haproxy_client_timeout`: Client timeout (default: `30s`)
- `profile_haproxy_server_timeout`: Server timeout (default: `30s`)
- `profile_haproxy_connect_timeout`: Connection timeout (default: `5s`)
- `profile_haproxy_retries`: Connection retry attempts (default: `3`)

### Statistics Interface

- `profile_haproxy_stats_enabled`: Enable stats interface (default: `true`)
- `profile_haproxy_stats_port`: Stats port (default: `9000`)
- `profile_haproxy_stats_uri`: Stats URI path (default: `/haproxy-stats`)
- `stats_user`: Stats username (injected by AAP credential)
- `stats_password`: Stats password (injected by AAP credential)

### SSL/TLS Configuration

- `profile_haproxy_ssl_enabled`: Enable SSL termination (default: `false`)
- `profile_haproxy_ssl_cert_path`: Certificate directory (default: `/etc/ssl/certs`)
- `profile_haproxy_ssl_key_path`: Private key directory (default: `/etc/ssl/private`)
- `profile_haproxy_ssl_ciphers`: Cipher suite (default: `ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256`)
- `profile_haproxy_ssl_min_version`: Minimum TLS version (default: `TLSv1.2`)
- `ssl_certificate`: Certificate content (injected by AAP credential)
- `ssl_private_key`: Private key content (injected by AAP credential)

### Logging

- `profile_haproxy_log_server`: Syslog server address (default: `127.0.0.1`)
- `profile_haproxy_log_facility`: Syslog facility (default: `local0`)
- `profile_haproxy_log_level`: Log level (default: `info`)

### Session Persistence

- `profile_haproxy_stick_table_enabled`: Enable stick-tables (default: `false`)
- `profile_haproxy_stick_table_size`: Stick-table size (default: `100k`)
- `profile_haproxy_stick_table_expire`: Entry expiration (default: `30m`)

### Firewall

- `profile_haproxy_firewall_provider`: Firewall provider (default: `ufw`)

### Backend Server Pools

- `profile_haproxy_backends`: Hash of backend configurations (deep-merged across hierarchy)

Example backend configuration:
```yaml
profile_haproxy_backends:
  webservers:
    balance: roundrobin
    port: 8080
    health_check: httpchk GET /health
    health_interval: 5s
    servers:
      - name: web1
        address: 10.0.1.10
        weight: 100
      - name: web2
        address: 10.0.1.11
        weight: 100
  api:
    balance: leastconn
    port: 3000
    health_check: httpchk GET /api/health
    health_interval: 10s
    servers:
      - name: api1
        address: 10.0.2.10
        weight: 100
```

## Dependencies

None.

## Example Playbook

```yaml
---
- name: Configure HAProxy load balancer
  hosts: loadbalancers
  become: true
  roles:
    - role: profile_haproxy
      vars:
        profile_haproxy_ssl_enabled: true
        profile_haproxy_global_maxconn: 32768
        profile_haproxy_stats_port: 9001
        profile_haproxy_backends:
          webservers:
            balance: roundrobin
            port: 8080
            servers:
              - name: web1
                address: 192.168.1.10
                weight: 100
              - name: web2
                address: 192.168.1.11
                weight: 100
```

## Multi-Level Variable Hierarchy

This role supports Puppet-style hierarchical variable overrides:

1. **defaults/main.yml**: Base defaults for all nodes
2. **vars/Debian.yml** or **vars/RedHat.yml**: OS-specific overrides
3. **vars/environment_<env>.yml**: Environment-specific (production, staging, dev)
4. **vars/datacenter_<dc>.yml**: Datacenter-specific overrides
5. **vars/cluster_<cluster>.yml**: Cluster-specific overrides
6. **vars/node_<hostname>.yml**: Node-specific overrides

Variables are loaded in order with later files overriding earlier ones. The `profile_haproxy_backends` hash is deep-merged across all levels.

Example hierarchy for production Frankfurt cluster:
```
defaults/main.yml (base)
  ↓
vars/Debian.yml (OS-specific)
  ↓
vars/environment_production.yml (environment)
  ↓
vars/datacenter_dc1_fra.yml (datacenter)
  ↓
vars/cluster_haproxy_prod_fra.yml (cluster)
  ↓
vars/node_lb01.fra.example.com.yml (node)
```

## AAP Credential Integration

This role requires AAP credential types for sensitive data:

### HAProxy Stats Credential Type
- `stats_user`: Statistics interface username
- `stats_password`: Statistics interface password

### HAProxy SSL Credential Type
- `ssl_certificate`: SSL certificate content (PEM format)
- `ssl_private_key`: SSL private key content (PEM format)

These variables are injected at runtime by AAP and should not be defined in inventory or variable files.

## Service Discovery

The original Puppet module used PuppetDB exported resources for dynamic backend discovery. In Ansible, backend servers are configured via:

1. **Static configuration**: Define backends in `profile_haproxy_backends` variable
2. **Inventory groups**: Use `groups['app_servers']` to dynamically build backend lists
3. **External service discovery**: Integrate with Consul, etcd, or cloud-native service discovery

## Files and Templates

### Configuration Files
- `/etc/haproxy/haproxy.cfg`: Main HAProxy configuration with inline backend definitions
- `/etc/haproxy/errors/503.http`: Service unavailable error page
- `/etc/haproxy/errors/408.http`: Request timeout error page
- `/etc/logrotate.d/haproxy`: Log rotation configuration

### SSL Certificates
- `/etc/ssl/certs/haproxy.pem`: Combined SSL certificate and private key (when SSL enabled)

## Handlers

- `reload haproxy`: Gracefully reload HAProxy configuration
- `restart haproxy`: Restart HAProxy service
- `reload systemd`: Reload systemd daemon configuration

## License

Apache-2.0

## Author Information

Migrated from Puppet module `profile_haproxy` to Ansible role.
