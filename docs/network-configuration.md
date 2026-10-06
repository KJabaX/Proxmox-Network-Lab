# Network Configuration

This document describes the practical network configuration used by the
Proxmox VE host. Environment-specific addresses are represented with
placeholders in this public version.

The main network configuration is stored in:

```text
/etc/network/interfaces
```

The environment uses Linux bridges to separate the WAN, trusted LAN,
Wi-Fi and cybersecurity lab networks.

---

## Interface Overview

| Interface | Type | Network | Purpose |
|---|---|---|---|
| `eno1` | Physical NIC | Upstream | WAN |
| `eno2` | Physical NIC | `<trusted-lan-subnet>` | Main LAN |
| `eno3` | Physical NIC | `<wifi-subnet>` | Wi-Fi network |
| `eno4` | Physical NIC | - | Currently unused |
| `vmbr0` | Linux bridge | DHCP | WAN |
| `vmbr1` | Linux bridge | `<trusted-lan-subnet>` | Main LAN |
| `vmbr2` | Linux bridge | `<wifi-subnet>` | Wi-Fi |
| `vmbrlab` | Linux bridge | `<lab-subnet>` | Cybersecurity lab |

---

## Loopback Interface

The standard loopback interface is configured as:

```text
auto lo
iface lo inet loopback
```

---

## vmbr0 - WAN Bridge

`vmbr0` connects the Proxmox host to the upstream network.

```text
auto vmbr0
iface vmbr0 inet dhcp
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
```

### Explanation

`bridge-ports eno1`

connects the physical Ethernet interface `eno1` to the bridge.

`inet dhcp`

means that the WAN-side IPv4 address is assigned dynamically.

`bridge-stp off`

disables Spanning Tree Protocol for this bridge.

`bridge-fd 0`

sets the bridge forwarding delay to zero.

The dynamically assigned WAN address is one reason why NAT rules use
MASQUERADE rather than a fixed source address.

---

## vmbr1 - Main LAN

`vmbr1` provides the primary trusted wired network.

```text
auto vmbr1
iface vmbr1 inet static
    address <trusted-lan-gateway>/24
    bridge-ports eno2
    bridge-stp off
    bridge-fd 0
```

Network:

```text
<trusted-lan-subnet>
```

Gateway provided by Proxmox:

```text
<trusted-lan-gateway>
```

Physical interface:

```text
eno2
```

The Proxmox host acts as the gateway for devices connected to this
network.

---

## vmbr2 - Wi-Fi Network

`vmbr2` provides a separate network for Wi-Fi connectivity.

```text
auto vmbr2
iface vmbr2 inet static
    address <wifi-gateway>/24
    bridge-ports eno3
    bridge-stp off
    bridge-fd 0
```

Network:

```text
<wifi-subnet>
```

Gateway:

```text
<wifi-gateway>
```

Physical interface:

```text
eno3
```

Keeping this network on a separate bridge allows different routing and
firewall policies to be applied independently from the main LAN.

---

## vmbrlab - Cybersecurity Lab

`vmbrlab` is an isolated virtual network used for cybersecurity
workloads.

```text
auto vmbrlab
iface vmbrlab inet static
    address <lab-gateway>/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
```

Network:

```text
<lab-subnet>
```

Gateway:

```text
<lab-gateway>
```

Physical interface:

```text
None
```

The important configuration line is:

```text
bridge-ports none
```

This means that the bridge is not directly connected to a dedicated
physical Ethernet interface.

Virtual machines connected to `vmbrlab` communicate through the Proxmox
host.

The current T-Pot VM uses:

```text
<honeypot-ip>
```

Detailed T-Pot firewall, DNAT and public ingress configuration is
documented in the separate Proxmox T-Pot Honeypot Lab repository.

---

## NAT for Internal Networks

The internal networks use the WAN-facing `vmbr0` interface for Internet
access.

The Proxmox host performs source NAT using MASQUERADE.

Conceptually:

```text
<trusted-lan-subnet> \
<wifi-subnet>  ---> Proxmox ---> vmbr0 ---> Internet
<lab-subnet>  /
```

Example:

```bash
iptables -t nat -A POSTROUTING \
-s <trusted-lan-subnet> \
-o vmbr0 \
-j MASQUERADE
```

Similar rules are used for the other internal networks.

More detailed NAT documentation is available in:

[Routing and NAT](routing-and-nat.md)

---

## IPv4 Forwarding

Because the Proxmox host acts as a router, IPv4 forwarding must be
enabled.

Verification:

```bash
sysctl net.ipv4.ip_forward
```

Expected result:

```text
net.ipv4.ip_forward = 1
```

Without IPv4 forwarding, packets would not be routed between interfaces.

---

## Persistent Configuration

Proxmox networking is configured through:

```text
/etc/network/interfaces
```

Additional firewall and NAT rules can be attached to interface lifecycle
events using:

```text
post-up
post-down
```

For example:

```text
post-up iptables ...
post-down iptables ...
```

`post-up` adds the required rule when the interface is brought up.

`post-down` removes the corresponding rule when the interface is brought
down.

This makes the configuration persistent across normal interface
reinitialization and system reboots.

---

## Proxmox Firewall Interfaces

When the Proxmox firewall is enabled for a VM, additional virtual
interfaces can appear.

Examples include:

```text
fwbr*
fwpr*
fwln*
tap*
```

Because of this, the runtime bridge membership may contain Proxmox
firewall interfaces even when the static configuration contains:

```text
bridge-ports none
```

For example, `vmbrlab` may show a runtime interface such as:

```text
fwpr<vmid>p0
```

This does not mean that `vmbrlab` has been connected to a physical NIC.
It is part of the Proxmox virtual firewall networking path.

---

## Configuration Verification

The configuration can be checked without restarting networking:

```bash
sudo ifquery --check -a
```

Interface parameters that match the running configuration are shown with:

```text
[pass]
```

Useful additional commands include:

```bash
ip -br addr
ip route
bridge link
bridge vlan show
```

NAT rules:

```bash
sudo iptables -t nat -L -v -n
```

Forwarding rules:

```bash
sudo iptables -L FORWARD -v -n
```

Raw connection tracking rules:

```bash
sudo iptables -t raw -L PREROUTING -v -n
```

---

## Safe Configuration Workflow

Network changes are normally made using the following process:

1. Inspect the current configuration.
2. Make one controlled change.
3. Validate syntax and interface state.
4. Inspect routing and firewall rules.
5. Test connectivity.
6. Test isolation boundaries.
7. Make temporary rules persistent only after successful testing.
8. Document the change.

A network restart is avoided when it is not necessary, especially when
the Proxmox server is being managed remotely.

---

## Related Documentation

- [Network Architecture](architecture.md)
- [Routing and NAT](routing-and-nat.md)
- [Security Design](security.md)
- [Troubleshooting](troubleshooting.md)
- [Implementation Notes](implementation-notes.md)
