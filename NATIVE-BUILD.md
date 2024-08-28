# Building a Real-Time Kernel for Raspberry Pi

## NATIVE BUILD

Sources:

- Kernel Source: [Linux Kernel Repository](https://github.com/raspberrypi/linux)
- RT Patch Archive: [PREEMPT_RT Patch Archive](https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/6.6/older/)

## PREREQUISITES

### UPDATE AND UPGRADE

First, update your package index:

```bash
sudo apt update
```

and upgrade installed packages, if necessary:

```bash
sudo apt full-upgrade
```

### INSTALL DEPENDENCIES AND TOOLS

Install the required dependencies and tools:

```bash
sudo apt install git bc bison flex libssl-dev make libncurses5-dev
```

## SOURCE CODE AND PATCH

Create a directory to store the source code and patches:

```bash
mkdir ~/develop
```

Move to the `develop` folder:

```bash
cd ~/develop
```

### DOWNLOAD SOURCE CODE

Clone the latest Raspberry Pi kernel source code:

```bash
git clone --depth=1 https://github.com/raspberrypi/linux
```

or alternatively, download the stable kernel source as a zip file:

```bash
wget https://github.com/raspberrypi/linux/archive/refs/tags/stable_20240529.zip
```

In this example, I will use the stable kernel.

Unzip the source code:

```bash
unzip stable_20240529.zip
```

Navigate to the kernel source directory and check the kernel version:

```bash
cd linux-stable_20240529
head Makefile -n 4
```

You should see output similar to:

```text
# SPDX-License-Identifier: GPL-2.0
VERSION = 6
PATCHLEVEL = 6
SUBLEVEL = 31
```

### DOWNLOAD PATCH

The kernel version is 6.6.31. Download the RT patch for this version.

Return to the previous directory:

```bash
cd ..
```

Download patch:

```bash
wget https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/6.6/older/patch-6.6.31-rt31.patch.xz
```

Additionally, download the patch for system lockup (for Raspberry Pi 3 only):

```bash
wget https://raw.githubusercontent.com/kdoren/linux/rpi-5.10.35-rt/0001-usb-dwc_otg-fix-system-lockup.patch
```

## PATCHING THE KERNEL

Navigate to the kernel source directory:

```bash
cd linux-stable_20240529
```

Apply the RT patch:

```bash
xzcat ../patch-6.6.31-rt31.patch.xz | patch -p1
```

Apply the system lockup patch (RPi 3 Only):

```bash
cat ../0001-usb-dwc_otg-fix-system-lockup.patch | patch -p1
```

## KERNEL CONFIGURATION

For the Raspberry Pi 3, it is recommended to build the 32-bit version, as the 64-bit rt-kernel (rpi-6.6.y-rt) is not compatible with the Raspberry Pi 3.

Prepare the default configuration:

```bash
KERNEL=kernel7
make bcm2709_defconfig
```

Customize the kernel configuration:

```bash
make menuconfig
```

In the configuration menu:

- set `Kernel Features` -> `Timer frequency` to `1000 HZ`
- set `CPU Power Management` -> `CPU Frequency Scaling` -> `Default CPUFreq governor` to `performance`
- set `General` -> `Local version`: change to "-v7-CUSTOM" or something else
- set `General` -> `Preemption Model` to `Fully Preeemptible Kernel (Real-Time)`
- set `General` -> `Timers subsystem` -> `Timer tick handling` -> `Full dynticks system (tikcless)`
- Save and exit

## BUILD THE KERNEL

Next, build the kernel. This step can take a long time, depending on your Raspberry Pi model.

Build the 32-bit kernel:

```bash
make -j4 zImage modules dtbs
```

## INSTALL THE KERNEL

Next, install the kernel modules:

```bash
sudo make -j4 modules_install
```

Then, install the kernel and Device Tree blobs into the boot partition, backing up your original kernel.

Backup the current kernel:

```bash
sudo cp /boot/firmware/$KERNEL.img /boot/firmware/$KERNEL-backup.img
```

Install the new kernel image:

```bash
sudo cp arch/arm/boot/zImage /boot/firmware/$KERNEL.img
```

Install Device Tree blobs:

```bash
sudo cp arch/arm/boot/dts/broadcom/*.dtb /boot/firmware/
```

```bash
sudo cp arch/arm/boot/dts/overlays/*.dtb* /boot/firmware/overlays/
```

```bash
sudo cp arch/arm/boot/dts/overlays/README /boot/firmware/overlays/
```

## INSTALL KERNEL HEADERS

Install the kernel headers:

```bash
make -j4 headers_install
```

then, reboot your raspberry:

```bash
sudo reboot
```

Check the kernel version:

```bash
uname -a
```

Output:

```text
Linux raspberrypi 6.6.31-rt31-v7-CUSTOM #1 PREEMPT_RT Mon Aug 26 11:50:20 WIB 2024 armv7l GNU/Linux
```

## OPTIMIZING REAL-TIME PERFORMANCE

### ADD isolcpus KERNEL PARAMETER

To `optimize` the `real-time performance` of your Raspberry Pi, you can isolate specific CPUs using the `isolcpus` kernel parameter. This will prevent the specified CPUs from being used by the default scheduler, allowing them to be dedicated to real-time tasks.

1. Open the boot configuration file for editing:

    ```bash
    sudo nano /boot/firmware/cmdline.txt
    ```

2. Add `isolcpus=1,2,3` at the end of the line.

    ```text
    isolcpus=1,2,3
    ```

    The file should look something like this:

    ```text
    console=serial0,115200 console=tty1 root=PARTUUID=xxxxxxxx-02 rootfstype=ext4 fsck.repair=yes rootwait quiet splash plymouth.ignore-consoles cfg80211.ieee80211_regdom=ID isolcpus=1,2,3
    ```

3. Save and exit the editor: Press `Ctrl + X`, then `Y`, and `Enter`.
4. Reboot to apply the changes:

    ```bash
    sudo reboot
    ```

## Performance Governor Settings

The `PREEMPT_RT` kernel is built with the CPU governor configured to `performance`. However, Raspberry Pi OS might include a startup script that changes this setting to `ondemand`.

For `real-time` applications that require the `performance` governor, you will need to `disable` this script. The service responsible for this is named `raspi-config`, which is different from the `raspi-config` configuration tool.

Check the current CPU governor setting:

```bash
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

Output:

```text
ondemand
ondemand
ondemand
ondemand
```

Check if the `raspi-config` service is running:

```bash
service --status-all | grep raspi-config
```

Output:

```text
[ + ]  raspi-config
```

Install the `rcconf` package and use it to `disable` the `raspi-config` service:

```bash
sudo apt -y install rcconf
sudo rcconf --off raspi-config
reboot
```

Check the New Setting:

```bash
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

Output:

```output
performance
performance
performance
performance
```

## REFERENCE

- Raspberry Pi Documentation: [Raspberry Pi Linux Kernel Documentation](https://www.raspberrypi.com/documentation/computers/linux_kernel.html)
- System Lockup Patch for RPi 3: [Fix System Lockup for RPi 3](https://github.com/kdoren/linux/wiki/Building-PREEMPT_RT-kernel-for-Raspberry-Pi)
