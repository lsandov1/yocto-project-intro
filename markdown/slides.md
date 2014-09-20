# Meeting the Yocto Project

## What is the Yocto Project?

<blockquote>
"The Yocto Project provides open source, high-quality infrastructure and tools to help
developers create their own custom Linux Distribution for any hardware architecture,
across multiple market segments. The Yocto Project is intended to provide a helpful
starting point for developers"
</blockquote>

<blockquote>
"It's not an embedded Linux distribution
- it creates a custom one for you"
</blockquote>

Visit the Yocto Project [Page](https://www.yoctoproject.org/)

## Terminology

* **Poky**: is the Yocto Project reference system and it is composed of a collection of tools
and metadata: `meta-yocto` for Yocto-specific Metadata and `meta-yocto-bsp` for Yocto-specific 
BSP 
* **OpenEmbedded-Core**: `metadata` collection that provides teh engine of Poky build: `meta` 
* **BitBake**:  task scheduler that parses Python and Shell Script  mixed code. The code
parsed generates and run *tasks* (`fetch`, `patch`, `configure`, `compile`, etc.)
* **Metadata**: Configuration files + Class files + Recipe files. Indicates `bitbake` how to
build an image

# Baking Our Poky-based System

## Configuring a host system

* Debian system 
	
~~~~{.bash}
$: sudo apt-get install gawk wget git-core diffstat unzip texinfo gcc-multilib \
	  build-essential chrpath libsdl1.2-dev xterm
~~~~

## Downloading the Poky and Freescale layers

* The `repo` tool helps us to do the download of multiple Git repositories

~~~~{.bash}
$: mkdir ~/bin
$: curl http://commondatastorage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
$: chmod a+x ~/bin/repo
~~~~

* Get the `manifest.xml` file containing the Git repositories links

~~~~{.bash}
$: PATH=${PATH}:~/bin
$: mkdir fsl-community-bsp
$: cd fsl-community-bsp
$: repo init -u https://github.com/Freescale/fsl-community-bsp-platform -b daisy
~~~~

* Finally, download the metadata

~~~~{.bash}
$: repo sync
~~~~

* Instructions may change so for latest instructions follow this [link](http://freescale.github.io/#download)

## Preparing the build environment

~~~~{.bash}
$: MACHINE=imx6qsabresd # other available machines are on sources/meta*/conf/machine*.conf
$: source setup-environment build
~~~~

## Knowing the `local.conf` file

* Located on `build/conf/local.conf`
* User configuration file

~~~~{.python}
BB_NUMBERS_THREADS ?= "${@oe.utils.cpu_count()}"
PARALLEL_MAKE ?= "${@oe.utils.cpu_count()}"
MACHINE ??= "imx6qsabresd"
~~~~

* List of available machines

~~~~{.bash}
$: ls sources/meta*/conf/machine/*.conf
~~~~

## Building a target image

* List of available images

~~~~{.bash}
$: ls meta*/recipes*/images/*.bb
~~~~

* Build

~~~~{.bash}
$: bitbake core-image-minimal
~~~~

* Flash a SD card (do not forget to umount any partitions when SD is inserted on host!)

~~~~{.bash}
$: sudo dd \
    if=tmp/deploy/images/$MACHINE/core-image-minimal-imx6qsabresd.sdcard \
    of=/dev/sdb \
    bs=4M
$: sync
~~~~

## Using Hob to bake an image

~~~~{.bash}
$: source setup-environment [build-directory]
$: hob
~~~~
	
# Grasping the BitBake Tool

## Metadata

The metadata used by `bitbake` can be classified into three major areas

* Configuration (the `.conf` files). Example [imx6qsabresd.conf](https://github.com/Freescale/meta-fsl-arm/blob/master/conf/machine/imx6qsabresd.conf)
* Classes (the `.bbclass` files). Example [image_types_fsl.bbclass](https://github.com/Freescale/meta-fsl-arm/blob/master/classes/image_types_fsl.bbclass)
* Recipes (the `.bb` and `.bbappend` files). Example [linux-imx_3.10.17.bb](https://github.com/Freescale/meta-fsl-arm/blob/master/recipes-kernel/linux/linux-imx_3.10.17.bb)

## Tasks

* Each recipe is composed of tasks, and these are executed in order with `bitbake`
	
	1. Fetching (`build/../downloads`)
	2. Source Preparation, for example unpacking a tarball and patching (`build/tmp/work`)
	3. Configure and building
	4. Installing (`build/tmp/work/<...>/image`)
	5. Wrapping the sysroot (`build/tmp/sysroot`)
	6. Creating a package

* To list the recipe's last

~~~~{.bash}
$: bitbake linux-imx -c listtasks
~~~~

* To execute one particular task

~~~~{.bash}
$: bitbake -c <task> linux-imx
~~~~

* To execute all tasks

~~~~{.bash}
$: bitbake linux-imx
~~~~

# Temporaly Build Directory

## `build/tmp` structure
* `conf`: configuration files
* `sstate-cache`: package data snapshots (needed to avoid recompilation)
* `tmp`: temporal directory
	* `deploy`: build products such as images, packages and SDKs
	* `sysroot`: shared libraries, headers and utilities that are used in the process of building recipes
	* `work`: working source code, a task's configuration, execution logs and the contents of generated packages
		* Split by architecture
		* `<arch>/<recipe name>/<software version>`
			* `<sources>`: extracted source code (pointed by `WORKDIR` variable)
			* `image`: files installed by the recipe (pointed by `D` variable)
			* `packages`: extracted content of packages
			* `packages-split`: extracted content of split packages
			* `temp`: task code and execution logs

# Developing with the Yocto Project

## Working with the SDK

* **Software Development Kit (SDK)** is a set of tools and files used to develop and debug
	* Compiler
	* Linkers
	* Debugger
	* External library headers
	* Binaries

* Commonly called a **toolchain**

## Developing applications on the target

* Create the toolchain

~~~~{.bash}
$: bitbake core-image-minimal -c populate_sdk
~~~~

* Install the toolchain

~~~~{.bash}
$: ./tmp/deploy/sdk/poky-eglibc-x86_64-core-image-minimal-cortexa9hf-vfp-neon-toolchain-1.6.1.sh 
Enter target directory for SDK (default: /opt/poky/1.6.1): 
Extracting SDK...done
Setting it up...done
SDK has been successfully set up and is ready to be used.
~~~~

## Using the SDK

* Compiling a `hello-world`

~~~~{.c}
#include <stdio.h>

int main(int argc, char **argv)
{
    printf("Hello World!\n");

    return 0;
}
~~~~

~~~~{.bash}
$: source /opt/poky/1.6.1/environment-setup-cortexa9hf-vfp-neon-poky-linux-gnueabi
$: ${CC} helloworld.c -o helloworld
$: file helloworld
helloworld: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.16, BuildID[sha1]=007629f5556cd73793956192010e4e7f299d667d, not stripped
~~~~

* Compiling a Linux Kernel

~~~~{.bash}
$: source /opt/poky/1.6.1/environment-setup-cortexa9hf-vfp-neon-poky-linux-gnueabi
$: unset LDFLAGS
$: cd <path to my kernel source code>
$: make <defconfig>
$: make uImage
~~~~

## Native compilation on target

* The native build is prefered on newer microprocessors with more processing power and memory
* Add `EXTRA_IMAGE_FEATURES += "dev-pkgs tools-sdk"` on your `build/conf/local.conf` file
* Build and flash your image

~~~~{.bash}
$: bitbake core-image-minimal
$: sudo dd if=tmp/deploy/images/$MACHINE/core-image-minimal-imx6qsabresd.sdcard of=/dev/sdb bs=4M;sync
~~~~

# Creating Custom Layers

## Creating a custom layer

~~~~{.bash}
$: cd fsl-community-bsp/sources
$: ./poky/scripts/yocto-layer create newlayer
Please enter the layer priority you'd like to use for the layer: [default: 6] 
Would you like to have an example recipe created? (y/n) [default: n] y
Please enter the name you'd like to use for your example recipe: [default: example] 
Would you like to have an example bbappend file created? (y/n) [default: n] y
Please enter the name you'd like to use for your bbappend file: [default: example] 
Please enter the version number you'd like to use for your bbappend file (this should match the recipe you're appending to): [default: 0.1] 

New layer created in meta-newlayer.

Don't forget to add it to your BBLAYERS (for details see meta-newlayer\README).
~~~~

## Adding Metadata: a new package recipe

Below recipe found on `meta-newlayer/recipes-example/example/example_0.1.bb`

~~~~{.python}
#
# This file was derived from the 'Hello World!' example recipe in the
# Yocto Project Development Manual.
#

DESCRIPTION = "Simple helloworld application"
SECTION = "examples"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"
PR = "r0"

SRC_URI = "file://helloworld.c"

S = "${WORKDIR}"

do_compile() {
       ${CC} helloworld.c -o helloworld
}

do_install() {
       install -d ${D}${bindir}
       install -m 0755 helloworld ${D}${bindir}
}
~~~~

## Adding Metadata: a new image

* On the new layer, create a `recipes-my/images` folder 

~~~~{.bash}
$: mkdir -p meta-newlayer/recipes-my/images
~~~~

* Under `recipes-my/images`, you can create any image you want:

  * `my-image.bb`: Same image as `core-image-minimal` but including the `helloworld`
    application

~~~~{.python}
require recipes-core/images/core-image-minimal.bb
CORE_IMAGE_EXTRA_INSTALL += "hello-world"
~~~~

  * `my-image-dev.bb`: Same image as `core-image-minimal` including development tools
    allowing *native* compilation

~~~~{.python}
require recipes-core/images/core-image-minimal.bb
IMAGE_FEATURES += "dev-pkgs tools-sdk tools-debug tools-profile"
~~~~
