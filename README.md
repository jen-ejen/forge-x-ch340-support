markdown
# CH340 Support for Forge-X Firmware (Flashforge Adventurer 5M / Pro)

> **Firmware:** Forge-X (custom ff5m-based)  
> **Goal:** Enable USB-to-Serial adapters based on the CH340 chip (e.g., for external MCUs, Klipper, or other serial devices)  
> **Kernel version:** `5.4.61+` (specific to the Adventurer 5M)  
> **Tested on:** Debian 13 (Trixie) with GCC 14.2.0

---

## 📌 Overview

The Forge-X firmware (based on [ff5m by DrA1ex](https://github.com/DrA1ex/ff5m)) does not include native support for the CH340 USB-to-Serial converter. This patch adds the required kernel modules (`usbserial` and `ch341`) and configures them to load automatically at boot.

**After applying this patch, your printer will be able to:**
- Detect CH340-based devices (VID:PID `1a86:7523`)
- Create a virtual serial port (`/dev/ttyUSB0`)
- Connect external MCUs (e.g., for Klipper)

---

## 🚀 Quick Start (using pre-built modules)

> **If you don't want to compile the modules yourself**, you can skip straight to **Step 6** and use the pre-built modules from the [`releases`](../../releases) page or the `modules/` folder in this repository.

**Prerequisites:**
- SSH access to your printer (root)
- The pre-built modules (`usbserial.ko` and `ch341.ko`) — download them from the **Releases** section of this repository.

**Steps:**

1. Copy the modules to the printer:
   ```bash
   scp -O usbserial.ko root@<PRINTER_IP>:/root/
   scp -O ch341.ko root@<PRINTER_IP>:/root/
Continue from Step 7 below (temporary module loading and auto-start configuration).

⚠️ Important: Pre-built modules are compiled for kernel 5.4.61+. Do not use them on other kernel versions — they will fail with invalid module format. Check your kernel version with uname -r before proceeding.

🧩 Requirements (for manual build)
Build host (PC):
Debian 13 (Trixie) or Ubuntu 24.04+ (other distros may work but are untested)

Internet access

Installed GCC 14.2.0 ARM cross-compiler

Build tools (see installation commands below)

Printer:
SSH access (root privileges required)

Kernel config file (available via /proc/config.gz)

Enough free space in /lib/modules/ (or use /root/ as fallback)

🛠️ Build & Installation Guide (Full)
1. Prepare your build host (Debian 13)
Install the required packages:

bash
sudo apt update
sudo apt install -y build-essential bc bison flex libssl-dev rsync file python3 pkg-config \
    gcc-arm-linux-gnueabi binutils-arm-linux-gnueabi \
    git wget xz-utils
Verify the cross‑compiler version:

bash
arm-linux-gnueabi-gcc --version
Expected output:

text
arm-linux-gnueabi-gcc (Debian 14.2.0-19) 14.2.0
2. Clone the kernel source
bash
cd ~
git clone https://github.com/cfelicio/tina-linux-5.4.git linux-5.4.61-tina
cd linux-5.4.61-tina
Note: This repository is a fork compatible with Allwinner Tina Linux. If you are using a different source, adjust the path accordingly.

3. Extract the kernel config from the printer
On the printer (via SSH):

bash
zcat /proc/config.gz > /root/printer.config
Copy it to your build host:

bash
scp -O root@<PRINTER_IP>:/root/printer.config ~/printer.config
4. Configure the kernel
bash
cp ~/printer.config .config
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- olddefconfig
Enable the USB serial and CH340 modules:

bash
./scripts/config --module CONFIG_USB_SERIAL
./scripts/config --module CONFIG_USB_SERIAL_CH341
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- olddefconfig
5. Build the modules
bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- modules_prepare
make -j$(nproc) ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- M=drivers/usb/serial modules
If successful, the files usbserial.ko and ch341.ko will appear in drivers/usb/serial/.

Verify the module version:

bash
modinfo -F vermagic drivers/usb/serial/ch341.ko
Expected output: 5.4.61+ SMP mod_unload ARMv7 p2v8

6. Copy modules to the printer
bash
scp -O drivers/usb/serial/usbserial.ko root@<PRINTER_IP>:/root/
scp -O drivers/usb/serial/ch341.ko root@<PRINTER_IP>:/root/
⚙️ Installation on the Printer
7. Test the modules (temporary load)
bash
insmod /root/usbserial.ko
insmod /root/ch341.ko
Plug in your CH340 device and verify:

bash
dmesg | tail
ls -l /dev/ttyUSB*
If /dev/ttyUSB0 appears, the driver works.

8. Move modules to the system directory
The printer has a kernel module directory /lib/modules/5.4.61+/. Copy the modules there:

bash
cp /root/usbserial.ko /lib/modules/5.4.61+/usbserial.ko
cp /root/ch341.ko /lib/modules/5.4.61+/ch341.ko
If the directory doesn't exist: mkdir -p /lib/modules/5.4.61+/

Important: On BusyBox-based systems, cp requires the full destination path (not just the directory), as shown above.

9. Set up auto‑loading
Create the startup script /etc/init.d/S50usbserial:

bash
cat > /etc/init.d/S50usbserial << 'EOF'
#!/bin/sh
case "$1" in
  start)
    insmod /lib/modules/5.4.61+/usbserial.ko
    insmod /lib/modules/5.4.61+/ch341.ko
    ;;
  stop)
    rmmod ch341 2>/dev/null
    rmmod usbserial 2>/dev/null
    ;;
  *)
    echo "Usage: $0 {start|stop}"
    exit 1
esac
exit 0
EOF
Make it executable:

bash
chmod +x /etc/init.d/S50usbserial
Test it:

bash
/etc/init.d/S50usbserial start
lsmod | grep ch341
10. Reboot and verify
bash
reboot
After reboot, check:

bash
lsmod | grep ch341
dmesg | tail -5
ls -l /dev/ttyUSB*
🔧 Klipper Integration
If you use Klipper, add this to your printer.cfg:

ini
[mcu]
serial: /dev/ttyUSB0

📁 Files Added to the Printer
text
/lib/modules/5.4.61+/
    ├── usbserial.ko      # Base USB serial driver
    └── ch341.ko          # CH340-specific driver
/etc/init.d/
    └── S50usbserial      # Auto‑load script
📝 Important Notes
These modules are built specifically for kernel 5.4.61+. Do not attempt to use them on other kernel versions — they will fail with invalid module format.

If you upgrade the firmware, these modules may be overwritten. Keep a backup of the script and modules for quick recovery.

For other USB-to-Serial chips (e.g., CP2102), the process is similar — just enable the corresponding module (CONFIG_USB_SERIAL_CP210X) during kernel config.

🙏 Credits
DrA1ex for the ff5m firmware

cfelicio for the Tina Linux kernel fork

Version: 1.0
Date: 2026-08-24
Tested on: Debian 13 (Trixie) with GCC 14.2.0
Maintainer: jen-ejen
