# Changelog

This file records major changes to the Proxmox network architecture,
configuration and documentation.

## 2026-10

### Documentation

- Reorganized the repository structure
- Added a portfolio-oriented `README.md`
- Split technical documentation into separate files
- Added architecture, network configuration, routing, security and
  troubleshooting documentation
- Added sanitized example configuration files

## 2026-09

### Cybersecurity Lab

- Added the isolated `vmbrlab` network
- Configured the `<lab-subnet>` lab subnet
- Added routing and NAT for the lab network
- Added isolation between the lab and trusted networks
- Added connection tracking support for Proxmox firewall bridges
- Deployed the T-Pot environment on the lab network

## 2025-09

### Initial Network

- Configured the WAN bridge `vmbr0`
- Configured the main LAN on `vmbr1`
- Configured the Wi-Fi network on `vmbr2`
- Enabled IPv4 forwarding
- Configured NAT for internal networks