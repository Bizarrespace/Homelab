# Homelab
Home lab I built to learn network infrastructure — Proxmox, Linux VMs, Pi-hole DNS, backups, and notes on what went wrong, and the general process

# Overview
Single-node virtualization lab built on repurposed laptop.
Runs a type 1 hypervisor, a linux VM, and a containerized DNS server providing network-wide name resolution and filtering.

# Hardware
Host:	Dell Inspiron 5481
CPU:	Intel, VT-x enabled
RAM:	8 GB
Storage: 256 GB NVMe SSD
Network:	USB 3.0 gigabit ethernet adapter (no onboard NIC)

# Architecture
```text
Internet
   │
   └── Router (192.168.50.1)  ─ DHCP, hands out Pi-hole as primary DNS
          │
          └── Wi-Fi range extender ─ ethernet ─ USB NIC
                 │
                 └── Proxmox VE host (192.168.50.200)
                        ├── vmbr0 (Linux bridge)
                        ├── VM 100 — Ubuntu Server 24.04 LTS
                        └── CT 101 — Debian 12 / Pi-hole (192.168.50.201)
```

# Components
- Proxmox VE (Host)
Type 1 hypervisor installed on bare metal. Wanted to get closer to how virtualization works in a production environment so did not do the desktop virtualization software. 

- Ubuntu Server 24.04 LTS (VM 100)
General-purpose linux VM. Configured with VirtIO disk and netowrk drivers for performance. Accessed over SSH.
Used for practicing user and group administration, file permissions, and shell scripting.

- Pi-hole (CT 101)
DNS server for the whole network, running in an LXC (Linux container) rather than a VM. A container was the right choice here: Pi-hole is a lightweight Linux-only service, and a container shares the host kernel instead of running its own, so it uses about 512 MB instead of the 2 GB a VM usually needs.
Router distributes the Pi-hole address as primary DNS via DHCP, with a public resolver configured as a fallback so that a lab outage does not take down name resolution for the entire household.
  - Currently blocking around 8% of queries. I want to expand the adlists beyond pi-hole's default list and measure the change in block rate.
 

# Configuration notes
Proxmox uses Linux bridge (vmbro0) to give access to physical network. The bridge holds the host's IP address and must be bound to a real network interface.  


auto vmbr0
iface vmbr0 inet static
        address 192.168.50.200/24
        gateway 192.168.50.1
        bridge-ports <interface>
        bridge-stp off
        bridge-fd 0

Because the host has no onboard NIC, the installer wrote a placeholder value for bridge-ports. See "Bridge bound to a nonexistent interface" below. 


# Backup script
A bash script archives a target directory to a timestamped .tar.gz and appends a line to a log file on each run. Scheduled through cron:
0 2 * * * /root/backup.sh

Verified by deleting the source directory and restoring it from the archive.
  - Known limitation: Script logs on completion but does not check the exist status of tar, so a failed archive would still write a success line. Silent failure and crong job that never ran look the same in the log. Checking the exit code and logging failures separately is the next change.

# Problems that I worked through

# Bridge bound to a nonexistent interface
- **Symptom** Proxmox installed successfully, but the web interface was unreachable from any other machine on the network

- **Investigation**. ip a at the console showed vmbr0 holding the correct static address and marked UP, so the configuration appeared correct. However, the wireless interface was DOWN and no wired interface was listed at all. 

- **Cause** The host's wireless chipset requires firmware not included in Proxmox's Debian base, so it never came up. And with no network interface present at install time, the installer had written a placeholder name into bridge-ports. The bridge held a valid IP address while being bound to an interface that did not exist. 

- **Solution** Added a USB ethernet adapter using Realtek chipset, which is supported by kernel driver and requires no addition configuration. Brought up interface with ip link set, then updated both the iface declaration and bridge-ports to refernce the real interface name and restarted networking. 

# Container service failing with 226/NAMESPACE

- **Symptom** Pi-hole installed without error, but pihole-FTL.service would not start. systemctl status showed Active: failed, exit code 226/NAMESPACE, and the unit had exceeded systemd's start rate limit. 

- **Investigation** journalctl -u pihole-FTL gave the specific failure. "Failed to set up mount namespacing Permission denied, occurring at step NAMESPACE before the service's prestart script could execute. 

- **Actual cause** I assumed the container was unprivileged. pct config confirmed it was not. What it lacked was the nesting feature flag. In Proxmox, running a container priviledged and permitting it to create namespaces are separate settings, and namespace creation was what the error was referring to. Turning on the nesting feature flag allowed pi-hole to work properly as it was a container not a VM so it needed the feature to do kernel level things. 

# Limitations of the project so far
- Single node. No clustering, no high availability, no shared storage.
- Pi-hole filters at the domain level and cannot distinguish advertisting served from the same domain as content.
- The backup script does not detect its own failures
- Not production-grade and not presented as such
