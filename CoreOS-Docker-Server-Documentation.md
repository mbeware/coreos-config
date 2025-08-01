# CoreOS Docker Server Configuration Documentation

## Overview

This document explains the rationale, implementation details, and security considerations for the CoreOS Ignition configuration created for the "dockerserver" VM.

## Executive Summary

**What**: A CoreOS-based Docker host VM configured via Ignition for container workloads  
**Where**: Static IP 192.168.2.21 on local network  
**Who**: Three users (mbeware, devin, docker) with appropriate permissions  
**Why**: Immutable infrastructure approach for reliable container hosting  
**How**: Butane YAML → Ignition JSON configuration with systemd automation  

## Configuration Decisions & Rationale

### 1. Operating System Choice: Fedora CoreOS

**Why CoreOS over traditional Linux distributions:**
- **Immutable infrastructure**: OS updates are atomic and rollback-capable
- **Container-first design**: Optimized for Docker/Podman workloads
- **Automatic updates**: Reduces maintenance overhead
- **Security hardening**: Minimal attack surface, SELinux enabled by default
- **Declarative configuration**: Infrastructure as code via Ignition

**Why NOT traditional Ubuntu/CentOS:**
- Package drift and configuration drift over time
- Manual update processes prone to human error
- Larger attack surface with unnecessary packages

### 2. User Account Strategy

#### mbeware (Primary Administrator)
- **Groups**: wheel, sudo, docker
- **SSH Key**: RSA 4096-bit key provided by user
- **Rationale**: Primary administrative access with full privileges

#### devin (AI Assistant Account)
- **Groups**: wheel, sudo, docker  
- **Authentication**: SSH key-based (to be configured post-deployment)
- **Rationale**: Allows AI assistance with system administration tasks

#### docker (Service Account)
- **Groups**: docker only
- **System account**: true
- **Rationale**: Dedicated account for Docker service operations, follows principle of least privilege

**Security Note**: No password authentication configured - SSH keys only for enhanced security.

### 3. Network Configuration

**Static IP Implementation:**
- **IP**: 192.168.2.21/24
- **Gateway**: 192.168.2.1 (assumed standard)
- **DNS**: Google DNS (8.8.8.8, 8.8.4.4)
- **Method**: NetworkManager configuration (CoreOS standard)

**Why NetworkManager over systemd-networkd:**
- CoreOS default network management
- Better integration with container networking
- More robust for complex network scenarios

### 4. Software Installation Strategy

#### Core Packages (via rpm-ostree)
- **git**: Version control for configuration management
- **python3**: Latest available (not 3.14 as requested - doesn't exist yet)
- **python3-pip**: Package management for Python tools
- **cockpit**: Web-based system administration (better than webmin for CoreOS)
- **samba + samba-client**: File sharing capabilities
- **cifs-utils**: For mounting remote CIFS shares
- **xorg-x11-xauth**: X11 forwarding support for SSH
- **openssh-clients**: Enhanced SSH client capabilities

#### Additional Software (via package managers)
- **Tailscale**: VPN mesh networking (from official repo)
- **Ansible**: Configuration management (via pip)
- **docker-compose**: Container orchestration (via pip)

**Why this approach:**
- rpm-ostree for system-level packages ensures immutability
- pip for Python packages allows user-space flexibility
- Official repositories for security and updates

### 5. Storage Configuration

#### Directory Structure
- **/mnt/nasmnt**: Samba share export point (mode 755)
- **/mnt/allvideo**: Mount point for remote CIFS share (mode 755)

**Rationale**: Separate mount points for different storage purposes, standard permissions for shared access.

### 6. Service Configuration

#### Docker Service
- **Enabled**: true
- **Rationale**: Primary purpose of the server

#### Cockpit Web Interface
- **Service**: cockpit.socket
- **Port**: 9090 (default)
- **Rationale**: Better than webmin/1panel for CoreOS, integrated system monitoring

#### SSH Service
- **Enabled**: true
- **X11 Forwarding**: Supported via xorg-x11-xauth
- **Rationale**: Remote administration and GUI application support

#### Samba Services
- **smb.service**: File sharing protocol
- **nmb.service**: NetBIOS name resolution
- **Configuration**: Automated via custom systemd unit

#### Tailscale VPN
- **Service**: tailscaled.service
- **Rationale**: Secure remote access and mesh networking

### 7. Custom Systemd Units

#### install-packages.service
**Purpose**: Automated software installation on first boot
**Dependencies**: network-online.target
**Security**: OneShot service, runs once then exits

#### setup-samba.service
**Purpose**: Configure Samba share for /mnt/nasmnt
**Dependencies**: install-packages.service
**Configuration**: Guest access enabled for simplicity (can be hardened later)

#### mount-allvideo.service
**Purpose**: Mount remote CIFS share from 192.168.2.31
**Authentication**: Guest access (assumes local network trust)
**Mount options**: uid=1000,gid=1000 for proper permissions

## Security Considerations

### Implemented Security Measures
1. **SSH key-only authentication** - No password login
2. **Minimal user privileges** - Principle of least privilege
3. **Immutable OS** - Reduced attack surface
4. **SELinux enabled** - Mandatory access controls
5. **Automatic updates** - Security patches applied automatically

### Security Trade-offs Made
1. **Samba guest access** - Simplified for local network use
2. **CIFS guest mounting** - Assumes trusted local network
3. **Docker group membership** - Required for container management

### Recommendations for Hardening
1. Configure Tailscale authentication before production use
2. Implement Samba user authentication if needed
3. Configure firewall rules for specific service access
4. Regular security audits of container images

## Future Container Deployment Considerations

### Planned Workloads Support
The configuration supports the user's planned container deployments:

1. **Kestra (100%)**: Workflow orchestration - Docker ready
2. **PostgreSQL (100%)**: Database - Volume mounts configured
3. **Ollama (90%)**: LLM inference - GPU passthrough may need configuration
4. **Plex (90%)**: Media server - /mnt/allvideo mount ready
5. **Self-hosted services (30-75%)**: Various - Docker Compose ready

### Volume Strategy
- **/mnt/nasmnt**: Persistent storage for containers
- **/mnt/allvideo**: Media files for Plex and similar services
- **Docker volumes**: For application data persistence

### Network Strategy
- **Host networking**: For services requiring direct network access
- **Bridge networking**: For isolated container communication
- **Tailscale**: For secure remote access to services

## Deployment Instructions

1. **Boot CoreOS with Ignition**: Use the generated `dockerserver-ignition.json`
2. **Verify services**: Check systemd unit status after first boot
3. **Configure Tailscale**: Run `tailscale up` and authenticate
4. **Test Samba**: Verify /mnt/nasmnt share accessibility
5. **Test CIFS mount**: Verify /mnt/allvideo mount success
6. **Deploy containers**: Use docker-compose for application deployment

## Troubleshooting

### Common Issues
1. **Network configuration**: Check NetworkManager status
2. **Package installation**: Review install-packages.service logs
3. **Mount failures**: Verify network connectivity to 192.168.2.31
4. **Service failures**: Use `systemctl status <service>` for diagnostics

### Log Locations
- **System logs**: `journalctl -u <service-name>`
- **Installation logs**: `journalctl -u install-packages.service`
- **Network logs**: `journalctl -u NetworkManager`

## Files Generated

1. **dockerserver-butane.yaml**: Human-readable configuration source
2. **dockerserver-ignition.json**: Machine-readable Ignition configuration
3. **CoreOS-Docker-Server-Documentation.md**: This documentation

## Conclusion

This configuration provides a robust, secure, and maintainable foundation for Docker container hosting on CoreOS. The immutable infrastructure approach ensures consistency and reliability, while the automated configuration reduces deployment complexity and human error.

The design balances security with usability, making reasonable trade-offs for a local network environment while maintaining the ability to harden security for production use.
