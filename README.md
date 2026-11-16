# Tailscale + Mullvad VPN Exit Node on Proxmox LXC

A self-hosted VPN gateway using **Tailscale, Mullvad VPN, WireGuard and nftables**, running inside a Debian LXC container on Proxmox VE.

The purpose of this guide is to route internet traffic from selected Tailscale devices through Mullvad VPN using a dedicated Proxmox LXC container.

Every value in this guide is parameterised. Set the variables in [Section 3](#3-set-your-variables) once and the commands can be pasted as-is.

Based on the approach described in *The Orange One – Tailscale + Mullvad*.

## Table of Contents

1. [Overview](#1-overview)
2. [Network Architecture](#2-network-architecture)
3. [Set Your Variables](#3-set-your-variables)
4. [Requirements](#4-requirements)
5. [Create the Proxmox LXC](#5-create-the-proxmox-lxc)
6. [Configure LXC TUN Passthrough](#6-configure-lxc-tun-passthrough)
7. [Initial Debian Setup](#7-initial-debian-setup)
8. [Install Tailscale](#8-install-tailscale)
9. [Configure Mullvad WireGuard](#9-configure-mullvad-wireguard)
10. [Enable IP Forwarding](#10-enable-ip-forwarding)
11. [Configure Policy Routing](#11-configure-policy-routing)
12. [Configure NAT](#12-configure-nat)
13. [Configure nftables Firewall](#13-configure-nftables-firewall)
14. [Advertise the Tailscale Exit Node](#14-advertise-the-tailscale-exit-node)
15. [Testing and Verification](#15-testing-and-verification)
16. [Persistence and Automatic Startup](#16-persistence-and-automatic-startup)
17. [Troubleshooting](#17-troubleshooting)
18. [Useful Commands](#18-useful-commands)

---

## 1. Overview

This guide creates a VPN gateway inside a Proxmox LXC container.

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
          +--------------------------+
          |                          |
          |  PROXMOX VE              |
          |                          |
          |  LXC $CTID               |
          |  Hostname: $CT_HOSTNAME  |
          |                          |
          |  Debian                  |
          |                          |
          |  IP: $CT_IP              |
          |                          |
          |  Tailscale               |
          |       |                  |
          |       v                  |
          |  Policy Routing          |
          |  Table $RT_TABLE         |
          |       |                  |
          |       v                  |
          |  nftables NAT            |
          |       |                  |
          |       v                  |
          |  WireGuard $WG_IF        |
          |                          |
          +--------------------------+
                      |
                 MULLVAD VPN
                      |
                   INTERNET
```

### Traffic flow

1. A client connects to the Tailscale network.
2. The client selects the gateway container as its exit node.
3. Traffic enters the LXC through the Tailscale interface.
4. Policy routing directs that traffic into the dedicated routing table.
5. The dedicated routing table sends traffic through the WireGuard interface.
6. nftables masquerades the Tailscale source address.
7. Mullvad forwards traffic to the public internet.

IPv6 forwarding from Tailscale is explicitly blocked in the custom firewall configuration.

---

## 3. Set Your Variables

**Edit every value below to match your own environment before running anything.** The examples are placeholders, not recommended values.

### On the Proxmox host

```bash
CTID=104                        # unused LXC container ID
CT_HOSTNAME=vpn-exit            # container hostname
CT_BRIDGE=vmbr0                 # Proxmox network bridge
CT_STORAGE=local-lvm            # storage backing the root disk
CT_DISK=16                      # root disk size in GB
CT_CORES=1                      # CPU cores
CT_MEMORY=512                   # RAM in MB
CT_SWAP=512                     # swap in MB
CT_IP=192.0.2.50/24             # static address/prefix free on your LAN
CT_GW=192.0.2.1                 # your LAN gateway
CT_TEMPLATE=debian-12-standard_12.7-1_amd64.tar.zst   # see Section 5
```

To use DHCP instead of a static address, set `CT_IP=dhcp` and omit `CT_GW` from the
`pct create` command. A static address is recommended for a gateway.

### Inside the LXC

```bash
WG_IF=wg0                       # WireGuard interface name
TS_IF=tailscale0                # Tailscale interface name
RT_TABLE=100                    # dedicated routing table number (1-252, unused)
RULE_PRIO=1000                  # ip rule priority
```

These variables only live for the duration of a shell session. Re-paste the block
after reconnecting, or append it to `~/.bashrc` inside the container.

---

## 4. Requirements

- Proxmox VE host with shell/root access.
- A Debian LXC template.
- A Tailscale account.
- A Mullvad VPN subscription.
- A Mullvad WireGuard configuration file.
- Internet connectivity on the Proxmox host.

---

## 5. Create the Proxmox LXC

### Sizing guidance

A VPN gateway is lightweight. 1 core, 512 MB RAM and a 16 GB disk is a reasonable
starting point; increase CPU and RAM if you expect sustained high throughput.

### Download a Debian template

On the Proxmox host:

```bash
pveam update
pveam available | grep debian
```

Pick a current Debian standard template from that list, set `CT_TEMPLATE` to its exact
filename, then download it:

```bash
pveam download local "$CT_TEMPLATE"
```

### Create the container

```bash
pct create "$CTID" "local:vztmpl/$CT_TEMPLATE" \
  --hostname "$CT_HOSTNAME" \
  --arch amd64 \
  --cores "$CT_CORES" \
  --memory "$CT_MEMORY" \
  --swap "$CT_SWAP" \
  --rootfs "$CT_STORAGE:$CT_DISK" \
  --net0 "name=eth0,bridge=$CT_BRIDGE,firewall=1,ip=$CT_IP,gw=$CT_GW" \
  --ostype debian \
  --unprivileged 1 \
  --features nesting=1
```

> **Note:** `nesting=1` is required so the container can manage its own network
> namespace tooling. The container is unprivileged, which is the safer default.

---

## 6. Configure LXC TUN Passthrough

WireGuard and Tailscale require access to `/dev/net/tun`, which an unprivileged
container does not get by default.

On the Proxmox host, append the passthrough lines to the container configuration:

```bash
cat >> "/etc/pve/lxc/$CTID.conf" <<'EOF'
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
EOF
```

Confirm the configuration:

```bash
pct config "$CTID"
```

Start the container and open a shell inside it:

```bash
pct start "$CTID"
pct enter "$CTID"
```

Verify the device is present inside the container:

```bash
ls -l /dev/net/tun
```

---

## 7. Initial Debian Setup

Run the following **inside the LXC**. Set the in-container variables from
[Section 3](#inside-the-lxc) first.

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
  iproute2 \
  openresolv
```

> `openresolv` is only needed if your Mullvad configuration contains a `DNS =` line;
> without it `wg-quick` fails with `resolvconf: command not found`.

Check network connectivity, substituting your own gateway:

```bash
ip a
ip route
ping -c 4 "$(ip route | awk '/^default/ {print $3; exit}')"
ping -c 4 1.1.1.1
```

---

## 8. Install Tailscale

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
tailscale ip -4
```

`tailscale ip -4` prints this node's address inside the `100.64.0.0/10` Tailscale
range. Later steps read it dynamically, so there is nothing to write down.

---

## 9. Configure Mullvad WireGuard

Generate a WireGuard configuration for your chosen Mullvad server from your Mullvad
account page, then place it inside the container at `/etc/wireguard/$WG_IF.conf`
(`/etc/wireguard/wg0.conf` with the default interface name).

Set secure permissions:

```bash
chmod 600 "/etc/wireguard/$WG_IF.conf"
```

> **Never commit this file to GitHub.** It contains your WireGuard private key.

### Add `Table = off`

The downloaded configuration uses `AllowedIPs = 0.0.0.0/0, ::/0`. By default
`wg-quick` turns that into routing rules that send *all* of the container's traffic
through Mullvad. This guide instead routes only Tailscale traffic through the tunnel
using an explicit policy routing table, so `wg-quick` must not install its own routes.

Add `Table = off` to the `[Interface]` section:

```ini
[Interface]
PrivateKey = <your-private-key>
Address    = <addresses issued by Mullvad>
DNS        = <dns server issued by Mullvad>
Table      = off

[Peer]
PublicKey  = <mullvad server public key>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint   = <mullvad server address>:51820
```

With `Table = off`, the container's own traffic continues to use your LAN gateway and
only forwarded Tailscale traffic is sent through Mullvad.

### Start WireGuard

```bash
wg-quick up "$WG_IF"
```

Check the interface:

```bash
ip addr show "$WG_IF"
wg show
```

A healthy tunnel shows a recent `latest handshake` and non-zero transfer counters.

### Test the connection

Mullvad's internal gateway is reachable at `10.64.0.1` from inside the tunnel:

```bash
ping -c 3 -I "$WG_IF" 10.64.0.1
```

Verify Mullvad connectivity:

```bash
curl --interface "$WG_IF" https://am.i.mullvad.net/connected
```

This should confirm that traffic through the WireGuard interface exits via Mullvad.

---

## 10. Enable IP Forwarding

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

## 11. Configure Policy Routing

Traffic arriving on the Tailscale interface is sent to a dedicated routing table whose
only route is the Mullvad tunnel.

Add the Mullvad default route to the table:

```bash
ip route replace default dev "$WG_IF" table "$RT_TABLE"
```

Add the policy routing rule:

```bash
ip rule add priority "$RULE_PRIO" iif "$TS_IF" lookup "$RT_TABLE"
```

Verify:

```bash
ip rule
ip route show table "$RT_TABLE"
```

Expected routing rule (with the default variable values):

```text
1000:   from all iif tailscale0 lookup 100
```

Expected routing table:

```text
default dev wg0
```

Test route selection using this node's own Tailscale address:

```bash
ip route get 1.1.1.1 from "$(tailscale ip -4)" iif "$TS_IF"
```

The output must show the WireGuard interface as the outgoing device. If it shows
`eth0` or your LAN interface, the rule or the table is not being matched.

---

## 12. Configure NAT

Tailscale devices use addresses within `100.64.0.0/10`. Those source addresses must be
masqueraded when forwarding traffic into the Mullvad tunnel.

Create the NAT table and chain:

```bash
nft add table ip mullvad_nat

nft "add chain ip mullvad_nat postrouting {
    type nat hook postrouting priority srcnat;
    policy accept;
}"
```

Add the masquerade rule:

```bash
nft add rule ip mullvad_nat postrouting \
  oifname "$WG_IF" \
  ip saddr 100.64.0.0/10 \
  masquerade
```

Verify:

```bash
nft list table ip mullvad_nat
```

---

## 13. Configure nftables Firewall

Create a dedicated forwarding policy for the Mullvad exit gateway. The heredoc below
is unquoted so the interface variables expand:

```bash
nft -f - <<EOF
table inet mullvad_exit {
    chain forward {
        type filter hook forward priority -10; policy accept;

        # Block IPv6 forwarding to prevent leaks
        iifname "$TS_IF" meta nfproto ipv6 drop

        # Allow traffic from Tailscale through Mullvad
        iifname "$TS_IF" oifname "$WG_IF" accept

        # Allow established replies back to Tailscale
        iifname "$WG_IF" oifname "$TS_IF" ct state established,related accept

        # Block any other forwarded Tailscale traffic
        iifname "$TS_IF" drop
    }
}
EOF
```

Verify:

```bash
nft list table inet mullvad_exit
```

This prevents forwarded Tailscale traffic from escaping through another network
interface if the tunnel drops.

> **Important:** This is a forwarding policy, not a complete firewall for traffic
> originating from the LXC itself. The IPv6 rule shown here also does not
> independently guarantee that every possible IPv6 leak path is blocked.

---

## 14. Advertise the Tailscale Exit Node

The LXC must advertise itself as an exit node:

```bash
tailscale set --advertise-exit-node
```

Verify:

```bash
tailscale status
```

The entry for this node should end with `offers exit node`, for example:

```text
100.x.y.z    <hostname>    ... idle; offers exit node
```

### Approve the exit node

Open the Tailscale administration console:

<https://login.tailscale.com/admin/machines>

Find the container by its hostname and approve its exit-node functionality if your
tailnet requires approval.

### Connect client devices

On a client device:

1. Open Tailscale.
2. Select **Exit Node**.
3. Choose the gateway container by hostname.
4. Connect.

The selected device now sends its internet traffic through the LXC and Mullvad.

---

## 15. Testing and Verification

### Check Tailscale

```bash
tailscale status
```

Confirm the container is connected and offering an exit node.

### Check WireGuard

```bash
wg show
```

Confirm the latest handshake is recent and traffic counters increase.

### Check routing

```bash
ip rule
ip route show table "$RT_TABLE"
```

### Check NAT and firewall

```bash
nft list table ip mullvad_nat
nft list table inet mullvad_exit
```

### Check forwarding

```bash
sysctl net.ipv4.ip_forward
```

### Check public IP

From a client device connected to the exit node:

```bash
curl https://am.i.mullvad.net/connected
```

The result should identify the connection as using Mullvad. Compare the public IP
before and after selecting the exit node.

---

## 16. Persistence and Automatic Startup

Nothing configured at runtime survives a reboot unless it is written to disk. Work
through all four items below, then reboot and re-run [Section 15](#15-testing-and-verification).

### Tailscale

```bash
systemctl enable --now tailscaled
```

The exit-node advertisement set with `tailscale set` is stored in Tailscale's own
state and persists across restarts.

### IP forwarding

Already persistent via `/etc/sysctl.d/99-vpn-gateway.conf` from
[Section 10](#10-enable-ip-forwarding).

### WireGuard and policy routing

Move the policy routing commands into the WireGuard configuration so they are applied
whenever the tunnel comes up and removed when it goes down. Add to the `[Interface]`
section of `/etc/wireguard/$WG_IF.conf`:

```ini
PostUp   = ip route replace default dev %i table 100
PostUp   = ip rule add priority 1000 iif tailscale0 lookup 100
PostDown = ip rule del priority 1000 iif tailscale0 lookup 100 || true
PostDown = ip route flush table 100 || true
```

`%i` expands to the interface name. Replace `100`, `1000` and `tailscale0` if you
changed `RT_TABLE`, `RULE_PRIO` or `TS_IF`.

The `ip rule` works even when the Tailscale interface does not exist yet; the kernel
attaches the rule once the interface appears.

Stop the manually started interface, then enable the service:

```bash
wg-quick down "$WG_IF"
systemctl enable --now "wg-quick@$WG_IF"
```

Verify:

```bash
systemctl status "wg-quick@$WG_IF"
wg show
ip rule
ip route show table "$RT_TABLE"
```

### nftables

Debian's `/etc/nftables.conf` starts with `flush ruleset`, so custom tables added at
runtime are lost on reload. Store them in a drop-in file and include it.

```bash
mkdir -p /etc/nftables.d

cat > /etc/nftables.d/mullvad-exit.conf <<EOF
table ip mullvad_nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "$WG_IF" ip saddr 100.64.0.0/10 masquerade
    }
}

table inet mullvad_exit {
    chain forward {
        type filter hook forward priority -10; policy accept;

        iifname "$TS_IF" meta nfproto ipv6 drop
        iifname "$TS_IF" oifname "$WG_IF" accept
        iifname "$WG_IF" oifname "$TS_IF" ct state established,related accept
        iifname "$TS_IF" drop
    }
}
EOF

grep -q 'nftables.d' /etc/nftables.conf \
  || echo 'include "/etc/nftables.d/*.conf"' >> /etc/nftables.conf
```

Interface names in nftables rules are matched per packet, so the rules load correctly
even before the tunnel is up.

Reload and enable:

```bash
systemctl enable nftables
systemctl restart nftables
nft list ruleset
```

### Reboot test

```bash
# On the Proxmox host
pct reboot "$CTID"
```

> **Reboot recovery is not complete until WireGuard, nftables and policy routing have
> all been verified after a restart, and a client device has been confirmed to still
> exit through Mullvad.**

---

## 17. Troubleshooting

### Mullvad is not connected

```bash
wg show
systemctl status "wg-quick@$WG_IF"
systemctl restart "wg-quick@$WG_IF"
```

If `wg-quick` fails with `resolvconf: command not found`, install `openresolv` or
remove the `DNS =` line from the configuration.

### Tailscale is not connected

```bash
systemctl status tailscaled
tailscale status
systemctl restart tailscaled
```

If `tailscaled` cannot create its interface, confirm TUN passthrough from
[Section 6](#6-configure-lxc-tun-passthrough) with `ls -l /dev/net/tun`.

### Traffic is not routing through Mullvad

```bash
ip rule
ip route show table "$RT_TABLE"
ip addr show "$WG_IF"
ip addr show "$TS_IF"
nft list ruleset
```

Then confirm route selection:

```bash
ip route get 1.1.1.1 from "$(tailscale ip -4)" iif "$TS_IF"
```

If the outgoing device is not the WireGuard interface, the policy rule is missing or
the dedicated routing table is empty.

### The whole container lost internet after starting WireGuard

`Table = off` is missing from the WireGuard configuration, so `wg-quick` installed its
own default routes. See [Section 9](#add-table--off).

### The client has no internet while using the exit node

Check IP forwarding and NAT:

```bash
sysctl net.ipv4.ip_forward
nft list table ip mullvad_nat
```

### Check all routes

```bash
ip route show table all
```

---

## 18. Useful Commands

Inside the LXC:

| Purpose | Command |
| --- | --- |
| Tailscale status | `tailscale status` |
| This node's Tailscale IP | `tailscale ip -4` |
| Restart Tailscale | `systemctl restart tailscaled` |
| WireGuard status | `wg show` |
| Start WireGuard | `wg-quick up "$WG_IF"` |
| Stop WireGuard | `wg-quick down "$WG_IF"` |
| Restart WireGuard service | `systemctl restart "wg-quick@$WG_IF"` |
| View routing rules | `ip rule` |
| View routing table | `ip route show table "$RT_TABLE"` |
| View all routes | `ip route show table all` |
| View nftables | `nft list ruleset` |
| View NAT | `nft list table ip mullvad_nat` |
| View firewall | `nft list table inet mullvad_exit` |
| Check forwarding | `sysctl net.ipv4.ip_forward` |
| Check Mullvad exit | `curl --interface "$WG_IF" https://am.i.mullvad.net/connected` |

On the Proxmox host:

| Purpose | Command |
| --- | --- |
| Enter the container | `pct enter "$CTID"` |
| Start | `pct start "$CTID"` |
| Stop | `pct shutdown "$CTID"` |
| Restart | `pct reboot "$CTID"` |
| View configuration | `pct config "$CTID"` |
| View status | `pct status "$CTID"` |

---

## Security Notes

- Never commit `/etc/wireguard/*.conf` to a repository. Add it to `.gitignore`.
- Never publish Mullvad account numbers or Tailscale authentication keys.
- Keep private configuration files backed up securely.
- Verify that traffic is routed through Mullvad after every significant network change.
- Verify firewall and routing persistence after restarting the container.
- This setup blocks forwarded IPv6 from Tailscale rather than tunnelling it. Clients
  that depend on IPv6 connectivity will lose it while using this exit node.

---

**Reference:** The Orange One – Tailscale + Mullvad
