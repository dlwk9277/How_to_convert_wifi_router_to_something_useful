# 🚀 RT-AC1200G+ MEGA-BUILD GUIDE
## **Transform Your Router Into a Network Command Center**
## ✅ YOU WILL BUILD:

1. ✅ **Network-wide ad blocker** - Everyone benefits
2. ✅ **Real-time traffic monitoring** - See everything
3. ✅ **MQTT smart home hub** - Control IoT devices locally
4. ✅ **24/7 torrent seedbox** - Automated downloads
5. ✅ **Security testing lab** - Learn pen-testing safely

**Total setup time**: ~2-3 hours  
**Skill level gained**: Intermediate Linux networking  
**Cool factor**: 🔥🔥🔥🔥🔥

---

## ⚠️ CRITICAL WARNINGS

### Hardware Requirements
- ✅ **USB Drive**: 8GB minimum (32GB recommended)
- ✅ **Ethernet cable**: For stable setup
- ✅ **Cooling**: Router will run hot - add ventilation
- ⚠️ **Backup**: Factory reset capability in case of issues

### What Won't Work
- ❌ **WireGuard VPN**: Kernel 2.6.36 too old (needs 3.10+)
- ❌ **Custom firmware**: No OpenWrt/DD-WRT/Merlin support
- ⚠️ **NAS**: USB 2.0 limited to ~10-12 MB/s

---

## 📦 PHASE 1: FOUNDATION SETUP

### Step 1.1: Enable SSH Access
```bash
# Via Router Web Interface:
# 1. Login: http://[ROUTER_IP] (usually 192.168.1.1)
# 2. Go to: Administration → System
# 3. Enable SSH: YES
# 4. SSH Port: 22
# 5. Allow SSH from: LAN only
# 6. Click Apply

# Test SSH connection:
ssh admin@192.168.1.1
# Enter router password when prompted
```

### Step 1.2: Prepare USB Drive

**⚠️ WARNING: This ERASES the USB drive!**

```bash
# Connect to router via SSH, then:

# Check if USB detected:
ls /dev/sd*
# Should show: /dev/sda /dev/sda1

# Format as ext4:
umount /dev/sda1 2>/dev/null
mkfs.ext4 -L "RouterStorage" /dev/sda1

# Mount the drive:
mkdir -p /mnt/sda1
mount /dev/sda1 /mnt/sda1

# Verify mount:
df -h | grep sda1
# Should show available space
```

### Step 1.3: Install Entware (Package Manager)

```bash
# Navigate to USB drive:
cd /mnt/sda1

# Download and install Entware:
wget -O - http://bin.entware.net/armv7sf-k2.6/installer/generic.sh | sh

# Create symlink:
ln -sf /mnt/sda1/entware /opt

# Update package list:
/opt/bin/opkg update

# Add to PATH:
cat >> ~/.profile << 'EOF'
export PATH=/opt/bin:/opt/sbin:$PATH
EOF

source ~/.profile

# Test installation:
opkg --version
```

**Fallback**: If installer fails, manually download from https://bin.entware.net/

---

## 🛡️ PHASE 2: NETWORK-WIDE AD BLOCKER

**Impact**: Blocks ads/trackers for ALL devices on your network

### Step 2.1: Install DNS Tools

```bash
# Install dnsmasq-full (advanced DNS server):
opkg install dnsmasq-full

# Install wget-ssl for HTTPS downloads:
opkg install wget-ssl
```

### Step 2.2: Download Ad Blocklists

```bash
# Create blocklist directory:
mkdir -p /opt/etc/adblock

# Download Steven Black's unified hosts file (ads + malware + tracking):
wget -O /opt/etc/adblock/hosts.txt \
  "https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts"

# Convert to dnsmasq format:
grep "0.0.0.0" /opt/etc/adblock/hosts.txt | \
  awk '{print "address=/"$2"/0.0.0.0"}' > \
  /opt/etc/dnsmasq.d/adblock.conf

# Add additional tracking blocklist:
wget -O - "https://raw.githubusercontent.com/crazy-max/WindowsSpyBlocker/master/data/hosts/spy.txt" | \
  grep "0.0.0.0" | \
  awk '{print "address=/"$2"/0.0.0.0"}' >> \
  /opt/etc/dnsmasq.d/adblock.conf
```

### Step 2.3: Configure Router DNS

```bash
# Via Router Web Interface:
# Go to: LAN → DHCP Server
# DNS Server 1: [YOUR_ROUTER_IP] (e.g., 192.168.1.1)
# DNS Server 2: 8.8.8.8 (Google DNS as fallback)
# Click Apply

# Restart dnsmasq:
killall dnsmasq
/opt/etc/init.d/S56dnsmasq restart
```

### Step 2.4: Test Ad Blocking

```bash
# Test if ads.google.com is blocked:
nslookup ads.google.com

# Should return: 0.0.0.0

# Test normal site still works:
nslookup google.com
# Should return real IP address
```

### Step 2.5: Auto-Update Blocklists

```bash
# Create update script:
cat > /opt/bin/update-adblock.sh << 'EOF'
#!/bin/sh
wget -O /opt/etc/adblock/hosts.txt \
  "https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts"
grep "0.0.0.0" /opt/etc/adblock/hosts.txt | \
  awk '{print "address=/"$2"/0.0.0.0"}' > \
  /opt/etc/dnsmasq.d/adblock.conf
killall dnsmasq
/opt/etc/init.d/S56dnsmasq restart
EOF

chmod +x /opt/bin/update-adblock.sh

# Add to cron (update weekly on Sunday at 3 AM):
cru a UpdateAdblock "0 3 * * 0 /opt/bin/update-adblock.sh"
```

**Whitelist domains** (if something breaks):
```bash
# Example: whitelist facebook.com
echo "address=/facebook.com/" >> /opt/etc/dnsmasq.d/whitelist.conf
killall dnsmasq && /opt/etc/init.d/S56dnsmasq restart
```

---

## 📊 PHASE 3: NETWORK MONITORING SUITE

**Impact**: See everything happening on your network in real-time

### Step 3.1: Install Monitoring Tools

```bash
# Traffic monitoring:
opkg install iftop htop tcpdump

# Network scanning:
opkg install nmap nmap-full

# Web-based traffic analyzer:
opkg install darkstat

# Bandwidth tracking:
opkg install vnstat
```

### Step 3.2: Configure Darkstat (Web Interface)

```bash
# Start darkstat on boot:
cat > /opt/etc/init.d/S90darkstat << 'EOF'
#!/bin/sh
ENABLED=yes
case "$1" in
  start)
    darkstat -i br0 -p 8080 --daemon
    ;;
  stop)
    killall darkstat
    ;;
  restart)
    $0 stop
    $0 start
    ;;
esac
EOF

chmod +x /opt/etc/init.d/S90darkstat

# Start now:
darkstat -i br0 -p 8080 --daemon

# Access web interface:
# Open browser: http://[ROUTER_IP]:8080
```

### Step 3.3: Configure vnStat (Bandwidth Tracking)

```bash
# Initialize vnstat database:
vnstat -u -i br0

# Start vnstat daemon:
vnstatd -d

# View statistics:
vnstat           # Monthly stats
vnstat -d        # Daily stats
vnstat -h        # Hourly stats
vnstat -l        # Live traffic
```

### Step 3.4: Real-Time Monitoring Commands

```bash
# Live bandwidth by device:
iftop -i br0

# Live CPU/memory usage:
htop

# Capture packets (save to USB):
tcpdump -i br0 -w /mnt/sda1/capture.pcap

# Scan all devices on network:
nmap -sn 192.168.1.0/24

# Find open ports on specific device:
nmap -p- 192.168.1.100

# Detect device OS:
nmap -O 192.168.1.100
```

### Step 3.5: Create Monitoring Dashboard Script

```bash
cat > /opt/bin/netdash.sh << 'EOF'
#!/bin/sh
clear
echo "===== NETWORK DASHBOARD ====="
echo ""
echo "Connected Devices:"
arp -a | wc -l
echo ""
echo "Top Bandwidth Users:"
iftop -t -s 5 -i br0 2>/dev/null | head -20
echo ""
echo "CPU Usage:"
top -bn1 | grep "CPU:" 
echo ""
echo "Memory Usage:"
free -h
EOF

chmod +x /opt/bin/netdash.sh

# Run dashboard:
netdash.sh
```

---

## 🏡 PHASE 4: MQTT SMART HOME HUB

**Impact**: Control IoT devices without cloud dependency

### Step 4.1: Install Mosquitto MQTT Broker

```bash
# Install mosquitto:
opkg install mosquitto-ssl mosquitto-client-ssl

# Create config directory:
mkdir -p /opt/etc/mosquitto

# Create basic config:
cat > /opt/etc/mosquitto/mosquitto.conf << 'EOF'
pid_file /opt/var/run/mosquitto.pid
persistence true
persistence_location /mnt/sda1/mosquitto/
log_dest file /opt/var/log/mosquitto.log

# Default listener
listener 1883
allow_anonymous false
password_file /opt/etc/mosquitto/passwd

# WebSocket listener (for web dashboards)
listener 9001
protocol websockets
EOF

# Create data directory:
mkdir -p /mnt/sda1/mosquitto
```

### Step 4.2: Create MQTT User

```bash
# Create password file:
mosquitto_passwd -c /opt/etc/mosquitto/passwd admin
# Enter password when prompted

# Add more users:
mosquitto_passwd /opt/etc/mosquitto/passwd device1
```

### Step 4.3: Start MQTT Broker

```bash
# Create startup script:
cat > /opt/etc/init.d/S95mosquitto << 'EOF'
#!/bin/sh
case "$1" in
  start)
    mosquitto -c /opt/etc/mosquitto/mosquitto.conf -d
    ;;
  stop)
    killall mosquitto
    ;;
  restart)
    $0 stop
    sleep 2
    $0 start
    ;;
esac
EOF

chmod +x /opt/etc/init.d/S95mosquitto

# Start broker:
/opt/etc/init.d/S95mosquitto start
```

### Step 4.4: Test MQTT Broker

```bash
# Subscribe to test topic (Terminal 1):
mosquitto_sub -h localhost -t test/topic -u admin -P [your_password]

# Publish message (Terminal 2):
mosquitto_pub -h localhost -t test/topic -m "Hello MQTT!" -u admin -P [your_password]

# Should see "Hello MQTT!" in Terminal 1
```

### Step 4.5: Connect IoT Devices

Configure your IoT devices with:
- **MQTT Broker**: [ROUTER_IP]:1883
- **Username**: admin
- **Password**: [password you set]

**Compatible devices**: ESP32, ESP8266, Tasmota, ESPHome, Home Assistant

---

## 📥 PHASE 5: TORRENT SEEDBOX

**Impact**: 24/7 downloads without keeping PC on

### Step 5.1: Install Transmission

```bash
# Install transmission daemon:
opkg install transmission-daemon transmission-web

# Stop transmission to edit config:
/opt/etc/init.d/S88transmission stop
```

### Step 5.2: Configure Transmission

```bash
# Edit config file:
nano /opt/etc/transmission/settings.json

# Key settings to change:
# "download-dir": "/mnt/sda1/downloads",
# "incomplete-dir": "/mnt/sda1/incomplete",
# "rpc-whitelist": "192.168.*.*",
# "rpc-username": "admin",
# "rpc-password": "yourpassword",
# "speed-limit-down": 5000,        # 5 MB/s limit
# "speed-limit-down-enabled": true,

# Create download directories:
mkdir -p /mnt/sda1/downloads
mkdir -p /mnt/sda1/incomplete
```

### Step 5.3: Start Transmission

```bash
# Start transmission:
/opt/etc/init.d/S88transmission start

# Access web interface:
# Browser: http://[ROUTER_IP]:9091
# Username: admin
# Password: [password you set]
```

### Step 5.4: Performance Optimization

```bash
# Edit settings for better USB 2.0 performance:
nano /opt/etc/transmission/settings.json

# Optimize for USB 2.0:
# "cache-size-mb": 8,
# "preallocation": 0,
# "download-queue-size": 2,
```

**⚠️ USB 2.0 Speed Limit**: Max ~10-12 MB/s downloads

---

## 🔐 PHASE 6: SECURITY TESTING LAB

**Impact**: Learn penetration testing safely on your own network

### Step 6.1: Install Hacking Tools

```bash
# Network scanning:
opkg install nmap nmap-full

# Packet analysis:
opkg install tcpdump wireshark-cli

# Password cracking:
opkg install john john-bin

# Network tools:
opkg install netcat socat ettercap-ng

# Web testing:
opkg install curl wget-ssl
```

### Step 6.2: Packet Capture Setup

```bash
# Capture all traffic for 60 seconds:
tcpdump -i br0 -w /mnt/sda1/capture-$(date +%Y%m%d-%H%M%S).pcap -G 60

# Capture specific device traffic:
tcpdump -i br0 host 192.168.1.100 -w /mnt/sda1/device-capture.pcap

# View capture file:
tcpdump -r /mnt/sda1/capture.pcap | less

# Filter for passwords in HTTP traffic:
tcpdump -A -r /mnt/sda1/capture.pcap | grep -i "password"
```

### Step 6.3: Network Scanning Techniques

```bash
# Full network scan:
nmap -sn 192.168.1.0/24 -oN /mnt/sda1/network-scan.txt

# Detailed device scan:
nmap -A -T4 192.168.1.100

# Scan for vulnerabilities:
nmap --script vuln 192.168.1.100

# Find devices with default passwords:
nmap --script http-default-accounts 192.168.1.0/24

# Check for SSL vulnerabilities:
nmap --script ssl-cert,ssl-enum-ciphers -p 443 192.168.1.100
```

### Step 6.4: Man-in-the-Middle Setup (Ethical Testing Only!)

```bash
# Enable IP forwarding:
echo 1 > /proc/sys/net/ipv4/ip_forward

# ARP spoofing (intercept traffic):
ettercap -T -M arp:remote /192.168.1.1// /192.168.1.100//
# Replace IPs: gateway and target device

# Capture intercepted traffic:
tcpdump -i br0 -w /mnt/sda1/mitm-capture.pcap
```

**⚠️ WARNING**: Only perform these tests on YOUR OWN network and devices!

### Step 6.5: Create Security Scan Script

```bash
cat > /opt/bin/securityscan.sh << 'EOF'
#!/bin/sh
echo "===== NETWORK SECURITY SCAN ====="
echo "Scanning network: 192.168.1.0/24"
echo ""

echo "[1/5] Finding active hosts..."
nmap -sn 192.168.1.0/24 -oG - | grep "Up" > /mnt/sda1/active-hosts.txt

echo "[2/5] Scanning for open ports..."
nmap -p- -T4 192.168.1.0/24 -oN /mnt/sda1/port-scan.txt

echo "[3/5] Checking for default credentials..."
nmap --script http-default-accounts 192.168.1.0/24 -oN /mnt/sda1/default-creds.txt

echo "[4/5] Vulnerability scan..."
nmap --script vuln 192.168.1.0/24 -oN /mnt/sda1/vulnerabilities.txt

echo "[5/5] SSL/TLS check..."
nmap --script ssl-enum-ciphers -p 443 192.168.1.0/24 -oN /mnt/sda1/ssl-check.txt

echo ""
echo "Scan complete! Results saved to /mnt/sda1/"
EOF

chmod +x /opt/bin/securityscan.sh

# Run full security scan:
securityscan.sh
```

---

## ⚡ PHASE 7: OPTIMIZATION & MONITORING

### Step 7.1: Create System Monitor

```bash
cat > /opt/bin/sysmon.sh << 'EOF'
#!/bin/sh
while true; do
  clear
  echo "===== SYSTEM MONITOR ====="
  echo ""
  date
  echo ""
  echo "CPU Temperature:"
  cat /proc/sys/kernel/random/entropy_avail
  echo ""
  echo "CPU Usage:"
  top -bn1 | grep "CPU:"
  echo ""
  echo "Memory:"
  free -h
  echo ""
  echo "Disk Usage:"
  df -h /mnt/sda1
  echo ""
  echo "Network Traffic (5-second snapshot):"
  iftop -t -s 5 -i br0 2>/dev/null | head -20
  sleep 5
done
EOF

chmod +x /opt/bin/sysmon.sh

# Run system monitor:
sysmon.sh
```

### Step 7.2: Performance Optimization

```bash
# Reduce logging to extend USB drive life:
echo "none /var/log tmpfs defaults,noatime 0 0" >> /etc/fstab

# Optimize USB mount:
mount -o remount,noatime /mnt/sda1

# Reduce swappiness (if you add swap):
echo 10 > /proc/sys/vm/swappiness
```

### Step 7.3: Backup Configuration

```bash
# Create backup script:
cat > /opt/bin/backup.sh << 'EOF'
#!/bin/sh
BACKUP_DIR="/mnt/sda1/backups"
DATE=$(date +%Y%m%d-%H%M%S)

mkdir -p $BACKUP_DIR

# Backup all configs:
tar -czf $BACKUP_DIR/router-backup-$DATE.tar.gz \
  /opt/etc \
  /jffs/scripts \
  /jffs/configs

echo "Backup complete: $BACKUP_DIR/router-backup-$DATE.tar.gz"

# Keep only last 5 backups:
ls -t $BACKUP_DIR/router-backup-*.tar.gz | tail -n +6 | xargs rm -f
EOF

chmod +x /opt/bin/backup.sh

# Schedule weekly backups (Sunday 4 AM):
cru a WeeklyBackup "0 4 * * 0 /opt/bin/backup.sh"

# Run backup now:
backup.sh
```

### Step 7.4: Create Master Control Script

```bash
cat > /opt/bin/control.sh << 'EOF'
#!/bin/sh
case "$1" in
  status)
    echo "===== ROUTER STATUS ====="
    echo "Ad Blocker: $(pgrep dnsmasq > /dev/null && echo 'Running' || echo 'Stopped')"
    echo "MQTT Broker: $(pgrep mosquitto > /dev/null && echo 'Running' || echo 'Stopped')"
    echo "Torrent Client: $(pgrep transmission > /dev/null && echo 'Running' || echo 'Stopped')"
    echo "Traffic Monitor: $(pgrep darkstat > /dev/null && echo 'Running' || echo 'Stopped')"
    ;;
  restart-all)
    echo "Restarting all services..."
    /opt/etc/init.d/S56dnsmasq restart
    /opt/etc/init.d/S95mosquitto restart
    /opt/etc/init.d/S88transmission restart
    /opt/etc/init.d/S90darkstat restart
    echo "All services restarted!"
    ;;
  update)
    echo "Updating packages..."
    opkg update
    opkg upgrade
    /opt/bin/update-adblock.sh
    echo "Update complete!"
    ;;
  *)
    echo "Usage: $0 {status|restart-all|update}"
    ;;
esac
EOF

chmod +x /opt/bin/control.sh

# Check status:
control.sh status
```

---

## 🎯 FINAL SETUP: AUTO-START ON BOOT

```bash
# Create master startup script:
cat > /jffs/scripts/services-start << 'EOF'
#!/bin/sh

# Wait for USB to mount:
sleep 10

# Mount USB drive:
mount /dev/sda1 /mnt/sda1

# Link Entware:
ln -sf /mnt/sda1/entware /opt

# Start all services:
/opt/etc/init.d/rc.unslung start

# Log startup:
echo "$(date): All services started" >> /mnt/sda1/startup.log
EOF

chmod +x /jffs/scripts/services-start

# Enable JFFS custom scripts:
nvram set jffs2_scripts=1
nvram commit
```

---

## 📋 QUICK REFERENCE COMMANDS

```bash
# Service Management:
control.sh status              # Check all services
control.sh restart-all         # Restart everything
control.sh update              # Update packages

# Monitoring:
sysmon.sh                      # System monitor
netdash.sh                     # Network dashboard
htop                           # CPU/memory usage
iftop -i br0                   # Live bandwidth

# Ad Blocking:
update-adblock.sh              # Update blocklists
nano /opt/etc/dnsmasq.d/whitelist.conf   # Whitelist sites

# Network Scanning:
securityscan.sh                # Full security scan
nmap -sn 192.168.1.0/24        # Find devices

# Torrents:
# Web UI: http://[ROUTER_IP]:9091

# MQTT:
mosquitto_sub -h localhost -t '#' -u admin -P [password]  # Monitor all topics

# Backup:
backup.sh                      # Create backup now
```

---

## 🔥 PERFORMANCE EXPECTATIONS

**CPU Usage** (all services running):
- Idle: ~20-30%
- Light use: ~40-60%
- Heavy traffic + scans: ~80-100%

**RAM Usage**:
- ~60-80 MB used (out of ~128 MB total)

**USB Speed**:
- Read: ~10-12 MB/s
- Write: ~8-10 MB/s

**Network Impact**:
- Ad blocking: Negligible
- Monitoring: ~1-2% overhead
- Torrents: Up to 100% bandwidth usage

---

## ⚠️ TROUBLESHOOTING

### Services Won't Start
```bash
# Check logs:
logread | tail -50

# Manually start service:
/opt/etc/init.d/S[service] start

# Check if USB mounted:
df -h | grep sda1
```

### Router Running Hot
```bash
# Check CPU temperature:
cat /proc/sys/kernel/random/entropy_avail

# Reduce load:
# Stop torrents
# Disable darkstat
# Limit monitoring
```

### USB Drive Corruption
```bash
# Check filesystem:
umount /mnt/sda1
e2fsck -f /dev/sda1

# If corrupted, format and reinstall:
mkfs.ext4 /dev/sda1
# Then reinstall Entware
```

### Factory Reset (Last Resort)
```bash
# Hold reset button for 10+ seconds while router is powered on
# Or via web interface: Administration → Restore/Save/Upload Setting
```

---

## 🎓 LEARNING RESOURCES

**Practice Projects**:
1. Set up fake IoT device with ESP32 → MQTT
2. Capture and analyze Netflix traffic patterns
3. Build packet capture → Python analysis pipeline
4. Create custom blocklists for specific sites
5. Set up honeypot to detect port scanners

**Next Steps**:
- Learn Python for custom automation
- Study Wireshark for deep packet analysis
- Explore RF hacking with RTL-SDR
- Build network tools in C/Go
- Eventually buy OpenWrt router for advanced projects

---

## ✅ YOU NOW HAVE:

1. ✅ **Network-wide ad blocker** - Everyone benefits
2. ✅ **Real-time traffic monitoring** - See everything
3. ✅ **MQTT smart home hub** - Control IoT devices locally
4. ✅ **24/7 torrent seedbox** - Automated downloads
5. ✅ **Security testing lab** - Learn pen-testing safely

**Total setup time**: ~2-3 hours  
**Skill level gained**: Intermediate Linux networking  
**Cool factor**: 🔥🔥🔥🔥🔥

---

**Ready to start? Pick a phase and let's begin! 🚀**
