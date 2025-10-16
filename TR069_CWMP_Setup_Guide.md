# TR-069 CWMP Agent Setup and Firmware Building Guide

## Table of Contents
1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [EasyCWMP Package Integration](#easycwmp-package-integration)
4. [Configuration Options](#configuration-options)
5. [Building Firmware with TR-069](#building-firmware-with-tr-069)
6. [Runtime Configuration](#runtime-configuration)
7. [Troubleshooting](#troubleshooting)
8. [Advanced Configuration](#advanced-configuration)

## Overview

This guide covers the integration and configuration of TR-069 CWMP (CPE WAN Management Protocol) agent in OpenWrt firmware. TR-069 is a technical specification for remote management of customer-premises equipment (CPE) connected to an IP network.

### What is TR-069?
- **TR-069**: Technical Report 069, also known as CWMP (CPE WAN Management Protocol)
- **Purpose**: Remote management and auto-configuration of network devices
- **Components**: Auto Configuration Server (ACS) and CPE (Customer Premises Equipment)
- **Data Models**: TR-098 (Internet Gateway Device) and TR-181 (Device:2)

## Prerequisites

### System Requirements
- OpenWrt build environment setup
- Minimum 4GB RAM for compilation
- 50GB+ free disk space
- Ubuntu 20.04+ or similar Linux distribution

### Required Dependencies
```bash
sudo apt-get update
sudo apt-get install build-essential git unzip ncurses-dev zlib1g-dev \
    subversion gawk gcc-multilib flex git-core gettext libssl-dev \
    libncurses5-dev unzip python3-distutils python3-dev rsync
```

## EasyCWMP Package Integration

### 1. EasyCWMP Package Structure
The EasyCWMP package is already integrated in your OpenWrt build system:
```
package/easycwmp/
├── Makefile                    # Package build configuration
├── src/                        # Source code
├── ext/openwrt/               # OpenWrt-specific files
│   ├── config/easycwmp        # UCI configuration
│   ├── init.d/easycwmpd       # Init script
│   └── scripts/               # Management scripts
└── easycwmp-1.8.6.tar.gz     # Source tarball
```

### 2. Package Dependencies
EasyCWMP requires the following packages:
- `libubus` - OpenWrt micro bus architecture
- `libuci` - Unified Configuration Interface
- `libubox` - OpenWrt utility library
- `libmicroxml` - Micro XML parser
- `libjson-c` - JSON library
- `curl` & `libcurl` - HTTP client library

### 3. Build Configuration Options
The package supports several compile-time options:

| Option | Description | Default |
|--------|-------------|---------|
| `CONFIG_EASYCWMP_DEBUG` | Enable debug output | Disabled |
| `CONFIG_EASYCWMP_DEVEL` | Development mode | Disabled |
| `CONFIG_EASYCWMP_BACKUP_DATA_CONFIG` | Backup data in config | Disabled |
| `CONFIG_EASYCWMP_SCRIPTS_FULL` | Full script support | Enabled |
| `CONFIG_EASYCWMP_DATA_MODEL_TR181` | Use TR-181 data model | Enabled |
| `CONFIG_EASYCWMP_DATA_MODEL_TR98` | Use TR-098 data model | Disabled |

## Configuration Options

### 1. Using make menuconfig
```bash
cd /home/abhiram-jalumuri/openwrt
make menuconfig
```

Navigate to:
```
Utilities → 
    [*] easycwmp (CWMP client using libcurl)
        [*] Full script support
        [*] TR-181 data model
        [ ] TR-098 data model (legacy)
        [ ] Debug mode
        [ ] Development mode
        [ ] Backup data in config files
```

### 2. Direct .config Configuration
Add these lines to your `.config` file:
```bash
# EasyCWMP Configuration
CONFIG_PACKAGE_easycwmp=y
CONFIG_EASYCWMP_SCRIPTS_FULL=y
CONFIG_EASYCWMP_DATA_MODEL_TR181=y
# CONFIG_EASYCWMP_DATA_MODEL_TR98 is not set
CONFIG_EASYCWMP_BACKUP_DATA_FILE=y
# CONFIG_EASYCWMP_BACKUP_DATA_CONFIG is not set
# CONFIG_EASYCWMP_DEBUG is not set
# CONFIG_EASYCWMP_DEVEL is not set
```

### 3. Pre-configured Build Configs
Your workspace contains several pre-configured builds:

- **mahipal.config**: Contains working EasyCWMP configuration
- **prasanth.config**: Contains working EasyCWMP configuration  
- **easycwmp.config**: Dedicated EasyCWMP build configuration

To use a pre-configured build:
```bash
cp mahipal.config .config
make oldconfig
```

## Building Firmware with TR-069

### 1. Complete Build Process
```bash
# Navigate to OpenWrt directory
cd /home/abhiram-jalumuri/openwrt

# Update feeds
./scripts/feeds update -a
./scripts/feeds install -a

# Configure build (if not using pre-configured .config)
make menuconfig
# OR use existing config
cp mahipal.config .config
make oldconfig

# Start build process
make -j$(nproc) V=s
```

### 2. Package-Only Build
To build only the EasyCWMP package:
```bash
make package/easycwmp/compile V=s
```

### 3. Build Output
After successful compilation, find the outputs in:
```
bin/targets/<architecture>/<subtarget>/
├── openwrt-<target>-generic-squashfs-sysupgrade.bin    # Firmware image
└── packages/
    └── easycwmp_1.8.6-1_<arch>.ipk                     # EasyCWMP package
```

### 4. Verify Package Inclusion
Check if EasyCWMP is included in the build:
```bash
# List packages in firmware
grep -r "easycwmp" bin/targets/*/generic/packages/
```

## Runtime Configuration

### 1. UCI Configuration
EasyCWMP uses UCI for configuration. Main config file: `/etc/config/easycwmp`

Basic configuration structure:
```bash
# View current configuration
uci show easycwmp

# Key configuration sections:
config device 'device'
    option manufacturer 'OpenWrt'
    option oui '00:25:9C'
    option product_class 'Router'
    option serial_number 'Unknown'

config local 'local'
    option interface 'br-lan'
    option port '7547'
    option ubus_socket '/var/run/ubus.sock'

config acs 'acs'
    option url 'http://acs.example.com:7547/'
    option username 'username'
    option password 'password'
    option periodic_enable '1'
    option periodic_interval '3600'
```

### 2. Service Management
```bash
# Start EasyCWMP daemon
/etc/init.d/easycwmpd start

# Enable at boot
/etc/init.d/easycwmpd enable

# Check status
/etc/init.d/easycwmpd status

# View logs
logread | grep easycwmp
```

### 3. Manual Configuration Example
```bash
# Configure ACS URL
uci set easycwmp.acs.url='http://your-acs-server.com:7547/'
uci set easycwmp.acs.username='your-username'
uci set easycwmp.acs.password='your-password'

# Enable periodic inform
uci set easycwmp.acs.periodic_enable='1'
uci set easycwmp.acs.periodic_interval='3600'

# Set device information
uci set easycwmp.device.manufacturer='YourCompany'
uci set easycwmp.device.product_class='YourRouter'
uci set easycwmp.device.serial_number='$(cat /proc/cpuinfo | grep Serial | cut -d: -f2 | tr -d " ")'

# Commit changes
uci commit easycwmp

# Restart service
/etc/init.d/easycwmpd restart
```

## Troubleshooting

### 1. Common Build Issues

**Issue**: EasyCWMP not found in menuconfig
```bash
# Solution: Update feeds and install
./scripts/feeds update packages
./scripts/feeds install easycwmp
```

**Issue**: Build fails with missing dependencies
```bash
# Solution: Install required system packages
sudo apt-get install libssl-dev libcurl4-openssl-dev
```

**Issue**: Package compilation fails
```bash
# Solution: Clean and rebuild
make package/easycwmp/clean
make package/easycwmp/compile V=s
```

### 2. Runtime Issues

**Issue**: EasyCWMP daemon not starting
```bash
# Check configuration syntax
easycwmpd -t

# Check UCI configuration
uci show easycwmp

# Check system logs
logread | grep -i error
```

**Issue**: Cannot connect to ACS
```bash
# Test ACS connectivity
curl -v http://your-acs-server.com:7547/

# Check firewall rules
iptables -L | grep 7547

# Verify network interface
ifconfig br-lan
```

**Issue**: TR-069 session failures
```bash
# Enable debug mode temporarily
easycwmpd -f -d

# Check detailed logs
tail -f /var/log/messages | grep easycwmp
```

### 3. Debug Mode
To enable comprehensive debugging:

1. **Compile-time debug**:
   ```bash
   # In menuconfig, enable:
   # Utilities → easycwmp → Debug mode
   CONFIG_EASYCWMP_DEBUG=y
   ```

2. **Runtime debug**:
   ```bash
   # Start daemon in foreground with debug
   easycwmpd -f -d
   ```

## Advanced Configuration

### 1. Custom Data Model Extensions
EasyCWMP supports custom data model extensions:

Create custom scripts in `/usr/share/easycwmp/functions/`:
```bash
# Example custom parameter script
cat > /usr/share/easycwmp/functions/custom_param << 'EOF'
#!/bin/sh
# Custom parameter implementation
case "$1" in
    get)
        echo "CustomValue"
        ;;
    set)
        # Handle parameter setting
        ;;
esac
EOF
chmod +x /usr/share/easycwmp/functions/custom_param
```

### 2. SSL/TLS Configuration
For secure ACS connections:
```bash
# Configure SSL certificates
uci set easycwmp.acs.ssl_verify='1'
uci set easycwmp.acs.ssl_cert='/etc/ssl/certs/client.crt'
uci set easycwmp.acs.ssl_key='/etc/ssl/private/client.key'
uci set easycwmp.acs.ssl_ca='/etc/ssl/certs/ca.crt'
uci commit easycwmp
```

### 3. Performance Tuning
```bash
# Adjust memory usage
uci set easycwmp.local.max_envelopes='10'
uci set easycwmp.local.max_data_length='65536'

# Configure timeouts
uci set easycwmp.acs.timeout='30'
uci set easycwmp.acs.retry_count='3'

uci commit easycwmp
```

### 4. Integration with Other Services
EasyCWMP can integrate with:
- **DHCP**: Auto-discovery via DHCP option 43
- **DNS**: ACS URL via DNS TXT records  
- **Firewall**: Automatic port management
- **Network**: Interface monitoring and reporting

## Example Build Commands Summary

### Quick Start Build
```bash
cd /home/abhiram-jalumuri/openwrt

# Use pre-configured setup
cp mahipal.config .config
make oldconfig

# Build firmware with TR-069
make -j$(nproc) V=s

# Find output
ls -la bin/targets/*/generic/*.bin
ls -la bin/targets/*/generic/packages/*easycwmp*
```

### Custom Configuration Build
```bash
cd /home/abhiram-jalumuri/openwrt

# Configure from scratch
make menuconfig
# Navigate to: Utilities → easycwmp
# Enable required options

# Build
make -j$(nproc) V=s
```

This completes the comprehensive TR-069 CWMP agent setup and firmware building guide for OpenWrt.