# Anti-DDoS System

## Overview

Sistem anti-DDoS imbunatatit pentru FreeBSD cu protectie avansata impotriva atacurilor DDoS, SYN flood, port scanning si brute force. Inlocuieste nginx+PHP cu un server Node.js lightweight pentru performanta sporita.

## Structura Workspace

```
LICENSE
README_EN.md
README.md
Binary/
    DDosCheck.hpp          # C++ client pentru validare IP
    UserInterface.cpp      # Exemplu de integrare
    Extern/
        include/curl/      # Header-uri libcurl
        library/           # Biblioteci libcurl
firewall/
    pf.conf.optimized      # Configuratie PF optimizata
    firewall.tgz           # Arhiva backup
    install/
        crontab.example    # Exemplu crontab pentru mentenanta
        deploy.sh          # Script automat de instalare
        flash_ddos.rc      # FreeBSD rc.d service script
        maintenance.sh     # Script de mentenanta
    node-validator/
        package.json       # Node.js dependencies
        server.js          # HTTP server pentru validare IP
```

## Componente

### 1. **Node.js FlasH Ddos Server** (`firewall/node-validator/server.js`)
- Server HTTP lightweight fara dependente externe
- Rate limiting per IP
- Doua moduri de validare:
  - **Simple mode** (`/simple`): Validare intr-un singur pas (backward compatible)
  - **Secure mode** (`/validate`): Validare challenge-response in doi pasi
- Logging automat
- Health check endpoint (`/health`)
- Protectie impotriva abuzului prin rate limiting agresiv

### 2. **PF Configuration Optimizata** (`firewall/pf.conf.optimized`)

#### Imbunatatiri majore:

**Protectie TCP Flags:**
- Detecteaza si blocheaza NULL packets, XMAS packets, SYN-FIN, SYN-RST
- Protectie impotriva nmap si altor scanere de porturi
- Blocare pachete invalide (FIN fara ACK, etc.)

**Protectie SYN Flood:**
- Rate limiting agresiv pe game ports si auth ports
- Tabele separate pentru `<syn_flood>`, `<port_scanners>`, `<brute_force>`
- Adaptive timeouts pentru gestionarea inteligenta a conexiunilor

**Protectie Port Scanning:**
- Detectare automata scanare porturi
- Adaugare automata in `<permanent_banned>` la comportament suspect
- Flush global pentru curatare agresiva a conexiunilor

**Timeouts Optimizate:**
```
tcp.first: 15s (redus de la 30s)
tcp.opening: 10s
tcp.established: 600s
tcp.closing: 20s (redus de la 30s)
tcp.finwait: 10s (redus de la 15s)
tcp.closed: 5s (redus de la 10s)
```

**Limite Crescute:**
```
states: 15M (de la 10M)
src-nodes: 150K (de la 100K)
table-entries: 2M (de la 1M)
```

**Bogon Filtering Extins:**
- Blocare IP-uri rezervate/invalide (RFC 1918, RFC 6598, etc.)
- Protectie impotriva IP spoofing
- Blocare broadcast/multicast

**Securitate Aditionala:**
- Blocare porturi Windows vulnerabile (SMB, RDP, SQL Server)
- Blocare protocoale nesigure (Telnet, TFTP, etc.)
- Scrubbing avansat cu MSS limiting
- ICMP blocat complet pentru prevenirea fingerprinting

### 3. **Updated C++ Client** (`Binary/DDosCheck.hpp`)

Caracteristici:
- Suport pentru ambele moduri (simple si secure)
- Timeouts configurabile
- Verificare raspuns HTTP status code
- Thread-safe prin `CURLOPT_NOSIGNAL`
- Parsing JSON simplu pentru modul secure

### 4. **Deployment Scripts**

**`firewall/install/deploy.sh`** - Script complet de instalare:
```bash
# Instalare completa
./deploy.sh install

# Update serviciu
./deploy.sh update

# Status sistem
./deploy.sh status

# Restart serviciu
./deploy.sh restart
```

**`firewall/install/flash_ddos.rc`** - FreeBSD rc.d service script
- Pornire automata la boot
- Logging integrat
- User unprivileged pentru securitate

## Instalare

### Prerequisite
- FreeBSD 12.x sau mai nou
- Root access
- Internet connection pentru instalare Node.js

### Pasi de instalare:

```bash
# 1. Copiaza fisierele pe server FreeBSD
root/

# 2. Pe server, ruleaza scriptul de instalare
cd /root/firewall/install/
chmod +x deploy.sh
./deploy.sh install

# 3. Revizuieste configuratia PF
ee /firewall/pf.conf
# Actualizeaza ext_if, IP-uri, porturi conform setup-ului tau

# 4. Activeaza PF
sysrc pf_enable=YES
sysrc pf_rules="/firewall/pf.conf"
sysrc pflog_enable=YES
sysrc pflog_logfile="/firewall/log/pflog"

# 5. Incarca regulile PF
pfctl -f /firewall/pf.conf

# 6. Porneste serviciul IP validator
service flash_ddos start

# 7. Verifica statusul
./deploy.sh status
```

## Configurare

### Node.js Server Configuration
Editeaza `firewall/node-validator/server.js`:

```javascript
const CONFIG = {
    port: 52488,                          // Port HTTP
    trustedIpsFile: '/firewall/trusted_ips',
    maxRequestsPerMinute: 10,             // Rate limit
    tokenExpiry: 300000,                  // 5 minute
    enableLogging: true
};
```

### PF Configuration
Editeaza `firewall/pf.conf.optimized`:

```
ext_if="vtnet0"  # Schimba cu interfata ta

# Actualizeaza porturile conform setup-ului tau
game_ports="{30013, 30015, ...}"
auth_ports="{30001, 30003, ...}"

# Adauga IP-uri de management
table <WebsiteAllowed> persist { 127.0.0.1, YOUR_ADMIN_IP }
```

### C++ Client Configuration
In `Binary/DDosCheck.hpp`:

```cpp
#define VALIDATOR_HOST "YOUR_SERVER_IP"  // sau 127.0.0.1 pentru local
#define VALIDATOR_PORT "52488"
#define VALIDATOR_ENDPOINT_SIMPLE "/simple"  // Simple one-step validation
#define VALIDATOR_ENDPOINT_SECURE "/validate"  // Two-step secure validation

// Alege modul de validare
bool DDosCheckResponse()
{
    return DDosCheckResponse_Simple();  // Rapid
    // sau
    // return DDosCheckResponse_Secure();  // Mai sigur
}
```

## Integrare in Aplicatie

### In C++ Application (Binary/UserInterface.cpp)

```cpp
#include "DDosCheck.hpp"

int APIENTRY WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow)
{
    WebBrowser_Startup(hInstance);
    
    // Valideaza IP-ul inainte de a continua
    if (!DDosCheckResponse()) 
    {
      MessageBoxA(NULL, "Verificare esuata!!!", "Metin2 | Anti-DDos", MB_OK | MB_ICONEXCLAMATION);
      return false;
    }
    
    // Continua cu aplicatia...
}
```

## Utilizare

### Endpoints Disponibile

1. **Health Check**
```bash
curl http://your-server:52488/health
```

2. **Simple Validation** (un pas)
```bash
curl http://your-server:52488/simple
# Response: "OK" daca IP-ul a fost adaugat
```

3. **Secure Validation** (doi pasi)
```bash
# Pasul 1: Obtine token
curl http://your-server:52488/validate
# Response: {"status":"challenge","token":"..."}

# Pasul 2: Valideaza cu token
curl http://your-server:52488/validate?token=YOUR_TOKEN
# Response: {"status":"success","ip":"..."}
```

### Comenzi Utile PF

```bash
# Incarca/reincarca reguli
pfctl -f /firewall/pf.conf

# Verifica status
pfctl -s info

# Vezi tabele
pfctl -t trusted_hosts -T show
pfctl -t abusive_hosts -T show
pfctl -t syn_flood -T show

# Adauga manual IP in trusted
pfctl -t trusted_hosts -T add 1.2.3.4

# Sterge IP din abusive
pfctl -t abusive_hosts -T delete 1.2.3.4

# Flush tabel
pfctl -t abusive_hosts -T flush

# Vezi statistici
pfctl -s states
pfctl -s rules -v
```

## Monitorizare

### Logs
```bash
# Node.js validator logs
tail -f /var/log/flash_ddos.log

# PF logs
tcpdump -n -e -ttt -i pflog0
```

### Statistici
```bash
# Script de monitoring
./deploy.sh status

# Vezi top IPs blocate
pfctl -t abusive_hosts -T show | head -20

# Numar conexiuni pe IP
pfctl -s states | awk '{print $3}' | cut -d: -f1 | sort | uniq -c | sort -rn | head
```

## Intretinere

### Curatare Periodica
Adauga in crontab (`crontab -e`):

```bash
# Curata tabela abusive_hosts la fiecare 6 ore
0 */6 * * * /sbin/pfctl -t abusive_hosts -T expire 86400

# Backup trusted_ips zilnic
0 2 * * * cp /firewall/trusted_ips /firewall/backups/trusted_ips.$(date +\%Y\%m\%d)

# Rotatie log-uri
0 0 * * 0 /usr/sbin/newsyslog -f /var/log/flash_ddos.log
```

### Update System
```bash
cd /firewall/install
./deploy.sh update
```

## Troubleshooting

### Serviciul nu porneste
```bash
# Verifica logs
tail -50 /var/log/flash_ddos.log

# Verifica daca portul e ocupat
sockstat -l | grep 52488

# Testeaza manual
cd /firewall/node-validator
node server.js
```

### PF nu blocheaza corect
```bash
# Verifica sintaxa
pfctl -nf /firewall/pf.conf

# Vezi reguli active
pfctl -s rules -vv

# Debug specific IP
pfctl -s states | grep 1.2.3.4
```

### IP-uri nu se adauga in trusted_hosts
```bash
# Verifica permisiuni
ls -la /firewall/trusted_ips

# Verifica daca IP-ul e deja in tabel
pfctl -t trusted_hosts -T show | grep YOUR_IP

# Test manual
pfctl -t trusted_hosts -T add YOUR_IP
```

## Performanta

### Benchmark Node.js Server
```bash
# Test cu ab (Apache Bench)
ab -n 10000 -c 100 http://your-server:52488/health

# Test cu wrk
wrk -t4 -c100 -d30s http://your-server:52488/simple
```

### Optimizari Suplimentare
1. Creste `kern.ipc.maxsockbuf` in `/etc/sysctl.conf`
2. Tune `net.inet.tcp.*` parameters
3. Foloseste `htop` pentru monitoring CPU/RAM

## Securitate

### Best Practices
1. **Ruleaza Node.js ca user neprivilegiat** (www)
2. **Foloseste HTTPS** in productie (adauga nginx reverse proxy)
3. **Limiteaza accesul la /validate** doar din IP-uri cunoscute
4. **Monitorizeaza constant** log-urile pentru activitate suspecta
5. **Backup periodic** pentru `trusted_ips` si configuratii

### Hardening Additional
```bash
# In pf.conf, limiteaza rate-ul si mai mult
max-src-conn-rate 10/1  # 10 conexiuni pe secunda

# Reduce timeout-urile
set timeout { tcp.first 10, tcp.opening 5 }

# Adauga blacklist-uri externe (optional)
# table <spamhaus> persist file "/firewall/tables/spamhaus.txt"
```

## Diferente fata de versiunea veche

| Caracteristica | Veche (nginx+PHP) | Noua (Node.js) |
|----------------|-------------------|----------------|
| Footprint RAM | ~50-100 MB | ~15-30 MB |
| Timp raspuns | ~50-100 ms | ~5-10 ms |
| Concurenta | ~1000 req/s | ~5000+ req/s |
| Rate limiting | Manual in PF | Built-in + PF |
| Challenge-response | Nu | Da (optional) |
| Health check | Nu | Da |
| Logging | nginx access.log | Dedicat |
| Dependinte | nginx, PHP-FPM | Doar Node.js |

## Suport

Pentru probleme sau intrebari:
1. Verifica log-urile (`/var/log/flash_ddos.log`)
2. Ruleaza `./deploy.sh status`
3. Verifica configuratia PF cu `pfctl -s rules -vv`

## Licenta

MIT License - Free to use and modify
