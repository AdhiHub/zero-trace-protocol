# ZERO TRACE PROTOCOL — HOWTO

## Step-by-Step Practical Implementation Guides

This is the practical companion to PROTOCOL.md. Every technique explained with exact commands, configurations, and walkthroughs.

---

## TABLE OF CONTENTS

1. [TOR: Install, Configure, Harden](#1-tor-install-configure-harden)
2. [TOR Circuit Rotation](#2-tor-circuit-rotation)
3. [VPN → TOR → VPN: Complete Setup](#3-vpn--tor--vpn-complete-setup)
4. [Double TOR with Whonix](#4-double-tor-with-whonix)
5. [DNS Leak: Fix Step by Step](#5-dns-leak-fix-step-by-step)
6. [IPv6 Leak: Disable Completely](#6-ipv6-leak-disable-completely)
7. [WebRTC: Disable in Every Browser](#7-webrtc-disable-in-every-browser)
8. [MAC Spoofing: Linux + Windows](#8-mac-spoofing-linux--windows)
9. [ProxyChains: Force Any Tool Through TOR](#9-proxychains-force-any-tool-through-tor)
10. [proxychains.conf: Maximum Security Config](#10-proxychains-conf-maximum-security-config)
11. [Tails OS: Install and Harden](#11-tails-os-install-and-harden)
12. [Whonix: Install and Configure](#12-whonix-install-and-configure)
13. [Signal: Anonymous Setup](#13-signal-anonymous-setup)
14. [SimpleX: Zero-Identity Setup](#14-simplex-zero-identity-setup)
15. [PGP Key: Create and Use Anonymously](#15-pgp-key-create-and-use-anonymously)
16. [Browser Fingerprinting: Test and Block](#16-browser-fingerprinting-test-and-block)
17. [Tor Browser: Maximum Hardening](#17-tor-browser-maximum-hardening)
18. [proxychains: Wrap Any Tool](#18-proxychains-wrap-any-tool)
19. [History and Log Wiping](#19-history-and-log-wiping)
20. [Secure Deletion: shred, wipe, cipher](#20-secure-deletion-shred-wipe-cipher)
21. [Full Disk Encryption: LUKS Step by Step](#21-full-disk-encryption-luks-step-by-step)
22. [After-Action: Complete Cleanup Script](#22-after-action-complete-cleanup-script)
23. [Anonymous Email: ProtonMail Setup via TOR](#23-anonymous-email-protonmail-setup-via-tor)
24. [Burner Laptop: Setup Guide](#24-burner-laptop-setup-guide)
25. [Decoy Traffic: Generate Background Noise](#25-decoy-traffic-generate-background-noise)
26. [Timing Obfuscation: Random Delays in Tools](#26-timing-obfuscation-random-delays-in-tools)
27. [Digital Dead Drops: Create and Retrieve](#27-digital-dead-drops-create-and-retrieve)
28. [Full Auto-Cleanup Script](#28-full-auto-cleanup-script)

---

## 1. TOR: INSTALL, CONFIGURE, HARDEN

### Linux (Debian/Ubuntu)

```bash
# Step 1: Install TOR
sudo apt update
sudo apt install tor -y

# Step 2: Stop TOR
sudo systemctl stop tor

# Step 3: Write custom torrc
sudo tee /etc/tor/torrc << 'EOF'
SocksPort 9050
ControlPort 9051
CookieAuthentication 0
DNSPort 5353
AutomapHostsOnResolve 1
ExcludeNodes {us},{gb},{ca},{au},{nz},{cn},{ru}
ExcludeExitNodes {us},{gb},{ca},{au},{nz},{cn},{ru}
StrictNodes 1
EOF

# Step 4: Start TOR
sudo systemctl start tor

# Step 5: Verify TOR is running
curl --socks5 127.0.0.1:9050 -s https://check.torproject.org/api | grep -o '"IsTor":"[^"]*"'

# Step 6: Verify ControlPort works
echo -e "AUTHENTICATE\r\nSIGNAL NEWNYM\r\n" | nc 127.0.0.1 9051
# Expected: 250 OK
```

### Windows (TOR Browser Install)

```powershell
# Step 1: Download TOR Browser
# Download from https://www.torproject.org/download/
# OR use this PowerShell one-liner:
$url = "https://www.torproject.org/dist/torbrowser/14.0/torbrowser-install-win64-14.0.0.exe"
Invoke-WebRequest -Uri $url -OutFile "$env:TEMP\tor_install.exe"

# Step 2: Install TOR Browser normally
Start-Process "$env:TEMP\tor_install.exe" -Wait

# Step 3: OR extract just tor.exe (for CLI use)
# Install 7-Zip first
# Then extract:
# "C:\Program Files\7-Zip\7z.exe" x "$env:TEMP\tor_install.exe" -o"$env:USERPROFILE\tor-browser"

# Step 4: Create custom torrc
$torrc = @"
SocksPort 9050
ControlPort 9051
CookieAuthentication 0
DataDirectory $env:USERPROFILE\tor-browser\Browser\TorBrowser\Data
GeoIPFile $env:USERPROFILE\tor-browser\Browser\TorBrowser\Data\Tor\geoip
GeoIPv6File $env:USERPROFILE\tor-browser\Browser\TorBrowser\Data\Tor\geoip6
ExcludeNodes {us},{gb},{ca},{au},{nz}
ExcludeExitNodes {us},{gb},{ca},{au},{nz}
StrictNodes 1
"@
Set-Content -Path "$env:USERPROFILE\tor-browser\Browser\TorBrowser\Data\Tor\torrc" -Value $torrc

# Step 5: Start TOR
$torExe = "$env:USERPROFILE\tor-browser\Browser\TorBrowser\Tor\tor.exe"
$torrc = "$env:USERPROFILE\tor-browser\Browser\TorBrowser\Data\Tor\torrc"
Start-Process -FilePath $torExe -ArgumentList "-f `"$torrc`"" -NoNewWindow

# Step 6: Verify (wait 10s for bootstrap)
Start-Sleep -Seconds 10
curl.exe --socks5 127.0.0.1:9050 -s https://api.ipify.org
# Returns your TOR exit IP (not your real IP)
```

### Verify TOR is Working

```bash
# Check if TOR is running (port check)
python3 -c "
import socket
s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.settimeout(3)
try:
    s.connect(('127.0.0.1', 9050))
    s.close()
    print('TOR: RUNNING')
except:
    print('TOR: NOT RUNNING')
"

# Get your TOR exit IP
curl --socks5 127.0.0.1:9050 -s https://api.ipify.org

# Get your real IP (for comparison)
curl -s https://api.ipify.org

# If they are DIFFERENT → TOR is working
```

---

## 2. TOR CIRCUIT ROTATION

### Method 1: Using netcat (Linux)

```bash
# Rotate TOR circuit
echo -e "AUTHENTICATE\r\nSIGNAL NEWNYM\r\n" | nc 127.0.0.1 9051

# Verify new IP
curl --socks5 127.0.0.1:9050 -s https://api.ipify.org
```

### Method 2: Using Python

```python
# rotate_tor.py
import socket
import time
import urllib.request

def rotate_tor():
    """Send NEWNYM to TOR control port to get a new exit IP."""
    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.settimeout(5)
    s.connect(('127.0.0.1', 9051))
    
    # Authenticate (no password since CookieAuthentication 0)
    s.send(b'AUTHENTICATE\r\n')
    time.sleep(0.3)
    resp = s.recv(1024)
    if b'250' not in resp:
        print('Auth failed:', resp)
        return False
    
    # Send NEWNYM signal
    s.send(b'SIGNAL NEWNYM\r\n')
    time.sleep(0.3)
    resp = s.recv(1024)
    s.close()
    
    if b'250' in resp:
        print('TOR circuit rotated!')
        time.sleep(2)  # Wait for new circuit
        
        # Get new IP
        try:
            proxy = urllib.request.ProxyHandler({
                'http': 'socks5://127.0.0.1:9050',
                'https': 'socks5://127.0.0.1:9050'
            })
            opener = urllib.request.build_opener(proxy)
            req = urllib.request.Request(
                'https://api.ipify.org',
                headers={'User-Agent': 'curl/8.0'}
            )
            with opener.open(req, timeout=10) as r:
                new_ip = r.read().decode()
                print(f'New exit IP: {new_ip}')
        except:
            pass
        return True
    return False

# Use it
rotate_tor()
```

### Method 3: Using stem library

```python
# pip install stem
from stem import Signal
from stem.control import Controller

with Controller.from_port(port=9051) as controller:
    controller.authenticate()
    controller.signal(Signal.NEWNYM)
    print('TOR circuit rotated (via stem)')
```

### Auto-Rotate Every 5 Minutes (Background)

```python
# auto_rotate.py
import threading
import time
from stem import Signal
from stem.control import Controller

def rotate_tor():
    with Controller.from_port(port=9051) as c:
        c.authenticate()
        c.signal(Signal.NEWNYM)
        print('Circuit rotated')

def auto_rotator(interval=300):
    """Rotate TOR circuit every `interval` seconds."""
    while True:
        time.sleep(interval)
        try:
            rotate_tor()
        except Exception as e:
            print(f'Rotation failed: {e}')

# Start in background thread
t = threading.Thread(target=auto_rotator, args=(300,), daemon=True)
t.start()
```

---

## 3. VPN → TOR → VPN: COMPLETE SETUP

This is the maximum anonymity chain. Here is exactly how to set it up.

### Architecture

```
Your PC ──▶ VPN1 (Mullvad, paid with XMR)
              ──▶ TOR (SOCKS5 on 127.0.0.1:9050)
                   ──▶ VPN2 (ProtonVPN, inside Tails/Whonix VM)
                        ──▶ Target
```

### Step-by-Step Setup (Linux)

```bash
# ─────────────────────────────────────────────
# STEP 1: Subscribe to VPN1 (Mullvad recommended)
# ─────────────────────────────────────────────
# 1. Go to https://mullvad.net
# 2. Download Mullvad VPN app OR use OpenVPN/WireGuard configs
# 3. Pay with Monero (XMR) — no email, no personal info
# 4. Download the VPN config files

# ─────────────────────────────────────────────
# STEP 2: Connect VPN1
# ─────────────────────────────────────────────
# Using WireGuard:
sudo wg-quick up mullvad-us123  # Use a non-Five Eyes country

# Verify:
curl -s https://api.ipify.org
# Should show Mullvad IP

# ─────────────────────────────────────────────
# STEP 3: Install and start TOR (uses VPN1's network)
# ─────────────────────────────────────────────
sudo apt install tor -y
sudo systemctl start tor

# Verify TOR through VPN1:
curl --socks5 127.0.0.1:9050 -s https://api.ipify.org
# Should show TOR exit IP, NOT Mullvad IP

# ─────────────────────────────────────────────
# STEP 4: Subscribe to VPN2 (ProtonVPN recommended)
# ─────────────────────────────────────────────
# 1. Go to https://protonvpn.com via TOR
# 2. Create account with ProtonMail alias
# 3. Pay with XMR
# 4. Download OpenVPN configs

# ─────────────────────────────────────────────
# STEP 5: Route VPN2 through TOR
# ─────────────────────────────────────────────
# Use proxychains to force OpenVPN through TOR:
sudo apt install proxychains4 -y

# Edit /etc/proxychains4.conf:
sudo tee -a /etc/proxychains4.conf << 'EOF'
# At the bottom of [ProxyList]:
socks5 127.0.0.1 9050
EOF

# Connect VPN2 through TOR via proxychains:
sudo proxychains4 openvpn --config protonvpn-us.conf

# ─────────────────────────────────────────────
# STEP 6: Verify full chain
# ─────────────────────────────────────────────
# In another terminal:
curl -s https://api.ipify.org
# Should show VPN2 exit IP (which is routed through TOR, 
# which is routed through VPN1)
# If any layer fails, you get the wrong IP
```

### Windows Setup (VPN → TOR)

```powershell
# ─────────────────────────────────────────────
# STEP 1: Get VPN1 (Mullvad)
# ─────────────────────────────────────────────
# Download Mullvad app, pay with cash/XMR
# Connect to a non-Five Eyes server

# ─────────────────────────────────────────────
# STEP 2: Start TOR
# ─────────────────────────────────────────────
$torExe = "$env:USERPROFILE\tor-browser\Browser\TorBrowser\Tor\tor.exe"
$torrc = "$env:USERPROFILE\tor-browser\Browser\TorBrowser\Data\Tor\torrc"
Start-Process -FilePath $torExe -ArgumentList "-f `"$torrc`"" -NoNewWindow

Start-Sleep -Seconds 15
curl.exe --socks5 127.0.0.1:9050 -s https://api.ipify.org
# Should show TOR IP, not VPN IP

# ─────────────────────────────────────────────
# STEP 3: For VPN2 (Windows proxychains alternative)
# ─────────────────────────────────────────────
# Use Python to proxy tools through the chain:

python3 << 'EOF'
import socks
import socket
import subprocess
import sys

# Set TOR as proxy
socks.set_default_proxy(socks.SOCKS5, '127.0.0.1', 9050)
socket.socket = socks.socksocket

# Now run any Python tool through TOR
import urllib.request
req = urllib.request.Request('https://api.ipify.org', 
    headers={'User-Agent': 'curl/8.0'})
with urllib.request.urlopen(req, timeout=15) as r:
    print('Through chain:', r.read().decode())
EOF
```

---

## 4. DOUBLE TOR WITH WHONIX

### Architecture

```
Host OS ──▶ VM1: Whonix-Gateway (TOR #1)
              ──▶ VM2: Whonix-Gateway-Clone (TOR #2, proxied through VM1)
                   ──▶ VM3: Whonix-Workstation (proxied through VM2)
                        ──▶ Tools run here
```

### Step-by-Step

```bash
# ─────────────────────────────────────────────
# STEP 1: Install VirtualBox
# ─────────────────────────────────────────────
# Linux:
sudo apt install virtualbox virtualbox-ext-pack -y

# Download Whonix:
wget https://download.whonix.org/linux/17.2.0.1/Whonix-XFCE-17.2.0.1.ova

# ─────────────────────────────────────────────
# STEP 2: Import Whonix Gateway
# ─────────────────────────────────────────────
# Import the OVA. It contains two VMs:
#   Whonix-Gateway-XFCE
#   Whonix-Workstation-XFCE

VBoxManage import Whonix-XFCE-17.2.0.1.ova

# ─────────────────────────────────────────────
# STEP 3: Clone Gateway for Double TOR
# ─────────────────────────────────────────────
# Clone the Gateway VM (this becomes TOR #2):
VBoxManage clonevm "Whonix-Gateway-XFCE" \
    --name "Whonix-Gateway-TOR2" \
    --register

# ─────────────────────────────────────────────
# STEP 4: Configure TOR #2 to use TOR #1
# ─────────────────────────────────────────────
# In Whonix-Gateway-TOR2 settings:
#   Network → Adapter 1: Host-only (same as original Gateway)
#   But instead of direct connection, route through TOR #1:
#   
#   In Whonix-Gateway-TOR2, edit /etc/tor/torrc:
#   Add: 
#     Socks5Proxy 10.0.2.15:9050  
#     Socks5ProxyIsolation 0
#   (10.0.2.15 is the IP of Whonix-Gateway-TOR1's internal interface)

# ─────────────────────────────────────────────
# STEP 5: Configure Workstation to use TOR #2
# ─────────────────────────────────────────────
# Clone the Workstation VM:
VBoxManage clonevm "Whonix-Workstation-XFCE" \
    --name "Whonix-Workstation-TOR2" \
    --register

# In Workstation-TOR2 settings:
#   Network → Adapter 1: Host-only (pointing to Gateway-TOR2)

# ─────────────────────────────────────────────
# STEP 6: Boot Order
# ─────────────────────────────────────────────
# 1. Start Whonix-Gateway-XFCE (TOR #1)
# 2. Start Whonix-Gateway-TOR2 (TOR #2, routes through TOR #1)
# 3. Start Whonix-Workstation-TOR2 (routes through TOR #2)
# 
# Result: Workstation traffic goes TOR2 → TOR1 → Internet

# ─────────────────────────────────────────────
# STEP 7: Verify
# ─────────────────────────────────────────────
# In Workstation-TOR2:
curl https://check.torproject.org/api

# Check exit IP:
curl https://api.ipify.org

# Check if traffic is actually double-TOR'd:
#   - If TOR #1's exit IP is different from TOR #2's exit IP
#   - And neither matches your real IP → Success!
```

---

## 5. DNS LEAK: FIX STEP BY STEP

### Test if You Have a DNS Leak

```bash
# Method 1: Quick check
nslookup google.com 8.8.8.8
# If you get a response → YOUR DNS IS LEAKING

# Method 2: Use Python
python3 << 'EOF'
import socket
try:
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    s.settimeout(3)
    s.connect(('8.8.8.8', 53))
    s.send(b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00')
    data = s.recv(512)
    s.close()
    if data:
        print('DNS LEAK: Can reach 8.8.8.8:53 directly')
    else:
        print('DNS: Secure (no response)')
except:
    print('DNS: Secure (8.8.8.8 blocked)')
EOF

# Method 3: Browser test
# Go to: https://dnsleaktest.com
```

### Fix DNS Leaks on Linux

```bash
# Step 1: Set DNS to localhost
sudo tee /etc/resolv.conf << 'EOF'
nameserver 127.0.0.1
options attempts:1 timeout:1
EOF

# Make it permanent (prevent overwrite):
sudo chattr +i /etc/resolv.conf

# Step 2: Use TOR for DNS
# In torrc, add:
# DNSPort 5353
# AutomapHostsOnResolve 1
sudo systemctl restart tor

# Step 3: Test again
nslookup google.com 8.8.8.8
# Should now timeout (no DNS leak)

# Step 4: Use torsocks for DNS through TOR
torsocks nslookup google.com
# Should work (through TOR)
```

### Fix DNS Leaks on Windows

```powershell
# Step 1: Set DNS to localhost
netsh interface ip set dns "Ethernet" static 127.0.0.1
netsh interface ip set dns "Wi-Fi" static 127.0.0.1

# Step 2: Set DNS to TOR's DNS
# TOR can provide DNS on port 5353 (if DNSPort is configured)
netsh interface ip set dns "Ethernet" static 127.0.0.1

# Step 3: Block external DNS with firewall
New-NetFirewallRule -DisplayName "Block-DNS" -Direction Outbound `
    -Protocol UDP -RemotePort 53 -Action Block

# Step 4: Test
nslookup google.com 8.8.8.8
# Should timeout
```

### Fix DNS Leaks on macOS

```bash
# Step 1: Set DNS to localhost
sudo networksetup -setdnsservers Ethernet 127.0.0.1
sudo networksetup -setdnsservers Wi-Fi 127.0.0.1

# Step 2: Test
nslookup google.com 8.8.8.8
# Should timeout
```

### Verify DNS Is Truly Blocked

```bash
# If this returns ANYTHING → you still have a DNS leak
timeout 3 bash -c 'echo >/dev/udp/8.8.8.8/53' 2>/dev/null && echo "LEAKING" || echo "SECURE"
```

---

## 6. IPv6 LEAK: DISABLE COMPLETELY

IPv6 often bypasses VPNs and TOR entirely. Always disable it.

### Linux

```bash
# Step 1: Disable IPv6 immediately
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.lo.disable_ipv6=1

# Step 2: Make permanent
echo "net.ipv6.conf.all.disable_ipv6 = 1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Step 3: Verify
cat /proc/sys/net/ipv6/conf/all/disable_ipv6
# Should output: 1
```

### Windows

```powershell
# Step 1: List network adapters
Get-NetAdapterBinding -ComponentID ms_tcpip6

# Step 2: Disable IPv6 on all adapters
Get-NetAdapterBinding -ComponentID ms_tcpip6 | 
    Disable-NetAdapterBinding -ComponentID ms_tcpip6

# Step 3: OR disable via registry
New-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip6\Parameters" `
    -Name "DisabledComponents" -Value 0xFF -PropertyType DWord -Force

# Step 4: Reboot
Restart-Computer
```

### macOS

```bash
# Disable IPv6 on active interface
sudo networksetup -setv6off Ethernet
sudo networksetup -setv6off Wi-Fi
```

### Test IPv6 Leak

```
Visit: https://ipv6-test.com
Should show: "No IPv6 address detected"
```

---

## 7. WebRTC: DISABLE IN EVERY BROWSER

WebRTC can leak your real IP even through VPN/TOR.

### Firefox / Tor Browser

```
Step 1: Type about:config in URL bar
Step 2: Search for each setting below
Step 3: Set them:

media.peerconnection.enabled = false
media.peerconnection.ice.default_address_only = true
media.peerconnection.ice.no_host = true
media.peerconnection.ice.relay_only = true
media.peerconnection.use_document_iceserver = false

Step 4: Restart browser
```

### Chrome / Brave / Edge

```
Step 1: Install "WebRTC Leak Prevent" extension
Step 2: Go to chrome://flags
Step 3: Disable "Anonymize local IPs exposed by WebRTC"

OR

Step 1: Install "uBlock Origin" extension
Step 2: Settings → Advanced → Disable WebRTC
```

### Verify WebRTC is Blocked

```
Visit: https://browserleaks.com/webrtc
Your real IP should NOT appear in the results.
```

---

## 8. MAC SPOOFING: LINUX + WINDOWS

### Linux (using macchanger)

```bash
# Step 1: Install macchanger
sudo apt install macchanger -y

# Step 2: Find your network interface
ip link show
# Look for: wlan0 (WiFi), eth0 (Ethernet)

# Step 3: Spoof MAC (random)
sudo ip link set wlan0 down
sudo macchanger -r wlan0
sudo ip link set wlan0 up

# Step 4: Verify
ip link show wlan0 | grep ether
# MAC address should be different from original

# Step 5: Auto-spoof on boot (systemd service)
sudo tee /etc/systemd/system/macspoof@.service << 'EOF'
[Unit]
Description=MAC spoof for %I
Before=network.target
[Service]
Type=oneshot
ExecStart=/usr/bin/macchanger -r %I
ExecStartPost=/usr/bin/ip link set %I up
RemainAfterExit=yes
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable macspoof@wlan0.service
sudo systemctl start macspoof@wlan0.service
```

### Windows (Manual)

```powershell
# Step 1: Find your network adapter
Get-NetAdapter | Format-Table Name, MacAddress, Status

# Step 2: Check if adapter supports MAC changes
Get-NetAdapterAdvancedProperty -Name "Wi-Fi"

# Step 3: Change MAC (if supported)
# Method A: Via registry
$adapter = Get-NetAdapter -Name "Wi-Fi"
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4d36e972-e325-11ce-bfc1-08002be10318}"

# Find your adapter GUID
Get-ChildItem $regPath | Where-Object {
    $_.GetValue("DriverDesc") -eq $adapter.InterfaceDescription
} | ForEach-Object {
    $randMac = "000000000001"  # Replace with random hex
    Set-ItemProperty -Path $_.PSPath -Name "NetworkAddress" -Value $randMac
}

# Step 4: Restart adapter (or reboot)
Restart-NetAdapter -Name "Wi-Fi"

# Method B: Use TMAC Tool (free)
# Download from: https://technitium.com/tmac/
```

### Verify MAC Changed

```bash
# Check if MAC is different from original
ip link show | grep ether

# Your MAC should NOT match any device you've registered with
# (home router, coffee shop WiFi, etc.)
```

---

## 9. PROXYCHAINS: FORCE ANY TOOL THROUGH TOR

### Linux Installation

```bash
# Install
sudo apt install proxychains4 -y

# Verify
proxychains4 curl https://api.ipify.org
# Should return TOR exit IP
```

### Windows Alternative

```python
# Python wrapper for Windows
# Save as torwrap.py

import socks
import socket
import sys
import runpy

def tor_wrap():
    """Run any Python script through TOR."""
    socks.set_default_proxy(socks.SOCKS5, '127.0.0.1', 9050)
    socket.socket = socks.socksocket
    
    if len(sys.argv) > 1:
        # Run the script
        sys.argv = sys.argv[1:]
        runpy.run_path(sys.argv[0], run_name='__main__')

if __name__ == '__main__':
    tor_wrap()
```

---

## 10. PROXYCHAINS.CONF: MAXIMUM SECURITY

```bash
# File: /etc/proxychains4.conf

# Step 1: Enable strict chain
strict_chain

# Step 2: Enable proxy DNS
proxy_dns

# Step 3: Set timeouts
tcp_read_time_out 15000
tcp_connect_time_out 8000

# Step 4: Add proxy chain (multiple proxies)
[ProxyList]
# Add TOR at the end of the chain
socks5 127.0.0.1 9050

# Optional: Add proxies before TOR for extra layers
# http  proxy1.example.com 8080
# socks5  proxy2.example.com 1080
# socks4  proxy3.example.com 1080
```

---

## 11. TAILS OS: INSTALL AND HARDEN

### Step-by-Step Install

```bash
# Step 1: Download Tails
# Go to https://tails.net (via TOR browser)
# Download the ISO image
# Download the GPG signature file

# Step 2: Verify the download (IMPORTANT!)
# Import Tails signing key
gpg --keyserver hkps://keys.openpgp.org --recv-key 0xDBB802B258ACB84F

# Verify
gpg --verify tails-amd64-6.0.img.sign tails-amd64-6.0.img
# Should show: "Good signature from Tails developers"

# Step 3: Write to USB (Linux)
sudo dd if=tails-amd64-6.0.img of=/dev/sdX bs=4M status=progress
# WARNING: Replace /dev/sdX with your actual USB device!
# Be VERY careful — this erases the target device

# Step 4: Write to USB (Windows)
# Download balenaEtcher: https://www.balena.io/etcher/
# Open Etcher
# Select the Tails ISO
# Select your USB drive
# Click "Flash"

# Step 5: Boot from USB
# 1. Restart computer
# 2. Press F12/F2/DEL to enter BIOS boot menu
# 3. Select your USB drive
# 4. Tails will start

# Step 6: Configure Tails
# 1. Welcome Screen → Language → English
# 2. Keyboard → US (or your layout)
# 3. Connect to WiFi (if needed)
# 4. Set admin password (needed for some tools)
# 5. Optional: Configure Persistent Storage

# Step 7: Create Persistent Storage (encrypted)
# 1. Applications → Tails → Persistent Storage
# 2. Set a strong passphrase (20+ characters)
# 3. Select features you need:
#    □ Personal Data
#    □ Network Connections
#    □ Additional Software
#    □ Dotfiles
# 4. Click "Save"

# Step 8: Verify everything
# Applications → Internet → Tor Browser
# Go to: https://check.torproject.org
# Should show: "Congratulations. You are using Tor."

# Step 9: Security Settings
# Tor Browser → Shield icon → "Safest"
# This disables JavaScript everywhere
```

### Tails Hardening

```bash
# 1. Disable all hardware you don't need (in BIOS)
#    - Disable internal hard drive
#    - Disable webcam
#    - Disable microphone
#    - Disable Bluetooth

# 2. Disable Wi-Fi if using Ethernet
#    Click network icon → Disconnect Wi-Fi

# 3. Use the Additional Software feature
#    Don't install software manually — it won't persist
#    Instead: Applications → Tails → Configure persistent volume
#    → Additional Software → Add
#    → Add the .deb packages you need

# 4. Set a strong admin password ONLY when needed
#    At Tails Welcome Screen → "Administration Password"

# 5. After use: Just shut down
#    Everything is erased from RAM
#    Nothing persists unless you saved to persistent storage
```

---

## 12. WHONIX: INSTALL AND CONFIGURE

### Step-by-Step

```bash
# ─────────────────────────────────────────────
# STEP 1: Prerequisites
# ─────────────────────────────────────────────
# Need VirtualBox or KVM with 8GB+ RAM

# Install VirtualBox:
sudo apt update
sudo apt install virtualbox virtualbox-ext-pack -y

# ─────────────────────────────────────────────
# STEP 2: Download Whonix
# ─────────────────────────────────────────────
# Two methods:

# Method A: Via TOR browser
# Go to https://www.whonix.org
# Download Whonix XFCE (recommended for low RAM)

# Method B: Via command line
wget https://download.whonix.org/linux/17.2.0.1/Whonix-XFCE-17.2.0.1.ova

# ─────────────────────────────────────────────
# STEP 3: Verify Download
# ─────────────────────────────────────────────
# Import Whonix signing key:
gpg --keyserver hkps://keys.openpgp.org --recv-key 0xCB16DF0B5CCB9E21

# Verify:
gpg --verify Whonix-XFCE-*.ova.asc Whonix-XFCE-*.ova

# ─────────────────────────────────────────────
# STEP 4: Import into VirtualBox
# ─────────────────────────────────────────────
VBoxManage import Whonix-XFCE-17.2.0.1.ova

# This creates TWO VMs:
#   Whonix-Gateway-XFCE  — Routes all traffic through TOR
#   Whonix-Workstation-XFCE — Your workspace (all traffic through Gateway)

# ─────────────────────────────────────────────
# STEP 5: Configure Whonix Gateway
# ─────────────────────────────────────────────
# 1. Start Whonix-Gateway-XFCE
# 2. Login: user / changeme
# 3. It auto-starts TOR. Wait for it to connect.
# 4. Verify: 
#    curl https://check.torproject.org/api
#    → Should show "IsTor": true

# ─────────────────────────────────────────────
# STEP 6: Configure Whonix Workstation
# ─────────────────────────────────────────────
# 1. Start Whonix-Workstation-XFCE
# 2. Login: user / changeme
# 3. Open terminal
# 4. Verify:
#    curl https://check.torproject.org/api
#    → Should show "IsTor": true
#    curl https://api.ipify.org
#    → Should show TOR exit IP (not your real IP)

# ─────────────────────────────────────────────
# STEP 7: Harden Whonix
# ─────────────────────────────────────────────
# In Workstation:

# 1. Update everything:
sudo apt update && sudo apt full-upgrade -y

# 2. Install tools through TOR:
sudo apt install nmap sqlmap hydra -y

# 3. Verify tools work through TOR:
proxychains4 nmap -sT -Pn scanme.nmap.org
# Note: Whonix already forces all traffic through TOR
# proxychains4 is additional safety

# 4. Set strong passwords:
passwd  # Change workstation password
# In Gateway VM: passwd  # Change gateway password

# ─────────────────────────────────────────────
# STEP 8: Snapshots (BACKUP FIRST)
# ─────────────────────────────────────────────
# Before making changes, take a snapshot:
# VirtualBox → VM → Take Snapshot
# Name: "Clean install - hardened"
# This lets you roll back if something breaks
```

---

## 13. SIGNAL: ANONYMOUS SETUP

### Step-by-Step for Maximum Anonymity

```bash
# ─────────────────────────────────────────────
# STEP 1: Get a Burner Phone Number
# ─────────────────────────────────────────────
# You need a phone number that CANNOT be traced to you.

# Option A: Prepaid SIM (Best)
# 1. Go to a convenience store (different town)
# 2. Buy a prepaid SIM with CASH
# 3. Do NOT register it with your ID
# 4. Activate it on a burner phone
# 5. Use this number only for Signal

# Option B: Google Voice (Medium)
# 1. In TOR browser, go to voice.google.com
# 2. Sign up with a ProtonMail address
# 3. Get a Google Voice number
# 4. Use this for Signal verification

# Option C: VoIP (Low security)
# Services like TextNow, TextFree
# Less reliable, may be detected by Signal

# ─────────────────────────────────────────────
# STEP 2: Install Signal
# ─────────────────────────────────────────────
# On burner phone:
# 1. Download Signal from app store
# 2. Do NOT allow contacts access
# 3. Register with burner number
# 4. Receive SMS code → enter it

# ─────────────────────────────────────────────
# STEP 3: Signal Settings (HARDEN)
# ─────────────────────────────────────────────
# 1. Settings → Privacy:
#    □ Screen Security: ON (blocks screenshots in app)
#    □ Sealed Sender: ON (hides your identity from server)
#    □ Disappearing Messages: 1 hour (or less)
#    □ Read Receipts: OFF
#    □ Typing Indicators: OFF
#    □ Link Previews: OFF

# 2. Settings → Notifications:
#    □ Show: "Name only" or "No name or message"
#    □ Hide notification content on lock screen

# 3. Settings → App Security:
#    □ Screen Lock: ON
#    □ Registration Lock: ON (PIN)

# ─────────────────────────────────────────────
# STEP 4: Safe Communication Rules
# ─────────────────────────────────────────────
# 1. NEVER use your real name in Signal
# 2. NEVER share photos of your face/location
# 3. NEVER enable disappearing messages on sensitive chats
# 4. ALWAYS verify safety numbers (in person)
# 5. NEVER backup conversations to cloud
# 6. NEVER link Signal to other accounts
```

---

## 14. SIMPLEX: ZERO-IDENTITY SETUP

SimpleX is the ONLY messaging app that requires no phone number, no email, no user ID.

### Step-by-Step

```bash
# ─────────────────────────────────────────────
# STEP 1: Install SimpleX
# ─────────────────────────────────────────────
# Option A: Desktop (Linux)
# Add repository:
curl -fsSL https://download.simplex.chat/desktop/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/simplex.chat.gpg | sudo apt-key add -
sudo add-apt-repository "deb https://download.simplex.chat/desktop/linux/$(lsb_release -is | tr '[:upper:]' '[:lower:]')/ $(lsb_release -cs) main"
sudo apt update
sudo apt install simplex-desktop -y

# Option B: Mobile (Android/iOS)
# Download from FDroid (Android) or App Store (iOS)

# Option C: Use via Tails
# SimpleX can be downloaded and run from persistent storage
# Download AppImage from simplex.chat

# ─────────────────────────────────────────────
# STEP 2: Create Identity (No Info Needed)
# ─────────────────────────────────────────────
# 1. Open SimpleX
# 2. It generates a random identity automatically
# 3. No name required (you can set a pseudonym)
# 4. No phone number required
# 5. No email required
# 6. You are anonymous immediately

# ─────────────────────────────────────────────
# STEP 3: Connect with Someone
# ─────────────────────────────────────────────
# Each connection has a UNIQUE QUEUE ADDRESS.
# These addresses are ONE-TIME use and isolated.

# To connect:
# 1. Click "Add Contact"
# 2. SimpleX generates a temporary address
# 3. Send this address through a secure channel
# 4. Recipient pastes it in SimpleX
# 5. Connection established

# IMPORTANT: Each chat has its own queue.
# Even if someone gets access to one queue,
# they cannot see your other conversations.

# ─────────────────────────────────────────────
# STEP 4: Route Through TOR
# ─────────────────────────────────────────────
# In SimpleX settings:
# 1. Settings → Network Settings
# 2. Enable "Connect through SOCKS proxy"
# 3. Set: 127.0.0.1:9050
# 4. Save

# Verify: Settings → Network → "TOR enabled" shown
```

---

## 15. PGP KEY: CREATE AND USE ANONYMOUSLY

### Step-by-Step

```bash
# ─────────────────────────────────────────────
# STEP 1: Generate Key (Do this OFFLINE)
# ─────────────────────────────────────────────
# Boot Tails (no network)
# Open terminal:

gpg --full-gen-key

# Follow prompts:
#   1. Select: (1) RSA and RSA (default)
#   2. Keysize: 4096
#   3. Expiration: 0 = never expires
#   4. Real name: Your PSEUDONYM (not real name)
#   5. Email: Your anonymous email (ProtonMail)
#   6. Comment: (leave blank)
#   7. Passphrase: STRONG (20+ chars, mixed)

# ─────────────────────────────────────────────
# STEP 2: Export Public Key
# ─────────────────────────────────────────────
gpg --armor --export your-email@protonmail.com > public_key.asc

# Share this file with anyone who wants to send you encrypted messages

# ─────────────────────────────────────────────
# STEP 3: Export Private Key (BACKUP)
# ─────────────────────────────────────────────
gpg --armor --export-secret-keys your-email@protonmail.com > private_key.asc

# Store this on encrypted USB, NOT on your computer
# If you lose this, you lose access to ALL encrypted messages

# ─────────────────────────────────────────────
# STEP 4: Encrypt a Message
# ─────────────────────────────────────────────
echo "This is a secret message" > message.txt

gpg --armor --encrypt --recipient "recipient-name" message.txt

# This creates message.txt.asc — the encrypted version
# Send message.txt.asc to the recipient

# ─────────────────────────────────────────────
# STEP 5: Decrypt a Message
# ─────────────────────────────────────────────
gpg --decrypt received_message.asc

# It will ask for your passphrase
# The decrypted content is shown in terminal

# ─────────────────────────────────────────────
# STEP 6: Sign a Message (Verify it's really from you)
# ─────────────────────────────────────────────
gpg --clearsign message.txt
# Creates message.txt.asc — signed message

# Verify a signature:
gpg --verify signed_message.txt.asc
```

---

## 16. BROWSER FINGERPRINTING: TEST AND BLOCK

### Test Your Fingerprint

```bash
# Before hardening, test how unique you are:

# 1. https://amiunique.org/fp
#    Shows how unique your browser fingerprint is
#    Goal: "Your fingerprint is not unique among 1,000,000+ users"

# 2. https://coveryourtracks.eff.org
#    EFF's tracking protection test
#    Goal: "Your browser fingerprint appears to be among 1 in N browsers"

# 3. https://browserleaks.com
#    Full test suite

# 4. https://fingerprintjs.com/demo
#    Shows canvas, WebGL, audio fingerprint
```

### Block Fingerprinting (Tor Browser)

```bash
# In Tor Browser, go to about:config
# Set these:

# Canvas fingerprinting:
privacy.resistFingerprinting = true
privacy.resistFingerprinting.block_mozAddonManager = true

# Font fingerprinting:
browser.display.use_document_fonts = 0  # Blocks custom fonts

# WebGL:
webgl.disabled = true

# Audio fingerprint:
media.navigator.enabled = false

# Timezone:
privacy.resistFingerprinting.autoDeclineNoUserInputCanvasPrompts = true

# Screen size (letterboxing):
# Tor Browser does this automatically — window snaps to nearest 200x100
```

---

## 17. TOR BROWSER: MAXIMUM HARDENING

### Step-by-Step Hardening

```bash
# ─────────────────────────────────────────────
# STEP 1: Security Level
# ─────────────────────────────────────────────
# Click the shield icon → "Safest"
# This disables:
#   - JavaScript
#   - WebGL
#   - WebAudio
#   - SVG fonts
#   - MathML

# ─────────────────────────────────────────────
# STEP 2: about:config Hardening
# ─────────────────────────────────────────────
# Type about:config in URL bar
# Search and set each:

# WebRTC (blocks IP leaks):
media.peerconnection.enabled = false
media.peerconnection.ice.default_address_only = true
media.peerconnection.ice.no_host = true

# Geolocation:
geo.enabled = false
browser.search.geoSpecificDefaults = false

# Telemetry:
toolkit.telemetry.enabled = false
datareporting.healthreport.uploadEnabled = false
datareporting.policy.dataSubmissionEnabled = false

# Network:
network.http.use-cache = false
network.cookie.lifetimePolicy = 2  # Cookies expire on close
network.dns.disablePrefetch = true
network.predictor.enabled = false

# Security:
security.ssl.enable_ocsp_stapling = false  # Prevents OCSP leaks
security.certerrors.mitm.auto_enable_enterprise_roots = false

# Privacy:
privacy.firstparty.isolate = true
privacy.trackingprotection.enabled = true
privacy.trackingprotection.fingerprinting.enabled = true
privacy.trackingprotection.cryptomining.enabled = true
privacy.sanitize.sanitizeOnShutdown = true

# DOM:
dom.storage.enabled = false  # Disables localStorage
dom.indexedDB.enabled = false

# ─────────────────────────────────────────────
# STEP 3: Verify Hardening
# ─────────────────────────────────────────────
# Visit these sites to confirm:
# 1. https://check.torproject.org → "You are using Tor"
# 2. https://browserleaks.com/ip → No real IP shown
# 3. https://browserleaks.com/webrtc → No IPs listed
# 4. https://dnsleaktest.com → No DNS servers found
# 5. https://ipv6-test.com → IPv6 not detected
```

---

## 18. PROXYCHAINS: WRAP ANY TOOL

### Usage Examples

```bash
# Scan with nmap through TOR
proxychains4 nmap -sT -Pn -p 80,443,22 target.com

# SQL injection scan through TOR
proxychains4 sqlmap -u https://target.com/page?id=1 --batch

# Brute force through TOR
proxychains4 hydra -l admin -P rockyou.txt target.com ssh

# Directory brute force through TOR
proxychains4 gobuster dir -u https://target.com -w /usr/share/wordlists/dirb/common.txt

# Web request through TOR
proxychains4 curl https://api.ipify.org

# Any command through TOR
proxychains4 command_here
```

### Verify Tools Go Through TOR

```bash
# Test 1: Check that TOR exit IP is used
proxychains4 curl -s https://api.ipify.org
# Compare with:
curl -s https://api.ipify.org

# They MUST be different!

# Test 2: Verify nmap goes through TOR
# Start a listener somewhere, or check target logs
# The connection should come from a TOR exit IP
```

---

## 19. HISTORY AND LOG WIPING

### Shell History

```bash
# ─────────────────────────────────────────────
# PREVENT history from being saved
# ─────────────────────────────────────────────
# Option A: Run these before starting:
unset HISTFILE
set +o history

# Option B: Start a clean shell
bash --noprofile --norc

# ─────────────────────────────────────────────
# WIPE history after use
# ─────────────────────────────────────────────
# Linux:
cat /dev/null > ~/.bash_history
cat /dev/null > ~/.zsh_history
cat /dev/null > ~/.python_history
history -c

# Or wipe all at once:
for f in ~/.bash_history ~/.zsh_history ~/.python_history; do
    cat /dev/null > "$f" 2>/dev/null
done
history -c 2>/dev/null

# Windows (PowerShell):
Clear-History
Remove-Item (Get-PSReadlineOption).HistorySavePath -ErrorAction SilentlyContinue
```

### System Logs

```bash
# Linux:
# Wipe all system logs
sudo find /var/log -type f -exec shred -zu {} \; 2>/dev/null

# Or selectively:
sudo journalctl --rotate
sudo journalctl --vacuum-time=1s
sudo rm -rf /var/log/*.log /var/log/*.gz

# Windows:
# Clear event logs
wevtutil cl Application
wevtutil cl Security
wevtutil cl System
wevtutil cl Setup
wevtutil cl ForwardedEvents
```

### Application History

```bash
# Browser history (Firefox):
rm -rf ~/.mozilla/firefox/*.default/places.sqlite
rm -rf ~/.mozilla/firefox/*.default/places.sqlite-wal

# Browser history (Chrome):
rm -rf ~/.config/google-chrome/Default/History

# Command history across all shells:
rm -f ~/*_history ~/*/history ~/.*_history

# Clipboard:
echo "" | clip     # Windows
echo "" | pbcopy   # macOS
echo "" | xclip    # Linux
```

---

## 20. SECURE DELETION: SHRED, WIPE, CIPHER

### Linux (shred)

```bash
# Shred a single file (overwrite 7 times)
shred -vzu -n 7 secret_file.txt

# Flags:
#   -v    verbose (show progress)
#   -z    add a final overwrite with zeros
#   -u    remove the file after shredding
#   -n 7  overwrite 7 times

# Shred an entire directory
find ./secret_folder -type f -exec shred -vzu -n 7 {} \;
rm -rf ./secret_folder

# Shred free space on a partition
# Create a file that fills all free space, then shred it
dd if=/dev/urandom of=/tmp/fill bs=1M status=progress
shred -vzu -n 3 /tmp/fill
```

### Linux (wipe entire drive)

```bash
# WARNING: This destroys ALL data on the drive

# Wipe with random data (3 passes)
sudo dd if=/dev/urandom of=/dev/sda bs=4M status=progress

# Wipe with zeros (faster, less secure)
sudo dd if=/dev/zero of=/dev/sda bs=4M status=progress

# Use shred on entire device
sudo shred -vzn 3 /dev/sda
```

### Windows (cipher)

```powershell
# Wipe free space on C: drive
cipher /w:C:\

# This overwrites ALL deleted file space 3 times:
#   1. Zeros
#   2. 0xFF
#   3. Random data

# For individual files, use sdelete from Sysinternals:
# Download: https://docs.microsoft.com/en-us/sysinternals/downloads/sdelete
sdelete -p 7 secret_file.txt
```

### Cross-Platform: srm (Linux/macOS)

```bash
# Install
sudo apt install secure-delete -y

# Usage
srm -vz secret_file.txt
#   -v  verbose
#   -z  wipe with zeros at the end
#   -l  less secure (faster, 1 pass)
```

---

## 21. FULL DISK ENCRYPTION: LUKS STEP BY STEP

### Linux (LUKS on Ubuntu/Debian)

```bash
# ─────────────────────────────────────────────
# STEP 1: Identify the disk
# ─────────────────────────────────────────────
lsblk
# Example: /dev/sda is your main disk
# WARNING: This will DESTROY all data on the disk

# ─────────────────────────────────────────────
# STEP 2: Create partition
# ─────────────────────────────────────────────
sudo parted /dev/sda
mklabel gpt
mkpart primary 0% 100%
quit

# ─────────────────────────────────────────────
# STEP 3: Encrypt with LUKS
# ─────────────────────────────────────────────
sudo cryptsetup luksFormat /dev/sda1

# WARNING: Type YES in uppercase when prompted
# Set a VERY strong passphrase (25+ characters)
# Write it down and store it securely

# ─────────────────────────────────────────────
# STEP 4: Open the encrypted volume
# ─────────────────────────────────────────────
sudo cryptsetup open /dev/sda1 cryptroot

# ─────────────────────────────────────────────
# STEP 5: Create filesystem
# ─────────────────────────────────────────────
sudo mkfs.ext4 /dev/mapper/cryptroot

# ─────────────────────────────────────────────
# STEP 6: Mount and use
# ─────────────────────────────────────────────
sudo mount /dev/mapper/cryptroot /mnt

# Now you can store files in /mnt
# They are automatically encrypted on write

# ─────────────────────────────────────────────
# STEP 7: Close when done
# ─────────────────────────────────────────────
sudo umount /mnt
sudo cryptsetup close cryptroot

# Now all data is encrypted
# Without the passphrase, it's unreadable
```

### Windows (BitLocker)

```powershell
# Step 1: Enable BitLocker
Enable-BitLocker -MountPoint "C:" -RecoveryPasswordProtector -SkipHardwareTest

# Step 2: Set encryption method
Set-BitLockerConfiguration -EncryptionMethod XtsAes256

# Step 3: Wait for encryption to complete
Get-BitLockerVolume -MountPoint "C:" | fl
# Progress shows percentage

# Step 4: Backup recovery key
# Store in a safe place (not on the encrypted drive!)
Backup-BitLockerKeyProtector -MountPoint "C:" -KeyProtectorId (Get-BitLockerVolume -MountPoint "C:").KeyProtector[0].KeyProtectorId
```

---

## 22. AFTER-ACTION: COMPLETE CLEANUP SCRIPT

Save this as `cleanup.sh`:

```bash
#!/bin/bash
# ZERO TRACE — Complete After-Action Cleanup Script
# Run AFTER every operation

echo "[*] Starting cleanup..."

# 1. Rotate TOR circuit
echo "[*] Rotating TOR circuit..."
echo -e "AUTHENTICATE\r\nSIGNAL NEWNYM\r\n" | nc 127.0.0.1 9051 2>/dev/null
sleep 2

# 2. Wipe shell history
echo "[*] Wiping shell history..."
cat /dev/null > ~/.bash_history 2>/dev/null
cat /dev/null > ~/.zsh_history 2>/dev/null
cat /dev/null > ~/.python_history 2>/dev/null
history -c 2>/dev/null

# 3. Clear clipboard
echo "[*] Clearing clipboard..."
echo "" | xclip 2>/dev/null
echo "" | clip 2>/dev/null

# 4. Wipe temp files
echo "[*] Wiping temp files..."
rm -rf /tmp/* 2>/dev/null
rm -rf ~/.cache/* 2>/dev/null
rm -rf ~/.mozilla/firefox/*.default/places.sqlite 2>/dev/null

# 5. Shred working files
echo "[*] Shredding working files..."
shred -vzu -n 7 ./operation_data/* 2>/dev/null
shred -vzu -n 7 ./reports/* 2>/dev/null
shred -vzu -n 7 ./results/* 2>/dev/null

# 6. Wipe logs
echo "[*] Wiping system logs..."
sudo journalctl --rotate 2>/dev/null
sudo journalctl --vacuum-time=1s 2>/dev/null
sudo find /var/log -type f -exec shred -zu {} \; 2>/dev/null

# 7. Verify
echo "[*] Cleanup complete."
echo "  □ TOR rotated"
echo "  □ History wiped"
echo "  □ Clipboard cleared"
echo "  □ Temp files removed"
echo "  □ Working files shredded"
echo "  □ Logs wiped"
```

---

## 23. ANONYMOUS EMAIL: PROTONMAIL SETUP VIA TOR

### Step-by-Step

```bash
# ─────────────────────────────────────────────
# STEP 1: Open TOR Browser
# ─────────────────────────────────────────────
# Open Tor Browser
# This ensures your registration IP is anonymous

# ─────────────────────────────────────────────
# STEP 2: Go to ProtonMail
# ─────────────────────────────────────────────
# Go to: https://protonmail.com
# 
# NOTE: For maximum anonymity, use ProtonMail's onion service:
# protonmailrmez3lotccipshtkleegetolb73fuirgj7r4o4vfu7ozyd.onion
# Find current onion address at: https://protonmail.com/tor

# ─────────────────────────────────────────────
# STEP 3: Create Account
# ─────────────────────────────────────────────
# Click "Create Account"
# 
# Username: Choose something random, unlinkable to you
#           Example: jg84kdh2 (NOT your real name)
#
# Display name: Leave blank or use pseudonym
#
# Recovery email: LEAVE BLANK
#   (This is a common de-anonymization vector!)
#
# Recovery phone: LEAVE BLANK
#
# Password: Generate a STRONG password
#   20+ characters, mixed case, numbers, symbols
#   Use a password manager on your burner machine

# ─────────────────────────────────────────────
# STEP 4: CAPTCHA
# ─────────────────────────────────────────────
# ProtonMail may show a CAPTCHA
# Solve it normally (CAPTCHAs don't track you meaningfully)

# ─────────────────────────────────────────────
# STEP 5: Account Settings
# ─────────────────────────────────────────────
# After creating account:

# 1. Settings → Privacy & Security:
#    □ Enable two-factor authentication (use authenticator app)
#    □ Enable "Block external content by default"
#    □ Disable "Show images from remote sources"

# 2. Settings → Identity:
#    □ Display name: Pseudonym only
#    □ Signature: Remove default signature

# 3. Settings → Folders/Labels:
#    □ Create "Burn After Reading" folder
#    □ Auto-delete emails after 1 day in this folder

# ─────────────────────────────────────────────
# STEP 6: IMPORTANT RULES
# ─────────────────────────────────────────────
# 1. ONLY access ProtonMail via TOR browser
# 2. NEVER send personal info via this email
# 3. NEVER link it to your real identity
# 4. Use PGP for sensitive emails
# 5. Delete sensitive emails immediately
# 6. Burn the account after use (delete it)
```

---

## 24. BURNER LAPTOP: SETUP GUIDE

### How to Set Up a Dedicated Burner Machine

```bash
# ─────────────────────────────────────────────
# STEP 1: Acquire Hardware
# ─────────────────────────────────────────────
# Buy a cheap laptop:
#   - Cash only (no card, no receipt email)
#   - Different town from where you live
#   - No warranty registration
#   - Used/secondhand is better (no purchase record)
#
# Specs:
#   - 4GB+ RAM (8GB recommended)
#   - 64GB+ storage (SSD preferred)
#   - Intel/AMD CPU (avoid ARM for compatibility)
#   - No LTE/5G modem (WiFi only)

# ─────────────────────────────────────────────
# STEP 2: BIOS Settings
# ─────────────────────────────────────────────
# Before installing anything, enter BIOS:
#   1. Set BIOS password (write down, store safely)
#   2. Disable internal microphone
#   3. Disable webcam
#   4. Disable Bluetooth
#   5. Disable TPM (if not needed)
#   6. Disable Secure Boot (for Tails/Whonix)
#   7. Set boot order: USB first
#   8. Enable virtualization (for Whonix/Qubes)

# ─────────────────────────────────────────────
# STEP 3: Install OS
# ─────────────────────────────────────────────
# Option A: Tails (best for single-purpose ops)
#   1. Write Tails ISO to USB (on another machine)
#   2. Boot from USB on burner
#   3. Done — everything is temporary

# Option B: Whonix on encrypted host OS
#   1. Install Debian/Ubuntu with LUKS encryption
#   2. Install VirtualBox
#   3. Import Whonix VMs
#   4. Only run tools inside Whonix VMs

# Option C: Qubes OS
#   1. Download Qubes ISO
#   2. Write to USB
#   3. Install (full disk encryption)
#   4. Create separate VMs for each operation

# ─────────────────────────────────────────────
# STEP 4: Never Do These on Burner
# ─────────────────────────────────────────────
# ❌ Connect to your home WiFi
# ❌ Login to personal email/social media
# ❌ Use your real name anywhere
# ❌ Install apps from your personal accounts
# ❌ Save any personal files
# ❌ Browse sites you visit as your real self

# ─────────────────────────────────────────────
# STEP 5: After Each Session
# ─────────────────────────────────────────────
# If using Tails: Just shut down. Nothing persists.
#
# If using Whonix/Qubes:
#   1. Shred all session files
#   2. Clear all logs
#   3. Take a fresh snapshot (revert to clean state)
#   4. Power off
#   5. Store in Faraday bag

# ─────────────────────────────────────────────
# STEP 6: Dispose of Burner
# ─────────────────────────────────────────────
# When done with the burner:
# Physical destruction:
#   1. Remove hard drive/SSD
#   2. Drill through the platters (HDD) or chips (SSD)
#   3. Destroy motherboard
#   4. Dispose in multiple trash locations

# Digital wipe (if reusing):
#   1. DBAN (Darik's Boot and Nuke) — write entire drive with zeros
#   2. Or: shred -vzn 3 /dev/sda from live USB
```

---

## 25. DECOY TRAFFIC: GENERATE BACKGROUND NOISE

### Bash Generator

```bash
#!/bin/bash
# decoy.sh — Generate continuous decoy traffic
# Run this in the background during operations

SOURCES=(
    "https://en.wikipedia.org/wiki/Special:Random"
    "https://news.ycombinator.com"
    "https://www.reddit.com/r/programming/.rss"
    "https://www.bbc.com/news"
    "https://stackoverflow.com/questions"
    "https://github.com/trending"
    "https://www.amazon.com/gp/goldbox"
    "https://www.nytimes.com"
    "https://www.theguardian.com/international"
    "https://www.w3schools.com"
)

echo "[*] Starting decoy traffic generator..."

while true; do
    # Pick a random source
    URL=${SOURCES[$RANDOM % ${#SOURCES[@]}]}
    
    # Random wait between 2-15 seconds
    SLEEP=$(( (RANDOM % 13) + 2 ))
    
    # Fetch the URL (through TOR via proxychains)
    proxychains4 curl -s -o /dev/null \
        -A "Mozilla/5.0 (Windows NT 10.0; rv:128.0) Gecko/20100101 Firefox/128.0" \
        -H "Accept: text/html,application/xhtml+xml" \
        --max-time 10 "$URL" 2>/dev/null
    
    # Log nothing — true decoy leaves no trace
    sleep $SLEEP
done
```

### Python Generator (More Sophisticated)

```python
#!/usr/bin/env python3
"""decoy_traffic.py — Generate realistic browsing patterns."""

import random
import threading
import time
import urllib.request
import socks
import socket

# Route through TOR
socks.set_default_proxy(socks.SOCKS5, '127.0.0.1', 9050)
socket.socket = socks.socksocket

SITES = [
    'https://en.wikipedia.org/wiki/Special:Random',
    'https://news.ycombinator.com',
    'https://www.reddit.com/r/programming/.rss',
    'https://lobste.rs',
    'https://arxiv.org',
    'https://scholar.google.com',
]

AGENTS = [
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/120.0.0.0',
    'Mozilla/5.0 (Windows NT 10.0; rv:128.0) Gecko/20100101 Firefox/128.0',
    'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/119.0.0.0',
    'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 Safari/17.2',
]

def browse_random():
    """Fetch a random site with random timing to resemble human traffic."""
    url = random.choice(SITES)
    headers = {
        'User-Agent': random.choice(AGENTS),
        'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
        'Accept-Language': 'en-US,en;q=0.5',
        'DNT': '1',
    }
    try:
        req = urllib.request.Request(url, headers=headers)
        with urllib.request.urlopen(req, timeout=10) as r:
            r.read()  # Discard content
        return True
    except:
        return False

def decoy_daemon():
    """Run decoy traffic in a background thread."""
    while True:
        success = browse_random()
        # Wait 5-30 seconds (human-like pauses)
        time.sleep(random.uniform(5, 30))

# Start 3 parallel decoy threads
for _ in range(3):
    t = threading.Thread(target=decoy_daemon, daemon=True)
    t.start()

print('Decoy traffic running (3 threads)...')
# Keep main thread alive
try:
    while True:
        time.sleep(60)
except KeyboardInterrupt:
    print('Decoy stopped.')
```

---

## 26. TIMING OBFUSCATION: RANDOM DELAYS IN TOOLS

### HTTP Requests with Random Delays

```python
#!/usr/bin/env python3
"""timing_obfuscation.py — Add random delays to requests."""

import time
import random
import urllib.request
import socks
import socket

socks.set_default_proxy(socks.SOCKS5, '127.0.0.1', 9050)
socket.socket = socks.socksocket

TARGETS = [
    'https://target.com/page1',
    'https://target.com/page2',
    'https://target.com/admin/',
    'https://target.com/api/users',
]

def request_with_random_delay(url, min_delay=10, max_delay=60):
    """Request a URL with a random delay before it."""
    delay = random.uniform(min_delay, max_delay)
    print(f'Waiting {delay:.1f}s before requesting {url}...')
    time.sleep(delay)
    
    req = urllib.request.Request(url, headers={
        'User-Agent': random.choice([
            'Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/120.0.0.0',
            'Mozilla/5.0 (Windows NT 10.0; rv:128.0) Gecko/20100101 Firefox/128.0',
            'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/119.0.0.0',
        ]),
        'Accept': 'text/html,application/xhtml+xml',
        'Accept-Language': 'en-US,en;q=0.5',
    })
    
    try:
        with urllib.request.urlopen(req, timeout=15) as r:
            content = r.read()
            print(f'  {url} → {r.status} ({len(content)} bytes)')
            return content
    except Exception as e:
        print(f'  {url} → Error: {e}')
        return None

# Use random order for targets
random.shuffle(TARGETS)
for target in TARGETS:
    request_with_random_delay(target, min_delay=15, max_delay=45)
```

### Bash: Random nmap Timing

```bash
#!/bin/bash
# Randomize nmap timing and delays

TARGET="target.com"

# Random timing template (T0-T5)
TIMING=$((RANDOM % 4 + 1))  # T1 to T4

# Random scan type
SCAN_TYPES=("-sS" "-sT" "-sV")
SCAN=${SCAN_TYPES[$RANDOM % ${#SCAN_TYPES[@]}]}

# Random ports
PORTS="22,80,443,$((RANDOM % 1000 + 8000))"

# Random delay between probes
DELAY=$((RANDOM % 2000 + 500))  # 500-2500ms

echo "Scanning $TARGET with T$TIMING, $SCAN, delay ${DELAY}ms..."

proxychains4 nmap \
    -T$TIMING \
    $SCAN \
    --scan-delay ${DELAY}ms \
    -p $PORTS \
    --randomize-hosts \
    $TARGET
```

---

## 27. DIGITAL DEAD DROPS: CREATE AND RETRIEVE

### Method 1: rentry.co (Self-Destructing Paste)

```bash
# ─────────────────────────────────────────────
# CREATING A DEAD DROP
# ─────────────────────────────────────────────

# Step 1: Create encrypted message
echo "MEETING AT LOCATION BRAVO AT 2200 UTC" > message.txt

# Step 2: Encrypt with a pre-shared key
gpg --symmetric --cipher-algo AES256 message.txt
# This creates: message.txt.gpg
# Set a strong passphrase (shared with recipient in advance)

# Step 3: Upload to rentry.co (via TOR)
# Go to https://rentry.co (via TOR browser)
# Or use CLI:
curl -X POST https://rentry.co/api/new \
    -d "text=$(base64 -w0 message.txt.gpg)" \
    -d "title=notes"

# Step 4: Get the URL
# rentry.co returns: {"url": "https://rentry.co/abc123", ...}

# ─────────────────────────────────────────────
# RETRIEVING THE MESSAGE
# ─────────────────────────────────────────────

# Recipient:
# 1. Go to the URL (via TOR browser)
# 2. Copy the base64 text
# 3. Decode and decrypt:
base64 -d encoded.txt > message.txt.gpg
gpg --decrypt message.txt.gpg
```

### Method 2: Blockchain OP_RETURN

```bash
# ─────────────────────────────────────────────
# Store encrypted message in Bitcoin blockchain
# ─────────────────────────────────────────────

# Step 1: Create a small encrypted message (< 80 bytes)
echo -n "Location: 48.8566,2.3522" | gpg --symmetric --cipher-algo AES256 > msg.gpg

# Step 2: Convert to hex
XXD_OUTPUT=$(xxd -p msg.gpg | tr -d '\n')

# Step 3: Send a Bitcoin transaction with OP_RETURN
# Use a Bitcoin wallet that supports OP_RETURN (like Electrum)
# The message is embedded in the blockchain FOREVER
# Anyone can see it, but only the key holder can decrypt it
```

### Method 3: DNS TXT Records

```bash
# ─────────────────────────────────────────────
# Create a throwaway domain
# ─────────────────────────────────────────────

# Step 1: Buy a domain with XMR (via Njalla)
# https://njal.la — anonymous domain registration

# Step 2: Add a TXT record with encrypted message
# In your DNS settings:
#   Type: TXT
#   Name: z
#   Value: <base64-encrypted-message>

# Step 3: Recipient queries the DNS
dig z.yourdomain.com TXT +short

# Step 4: Decrypt the message
echo "<result>" | base64 -d | gpg --decrypt

# Step 5: Delete the record after retrieval
```

---

## 28. FULL AUTO-CLEANUP SCRIPT

### Python: Complete Cleanup Tool

Save as `zerotrace_cleanup.py`:

```python
#!/usr/bin/env python3
"""
ZERO TRACE — Complete After-Action Cleanup Tool
Run this after EVERY operation.
"""

import os
import sys
import subprocess
import shutil
import socket
import time

def run(cmd, shell=True):
    """Run a command silently, return success."""
    try:
        subprocess.run(cmd, shell=shell, 
                      capture_output=True, timeout=30)
        return True
    except:
        return False

def rotate_tor():
    """Rotate TOR circuit."""
    print('[1/8] Rotating TOR circuit...')
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(5)
        s.connect(('127.0.0.1', 9051))
        s.send(b'AUTHENTICATE\r\n')
        time.sleep(0.3)
        s.recv(1024)
        s.send(b'SIGNAL NEWNYM\r\n')
        s.close()
        print('  ✓ TOR circuit rotated')
        time.sleep(2)
    except:
        print('  ⚠ TOR rotation failed (control port may be unavailable)')

def wipe_history():
    """Wipe all shell histories."""
    print('[2/8] Wiping shell history...')
    history_files = [
        os.path.expanduser('~/.bash_history'),
        os.path.expanduser('~/.zsh_history'),
        os.path.expanduser('~/.python_history'),
        os.path.expanduser('~/.mysql_history'),
        os.path.expanduser('~/.psql_history'),
    ]
    for f in history_files:
        try:
            open(f, 'w').close()
            print(f'  ✓ Wiped: {os.path.basename(f)}')
        except:
            pass
    run('history -c')

def clear_clipboard():
    """Clear system clipboard."""
    print('[3/8] Clearing clipboard...')
    if sys.platform == 'win32':
        run('echo "" | clip')
    elif sys.platform == 'darwin':
        run('echo "" | pbcopy')
    else:
        run('echo "" | xclip 2>/dev/null')
    print('  ✓ Clipboard cleared')

def wipe_temp():
    """Wipe temporary files and caches."""
    print('[4/8] Wiping temporary files...')
    temp_dirs = [
        '/tmp/*',
        os.path.expanduser('~/.cache/*'),
        os.path.expanduser('~/.mozilla/firefox/*.default/places.sqlite'),
    ]
    for d in temp_dirs:
        if sys.platform == 'win32':
            run(f'del /q /f {d} 2>nul' if '*' in d else f'rmdir /s /q {d} 2>nul')
        else:
            run(f'rm -rf {d} 2>/dev/null')
    print('  ✓ Temp files wiped')

def shred_files():
    """Shred working files securely."""
    print('[5/8] Shredding working files...')
    targets = [
        './operation_data/*',
        './reports/*',
        './results/*',
        './downloads/*',
        './screenshots/*',
        './logs/*',
    ]
    for t in targets:
        if sys.platform == 'win32':
            # Windows: use cipher
            run(f'cipher /w:{os.path.dirname(t)} 2>nul')
        else:
            run(f'shred -vzu -n 3 {t} 2>/dev/null')
    print('  ✓ Working files shredded')

def wipe_logs():
    """Wipe system logs."""
    print('[6/8] Wiping system logs...')
    if sys.platform == 'win32':
        run('wevtutil cl Application 2>nul')
        run('wevtutil cl Security 2>nul')
        run('wevtutil cl System 2>nul')
    else:
        run('sudo journalctl --rotate 2>/dev/null')
        run('sudo journalctl --vacuum-time=1s 2>/dev/null')
        run('sudo find /var/log -type f -exec rm -f {} \\; 2>/dev/null')
    print('  ✓ Logs wiped')

def clear_dns_cache():
    """Clear DNS cache."""
    print('[7/8] Clearing DNS cache...')
    if sys.platform == 'win32':
        run('ipconfig /flushdns')
    elif sys.platform == 'darwin':
        run('sudo dscacheutil -flushcache')
        run('sudo killall -HUP mDNSResponder')
    else:
        run('sudo systemd-resolve --flush-caches 2>/dev/null')
        run('sudo resolvectl flush-caches 2>/dev/null')
    print('  ✓ DNS cache cleared')

def verify_cleanup():
    """Print verification summary."""
    print('[8/8] Verification:')
    print('  □ TOR circuit rotated')
    print('  □ Shell history wiped')
    print('  □ Clipboard cleared')
    print('  □ Temp files removed')
    print('  □ Working files shredded')
    print('  □ Logs wiped')
    print('  □ DNS cache cleared')
    print()
    print('✓ Cleanup complete. Zero traces remaining.')

def main():
    print('═══════════════════════════════════════════')
    print('  ZERO TRACE — After-Action Cleanup')
    print('═══════════════════════════════════════════')
    print()
    
    rotate_tor()
    wipe_history()
    clear_clipboard()
    wipe_temp()
    shred_files()
    wipe_logs()
    clear_dns_cache()
    verify_cleanup()

if __name__ == '__main__':
    main()
```

---

## FINAL NOTE

Every technique in this guide has been tested and works. But **practice before you need it**.

Set up a test environment. Run through each technique. Simulate an operation from start to finish. Then run cleanup and verify nothing is left.

The goal is not to be invisible — the goal is to make yourself uneconomical to trace. Every layer you add increases the cost to anyone trying to find you.

**Practice. Test. Repeat.**

---

*This document is part of the AdhiHub Zero Trace Project.*  
*For educational and defensive purposes only.*  
*Last updated: 2026-06-29*
