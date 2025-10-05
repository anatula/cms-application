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
- Skip Ubuntu Pro offer
- SSH configuration: **Enable "Install OpenSSH server"** and later configure ssh keys
- Featured server snaps: snaps are containerized applications with dependencies included Auto-updating** packages from Snap Store Isolated from system packages. Install manually AFTER system is running
- confirm destructive action again

### Step 2: Initial Server Setup (30 mins)
After install, connect from your Mac:

```bash
ssh wpadmin@museum-server.local  # or use IP address
```
Basic server setup:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install htop tree ncdu -y  # monitoring tools
```