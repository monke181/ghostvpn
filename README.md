# ghostvpn

A pocket-sized Raspberry Pi 4 travel router that turns any public WiFi (hotels, airports, cafes, school networks) into a secure private network. All traffic from connected devices is encrypted through a WireGuard VPN tunnel to a cloud VPS before reaching the internet.

## What This Is

This project is a portable networking device built from a Raspberry Pi 4 that:

- Connects to public/untrusted WiFi networks (handles captive portal logins)
- Broadcasts its own private, password-protected WiFi hotspot
- Routes all hotspot traffic through an encrypted WireGuard tunnel
- Exits to the internet via a cloud VPS with a static IP
- Runs on a rechargeable LiPo battery with safe shutdown circuitry
- Auto-starts everything on boot — no laptop or monitor required

## Why

Public WiFi networks are inherently insecure. Anyone else on the same network can potentially intercept unencrypted traffic, and you have no control over what the network operator logs or monitors. This device creates a "home-like" secure bubble around your devices no matter where you are.

## High-Level Architecture

```
[Your Devices: Laptop/Phone]
        |
   (Private Encrypted Hotspot - 5GHz)
        |
[Raspberry Pi 4 Travel Router]
        |
   (Public WiFi - hotel/cafe/school)
        |
   Captive Portal (manual login if required)
        |
   Encrypted WireGuard Tunnel
        |
[Cloud VPS - WireGuard Server]
        |
     Internet
```

## Hardware

- Raspberry Pi 4
- TP-Link Archer T2U Plus USB WiFi Adapter (RTL8821AU chipset) — used for the hotspot
- 3.7V 3000mAh LiPo battery (654065 size)
- TP4056 USB-C charging module
- MT3608 boost converter (set to 5V output)
- SPDT slide switch (main power)
- Momentary push button (safe shutdown trigger)
- LED + 330Ω resistor (power/ready indicator)

## Cloud Component

A small VPS (DigitalOcean, $6/month droplet) runs as the WireGuard server endpoint, providing:
- A static public IP address
- Faster upload/download speeds than a typical home connection
- Always-on availability

## Software Stack

- Raspberry Pi OS Lite (64-bit)
- `hostapd` — WiFi hotspot (5GHz, AP mode)
- `dnsmasq` — DHCP/DNS for hotspot clients
- `iptables` / `netfilter-persistent` — NAT and traffic forwarding
- `WireGuard` — VPN tunnel (client on Pi, server on VPS)
- `RaspAP` — web dashboard for managing WiFi connections without SSH
- Custom Python/systemd service for safe shutdown via GPIO button

## Security Model

**Protected against:**
- Packet sniffing on public WiFi
- Local network MITM attacks
- Other devices on the same public network
- Exposure of browsing activity to the local network operator

- Network operators seeing that *a* device is connected (device presence is still visible — only the contents of its traffic are hidden)


## 3D DESIGN
<img width="1482" height="701" alt="ghostvpn3" src="https://github.com/user-attachments/assets/0af1b1d6-415b-4aeb-a1d6-898616bef86f" />
<img width="1482" height="701" alt="ghostvpn2" src="https://github.com/user-attachments/assets/6fbb6cf8-2f28-47e0-ac11-14a77e62be5c" />
<img width="1482" height="701" alt="ghostvpn1" src="https://github.com/user-attachments/assets/788f7f47-dc54-4133-bccb-7716e709f95d" />


## NOTES:

Project last updated on 6/27/26. There may be some design inconsistencies and / or unfitting parts. 

Design is provided "as is". 

See [SETUP.md](./SETUP.md) for full build instructions.


