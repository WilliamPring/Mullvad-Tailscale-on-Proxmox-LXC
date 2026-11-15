# Tailscale + Mullvad VPN Exit Node on Proxmox LXC

A self-hosted VPN gateway using **Tailscale, Mullvad VPN, WireGuard and nftables**, running inside a Debian LXC container on Proxmox VE.

The purpose of this project is to route internet traffic from selected Tailscale devices through Mullvad VPN using a dedicated Proxmox LXC container.

Based on the approach described in *The Orange One – Tailscale + Mullvad*.

## Table of Contents

1. [Overview](#1-overview)
2. [Network Architecture](#2-network-architecture)
3. [Requirements](#3-requirements)
4. [Create the Proxmox LXC](#4-create-the-proxmox-lxc)
5. [Configure LXC TUN Passthrough](#5-configure-lxc-tun-passthrough)
6. [Initial Debian Setup](#6-initial-debian-setup)
7. [Install Tailscale](#7-install-tailscale)
8. [Configure Mullvad WireGuard](#8-configure-mullvad-wireguard)
9. [Enable IP Forwarding](#9-enable-ip-forwarding)
10. [Configure Policy Routing](#10-configure-policy-routing)
11. [Configure NAT](#11-configure-nat)
12. [Configure nftables Firewall](#12-configure-nftables-firewall)
13. [Advertise the Tailscale Exit Node](#13-advertise-the-tailscale-exit-node)
14. [Testing and Verification](#14-testing-and-verification)
15. [Automatic Startup and Persistence](#15-automatic-startup-and-persistence)
16. [Troubleshooting](#16-troubleshooting)
17. [Useful Commands](#17-useful-commands)

---

## 1. Overview

This project creates a VPN gateway inside a Proxmox LXC container.

Tailscale provides secure remote connectivity between devices, while Mullvad provides the public internet exit point.

When a device selects this LXC as its Tailscale exit node, its IPv4 internet traffic is forwarded through Mullvad's WireGuard tunnel.

### Main components

| Component | Purpose |
| --- | --- |
| Proxmox VE | Virtualization platform |
| Debian LXC | VPN gateway |
| Tailscale | Private network and exit node |
| Mullvad VPN | Public VPN exit |
| WireGuard | Encrypted Mullvad tunnel |
| nftables | Firewall and NAT |
| Policy routing | Direct Tailscale traffic through Mullvad |

---

## 2. Network Architecture

```text
                REMOTE DEVICES
          Laptop / Phone / Desktop
                      |
               TAILSCALE NETWORK
                      |
                  tailscale0
                      |
          +----------------------+
          |                      |
          |  PROXMOX VE          |
          |                      |
          |  LXC 104             |
          |  Hostname: vpn-exit  |
          |                      |
          |  Debian              |
          |                      |
          |  IP: 192.168.68.51   |
          |                      |
          |  Tailscale           |
          |       |              |
          |       v              |
          |  Policy Routing      |
          |  Table 100           |
          |       |              |
          |       v              |
          |  nftables NAT        |
          |       |              |
          |       v              |
          |  WireGuard wg0       |
          |                      |
          +----------------------+
                      |
                 MULLVAD VPN
                      |
                   INTERNET
```

### Traffic flow

1. A client connects to the Tailscale network.
2. The client selects `vpn-exit` as its exit node.
3. Traffic enters the LXC through `tailscale0`.
4. Policy routing directs traffic into routing table `100`.
5. Routing table `100` sends traffic through `wg0`.
6. nftables masquerades the Tailscale source address.
7. Mullvad forwards traffic to the public internet.

IPv6 forwarding from Tailscale is explicitly blocked in the custom firewall configuration.

---

## 3. Requirements

- Proxmox VE host.
- Debian LXC container.
- Tailscale account.
- Mullvad VPN subscription.
- Mullvad WireGuard configuration.
- Internet connectivity on the Proxmox host.
- Access to the Proxmox shell.

---

## 4. Create the Proxmox LXC

The following reflects the actual configuration of the deployed container.

### Container specifications

| Setting | Value |
| --- | --- |
| Container ID | 104 |
| Hostname | vpn-exit |
| Architecture | amd64 |
| Operating System | Debian |
| CPU | 1 core |
| RAM | 512 MB |
| Swap | 512 MB |
| Disk | 16 GB |
| Storage | local-lvm |
| Network bridge | vmbr0 |
| IP address | 192.168.68.51/22 |
| Gateway | 192.168.68.1 |
| Unprivileged | Yes |
| Nesting | Enabled |

### Download Debian template

On the Proxmox host:

```bash
pveam update
pveam available | grep debian
```

Download the desired Debian standard template:

```bash
pveam download local debian-12-standard_12.7-1_amd64.tar.zst
```

> **Note:** Replace the template filename with the currently available version shown by `pveam available`.

### Create the container

Example command matching the deployed configuration:

```bash
pct create 104 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname vpn-exit \
  --arch amd64 \
  --cores 1 \
  --memory 512 \
  --swap 512 \
  --rootfs local-lvm:16 \
  --net0 name=eth0,bridge=vmbr0,firewall=1,gw=192.168.68.1,ip=192.168.68.51/22 \
  --ostype debian \
  --unprivileged 1 \
  --features nesting=1
```

The template version above is an example; use the downloaded template's actual filename.

---

## 5. Configure LXC TUN Passthrough

WireGuard requires access to `/dev/net/tun`.

On the Proxmox host, edit:

```bash
nano /etc/pve/lxc/104.conf
```

Add:

```ini
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

The relevant container configuration should include:

```ini
arch: amd64
cores: 1
features: nesting=1
hostname: vpn-exit
memory: 512
net0: name=eth0,bridge=vmbr0,firewall=1,gw=192.168.68.1,ip=192.168.68.51/22,type=veth
ostype: debian
rootfs: local-lvm:vm-104-disk-0,size=16G
swap: 512
unprivileged: 1
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Start the container:

```bash
pct start 104
pct enter 104
```

---

## 6. Initial Debian Setup

Run inside the LXC:

```bash
apt update
apt full-upgrade -y
```

Install required packages:

```bash
apt install -y \
  curl \
  netcat-openbsd \
  wireguard \
  wireguard-tools \
  nftables \
  iproute2
```

Check network connectivity:

```bash
ip a
ip route
ping -c 4 192.168.68.1
ping -c 4 1.1.1.1
```

---

## 7. Install Tailscale

Install Tailscale using the official installation script:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Enable the service:

```bash
systemctl enable --now tailscaled
```

Authenticate:

```bash
tailscale up
```

Follow the authentication URL and authorize the container.

Verify:

```bash
tailscale status
```

The deployed container has the following Tailscale address:

```text
100.115.81.73
```

---

## 8. Configure Mullvad WireGuard

Generate a WireGuard configuration through your Mullvad account.

Download the configuration for the desired Mullvad server and place it inside the container.

The configuration file should be:

```text
/etc/wireguard/wg0.conf
```

Set secure permissions:

```bash
chmod 600 /etc/wireguard/wg0.conf
```

> **Never commit this configuration file to GitHub.** It contains your WireGuard private key.

### Start WireGuard

```bash
wg-quick up wg0
```

Check the interface:

```bash
ip addr show wg0
wg show
```

The deployed configuration used:

```text
Interface: wg0
Endpoint: 178.249.214.15:51820
```

### Test the connection

Check connectivity to the Mullvad tunnel:

```bash
ping -c 3 -I wg0 10.64.0.1
```

Verify Mullvad connectivity:

```bash
curl --interface wg0 https://am.i.mullvad.net/connected
```

This should confirm that traffic through `wg0` is connected to Mullvad.

---

## 9. Enable IP Forwarding

IP forwarding allows the LXC to forward traffic between Tailscale and Mullvad.

Enable it immediately:

```bash
sysctl -w net.ipv4.ip_forward=1
```

Make it persistent:

```bash
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-vpn-gateway.conf
```

Verify:

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

---

## 10. Configure Policy Routing

Create a dedicated routing table for Tailscale traffic.

The table number used in this configuration is `100`.

Add the Mullvad default route:

```bash
ip route replace default dev wg0 table 100
```

Add the policy routing rule:

```bash
ip rule add priority 1000 iif tailscale0 lookup 100
```

This instructs Linux to use routing table `100` for incoming traffic from Tailscale.

Verify:

```bash
ip rule
ip route show table 100
```

Expected routing rule:

```text
1000: from all iif tailscale0 lookup 100
```

Expected routing table:

```text
default dev wg0
```

Test route selection:

```bash
ip route get 1.1.1.1 from 100.115.81.73 iif tailscale0
```

The expected outgoing interface is:

```text
dev wg0
```

---

## 11. Configure NAT

Tailscale devices use addresses within:

```text
100.64.0.0/10
```

We need to masquerade those addresses when forwarding traffic through Mullvad.

Create the NAT table:

```bash
nft add table ip mullvad_nat
```

Create the postrouting chain:

```bash
nft 'add chain ip mullvad_nat postrouting {
    type nat hook postrouting priority srcnat;
    policy accept;
}'
```

Add the masquerade rule:

```bash
nft add rule ip mullvad_nat postrouting \
  oifname "wg0" \
  ip saddr 100.64.0.0/10 \
  masquerade
```

Verify:

```bash
nft list table ip mullvad_nat
```

---

## 12. Configure nftables Firewall

Create a dedicated firewall table for the Mullvad exit gateway.

Run:

```bash
nft -f - <<'EOF'
table inet mullvad_exit {
    chain forward {
        type filter hook forward priority -10; policy accept;

        # Block IPv6 forwarding to prevent leaks
        iifname "tailscale0" meta nfproto ipv6 drop

        # Allow traffic from Tailscale through Mullvad
        iifname "tailscale0" oifname "wg0" accept

        # Allow established replies back to Tailscale
        iifname "wg0" oifname "tailscale0" ct state established,related accept

        # Block any other forwarded Tailscale traffic
        iifname "tailscale0" drop
    }
}
EOF
```

Verify:

```bash
nft list table inet mullvad_exit
```

The firewall is designed to prevent forwarded Tailscale traffic from escaping through another network interface.

> **Important:** This is a forwarding policy, not a complete firewall for all traffic originating from the LXC itself. The IPv6 rule shown here also does not independently guarantee that every possible IPv6 leak path is blocked.

---

## 13. Advertise the Tailscale Exit Node

The LXC must advertise itself as an exit node.

The Tailscale configuration command is:

```bash
tailscale set --advertise-exit-node
```

Verify:

```bash
tailscale status
```

The deployed container appeared as:

```text
100.115.81.73    vpn-exit    ... idle; offers exit node
```

### Approve the exit node

Open the Tailscale administration console:

<https://login.tailscale.com/admin/machines>

Find `vpn-exit` and approve its exit-node functionality if required.

### Connect client devices

On a client device:

1. Open Tailscale.
2. Select **Exit Node**.
3. Choose `vpn-exit`.
4. Connect.

The selected device should now send its internet traffic through the LXC and Mullvad.

---

## 14. Testing and Verification

### Check Tailscale

```bash
tailscale status
```

Confirm that `vpn-exit` is connected and offering an exit node.

### Check WireGuard

```bash
wg show
```

Confirm that the latest handshake is recent and that traffic counters increase.

### Check routing

```bash
ip rule
ip route show table 100
```

### Check NAT

```bash
nft list table ip mullvad_nat
```

### Check firewall

```bash
nft list table inet mullvad_exit
```

### Check forwarding

```bash
sysctl net.ipv4.ip_forward
```

### Check public IP

From a device connected to the `vpn-exit` exit node:

```bash
curl https://am.i.mullvad.net/connected
```

The result should identify the connection as using Mullvad.

Also compare the public IP before and after selecting the exit node.

---

## 15. Automatic Startup and Persistence

This section is especially important.

The original deployment has some settings saved persistently and others configured manually.

### Tailscale

Confirmed enabled:

```bash
systemctl enable --now tailscaled
```

### IP forwarding

Confirmed persistent:

```text
/etc/sysctl.d/99-vpn-gateway.conf
```

Contents:

```ini
net.ipv4.ip_forward=1
```

### nftables

The nftables service is enabled and active.

However, the original `/etc/nftables.conf` only contains the default filter table. The custom `mullvad_nat` and `mullvad_exit` tables were added manually and must be incorporated into the persistent configuration.

Do not assume the custom rules survive a reboot until this has been corrected and tested.

### WireGuard

The original deployment has:

```text
wg-quick@wg0: disabled
```

The interface was started manually with:

```bash
wg-quick up wg0
```

To configure automatic startup, first ensure the existing manually started interface is stopped:

```bash
wg-quick down wg0
```

Then enable and start its systemd service:

```bash
systemctl enable --now wg-quick@wg0
```

Verify:

```bash
systemctl status wg-quick@wg0
wg show
```

### Policy routing

The original policy routing commands were entered manually:

```bash
ip route replace default dev wg0 table 100
ip rule add priority 1000 iif tailscale0 lookup 100
```

These commands also need a persistent startup mechanism.

They should be restored after both `wg0` and `tailscale0` are available.

> **Reboot recovery is not considered complete until WireGuard, nftables, and policy routing have all been verified after a restart.**

---

## 16. Troubleshooting

### Mullvad is not connected

Check:

```bash
wg show
systemctl status wg-quick@wg0
```

Restart:

```bash
systemctl restart wg-quick@wg0
```

### Tailscale is not connected

Check:

```bash
systemctl status tailscaled
tailscale status
```

Restart:

```bash
systemctl restart tailscaled
```

### Traffic is not routing through Mullvad

Check:

```bash
ip rule
ip route show table 100
ip addr show wg0
ip addr show tailscale0
```

Check firewall:

```bash
nft list ruleset
```

### IP forwarding is disabled

```bash
sysctl -w net.ipv4.ip_forward=1
```

### Mullvad connection verification

```bash
curl --interface wg0 https://am.i.mullvad.net/connected
```

### Check network routes

```bash
ip route show table all
```

---

## 17. Useful Commands

| Purpose | Command |
| --- | --- |
| Tailscale status | `tailscale status` |
| Restart Tailscale | `systemctl restart tailscaled` |
| WireGuard status | `wg show` |
| Start WireGuard | `wg-quick up wg0` |
| Stop WireGuard | `wg-quick down wg0` |
| Restart WireGuard service | `systemctl restart wg-quick@wg0` |
| View routing rules | `ip rule` |
| View routing table | `ip route show table 100` |
| View all routes | `ip route show table all` |
| View nftables | `nft list ruleset` |
| View NAT | `nft list table ip mullvad_nat` |
| View firewall | `nft list table inet mullvad_exit` |
| Check forwarding | `sysctl net.ipv4.ip_forward` |
| Check Mullvad IP | `curl https://am.i.mullvad.net/connected` |

### Proxmox Management

All the following commands run on the Proxmox host, not inside the LXC.

Enter the container:

```bash
pct enter 104
```

Start:

```bash
pct start 104
```

Stop:

```bash
pct shutdown 104
```

Restart:

```bash
pct reboot 104
```

View configuration:

```bash
pct config 104
```

View status:

```bash
pct status 104
```

---

## Security Notes

- Never commit `/etc/wireguard/wg0.conf` to GitHub.
- Never publish Mullvad account credentials.
- Never publish Tailscale authentication keys.
- Keep private configuration files backed up securely.
- Verify that traffic is routed through Mullvad after every significant network change.
- Verify firewall and routing persistence after restarting the container.

---

## Project Status

| Component | Status |
| --- | --- |
| Proxmox LXC 104 | Configured |
| Debian | Installed |
| TUN passthrough | Configured |
| Tailscale | Installed and active |
| Tailscale exit-node advertisement | Confirmed |
| Mullvad WireGuard | Connected |
| IP forwarding | Enabled and persistent |
| Policy routing | Configured |
| nftables NAT | Configured at runtime |
| nftables forwarding firewall | Configured at runtime |
| nftables custom-rule persistence | Needs completion |
| WireGuard automatic startup | Needs completion |
| Policy routing automatic startup | Needs completion |

---

**Reference:** The Orange One – Tailscale + Mullvad

| | |
| --- | --- |
| **Platform** | Proxmox VE / Debian LXC |
| **Container** | 104 (vpn-exit) |
| **VPN** | Mullvad / WireGuard |
| **Private network** | Tailscale |
| **Firewall** | nftables |
