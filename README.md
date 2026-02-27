# Minimal-Debian-Trixie-CLI-Install-Server
This is a minimal Debian CLI server meant to run in an enviroment with limited hardware like an old laptop or quite likely even a rasberry pi (3 and above) if it has sufficient RAM

Debian Trixie (Testing) Server Setup
​This documentation covers the configuration of a minimal Debian Trixie environment acting as a secure gateway for remote development and networking.

System Overview
​- OS: Debian 13 (Trixie) - Testing Branch.
​- Architecture: Minimal CLI / Headless.
- ​Network Role: Twingate Connector & Docker VPN Host.
​- Hardware: Laptop + Lenovo Docking Station.

​# 🛠️ Phase 1: Trixie-Specific Hardware Detection
​- Trixie uses newer versions of kmod and the kernel. Minimal installs still require manual firmware injection for proprietary dock chips.
- In this case particularly i needed to extend the number of ports but due to the hardware needing specific drivers and available ones were either unstable or non existent the display ports do not work(at least in this build) but it still works fine for tasks like data transfer and Poe(power over ethernet)

​# 1. Essential Tooling (Trixie Repos)
The "Dock Detection" Script (check_dock.sh)

- #!/bin/bash
- echo "--- Hardware Bus Check ---"
- lsusb | grep -i "Lenovo" || echo "USB Dock not found."
- boltctl list | grep -i "Lenovo" || echo "Thunderbolt Dock not authorized/found."

- echo -e "\n--- Network Interface Check ---"

# OS Naming Interface Check
- Trixie often uses Predictable Network Interface Names so finding out which are the correct interfaces to modify or configure
- INTERFACE=$(ls /sys/class/net | grep -E 'enx|enp0s20')

- if [ -z "$INTERFACE" ]; then
-     echo "No Ethernet interface detected."
- else
-     STATE=$(cat /sys/class/net/$INTERFACE/operstate)
-     echo "Interface: $INTERFACE | Status: $STATE"
-    ip -4 addr show $INTERFACE | grep "inet"
fi

# Phase 2: Security & Remote Access
​- Because Trixie is a "testing" branch, security packages update frequently. We use a "Jump Host" configuration

- ​1. SSH Hardening (/etc/ssh/sshd_config)
- ​Port: 2222
- ​Protocol: 2
- ​PermitRootLogin: no

# ​2. UFW Configuration

- sudo ufw allow 2222/tcp  # Custom SSH or any port you want or can open bt beware of already occupied ports that are reserved for certain services
- sudo ufw allow 51820/udp # WireGuard VPN, this just the port they use. Double check incase they change it, or alternatively you can use OpenVPN
- sudo ufw enable
- sudo apt update
- sudo apt install -y usbutils pciutils firmware-realtek net-tools iproute2 bolt     

- debian trixie blocks thunderbolt by default

# Phase 3: Dockerized VPN (WireGuard)
​- In Trixie, the kernel usually has WireGuard built-in, so the container setup is highly efficient, or again you can use OpenVPN its up to you
​
- docker-compose.yml

- version: "3.8"
- services:
- wireguard:
-    image: lscr.io/linuxserver/wireguard:latest
-    container_name: wireguard
-     cap_add:
-       - NET_ADMIN
-     environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - SERVERPORT=51820
      - PEERS=1
      - ALLOWEDIPS=0.0.0.0/0
-     volumes:
      - ./config:/config
      - /lib/modules:/lib/modules # Required for Trixie kernel modules
-     ports:
      - 51820:51820/udp
-     sysctls:
      - net.ipv4.ip_forward=1
-     restart: unless-stopped

# Commands & Fixes Recap

# ISSUE                CAUSE                  FIX
- Command Not Found   - Minimal Path (/sbin) - Use sudo or add /usr/sbin to $PATH.
- Dock Not Working    - Realtek Firmware     - sudo apt install firmware-realtek
- VPN No Traffic      - IP Forwarding        - sysctl -w net.ipv4.ip_forward=1
- Windows SSH Access  - Permission Denied    - Run PowerShell as Admin and open Port 22

# This is a headsup 
- Debian Trixie is fairly new with repos changing constantly or frequently 
- This is a minimal CLI installation so a bunch of tools wont be pre-installed apon flashing the os which you need to complete some of these steps

# Sources:
​- Debian Wiki: Trixie (Testing) Release Notes
- ​Twingate Admin Guide (Connector Deployment)
​- Linux Kernel Docs: Sysfs Network Class

cinoo@cinoo-server:~$ tracepath <IP Here>
 1?: [LOCALHOST]                      pmtu 1500
 1:  Pineapple.lan                                         2.366ms
 1:  Pineapple.lan                                         2.447ms
 3:  100.69.96.1                                           4.793ms
