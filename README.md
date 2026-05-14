# Garmin OneHelm Reverse Engineering Notes
## Running a Custom HTML App on a Garmin GPSMAP 923

This document describes how a custom HTML application was successfully integrated into a Garmin GPSMAP 923 using the undocumented OneHelm HTML application infrastructure.

The setup was tested using:

- Garmin GPSMAP 923
- Raspberry Pi
- Garmin Marine Network Ethernet connection
- Python HTTP server
- Avahi/mDNS
- Custom SSDP responder

---

# Disclaimer

This is undocumented behavior discovered via network inspection and experimentation.

No official Garmin SDK or API documentation was used.

Use at your own risk.

Marine electronics occasionally behave like emotionally unavailable Scandinavian furniture:
- reliable
- elegant
- difficult to negotiate with

---

# Overview

The Garmin MFD performs the following discovery sequence:

```text
mDNS discovery
    ↓
Fetch JSON configuration
    ↓
Fetch icon
    ↓
Launch embedded web application
```

Additionally:

```text
SSDP / UPnP discovery
    ↓
Fetch UPnP root descriptor
```

appears to assist discovery/bootstrap.

---

# Network Architecture

```text
Garmin GPSMAP 923
        |
   Ethernet / Marine Network
        |
    Raspberry Pi
```

Observed Garmin IP:

```text
172.16.6.0
```

Pi IP used:

```text
172.16.85.190
```

---

# Observed Garmin Discovery Traffic

Garmin continuously emits:

## SSDP Discovery

```text
M-SEARCH * HTTP/1.1
Host: 239.255.255.250:1900
Man: "ssdp:discover"
MX: 3
ST: upnp:rootdevice
```

## mDNS Queries

Garmin also performs mDNS discovery and looks for:

```text
_garmin-mrn-html._tcp
```

---

# Required Components

The following services must exist on the Raspberry Pi:

| Service | Purpose |
|---|---|
| mDNS (Avahi) | Advertise Garmin HTML app |
| HTTP server | Serve config + app |
| SSDP responder | Optional but appears helpful |
| HTML application | Rendered inside Garmin MFD |

---

# 1. Configure Static IP

```bash
sudo ip addr flush dev eth0
sudo ip addr add 172.16.85.190/16 dev eth0
sudo ip link set eth0 up
```

Verify:

```bash
ping 172.16.6.0
```

---

# 2. Install Required Packages

```bash
sudo apt update
sudo apt install -y avahi-daemon avahi-utils python3 python3-pip tcpdump
```

---

# 3. Project Structure

```text
onehelm-test/
├── app/
│   └── index.html
├── onehelm/
│   ├── config.json
│   ├── icon.png
│   └── upnp.xml
├── rootDesc.xml
└── ssdp_responder.py
```

---

# 4. Create UUID

Generate an application UUID:

```bash
uuidgen
```

Example:

```text
99223b99-e4d3-4552-b43b-7b3360c937f7
```

This UUID should remain consistent everywhere.

---

# 5. Configure mDNS (Avahi)

Create:

```bash
sudo nano /etc/avahi/services/onehelm.service
```

Contents:

```xml
<?xml version="1.0" standalone="no"?>
<!DOCTYPE service-group SYSTEM "avahi-service.dtd">

<service-group>
  <name replace-wildcards="yes">Pi OneHelm Test</name>

  <service>
    <type>_garmin-mrn-html._tcp</type>
    <port>8000</port>

    <txt-record>protovers=1</txt-record>
    <txt-record>path=/onehelm/config.json</txt-record>
  </service>
</service-group>
```

Restart Avahi:

```bash
sudo systemctl restart avahi-daemon
```

Verify:

```bash
avahi-browse -r _garmin-mrn-html._tcp
```

---

# 6. Create JSON Config

File:

```bash
~/onehelm-test/onehelm/config.json
```

Contents:

```json
{
  "path": "/app/",
  "title": "Pi OneHelm Test",
  "icon": "/onehelm/icon.png",
  "id": "99223b99-e4d3-4552-b43b-7b3360c937f7"
}
```

## Required Fields

| Field | Description |
|---|---|
| path | URL path to web app |
| title | Display name on Garmin |
| icon | Relative URL to icon |
| id | Unique application GUID |

---

# 7. Create Application HTML

File:

```bash
~/onehelm-test/app/index.html
```

Contents:

```html
<!doctype html>
<html>
<head>
  <title>Pi OneHelm Test</title>

  <meta name="viewport"
        content="width=device-width, initial-scale=1">

  <style>
    body {
      background: #101820;
      color: white;
      font-family: sans-serif;
      padding: 40px;
      font-size: 32px;
    }
  </style>
</head>

<body>
  <h1>Pi OneHelm Test</h1>

  <p>Hello Garmin GPSMAP 923.</p>

  <script>
    console.log("OneHelm app started");
    alert(navigator.userAgent);
  </script>
</body>
</html>
```

---

# 8. Create Icon

Example PNG:

```bash
python3 - <<'PY'
from pathlib import Path
import base64

png = b'iVBORw0KGgoAAAANSUhEUgAAAEAAAABACAQAAAAAYLlVAAAAKElEQVR42u3OIQEAAAgDINc/9K3hHBQgkKmtmQAAAAAAAAAAAAAAwG8CNWAAAU9SqYIAAAAASUVORK5CYII='

Path('/home/morten/onehelm-test/onehelm/icon.png').write_bytes(
    base64.b64decode(png)
)
PY
```

---

# 9. Create UPnP Descriptor

File:

```bash
~/onehelm-test/rootDesc.xml
```

Contents:

```xml
<?xml version="1.0" encoding="utf-8"?>

<root xmlns="urn:schemas-upnp-org:device-1-0">

  <specVersion>
    <major>1</major>
    <minor>0</minor>
  </specVersion>

  <device>
    <deviceType>
      urn:schemas-upnp-org:device:Basic:1
    </deviceType>

    <friendlyName>Pi OneHelm Test</friendlyName>

    <manufacturer>Morten Labs</manufacturer>

    <modelName>Pi OneHelm App</modelName>

    <modelNumber>1</modelNumber>

    <serialNumber>001</serialNumber>

    <presentationURL>/</presentationURL>

    <UDN>
      uuid:99223b99-e4d3-4552-b43b-7b3360c937f7
    </UDN>
  </device>

</root>
```

---

# 10. SSDP Responder

File:

```bash
~/onehelm-test/ssdp_responder.py
```

Contents:

```python
import socket

PI_IP = "172.16.85.190"
PORT = 8000

UUID = "99223b99-e4d3-4552-b43b-7b3360c937f7"

MCAST_GRP = "239.255.255.250"
MCAST_PORT = 1900

response_template = (
    "HTTP/1.1 200 OK\r\n"
    "CACHE-CONTROL: max-age=120\r\n"
    "ST: upnp:rootdevice\r\n"
    "USN: uuid:{uuid}::upnp:rootdevice\r\n"
    "EXT:\r\n"
    "SERVER: Debian/11.2 UPnP/1.1 PiOneHelm/0.1\r\n"
    "LOCATION: http://{ip}:{port}/rootDesc.xml\r\n"
    'OPT: "http://schemas.upnp.org/upnp/1/0/"; ns=01\r\n'
    "01-NLS: 123456789\r\n"
    "BOOTID.UPNP.ORG: 123456789\r\n"
    "CONFIGID.UPNP.ORG: 1337\r\n"
    "\r\n"
)

sock = socket.socket(
    socket.AF_INET,
    socket.SOCK_DGRAM,
    socket.IPPROTO_UDP
)

sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

sock.bind(("", MCAST_PORT))

mreq = (
    socket.inet_aton(MCAST_GRP) +
    socket.inet_aton(PI_IP)
)

sock.setsockopt(
    socket.IPPROTO_IP,
    socket.IP_ADD_MEMBERSHIP,
    mreq
)

print("SSDP responder running")

while True:
    data, addr = sock.recvfrom(2048)

    text = data.decode(errors="ignore")

    if "M-SEARCH" in text:
        response = response_template.format(
            ip=PI_IP,
            port=PORT,
            uuid=UUID
        )

        sock.sendto(response.encode(), addr)

        print("Sent SSDP response")
```

Run:

```bash
sudo python3 ssdp_responder.py
```

---

# 11. Start HTTP Server

```bash
cd ~/onehelm-test
python3 -m http.server 8000
```

---

# 12. Packet Capture Examples

## Watch Garmin Discovery

```bash
sudo tcpdump -ni eth0 -A udp port 1900
```

## Watch Garmin Requests

```bash
sudo tcpdump -ni eth0 -s0 -A \
'host 172.16.6.0 and host 172.16.85.190'
```

---

# Successful Request Sequence

Observed successful onboarding:

```text
GET /rootDesc.xml HTTP/1.1
GET /onehelm/config.json HTTP/1.1
GET /onehelm/icon.png HTTP/1.1
GET /app/ HTTP/1.1
```

---

# Successful Result

The Garmin GPSMAP 923 displayed:

```text
Pi OneHelm Test
```

inside the OneHelm menu and successfully launched the embedded HTML application.

---

# Future Possibilities

Now possible:

- Signal K dashboards
- Engine monitoring
- Vessel automation
- Home Assistant frontend
- Camera control
- Custom NMEA2000 visualizations
- CAN bus diagnostics
- Remote switching/control
- Vessel telemetry
- MQTT integration

---

# Browser Notes

The Garmin MFD appears to contain an embedded browser capable of:

- HTML
- CSS
- JavaScript

Further testing recommended:

```javascript
alert(navigator.userAgent)
```

to identify rendering engine and browser version.

---

# Known Behavior

| Behavior | Status |
|---|---|
| mDNS required | Yes |
| SSDP helpful | Yes |
| HTTPS required | No |
| Relative icon paths required | Yes |
| JSON config required | Yes |
| Embedded browser present | Yes |
| UUID consistency required | Yes |

---

# Security Notes

Current observations:

- HTTP accepted
- No TLS required
- No authentication observed
- Garmin trusts local network advertisement

This may have security implications on shared marine networks.

---

# Credits

This work was derived from:
- packet capture analysis
- SSDP reverse engineering
- mDNS experimentation
- Garmin network observation
- community discoveries

---

# Example Garmin User-Agent

Observed:

```text
User-Agent: Garmin GPSMAP 923/3901
```

---

# Final Notes

This setup effectively enables:

```text
Custom web applications running natively
inside Garmin OneHelm.
```

Which is simultaneously:
- extremely useful
- slightly alarming
- very fun

Like most marine electronics projects.


# More notes
-The Icon must be a PNG 256x256
-Browser capabilities:

Mozilla/5.0 (Linux; X86_64 GNU/Linux)
AppleWebKit/601.1 (KHTML, like Gecko)
Safari/601.1 WPE

Garmin is basically running:

WPE WebKit

And confirmed support for:

Feature	Status:
fetch()	✅
Promise	✅
WebSocket	✅
localStorage	✅
sessionStorage	✅
CSS Grid	✅
Flexbox	✅
Touch events	✅
Canvas	✅
SVG	✅
ES6 let/const	✅
