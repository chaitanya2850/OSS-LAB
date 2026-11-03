# Custom Linux Kernel 5.15.224: Build and Boot Guide

This guide records the steps followed on Ubuntu 24.04.5 LTS on an HP
Gaming Laptop. The custom kernel was **5.15.224**. The final working
boot used `nomodeset`; the AMD graphics freeze was not permanently
fixed.

> Keep the original Ubuntu kernel installed as a fallback. `nomodeset`
> is a workaround that disables normal kernel graphics mode-setting and
> may limit graphics acceleration and display functionality.

## 1. Install build dependencies

```bash
sudo apt update
sudo apt upgrade
sudo apt install build-essential libncurses-dev bison flex \
  libssl-dev libelf-dev bc dwarves fakeroot cpio rsync git wget
```

## 2. Download and extract the kernel source

The source is available from the [official Linux Kernel
Archives](https://www.kernel.org/).

```bash
git clone --depth 1 --branch v5.15.224 https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git linux-5.15.224
cd linux-5.15.224
```

## 3. Configure the kernel

Preferably, use a generic kernel configuration of Linux distributions.

[Collection] (https://github.com/nyrahul/linux-kernel-configs/blob/main/Ubuntu%2022.04.5%20LTS/5.15.0-144-generic/bootconfig.md)

Or 

Start from the current Ubuntu kernel configuration:

```bash
cp /boot/config-$(uname -r) .config
make olddefconfig
```

Optionally open the configuration menu:

```bash
make menuconfig
```

Relevant graphics options include `CONFIG_DRM`, `CONFIG_DRM_AMDGPU`, and
`CONFIG_DRM_RADEON`. During troubleshooting, the configuration included:

```text
CONFIG_DRM=m
CONFIG_DRM_AMDGPU=m
```

`=m` means the driver is built as a module. This is valid, but does not
guarantee successful driver initialization.

## 4. Compile the kernel

```bash
make -j$(nproc)
```

## 5. Install modules

```bash
sudo make modules_install
```

Modules are installed under `/lib/modules/5.15.224/`.

## 6. Install the kernel

```bash
sudo make install
```

### DKMS/NVIDIA hook issue encountered

The install process encountered an issue involving DKMS and the NVIDIA
driver. Since NVIDIA support was not needed for the practical, the DKMS
post-install hook was temporarily disabled:

```bash
sudo chmod -x /etc/kernel/postinst.d/dkms
sudo make install
sudo chmod +x /etc/kernel/postinst.d/dkms
```

This was a workaround for the hook, not a repair of the NVIDIA driver.
The executable permission was restored afterward.

## 7. Update GRUB and check files

```bash
sudo update-grub
ls -lh /boot/vmlinuz-5.15.224
ls -lh /boot/initrd.img-5.15.224
```

Keep the original Ubuntu kernel installed.

## 8. Refresh firmware

During troubleshooting, an AMD firmware warning referenced
`amdgpu/green_sardine_sdma.bin`. The firmware package was reinstalled:

```bash
sudo apt install --reinstall linux-firmware
```

Check for matching files:

```bash
ls /lib/firmware/amdgpu/green_sardine*
```

## 9. First boot attempt

Restart:

```bash
sudo reboot
```

Select the custom `5.15.224` kernel in GRUB. A normal boot froze during
AMD graphics initialization.

The laptop has AMD Radeon Vega integrated graphics and an NVIDIA GA107M
discrete GPU. NVIDIA support was not required for this practical.

## 10. Boot with `nomodeset` (working workaround)

At the GRUB menu:

1.  Highlight the custom `5.15.224` entry.

2.  Press `e`.

3.  Find the line beginning with `linux`. Remove quiet splash and contents after that.

4.  After removing, add `nomodeset` to the end of that line. For example:

    ```text
    ro nomodeset
    ```

5.  Press `Ctrl + X` or `F10` to boot.

This allowed the custom kernel to reach a usable text-mode environment.

### Make `nomodeset` permanent (optional)

Edit GRUB defaults:

```bash
sudo nano /etc/default/grub
```

Change:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

to:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash nomodeset"
```

Then run:

```bash
sudo update-grub
sudo reboot
```

This may apply `nomodeset` to other kernels too. To apply it only to the
custom kernel, add it manually in that kernel's GRUB entry instead.

## 11. Troubleshooting performed

---

Check or test Purpose / result

---

`lspci -k` Identified AMD integrated graphics
and NVIDIA discrete graphics.

`modinfo amdgpu` Confirmed the AMDGPU module file
existed.

`lsmod` Checked loaded graphics modules.

`sudo dmesg \| grep -i amdgpu` Examined AMD graphics
initialization.

Reinstalled `linux-firmware` Refreshed firmware files.

Checked `systemctl status sddm` Investigated the graphical login
manager.

Tried `systemd.unit=multi-user.target` Tested booting without the
graphical login manager; freeze
remained.

Removed Tested without extra boot
`initcall_blacklist=simpledrm_platform_driver_init` parameters; freeze remained.
and `amdgpu.backlight=0` if present

---

The logs showed AMDGPU attempting kernel mode-setting, including:

```text
amdgpu: kernel modesetting enabled
fb0: switching to (null) from EFI VGA
```

The freeze occurred around the AMD display handoff. The framebuffer
message alone does not prove the cause.

The possible diagnostic parameter `amdgpu.dc=0` was discussed but **was
not tested**.

## 12. Verify the working boot

After booting with `nomodeset`:

```bash
uname -r
cat /proc/cmdline
ls /lib/modules/5.15.224/
ls -lh /boot/vmlinuz-5.15.224
```

`uname -r` should report `5.15.224`, and `/proc/cmdline` should include
`nomodeset`.

## 13. Summary

---

Problem Action Result

---

DKMS/NVIDIA hook Temporarily disabled Installation proceeded
interrupted the hook, installed  
 installation kernel, restored its  
 executable permission

AMD firmware warning Reinstalled Firmware package
`linux-firmware` refreshed

Freeze during normal Investigated AMDGPU/DRM Freeze remained
graphics initialization and tried text-only  
 boot

Need to boot custom Added `nomodeset` in Custom kernel booted
kernel GRUB

Recovery option Retained original Fallback preserved
Ubuntu kernel

---

## Final result

Linux kernel **5.15.224** was installed and booted on Ubuntu 24.04.5 LTS
using `nomodeset`. The AMD graphics initialization issue remains
unresolved; `nomodeset` is a boot workaround, not a permanent AMDGPU
fix.
