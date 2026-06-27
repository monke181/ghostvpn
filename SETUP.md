# Setup Instructions

This guide covers the full software setup for both the Raspberry Pi (travel router) and the cloud VPS (WireGuard server).

## Prerequisites

- Raspberry Pi 4 with Raspberry Pi OS Lite (64-bit) flashed via Raspberry Pi Imager
  - Enable SSH and set a username/password during flashing (OS Customisation settings)
- TP-Link Archer T2U Plus (or other RTL8821AU-based) USB WiFi adapter
- A VPS with Ubuntu 22.04 (DigitalOcean, Vultr, etc.) — needs a static public IPv4 address

> **Note on kernel version:** The RTL8821AU driver is built into the Linux kernel as of version 6.14+. If `iwconfig` doesn't show a second wireless interface after plugging in the adapter, run `sudo rpi-update` and reboot before continuing.

---

## Part 1: Raspberry Pi — WiFi Driver Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git dkms bc build-essential

# Load the in-kernel RTL8821AU driver
sudo modprobe rtw88_8821au

# Make it persistent across reboots
echo "rtw88_8821au" | sudo tee /etc/modules-load.d/rtw88_8821au.conf

# Verify - you should now see wlan1
iwconfig
```

---

## Part 2: Raspberry Pi — Hotspot Setup (hostapd + dnsmasq)

### Prevent NetworkManager from managing wlan1

```bash
sudo nano /etc/NetworkManager/NetworkManager.conf
```

Add to the bottom:
```ini
[keyfile]
unmanaged-devices=interface-name:wlan1
```

```bash
sudo systemctl restart NetworkManager
```

### Assign a static IP to wlan1

```bash
sudo ip addr add 192.168.4.1/24 dev wlan1
sudo ip link set wlan1 up
```

Make it persistent with a systemd service:

```bash
sudo nano /etc/systemd/system/wlan1-static.service
```

```ini
[Unit]
Description=Set static IP for wlan1
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c '/sbin/ip addr add 192.168.4.1/24 dev wlan1 || true'
ExecStart=/sbin/ip link set wlan1 up
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable wlan1-static
sudo systemctl start wlan1-static
```

### Install and configure hostapd

```bash
sudo apt install -y hostapd dnsmasq
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq
```

```bash
sudo nano /etc/hostapd/hostapd.conf
```

```ini
interface=wlan1
driver=nl80211
ssid=YourNetworkName
hw_mode=a
channel=36
wmm_enabled=1
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=YourPassword
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
ieee80211n=1
ieee80211ac=1
ht_capab=[HT40+][SHORT-GI-20][SHORT-GI-40]
vht_capab=[SHORT-GI-80]
vht_oper_chwidth=1
vht_oper_centr_freq_seg0_idx=42
```

```bash
sudo nano /etc/default/hostapd
```

Set:
```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

```bash
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl start hostapd
```

### Configure dnsmasq

```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.backup
sudo nano /etc/dnsmasq.conf
```

```ini
interface=wlan1
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
domain=local
address=/gw.local/192.168.4.1
```

```bash
sudo systemctl enable dnsmasq
sudo systemctl start dnsmasq
```

---

## Part 3: Raspberry Pi — Routing & NAT

```bash
# Enable IP forwarding
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-forwarding.conf
sudo sysctl -p /etc/sysctl.d/99-forwarding.conf

# Install iptables
sudo apt install -y iptables iptables-persistent

# NAT rules (wlan0 = public WiFi uplink, wlan1 = hotspot)
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i wlan0 -o wlan1 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan1 -o wlan0 -j ACCEPT

sudo netfilter-persistent save
```

---

## Part 4: VPS — WireGuard Server Setup

SSH into your VPS:

```bash
apt update && apt upgrade -y
apt install -y wireguard

# Generate server keys
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey | tee /etc/wireguard/server_public.key
```

```bash
nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address = 10.0.0.1/24
SaveConfig = true
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

> Leave out the `[Peer]` section until you have the Pi's public key (Part 5), otherwise `wg-quick` will fail to start.

```bash
echo "net.ipv4.ip_forward=1" | tee /etc/sysctl.d/99-forwarding.conf
sysctl -p /etc/sysctl.d/99-forwarding.conf

systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

**Open the firewall:** allow inbound UDP port 51820 (or all UDP) in your VPS provider's firewall settings.

---

## Part 5: Raspberry Pi — WireGuard Client Setup

```bash
sudo apt install -y wireguard

# Generate client keys
wg genkey | sudo tee /etc/wireguard/client_private.key | wg pubkey | sudo tee /etc/wireguard/client_public.key
sudo cat /etc/wireguard/client_public.key
```

Copy the public key output, then on the **VPS** add it as a peer:

```bash
wg set wg0 peer <PI_PUBLIC_KEY> allowed-ips 10.0.0.2/32
wg-quick save wg0
```

Back on the **Pi**, create the client config:

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address = 10.0.0.2/24
PrivateKey = <PI_PRIVATE_KEY>

[Peer]
PublicKey = <VPS_PUBLIC_KEY>
Endpoint = <VPS_PUBLIC_IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

### Route hotspot traffic through the tunnel

```bash
sudo iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
sudo netfilter-persistent save
```

### Test the tunnel

```bash
ping -c 3 10.0.0.1   # should succeed - VPS over the tunnel
sudo wg show         # should show a recent handshake
```

Connect a device to the Pi's hotspot and check [whatismyip.com](https://whatismyip.com) — it should show the VPS's IP address.

---

## Part 6: RaspAP (WiFi Management Dashboard)

Allows connecting the Pi to new WiFi networks (e.g. hotel/school WiFi) from a browser, without SSH.

```bash
curl -sL https://install.raspap.com | bash
```

Accept the default prompts. Access the dashboard at `http://10.67.0.1` while connected to the Pi's hotspot.

- Default login: `admin` / `secret`
- Use **WiFi Client** → **Scan** to find and connect to a new network

> **Important:** RaspAP installs its own hostapd config on `wlan0` by default, which will overwrite the hotspot config from Part 2. After installing, re-check `/etc/hostapd/hostapd.conf` and restore the `interface=wlan1` config if needed.

---

## Part 7: Safe Shutdown Button & Status LED

Wiring (GPIO, BCM numbering):

| Component | GPIO Pin | Physical Pin |
|---|---|---|
| Push button (leg 1) | GPIO 21 | Pin 40 |
| Push button (leg 2) | GND | Pin 39 |
| LED (+) via 330Ω resistor | GPIO 20 | Pin 38 |
| LED (-) | GND | Pin 34 |

Create the script:

```bash
sudo nano /usr/local/bin/shutdown-button.py
```

```python
import RPi.GPIO as GPIO
import subprocess
import time

BUTTON_PIN = 21
LED_PIN = 20

GPIO.setmode(GPIO.BCM)
GPIO.setup(BUTTON_PIN, GPIO.IN, pull_up_down=GPIO.PUD_UP)
GPIO.setup(LED_PIN, GPIO.OUT)

GPIO.output(LED_PIN, GPIO.HIGH)

def shutdown(channel):
    for i in range(3):
        GPIO.output(LED_PIN, GPIO.LOW)
        time.sleep(0.2)
        GPIO.output(LED_PIN, GPIO.HIGH)
        time.sleep(0.2)
    GPIO.output(LED_PIN, GPIO.LOW)
    subprocess.call(['sudo', 'shutdown', '-h', 'now'])

GPIO.add_event_detect(BUTTON_PIN, GPIO.FALLING, callback=shutdown, bouncetime=2000)

try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    GPIO.cleanup()
```

```bash
sudo chmod +x /usr/local/bin/shutdown-button.py
```

Create the systemd service:

```bash
sudo nano /etc/systemd/system/shutdown-button.service
```

```ini
[Unit]
Description=Shutdown Button
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/shutdown-button.py
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable shutdown-button
sudo systemctl start shutdown-button
```

**Usage:** LED turns on a few seconds after boot (Pi is ready). Press the button to trigger a safe shutdown — LED blinks 3 times, then turns off when it's safe to cut power via the main switch.

---

## Part 8: Power Circuit (LiPo Battery)

```
LiPo battery → TP4056 (B+ / B-)
TP4056 OUT+ → SPDT switch → MT3608 VIN+
TP4056 OUT- → MT3608 VIN- (direct, no switch)
MT3608 OUT+ → Pi GPIO Pin 2 (5V)
MT3608 OUT- → Pi GPIO Pin 6 (GND)
```

**Before connecting MT3608 to the Pi:** use a multimeter (set to DC voltage) to measure MT3608 OUT+ to OUT-, and adjust the onboard potentiometer until it reads exactly **5.0V**. Connecting at a higher voltage can damage the Pi.

The TP4056 has built-in overcharge/overdischarge protection, so it's safe to leave it charging via USB-C unattended.

---

## Daily Usage

1. Flip the main power switch on
2. Wait ~20-30 seconds for the status LED to turn on (Pi is ready)
3. Connect your laptop/phone to the Pi's hotspot (SSID/password set in Part 2)
4. If the public WiFi has a captive portal, open a browser and complete the login
5. The WireGuard tunnel connects automatically — all traffic is now encrypted
6. To power off: press the shutdown button, wait for the LED to blink 3x and turn off, then flip the main power switch

## Troubleshooting

- **No `wlan1` after plugging in adapter:** run `sudo rpi-update`, reboot, then `sudo modprobe rtw88_8821au`
- **Hotspot not visible after reboot:** check `sudo systemctl status hostapd` and `ip addr show wlan1`
- **No internet on hotspot:** check `sudo sysctl net.ipv4.ip_forward` (should be `1`) and `sudo iptables -t nat -L -n`
- **VPN not connecting:** check `sudo wg show` for a recent handshake; verify both public keys match on Pi and VPS configs and that the VPS firewall allows UDP 51820
