# Proxmox Network Lab

## 1. Introduction

This project documents my Proxmox network and cybersecurity lab built around
a Proxmox VE server.

The Proxmox server acts as a virtualization host, network gateway and
platform for experimenting with networking and cybersecurity technologies.

---

## 2. Environment

> Public version: hostnames, usernames, management addresses, and exact private addressing are intentionally sanitized.

### Server

- Hardware: HP ProLiant DL380e G8
- CPU: Intel Xeon E5-2450L
- Proxmox VE
- Hostname: `<proxmox-host>`
- Four physical Ethernet interfaces: `eno1` - `eno4`

### Client

- Arch Linux desktop
- Connected to the main LAN
- IP address: `<trusted-client-ip>`

---

## 3. Network Topology

Sanitized network topology:

                    Internet
                       |
                       |
                 eno1 / vmbr0
                 Public / WAN
                       |
              +------------------+
              |  Proxmox Server  |
              |   "<proxmox-host>"   |
              +------------------+
                 /       |       \
                /        |        \
        eno2/vmbr1   eno3/vmbr2   vmbrlab
            |            |           |
    <trusted-lan-subnet> <wifi-subnet> <lab-subnet>
            |            |           |
      Arch Desktop     Wi-Fi      Lab Network
    <trusted-client-ip>               (virtual only)

`vmbr0` provides the upstream Internet connection.

`vmbr1` is the primary wired LAN.

`vmbr2` provides a separate network used for Wi-Fi connectivity.

`vmbrlab` is an isolated virtual network with no physical bridge port.
It is intended for lab and cybersecurity environments.

---

## 4. Network Interfaces

### vmbr0 - WAN

Physical interface:

    eno1

Configuration:

    auto vmbr0
    iface vmbr0 inet dhcp
        bridge-ports eno1
        bridge-stp off
        bridge-fd 0

`vmbr0` receives its address using DHCP from the upstream network.

---

### vmbr1 - Main LAN

Physical interface:

    eno2

Network:

    <trusted-lan-subnet>

Gateway:

    <trusted-lan-gateway>

Configuration:

    auto vmbr1
    iface vmbr1 inet static
        address <trusted-lan-gateway>/24
        bridge-ports eno2
        bridge-stp off
        bridge-fd 0

Traffic from this network is NATed through `vmbr0`.

The Arch Linux desktop is connected to this network.

---

### vmbr2 - Wi-Fi Network

Physical interface:

    eno3

Network:

    <wifi-subnet>

Gateway:

    <wifi-gateway>

Configuration:

    auto vmbr2
    iface vmbr2 inet static
        address <wifi-gateway>/24
        bridge-ports eno3
        bridge-stp off
        bridge-fd 0

Traffic from this network is also NATed through `vmbr0`.

This network is used to provide connectivity to the Wi-Fi network.

---

### vmbrlab - Isolated Lab Network

Physical interface:

    None

Network:

    <lab-subnet>

Gateway:

    <lab-gateway>

Configuration:

    auto vmbrlab
    iface vmbrlab inet static
        address <lab-gateway>/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0

Unlike `vmbr1` and `vmbr2`, this bridge is not connected to a physical
Ethernet interface.

It is intended to provide an isolated network for virtual machines used
in cybersecurity experiments.

---

## 5. Network Summary

| Bridge | Physical NIC | Network | Purpose |
|---|---|---|---|
| `vmbr0` | `eno1` | DHCP / WAN | Internet |
| `vmbr1` | `eno2` | `<trusted-lan-subnet>` | Main LAN |
| `vmbr2` | `eno3` | `<wifi-subnet>` | Wi-Fi network |
| `vmbrlab` | None | `<lab-subnet>` | Isolated lab |
| - | `eno4` | - | Currently unused |

---

## 6. Routing and NAT

The Proxmox host performs routing between the internal networks and the
upstream network.

IPv4 forwarding is enabled on the Proxmox host.

NAT is configured for the internal networks:

    <trusted-lan-subnet> -> vmbr0
    <wifi-subnet> -> vmbr0
    <lab-subnet>  -> vmbr0

The following MASQUERADE rules allow these private networks to access
external networks through `vmbr0`.

For the T-Pot lab network, outbound access is additionally restricted
with firewall rules. The honeypot is not given unrestricted outbound
access even though NAT is configured.
---

## 7. T-Pot Honeypot Network

T-Pot is deployed as a virtual machine inside the isolated `vmbrlab`
network.

Network configuration:

    T-Pot VM:       <honeypot-ip>
    Network:        <lab-subnet>
    Gateway:        <lab-gateway>
    Proxmox bridge: vmbrlab

The T-Pot VM is intentionally separated from the trusted `vmbr1` and
`vmbr2` networks.

### Incoming Traffic

Selected honeypot services are exposed through the Proxmox host using
DNAT.

Documented honeypot services:

| Port | Protocol | Honeypot | Service |
|---|---|---|---|
| 22 | TCP | Cowrie | SSH |
| 23 | TCP | Cowrie | Telnet |
| 25 | TCP | Mailoney | SMTP |
| 69 | UDP | Dionaea | TFTP |
| 80 | TCP | Snare | HTTP |
| 135 | TCP | Dionaea | RPC |
| 443 | TCP | h0neytr4p | HTTPS |
| 445 | TCP | Dionaea | SMB |
| 587 | TCP | Mailoney | SMTP Submission |
| 1433 | TCP | Dionaea | MSSQL |
| 1723 | TCP | Dionaea | PPTP |
| 1883 | TCP | Dionaea | MQTT |
| 3306 | TCP | Dionaea | MySQL |
| 3389 | TCP | RDP honeypot | RDP |
| 5060 | TCP/UDP | SentryPeer | SIP |
| 5432 | TCP | Heralding | PostgreSQL |
| 6379 | TCP | RedisHoneypot | Redis |
| 27017 | TCP | Dionaea | MongoDB |

Only explicitly selected honeypot services are forwarded from `vmbr0`.
Other incoming traffic to the T-Pot VM is dropped by the
`TPOT-INGRESS` chain.
Incoming traffic follows this path:

                    Internet
                       |
                       v
                     vmbr0
                       |
                  NAT PREROUTING
                     (DNAT)
                       |
                       v
                  FORWARD chain
                       |
                       v
                  TPOT-INGRESS
                       |
                 +-----+-----+
                 |           |
              allowed       other
               ports        ports
                 |           |
              ACCEPT        DROP
                 |
                 v
                   vmbrlab
                       |
                       v
                <honeypot-ip>
                     T-Pot

For example, an incoming HTTPS connection is translated as:

    Proxmox TCP/443 -> <honeypot-ip>:443

DNAT determines where the incoming connection is forwarded, while the
`TPOT-INGRESS` chain determines whether the connection is allowed.

### TPOT-INGRESS Firewall Chain

A dedicated iptables chain called `TPOT-INGRESS` controls incoming
connections to the honeypot.

Only explicitly selected TCP and UDP honeypot services are accepted.
All other traffic reaching the end of the chain is dropped.

The basic structure is:

    Internet
       |
       v
    vmbr0
       |
       v
    DNAT
       |
       v
    FORWARD
       |
       v
    TPOT-INGRESS
       |
       +-- selected honeypot ports -> ACCEPT
       |
       +-- everything else -> DROP

The final DROP rule must remain after all ACCEPT rules.

### DNAT

DNAT rules in the `nat` table redirect selected connections arriving
through `vmbr0` to the T-Pot VM at `<honeypot-ip>`.

For example:

    vmbr0 TCP/22   -> <honeypot-ip>:22
    vmbr0 TCP/443  -> <honeypot-ip>:443
    vmbr0 TCP/445  -> <honeypot-ip>:445
    vmbr0 UDP/5060 -> <honeypot-ip>:5060

Both DNAT and an ACCEPT rule are required.

DNAT answers:

    Where should the connection go?

TPOT-INGRESS answers:

    Is the connection allowed?

### Connection Tracking

Connection tracking is used to track active connections.

Established and related traffic is accepted:

    ESTABLISHED,RELATED -> ACCEPT

This allows response traffic for connections that have already been
permitted.

### Honeypot Isolation

The honeypot network is prevented from initiating connections toward the
trusted networks:

    vmbrlab -> vmbr1 -> DROP
    vmbrlab -> vmbr2 -> DROP

This prevents the T-Pot VM from using the lab network to access systems
on the trusted LANs.

### Restricted Outbound Traffic

Outbound traffic from `vmbrlab` to `vmbr0` is restricted.

New connections are currently allowed for:

    DNS UDP/53 -> 1.1.1.1
    DNS TCP/53 -> 1.1.1.1
    HTTP TCP/80
    HTTPS TCP/443

Other new outbound connections are logged with:

    TPOT-EGRESS:

and then dropped.

This reduces the ability of a compromised honeypot to initiate arbitrary
connections to external systems.

### Persistent Firewall Configuration

The firewall and NAT rules are configured using `post-up` and
`post-down` commands in:

    /etc/network/interfaces

Rules added manually with `iptables` affect the currently running
firewall. After testing, the required rules are added to
`/etc/network/interfaces` so that they are recreated when the network
interface is brought up.

When exposing a new honeypot service, the workflow is:

1. Verify that T-Pot is listening on the port.
2. Add a DNAT rule.
3. Add an ACCEPT rule to `TPOT-INGRESS` before the final DROP rule.
4. Verify the rules and packet counters.
5. Add the tested configuration to `/etc/network/interfaces`.

## 8. Remote Management

### SSH

The Proxmox host runs OpenSSH on:

    TCP 22

SSH listens on IPv4 and IPv6.

Security configuration:

- Root SSH login disabled
- Administrative user: `<admin-user>`
- SSH access available from the management network

Example:

    ssh <admin-user>@<trusted-lan-gateway>

### Proxmox Web Interface

The Proxmox management interface is available on:

    TCP 8006

From the main LAN:

    https://<trusted-lan-gateway>:8006

### Remote Administration

Remote administration is designed so that Proxmox management services do not
need to be exposed directly to the public Internet.

Environment-specific remote-access tooling, account details, and addressing
are intentionally omitted from this public documentation.

---

## 9. Security

Current security measures include:

- Separate network bridges for different purposes
- Dedicated internal LAN subnets
- Isolated T-Pot network on `vmbrlab`
- T-Pot blocked from accessing `vmbr1` and `vmbr2`
- Dedicated `TPOT-INGRESS` firewall chain
- Only selected honeypot ports exposed through DNAT
- Restricted outbound traffic from the honeypot network
- Logging of blocked T-Pot outbound connection attempts
- Root SSH login disabled
- Dedicated administrative SSH account
- Proxmox management services not intentionally exposed as honeypot services
- NAT for internal networks
- Remote management kept separate from honeypot services

---

## 10. Future Development

Planned improvements include:

- Continue improving secure remote administration
- Continue hardening administrative authentication
- Monitor and analyze events collected by the exposed honeypot services
- Improve and simplify firewall rule management
- Forward and analyze security events in a SIEM
- Expand attack investigation documentation
- Configure automated backups

### Current Honeypot Architecture

                    Internet
                       |
                     vmbr0
                       |
                  DNAT / Firewall
                       |
                 TPOT-INGRESS
                       |
                     vmbrlab
                       |
                 <honeypot-ip>
                     T-Pot

              X----------------X
              |                |
            vmbr1            vmbr2
         Trusted LAN      Trusted LAN

        T-Pot access to trusted
        networks is blocked.
