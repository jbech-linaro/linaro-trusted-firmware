How to run ARM-Trusted-Firmware and U-Boot with verification enabled
--------------------------------------------------------------------
Contents:

1. Prerequisites
2. Obtain the software
3. Prepare the software
4. Run the system in FVP
5. Verified U-Boot

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

### RAM-disk / initrd
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
    $ make -j8 ARCH=arm64
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
    $ ln -s <arm_tf_path>/build/fvp/release/bl2.bin .
    $ ln -s <arm_tf_path>/build/fvp/release/bl31.bin .
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
        --data=fdt.dtb@0xa0000000 \
        --data=filesystem.cpio.gz@0xa1000000
```
Use U-boots ```bootm``` command to start the Linux kernel:

```
    $ bootm 0x90000000 0xa1000000:size 0xa0000000
```
This maps to:
```
bootm ${kernel_addr} ${ramdisk_addr} ${fdt_addr}
```

# 5. Verified U-Boot
1. Since we have verified this using Foundation Models, we have been using the vexpress_aemv8a as the target board for U-Boot.
Edit ```./include/configs/vexpress_aemv8a.h``` file, add macros as below:
```
    #define CONFIG_OF_CONTROL
    #define CONFIG_RSA
    #define CONFIG_FIT_SIGNATURE
    #define CONFIG_FIT
    #define CONFIG_OF_SEPARATE
```
Recompile again. It might happen that it will fail compiling due to lack of a gpio.h file. The reason for this is because of U-Boot's dependency design. We need to add a empty gpio.h file to the path ```./arch/arm/include/asm/arch-armv8```, just like for other boards, do like this:
```
mkdir -p ./arch/arm/include/asm/arch-armv8
touch ./arch/arm/include/asm/arch-armv8/gpio.h
```

2. Generate RSA Key pairs with OpenSSL
```
    export key_dir=/path/to/your/keys/"
    export key_name="dev"  
```
3. Generate the private signing key as:  
```
    $ openssl genrsa -F4 -out "${key_dir}"/"${key_name}".key 2048  
```
4. Generate the certificate containing the public key as:
```
    $ openssl req -batch -new -x509 -key "${key_dir}"/"${key_name}".key \
       -out "${key_dir}"/"${key_name}".crt  
```

5. Create the FIT file.
Verified boot is based on the new U-Boot image format called FIT (Flattened uImage Tree), so we need to create a image tree source file (*.its) that describes the images in use, like the kernel image, FDT blob and the RAM-disk. An its source file could look like below
```
{
　　description = "Verified boot";
　　\#address-cells = <1>;
　　images {
　　　　　　kernel@1 {
　　　　　　　　description = “kernel image”;
　　　　　　　　data = /incbin/("Image");
　　　　　　　　type = "kernel_noload";
　　　　　　　　arch = "arm64";
　　　　　　　　os = "linux";
　　　　　　　　compression = "none";
　　　　　　　　load = <0x80080000>;
　　　　　　　　entry = <0x80080000>;
　　　　　　　　kernel-version = <1>;
　　　　　　　　signature@1 {
　　　　　　　　　　algo = "sha1,rsa2048";
　　　　　　　　　　key-name-hint = "dev";
　　　　　　　　};
　　　　　　};
　　　　　　fdt@1 {
　　　　　　　　description = "fdb blob";
　　　　　　　　data = /incbin/("atf_psci.dtb");
　　　　　　　　type = "flat_dt";
　　　　　　　　arch = "arm64";
　　　　　　　　compression = "none";
　　　　　　　　fdt-version = <1>;
　　　　　　　　signature@1 {
　　　　　　　　　　algo = "sha1,rsa2048";
　　　　　　　　　　key-name-hint = "dev";
　　　　　　　　};
　　　　　　};  
　　　　　　ramsidk@1 {
　　　　　　　　description=”ramdisk”;
　　　　　　　　data = /incbin/(“filesystem.cpio.gz”);
　　　　　　　　type = “ramdisk”;
　　　　　　　　arch = "arm64";
　　　　　　　　os = "linux";
　　　　　　　　compression = "gzip";
　　　　　　　　load = <0xaa000000>;
　　　　　　　　entry = <0xaa000000>;
　　　　　　　　signature@1 {
　　　　　　　　　　algo = "sha1,rsa2048";
　　　　　　　　　　key-name-hint = "dev";
　　　　　　　　};
　　　　　　};
　　};
　　configurations {
　　　　　　default = "conf@1";
　　　　　　conf@1 {
　　　　　　　　kernel = "kernel@1";
　　　　　　　　fdt = "fdt@1";
　　　　　　　　ramdisk = “ramdisk@1”;
　　　　　　};
　　};
};
```

Pay attention to section ```key-name-hint```, this points to the path of key  generated in our steps using OpenSSL above. Before we build the FIT image the kernel image, FDT blob and the ramdisk must be prepared. How to configure depends on the board you plan to use. You need to select load- and entry-address for each sub image. Build the FIT image and sign the DTB file for U-Boot as below:
```
    $ cp fvp-psci-gicv2.dtb atf_psci_public.dtb  
    $ mkimage -D "-I dts -O dtb -p 2000" -f kernel.its \
        -k <path_to_key> -K atf_psci_public.dtb -r image.fit  
```
I selected fvp-psci-gicv2.dtb that located in firmware's dts directory to be signed with public key.  


+ Build FDT for U-Boot
```
    $ make distclean  
    $ make vexpress_aemb8a_config  
    $ make CROSS_COMPILE=<> DEVICE_TREE=foundation all  
    $ make CROSS_COMPILE=<> EXT_DTB=<dtb file>  
```
Note that, we copied the device tree file ```foundation.dts``` to U-Boot's ```arch/arm/dts``` file and made the corresponding modifications to the Makefile. This is the object that ```DEVICE_TREE``` points to. ```EXT_DTB``` is the DTB file that we signed before in make FIT image step: ```atf_psci_public.dtb```. After this step was completed, public key is found in the device tree.
U-Boot can use this to verify the image that was signed with the private key.
```U-boot-dtb.bin``` is the file that we need. Next build ARM-TF and point BL33 to u-boot-dtb.bin.

+ Run the FIT image as below
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

Note: There exist an alignment problem in U-Boot's mkimage, we have debugged the source code and found that sometimes you may encounter this problem. To avoid this problem, modify the address that FIT image be loaded.
I.e, load ```image.fit``` to ```0xa2000004``` and it should be OK, however, when I load it to ```0xB0000000``` there will be a abortion exception. This problem is mainly related to mkimage tools and FIT format image parsing routines.

After loaded images are verified, Use bootm command to boot kernel as :  
```
    $ bootm 0xB0000004
```
PS: There are some problems in he latest version of U-Boot (v2014.10). You may need to rollback ```gic_v64.S``` file as the older version when you open ```CONFIG_GICV2``` macro. Although we have reported the problem to U-Boot's maintainer, we still recommend you to use the older version of U-Boot if you want to verify this using Foundation Model. As an alternative you could substitute the file ```gic_v64.S``` with the older version and add the ```#define CONFIG_GICV2``` macro in ```./include/configs/vexpress_aemv8a.h```.
