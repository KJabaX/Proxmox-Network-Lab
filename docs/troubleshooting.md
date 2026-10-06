# Troubleshooting Notes

This document contains real problems encountered while building and
maintaining the Proxmox network environment. Environment-specific identifiers
and addresses are replaced with generic placeholders in this public version.

The goal is to preserve not only the final solution, but also the symptoms,
investigation process, root cause and verification steps.

---

## 1. Virtual Machine Had No Internet Access

### Symptoms

A virtual machine connected to an internal Proxmox bridge could communicate
with the local gateway, but Internet connectivity failed.

Testing resulted in packet loss even though the VM had:

- a valid IP address
- a default gateway
- a Proxmox bridge connection
- a NAT rule

This showed that having a bridge and a MASQUERADE rule alone was not enough.

---

### Investigation

The following areas were checked:

```bash
ip -br addr
ip route
sysctl net.ipv4.ip_forward
```

Firewall forwarding:

```bash
sudo iptables -L FORWARD -v -n --line-numbers
```

NAT:

```bash
sudo iptables -t nat -L POSTROUTING -v -n --line-numbers
```

Proxmox firewall networking:

```bash
ip link
bridge link
```

Connection tracking:

```bash
sudo iptables -t raw -L PREROUTING -v -n --line-numbers
```

---

### Cause

The Proxmox VM firewall introduces additional virtual firewall bridges,
typically using interface names beginning with:

```text
fwbr
```

Traffic passing through these interfaces required a separate Linux
connection tracking zone.

Without this, NAT and connection tracking did not behave correctly for the
VM traffic.

---

### Fix

The following raw-table rule was added:

```bash
sudo iptables -t raw -I PREROUTING \
-i fwbr+ \
-j CT --zone 1
```

Explanation:

```text
-t raw
```

Uses the raw iptables table.

```text
-I PREROUTING
```

Inserts the rule before the routing decision.

```text
-i fwbr+
```

Matches Proxmox firewall bridge interfaces whose names begin with `fwbr`.

```text
-j CT --zone 1
```

Assigns matching connections to conntrack zone `1`.

---

### Verification

Check the rule:

```bash
sudo iptables -t raw -L PREROUTING -v -n --line-numbers
```

Expected result includes:

```text
CT ... fwbr+ ... CT zone 1
```

Internet connectivity was then tested again from the VM.

---

### What I Learned

NAT depends on Linux connection tracking.

Proxmox firewall-enabled VM networking adds additional bridge interfaces,
which can affect normal Linux NAT behavior.

When debugging VM connectivity, checking only the NAT table is not enough.

The complete path may include:

```text
VM
 |
 v
tap interface
 |
 v
Proxmox firewall bridge
 |
 v
Linux bridge
 |
 v
FORWARD
 |
 v
NAT
 |
 v
WAN
```

---

## 2. Internal Network Had NAT but Traffic Was Still Blocked

### Symptoms

An internal network had a valid MASQUERADE rule, but traffic still could not
reach the Internet.

Example:

```text
<lab-subnet> -> MASQUERADE -> vmbr0
```

The NAT rule existed, but connectivity still failed.

---

### Cause

NAT and firewall forwarding are separate operations.

A packet may be translated correctly by NAT while still being blocked by
the Linux `FORWARD` chain.

Conceptually:

```text
Routing
   |
   v
Firewall
   |
   v
NAT
```

A valid NAT rule does not automatically create firewall permission.

---

### Investigation

Check forwarding:

```bash
sudo iptables -L FORWARD -v -n --line-numbers
```

Check NAT:

```bash
sudo iptables -t nat -L POSTROUTING -v -n --line-numbers
```

Check IPv4 forwarding:

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

---

### Fix

The required forwarding rules were added separately from the NAT rules.

The environment therefore uses both:

```text
FORWARD rules
```

and:

```text
POSTROUTING MASQUERADE rules
```

---

### What I Learned

A useful mental model is:

```text
Routing
    determines where the packet should go

Firewall
    determines whether the packet may go there

NAT
    modifies addressing when required
```

These functions should be troubleshooted independently.

---

## 3. VPN Policy Routing Interfered With Access to the Lab Network

### Symptoms

The Arch Linux workstation was connected to the trusted LAN, but access to
a system in the lab network did not behave as expected.

The workstation used:

```text
<trusted-lan-subnet>
```

while the lab network used:

```text
<lab-subnet>
```

The workstation also had Mullvad VPN active through:

```text
wg0
```

Traffic intended for the local lab network was being affected by the VPN
routing policy.

---

### Investigation

The normal routing table was checked with:

```bash
ip route
```

The expected local path was through:

```text
<trusted-lan-gateway>
```

However, VPN software can use Linux policy routing in addition to the main
routing table.

Useful commands include:

```bash
ip rule
```

and:

```bash
ip route show table all
```

A specific destination can be tested using:

```bash
ip route get <honeypot-ip>
```

---

### Cause

Mullvad used policy routing associated with the WireGuard interface.

This meant that looking only at:

```bash
ip route
```

did not necessarily show the full routing decision.

Traffic toward the private lab network could be captured by VPN policy
routing instead of following the expected local route.

---

### Fix

Local lab traffic was excluded from the VPN path so that traffic toward:

```text
<lab-subnet>
```

used the normal local gateway instead.

The intended traffic path is:

```text
Arch workstation
<trusted-client-ip>
       |
       v
<trusted-lan-gateway>
Proxmox
       |
       v
vmbrlab
       |
       v
<lab-subnet>
```

rather than:

```text
Workstation
     |
     v
Mullvad / wg0
     |
     v
VPN tunnel
```

---

### Verification

Useful commands:

```bash
ip rule
ip route
ip route get <honeypot-ip>
```

The result should show that the lab subnet uses the local network path
rather than the VPN tunnel.

---

### What I Learned

Linux can have multiple routing tables.

The command:

```bash
ip route
```

usually shows the main routing table, but VPN software may use:

```text
policy routing
```

through:

```bash
ip rule
```

When routing behavior appears inconsistent with the main routing table,
policy routing should also be inspected.

---

## 4. `vmbrlab` Showed `fwpr<vmid>p0` Even Though `bridge-ports none` Was Configured

### Symptoms

The static configuration for the lab bridge contains:

```text
bridge-ports none
```

However, a configuration check showed a runtime bridge member similar to:

```text
bridge-ports (fwpr<vmid>p0)
```

At first this looked as if `vmbrlab` was no longer an isolated
virtual-only bridge.

---

### Investigation

Static configuration:

```bash
grep -A10 "iface vmbrlab" /etc/network/interfaces
```

Runtime bridge state:

```bash
bridge link
```

Configuration validation:

```bash
sudo ifquery --check -a
```

---

### Cause

The interface:

```text
fwpr<vmid>p0
```

is part of the Proxmox firewall networking path.

Proxmox creates additional virtual interfaces when the VM firewall is
enabled.

Common prefixes include:

```text
fwbr
fwpr
fwln
tap
```

These are virtual networking components.

They are not physical Ethernet interfaces.

---

### Result

The configuration:

```text
bridge-ports none
```

is still correct.

It means that no physical interface such as:

```text
eno1
eno2
eno3
eno4
```

is directly assigned to `vmbrlab`.

The presence of a runtime Proxmox firewall interface does not change the
intended virtual-only design.

---

### What I Learned

Static bridge configuration and runtime bridge membership are not always
identical.

Proxmox can dynamically attach virtual firewall interfaces to a bridge.

Therefore:

```text
bridge-ports none
```

does not mean that the bridge will always appear completely empty at
runtime.

It means that no physical bridge port has been configured for it.

---

## 5. `ifquery --check -a` Showed `[]` for Firewall Commands

### Symptoms

Running:

```bash
sudo ifquery --check -a
```

showed interface settings with:

```text
[pass]
```

but `post-up` and `post-down` commands appeared with:

```text
[]
```

Example:

```text
post-up iptables ... []
```

This initially looked like the firewall command had failed validation.

---

### Cause

`ifquery` can validate interface attributes such as:

```text
address
bridge-ports
bridge-stp
bridge-fd
```

against the running interface state.

Commands such as:

```text
post-up
post-down
```

are lifecycle hooks.

They are not state attributes that can be compared in the same way.

Therefore:

```text
[]
```

does not by itself indicate that the iptables rule failed.

---

### Correct Verification

Firewall rules must be checked directly.

FORWARD:

```bash
sudo iptables -L FORWARD -v -n --line-numbers
```

NAT:

```bash
sudo iptables -t nat -L -v -n --line-numbers
```

Raw table:

```bash
sudo iptables -t raw -L PREROUTING -v -n --line-numbers
```

---

### What I Learned

Configuration parsing and runtime firewall verification are two different
checks.

Use:

```text
ifquery
```

for network-interface configuration.

Use:

```text
iptables
```

for actual firewall state.

---

## 6. Testing Firewall Rules Before Making Them Persistent

### Problem

Adding firewall rules directly to:

```text
/etc/network/interfaces
```

before testing them can make troubleshooting harder and may cause incorrect
rules to return after reboot.

---

### Workflow Used

New firewall behavior is first tested in the running system.

Example:

```bash
sudo iptables ...
```

Then the result is inspected:

```bash
sudo iptables -L -v -n --line-numbers
```

or:

```bash
sudo iptables -t nat -L -v -n --line-numbers
```

Connectivity and isolation are tested.

Only after the rule works as intended is it added to:

```text
/etc/network/interfaces
```

using:

```text
post-up
```

and the corresponding cleanup command:

```text
post-down
```

---

### Why This Helps

The workflow separates two questions:

```text
Does the rule work?
```

from:

```text
Will the rule survive a reboot?
```

Debugging these separately makes failures easier to understand.

---

## 7. Avoiding Unnecessary Network Restarts

### Problem

Restarting networking on a remotely managed Proxmox server can interrupt
the management connection.

A configuration mistake may also make the host temporarily unreachable.

---

### Safer Workflow

Before changing interface state:

```bash
sudo ifquery --check -a
```

Inspect the configuration:

```bash
cat /etc/network/interfaces
```

Inspect runtime state:

```bash
ip -br addr
ip route
bridge link
```

For firewall-only changes, inspect or modify the firewall directly instead
of restarting the entire networking stack.

---

### What I Learned

A network restart should not be used simply as a configuration validation
tool.

When working on a remotely managed hypervisor, changes should be small,
testable and reversible.

---

## 8. Packet Counters as a Troubleshooting Tool

iptables counters are useful for determining where traffic stops.

Example:

```bash
sudo iptables -L FORWARD -v -n --line-numbers
```

Output includes:

```text
pkts
bytes
```

If the counter for an expected rule increases, the packet is reaching and
matching that rule.

If it does not increase, the problem is probably earlier in the packet
path.

The same technique can be used with NAT:

```bash
sudo iptables -t nat -L PREROUTING -v -n --line-numbers
```

and:

```bash
sudo iptables -t nat -L POSTROUTING -v -n --line-numbers
```

---

## Packet Troubleshooting Strategy

When connectivity fails, troubleshoot the path in order rather than
changing several firewall rules at once.

```text
Source
  |
  v
IP configuration
  |
  v
Default gateway
  |
  v
Routing
  |
  v
Firewall
  |
  v
NAT
  |
  v
Destination
```

For a VM attempting to reach the Internet:

```text
1. Does the VM have the correct IP?
2. Does it have the correct default gateway?
3. Can it reach the Proxmox gateway?
4. Is IPv4 forwarding enabled?
5. Does the packet reach the FORWARD chain?
6. Does the expected firewall rule counter increase?
7. Does the POSTROUTING MASQUERADE rule match?
8. Is conntrack working correctly?
9. Does vmbr0 have upstream connectivity?
```

Useful commands:

```bash
ip -br addr
ip route
ip rule
sysctl net.ipv4.ip_forward
sudo iptables -L FORWARD -v -n --line-numbers
sudo iptables -t nat -L -v -n --line-numbers
sudo iptables -t raw -L PREROUTING -v -n --line-numbers
bridge link
```

---

## Troubleshooting Principles

The most useful lessons from building the network have been:

1. Change one thing at a time.
2. Verify runtime state instead of assuming configuration was applied.
3. Use packet counters to determine where traffic stops.
4. Check routing before blaming the firewall.
5. Remember that NAT and firewalling are separate.
6. Check policy routing when VPN software is involved.
7. Understand Proxmox virtual firewall interfaces before treating them as
   unexpected devices.
8. Test temporary firewall rules before making them persistent.
9. Avoid unnecessary network restarts on a remotely managed hypervisor.
10. Document the final cause and solution after fixing a problem.

---

## Template for Future Problems

New troubleshooting cases can be added using this structure:

```markdown
## Problem Title

### Symptoms

What happened?

### Expected Behavior

What should have happened?

### Investigation

What commands and tests were used?

### Cause

What was the actual root cause?

### Fix

What was changed?

### Verification

How was the fix confirmed?

### What I Learned

What should I remember next time?
```

---

## Related Documentation

- [Network Architecture](architecture.md)
- [Network Configuration](network-configuration.md)
- [Routing and NAT](routing-and-nat.md)
- [Security Design](security.md)
- [Implementation Notes](implementation-notes.md)