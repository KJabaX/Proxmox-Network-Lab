# Security Design

This document describes the security architecture and network isolation
controls used in the Proxmox VE environment. Environment-specific hostnames,
usernames, and private addresses are sanitized in this public version.

The primary security principle is to separate trusted systems from
experimental and potentially hostile cybersecurity workloads.

The detailed T-Pot honeypot ingress configuration is maintained in the
separate **Proxmox T-Pot Honeypot Lab** repository.

---

## Security Goals

The network design aims to:

- Separate trusted systems from cybersecurity lab workloads
- Prevent lab systems from initiating connections to trusted networks
- Provide controlled Internet access where required
- Use stateful firewalling for permitted connections
- Keep Proxmox management services separate from Internet-facing services
- Avoid exposing unnecessary services
- Keep network and firewall configuration persistent
- Make firewall behavior observable through logging and packet counters
- Support future network segmentation without redesigning the environment

---

## Trust Zones

The environment can be viewed as several different trust zones.

```text
                    Internet
                       |
                       v
                     vmbr0
                       |
                       v
                  Proxmox Host
                  /     |      \
                 /      |       \
                v       v        v
             vmbr1    vmbr2    vmbrlab
               |        |         |
               v        v         v
           Main LAN   Wi-Fi   Security Lab
           Trusted            Untrusted /
                              Experimental
```

The most important security boundary is between:

```text
vmbrlab
```

and:

```text
vmbr1
vmbr2
```

The cybersecurity lab is treated as a less trusted environment.

---

## Network Segmentation

Separate Linux bridges are used for different network purposes.

| Bridge | Purpose | Trust Level |
|---|---|---|
| `vmbr0` | WAN / upstream | External |
| `vmbr1` | Main LAN | Trusted |
| `vmbr2` | Wi-Fi network | Internal |
| `vmbrlab` | Cybersecurity lab | Experimental / Untrusted |

The use of separate bridges allows routing and firewall policies to be
applied independently.

This provides a stronger security boundary than placing all virtual
machines and physical devices on the same Layer 2 network.

---

## Cybersecurity Lab Isolation

The lab network uses:

```text
<lab-subnet>
```

with Proxmox acting as its gateway:

```text
<lab-gateway>
```

New connections originating from the lab toward the trusted networks are
blocked.

Conceptually:

```text
vmbrlab -> vmbr1    DROP
vmbrlab -> vmbr2    DROP
```

The important direction is:

```text
Lab -> Trusted network
```

because the goal is to prevent compromised or intentionally vulnerable
lab systems from moving laterally into trusted environments.

---

## Stateful Firewalling

The firewall uses Linux connection tracking.

Traffic belonging to existing permitted connections is accepted using:

```text
ESTABLISHED,RELATED
```

Example:

```bash
iptables -I FORWARD 1 \
-m conntrack \
--ctstate ESTABLISHED,RELATED \
-j ACCEPT
```

This allows response traffic for connections that have already been
permitted.

For example:

```text
Trusted system
      |
      | permitted connection
      v
Lab system
      |
      | response
      v
Trusted system
```

The response can be allowed because it belongs to an established
connection.

However:

```text
Lab system
      |
      | NEW connection
      v
Trusted system
```

can still be blocked.

---

## Connection States

Important connection tracking states include:

| State | Meaning |
|---|---|
| `NEW` | Beginning of a new connection |
| `ESTABLISHED` | Traffic belonging to an existing connection |
| `RELATED` | New traffic associated with another valid connection |
| `INVALID` | Traffic that cannot be associated with a valid connection |

This makes it possible to build firewall rules based on connection state
instead of treating every packet independently.

---

## Lab Internet Access

The lab network requires limited Internet connectivity for functions such
as:

- DNS resolution
- HTTP access
- HTTPS access
- software updates
- selected security tooling

However, giving an Internet-facing cybersecurity system unrestricted
outbound access would increase risk.

For this reason, outbound traffic from the lab is restricted separately
from NAT.

Current policy:

```text
ESTABLISHED / RELATED               ACCEPT

vmbrlab -> vmbr1                    DROP
vmbrlab -> vmbr2                    DROP

DNS UDP/53 -> 1.1.1.1               ACCEPT
DNS TCP/53 -> 1.1.1.1               ACCEPT

HTTP TCP/80                         ACCEPT
HTTPS TCP/443                       ACCEPT

Other NEW outbound traffic          LOG
Other outbound traffic              DROP
```

This provides the lab with the network access currently required while
reducing the ability of a compromised system to initiate arbitrary
outbound connections.

---

## NAT Is Not a Security Policy

The lab network has a MASQUERADE rule that allows private addresses to
communicate through the WAN interface.

However:

```text
NAT != Firewall
```

NAT controls address translation.

The firewall controls whether traffic is permitted.

Therefore, having:

```text
<lab-subnet> -> MASQUERADE -> vmbr0
```

does not automatically give every lab workload unrestricted Internet
access.

Firewall rules are evaluated separately.

---

## Egress Filtering

Egress filtering controls connections initiated from the security lab.

Current permitted outbound services include:

```text
DNS
HTTP
HTTPS
```

DNS is restricted to:

```text
1.1.1.1
```

for both:

```text
UDP/53
TCP/53
```

HTTP and HTTPS are currently permitted on:

```text
TCP/80
TCP/443
```

Other new outbound connections are logged before being dropped.

---

## Blocked Traffic Logging

Unexpected new outbound connections from the lab are logged using the
prefix:

```text
TPOT-EGRESS:
```

The logging rule is rate limited.

This is important because an Internet-facing honeypot can generate a large
amount of traffic.

Without rate limiting, firewall logging could produce excessive log
volume.

The purpose of this logging is to make unexpected egress behavior
observable while avoiding unnecessary log flooding.

---

## Defense in Depth

The security model does not rely on a single control.

Instead, several controls are combined:

```text
Network segmentation
        +
Firewall isolation
        +
Stateful connection tracking
        +
Restricted egress
        +
NAT
        +
Logging
        +
Management separation
```

If one layer is misconfigured, other controls can still reduce the impact
of the failure.

---

## Proxmox Firewall Networking

When the Proxmox firewall is enabled for virtual machines, additional
virtual network interfaces are introduced.

Common examples include:

```text
fwbr*
fwpr*
fwln*
```

These interfaces form part of the Proxmox firewall path.

A VM connected to `vmbrlab` may therefore appear at runtime through an
interface such as:

```text
fwpr<vmid>p0
```

even though the static bridge configuration contains:

```text
bridge-ports none
```

This is expected behavior and does not mean that the lab bridge is
connected to a physical NIC.

---

## Connection Tracking Zone

The Proxmox firewall bridge architecture also affects Linux connection
tracking.

The environment uses:

```bash
iptables -t raw -I PREROUTING \
-i fwbr+ \
-j CT --zone 1
```

This assigns traffic arriving through Proxmox firewall bridges to
connection tracking zone `1`.

The rule became necessary during troubleshooting when NAT did not operate
correctly with the Proxmox VM firewall enabled.

More information is available in:

[Routing and NAT](routing-and-nat.md)

and:

[Troubleshooting](troubleshooting.md)

---

## Management Plane

Proxmox management is kept separate from intentionally exposed lab
services.

Administrative access is performed from the trusted network.

### SSH

Administrative SSH access is available through the trusted LAN.

Example:

```bash
ssh <admin-user>@<trusted-lan-gateway>
```

Direct root SSH login is disabled.

### Proxmox Web Interface

The Proxmox web interface is available internally on:

```text
TCP/8006
```

Example:

```text
https://<trusted-lan-gateway>:8006
```

The Proxmox management interface is not intentionally forwarded as a
honeypot service.

---

## Honeypot Security Boundary

The T-Pot virtual machine currently resides at:

```text
<honeypot-ip>
```

inside:

```text
vmbrlab
```

The Proxmox network provides the security boundary around the honeypot.

Conceptually:

```text
                    Internet
                       |
                       v
                     vmbr0
                       |
                 Proxmox Firewall
                       |
                       v
                    vmbrlab
                       |
                       v
                  T-Pot VM
               <honeypot-ip>

                     X
                  /     \
                 /       \
              vmbr1     vmbr2
             Trusted    Internal
```

Internet-facing honeypot services require additional DNAT and ingress
firewall rules.

Those rules are intentionally documented in the separate T-Pot
repository rather than duplicated here.

---

## Ingress vs Egress

It is useful to distinguish the two directions.

### Ingress

Traffic entering the lab:

```text
Internet -> Proxmox -> vmbrlab
```

Example use:

```text
Internet -> honeypot service
```

### Egress

Traffic initiated by the lab:

```text
vmbrlab -> Proxmox -> Internet
```

Example use:

```text
T-Pot -> HTTPS update server
```

The two directions use separate firewall policies.

This is important because a service may need to receive Internet traffic
without being allowed unrestricted outbound connectivity.

---

## Persistent Firewall Rules

Firewall rules that are part of the permanent network architecture are
attached to the Proxmox interface configuration using:

```text
post-up
post-down
```

Example concept:

```text
post-up   -> create rule
post-down -> remove rule
```

Temporary rules are first tested in the running firewall.

Only after successful verification are required rules added to the
persistent configuration.

This reduces the risk of making an incorrect firewall rule permanent.

---

## Verification

Firewall state can be inspected using:

```bash
sudo iptables -L FORWARD -v -n --line-numbers
```

NAT:

```bash
sudo iptables -t nat -L -v -n --line-numbers
```

Connection tracking rules:

```bash
sudo iptables -t raw -L PREROUTING -v -n --line-numbers
```

Network configuration:

```bash
sudo ifquery --check -a
```

Useful tests include:

```text
Trusted network -> Internet
Lab network -> permitted Internet service
Lab network -> trusted LAN
Lab network -> Wi-Fi network
```

The expected isolation tests are:

```text
vmbrlab -> vmbr1    BLOCKED
vmbrlab -> vmbr2    BLOCKED
```

---

## Packet Counters

Firewall packet counters are used when troubleshooting.

Example:

```bash
sudo iptables -L FORWARD -v -n --line-numbers
```

Counters help answer questions such as:

```text
Did the packet reach this rule?

Did the rule match?

Was traffic accepted or dropped?
```

This is often more useful than only checking whether a rule exists.

---

## Security Assumptions

The current architecture assumes:

- The Proxmox host itself remains trusted
- The lab network may contain vulnerable or compromised systems
- Trusted LAN devices should not be reachable through new connections
  initiated by the lab
- Internet-facing honeypot services may receive hostile traffic
- Firewall rules must therefore protect both internal systems and external
  systems from misuse of the lab environment

The honeypot is intentionally exposed to hostile traffic, but the
surrounding infrastructure should not be treated as expendable.

---

## Current Limitations

The current environment still has areas that can be improved.

Examples include:

- Firewall rules are maintained directly with iptables
- There is no dedicated physical management network
- Network policy is not yet managed using infrastructure automation
- Firewall configuration will become more complex as additional networks
  are added
- Monitoring of the Proxmox host itself can be expanded

These limitations are documented so that future changes can be made
deliberately instead of allowing the environment to grow without a clear
design.

---

## Future Improvements

Possible security improvements include:

- Dedicated management VLAN or network
- VLAN-based segmentation
- Centralized firewall policy management
- Configuration management with Ansible
- Automated configuration backups
- Additional Proxmox host monitoring
- Centralized firewall log collection
- SIEM integration
- Improved alerting
- Regular firewall rule review
- Documented recovery procedures

---

## What I Learned

Building the environment demonstrated several practical security
principles:

- Network segmentation reduces the impact of a compromised system.
- Routing capability should not automatically imply permission to
  communicate.
- Stateful firewalling allows return traffic without allowing arbitrary
  new connections.
- NAT and firewalling solve different problems.
- Egress filtering is important for intentionally vulnerable systems.
- Management interfaces should be separated from intentionally exposed
  services.
- Packet counters and logs are useful when verifying firewall behavior.
- Security controls should be tested before being made persistent.
- Isolation should be verified from the potentially hostile side of the
  security boundary.

---

## Related Documentation

- [Network Architecture](architecture.md)
- [Network Configuration](network-configuration.md)
- [Routing and NAT](routing-and-nat.md)
- [Troubleshooting](troubleshooting.md)
- [Implementation Notes](implementation-notes.md)
