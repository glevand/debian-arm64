# Debian 10 Buster On Cavium ThunderX2

Some utilities and notes for running [Debian 10 Buster](https://wiki.debian.org/DebianBuster) on machines with Cavium's ARM64 [ThunderX2](https://www.cavium.com/product-thunderx2-arm-processors.html) processors.

Please report any problems encountered during install, any missing kernel config options, etc. through the [issue tracker](../../issues) of this project.

## Debian 10 Install

The current Debian 10 Buster [`firmware-nonfree`](https://packages.debian.org/buster/firmware-qlogic) package includes the firmware files needed for the Qlogic network adapter found on many ThunderX2 machines.  Debian policy does not allow these firmware files to be included in the Debian distribution proper and the user must arrange for the files to be available to the Debian installer.  Firmware files available during the installation will be copied to the installed system.  Once the system reboots, install the [firmware-qlogic package](#install-firmware-qlogic-package).


Typical Debian installer dialog message for missing non-free firmware:

```
    ┌───────────────────┤ [!] Detect network hardware ├────────────────────┐
    │                                                                      │
    │ Some of your hardware needs non-free firmware files to operate. The  │
    │ firmware can be loaded from removable media, such as a USB stick or  │
    │ floppy.                                                              │
    │                                                                      │
    │ The missing firmware files are:                                      │
    │ qed/qed_init_values_zipped-8.37.2.0.bin                              │
    │                                                                      │
    │ If you have such media available now, insert it, and continue.       │
    │                                                                      │
    │ Load missing firmware from removable media?                          │
    │                                                                      │
    │     <Yes>                                                   <No>     │
    │                                                                      │
    └──────────────────────────────────────────────────────────────────────┘
```


For more info on Debian installation and installation with firmware files see:

* [Debian Installation Guide](https://d-i.debian.org/manual/en.arm64/)
* [Firmware during the installation](https://wiki.debian.org/Firmware#Firmware_during_the_installation)
* [Netbooting and Firmware](https://wiki.debian.org/DebianInstaller/NetbootFirmware)

### Firmware From Removable Media

Firmware can be loaded from removable media.  See: [Loading Missing Firmware](https://d-i.debian.org/manual/en.arm64/ch06s04.html).

### Firmware From Custom Initrd

A custom installer initrd that includes the needed firmware files can be created.

The [releases page](../../releases) of this project has pre-build netboot initrd images that can be used.

To create a custom initrd yourself use commands like these:

```sh
# download
fw_version="8.37.2.0"
wget -O initrd-orig.gz https://d-i.debian.org/daily-images/arm64/daily/netboot/debian-installer/arm64/initrd.gz
wget https://d-i.debian.org/daily-images/arm64/daily/MD5SUMS
egrep 'netboot/debian-installer/arm64/initrd.gz' MD5SUMS
md5sum initrd-orig.gz
wget https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/tree/qed/qed_init_values_zipped-${fw_version}.bin

# extract files
rm -rf initrd-files
mkdir initrd-files
(cd initrd-files && cat ../initrd-orig.gz | gunzip | cpio --extract --make-directories --preserve-modification-time --verbose)

# edit files
mkdir -p initrd-files/lib/firmware/qed
cp -v qed_init_values_zipped-${fw_version}.bin initrd-files/lib/firmware/qed/
echo 'base-config     apt-setup/non-free      boolean true' > initrd-files/preseed.cfg

# archive
(cd initrd-files && find . | cpio --create --format='newc' --owner=root:root | gzip > ../initrd-qed-${fw_version}.gz)
```

### Install via Temporary Ethernet Device

Installation can be done with a USB-to-Ethernet adapter or an Ethernet PCI adapter temporarily installed to the system.  The alternate device itself must not require additional firmware to operate.  When prompted for missing qed firmware files choose to continue the installation without loading those firmware files.  Once the system reboots, install the [firmware-qlogic package](#install-firmware-qlogic-package) and update the system's [network configuration](https://www.debian.org/doc/manuals/debian-reference/ch05) to use the Qlogic network adapter and remove the temporary network adapter from the system. 

### Install firmware-qlogic package

To install firmware files needed for the Qlogic network adapter, once the OS installation is complete and the system has been re-booted to the newly installed OS, update the sources.list to include `non-free` packages, install the Debian `firmware-qlogic` package, and then re-boot:

```sh
sudo sed --in-place='.orig' 's/main$/main contrib non-free/' /etc/apt/sources.list
sudo apt update
sudo apt install firmware-qlogic
```

The Qlogic network adapter should then be operational.  The system should automatically update the installed firmware to match the installed kernel.

## Utilities

* `build-kernel-builder.sh` - Builds a Debian based Docker container (buster-kernel-builder) that has all the packages pre-installed that are needed to build the Debian Linux kernel.
* `run-kernel-builder.sh` - Starts the buster-kernel-builder container in interactive mode.
* `build-debian-kernel.sh` - Will build the Debian Buster kernel when run from inside the buster-kernel-builder container.

Note that these utilities must be run on an ARM64 machine or through ARM64 emulation.

To install Docker on Debian systems see [install-docker-ce](https://docs.docker.com/install/linux/docker-ce/debian/#install-docker-ce).  Check the [buster package pool](https://download.docker.com/linux/debian/dists/buster/pool) to see what Docker versions are available in the `stable` and `edge` repositories.

### To Build A Custom Debian Kernel

To build a custom kernel use commands like these:

On the host:

```sh
$ ./docker/build-kernel-builder.sh --help
$ ./docker/run-kernel-builder.sh --help

$ ./docker/build-kernel-builder.sh
$ ./docker/run-kernel-builder.sh
```

Inside the `buster-kernel-builder` container:

```sh
# /thunderx-debian/utils/build-debian-kernel.sh --help
# /thunderx-debian/utils/build-debian-kernel.sh
```

Or

```sh
# cp -a /usr/src/linux-4.xx.yy .
# cd linux-4.xx.yy
# make -f debian/rules.gen setup_arm64_none

# cp debian/build/build_arm64_none_arm64/.config .
# sed --in-place 's|CONFIG_SYSTEM_TRUSTED_KEYS=.*|CONFIG_SYSTEM_TRUSTED_KEYS=""|' .config
# make oldconfig
# make-kpkg clean
# make-kpkg -j200 --revision=1.0tx2 --initrd binary-arch
# make-kpkg -j200 --revision=1.0tx2 kernel_source
```

To install, on the host:

```sh
# dpkg -i ${WORK_DIR}/linux-image-4.xx.yy_1.0tx2_arm64.deb ${WORK_DIR}/linux-headers-4.xx.yy_1.0tx2_arm64.deb
```

For more info on building a custom Debian kernel see the Debian [kernel-package manpage](https://manpages.debian.org/testing/kernel-package/kernel-package.5.en.html).
