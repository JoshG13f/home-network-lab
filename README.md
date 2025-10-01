
# 🔒 FirewallPi + NASPi: Secure Home Network & Storage Solution

## 📌 Overview

This project demonstrates how to build a **secure, VLAN-aware home network** with a Raspberry Pi firewall and a dedicated Raspberry Pi NAS, using **OpenWRT**, **Netgear GS108E VLAN switch**, **Linksys EA6100 (AP mode)**, and **OpenMediaVault (OMV 7)**.

The goal:

* Segment WAN/LAN traffic with VLANs
* Route all traffic through a **Raspberry Pi firewall**
* Provide **Wi-Fi access** via a dedicated AP in bridge mode
* Deploy a **NAS (Network Attached Storage)** with SMB/CIFS for file sharing across the LAN

---

## 🏗️ Architecture

**Hardware Used**

* Raspberry Pi 4 (FirewallPi, running OpenWRT)
* Raspberry Pi 4 (NASPi, running OpenMediaVault 7)
* Netgear GS108E Plus switch (VLAN support)
* Linksys EA6100 Wi-Fi Router (in AP/Bridge mode)
* Starlink Ethernet Adapter (WAN uplink)
* External HDD/SSD for NAS storage

**Network Layout**

* **VLAN 10 (WAN)** → Starlink Internet → FirewallPi WAN (eth0.10)
* **VLAN 20 (LAN)** → FirewallPi LAN (eth0.20) → Switch → EA6100 (Wi-Fi AP) + NASPi
* All clients get DHCP leases from FirewallPi on VLAN 20

---

## 🔧 Configuration Steps

### 1. FirewallPi (OpenWRT)

* Defined VLAN subinterfaces:

  ```sh
  config device
      option name 'eth0.10'
      option type '8021q'
      option ifname 'eth0'
      option vid '10'

  config device
      option name 'eth0.20'
      option type '8021q'
      option ifname 'eth0'
      option vid '20'
  ```

* Configured interfaces:

  ```sh
  config interface 'wan'
      option proto 'dhcp'
      option device 'eth0.10'
      option dns '1.1.1.1 8.8.8.8'

  config interface 'lan'
      option proto 'static'
      option device 'eth0.20'
      option ipaddr '192.168.20.1'
      option netmask '255.255.255.0'
  ```

* Verified internet and local routing with:

  ```sh
  ping -c3 8.8.8.8
  ping -c3 192.168.20.1
  ```

---

### 2. Netgear GS108E Switch

* VLAN 10 → Port 1 (WAN trunk to FirewallPi)
* VLAN 20 → Ports 2–8 (LAN trunk to FirewallPi, AP, NASPi, and clients)
* PVID set accordingly per port

---

### 3. Linksys EA6100 (Wi-Fi Access Point)

* Put into **Bridge Mode** (disables router features)
* Static IP set → `192.168.20.2`
* Wi-Fi SSIDs created (WPA2/3 security)
* Connected via switch on VLAN 20

---

### 4. NASPi (OpenMediaVault 7)

* Installed OMV via official script

* Configured storage mount point `/srv/dev-disk-by-uuid-xxxx/`

* Enabled **SMB/CIFS Service** for file sharing

* Created user accounts (`userNas`, `nas`) with access rights

* Shared folder:

  * Name: `shared`
  * Path: `/srv/dev-disk-by-uuid-xxxx/shared/`
  * Permissions: RW for `userNas`, RO for guests

* Verified share from client Pi:

  ```sh
  smbclient -L //192.168.20.3 -U userNas
  ```

---

## 🌐 Service Discovery (Optional)

To make NAS discoverable on Windows networks, `wsdd` was enabled via a custom **systemd unit file**:

```ini
[Unit]
Description=Web Services Dynamic Discovery host daemon (WS-Discovery)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/wsdd --shortlog -i eth0
User=nobody
Group=nogroup
Restart=always
RestartSec=2s

[Install]
WantedBy=multi-user.target
```

Enabled with:

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now wsdd
```

---

## ✅ Results

* Clients (wired + Wi-Fi) receive IPs from FirewallPi
* Internet access works seamlessly via Starlink uplink
* Local NAS is accessible at `\\naspi\` from Windows/macOS/Linux
* VLAN segmentation isolates WAN/LAN traffic securely

---

## 📸 Screenshots

* FirewallPi VLAN configs
* OMV 7 Web UI with SMB/CIFS enabled
* Successful `smbclient` test

---

## 🚀 Future Improvements

* Add Pi-hole or AdGuard for network-wide DNS filtering
* Enable WireGuard VPN for remote access
* Configure automatic NAS backups to cloud storage
