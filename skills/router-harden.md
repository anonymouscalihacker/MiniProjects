# GL-MT3000 Router Hardening Skill

SSH into the GL-MT3000 travel router at 192.168.8.1 and check every hardening item. For anything missing or broken, apply the fix automatically. Report a clear pass/fail for each item at the end.

## Router Details
- IP: 192.168.8.1
- User: root
- SSH key already installed
- WireGuard interface: wgclient1
- WAN interfaces: eth0, apcli0

## Instructions

Run all checks over SSH. Do not ask the user for confirmation on fixes — just apply them and report what was done. Work through each section in order.

---

### 1. Kill Switch

Check that exactly one DROP rule exists in the FORWARD chain for br-lan:

```bash
ssh root@192.168.8.1 "iptables -L FORWARD -n -v | grep 'br-lan'"
```

**Pass**: exactly one line matching `DROP all -- br-lan !wgclient1`  
**Fail**: missing, or more than one (duplicates)

If missing — write the firewall.user script and register the UCI include:

```bash
ssh root@192.168.8.1 "
cat > /etc/firewall.user << 'EOF'
#!/bin/sh
iptables -C FORWARD -i br-lan ! -o wgclient1 -j DROP 2>/dev/null || iptables -I FORWARD -i br-lan ! -o wgclient1 -j DROP
iptables -C OUTPUT -o apcli0 -p udp --dport 123 -j DROP 2>/dev/null || iptables -I OUTPUT -o apcli0 -p udp --dport 123 -j DROP
iptables -C OUTPUT -o eth0 -p udp --dport 123 -j DROP 2>/dev/null || iptables -I OUTPUT -o eth0 -p udp --dport 123 -j DROP
EOF
uci set firewall.killswitch=include
uci set firewall.killswitch.type='script'
uci set firewall.killswitch.path='/etc/firewall.user'
uci set firewall.killswitch.reload='1'
uci commit firewall
/etc/init.d/firewall reload
"
```

If duplicates exist — remove all and reload:

```bash
ssh root@192.168.8.1 "
while iptables -D FORWARD -i br-lan ! -o wgclient1 -j DROP 2>/dev/null; do true; done
/etc/init.d/firewall reload
"
```

---

### 2. UCI Firewall Include Registration

Check that the killswitch script is registered so it survives reboots:

```bash
ssh root@192.168.8.1 "uci show firewall.killswitch 2>/dev/null"
```

**Pass**: returns `firewall.killswitch=include` with path `/etc/firewall.user`  
**Fail**: empty or missing

Fix is included in the kill switch fix above.

---

### 3. NTP Leak Block

Check that NTP (port 123) is blocked on WAN interfaces:

```bash
ssh root@192.168.8.1 "iptables -L OUTPUT -n | grep 'dpt:123'"
```

**Pass**: two DROP rules — one for eth0, one for apcli0  
**Fail**: missing

If missing — add to firewall.user (idempotent) and reload:

```bash
ssh root@192.168.8.1 "
grep -q 'dport 123' /etc/firewall.user 2>/dev/null || {
cat >> /etc/firewall.user << 'EOF'
iptables -C OUTPUT -o apcli0 -p udp --dport 123 -j DROP 2>/dev/null || iptables -I OUTPUT -o apcli0 -p udp --dport 123 -j DROP
iptables -C OUTPUT -o eth0 -p udp --dport 123 -j DROP 2>/dev/null || iptables -I OUTPUT -o eth0 -p udp --dport 123 -j DROP
EOF
}
/etc/init.d/firewall reload
"
```

---

### 4. VPN Watchdog

Check that the watchdog script exists and the cron job is active:

```bash
ssh root@192.168.8.1 "
ls -la /usr/bin/vpn-watchdog.sh 2>/dev/null || echo 'MISSING'
cat /etc/crontabs/root 2>/dev/null | grep vpn-watchdog || echo 'CRON MISSING'
/etc/init.d/cron status
"
```

**Pass**: script exists, cron entry present, cron running  
**Fail**: any of the three missing

If missing — create script, add cron, restart cron:

```bash
ssh root@192.168.8.1 "
cat > /usr/bin/vpn-watchdog.sh << 'EOF'
#!/bin/sh
if ! ping -c 2 -W 5 -I wgclient1 1.1.1.1 >/dev/null 2>&1; then
    logger -t vpn-watchdog 'VPN tunnel dead - restarting'
    /etc/init.d/vpn-client restart
fi
EOF
chmod +x /usr/bin/vpn-watchdog.sh
grep -q vpn-watchdog /etc/crontabs/root 2>/dev/null || echo '*/5 * * * * /usr/bin/vpn-watchdog.sh' >> /etc/crontabs/root
/etc/init.d/cron enable
/etc/init.d/cron restart
"
```

---

### 5. Traceroute Block

Check that ICMP time-exceeded (type 11) is blocked in OUTPUT:

```bash
ssh root@192.168.8.1 "iptables -L OUTPUT -n | grep 'icmp type 11'"
```

**Pass**: at least one DROP rule for icmptype 11  
**Fail**: missing

If missing:

```bash
ssh root@192.168.8.1 "iptables -I OUTPUT -p icmp --icmp-type time-exceeded -j DROP"
```

Also add to firewall.user for persistence:

```bash
ssh root@192.168.8.1 "
grep -q 'time-exceeded' /etc/firewall.user 2>/dev/null || echo 'iptables -C OUTPUT -p icmp --icmp-type time-exceeded -j DROP 2>/dev/null || iptables -I OUTPUT -p icmp --icmp-type time-exceeded -j DROP' >> /etc/firewall.user
"
```

---

### 6. IPv6 Disabled

Check kernel-level and UCI:

```bash
ssh root@192.168.8.1 "
sysctl net.ipv6.conf.all.disable_ipv6
uci get network.wan.ipv6 2>/dev/null
uci get network.lan.ipv6 2>/dev/null
uci get dhcp.lan.dhcpv6 2>/dev/null
"
```

**Pass**: disable_ipv6 = 1, wan.ipv6 = 0, lan.ipv6 = 0, dhcpv6 = disabled  
**Fail**: any showing enabled or missing

If missing — apply full IPv6 disable:

```bash
ssh root@192.168.8.1 "
uci set network.lan.ipv6='0'
uci set network.wan.ipv6='0'
uci set network.wan6.disabled='1'
uci commit network
grep -q 'disable_ipv6=1' /etc/sysctl.conf || {
  echo 'net.ipv6.conf.all.disable_ipv6=1' >> /etc/sysctl.conf
  echo 'net.ipv6.conf.default.disable_ipv6=1' >> /etc/sysctl.conf
}
sysctl -p
uci set dhcp.lan.dhcpv6='disabled'
uci set dhcp.lan.ra='disabled'
uci set dhcp.lan.ra_management='0'
uci set dhcp.lan.ra_slaac='0'
uci commit dhcp
"
```

---

### 7. DNS Hardening

Check that dnsmasq ignores ISP DNS and uses Cloudflare only:

```bash
ssh root@192.168.8.1 "
uci get dhcp.@dnsmasq[0].noresolv 2>/dev/null
uci get dhcp.@dnsmasq[0].server 2>/dev/null
uci get network.wan.peerdns 2>/dev/null
"
```

**Pass**: noresolv = 1, server includes 1.1.1.1 and 1.0.0.1, peerdns = 0  
**Fail**: any missing or wrong

If missing:

```bash
ssh root@192.168.8.1 "
uci set network.wan.peerdns='0'
uci set dhcp.@dnsmasq[0].noresolv='1'
uci set dhcp.@dnsmasq[0].localuse='1'
uci delete dhcp.@dnsmasq[0].server 2>/dev/null
uci add_list dhcp.@dnsmasq[0].server='1.1.1.1'
uci add_list dhcp.@dnsmasq[0].server='1.0.0.1'
uci commit dhcp
uci commit network
/etc/init.d/dnsmasq restart
"
```

---

### 8. Hostname

Check the hostname is not revealing the router brand:

```bash
ssh root@192.168.8.1 "uci get system.@system[0].hostname"
```

**Pass**: returns something generic (not `GL-MT3000`, `GL-MT2500`, etc.)  
**Fail**: returns a GL.iNet model name

If failing:

```bash
ssh root@192.168.8.1 "
uci set system.@system[0].hostname='router'
uci commit system
/etc/init.d/system restart
"
```

---

### 9. VPN Status

Check that WireGuard is connected and passing traffic:

```bash
ssh root@192.168.8.1 "
ip link show wgclient1 2>/dev/null | grep -E 'UP|DOWN|UNKNOWN'
wg show wgclient1 2>/dev/null | grep -E 'transfer|endpoint|peer'
"
```

**Pass**: interface UP, peer present, transfer shows received bytes  
**Warn**: interface up but 0 bytes received (tunnel established but no traffic yet)  
**Fail**: interface missing or down

If VPN is down, restart:

```bash
ssh root@192.168.8.1 "/etc/init.d/vpn-client restart"
```

---

## Final Report

After all checks and fixes, collect the final state and present a clear summary table:

```bash
ssh root@192.168.8.1 "
echo '=== KILL SWITCH ===' && iptables -L FORWARD -n | grep 'br-lan' | wc -l
echo '=== UCI INCLUDE ===' && uci show firewall.killswitch 2>/dev/null | head -1
echo '=== NTP BLOCK ===' && iptables -L OUTPUT -n | grep 'dpt:123' | wc -l
echo '=== TRACEROUTE BLOCK ===' && iptables -L OUTPUT -n | grep 'icmp type 11' | wc -l
echo '=== IPV6 KERNEL ===' && sysctl -n net.ipv6.conf.all.disable_ipv6
echo '=== DNS NORESOLV ===' && uci get dhcp.@dnsmasq[0].noresolv 2>/dev/null
echo '=== HOSTNAME ===' && uci get system.@system[0].hostname
echo '=== WATCHDOG ===' && grep -c vpn-watchdog /etc/crontabs/root 2>/dev/null
echo '=== VPN INTERFACE ===' && ip link show wgclient1 2>/dev/null | grep -o 'UP\|UNKNOWN\|DOWN' | head -1
echo '=== CRON RUNNING ===' && /etc/init.d/cron status
"
```

Present results as a table with ✅ / ⚠️ / ❌ for each item. Explain any failures and confirm what was fixed.
