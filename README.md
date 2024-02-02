Hobgoblin Repo Manifests for the Yocto Project Build System
=============================================
This repository provides Repo manifests to setup the Yocto build
system for the Codasip Hobgoblin FPGA platform.

The Yocto Project allows the creation of custom linux distributions
for embedded systems.  It is a collection of git repositories known as
*layers* each of which provides *recipes* to build software packages
as well as configuration information.

Repo is a tool that enables the management of many git repositories
given a single *manifest* file.  Tell repo to fetch a manifest from
this repository and it will fetch the git repositories specified in
the manifest and, by doing so, setup a Yocto Project build environment
for you!

Getting Started
---------------
**1.  Install Repo.**

Download the Repo script:

    $ curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > repo

Make it executable:

    $ chmod a+x repo

Move it on to your system path:

    $ sudo mv repo /usr/local/bin/

If it is correctly installed, you should see a Usage message when invoked
with the help flag.

    $ repo --help

**2.  Set up ssh-agent.**

The Yocto recipes download from Codasip's Gitlab using ssh. To avoid
having to enter a passphrase every time ssh-agent needs to be set up.
Most desktop environments run ssh-agent when the system starts, so its
just necessary to add the key used for access to Gitlab:

    $ ssh-add

If this fails it probably means ssh-agent isn't running. It can be started by:

    $ eval `ssh-agent`

**3.  Initialize a Repo client.**

Create an empty directory to hold your working files:

    $ mkdir yocto
    $ cd yocto

Tell Repo where to find the manifest:

    $ repo init -u git@gitlab.codasip.com:cheri/software/cherilinux/linux/codasip-yocto-manifest.git -b <branch>

A successful initialization will end with a message stating that Repo is
initialized in your working directory. Your directory should now
contain a .repo directory where repo control files such as the manifest are
stored but you should not need to touch this directory.

***
**Note:**
You can use the **-b** switch to specify the branch of the repository
to use.  The **hobgoblin** branch is the default for non-Cheri Hobgoblin
support.

The **-m** switch selects the manifest file (default is *default.xml*).

To learn more about repo, look at [Repo Command Reference](https://source.android.com/source/using-repo "Using repo")
***

**4.  Fetch all the repositories:**

    $ repo sync

Now go turn on the coffee machine as this may take 20 minutes depending on
your connection.

**5.  Initialize the Yocto Project Build Environment.**

    $ source ./meta-codasip/hobgoblin_yocto_setup.sh

This copies default configuration information into the **build/conf**
directory and sets up some environment variables for the build system.  This configuration
directory is not under revision control; you may wish to edit these configuration
files for your specific setup.

**6.  Build an image:**

This process downloads over 10 gigabytes of source code and then
proceeds to do an awful lot of compilation so make sure you have
plenty of space (50GB minimum), and expect several hours of build time
depending on your network connection and host speed.  Don't worry---it
is just the first build that takes a while.

    $ bitbake core-image-minimal

If everything goes well, you should have a compressed root filesystem
tarball as well as kernel and bootloader binaries available in your
**tmp/deploy/images/hobgoblin** directory.  If you run into problems, the most likely
candidate is missing software packages.  Check out
[The Build Host Packages](http://www.yoctoproject.org/docs/current/yocto-project-qs/yocto-project-qs.html#resources "Yocto quick start guide")
for the list of required packages for operating system. Also, take
a look to be sure your operating system is supported:
[Distribution Supported List](https://wiki.yoctoproject.org/wiki/Distribution_Support "Yocto wiki")


**7.  Create a bootable micro SD card:**

To build an SD card image run:

    $ wic create -o sdcard hobgoblin -e core-image-minimal

The card image will be **sdcard/hobgoblin-<timestamp>-mmcblk0.direct**

Follow the instructions on the
[Linux Hobgoblin from scratch](https://codasip.atlassian.net/wiki/spaces/CHERI/pages/709394589/Linux+Hobgoblin+from+scratch) page for booting the SD card image.

Note that wic may not generate an image which is a power of two in size. This ca
be remided by running:

    $ qemu-img resize -f raw sdcard/hobgoblin-<timestamp>-mmcblk0.direct 256M

for example.


Staying Up to Date
------------------
To pick up the latest changes for all source repositories, run:

    $ repo sync

Enter the Yocto Project build environment:

    $ source poky/oe-init-build-env build

If you forget to setup these environment variables prior to bitbaking, your
OS will complain that it can't find bitbake on the path.  Don't try to
install bitbake using a package manager, just run the above command.

You can then rebuild as before:

    $ bitbake core-image-minimal

Starting Fresh
-------------------
So something broke... what do you do now?

There are several degrees of *starting fresh*: individual packages can be
rebuilt or the whole system can be reconstructed.

 1. clean a package: bitbake <package-name> -c cleansstate
 2. re-download package: bitbake <package-name> -c cleanall
 3. destroy everything but downloads: rm -rf build/sstate-cache build/tmp (or wherever your sstate and work directories are)
 4. destroy it all (not recommended): rm -rf build

***
**Note:**
If you've made a change to a recipe and want the package to be rebuilt, just
increment the recipe version (the PR variable); cleaning is not necessary.

To understand better how bitbake processes recipes, look at the excellent
documentation:
[Yocto Project Reference Manual](http://www.yoctoproject.org/docs/current/poky-ref-manual/poky-ref-manual.html)
***

To make sense of the differences between these cleaning methods, it is useful to
understand that Yocto caches both the downloaded source files for all the
packages it tries to build (the DL_DIR configuration parameter) and the packages
once built (the SSTATE_DIR configuration parameter). Typically, deleting the
downloaded source is a bad idea---this just means re-fetching gigabytes of code
which wastes network bandwidth. Cleaning the sstate cache for a particular
package ensures that it actually gets rebuilt from source rather than simply
restored from the cache.
