# Klipper-Offload
PROOF OF CONCEPT, offloading Klipper to a different host (MCU, THR, MMU, CAMERA).
Comprehensive Decoupled Klipper Network Bridge Guide
This guide outlines the complete setup used to split a QIDI/Makerbase MKS AP Board hardware interface from a main processing host machine (GK41). By leveraging `socat` with `tcp-nodelay`, the processing load is entirely shifted to the new host machine while keeping hardware transmission latencies at near-zero.
---
1. System Topology Overview
Original AP Board (`192.168.2.200`): Acts as the hardware bridge. Holds the physical connections for the Mainboard MCU, MMU Box MCU, THR Toolhead UART port, and USB Camera.
New Host Machine (`192.168.2.240`): Mini PC (GK41) handling all heavy Klipper processing, macro evaluations, and Moonraker/Mainsail web dashboards.
---
2. Part A: The Original AP Board Setup (`192.168.2.200`)
The stock AP board shares raw physical microcontroller serial interfaces directly across independent TCP network ports.
A.1 Package Installation
```bash
sudo apt update && sudo apt install socat -y
```
(Note: System warnings regarding masked background update tools like `packagekit.service` are typical for optimized headless printer distros and can safely be ignored).
A.2 Deploying the Forwarder Service
Create a permanent, root-level background systemd service to map and listen on structural ports during hardware bootups:
```bash
sudo nano /etc/systemd/system/klipper-forwarder.service
```
Paste this configuration block:
```ini
[Unit]
Description=Klipper Serial and Camera Forwarder over Network
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c "\
  /usr/bin/socat TCP-LISTEN:7001,reuseaddr,fork,tcp-nodelay FILE:/dev/serial/by-id/usb-Klipper_QIDI_MAIN_V2_1.0.4_833630210C6439034B363338-if00,b115200,raw,echo=0 & \
  /usr/bin/socat TCP-LISTEN:7002,reuseaddr,fork,tcp-nodelay FILE:/dev/serial/by-id/usb-Klipper_QIDI_BOX_V2_1.1.3_220050001851353135353636-if00,b115200,raw,echo=0 & \
  /usr/bin/socat TCP-LISTEN:7003,reuseaddr,fork,tcp-nodelay FILE:/dev/ttyS4,b500000,raw,echo=0 & \
  /usr/bin/ustreamer --device=/dev/v4l/by-id/usb-UnionImage_Co._Ltd_CCX2F3298_1234567890-video-index0 --host=0.0.0.0 --port=8080; \
  wait"
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
A.3 Activating the Infrastructure
Reload system units, toggle the service to start automatically on every power-up, and boot the sockets:
```bash
sudo systemctl daemon-reload
sudo systemctl enable klipper-forwarder.service
sudo systemctl start klipper-forwarder.service
```
---
3. Part B: The New Host Machine Setup (`192.168.2.240`)
The GK41 client processes the remote TCP streams, mounts them back down locally as pseudo-terminal links (`pty`), and establishes a dependent timeline for Klipper.
B.1 Package Installation
```bash
sudo apt update && sudo apt install socat -y
```
B.2 Deploying the Client Receiver Bridge
Create the corresponding network receptor system service file:
```bash
sudo nano /etc/systemd/system/klipper-client-bridge.service
```
Paste this configuration block, designed with `PrivateTmp=false` to completely bypass systemd path-sandboxing restrictions, and staggered `sleep 1` steps to prevent simultaneous connection collisions on the AP Board's TCP handshake tracker:
```ini
[Unit]
Description=Klipper Network Serial Receiver Bridge
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
PrivateTmp=false
ExecStartPre=/bin/rm -f /tmp/ttyNet_Main /tmp/ttyNet_MMU /tmp/ttyNet_THR
ExecStart=/bin/bash -c "\
  /usr/bin/socat pty,link=/tmp/ttyNet_Main,raw,echo=0,mode=666 TCP:192.168.2.200:7001,tcp-nodelay & \
  sleep 1; \
  /usr/bin/socat pty,link=/tmp/ttyNet_MMU,raw,echo=0,mode=666 TCP:192.168.2.200:7002,tcp-nodelay & \
  sleep 1; \
  /usr/bin/socat pty,link=/tmp/ttyNet_THR,raw,echo=0,mode=666 TCP:192.168.2.200:7003,tcp-nodelay & \
  while true; do sleep 3600; done"
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
B.3 Injecting Klipper Order Dependencies
To prevent Klipper from erroring out with `[Errno 2] No such file or directory` at boot, force Klipper to wait for the bridge links to finalize:
```bash
sudo systemctl edit klipper.service
```
Add the following lines into the blank space between the lines or override file, save, and close the editor:
```ini
[Unit]
Requires=klipper-client-bridge.service
After=klipper-client-bridge.service
```
B.4 Activating the Receiving Services
```bash
sudo systemctl daemon-reload
sudo systemctl enable klipper-client-bridge.service
sudo systemctl start klipper-client-bridge.service
sudo systemctl restart klipper
```
---
4. Part C: Verification and Klipper Binding
C.1 Mapping Klipper Paths (`printer.cfg`)
Now that the virtual ports generate globally inside `/tmp/`, access your web interface (Mainsail/Fluidd) and edit your connection block to target the new virtual addresses:
```ini
[mcu]
serial: /tmp/ttyNet_Main

[mcu mmu]
serial: /tmp/ttyNet_MMU

[mcu THR]
serial: /tmp/ttyNet_THR
restart_method: command
baud: 500000
```
C.2 Verifying Local Serial Mounts
Confirm all connections have established active pseudo-terminals by inspecting your local directory structure:
```bash
ls -la /tmp/ttyNet_*
```
Expected successful terminal return:
```text
lrwxrwxrwx 1 root root 10 Oct  6 02:12 /tmp/ttyNet_Main -> /dev/pts/2
lrwxrwxrwx 1 root root 10 Oct  6 02:12 /tmp/ttyNet_MMU -> /dev/pts/3
lrwxrwxrwx 1 root root 10 Oct  6 02:12 /tmp/ttyNet_THR -> /dev/pts/4
```
C.3 Mapping the Webcam Stream
Open your web control panel (Mainsail/Fluidd).
Go to Settings -> Cameras -> Add Camera.
Set the stream source URL path directly to your AP Board webstream output:
`http://192.168.2.200:8080/?action=stream`
---
5. Architectural Optimization Glossary
`tcp-nodelay`: Disables the TCP Nagle Algorithm on both ends of the sockets. This eliminates automated multi-packet aggregation wait loops, forcing the Linux kernel to transmit small serial gcode messages instantly to eliminate printer stuttering.
Staggered Handshake (`sleep 1`): Paces the execution loop so the AP board parses incoming client socket bindings sequentially without losing track of network threads.
`b500000`: Locks the UART hardware bus on `/dev/ttyS4` directly to the specific high-frequency clock cycles expected by the stm32f103xe THR toolhead firmware.
