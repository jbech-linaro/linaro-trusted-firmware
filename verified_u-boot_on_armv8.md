How to run ARM-Trusted-Firmware and U-Boot with verification enabled
--------------------------------------------------------------------
Contents:

1. Prerequisites
2. Obtain the software
3. Prepare the software
4. Run the system in FVP
5. Verified U-Boot
    1. Overview
    2. Edit U-Boot’s board header file
    3. Obtain suitable images
    4. Generate RSA Key pairs with OpenSSL
    5. Create the Image Tree Source file
    6. Sign the kernel
    7. Build U-Boot FDT and put the public key into U-Boot's image
    8. Run the FIT image as below

# 1. Prerequisites
### Desktop
As hosting enviroment this has been tested using:

* Ubuntu 12.04.04
* Linux Mint 17

### FVP (Foundation_v8p)
This is the simulator used in this setup. It can be downloaded from
[ARMs](http://www.arm.com/zh/products/tools/models/fast-models/foundation-model.php)
site.

### Toolchain
We recommend that you follow the [ARM Trusted Firmware User
Guide](https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/user-guide.md#3--tools)
(section: Tools) to get and setup the correct toolchain.

### OpenSSL
U-Boot will need development libraries when compiling with verification enabled.
On a Ubuntu based system you will get this package by typing:
```
sudo apt-get install libssl-dev
```

### Device Tree Compiler
When enabling verified boot you are going to build device tree files, therefore
you also must install the device tree compiler.
```
sudo apt-get install device-tree-compiler
```

# 2. Obtain the software
### ARM Trusted Firmware
Obtain the latest version of ARM Trusted Firmware by issue:
```
git clone https://github.com/ARM-software/arm-trusted-firmware.git
```
Commit used when testing this:

* ```e73f4ef6072096584f44cb0046c78194df359e8a Merge pull request #219 from
jcastillo-arm/jc/tf-issues/253 ```

### RAM-disk
Obtain the RAM-disk by following the [ARM Trusted Firmware User
Guide](https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/user-guide.md#prepare-ram-disk)
(section: Prepare RAM-disk).

### Linux kernel
Obtain the Linux kernel by following the [ARM Trusted Firmware User
Guide](https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/user-guide.md#obtaining-a-linux-kernel)
(the first section called 1. Clone Linux:).
Commit used when testing this:

* ```b07e2d35062c1005d8a733851d4bc44bb6b790cc arm64: defconfig: disable unstable
features```

### U-Boot
The latest version of U-Boot can be downloaded from
[U-Boots](http://www.denx.de/wiki/U-Boot/SourceCode) site, or by cloning it from
here:
```
git clone git://git.denx.de/u-boot.git
```
Commit used when testing this:

* ```0d485b9095328cdc81b2ee94ff59b988c69b9127 Merge branch 'master' of
git://git.denx.de/u-boot-sunxi```

### Tree structure
In this setup we have had the following folder structure, so please refer to
this when reading about symlinks, its-files etc.
```
$HOME/devel
        ├── arm-trusted-firmware
        ├── Foundation_v8pkg
        ├── linux
        └── u-boot
```

# 3. Prepare the software
### Build U-boot
Before building U-Boot you need to make a couple of changes in the file
```.include/configs/vexpress_aemv8a.h``` since we must use GIC2 for the kernel
and we also must use the correct base address for U-Boot, therefore change
accordingly:
```
 #ifndef CONFIG_BASE_FVP
 /* Base FVP not using GICv3 yet */
-#define CONFIG_GICV3
+#define CONFIG_GICV2
 #endif
```
and later down in the same file
```
 #ifdef CONFIG_BASE_FVP
 /* ATF loads u-boot here for BASE_FVP model */
 #define CONFIG_SYS_TEXT_BASE           0x88000000
 #define CONFIG_SYS_INIT_SP_ADDR         (CONFIG_SYS_SDRAM_BASE + 0x03f00000)
 #else
-#define CONFIG_SYS_TEXT_BASE           0x80000000
+#define CONFIG_SYS_TEXT_BASE           0x88000000
 #define CONFIG_SYS_INIT_SP_ADDR         (CONFIG_SYS_SDRAM_BASE + 0x7fff0)
 #endif
```

Then build U-Boot.
```
    $ cd u-boot
    $ export CROSS_COMPILE=<toolchain_path>/bin/aarch64-none-elf-
    $ make vexpress_aemv8a_defconfig
    $ make all -j8
```

### Build Linux kernel and create uImage
If you haven't build the kernel for ARCH64, then please do as follows:
```
    $ export CROSS_COMPILE=<toolchain_path>/bin/aarch64-none-elf-
    $ cd <linux_kernel_path>
    $ make mrproper
    $ make ARCH=arm64 defconfig
```
Next you need to point to the root fs using the INITRAMFS_SOURCE flag in kernel.
Point to the file filesystem.cpio.gz.
```
    $ make ARCH=arm64 menuconfig
      ...
       |   Location:
       |     -> General setup
       |       -> Initial RAM filesystem and RAM disk (initramfs/initrd) support
```

Final step is to compile
```
    $ make ARCH=arm64 -j8
```

When the kernel image has been created we need to create images for use with the
U-Boot boot loader, this is achieved by:
```
    $ cd <linux_kernel_path>/arch/arm64/boot
    $ <u-boot_path>/tools/mkimage -A arm64 -O linux -T kernel \
        -C none -a 0x80080000 -e 0x80080000  -n 'linux-3.15' \
        -d Image uImage
```

Both load address and link address will and should be ```0x80080000```. The -n
parameter could be any name, for simplicity we use the same name as for the
version of the kernel we are using.

### Make the firmware image package (fip)
Next step is to build the firmware package in ARM-Trusted-Firmware Git. We need
to build:

* bl1.bin
* bl2.bin
* bl31.bin
* bl33.bin (which in this case should be/is u-boot.bin)

This could be achieved by typing following
```
    $ export BL33=<u-boot_path>/u-boot.bin
    $ export CROSS_COMPILE=<toolchain_path>/bin/aarch64-none-elf-
    $ cd <arm_tf_path>
    $ make -j8 PLAT=fvp all fip
```

### 4. Run the system in FVP
Start by creating symlinks to all of the images to the FVP directoy.
```
    $ cd <fvp_path>
    $ ln -s <arm_tf_path>/build/fvp/release/bl1.bin .
    $ ln -s <arm_tf_path>/build/fvp/release/fip.bin .
    $ ln -s <arm_tf_path>/fdts/fvp-foundation-gicv2-psci.dtb fdt.dtb
    $ ln -s <u-boot_path>/u-boot.bin .
    $ ln -s <linux_kernel_path>/arch/arm64/boot/uImage .
```

Starting Foundation as below:
```
$ /<fvp_path>/models/Linux64_GCC-4.1/Foundation_v8 \
        --cores=4 \
        --no-secure-memory \
        --visualization \
        --gicv3 \
        --data=bl1.bin@0x0 \
        --data=fip.bin@0x8000000 \
        --data=uImage@0x90000000 \
        --data=fdt.dtb@0xa0000000
```
Use U-boots ```bootm``` command to start the Linux kernel:

```
    $ bootm 0x90000000 - 0xa0000000
```
This maps to:
```
bootm ${kernel_addr} ${ramdisk_addr} ${fdt_addr}
```

# 5. Verified U-Boot
### 5.1. Overview
U-Boot’s verified boot was based on RSA signature. It uses cryptographic
algorithms to 'sign' software images. Images are signed using a private key
known only to the signer, but can be verified using a public key. The private
and public keys are mathematically related. The process looks like this:
```
      Signing                                      Verification
      =======                                      ============

 +--------------+                   *
 | RSA key pair |                   *             +---------------+
 | .key  .crt   |                   *             | Public key in |
 +--------------+       +------> public key ----->| trusted place |
       |                |           *             +---------------+
       |                |           *                    |
       v                |           *                    v
   +---------+          |           *              +--------------+
   |         |----------+           *              |              |
   | signer  |                      *              |    U-Boot    |
   |         |----------+           *              |  signature   |--> yes/no
   +---------+          |           *              | verification |
      ^                 |           *              |              |
      |                 |           *              +--------------+
      |                 |           *                    ^
 +----------+           |           *                    |
 | Software |           +----> signed image -------------+
 |  image   |
 +----------+
```


The FIT format is already widely used in U-Boot. It is a flattened device tree
(FDT) in a particular format, with images contained within. FITs include hashes
to verify images, so it is relatively straightforward to add signatures as well.
The public key can be stored in U-Boot's CONFIG_OF_CONTROL device tree in a
standard place. Then when a FIT it loaded it can be verified using that public
key. The steps are roughly as follows:

1. Edit U-Boot’s board head file, with the verified boot options enabled.
2. Obtain suitable images (kernel, FDT, Ram Disk etc)
3. Create a key pair
4. Create a Image Tree Source file (ITS)
5. Sign the kernel
6. Put the public key into U-Boot's image
7. Put U-Boot and the kernel onto the board

Also, you can follow U-boot’s official guide to setup verified boot, which is
located in `$uboot/doc/uImage.FIT/ beaglebone_vboot.txt`.

### 5.2. Edit U-Boot’s board header file, with the verified boot options enabled
Since we have verified this using Foundation Models, we have been using the
vexpress_aemv8a as the target board for U-Boot. Edit
```./include/configs/vexpress_aemv8a.h``` file, add macros as below:
```
    #define CONFIG_OF_CONTROL
    #define CONFIG_RSA
    #define CONFIG_FIT_SIGNATURE
    #define CONFIG_FIT
    #define CONFIG_OF_SEPARATE
```

It will fail compiling due to lack of a gpio.h file. The reason for this is
because of U-Boot's dependency design. We need to add a empty gpio.h file to the
path ```./arch/arm/include/asm/arch-armv8```, just like for other boards, do
like this:
```
mkdir -p ./arch/arm/include/asm/arch-armv8
touch ./arch/arm/include/asm/arch-armv8/gpio.h
```

### 5.3. Obtain suitable images
Reference step 1-4.

### 5.4 Generate RSA Key pairs with OpenSSL
```
    export key_dir=/path/to/your/keys/"
    export key_name="dev"
```
##### 5.4.1 Generate the private signing key as:
```
    $ openssl genrsa -F4 -out "${key_dir}"/"${key_name}".key 2048
```
##### 5.4.2. Generate the certificate containing the public key as:
```
    $ openssl req -batch -new -x509 -key "${key_dir}"/"${key_name}".key \
       -out "${key_dir}"/"${key_name}".crt
```

### 5.5 Create the Image Tree Source file
First, We need to generate a very simple u-boot device tree file to accommodate
the public key, lets call the file u-boot.dts and write the following in that
file:
```
/dts-v1/;

/ {
        model = "Keys";
        compatible = "linaroSwg";
        signature {
                key-dev {
                        required = "conf";
                        algo = "sha1,rsa2048";
                        key-name-hint = "dev";
                };
        };
};
```

Verified boot is based on the new U-Boot image format called FIT (Flattened
uImage Tree), so we need to create a image tree source file (*.its) that
describes the images in use, like the kernel image, FDT blob and the RAM-disk.
An its source file could look like below, create it using the name
fit-image.its. Alternatively, you can also create a new image tree source file
based on official’s example: `$uboot/doc/uImage.FIT /sign-configs.its`
```
/dts-v1/;

/ {
	description = "Linux kernel";
	#address-cells = <1>;
	images {
		kernel@1 {
			description = "Linux kernel";
			data = /incbin/("../linux/arch/arm64/boot/Image");
			arch = "arm64";
			os = "linux";
			type = "kernel_noload";
			compression = "none";
			load = <0x80080000>;
			entry = <0x80080000>;
			kernel-version = <1>;
            hash@1 {
                algo = "sha1";
            };
		};
		fdt@1 {
			description = "fdt blob";
			data = /incbin/("./arch/arm/dts/fvp-foundation-gicv2-psci.dtb");
			type = "flat_dt";
			arch = "arm64";
			compression = "none";
			fdt-version = <1>;
            hash@1 {
                algo = "sha1";
            };
		};
		ramdisk@1 {
			description= "ramdisk";
			data = /incbin/("../Foundation_v8pkg/filesystem.cpio.gz");
			type = "ramdisk";
			arch = "arm64";
			os = "linux";
			compression = "gzip";
			load = <0xaa000000>;
			entry = <0xaa000000>;
            hash@1 {
                algo = "sha1";
            };
		};
	};
	configurations {
		default = "conf@1";
		conf@1 {
			description = "Boot Linux kernel";
			kernel = "kernel@1";
			fdt = "fdt@1";
            ramdisk = "ramdisk@1";
            signature@1 {
                algo = "sha1, rsa2048 ";
                key-name-hint = "dev";
                sign-images = "fdt", "kernel", "ramdisk";
            };
		};
	};
};
```

Pay attention to section ```key-name-hint```, this points to the path of key
generated in our steps using OpenSSL above. Before we build the FIT image the
kernel image, FDT blob and the ramdisk must be prepared. How to configure
depends on the board you plan to use. You need to select load- and entry-address
for each sub image.

### 5.6. Sign the kernel
Build DTB blob file use u-boot.dts as below:
```
$ dtc -p 0x1000 u-boot.dts -O dtb -o u-boot.dtb
```

Build the FIT image and sign the DTB file for U-Boot as below:
```
$mkimage -D "-I dts -O dtb -p 2000" -f fit_image.its image.fit
```

Sign the kernel images and put the public key into u-boot’s image
```
mkimage -D "-I dts -O dtb -p 2000" -F -k key -K u-boot.dtb -r image.fit
```


### 5.7 Build U-Boot FDT and put the public key into U-Boot's image

```
$ make distclean
$ make vexpress_aemb8a_config
$ make CROSS_COMPILE=<> DEVICE_TREE=foundation all
$ make CROSS_COMPILE=<> EXT_DTB=<dtb file>
```
Note that, we copied the device tree file ```foundation.dts``` to U-Boot's
```arch/arm/dts``` file and made the corresponding modifications to the Makefile
(add  `dtb-$(CONFIG_ARM64) += foundation.dtb`). This is the object that
```DEVICE_TREE``` points to. ```EXT_DTB``` is the DTB file that we signed before
in make FIT image step: ```u-boot.dtb```. After this step was completed, public
key is found in the device tree. U-Boot can use this to verify the image that
was signed with the private key. ```u-boot-dtb.bin``` is the file that we need.
Next build ARM-TF and point BL33 to u-boot-dtb.bin.

### 5.8 Run the FIT image as below
```
$ /<path_to_fvp>/Foundation_v8 \
        --cores=4 \
        --no-secure-memory \
        --visualization \
        --gicv3 \
        --data=bl1.bin@0x0 \
        --data=fip.bin@0x8000000 \
        --data=image.fit@0xa2000004
```

Note: There exist an alignment problem in U-Boot's mkimage, we have debugged the
source code and found that sometimes you may encounter this problem. If you
encounter this problem you could try to modify the address that FIT image be
loaded. I.e, load ```image.fit``` to ```0xa2000004``` and it should be OK,
however, when we load it to ```0xA0000000``` there will be a abortion exception.
This problem is mainly related to mkimage tools and FIT format image parsing
routines.

After loaded images are verified, Use bootm command to boot kernel as:
```
    $ bootm 0xA0000004
```
PS: There are some problems in he latest version of U-Boot (v2014.10). You may
need to rollback ```gic_v64.S``` file as the older version when you open
```CONFIG_GICV2``` macro. Although we have reported the problem to U-Boot's
maintainer, we still recommend you to use the older version of U-Boot if you
want to verify this using Foundation Model. As an alternative you could
substitute the file ```gic_v64.S``` with the older version and add the
```#define CONFIG_GICV2``` macro in ```./include/configs/vexpress_aemv8a.h```.
