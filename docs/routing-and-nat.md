# Routing and NAT

This document describes how routing and Network Address Translation (NAT)
are implemented on the Proxmox VE host. Environment-specific private
addresses are represented with placeholders in this public version.

The Proxmox server acts as the default gateway for several internal
networks and routes selected traffic toward the upstream network through
`vmbr0`.

---

## Routing Overview

The public documentation uses placeholders for environment-specific private network ranges:

| Network | Proxmox Interface | Gateway |
|---|---|---|
| WAN / Upstream | `vmbr0` | Assigned upstream |
| `<trusted-lan-subnet>` | `vmbr1` | `<trusted-lan-gateway>` |
| `<wifi-subnet>` | `vmbr2` | `<wifi-gateway>` |
| `<lab-subnet>` | `vmbrlab` | `<lab-gateway>` |

The Proxmox host therefore operates as a router between the internal
network segments.

Traffic between networks is still subject to firewall policy.

---

## IPv4 Forwarding

Linux does not route IPv4 traffic between interfaces unless IPv4
forwarding is enabled.

Verification:

```bash
sysctl net.ipv4.ip_forward
```

Expected result:

```text
net.ipv4.ip_forward = 1
```

Conceptually:

```text
VM / Device
    |
    v
Internal bridge
    |
    v
Proxmox
    |
IPv4 forwarding
    |
    v
Another network
```

Without IP forwarding, the Proxmox host could communicate with each
network itself, but it would not forward packets between them.

---

## Routing Decision

When a packet reaches the Proxmox host, Linux checks the routing table to
determine where the packet should be sent next.

Useful commands:

```bash
ip route
```

and:

```bash
ip route get 1.1.1.1
```

For example, Internet traffic originating from an internal network is
normally routed toward:

```text
vmbr0
```

---

## Private Networks

The internal networks use RFC1918 private IPv4 address space. Exact environment-specific ranges are represented by placeholders:

```text
<trusted-lan-subnet>
<wifi-subnet>
<lab-subnet>
```

These addresses are not directly routable on the public Internet.

Source NAT is therefore required when internal systems access external
networks.

---

## Source NAT

Outgoing traffic uses source NAT before leaving through `vmbr0`.

The environment uses:

```text
MASQUERADE
```

rather than a fixed SNAT address.

Example:

```bash
iptables -t nat -A POSTROUTING \
-s <trusted-lan-subnet> \
-o vmbr0 \
-j MASQUERADE
```

Equivalent rules are used for the other internal networks.

Conceptually:

```text
<trusted-client-ip>
      |
      v
   vmbr1
      |
      v
   Proxmox
      |
      v
POSTROUTING
MASQUERADE
      |
      v
   vmbr0
      |
      v
 Internet
```

---

## Why MASQUERADE Is Used

`vmbr0` receives its WAN-side address dynamically.

Because the address may change, a fixed SNAT address would require the
firewall configuration to be updated whenever the upstream address
changes.

MASQUERADE automatically uses the current address of the outgoing
interface.

Memory aid:

```text
SNAT / MASQUERADE = change the SOURCE address
DNAT              = change the DESTINATION address
```

---

## NAT Table

NAT rules can be inspected with:

```bash
sudo iptables -t nat -L -v -n --line-numbers
```

Important chains include:

```text
PREROUTING
POSTROUTING
```

### PREROUTING

Used when a destination must be modified before Linux makes the routing
decision.

Typical use:

```text
DNAT
```

### POSTROUTING

Used after the routing decision, immediately before traffic leaves the
system.

Typical use:

```text
SNAT
MASQUERADE
```

---

## Packet Flow - Outbound Connection

For an outbound connection from the main LAN:

```text
<trusted-client-ip>
      |
      v
    vmbr1
      |
      v
   Proxmox
      |
      |  Routing decision
      v
    FORWARD
      |
      v
 NAT POSTROUTING
   MASQUERADE
      |
      v
    vmbr0
      |
      v
   Internet
```

The original source address:

```text
<trusted-client-ip>
```

is replaced with the current WAN-side address used by `vmbr0`.

---

## Return Traffic

When the remote server responds, Linux connection tracking remembers the
NAT mapping.

Conceptually:

```text
Internet response
       |
       v
     vmbr0
       |
       v
Connection Tracking
       |
       v
Reverse NAT
       |
       v
     vmbr1
       |
       v
<trusted-client-ip>
```

The internal device therefore receives the response even though its
private IP address was not visible to the external server.

---

## Connection Tracking

Linux connection tracking keeps state information about active network
connections.

Common states include:

```text
NEW
ESTABLISHED
RELATED
INVALID
```

A typical stateful firewall rule allows traffic belonging to already
approved connections:

```bash
iptables -A FORWARD \
-m conntrack \
--ctstate ESTABLISHED,RELATED \
-j ACCEPT
```

This allows reply traffic without allowing arbitrary new inbound
connections.

---

## Proxmox Firewall and Conntrack Zones

When the Proxmox firewall is enabled for virtual machines, Proxmox uses
additional virtual firewall bridges.

These commonly use interface names beginning with:

```text
fwbr
```

During testing, NAT did not work correctly until traffic from these
bridges was assigned to a connection tracking zone.

The rule used is:

```bash
iptables -t raw -I PREROUTING \
-i fwbr+ \
-j CT --zone 1
```

The `+` acts as an interface-name wildcard.

Therefore:

```text
fwbr+
```

matches interfaces whose names begin with `fwbr`.

The rule assigns matching traffic to connection tracking zone:

```text
1
```

Verification:

```bash
sudo iptables -t raw -L PREROUTING -v -n --line-numbers
```

---

## DNAT

Destination NAT changes the destination of an incoming packet.

Conceptually:

```text
Internet
   |
   v
vmbr0
   |
   v
NAT PREROUTING
   |
   | DNAT
   v
Internal VM
```

Example:

```text
Public-facing TCP port
        |
        v
Proxmox vmbr0
        |
       DNAT
        |
        v
Internal VM address
```

DNAT itself does not automatically mean that the packet is permitted by
the firewall.

Two separate questions must be answered:

```text
DNAT:
Where should this packet go?

Firewall:
Is this packet allowed to go there?
```

Detailed Internet-facing DNAT rules for the T-Pot honeypot are documented
in the separate **Proxmox T-Pot Honeypot Lab** repository.

---

## DNAT vs SNAT

A useful comparison:

| Feature | DNAT | SNAT / MASQUERADE |
|---|---|---|
| Changes | Destination | Source |
| Typical chain | `PREROUTING` | `POSTROUTING` |
| Typical use | Incoming port forwarding | Outbound Internet access |
| Example | WAN -> internal VM | Private VM -> Internet |

Memory aid:

```text
D = Destination
S = Source
```

---

## Firewall vs NAT

NAT and firewalling solve different problems.

### NAT

Changes packet addressing.

Examples:

```text
DNAT
SNAT
MASQUERADE
```

### Firewall

Controls whether traffic is permitted.

Examples:

```text
ACCEPT
DROP
LOG
```

A packet can therefore have a valid NAT translation and still be blocked
by the firewall.

---

## Lab Network

The cybersecurity lab uses:

```text
<lab-subnet>
```

with gateway:

```text
<lab-gateway>
```

The network has a MASQUERADE rule so that selected lab traffic can reach
the Internet through `vmbr0`.

However, NAT does not mean that the lab has unrestricted Internet access.

Firewall rules independently control which outbound connections are
permitted.

This distinction is important:

```text
Routing
    determines the path

NAT
    changes addressing

Firewall
    determines whether traffic is allowed
```

---

## Persistent NAT Configuration

The NAT configuration is attached to the Proxmox network configuration
using interface lifecycle commands.

Example:

```text
post-up iptables -t nat -A POSTROUTING ...
post-down iptables -t nat -D POSTROUTING ...
```

The `post-up` command creates the rule when the interface is brought up.

The corresponding `post-down` command removes it when the interface is
brought down.

This prevents stale rules from remaining after interface lifecycle
changes.

---

## Verification Commands

### Routing table

```bash
ip route
```

### Test routing decision

```bash
ip route get 1.1.1.1
```

### IPv4 forwarding

```bash
sysctl net.ipv4.ip_forward
```

### NAT rules

```bash
sudo iptables -t nat -L -v -n --line-numbers
```

### Forwarding rules

```bash
sudo iptables -L FORWARD -v -n --line-numbers
```

### Connection tracking zone

```bash
sudo iptables -t raw -L PREROUTING -v -n --line-numbers
```

### Interface addresses

```bash
ip -br addr
```

---

## Troubleshooting Checklist

If an internal VM cannot reach the Internet, check the path in order:

```text
1. VM IP address
       ↓
2. Default gateway
       ↓
3. Proxmox bridge
       ↓
4. IPv4 forwarding
       ↓
5. FORWARD firewall rules
       ↓
6. NAT POSTROUTING rule
       ↓
7. Conntrack configuration
       ↓
8. vmbr0 / upstream connection
```

Useful commands:

```bash
ip -br addr
ip route
sysctl net.ipv4.ip_forward
sudo iptables -L FORWARD -v -n
sudo iptables -t nat -L POSTROUTING -v -n
sudo iptables -t raw -L PREROUTING -v -n
```

Packet counters are especially useful because they show whether traffic
is actually matching the expected firewall or NAT rule.

---

## What I Learned

While building the environment, several networking concepts became
important in practice:

- Routing and NAT are separate functions.
- NAT does not automatically permit traffic through the firewall.
- DNAT modifies the destination before routing.
- MASQUERADE modifies the source address for outbound traffic.
- Linux connection tracking maintains state for NAT and stateful
  firewall rules.
- Proxmox firewall bridges can affect connection tracking behavior.
- Packet counters are useful for identifying where traffic stops.
- Temporary firewall changes should be tested before being made
  persistent.

---

## Related Documentation

- [Network Architecture](architecture.md)
- [Network Configuration](network-configuration.md)
- [Security Design](security.md)
- [Troubleshooting](troubleshooting.md)
- [Implementation Notes](implementation-notes.md)
