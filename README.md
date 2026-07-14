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
- Installed Docker and deployed n8n for workflow automation
- Built a battery alert workflow that sends a Discord notification when the server is running on battery power
- Set up an internal `10.0.0.0/24` network for containers with NAT through WiFi
- Deployed Pi-hole in an LXC container for DNS filtering and ad blocking
- Set up nginx reverse proxy to access Pi-hole from the home network
- Installed Tailscale VPN for secure remote access to the server from anywhere without opening router ports

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

### 9. Installing Docker

```bash
apt install docker.io
systemctl enable docker
systemctl start docker
```

### 10. Deploying n8n for Workflow Automation
Ran n8n as a Docker container with persistent storage:

```bash
docker run -d \
  --name n8n \
  --restart always \
  -p 5678:5678 \
  -e N8N_SECURE_COOKIE=false \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n
```

Accessible at `http://<proxmox-ip>:5678` from any device on the network.

### 11. Building a Battery Alert Workflow in n8n
Created an automated workflow that checks battery status every 5 minutes and sends a Discord alert if the server is running on battery power.

Workflow steps:
1. **Schedule Trigger** - runs every 5 minutes
2. **SSH Execute Command** - runs `acpi -b` on the Proxmox server to get battery status
3. **IF node** - checks if the output contains `Discharging`
4. **Discord node** - sends a warning message to a private Discord channel if true

Alert message:
```
⚠️ WARNING: Proxmox is running on battery power! Check the server immediately.
```

### 12. Setting Up Internal Container Network
Configured vmbr0 as an isolated internal bridge with NAT through WiFi so containers can access the internet without being on the home network:

```
auto vmbr0
iface vmbr0 inet static
        address 10.0.0.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o wlp0s20f3 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s 10.0.0.0/24 -o wlp0s20f3 -j MASQUERADE
```

### 13. Deploying Pi-hole in an LXC Container
Created an Ubuntu 24.04 LXC container with IP `10.0.0.2/24` and installed Pi-hole:

```bash
curl -sSL https://install.pi-hole.net | bash
```

### 14. Setting Up nginx Reverse Proxy for Pi-hole
Installed nginx on the Proxmox host to expose Pi-hole to the home network:

```bash
apt install nginx -y
```

Created `/etc/nginx/sites-available/pihole`:

```
server {
    listen 8888;
    location / {
        proxy_pass http://10.0.0.2;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/pihole /etc/nginx/sites-enabled/
systemctl restart nginx
```

Pi-hole dashboard accessible at `http://192.168.1.159:8888/admin`.

### 15. Setting Up Tailscale VPN
Installed Tailscale on the Proxmox host for secure remote access from anywhere:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
apt install tailscale -y
tailscale up
```

After authenticating via the Tailscale URL, all services are accessible remotely via the Tailscale IP without opening any ports on the router. Traffic is peer to peer and never passes through Tailscale's servers.

---

## Issues Faced and How They Were Fixed

| Issue | Cause | Fix |
|-------|-------|-----|
| `apt` couldn't find packages | No internet, missing repos | Brought up dock ethernet, added Debian repo |
| DNS resolution failing | No DNS server configured | Added Google DNS: `echo "nameserver 8.8.8.8" > /etc/resolv.conf` |
| Pings going to wrong interface | Default route via vmbr0 | Deleted vmbr0 route, added route via WiFi interface |
| apt hanging for minutes | Proxmox enterprise repo timing out | Disabled enterprise and ceph repos |
| Web UI unreachable from other devices | Traffic routing through disconnected vmbr0 | Removed vmbr0 subnet route |
| n8n showing secure cookie error | Accessing over HTTP without TLS | Set `N8N_SECURE_COOKIE=false` environment variable |
| LXC container had no internet | vmbr0 had no NAT rule after reboot | Added iptables MASQUERADE rule manually and made it persistent in interfaces file |
| Pi-hole unreachable from home network | Container on isolated 10.0.0.0/24 subnet | Set up nginx reverse proxy on Proxmox host |

---

## Next Steps

- [ ] Create first VM
- [ ] Set up automated backups
- [ ] Expand storage
- [ ] Set up TLS/HTTPS for n8n via Tailscale
- [ ] Build more automation workflows in n8n
- [ ] Add CPU and RAM usage alerts
- [ ] Configure Pi-hole as network-wide DNS

---

## Notes

- Proxmox on WiFi requires NAT instead of bridging for VM networking since most WiFi adapters don't support bridging
- The Proxmox enterprise repository requires a paid subscription; disable it for home lab use
- Routes set with `ip route` are not persistent across reboots; they need to be added to `/etc/network/interfaces`
- LXC containers on an isolated bridge need NAT rules on the host to access the internet
- nginx reverse proxy is a lightweight way to expose internal services without opening them directly to the home network
- Tailscale provides zero-config peer to peer VPN without requiring port forwarding or a static IP
