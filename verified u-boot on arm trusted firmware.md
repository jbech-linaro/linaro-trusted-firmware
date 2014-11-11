# Tools
1. FVP : Foundation_v8p. The simulator used to test. It can be downloaded from
[ARMs](http://www.arm.com/zh/products/tools/models/fast-models/foundation-model.php) site.
2. Linux server: Host working environment, we have verified this using Ubuntu 12.04.04 and Linux Mint 17.
3. Cross compiler: [gcc-linaro-aarch64-none-elf-4.8-2013.11_linux.tar.xz](http://releases.linaro.org/13.11/components/toolchain/binaries/gcc-linaro-aarch64-none-elf-4.8-2013.11_linux.tar.xz)
4. FIXME: We need to pre-install OpenSSL packages also, which packages? For me it was enough with ```sudo apt-get install libssl-dev```.

# Prepare Software
1. RAM-disk initrd: Test version can be downloaded from Linaro, Details can be found in ARM-Trusted-Firmware's [User Guide](https://github.com/ARM-software/arm-trusted-firmware/blob/master/docs/user-guide.md). FIXME: Where? Initrd isn't stated there.

2. Linux kernel. FIXME: Nothing to do here? Why state it?

3. Arm Trusted Firmware. FIXME: Nothing to do here? Why state it?

4. U-Boot: Universal boot loader
The latest version of U-Boot can be downloaded from [U-Boots](http://www.denx.de/wiki/U-Boot/SourceCode) site, or by cloning it from here ```git clone git://git.denx.de/u-boot.git```

# Boot system

### 1) Build U-boot
We are going to run the system on Foundation Models, therefore out target board will be vexpress_aemv8a. ```vexpress_aemv8a_semi_config``` can be selected when you run on FVP platform. Modify the macro ```CONFIG_SYS_TEXT_BASE``` which is located in the file ```include/configs/vexpress_aemv8a.h```. BL31 will jump to address ```0x88000000```, ```CONFIG_SYS_TEXT_BASE``` should be modified to this value. FIXME: It is already this value when building with value CONFIG_BASE_FVP.

Compile U-Boot as below:
```
    $ cd uboot
    $ make CROSS_COMPILE=<path>/bin/aarch64-none-elf- distclean
    
    $ make vexpress_aemv8a_config
    # The latest version you should use vexpress_aemv8a_defconfig

    $ make CROSS_COMPILE=<path>/bin/aarch64-none-elf- all
```

### 2) Make uImage
Supposed that Linux image has created. FIXME: How to we create the Linux image?
```
    $ cd <linux_kernel_path>/arch/arm64/boot
    $ /<uboot_path>/tools/mkimage -A arm64 -O linux -T kernel -C none -a 0x80080000 -e 0x80080000  -n 'linux-3.15' -d Image uImage
```
    
Both load address and link address will and should be ```0x80080000```.

### 3) Make Firmware Package (fip)
Firmware package includes: ```bl1.bin/ bl2.bin/ bl31.bin/ bl33.bin (uboot.bin)```. Set the variable BL33 as ```<path_to_uboot_directory>/uboot.bin```
```
    $ cd <firmware_path>
    $ make CROSS_COMPILE=<path>/bin/aarch64-none-elf- PLAT=fvp BL33=<path_to_uboot_directory>/uboot.bin all fip.bin
```

### 4) Running the system
+ Copy all images including ```bl1.bin, fip.bin, uImage, ramdisk, fdt blob``` to the work directory, then enter.
+ Starting Foundation as below:
```
$ /<path_to_fvp>/Foundation_v8 \
        --cores=4 \
        --no-secure-memory \
        --visualization \
        --gicv3 \
        --data=bl1.bin@0x0 \
        --data=fip.bin@0x8000000 \
        --data=uImage@0x90000000 \
        --data=ramdisk_file@0xa1000000 \
        --data=fdt.dtb@0xa0000000
```
PS: --data command can be used to load the image into FVP’s memory
Next, boot the kernel and once the firmware has successfully been started, the system will stop at U-Boot’s shell.
+ We can use U-boots ```bootm``` command to start Linux kernel as below:
```
    $ bootm 0x90000000 0xa1000000:size 0xa0000000
```
```0x90000000``` is the kernel’s address and ```0xa0000000``` is device tree DTB address. ```0xa1000000``` is the ramdisk’s address, also as a final step we need to provide the size for the ramdisk.

### 5) Verified U-Boot
+ Since we have verified this using Foundation Models, we chosen the vexpress_aemv8a as the target board for U-Boot.
Edit ```vexpress_aemv8a.h``` file, add macros as below:
```
    #define CONFIG_OF_CONTROL
    #define CONFIG_RSA
    #define CONFIG_FIT_SIGNATURE
    #define CONFIG_FIT
    #define CONFIG_OF_SEPARATE
```
Recompile again. It might happen that it will fail compiling due to lack of a gpio.h file. The reason for this is because of U-Boot's dependency design.
We need to add a empty gpio.h file to the path ```arch\arm\include\asm\arch-armv8```, just like other boards. FIXME: Didn't solve the problem.

+ Generate RSA Key pairs with OpenSSL
```
    key_dir=/path/to/your/keys/"
    key_name="dev"  
```
Generate the private signing key as:  
```
    $ openssl genrsa -F4 -out "${key_dir}"/"${key_name}".key 2048  
```
Generate the certificate containing the public key as:
```
    $ openssl req -batch -new -x509 -key "${key_dir}"/"${key_name}".key \
       -out "${key_dir}"/"${key_name}".crt  
```

+ Edit FIT descriptor file.
Verified boot is based on new U-Boot image format FIT, so we need to create a device tree file that describes the information about images, including kernel image, FDT blob and the RAMDISK. A FIT file could look like below
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
    $ mkimage -D "-I dts -O dtb -p 2000" -f kernel.its -k <path_to_key> -K atf_psci_public.dtb -r image.fit  
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

Note: There exist an alignment problem in U-Boot’s mkimage, we have debugged the source code and found that sometimes you may encounter this problem. To avoid this problem, modify the address that FIT image be loaded.
I.e, load ```image.fit``` to ```0xa2000004``` and it should be OK, however, when I load it to ```0xB0000000``` there will be a abortion exception. This problem is mainly related to mkimage tools and FIT format image parsing routines.

After loaded images are verified, Use bootm command to boot kernel as :  
```
    $ bootm 0xB0000004
```
PS: There are some problems in he latest version of U-Boot (v2014.10). You may need to rollback ```gic_v64.S``` file as the older version when you open ```CONFIG_GICV2``` macro. Although we have reported the problem to U-Boot's maintainer, we still recommend you to use the older version of U-Boot if you want to verify this using Foundation Model. As an alternative you could substitute the file ```gic_v64.S``` with the older version and add the ```CONFIG_GICV2``` macro in ```vexpress_aemv8a.h```.
