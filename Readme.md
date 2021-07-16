### Activating KVM on Jetson Nano

This is based on https://developer.ridgerun.com/wiki/index.php?title=Jetson_Nano/Development/Building_the_Kernel_from_Source

Jetson Nano's original image does not come with KVM enabled, thus we have to recompile the kernel and activate it. In this article, we're gonna do everything inside the Nano so we don't have to take out the SD card or flash a new image. You might be surprised that recompiling the kernel in the Nano itself takes less than 30 minutes.

So, put your Nano to work with the latest Ubuntu image provided by Jetson Nano, and then boot it and install the dependencies needed to build the kernel:

```bash
sudo apt update && sudo apt-get install -y build-essential bc git curl wget xxd kmod libssl-dev
```

Now, we should get the kernel source at [https://developer.nvidia.com/embedded/downloads](https://developer.nvidia.com/embedded/downloads). As af today, (july 15) the latest is [https://developer.nvidia.com/embedded/l4t/r32_release_v5.1/r32_release_v5.1/sources/t210/public_sources.tbz2](https://developer.nvidia.com/embedded/l4t/r32_release_v5.1/r32_release_v5.1/sources/t210/public_sources.tbz2). But wait, use the script below to download and unpack everything.

The linux kernel has a config file which dictates which kernel options are enabled in the compilation process. What we need to do is enable these options, which are

```
CONFIG_KVM=y
CONFIG_VHOST_NET=m
```

When uncompressed, the `public_sources.tbz2` file will appear at `Linux_for_Tegra`. We also need to unpack at `Linux_for_Tegra/source/public/kernel_src.tbz2`.
The config file for tegra is at `Linux_for_Tegra/source/public/kernel/kernel-4.9/arch/arm64/configs/tegra_defconfig`

So let's do all of this in one shot. Remember that you'd have to change the kernel version and the link if you want newer kernels, and you should pick the kernel that matches your release for better compatibility. So:

```bash
#Installs dependencies for getting/building the kernel
sudo apt update && sudo apt-get install -y build-essential bc git curl wget xxd kmod libssl-dev

#Gets the kernel
cd ~/
wget https://developer.nvidia.com/embedded/l4t/r32_release_v5.1/r32_release_v5.1/sources/t210/public_sources.tbz2
tar -jxvf public_sources.tbz2
JETSON_NANO_KERNEL_SOURCE=~/Linux_for_Tegra/source/public/
tar -jxvf kernel_src.tbz2

# Applies the new configs to tegra_defconfig so KVM option is enabled
cd ${JETSON_NANO_KERNEL_SOURCE}/kernel/kernel-4.9
echo "CONFIG_KVM=y
CONFIG_VHOST_NET=m" >> arch/arm64/configs/tegra_defconfig
```

Compiling the kernel now would already activate KVM, but we would still miss an important feature that makes virtualization much faster: the irq chip. Without it, virtualization is still possible but an emulated irq chip is much slower. On `firecracker` (a virtualization tool written by AWS), it will not work as it requires this.

What we need to do is specify, in the device tree, the features of the irq chip on the CPU. The device tree is a file that contains addresses for all devices on the Jetson Nano chip. 

This must be done by hand. Apply the patch below to the file `Linux_for_Tegra/source/public/kernel_src/hardware/nvidia/soc/t210/kernel-dts/tegra210-soc/tegra210-bthrot-cdev.dtsi`. Don't use the patch tool as it'll likely not work, just do it by hand:

```
--- a/hardware/nvidia/soc/t210/kernel-dts/tegra210-soc/tegra210-soc-base.dtsi     2020-08-31 08:40:36.602176618 +0800
+++ b/hardware/nvidia/soc/t210/kernel-dts/tegra210-soc/tegra210-soc-base.dtsi     2020-08-31 08:41:45.223679918 +0800
@@ -351,7 +351,10 @@
                #interrupt-cells = <3>;
                interrupt-controller;
                reg = <0x0 0x50041000 0x0 0x1000
-                      0x0 0x50042000 0x0 0x0100>;
+                       0x0 0x50042000 0x0 0x2000
+                       0x0 0x50044000 0x0 0x2000
+                       0x0 0x50046000 0x0 0x2000>;
+               interrupts = <GIC_PPI 9 (GIC_CPU_MASK_SIMPLE(4) | IRQ_TYPE_LEVEL_HIGH)>;
                status = "disabled";
        };
```

as you see, we added more `reg` and `interrupts`. Now, when we compile the kernel image, we'll also compile device tree files from this `dsti` file.

Now we should compile everything:

```bash
JETSON_NANO_KERNEL_SOURCE=~/Linux_for_Tegra/source/public
TEGRA_KERNEL_OUT=$JETSON_NANO_KERNEL_SOURCE/build
KERNEL_MODULES_OUT=$JETSON_NANO_KERNEL_SOURCE/modules
cd $JETSON_NANO_KERNEL_SOURCE
# Generates the config file (you should manually enable/disable some missing by pressing y/n and enter)
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra tegra_defconfig
# Generates the Image that we're gonna place on /boot/Image
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra -j4 --output-sync=target zImage
# Generates the drivers. This is needed because the old driver will not work with our new Image
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra -j4 --output-sync=target modules
# Generates our modified device file trees
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra -j4 --output-sync=target dtbs
# Installs the modules on the build folder ~/Linux_for_Tegra/source/public/build
make -C kernel/kernel-4.9/ ARCH=arm64 O=$TEGRA_KERNEL_OUT LOCALVERSION=-tegra INSTALL_MOD_PATH=$KERNEL_MODULES_OUT modules_install
```

Now that we have our Image, the drivers and the file trees, we should override them, but before, make a manual backup of folders we're gonna change so you can rollback if something goes wrong.

```bash
sudo cp /boot /boot_original
sudo cp -r /lib /lib_original
```

```
cd $JETSON_NANO_KERNEL_SOURCE/modules/lib/
sudo cp -r firmware /lib/firmware
sudo cp -r modules /lib/modules
```

Now we can `rsync` the files with the system ones (warning, untested, I used `sudo nautilus` and moved by hand on mine).

```bash
rsync -avh firmware /lib/firmware
rsync -avh modules /lib/modules
```

Now we must also update the boot folder:

```bash
cd $JETSON_NANO_KERNEL_SOURCE/build/arc/arm64/
rsync -avh boot /boot
```

Notice that we copied all of the dtb files, there are many for different models, but just one that we should use. Run

```bash
   sudo dmesg | grep -i kernel
```

to discover yours. Example of mine:

```
[    0.236710] DTS File Name: /home/lz/Linux_for_Tegra/source/public/kernel/kernel-4.9/arch/arm64/boot/dts/../../../../../../hardware/nvidia/platform/t210/porg/kernel-dts/tegra210-p3448-0000-p3449-0000-a00.dts
```

Wait, wtf? Why this is a local file? I don't know what's happening, but this should show you which one is being used. You're gonna need its name. The file is already at `/boot`.

You might wonder that since we replaced all the device tree files on `/boot`, then it should load the modified one already. Somehow, in my case, it didn't. I think it has to do with the fact that it's loading a local one like shown above. If you know how to change this, open an issue please. Anyways, to bypass this, we have to inform the `/boot/extlinux/extlinux.conf` where to locate our file. Change from

```
TIMEOUT 30
DEFAULT primary

MENU TITLE L4T boot options

LABEL primary
      MENU LABEL primary kernel
      LINUX /boot/Image
      INITRD /boot/initrd
      APPEND ${cbootargs} quiet root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 loglevel=7 console=ttyS0,115200n8 console=tty0 fbcon=map:0 net.ifnames=0

```

to

```
TIMEOUT 30
DEFAULT primary

MENU TITLE L4T boot options

LABEL primary
      MENU LABEL primary kernel
      LINUX /boot/Image
      INITRD /boot/initrd
      FDT /boot/tegra210-p3448-0000-p3449-0000-a00.dtb
      APPEND ${cbootargs} quiet root=/dev/mmcblk0p1 rw rootwait rootfstype=ext4 loglevel=7 console=ttyS0,115200n8 console=tty0 fbcon=map:0 net.ifnames=0

```

that is, add the path to your dtb file. In my case, `FDT /boot/tegra210-p3448-0000-p3449-0000-a00.dtb`.

Note that you can add a second testing profile, which can be selected at boot time if you have a serial device to plug into the jetson nano like in this video https://www.youtube.com/watch?v=Kwpxhw41W50. When you boot you can select your second `LABEL` by typing its number. This is useful if you want to test different `Image`s without substituting the original one like we did.

Now reboot, and then run `ls /dev | grep kvm` to confirm if the `kvm` file exists. This means it's working. You should also run 

```bash
ls  /proc/device-tree/interrupt-controller
 compatible  '#interrupt-cells'   interrupt-controller   interrupt-parent   interrupts   linux,phandle   name   phandle   reg   status
```

and see that the node `interrupts`, which didn't exist before, was added. This means the irc interrupt activation worked.

You can run qemu/firecracker now. I only tested with firecracker though.