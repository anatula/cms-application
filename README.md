# CMS Application

Manual deployment optimized for dual-core, low-power hardware (Beelink T4 Pro - Intel N3350 2-core, 4GB DDR3, 64GB eMMC). Learning resource-constrained deployment on limited hardware.

## Hardware Specifications
- **Device**: Beelink T4 Pro Mini PC
- **Processor**: Intel Celeron N3350 (2-core, 1.1-2.4GHz)
- **Memory**: 4GB DDR3
- **Storage**: 64GB eMMC

## Step 0 Pre-Deployment
- Download: https://www.balena.io/etcher/
- Downlaod: https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso
- Click "Flash from file" → select ISO → Select USB drive
- Another option could be:
```bash
# Identify USB drive (BE CAREFUL - double-check disk number!)
diskutil list

# Unmount USB drive (disk2 is my drive)
sudo diskutil unmountDisk /dev/disk2

# Write ISO to USB (takes 5-10 minutes)
sudo dd if=~/Downloads/ubuntu-22.04.5-live-server-amd64.iso of=/dev/disk2 bs=1m

# Eject when complete
sudo diskutil eject /dev/disk2
```

## Bmax black screen problem

After changing the boot order to the bootbale usb, and selecting "Try or Install Ubuntu Server", the screen is blank, doesn't do anything, so I can't continue. Need to add `nomodeset i915.modeset=0`


## Step 1 Ubuntu Server Installation
- Boot from USB (may need to enter boot menu and select the USB drive)
- Press ENTER on "Try or Install Ubuntu Server"
- Language Selection: Choose English and continue
- Keyboard Configuration: Select English (US) layout
- Select "Ubuntu Server"
- Network Configuration: there are 2 network interfaces showing:
    - `enp1s0`: Ethernet (wired) - currently disabled due to autoconfig failure
    - `wlp2s0`: WiFi (wireless) - not connected
- Step WiFi Setup:
    - Select "wlp2s0" (or your WiFi interface)
    - Choose "Edit WiFi"
    - It will scan for networks - wait a moment
    - Select your WiFi network from the list
    - Enter your WiFi password
    - The network IS working if it has an IP address assigned, will show "IP DHCPv4" (means it successfully got an IP address from your router)
- Proxy configuration: skip since server will use direct WiFi connection
- Ubuntu archive mirror configuration: The system has confirmed network connectivity and that it can reach Ubuntu package repositories and is ready to download system packages
- Installer update available: skip the update (continue without installing), can be done later with `sudo apt update && sudo apt upgrade -y`
- Guided storage configuration: skip encryption for now and choose Custom Storage Layout only choose if you have specific partitioning needs. Otherwise:
    - Select: "Use entire disk"
    - Enable: "Set this disk as an LVM group"
- Storage Configuration: LVM (Logical Volume Manager) is a storage management system that creates an abstraction layer between physical disks and filesystems.
```
ubuntu-vg (Volume Group - pools all disk space)
├── ubuntu-lv (27.3GB Logical Volume) → mounted as / (ext4)
└── 54.6GB free space (for future Logical Volumes)
ubuntu-vg = Volume Group (storage pool)
ubuntu-lv = Logical Volume (like a partition)
Free space = Available to create new logical volumes

| Mount Point | Size     | Filesystem | Type           |
|-------------|----------|------------|----------------|
| /           | 27.281G  | ext4       | LVM Volume     |
| /boot       |  2.000G  | ext4       | Partition      |
| /boot/efi   |  1.049G  | FAT32      | Partition      |
```


- Confirm destructive action
- Profile configuration: set user, server and password
- Skip Ubuntu Pro offer
- SSH configuration: **Enable "Install OpenSSH server"** and later configure ssh keys
- Featured server snaps: snaps are containerized applications with dependencies included Auto-updating** packages from Snap Store Isolated from system packages. Install manually AFTER system is running
- confirm destructive action again
- Wait to finish installation
- Reboot now
- Shows Welcome to Ubuntu 22.05.5 LTS

# Server Naming Clarification

## Server Name Purpose (NOT an IP address nor localhost -reserved for the machine itself-) - this is a hostname for network identification

For example, pick lab-admin for user and lab-server for server

## What "lab-server" Means:
- On your local network: `lab-server.local`
- SSH access: `ssh lab-admin@lab-server.local`
- Other devices can find it via: `ping lab-server.local`

## IP Address Handling:
- The server gets IP automatically via DHCP
- You can find it later with: `ip addr show`
- Or from Mac: `ping lab-server.local`

### Step 2: Initial Server Setup (30 mins)
After install, connect from to connect from other machine:

```bash
ssh wpadmin@museum-server.local  # or use IP address
```
Basic server setup:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install htop tree ncdu -y  # monitoring tools
```

```bash
ssh lab-admin@lab-server.local
```

ssh lab-admin@192.168.1.139

## Quick Fix for .local Resolution
**After SSH login, install:**
```bash
sudo apt install avahi-daemon
sudo systemctl enable avahi-daemon
sudo systemctl start avahi-daemon
```

Graceful Shutdown (Recommended)
`sudo shutdown -h now`


### Fix wifi network configuration (change wifi once installed)

```bash
# Remove ALL netplan files
`sudo rm /etc/netplan/*.yaml`

# Remove cloud-init network interference
`echo "network: {config: disabled}" | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`

3. Create ONE simple netplan file:

```bash
# Find your Wi-Fi interface name
WIFI_INTERFACE=$(ip link show | grep wl | awk -F: '{print $2}' | xargs)
echo "Your Wi-Fi interface is: $WIFI_INTERFACE"

# Create the config file
sudo tee /etc/netplan/01-wifi.yaml << EOF
network:
  version: 2
  wifis:
    $WIFI_INTERFACE:
      dhcp4: yes
      access-points:
        "YOUR_SSID":
          password: "YOUR_PASSWORD"
EOF
```
4. Apply it:
```bash
sudo netplan apply
```
5. Test it works:
```bash
ping -c 4 google.com
```
6. Make it survive reboot:
```bash
sudo reboot
```

### Problem: Beelink T4 Pro wouldn't fully power off via SSH - stayed in "zombie" state (LED on, needed unplug).

Cause: "Buggy" BIOS ACPI power management on mini PC.
Fix: Added to /etc/default/grub:
```text
GRUB_CMDLINE_LINUX_DEFAULT="acpi=force apm=power_off"
```
Then: `sudo update-grub` && `sudo reboot`

Why: apm=power_off uses older, more reliable power control method that bypasses broken BIOS ACPI.

### Problem: Machine auto-powered-on after power outages

Fix: BIOS (Critical)
**Setting:** `Chipset → South Cluster → RESTORE AC POWER LOSS`
**Change:** `Power On` → `Power Off`
**Result:** No auto-power-on after outages
