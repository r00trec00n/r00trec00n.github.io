---
title: "Building a Dedicated Parrot OS Attack Node"
date: 2026-08-11 12:00:00 +0500
categories: 
    - Lab Setup
    - Parrot OS
tags:
    - parrot-os
    - xrdp
    - x11vnc
    - networking
    - linux
    - htb-lab
    - sysadmin
    - cybersecurity
    - penetration-testing
    - lab-setup
    - grub
    - bare-metal
pin: false
math: false
mermaid: false
image:
  path: /assets/img/social-pics/Banner.jpeg
  alt: "Parrot OS attack node lab setup with dual monitors"

---

Transitioning from resource-heavy virtual machines to bare-metal Linux on a secondary laptop provides dedicated hardware performance for penetration testing, CTF challenges, and local lab environments.

However, bare-metal deployments on consumer hardware can introduce several operational challenges, including persistent UEFI boot issues, changing IP addresses, remote desktop session conflicts, and the need for isolated networks during security testing.

This guide documents the process of configuring a secondary **Parrot OS** laptop as a dedicated and remotely accessible attack node.

The setup covers:

1. **Fixing HP UEFI boot overrides** and enforcing persistent GRUB boot.
2. **Configuring static network addressing** with `nmcli`.
3. **Mirroring the physical desktop remotely** using `x11vnc` and `xrdp`.
4. **Creating isolated private networks** using Ethernet connections.

---

## **Lab Environment**

This setup is part of my personal cybersecurity lab. I use the Parrot OS laptop as a dedicated bare-metal attack node for:

- Hack The Box and CTF environments
- Penetration-testing practice
- Network enumeration
- Web application testing
- Vulnerable machine labs
- Isolated IoT and hardware testing

Instead of running Parrot OS inside a VM on my main machine, I moved it to a dedicated laptop. This gives me direct access to the laptop's CPU, RAM, and network interfaces while keeping my main workstation available for other tasks.

---

## Fixing HP UEFI Boot Overrides for Parrot OS

Some HP laptops and desktops may ignore or reset Linux NVRAM boot entries created during installation. This can cause the system to bypass GRUB and boot directly into Windows Boot Manager.

There are multiple ways to work around this behavior.

In this guide, I cover **two methods**:

1. **HP Custom Boot Path** — the preferred and non-destructive method.
2. **UEFI Fallback Path** — an alternative workaround that places GRUB in standard UEFI fallback locations.

> **Note:** I personally used and tested the **Custom Boot Path method** on my HP system. The second method is provided as an alternative for systems where the first method is unavailable or does not work.
{: .prompt-warning }

### 1.1 Method 1 — HP Custom Boot Path

HP firmware includes a built-in feature called **Custom Boot Path** on some models. This allows you to specify an EFI executable directly from the BIOS/UEFI setup without modifying the Windows EFI bootloader.

#### 1.1.1 Access the HP Firmware Setup

Turn off your system completely.

Power on the machine and repeatedly press **F10** to enter the BIOS/UEFI Setup Utility.

Use the arrow keys to navigate to the **Advanced** or **System Configuration** tab. The exact location may vary depending on the HP model.

#### 1.1.2 Set the Custom Boot Path

Select **Boot Options** and press **Enter**.

Scroll down to **Custom Boot Path** or **Define Custom Boot Path**.

Select **Add** or **Set Custom Boot Path**, then enter the path to the Parrot GRUB binary:

```text
\EFI\Parrot\grubx64.efi
```

> **Note:** Use backslashes \ when entering UEFI paths within the HP firmware interface.
{: .prompt-info }

#### 1.1.3 Adjust the UEFI Boot Order

While still under **Boot Options**, locate the **UEFI Boot Order** section.

Find **Custom Boot** or **UEFI - Custom** in the list.

Move **Custom Boot** to the top of the boot order, usually using **F5/F6 or +/- keys**, depending on the HP firmware version.

This ensures that the custom GRUB entry takes precedence over **Windows Boot Manager**.

Navigate to the Exit tab, select Save Changes and Exit, and press Enter.

#### 1.1.4 How It Works

The HP UEFI firmware maintains an internal setting that points directly to a specified `.efi` executable on the EFI System Partition (ESP).

By pointing the Custom Boot Path to: `\EFI\Parrot\grubx64.efi`

and moving Custom Boot to the top of the UEFI boot order, the firmware can execute GRUB directly during startup.

This approach does not require replacing the standard Windows Boot Manager EFI file.

### 1.2 Method 2 — UEFI Fallback Path

If the Custom Boot Path option is not available on your HP system, another workaround is to place the GRUB EFI binary in the standard UEFI fallback locations.

> **Warning:** The commands below overwrite the Windows Boot Manager EFI path. Only use this approach on a system where you understand the consequences and have appropriate backups.
{: .prompt-danger }

#### 1.2.1 Create the EFI Directories

```bash
sudo mkdir -p /boot/efi/EFI/BOOT
sudo mkdir -p /boot/efi/EFI/Microsoft/Boot

```

#### 1.2.2 Copy the Parrot GRUB Binary

First, verify that the Parrot EFI files exist:

```bash
ls -lah /boot/efi/EFI/Parrot/
```

Then copy the GRUB executable:

```bash
sudo cp /boot/efi/EFI/Parrot/grubx64.efi \
    /boot/efi/EFI/BOOT/BOOTX64.EFI

sudo cp /boot/efi/EFI/Parrot/grubx64.efi \
    /boot/efi/EFI/Microsoft/Boot/bootmgfw.efi
```

#### 1.2.3 Regenerate GRUB

```bash
sudo update-grub
```

#### 1.2.4 How It Works

UEFI firmware can use the fallback path:

```text
/EFI/BOOT/BOOTX64.EFI
```

Some systems also look for the Windows Boot Manager at:

```text
/EFI/Microsoft/Boot/bootmgfw.efi
```

Placing GRUB at these locations can allow the firmware to launch the Parrot bootloader even when the normal UEFI boot entry is not being respected.

---

## 2. Configure a Static Network Address

A stable IP address is useful when remotely managing the attack node from another workstation.

Instead of relying on DHCP to assign a potentially changing address, NetworkManager can be configured with a static IPv4 address.

### 2.1 Identify the Network Interface

```bash
ip link
```

List active NetworkManager profiles:

```bash
nmcli connection show --active
```

Example:

```text
NAME        DEVICE
PTCL-BB     eth0
```

### 2.2 Configure the Static Address

Replace `PTCL-BB` with your actual NetworkManager connection name.

```bash
sudo nmcli con mod "PTCL-BB" \
    ipv4.addresses 192.168.0.121/24

sudo nmcli con mod "PTCL-BB" \
    ipv4.gateway 192.168.0.1

sudo nmcli con mod "PTCL-BB" \
    ipv4.dns "1.1.1.1,8.8.8.8"

sudo nmcli con mod "PTCL-BB" \
    ipv4.method manual
```

### 2.3 Reactivate the Connection

```bash
sudo nmcli con up "PTCL-BB"
```

### 2.4 Verify the Configuration

```bash
ip addr show eth0
```

You should see:

```text
inet 192.168.0.121/24
```

> **Note:** Make sure the selected static address is outside the DHCP pool configured on your router, or reserve that address for the device on the router to avoid IP conflicts.
{: .prompt-info }

---

## 3. Remote Desktop Mirroring with x11vnc and xrdp

A standard `xrdp` installation normally creates a separate graphical session using `xorgxrdp`.

This is useful when an independent remote desktop session is desired, but it does not provide a mirror of the physical desktop already running on display `:0`.

In this setup, **`x11vnc` shares the existing physical X11 display**, while `xrdp` acts as the RDP frontend.

The resulting architecture is:

```text
Windows Workstation
        │
        │ RDP
        ▼
      xrdp
        │
        │ VNC
        ▼
    x11vnc :5900
        │
        ▼
Physical X11 Desktop :0
```
{: .nolineno }

This allows the remote RDP client to interact with the same desktop session visible on the physical laptop.

### 3.1 Install the Required Packages

```bash
sudo apt update
sudo apt install xrdp x11vnc -y
```

### 3.2 Create the VNC Password

Generate a password file for `x11vnc`:

```bash
sudo x11vnc -storepasswd /etc/x11vnc.pass
```

Set appropriate permissions:

```bash
sudo chmod 644 /etc/x11vnc.pass
```

### 3.3 Create the `x11vnc` Systemd Service

Create the service file:

```bash
sudo nano /etc/systemd/system/x11vnc.service
```

Add:

```ini
[Unit]
Description=x11vnc service for physical display session
After=display-manager.service network.target multi-user.target

[Service]
Type=simple
ExecStart=/usr/bin/x11vnc -display :0 -auth guess -forever -loop -noxdamage -repeat -rfbauth /etc/x11vnc.pass -rfbport 5900 -shared
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

The service starts `x11vnc` against the physical X11 display:

```text
:0
```

and exposes the VNC server on:

```text
127.0.0.1:5900
```

### 3.4 Configure the XRDP VNC Backend

Edit:

```bash
sudo nano /etc/xrdp/xrdp.ini
```

Add the following session profile:

```ini
[Physical-Desktop]
name=Physical-Desktop
lib=libvnc.so
ip=127.0.0.1
port=5900
username=ask
password=ask
```

This instructs XRDP to connect to the locally running VNC server rather than spawning another desktop session.

### 3.5 Enable the Services

Reload systemd:

```bash
sudo systemctl daemon-reload
```

Enable and start `x11vnc`:

```bash
sudo systemctl enable --now x11vnc
```

Enable and start `xrdp`:

```bash
sudo systemctl enable --now xrdp
```

Restart both services:

```bash
sudo systemctl restart x11vnc
sudo systemctl restart xrdp
```

### 3.6 Verify the VNC Listener

Check port `5900`:

```bash
ss -tulpn | grep 5900
```

Expected output should indicate that `x11vnc` is listening on TCP port `5900`.

Check the service status if necessary:

```bash
sudo systemctl status x11vnc --no-pager
```

### 3.7 Connect from Windows

From the Windows workstation:

1. Launch **Remote Desktop Connection** (`mstsc.exe`).
2. Enter the Parrot OS IP address:
```text
192.168.0.121
```


3. Accept the RDP certificate warning if presented.
4. Select the **`Physical-Desktop`** session.
5. Enter the VNC password created earlier.

The remote connection should now display the same physical desktop session running on the Parrot laptop.

> **Troubleshooting:** If you encounter `Could not acquire name on session bus`, verify that XRDP is connecting to the VNC-backed `Physical-Desktop` profile rather than creating a separate `xorgxrdp` session.
{: .prompt-warning }

---

## 4. Deploying Isolated Private Networks

A dedicated attack node is particularly useful when testing IoT devices, embedded systems, vulnerable VMs, or CTF targets.

Instead of placing these devices directly on the home network, isolated subnets can be created using direct Ethernet connections and NetworkManager profiles.

This configuration allows test traffic to remain inside a dedicated laboratory network.

### Direct Ethernet Peering

A direct Ethernet connection can create a simple point-to-point network without requiring a router.

Example topology:

```text
┌──────────────────────────┐
│        Parrot OS         │
│                          │
│   Ethernet: 192.168.1.2  │
└────────────┬─────────────┘
             │
             │ Ethernet
             │
┌────────────▼─────────────┐
│        Peer Host         │
│                          │
│   Ethernet: 192.168.1.1  │
└──────────────────────────┘
```
{: .nolineno }

The network is:

```text
192.168.1.0/24
```

No DHCP server or router is required.

#### 4.1 Configure Parrot OS

Identify the Ethernet interface:

```bash
ip link show
```

Create a persistent NetworkManager profile:

```bash
sudo nmcli connection add \
    type ethernet \
    con-name "Private-Wired" \
    ifname enp0s25 \
    ip4 192.168.1.2/24
```

Enable automatic connection:

```bash
sudo nmcli connection modify \
    "Private-Wired" connection.autoconnect yes
```

Activate the profile:

```bash
sudo nmcli connection up "Private-Wired"
```

Verify:

```bash
ip addr show enp0s25
```

#### 4.2 Configure a Windows Peer

Open **PowerShell as Administrator**:

```powershell
New-NetIPAddress `
    -InterfaceAlias "Ethernet" `
    -IPAddress 192.168.1.1 `
    -PrefixLength 24 `
    -PolicyStore PersistentStore
```

Set the network profile to Private:

```powershell
Set-NetConnectionProfile `
    -InterfaceAlias "Ethernet" `
    -NetworkCategory Private
```

Test connectivity from Parrot:

```bash
ping -c 4 192.168.1.1
```

#### 4.3 Configure a Linux Peer

On the Linux peer:

```bash
sudo nmcli connection add \
    type ethernet \
    con-name "Private-Wired" \
    ifname eth0 \
    ip4 192.168.1.1/24
```

Bring the connection up:

```bash
sudo nmcli connection up "Private-Wired"
```

Verify connectivity:

```bash
ping -c 4 192.168.1.2
```

---

## 5. Network Architecture

The resulting laboratory configuration can be summarized as follows:

| Mode | Local Interface IP | Peer / Gateway IP | Subnet Mask | DHCP Manager |
| --- | --- | --- | --- | --- |
| Direct Ethernet | `192.168.1.2/24` | `192.168.1.1` | `255.255.255.0` | None |

---

# 6. Final Attack-Node Architecture

The complete setup provides several independent components:

```text
                         ┌──────────────────────────┐
                         │    Windows Workstation   │
                         │                          │
                         │        RDP Client        │
                         └────────────┬─────────────┘
                                      │
                                      │ RDP
                                      ▼
                         ┌──────────────────────────┐
                         │         Parrot OS        │
                         │                          │
                         │          xrdp            │
                         │            │             │
                         │            ▼             │
                         │         x11vnc           │
                         │            │             │
                         │            ▼             │
                         │        Physical :0       │
                         └────────────┬─────────────┘
                                      │
                              Ethernet│
                                      │
                                      ▼
                                 Private LAN
                                192.168.1.0/24
```
{: .nolineno }

This provides:

* Bare-metal performance for security tooling.
* Persistent boot configuration.
* Stable remote-management addressing.
* Remote access to the physical desktop.
* Dedicated wired testing networks.

---

## Conclusion

A dedicated Parrot OS laptop can serve as a flexible attack node for penetration testing, CTFs, security research, and isolated laboratory environments.

By addressing firmware boot behavior, configuring predictable network addressing, sharing the physical desktop through `x11vnc`, and deploying isolated wired networks, the system becomes significantly more practical than repeatedly launching a resource-heavy virtual machine.

The final result is a lightweight security workstation that can remain powered on remotely while providing dedicated hardware resources for enumeration, exploitation labs, CTF workloads, and controlled network testing.

> **Lab Safety:** Keep isolated testing networks separate from production or home networks unless routing is explicitly required. When experimenting with local subnets, use dedicated test devices and networks to avoid disrupting unrelated systems.
{: .prompt-warning }