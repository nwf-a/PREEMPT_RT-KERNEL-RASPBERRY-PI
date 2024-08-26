# NATIVELY BUILD KERNEL LINUX

Sources:

- Kernel Source: [Linux Kernel Repository](https://github.com/raspberrypi/linux)
- RT Patch Archive: [PREEMPT_RT Patch Archive](https://mirrors.edge.kernel.org/pub/linux/kernel/projects/rt/6.6/older/)

## Update

Update the package index:

```bash
sudo apt update
```

Upgrade all installed packages to their latest versions, if necessary:

```bash
sudo apt full-upgrade
```

## INSTALL DEPENDENCIES AND TOOLS

Install the required dependencies and tools:

```bash
sudo apt install git bc bison flex libssl-dev make libncurses5-dev
```

## DOWNLOAD SOURCE CODE AND PATCH

Create a directory to store the source code and patches:

```bash
mkdir ~/develop
```

Move to the `develop` folder:

```bash
cd ~/develop
```

Clone the latest Raspberry Pi kernel source code:

```bash
git clone --depth=1 https://github.com/raspberrypi/linux
```

or download the stable kernel source as a zip file:

```bash
wget https://github.com/raspberrypi/linux/archive/refs/tags/stable_20240529.zip
```

In this example, I will use the stable kernel.

Unzip the source:

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

## APPLY THE PATCH

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

## BUILD CONFIGURATION

For the Raspberry Pi 3, it is recommended to build the 32-bit version, as the 64-bit kernel (rpi-6.6.y) is not compatible with the Raspberry Pi 3.

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

Copy overlays and README:

```bash
sudo cp arch/arm/boot/dts/overlays/*.dtb* /boot/firmware/overlays/
```

```bash
sudo cp arch/arm/boot/dts/overlays/README /boot/firmware/overlays/
```

Finally, reboot your Raspberry Pi:

```bash
sudo reboot
```

## REFERENCE

- Raspberry Pi Documentation: [Raspberry Pi Linux Kernel Documentation](https://www.raspberrypi.com/documentation/computers/linux_kernel.html)
- System Lockup Patch for RPi 3: [Fix System Lockup for RPi 3](https://github.com/kdoren/linux/wiki/Building-PREEMPT_RT-kernel-for-Raspberry-Pi)
