# Proxmox Homelab

Setting up a Proxmox home lab on a laptop with WiFi networking.

---

## Hardware

| Component | Details |
|-----------|---------|
| Laptop | Dell Inspiron |
| CPU | Intel Core i7-11800H @ 2.30GHz (16 threads) |
| RAM | 16GB |
| Storage | ~94GB |
| Networking | WiFi (Intel AX) + USB-C Dock (Ethernet) |

---

## What Was Accomplished

- Installed Proxmox VE 9.2.2 on a Dell Inspiron laptop
- Configured WiFi networking on a headless Proxmox server
- Diagnosed and fixed routing, DNS, and network interface issues
- Accessed the Proxmox web UI remotely from another device
- Configured the server to run with the laptop lid closed

---

## Steps Taken

### 1. Installing Proxmox
Flashed Proxmox VE 9.2.2 onto a USB drive and installed it on the Dell Inspiron.

### 2. Getting Internet Access
The laptop was connected via a USB-C dock with ethernet. The dock interface (`enx843a5bdb264b`) was brought up manually:

```bash
ip link set enx843a5bdb264b up
dhclient enx843a5bdb264b
```

### 3. Fixing Package Repositories
The Proxmox enterprise repos required a paid subscription and were causing apt to hang. They were disabled:

```bash
echo '# disabled' > /etc/apt/sources.list.d/pve-enterprise.list
echo '# disabled' > /etc/apt/sources.list.d/ceph.list
```

The Debian repo was added to sources.list:

```bash
deb http://deb.debian.org/debian bullseye main contrib non-free
```

### 4. Installing WiFi Packages

```bash
apt update && apt install wpasupplicant wireless-tools
```

### 5. Configuring WiFi
Edited `/etc/network/interfaces` to add the WiFi interface:

```
auto wlp0s20f3
iface wlp0s20f3 inet dhcp
    wpa-ssid YOUR_WIFI_NAME
    wpa-psk YOUR_WIFI_PASSWORD
```

Brought the interface up:

```bash
ifup wlp0s20f3
```

### 6. Fixing Routing
The default route was going through `vmbr0` instead of WiFi. Fixed by removing the incorrect route and adding the correct one:

```bash
ip route del 192.168.1.0/24 dev vmbr0
ip route add default via 192.168.1.1 dev wlp0s20f3
```

### 7. Accessing the Web UI
Proxmox web UI accessible at `https://<proxmox-ip>:8006` from any device on the network.

### 8. Lid Close Behavior
Configured the server to keep running with the lid closed:

```bash
nano /etc/systemd/logind.conf
# Set: HandleLidSwitch=ignore
systemctl restart systemd-logind
```

---

## Issues Faced and How They Were Fixed

| Issue | Cause | Fix |
|-------|-------|-----|
| `apt` couldn't find packages | No internet, missing repos | Brought up dock ethernet, added Debian repo |
| DNS resolution failing | No DNS server configured | Added Google DNS: `echo "nameserver 8.8.8.8" > /etc/resolv.conf` |
| Pings going to wrong interface | Default route via vmbr0 | Deleted vmbr0 route, added route via WiFi interface |
| apt hanging for minutes | Proxmox enterprise repo timing out | Disabled enterprise and ceph repos |
| Web UI unreachable from other devices | Traffic routing through disconnected vmbr0 | Removed vmbr0 subnet route |

---

## Next Steps

- [ ] Create first VM
- [ ] Set up LXC containers
- [ ] Configure persistent network settings so routes survive reboot
- [ ] Set up automated backups
- [ ] Expand storage

---

## Notes

- Proxmox on WiFi requires NAT instead of bridging for VM networking since most WiFi adapters don't support bridging
- The Proxmox enterprise repository requires a paid subscription; disable it for home lab use
- Routes set with `ip route` are not persistent across reboots; they need to be added to `/etc/network/interfaces`
