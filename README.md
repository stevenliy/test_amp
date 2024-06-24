# Test AMP

Download this project
```
git clone https://github.com/yli147/test_amp.git -b k1-amp test_amp
cd test_amp
export WORKDIR=`pwd`
```

Compile OpenSBI
```
cd $WORKDIR
git clone https://github.com/yli147/opensbi.git -b k1-amp opensbi
pushd opensbi
CROSS_COMPILE=/opt/riscv/bin/riscv64-unknown-linux-gnu- make -j$(nproc) PLATFORM_DEFCONFIG=k1_defconfig PLATFORM=generic
export OPENSBI=`realpath build/platform/generic/firmware/fw_dynamic.bin`
popd
```

Compile Test Applications
```
git clone https://github.com/yli147/test_context_switch.git -b k1-amp
pushd test_context_switch
make
popd
```

Compile U-boot (Increase the SPL image payload size is only optional for Linux OS as paylaod)
```
cd $WORKDIR
git clone https://github.com/yli147/u-boot.git -b k1-amp u-boot
pushd u-boot
make k1_defconfig
CROSS_COMPILE=/opt/riscv/bin/riscv64-unknown-linux-gnu- make menuconfig
(0x3100000) Maximum size of the SPL image, excluding BSS                                                    x x
(0x3000000) Size of the SPL malloc pool 
CROSS_COMPILE=/opt/riscv/bin/riscv64-unknown-linux-gnu- make -j8
popd
```

# Option1: Test AMP (Two baremetal s-mode applications)
Compile U-boot (The ns-hello.bin must be 64 bits aligned, otherwise the device tree blob won't be in 64 bits address which will cause opensbi boot failue)
```
pushd u-boot
cp ../opensbi/build/platform/generic/firmware/fw_dynamic.itb .
cp ../test_context_switch/build/s-hello/s-hello.bin .
cp ../test_context_switch/build/ns-hello/ns-hello.bin .
truncate -s %64 s-hello.bin
truncate -s %64 ns-hello.bin
./tools/mkimage -f u-boot-new.its -A riscv -O u-boot -T firmware u-boot-new.itb
popd
```

# Option2: Test AMP (Uboot +  one baremetal s-mode application)
```
pushd u-boot
CROSS_COMPILE=/opt/riscv/bin/riscv64-unknown-linux-gnu- make -j8
cp ../opensbi/build/platform/generic/firmware/fw_dynamic.itb .
cp ../test_context_switch/build/s-hello/s-hello.bin .
truncate -s %64 s-hello.bin
truncate -s %64 u-boot-nodtb.bin
./tools/mkimage -f u-boot-new1.its -A riscv -O u-boot -T firmware u-boot-new.itb
popd
```

# Option3: Test AMP (Uboot +  Linux)
```
git clone https://gitee.com/bianbu-linux/linux-6.1 -b bl-v1.0.y
pushd linux-6.1
make ARCH=riscv CROSS_COMPILE="/opt/spacemit-toolchain-linux-glibc-x86_64-v1.0.1/bin/riscv64-unknown-linux-gnu-" k1_defconfig
make ARCH=riscv CROSS_COMPILE="/opt/spacemit-toolchain-linux-glibc-x86_64-v1.0.1/bin/riscv64-unknown-linux-gnu-" -j16
popd
pushd u-boot
CROSS_COMPILE=/opt/riscv/bin/riscv64-unknown-linux-gnu- make -j8
cp ../opensbi/build/platform/generic/firmware/fw_dynamic.itb .
cp ../linux-6.1/arch/riscv/boot/Image .
dtc -I dts -O dtb -o k1-x_rt-deb1.dtb ../k1-x_rt-deb1.dts
truncate -s %64 Image
truncate -s %64 u-boot-nodtb.bin
./tools/mkimage -f u-boot-new2.its -A riscv -O u-boot -T firmware u-boot-new.itb
popd
```

# Flash Image
Flash Image (Make sure you have biandu_sdcard.img already flashed to the SDCard, then upgrade the SDCard with following steps )
```
dd if=u-boot/fw_dynamic.itb of=/dev/sdX3
dd if=u-boot/u-boot-new.itb of=/dev/sdX4
dd if=u-boot/FSBL.bin of=/dev/sdX1
```
Or
```
dd if=/dev/zero of=disk.img bs=1M count=40
sudo sgdisk -g --clear --set-alignment=1 \
       --new=1:256:+256K:    --change-name=1:'fsbl' --attributes=3:set:2 \
       --new=2:768:+64K:  --change-name=2:'env' --attributes=3:set:2 \
       --new=3:896:+384K:   --change-name=3:'opensbi' --attributes=3:set:2  \
        --new=4:1664:+36M:   --change-name=4:'uboot'  --attributes=3:set:2  \
       disk.img
loopdevice=`sudo losetup --partscan --find --show ./disk.img`
sudo dd if=u-boot/u-boot-new.itb of=${loopdevice}p4
sudo dd if=u-boot/fw_dynamic.itb of=${loopdevice}p3
sudo dd if=u-boot/FSBL.bin of=${loopdevice}p1
sudo dd if=env.bin of=${loopdevice}p2
sudo dd if=bootinfo_sd.bin of=${loopdevice}
sudo losetup -D ${loopdevice}
sudo dd if=disk.img of=/dev/sdX bs=8M status=progress
```

# Notes:
In the latest code u-boot device tree, I have added uart9 (GPIO72/GPIO73) as the default stdout uart, 
so you need connect GPIO72/73 with a UART terminal to see the opensbi log.
The u-boot-spl code also use the device tree, and keeps to use uart0 as the output

![image](https://github.com/yli147/test_amp/assets/21300636/8fcba632-7797-451c-860e-3fc0e64a3b99)


![image](https://github.com/yli147/test_amp/assets/21300636/b4f480e9-853b-43a6-9eaf-486ec4c2b945)
