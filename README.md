# Anti-DDoS System

## Overview

Improved anti-DDoS system for FreeBSD with advanced protection against DDoS attacks, SYN flood, port scanning and brute force. Replaces nginx+PHP with a lightweight Node.js server for enhanced performance.

## Workspace Structure

```
LICENSE
README_EN.md
README.md
Binary/
    DDosCheck.hpp          # C++ client for IP validation
    UserInterface.cpp      # Integration example
    Extern/
        include/curl/      # libcurl headers
        library/           # libcurl libraries
firewall/
    pf.conf.optimized      # Optimized PF configuration
    firewall.tgz           # Backup archive
    install/
        crontab.example    # Crontab example for maintenance
        deploy.sh          # Automated installation script
        flash_ddos.rc      # FreeBSD rc.d service script
        maintenance.sh     # Maintenance script
    node-validator/
        package.json       # Node.js dependencies
        server.js          # HTTP server for IP validation
```

## Components

### 1. **Node.js FlasH Ddos Server** (`firewall/node-validator/server.js`)
- Lightweight HTTP server without external dependencies
- Per-IP rate limiting
- Two validation modes:
  - **Simple mode** (`/simple`): Single-step validation (backward compatible)
  - **Secure mode** (`/validate`): Two-step challenge-response validation
- Automatic logging
- Health check endpoint (`/health`)
- Protection against abuse through aggressive rate limiting

### 2. **Optimized PF Configuration** (`firewall/pf.conf.optimized`)

#### Major improvements:

**TCP Flags Protection:**
- Detects and blocks NULL packets, XMAS packets, SYN-FIN, SYN-RST
- Protection against nmap and other port scanners
- Blocking invalid packets (FIN without ACK, etc.)

**SYN Flood Protection:**
- Aggressive rate limiting on game ports and auth ports
- Separate tables for `<syn_flood>`, `<port_scanners>`, `<brute_force>`
- Adaptive timeouts for intelligent connection management

**Port Scanning Protection:**
- Automatic port scan detection
- Automatic addition to `<permanent_banned>` on suspicious behavior
- Global flush for aggressive connection cleanup

**Optimized Timeouts:**
```
tcp.first: 15s (reduced from 30s)
tcp.opening: 10s
tcp.established: 600s
tcp.closing: 20s (reduced from 30s)
tcp.finwait: 10s (reduced from 15s)
tcp.closed: 5s (reduced from 10s)
```

**Increased Limits:**
```
states: 15M (from 10M)
src-nodes: 150K (from 100K)
table-entries: 2M (from 1M)
```

**Extended Bogon Filtering:**
- Blocking reserved/invalid IPs (RFC 1918, RFC 6598, etc.)
- Protection against IP spoofing
- Blocking broadcast/multicast

**Additional Security:**
- Blocking vulnerable Windows ports (SMB, RDP, SQL Server)
- Blocking insecure protocols (Telnet, TFTP, etc.)
- Advanced scrubbing with MSS limiting
- ICMP completely blocked to prevent fingerprinting

### 3. **Updated C++ Client** (`Binary/DDosCheck.hpp`)

Features:
- Support for both modes (simple and secure)
- Configurable timeouts
- HTTP status code response verification
- Thread-safe through `CURLOPT_NOSIGNAL`
- Simple JSON parsing for secure mode

### 4. **Deployment Scripts**

**`firewall/install/deploy.sh`** - Complete installation script:
```bash
# Complete installation
./deploy.sh install

# Service update
./deploy.sh update

# System status
./deploy.sh status

# Restart service
./deploy.sh restart
```

**`firewall/install/flash_ddos.rc`** - FreeBSD rc.d service script
- Automatic startup at boot
- Integrated logging
- Unprivileged user for security

## Installation

### Prerequisites
- FreeBSD 12.x or newer
- Root access
- Internet connection for Node.js installation

### Installation steps:

```bash
# 1. Copy files to FreeBSD server
root/

# 2. On server, run installation script
cd /root/firewall/install/
chmod +x deploy.sh
./deploy.sh install

# 3. Review PF configuration
ee /firewall/pf.conf
# Update ext_if, IPs, ports according to your setup

# 4. Enable PF
sysrc pf_enable=YES
sysrc pf_rules="/firewall/pf.conf"
sysrc pflog_enable=YES
sysrc pflog_logfile="/firewall/log/pflog"

# 5. Load PF rules
pfctl -f /firewall/pf.conf

# 6. Start IP validator service
service flash_ddos start

# 7. Check status
./deploy.sh status
```

## Configuration

### Node.js Server Configuration
Edit `firewall/node-validator/server.js`:

```javascript
const CONFIG = {
    port: 52488,                          // HTTP Port
    trustedIpsFile: '/firewall/trusted_ips',
    maxRequestsPerMinute: 10,             // Rate limit
    tokenExpiry: 300000,                  // 5 minutes
    enableLogging: true
};
```

### PF Configuration
Edit `firewall/pf.conf.optimized`:

```
ext_if="vtnet0"  # Change to your interface

# Update ports according to your setup
game_ports="{30013, 30015, ...}"
auth_ports="{30001, 30003, ...}"

# Add management IPs
table <WebsiteAllowed> persist { 127.0.0.1, YOUR_ADMIN_IP }
```

### C++ Client Configuration
In `Binary/DDosCheck.hpp`:

```cpp
#define VALIDATOR_HOST "YOUR_SERVER_IP"  // or 127.0.0.1 for local
#define VALIDATOR_PORT "52488"
#define VALIDATOR_ENDPOINT_SIMPLE "/simple"  // Simple one-step validation
#define VALIDATOR_ENDPOINT_SECURE "/validate"  // Two-step secure validation

// Choose validation mode
bool DDosCheckResponse()
{
    return DDosCheckResponse_Simple();  // Fast
    // or
    // return DDosCheckResponse_Secure();  // More secure
}
```

## Application Integration

### In C++ Application (Binary/UserInterface.cpp)

```cpp
#include "DDosCheck.hpp"

int APIENTRY WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow)
{
    WebBrowser_Startup(hInstance);
    
    // Validate IP before continuing
    if (!DDosCheckResponse()) 
    {
      MessageBoxA(NULL, "Verification failed!!!", "Metin2 | Anti-DDos", MB_OK | MB_ICONEXCLAMATION);
      return false;
    }
    
    // Continue with application...
}
```

## Usage

### Available Endpoints

1. **Health Check**
```bash
curl http://your-server:52488/health
```

2. **Simple Validation** (one step)
```bash
curl http://your-server:52488/simple
# Response: "OK" if IP was added
```

3. **Secure Validation** (two steps)
```bash
# Step 1: Get token
curl http://your-server:52488/validate
# Response: {"status":"challenge","token":"..."}

# Step 2: Validate with token
curl http://your-server:52488/validate?token=YOUR_TOKEN
# Response: {"status":"success","ip":"..."}
```

### Useful PF Commands

```bash
# Load/reload rules
pfctl -f /firewall/pf.conf

# Check status
pfctl -s info

# View tables
pfctl -t trusted_hosts -T show
pfctl -t abusive_hosts -T show
pfctl -t syn_flood -T show

# Manually add IP to trusted
pfctl -t trusted_hosts -T add 1.2.3.4

# Delete IP from abusive
pfctl -t abusive_hosts -T delete 1.2.3.4

# Flush table
pfctl -t abusive_hosts -T flush

# View statistics
pfctl -s states
pfctl -s rules -v
```

## Monitoring

### Logs
```bash
# Node.js validator logs
tail -f /var/log/flash_ddos.log

# PF logs
tcpdump -n -e -ttt -i pflog0
```

### Statistics
```bash
# Monitoring script
./deploy.sh status

# View top blocked IPs
pfctl -t abusive_hosts -T show | head -20

# Count connections per IP
pfctl -s states | awk '{print $3}' | cut -d: -f1 | sort | uniq -c | sort -rn | head
```

## Maintenance

### Periodic Cleanup
Add to crontab (`crontab -e`):

```bash
# Clean abusive_hosts table every 6 hours
0 */6 * * * /sbin/pfctl -t abusive_hosts -T expire 86400

# Daily backup of trusted_ips
0 2 * * * cp /firewall/trusted_ips /firewall/backups/trusted_ips.$(date +\%Y\%m\%d)

# Log rotation
0 0 * * 0 /usr/sbin/newsyslog -f /var/log/flash_ddos.log
```

### Update System
```bash
cd /firewall/install
./deploy.sh update
```

## Troubleshooting

### Service won't start
```bash
# Check logs
tail -50 /var/log/flash_ddos.log

# Check if port is occupied
sockstat -l | grep 52488

# Test manually
cd /firewall/node-validator
node server.js
```

### PF not blocking correctly
```bash
# Check syntax
pfctl -nf /firewall/pf.conf

# View active rules
pfctl -s rules -vv

# Debug specific IP
pfctl -s states | grep 1.2.3.4
```

### IPs not being added to trusted_hosts
```bash
# Check permissions
ls -la /firewall/trusted_ips

# Check if IP is already in table
pfctl -t trusted_hosts -T show | grep YOUR_IP

# Manual test
pfctl -t trusted_hosts -T add YOUR_IP
```

## Performance

### Benchmark Node.js Server
```bash
# Test with ab (Apache Bench)
ab -n 10000 -c 100 http://your-server:52488/health

# Test with wrk
wrk -t4 -c100 -d30s http://your-server:52488/simple
```

### Additional Optimizations
1. Increase `kern.ipc.maxsockbuf` in `/etc/sysctl.conf`
2. Tune `net.inet.tcp.*` parameters
3. Use `htop` for CPU/RAM monitoring

## Security

### Best Practices
1. **Run Node.js as unprivileged user** (www)
2. **Use HTTPS** in production (add nginx reverse proxy)
3. **Limit access to /validate** only from known IPs
4. **Constantly monitor** logs for suspicious activity
5. **Periodic backup** for `trusted_ips` and configurations

### Additional Hardening
```bash
# In pf.conf, limit rate even more
max-src-conn-rate 10/1  # 10 connections per second

# Reduce timeouts
set timeout { tcp.first 10, tcp.opening 5 }

# Add external blacklists (optional)
# table <spamhaus> persist file "/firewall/tables/spamhaus.txt"
```

## Differences from old version

| Feature | Old (nginx+PHP) | New (Node.js) |
|---------|-----------------|---------------|
| RAM Footprint | ~50-100 MB | ~15-30 MB |
| Response Time | ~50-100 ms | ~5-10 ms |
| Concurrency | ~1000 req/s | ~5000+ req/s |
| Rate limiting | Manual in PF | Built-in + PF |
| Challenge-response | No | Yes (optional) |
| Health check | No | Yes |
| Logging | nginx access.log | Dedicated |
| Dependencies | nginx, PHP-FPM | Only Node.js |

## Support

For issues or questions:
1. Check logs (`/var/log/flash_ddos.log`)
2. Run `./deploy.sh status`
3. Check PF configuration with `pfctl -s rules -vv`

## License

MIT License - Free to use and modify
