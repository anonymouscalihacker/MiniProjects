# Ultimate GL.iNet VPN Security Guide - Zero Location Leaks

> **Goal:** Configure your GL.iNet router to maintain your home location appearance from anywhere in the world, with zero IP/DNS leaks and military-grade OpSec.

---

## Table of Contents

1. [Pre-Setup: Know Your Home IP](#pre-setup-know-your-home-ip)
2. [Prerequisites: WireGuard Server Setup](#prerequisites-wireguard-server-setup)
3. [Pre-Travel Security Checklist](#pre-travel-security-checklist)
4. [Phase 1: Web UI Configuration](#phase-1-web-ui-configuration)
   - [WireGuard Client Setup](#wireguard-client-setup)
   - [DNS Configuration](#dns-configuration)
   - [Initial Connection Test](#initial-connection-test)
5. [Phase 2: Terminal Hardening](#phase-2-terminal-hardening)
   - [DNS Security Options](#dns-security-options)
   - [IPv6 Complete Disable](#ipv6-complete-disable)
   - [Firewall Rules](#firewall-rules)
   - [Block Traceroute Responses](#block-traceroute-responses-privacy-enhancement)
   - [Kill Switch Implementation](#kill-switch-implementation)
   - [NTP Leak Prevention](#ntp-leak-prevention)
   - [VPN Watchdog](#vpn-watchdog)
6. [Phase 3: Device Security](#phase-3-device-security)
   - [Work Laptop Configuration](#work-laptop-configuration)
   - [MDM & Hardware Considerations](#mdm--hardware-considerations)
   - [Router SSID Configuration](#router-ssid-configuration)
7. [Phase 4: Comprehensive Testing](#phase-4-comprehensive-testing)
   - [Command Line Tests](#command-line-tests)
   - [Browser Leak Tests](#browser-leak-tests)
   - [Kill Switch Verification](#kill-switch-verification)
8. [Advanced Security Enhancements](#advanced-security-enhancements)
9. [Operational Security (OPSEC) Guidelines](#operational-security-opsec-guidelines)
10. [Backup and Recovery](#backup-and-recovery)
11. [Troubleshooting Guide](#troubleshooting-guide)
12. [Advanced Detection Methods & Mitigations](#advanced-detection-methods--mitigations)

---

## Pre-Setup: Know Your Home IP

**CRITICAL**: Before traveling, identify your home IP address:

```bash
# FROM YOUR HOME NETWORK (before traveling):
curl -s ifconfig.me
```

**Write this IP down** - this is what should appear when your VPN is working correctly.

Alternative: Check your WireGuard config file for the `Endpoint` line - that's your home IP.

---

## Prerequisites: WireGuard Server Setup

You'll need:
1. **Travel Client Router** (recommend higher-end model for better CPU/speed)
2. **One or more WireGuard servers** in different locations

**Note**: WireGuard performance is CPU-dependent. More powerful routers = faster VPN speeds.

### Critical Configuration Tips for Multiple Servers

**DNS STANDARDIZATION**:
```ini
# ✅ GOOD - Standardized DNS across all servers
[Interface]
Address = xxx.x.x.xx/xx    # Unique IP per server
DNS = 1.1.1.1              # Same DNS for all configs

# ❌ BAD - Local DNS that causes routing conflicts
[Interface]
Address = xxx.x.x.xx/xx
DNS = 192.168.1.1          # Will fail due to routing conflicts!
```

**Best Practice Checklist**:
- ✅ Use Cloudflare DNS (1.1.1.1) in ALL WireGuard configs
- ✅ Set router DNS to Manual with 1.1.1.1
- ✅ Enable "Allow Custom DNS to Override VPN DNS"
- ✅ Avoid 192.168.x.x addresses for WireGuard IPs
- ✅ Test each server connection independently

Server Setup Resources:
- [GL.iNet WireGuard Server Setup Video 1](https://www.youtube.com/watch?v=qLEj9zoiYRs)
- [GL.iNet WireGuard Server Setup Video 2](https://www.youtube.com/watch?v=LXbDg1v65Qs)

---

## Pre-Travel Security Checklist

### Router Preparation
- [ ] Update router firmware to latest version
- [ ] Test entire setup for at least 1 week at home
- [ ] Create configuration backup
- [ ] Document all settings
- [ ] Test failover scenarios

### Work Device Preparation
- [ ] Disable Wi-Fi adapter in Device Manager/System Preferences
- [ ] Set timezone to home timezone (DO NOT change while traveling)
- [ ] Disable location services completely
- [ ] Disable automatic time zone updates
- [ ] Clear all saved Wi-Fi networks
- [ ] Disable Bluetooth
- [ ] Install WebRTC blocking extension
- [ ] Disable DNS over HTTPS in browser settings
- [ ] Check for WWAN/LTE modem and disable in BIOS if present

### Mobile Device Preparation
- [ ] Remove work apps or use separate work phone
- [ ] Disable cellular data for work apps
- [ ] Configure VPN on phone if needed
- [ ] Disable location services for all work apps

---

## Phase 1: Web UI Configuration

### WireGuard Client Setup

1. **Access Admin Panel**
   - Connect to router via Ethernet (Wi-Fi MUST be disabled on laptop!)
   - Navigate to `192.168.8.1`
   - Login with admin credentials

2. **Upload WireGuard Configuration**
   - Go to **VPN** → **WireGuard Client**
   - Click **Set Up WireGuard Manually**
   - Upload your home WireGuard `.conf` file
   - **DO NOT CONNECT YET**

3. **Configure WireGuard Options** (Click gear icon)
   - **Remote Access LAN**: OFF (disabled)
   - **IP Masquerading**: ON (enabled) ✓
   - **MTU**: Leave empty (auto-detection)
   - Click **Apply**

4. **Enable Global VPN Settings**
   - Go to **VPN Dashboard**
   - Click **Global Options** (top right)
   - **Block Non-VPN Traffic**: ON (enabled) ✓
   - **Services from GL.iNet Use VPN**: OFF (disabled)
   - Click **Apply**

5. **Set VPN Mode**
   - Select **Global Proxy** (NOT Policy Mode)
   - This forces ALL traffic through VPN

### DNS Configuration

**For Multiple WireGuard Servers (Recommended)**:

1. Navigate to **Network** → **DNS**
2. Configure:
   - **Mode**: Manual DNS
   - **DNS Server 1**: `1.1.1.1`
   - **DNS Server 2**: `1.0.0.1`
   - **Allow Custom DNS to Override VPN DNS**: ON ✓
   - **DNS over HTTPS**: OFF
   - **DNS over TLS**: OFF
   - **DNS Rebinding Protection**: OFF
   - Click **Apply**

3. **Update ALL WireGuard configs to use matching DNS**:
   ```ini
   # Example: ALL configs should use 1.1.1.1
   [Interface]
   Address = 10.0.0.x/32  # Unique for each server
   DNS = 1.1.1.1          # Same DNS for all
   PrivateKey = YOUR_KEY
   
   [Peer]
   PublicKey = SERVER_PUBLIC_KEY
   Endpoint = your-server.com:51820
   AllowedIPs = 0.0.0.0/0, ::/0
   PersistentKeepalive = 25
   ```

**Why This Works Best**:
- ✅ No routing conflicts between different servers
- ✅ Consistent DNS behavior across all connections
- ✅ Fast, reliable Cloudflare DNS
- ✅ Works from any location globally

**Note**: While this doesn't show your "home ISP" DNS servers, it provides maximum reliability and avoids all the routing conflicts that occur with local DNS servers like 192.168.1.1.

### Initial Connection Test

1. **Start VPN Connection**
   - Go to **VPN** → **WireGuard Client**
   - Toggle connection ON
   - Wait for "Connected" status

2. **Initial Testing** (FROM YOUR LAPTOP TERMINAL)
   ```bash
   # Test 1: Check your public IP
   curl -s ifconfig.me
   # Should show the IP of whichever WireGuard server you're connected to

   # Test 2: Check DNS
   nslookup google.com
   # Should show DNS server from your WireGuard config
   ```

3. **Browser Quick Test**
   - Visit: https://ipleak.net
   - IP should show your WireGuard server's location
   - DNS servers should match what's configured in that specific WireGuard profile

**⚠️ STOP if the expected IP doesn't appear!**

---

## Phase 2: Terminal Hardening

### DNS Security Options

```bash
# FROM LAPTOP: SSH into router
ssh root@192.168.8.1
```

```bash
# FROM ROUTER TERMINAL:
# Force router to use ONLY Cloudflare DNS
uci set network.wan.peerdns='0'
uci delete network.wan.dns
uci add_list network.wan.dns='1.1.1.1'
uci add_list network.wan.dns='1.0.0.1'
uci commit network

# Configure dnsmasq to use only Cloudflare and ignore ISP DNS
uci delete dhcp.@dnsmasq[0].server
uci add_list dhcp.@dnsmasq[0].server='1.1.1.1'
uci add_list dhcp.@dnsmasq[0].server='1.0.0.1'
uci set dhcp.@dnsmasq[0].noresolv='1'  # Critical: Ignore ISP DNS
uci set dhcp.@dnsmasq[0].localuse='1'
uci commit dhcp

# Restart services
/etc/init.d/network restart
/etc/init.d/dnsmasq restart

# Verify configuration
cat /var/etc/dnsmasq.conf* | grep -E "server=|resolv"
# Should show:
# no-resolv
# server=1.1.1.1
# server=1.0.0.1

# Check dnsmasq is using correct servers
logread | grep dnsmasq | grep "using nameserver" | tail -5
# Should ONLY show 1.1.1.1 and 1.0.0.1
```

**Note**: Router will show `Server: 127.0.0.1` when running nslookup locally - this is normal. The important part is that it forwards ONLY to Cloudflare.

**TEST VPN**: Toggle VPN OFF/ON in Web UI, verify it connects

### IPv6 Complete Disable

```bash
# FROM ROUTER TERMINAL: Disable IPv6 using UCI
uci set network.lan.ipv6='0'
uci set network.wan.ipv6='0'
uci set network.wan6.disabled='1'
uci commit network

# Disable IPv6 at kernel level
echo "net.ipv6.conf.all.disable_ipv6=1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6=1" >> /etc/sysctl.conf
sysctl -p

# Disable IPv6 DHCP advertisements
uci set dhcp.lan.dhcpv6='disabled'
uci set dhcp.lan.ra='disabled'
uci set dhcp.lan.ra_management='0'
uci set dhcp.lan.ra_slaac='0'
uci commit dhcp

# Restart network
/etc/init.d/network restart
/etc/init.d/dnsmasq restart
```

**Note**: The `odhcpd` service may not exist on all GL.iNet models. The UCI commands above are sufficient to disable IPv6.

**TEST VPN**: Wait 30 seconds, toggle VPN OFF/ON, verify it connects

### Firewall Rules

```bash
# FROM ROUTER TERMINAL: Backup firewall
cp /etc/config/firewall /etc/config/firewall.backup

# Add proven working kill switch rules using UCI
# Rule 1: Block all IPv6
uci add firewall rule
uci set firewall.@rule[-1].name='Block_All_IPv6'
uci set firewall.@rule[-1].family='ipv6'
uci set firewall.@rule[-1].proto='all'
uci set firewall.@rule[-1].target='DROP'

# Rule 2: Force VPN for LAN to WAN
uci add firewall rule
uci set firewall.@rule[-1].name='Force_VPN_LAN_WAN'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest='wan'
uci set firewall.@rule[-1].proto='all'
uci set firewall.@rule[-1].target='REJECT'

# Rule 3: Allow VPN Traffic
uci add firewall rule
uci set firewall.@rule[-1].name='Allow_VPN_Traffic'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest='*'
uci set firewall.@rule[-1].proto='all'
uci set firewall.@rule[-1].family='ipv4'
uci set firewall.@rule[-1].device='wgclient'
uci set firewall.@rule[-1].target='ACCEPT'

# Rule 4: Block Non-VPN Traffic to WAN
uci add firewall rule
uci set firewall.@rule[-1].name='Block_Non_VPN_Traffic_to_WAN'
uci set firewall.@rule[-1].src='lan'
uci set firewall.@rule[-1].dest='wan'
uci set firewall.@rule[-1].proto='all'
uci set firewall.@rule[-1].family='ipv4'
uci set firewall.@rule[-1].target='REJECT'

# Commit and apply
uci commit firewall
/etc/init.d/firewall reload
```

**TEST VPN**: Toggle VPN OFF/ON, verify it connects

### Block Traceroute Responses (Privacy Enhancement)

This makes your router "invisible" to traceroute operations, preventing network reconnaissance.

```bash
# FROM ROUTER TERMINAL: Edit firewall.user
vi /etc/firewall.user
```

Add these lines to block ICMP time-exceeded messages:

```bash
# Block traceroute responses
iptables -I OUTPUT -p icmp --icmp-type time-exceeded -j DROP
iptables -I INPUT -p icmp --icmp-type time-exceeded -j DROP
```

Save and apply:
```bash
/etc/init.d/firewall reload
```

**What this does**: When someone runs `traceroute`, your router won't appear in the trace (shows as `* * *` instead of your IP)

**Note**: On newer GL.iNet firmware this rule may already be in place. Verify with:
```bash
iptables -L OUTPUT -n | grep 'icmp type 11'
```

### Kill Switch Implementation

> **⚠️ Interface Name Warning**: Newer GL.iNet firmware (SDK4) names the WireGuard client interface `wgclient1` instead of `wgclient`. Check yours first:
> ```bash
> ip link show | grep wg
> ```
> Use whichever name appears (`wgclient` or `wgclient1`) in the commands below.

**Step 1: Write the kill switch script**

```bash
# FROM ROUTER TERMINAL:
cat > /etc/firewall.user << 'EOF'
#!/bin/sh
# Kill switch - block all LAN traffic if VPN is down
# Uses -C to check before inserting - prevents duplicate rules on reload
iptables -C FORWARD -i br-lan ! -o wgclient1 -j DROP 2>/dev/null || iptables -I FORWARD -i br-lan ! -o wgclient1 -j DROP

# Block NTP from leaking your real WAN IP to time servers
iptables -C OUTPUT -o apcli0 -p udp --dport 123 -j DROP 2>/dev/null || iptables -I OUTPUT -o apcli0 -p udp --dport 123 -j DROP
iptables -C OUTPUT -o eth0 -p udp --dport 123 -j DROP 2>/dev/null || iptables -I OUTPUT -o eth0 -p udp --dport 123 -j DROP
EOF
```

> Replace `wgclient1` with `wgclient` if that is your interface name.

**Step 2: Register the script as a firewall include**

> **Critical**: On newer GL.iNet firmware, `/etc/firewall.user` does NOT run automatically on firewall reload. You must register it explicitly, otherwise the kill switch silently does nothing after a reboot.

```bash
uci set firewall.killswitch=include
uci set firewall.killswitch.type='script'
uci set firewall.killswitch.path='/etc/firewall.user'
uci set firewall.killswitch.reload='1'
uci commit firewall
```

**Step 3: Apply**

```bash
/etc/init.d/firewall reload
```

**Step 4: Verify — exactly one rule should appear**

```bash
iptables -L FORWARD -n -v | grep 'br-lan'
# Should show exactly ONE DROP rule for br-lan !wgclient1
```

> **Duplicate Rule Warning**: If you run the old-style `echo 'iptables -I FORWARD ...' >> /etc/firewall.user` without the `-C` check, every firewall reload stacks another copy of the rule. This causes unpredictable behavior. The idempotent version above (using `-C`) is safe to run multiple times.

**TEST KILL SWITCH**:
```bash
# FROM ROUTER: Check kill switch is catching traffic
iptables -Z FORWARD 1   # Zero the counter
# Wait 30 seconds with devices connected
iptables -L FORWARD -n -v | head -5
# Counter on rule 1 should be incrementing - traffic is being caught and dropped
```

```bash
# FROM LAPTOP: With VPN down, internet should fail
ping -c 2 8.8.8.8        # Should fail
curl -s ifconfig.me       # Should fail or hang
```

```bash
# FROM ROUTER: Restart VPN
/etc/init.d/vpn-client restart
```

```bash
# FROM LAPTOP: With VPN up, should work and show home IP
curl -s ifconfig.me
```

### NTP Leak Prevention

By default, your router syncs its clock by sending NTP requests directly over the WAN interface — bypassing the VPN entirely. This reveals your router's real WAN IP to time servers and the upstream hotel/ISP network.

The kill switch script above already includes NTP blocking. To verify it is active:

```bash
iptables -L OUTPUT -n | grep 'dpt:123'
# Should show DROP rules for eth0 and apcli0
```

**Note**: With NTP blocked, your router clock will not auto-sync. This is acceptable and actually desirable — you want the clock locked to your home timezone, not auto-updating to the local timezone wherever you travel.

### VPN Watchdog

The kill switch blocks traffic when the VPN drops, but it does not restart the VPN automatically. Without a watchdog, your internet stays dead until you manually reconnect. This script fixes that.

```bash
# FROM ROUTER TERMINAL: Create watchdog script
cat > /usr/bin/vpn-watchdog.sh << 'EOF'
#!/bin/sh
# Ping through the VPN tunnel - if no response, restart VPN client
if ! ping -c 2 -W 5 -I wgclient1 1.1.1.1 >/dev/null 2>&1; then
    logger -t vpn-watchdog 'VPN tunnel dead - restarting'
    /etc/init.d/vpn-client restart
fi
EOF
chmod +x /usr/bin/vpn-watchdog.sh

# Add to crontab - runs every 5 minutes
grep -q vpn-watchdog /etc/crontabs/root 2>/dev/null || \
  echo '*/5 * * * * /usr/bin/vpn-watchdog.sh' >> /etc/crontabs/root

# Enable and start cron
/etc/init.d/cron enable
/etc/init.d/cron restart
```

> Replace `wgclient1` with your actual interface name if different.

**Verify it is running**:
```bash
cat /etc/crontabs/root | grep vpn-watchdog
/etc/init.d/cron status
```

**Resource impact**: Negligible. Two ping packets every 5 minutes. The GL-MT3000 and similar routers handle WireGuard encryption continuously — this watchdog is a rounding error by comparison.

---

## Phase 3: Device Security

### Work Laptop Configuration

#### Network Settings
1. **Disable Wi-Fi Completely**:
   - Windows: Device Manager → Network Adapters → Disable Wi-Fi
   - macOS: System Preferences → Network → Wi-Fi → Turn Off
   - Linux: `rfkill block wifi`

2. **Ethernet Only**:
   - Use quality shielded Ethernet cable
   - Disable all other network interfaces

#### System Settings
1. **Time and Location**:
   - Set timezone to home timezone
   - Disable "Set time automatically"
   - Disable all location services

2. **Browser Configuration**:
   - Install WebRTC Leak Prevent extension
   - Disable location permissions
   - Clear cookies/cache before travel
   - **Disable DNS over HTTPS (DoH)**: Chrome → Settings → Privacy → Security → Use secure DNS → OFF. Firefox → Settings → Network Settings → DNS over HTTPS → OFF. DoH bypasses your router's DNS entirely, leaking queries directly to Google or Cloudflare from the browser.

#### macOS Specific IPv6 Cleanup
```bash
# Remove IPv6 from network interface
sudo networksetup -setv6off "USB 10/100/1000 LAN"

# Clear DNS cache
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

### MDM & Hardware Considerations

#### MDM Software (Jamf, Intune, etc.)

If your employer installed Mobile Device Management software on your work laptop, it can report device location, public IP, and compliance status directly to your company — completely independent of your router setup. The VPN protects network traffic; it does not protect what software running on the device reports.

**Check for MDM**:
- macOS: Look for **Jamf**, **Kandji**, or **Company Portal** in Applications or System Preferences → Profiles
- Windows: Settings → Accounts → Access work or school → look for enrolled management

**What MDM can see through your VPN**:
- Public IP → shows your home IP (handled) ✓
- WiFi-based location → no WiFi chip = no data ✓
- Bluetooth beacons → no Bluetooth chip = no data ✓

**What MDM can still see even with VPN**:
- WWAN/LTE cellular modem (if present) — reports location over cell network, completely bypassing your router
- System timezone and locale settings
- Any GPS chip (rare on laptops)

#### Physical Hardware Approach (Maximum Protection)

Removing the WiFi and Bluetooth chips from the work laptop and connecting via Ethernet only is the most effective hardware-level approach:

- No WiFi = no WiFi positioning, no SSID scanning
- No Bluetooth = no beacon tracking
- Ethernet through VPN router = all network traffic goes through your tunnel
- MDM's network reporting sees your home VPN IP

**Check for WWAN/LTE modem** — some business laptops (ThinkPad, Dell Latitude, HP EliteBook) include a cellular modem that operates independently:

```bash
# macOS
system_profiler SPModemDataType

# Windows (in PowerShell)
Get-PnpDevice | Where-Object {$_.FriendlyName -like "*Mobile Broadband*" -or $_.FriendlyName -like "*WWAN*"}
```

If found, disable it in BIOS/UEFI settings or physically remove the card.

### Router SSID Configuration

If you must use Wi-Fi:
- **SSID**: Generic name like "Router" or "Network"
- **Hide SSID**: Enabled
- **Encryption**: WPA3 (WPA2 minimum)
- **Password**: 20+ random characters

---

## Phase 4: Comprehensive Testing

### Command Line Tests

**FROM LAPTOP TERMINAL:**

```bash
# Test 1: Verify Home IP
curl -s ifconfig.me
# Must show your WireGuard server IP

# Test 2: DNS Resolution
nslookup google.com
# Server should be 192.168.8.1

# Test 3: Check all DNS servers (macOS)
scutil --dns | grep nameserver
# Should show your router IPs (192.168.8.1, possibly IPv6 locals)

# Test 4: Cloudflare DNS leak test
dig +short txt ch whoami.cloudflare @1.1.1.1
# Should show your WireGuard server IP
```

**FROM ROUTER TERMINAL:**

```bash
# Test 5: Verify DNS configuration
cat /var/etc/dnsmasq.conf* | grep -E "server=|resolv"
# Must show:
# no-resolv
# server=1.1.1.1
# server=1.0.0.1

# Test 6: Check active DNS servers
logread | grep dnsmasq | grep "using nameserver" | tail -5
# Should ONLY show 1.1.1.1 and 1.0.0.1

# Test 7: Direct DNS test
nslookup google.com 1.1.1.1
# Should resolve quickly via Cloudflare
```

### Browser Leak Tests

Run these from your work laptop every time you connect somewhere new — before opening any work apps.

1. **https://ipleak.net**
   - ✅ IP: Your home IP only
   - ✅ DNS: Home ISP/Cloudflare (if home uses it)
   - ✅ WebRTC: No local IPs

2. **https://dnsleaktest.com**
   - Run "Extended Test"
   - ✅ All servers from home ISP/location

3. **https://browserleaks.com/webrtc**
   - ✅ No local IP addresses visible

4. **https://test-ipv6.com**
   - ✅ "No IPv6 address detected"
   - ✅ Score: 0/10 (this is GOOD)

### Kill Switch Verification

```bash
# FROM ROUTER: Take VPN interface down
ip link set wgclient1 down
```

```bash
# FROM LAPTOP: Should fail immediately
ping -c 2 8.8.8.8
curl -s --max-time 5 ifconfig.me
```

```bash
# FROM ROUTER: Check kill switch caught the traffic
iptables -L FORWARD -n -v | head -5
# Packet counter on rule 1 should have incremented
```

```bash
# FROM ROUTER: Bring VPN back up
/etc/init.d/vpn-client restart
```

```bash
# FROM LAPTOP: Should work and show home IP
curl -s ifconfig.me
```

---

## Advanced Security Enhancements

### Hostname Masking (Reduce Network Fingerprinting)

By default, your router advertises `GL-MT3000` or `console.gl-inet.com` as its hostname, which reveals your router brand to anyone performing network reconnaissance.

**Recommended: Change to something generic**
```bash
uci set system.@system[0].hostname='router'
uci commit system
/etc/init.d/system restart
```

**Verify**:
```bash
uci get system.@system[0].hostname
# Should return: router
```

### MAC Address Randomization

Your router's WAN interface uses a factory MAC address with a GL.iNet OUI prefix, identifying the manufacturer to every network you join. Randomizing it prevents hotel/venue networks from tracking your device across visits.

```bash
# Generate random MAC (locally administered prefix 02: ensures it won't conflict)
RANDOM_MAC=$(printf '02:%02X:%02X:%02X:%02X:%02X' \
  $((RANDOM%256)) $((RANDOM%256)) $((RANDOM%256)) \
  $((RANDOM%256)) $((RANDOM%256)))
uci set network.wan.macaddr="$RANDOM_MAC"
uci commit network
/etc/init.d/network restart
```

**Note**: This affects what the hotel/upstream network sees — not your employer. Your employer only sees the VPN IP regardless of MAC address.

### Traffic Pattern Normalization

> **Skip if your kill switch is in place.** With a working kill switch, all LAN traffic goes through the WireGuard tunnel. The hotel/ISP network only sees encrypted WireGuard UDP — individual packet TTLs from LAN devices are never visible to them. TTL normalization is redundant in this configuration.

If you have a specific reason to apply it anyway:
```bash
echo "iptables -t mangle -A POSTROUTING -j TTL --ttl-set 64" >> /etc/firewall.user
/etc/init.d/firewall reload
```

---

## Operational Security (OPSEC) Guidelines

### Critical Rules

1. **The Golden Rule**: VPN must NEVER be disabled
   - Not for troubleshooting
   - Not for speed
   - Not for personal use

2. **Physical Security**:
   - Keep router hidden
   - Generic carrying case
   - Never discuss setup

3. **Digital Hygiene**:
   - No real-time social posts
   - No location tags
   - Maintain home timezone on ALL devices
   - Consistent work patterns
   - No unusual hours that don't match home timezone

4. **Video Call Discipline**:
   - Use a virtual background or photo of your home office
   - Be aware of window light angle revealing time of day
   - Keep backgrounds consistent

5. **Emergency Protocol**:
   - If VPN fails: Disconnect immediately
   - Use different device/network if critical
   - Report "internet issues" if asked

---

## Backup and Recovery

### Create Full Backup

```bash
# FROM ROUTER: Create complete backup including VPN configs
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p /root/backups

tar -czf /root/backups/config_$BACKUP_DATE.tar.gz \
  /etc/config/network \
  /etc/config/firewall \
  /etc/config/dhcp \
  /etc/config/wireless \
  /etc/config/system \
  /etc/config/wireguard \
  /etc/config/wireguard_server \
  /etc/config/ovpnclient \
  /etc/firewall.user \
  /etc/sysctl.conf \
  /etc/crontabs/root \
  /usr/bin/vpn-watchdog.sh

echo "Backup created: /root/backups/config_$BACKUP_DATE.tar.gz"
```

```bash
# FROM LAPTOP: Download backup via SSH pipe (use this if SCP fails)
ssh root@192.168.8.1 "cat /root/backups/config_*.tar.gz" > ~/Downloads/gl-router-backup.tar.gz
```

**What this backup contains**: All network, firewall, DNS, wireless, VPN (WireGuard + OpenVPN) configs, kill switch script, NTP block rules, VPN watchdog, and system settings. Restoring this gets you back to a fully configured state.

### Emergency Restore
```bash
# FROM ROUTER:
cd /
tar -xzf /path/to/backup.tar.gz
reboot
```

---

## Troubleshooting Guide

### VPN Won't Connect / Location-Specific Issues

If a WireGuard profile works sometimes but not others:

```bash
# 1. Check for IP conflicts
ip addr show
# Look for conflicts with tunnel addresses

# 2. Clear GL.iNet routing cache
ip route flush cache

# 3. Restart WireGuard completely
/etc/init.d/vpn-client restart

# 4. Check DNS resolution
nslookup your-endpoint.duckdns.org
# If this fails, your router's DNS might be blocked

# 5. For persistent issues, power cycle the router
reboot
```

**Common Issues by Location**:
- **Hotels/Corporate Networks**: Often block VPN ports
- **Mobile Hotspots**: May use strict NAT
- **Different Countries**: May have different VPN restrictions

**If one server works but another doesn't**:
1. Check for IP range conflicts (192.168.x.x is problematic)
2. Try deleting and re-adding the problematic profile
3. Verify the server is actually online and accessible
4. Consider using port 443 instead of 51820

### Kill Switch Showing Duplicate Rules

If `iptables -L FORWARD -n | grep br-lan` shows multiple identical DROP rules, the kill switch script ran multiple times without the idempotency check. Fix:

```bash
# Remove all duplicates
while iptables -D FORWARD -i br-lan ! -o wgclient1 -j DROP 2>/dev/null; do true; done

# Reload firewall - the idempotent script will add exactly one rule
/etc/init.d/firewall reload

# Verify exactly one rule remains
iptables -L FORWARD -n -v | grep br-lan
```

This only happens with the old-style script. The updated idempotent version (using `-C`) in this guide prevents it.

### Persistent IPv6 Addresses on macOS

If you still see IPv6 nameservers (fd00::) on macOS:

```bash
# On macOS, disable IPv6 for your network service
sudo networksetup -setv6off "USB 10/100/1000 LAN"
# or for Wi-Fi:
sudo networksetup -setv6off "Wi-Fi"

# Clear DNS cache
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

### Emergency Recovery
```bash
# Restore firewall only
cp /etc/config/firewall.backup /etc/config/firewall
/etc/init.d/firewall restart

# Remove kill switch
rm /etc/firewall.user
uci delete firewall.killswitch
uci commit firewall
/etc/init.d/firewall restart
```

---

## Advanced Detection Methods & Mitigations

### ⚠️ Potential Detection Vectors

**1. MDM Software (Highest Risk)**
- If employer MDM is installed, it can report IP, location, and device posture independently
- **Mitigation**: Remove WiFi/Bluetooth chips, use Ethernet-only through VPN router, check for and disable WWAN/LTE modem

**2. DNS over HTTPS in Browser**
- Chrome and Firefox can bypass your router's DNS using DoH, leaking queries directly to Google/Cloudflare
- **Mitigation**: Disable DoH in all browsers on work devices

**3. Timezone/Locale Slip**
- Email headers, calendar invites, and Slack timestamps embed timezone data
- **Mitigation**: Lock timezone manually on every device, disable auto-update

**4. Packet Timing Analysis**
- VPN adds latency (~30-50ms to home server)
- Sophisticated monitoring could detect round-trip delay
- **Mitigation**: Requires targeted deep packet inspection — unlikely in corporate environments

**5. VPN Protocol Detection**
- WireGuard has identifiable packet patterns
- **Mitigation**: Would require specifically looking for VPN usage

**6. Bandwidth/Usage Patterns**
- Sudden bandwidth changes could raise flags
- **Mitigation**: Maintain normal work patterns

**7. Device Fingerprinting**
- Browser fingerprints, OS metrics
- **Mitigation**: Don't install new software or change settings while traveling

**8. Network Reconnaissance**
- Home network device scans could reveal absence
- **Mitigation**: Keep some devices at home powered on (smart plugs, lights on timers)

### Real-World Assessment

**For 99% of companies**, this setup is **more than sufficient**. You would defeat:
- Standard IT monitoring tools
- Geo-location restrictions
- Basic security audits
- Automated compliance checks

**Only nation-state level actors or companies with advanced threat detection** would potentially catch this, and they'd need to:
1. Be specifically looking for VPN usage
2. Correlate multiple data points
3. Have deep packet inspection
4. Care enough to investigate

### Confidence Level: 95%

Your setup would fool any standard corporate monitoring. The only improvements would be:
- Using OpenVPN (older, less identifiable protocol)
- Adding traffic obfuscation
- Using a VPS instead of home connection

But these add complexity with minimal security gain for your use case.

**Bottom line**: Unless your company is doing NSA-level surveillance (they're not), you're golden. Just maintain good OpSec:
- Keep consistent hours
- Don't brag about travel
- Maintain normal work patterns
- Keep your timezone set to home

---

## Final Security Checklist

### Router
- [ ] WireGuard shows active connection (`wg show`)
- [ ] IP shows your home address (`curl ifconfig.me`)
- [ ] DNS shows your home router (`nslookup google.com`)
- [ ] Kill switch blocks traffic when VPN down
- [ ] Kill switch registered as UCI firewall include
- [ ] No duplicate kill switch rules (`iptables -L FORWARD -n | grep br-lan` shows exactly 1)
- [ ] NTP leak blocked (`iptables -L OUTPUT -n | grep dpt:123`)
- [ ] VPN watchdog running (`/etc/init.d/cron status`)
- [ ] IPv6 completely disabled (`test-ipv6.com` shows 0/10)
- [ ] Hostname masked (`uci get system.@system[0].hostname` returns generic name)
- [ ] Traceroute responses blocked
- [ ] Backup created and stored safely (includes WireGuard configs)

### Work Device
- [ ] All browser leak tests pass
- [ ] Timezone set to home — manually, auto-update OFF
- [ ] Location services disabled
- [ ] Wi-Fi disabled or chip removed
- [ ] Bluetooth disabled or chip removed
- [ ] DNS over HTTPS disabled in browser
- [ ] WWAN/LTE modem checked and disabled if present
- [ ] WebRTC blocked (extension or uBlock Origin)
- [ ] MDM software checked — understand what it can report

---

## Success Indicators

✅ **Perfect Setup**:
- All leak tests show home location only
- DNS matches your home network exactly
- Kill switch tested and working — blocks traffic instantly when VPN drops, auto-reconnects via watchdog
- No IPv6 connectivity
- Indistinguishable from being at home
- Router doesn't reveal brand/model in network traces
- No NTP requests leaking via WAN

Remember: **One mistake can compromise everything**. Test thoroughly before every trip and maintain strict discipline.

**Stay safe and enjoy your travels!**
