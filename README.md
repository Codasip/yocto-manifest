# Hobgoblin Repo Manifests for the Yocto Project Build System

This repository provides Repo manifests to setup the Yocto build
system for the Codasip Hobgoblin FPGA platform.

The Yocto Project allows the creation of custom linux distributions
for embedded systems.  It is a collection of git repositories known as
*layers* each of which provides *recipes* to build software packages
as well as configuration information. The reference distribution built by
Yocto is called Poky.

The version of Yocto provided here has been modified to support the
Codasip Hobgoblin platform, which consists of a Codasip A730
Application Processor with various peripherals to make a complete
system. This can be provided by an FPGA or QEMU simulator.

Repo is a tool that enables the management of many git repositories
given a single *manifest* file.  Tell Repo to fetch a manifest from
this repository and it will fetch the git repositories specified in
the manifest and, by doing so, setup a Yocto Project build environment
for you!

# Using the Prebuilt Release

A prebuilt release is available which provides:
* An image which can be copied to an SD card to boot Poky Linux
* An SDK for building applications to run on Linux
* An SDK for building baremetal programs which run without an operating system

# Using the Poky distribution SD card image

A tar file containing the images built by the Yocto system is available
to download from the releases page. Included in this is an
image which can be used to boot Poky Linux onto the Hobgoblin FPGA
platform or the QEMU simulator.

Download the tar file from the releases page, for example
`poky-image-R1.6.tar.gz`, and unpack it:
```
% tar xf poky-image-R1.6.tar.gz images
```
The image file is called
`images/hobgoblin/core-image-full-cmdline-hobgoblin.sdcard.wic`.

## Using the SD card image with QEMU

To use the SD card image with QEMU it is necessary to first install the
Poky SDK, which includes a version of QEMU modified to emulate the
Hobgoblin platform.

Download the Poky SDK from the releases page, for example
`codasip-poky-glibc-x86_64-meta-toolchain-riscv64-hobgoblin-toolchain-R1.6.sh`
and install it:
```
% chmod a+x codasip-poky-glibc-x86_64-meta-toolchain-riscv64-hobgoblin-toolchain-R1.6.sh
% ./codasip-poky-glibc-x86_64-meta-toolchain-riscv64-hobgoblin-toolchain-R1.6.sh
```
To use the SDK source the environment setup script:
```
% source /opt/codasip-poky/R1.6/environment-setup-riscv64-codasip-linux
```

QEMU requires that the SD card image size is a power of two, so first
resize it, for example:
```
% qemu-img resize -f raw core-image-full-cmdline-hobgoblin.sdcard.wic 512M
```
Then the SD card image can be used directly with QEMU:
```
% qemu-system-riscv64 \
  -machine hobgoblin,boot-from-rom=true \
  -nographic \
  -no-reboot \
  -drive file=core-image-full-cmdline-hobgoblin.sdcard.wic,format=raw,if=sd
```

## Using the SD card image on the Hobgoblin FPGA platform

To use the SD card image on the Hobgoblin FPGA platform you need a
bitstream file, which has the `.bit` extension, for example `system.bit`.
The contents of this file are loaded into the FPGA's
SRAM cells to configure the logic functions and circuit connections.

In addition, if the bitstream is configured to only support secure
boot then a secure boot ROM image is needed, for example `flash.bin`.

First copy the SD card image to an SD card. The device name for the
card will depend on the host system, but is likely to be something
like `/dev/sda`. Double and triple check the device name, as using
the wrong device may trash the host system.
```
% sudo dd if=core-image-full-cmdline-hobgoblin.sdcard.wic of=/dev/sda
```
To add the bitstream file to the SD card mount the first partition on the
card and copy the file:
```
% sudo mkdir -p /mnt/rootfs
% sudo mount /dev/sda1 /mnt/rootfs
% sudo cp system.bit /mnt/rootfs
```
If a secure boot ROM is needed then copy this to the card as well:
```
% sudo mkdir /mnt/rootfs/flash
% sudo cp flash.bin /mnt/rootfs/flash/
```
Finally unmount the SD card:
```
% sudo umount /mnt/rootfs
```
Set up a serial connection to the board using minicom or picocom, the
device is usually `/dev/USB0`. Baud rate is 115200, 8N1.

### Genesys II FPGA board

For the the
[Genesys II](https://digilent.com/reference/programmable-logic/genesys-2/start)
board plug the SD card into the slot J3. Set the board to boot from SD
card by setting jumper JP5 to "USB/SD" and JP4 to "SD". Power on the
board, or press the "PROG" button. It takes around 30 seconds to load the
bitstream, after which the orange busy light should go out and the
orange done light come on. The fan should switch off. If the busy
light pulses it indicates a programming error, usually this means it has been
unable to find the `.bit` file.

The serial port should show text from the various boot loaders, followed by
the kernel loading and then user-space starting up.

# Building from Source

If you want to customise the image, it is probably easiest to rebuild
it from scratch. Full instructions are provided in the
[Yocto Project Quick Build](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html)
however a brief overview is provided here specifically for the
Hobgoblin platform

## 1.  Install Repo

Download the Repo script:
```
% curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > repo
```
Make it executable:
```
% chmod a+x repo
```
Move it on to your system path:
```
% sudo mv repo /usr/local/bin/
```
If it is correctly installed, you should see a Usage message when invoked
with the help flag.
```
% repo --help
```

## 2.  Initialize Repo directory structure

Create an empty directory to hold your working files:
```
% mkdir yocto
% cd yocto
```
Tell Repo where to download the manifest:
```
% repo init -u https://github.com/codasip/yocto-manifest.git -b hobgoblin
```

A successful initialization will end with a message stating that Repo is
initialized in your working directory. Your directory should now
contain a .repo directory where repo control files such as the manifest are
stored but you should not need to touch this directory.

> [!NOTE]
> You can use the `-b` switch to specify the branch of the repository
> to use.

To learn more about repo, look at [Repo Command Reference](https://source.android.com/source/using-repo "Using repo")


## 3.  Fetch all the repositories

Download the various layers used in the Yocto build:
```
% repo sync
```

## 4.  Initialize the Yocto Project Build Environment

Setup the local environment for building using Yocto:
```
% source ./meta-codasip/hobgoblin_yocto_setup.sh
```
This copies default configuration information into the `build/conf`
directory and sets up some environment variables for the build system.  This configuration
directory is not under revision control; you may wish to edit these configuration
files for your specific setup.

## 5.  Accept the licence

All the software used in this Yocto build is covered by open source licences,
with one exception. The FSBL used to boot the platform is covered by a
Codasip commercial license. This license is provided as part of the
`meta-codasip` layer, and can be inspected
[here](https://github.com/Codasip/meta-codasip/blob/hobgoblin/licenses/Codasip).

To build the image it necessary to accept this licence by adding a line
to the file `conf/local.conf`:
```
% echo 'LICENSE_FLAGS_ACCEPTED += "commercial"' >> conf/local.conf
```

## 6.  Build an image

This process downloads over 10 gigabytes of source code and then
proceeds to do an awful lot of compilation so make sure you have
plenty of space (50GB minimum), and expect several hours of build time
depending on your network connection and host speed.  Don't worry---it
is just the first build that takes a while.
```
% bitbake core-image-minimal
```
If everything goes well, this should generate an image suitable for
using with the FPGA or QEMU Hobgoblin platform in the
`tmp/deploy/images/hobgoblin` directory. See the instructions above on how
to use this.

If you run into problems, the most likely
cause is missing software packages on the host system.  Check out
[The Build Host Packages](http://www.yoctoproject.org/docs/current/yocto-project-qs/yocto-project-qs.html#resources "Yocto quick start guide")
for the list of required packages for operating system. Also, take
a look to be sure your operating system is supported:
[Distribution Supported List](https://wiki.yoctoproject.org/wiki/Distribution_Support "Yocto wiki")

## Staying Up to Date

To pick up the latest changes for all source repositories, run:
```
% repo sync
```

Enter the Yocto Project build environment:
```
% source poky/oe-init-build-env build
```
If you forget to setup these environment variables prior to bitbaking, your
OS will complain that it can't find bitbake on the path.  Don't try to
install bitbake using a package manager, just run the above command.

You can then rebuild as before:
```
% bitbake core-image-minimal
```
Try not to run the script `meta-codasip/hobgoblin_yocto_setup.sh` a second time,
as this will modify the files under `conf` which you may have modified.

## Starting Fresh

So something broke... what do you do now?

There are several degrees of *starting fresh*: individual packages can be
rebuilt or the whole system can be reconstructed.

* clean a package: `bitbake <package-name> -c cleansstate`
* re-download package: `bitbake <package-name> -c cleanall`
* destroy everything but downloads: `rm -rf build/sstate-cache build/tmp` (or wheever your sstate and work directories are)
* destroy it all and start from scratch: `rm -rf build`

To understand better how bitbake processes recipes, look at the excellent
documentation:
[Yocto Project Reference Manual](http://www.yoctoproject.org/docs/current/poky-ref-manual/poky-ref-manual.html)

To make sense of the differences between these cleaning methods, it is useful to
understand that Yocto caches both the downloaded source files for all the
packages it tries to build (the `DL_DIR` configuration parameter) and the packages
once built (the `SSTATE_DIR` configuration parameter). Typically, deleting the
downloaded source is a bad idea---this just means re-fetching gigabytes of code
which wastes network bandwidth. Cleaning the sstate cache for a particular
package ensures that it actually gets rebuilt from source rather than simply
restored from the cache.
