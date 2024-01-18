VERSION --arg-scope-and-set --try --use-function-keyword 0.7

ENSURE_CONFIG_EXISTS:
    FUNCTION
    IF [ ! -f ".config" ]
        RUN make blade3-rk3588_defconfig 
    END

APT_INSTALL:
    FUNCTION
    ARG --required PACKAGES
    RUN apt update && DEBIAN_FRONTEND=noninteractive apt install --no-install-recommends -y $PACKAGES

menuconfig:
    FROM ubuntu:22.04
    WORKDIR /src

    DO +APT_INSTALL --PACKAGES="make bison flex libncurses-dev gcc"

    COPY . .
    DO +ENSURE_CONFIG_EXISTS
    RUN --interactive make menuconfig
    SAVE ARTIFACT ./.config AS LOCAL ./.config

build-spl-loader:
    FROM alpine/git:2.43.0
    WORKDIR /src
    RUN git clone --depth 1 --single-branch --branch master https://github.com/rockchip-linux/rkbin.git .
    RUN ./tools/boot_merger ./RKBOOT/RK3588MINIALL.ini
    LET OUTPUT_FILE=$(sed -n 's/PATH=\(.*\)/\1/p' ./RKBOOT/RK3588MINIALL.ini)
    SAVE ARTIFACT $OUTPUT_FILE AS LOCAL ./rk3588_spl_loader.bin

build-u-boot:
    ARG SOURCE_DATE_EPOCH

    FROM ubuntu:22.04
    WORKDIR /src
    
    RUN apt update && DEBIAN_FRONTEND=noninteractive apt install --no-install-recommends -y \
        gcc-aarch64-linux-gnu python-is-python3 \
        bc bison build-essential coccinelle \
        device-tree-compiler dfu-util efitools flex gdisk graphviz imagemagick \
        liblz4-tool libgnutls28-dev libguestfs-tools libncurses-dev \
        libpython3-dev libsdl2-dev libssl-dev lz4 lzma lzma-alone openssl \
        pkg-config python3 python3-asteval python3-coverage python3-filelock \
        python3-pkg-resources python3-pycryptodome python3-pyelftools \
        python3-pytest python3-pytest-xdist python3-sphinxcontrib.apidoc \
        python3-sphinx-rtd-theme python3-subunit python3-testtools \
        python3-virtualenv swig uuid-dev
    RUN wget https://github.com/rockchip-linux/rkbin/raw/master/bin/rk35/rk3588_ddr_lp4_2112MHz_lp5_2736MHz_v1.12.bin && \
        wget https://github.com/rockchip-linux/rkbin/raw/master/bin/rk35/rk3588_bl31_v1.40.elf
    ENV CROSS_COMPILE=aarch64-linux-gnu-
    ENV ROCKCHIP_TPL=./rk3588_ddr_lp4_2112MHz_lp5_2736MHz_v1.12.bin
    ENV BL31=./rk3588_bl31_v1.40.elf

    LET NUM_MAKE_PROC=$(nproc)
    COPY . .

    RUN make -j$NUM_MAKE_PROC clean
    DO +ENSURE_CONFIG_EXISTS

    IF [ -n "$SOURCE_DATE_EPOCH" ]
        BUILD_ARGS="SOURCE_DATE_EPOCH=$SOURCE_DATE_EPOCH"
    END
    RUN make -j$NUM_MAKE_PROC $BUILD_ARGS

    SAVE ARTIFACT ./idbloader.img AS LOCAL ./idbloader.img
    SAVE ARTIFACT ./u-boot.itb AS LOCAL ./u-boot.itb
    SAVE ARTIFACT ./u-boot.dtb AS LOCAL ./u-boot.dtb

build-boot-config:
    ARG --required PARTITION_SIZE_BYTES

    FROM ubuntu:22.04
    WORKDIR /build

    DO +APT_INSTALL --PACKAGES="dosfstools mtools"

    COPY +build-u-boot/u-boot.dtb ./u-boot.dtb

    LET IMAGE_FILE_PATH="./boot-config.img"
    RUN truncate --size=$PARTITION_SIZE_BYTES $IMAGE_FILE_PATH      # Create the 0'd out image file
    RUN mkfs.vfat -F32 -n "boot-config" --mbr=no $IMAGE_FILE_PATH   # Create a FAT32 partition
    RUN mcopy -i $IMAGE_FILE_PATH ./u-boot.dtb ::                   # Copy the device tree to the image file
    RUN mdir -i $IMAGE_FILE_PATH                                    # List the files copied to the image

    SAVE ARTIFACT $IMAGE_FILE_PATH AS LOCAL ./boot-config.img

build-firmware-image:
    FROM --allow-privileged ubuntu:22.04    # Privliged required to setup loopback device
    WORKDIR /build

    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt update && apt install --no-install-recommends -y parted fdisk udev

    ARG SECTOR_SIZE=512
    LET MIN_SECT_COUNT=$(echo $(( 1 + 63 + 7104 + 512 + 384 + 128 + 64 + 8192 + 8192 + 8192 + 229376 + 33 )))   # Calculated from https://opensource.rock-chips.com/wiki_Partitions
    ARG IMAGE_SIZE_BYTES=$(echo $(( $SECTOR_SIZE * $MIN_SECT_COUNT )))

    LET IDBLOADER_FIRST_SECTOR=64       # Record where the partitions start so that they can be written to directly later
    LET U_BOOT_FIRST_SECTOR=16384
    LET BOOT_CONFIG_FIRST_SECTOR=32768
    LET BOOT_CONFIG_END_SECTOR=163839
    LET BOOT_CONFIG_PARTITION_SIZE_BYTES=$(echo $(( ($BOOT_CONFIG_END_SECTOR - $BOOT_CONFIG_FIRST_SECTOR + 1) * $SECTOR_SIZE )))

    # Create the image file and setup a partition table
    LET IMAGE_FILE_PATH="./firmware.img"
    RUN truncate --size=$IMAGE_SIZE_BYTES $IMAGE_FILE_PATH                              # Create the 0'd out image file
    RUN parted --script "$IMAGE_FILE_PATH" \
            mklabel gpt \                                                               # Create the GUID partition table
            unit s \                                                                    # Set the unit for the remainder of the command to "sectors"
            mkpart primary $IDBLOADER_FIRST_SECTOR 2111 \                               # Create the SPL/TPL partition that idbloader.img will be written to
            mkpart primary $U_BOOT_FIRST_SECTOR 20479 \                                 # Create the U-Boot partition that u-boot.itb will be written to
            mkpart primary fat32 $BOOT_CONFIG_FIRST_SECTOR $BOOT_CONFIG_END_SECTOR \    # Create the boot configuration partition that the device tree and boot script will be written to
            name 1 "idbloader" \                                                        # Name the partitions
            name 2 "u-boot" \
            name 3 "boot-config" \
            set 3 boot on                                                               # Set the boot config partition GUID to "boot"
    RUN sfdisk --part-type "$IMAGE_FILE_PATH" 3 "Linux extended boot"                   # Label the boot-config partition as a /boot partition
    RUN sfdisk --dump "$IMAGE_FILE_PATH"                                                # Print the partition table

    # Build the partition image files
    COPY +build-u-boot/idbloader.img ./idbloader.img
    COPY +build-u-boot/u-boot.itb ./u-boot.itb
    COPY --build-arg=PARTITION_SIZE_BYTES=$BOOT_CONFIG_PARTITION_SIZE_BYTES +build-boot-config/boot-config.img ./boot-config.img

    # Write the partition image file
    RUN dd of=$IMAGE_FILE_PATH bs=$SECTOR_SIZE if=./idbloader.img seek=$IDBLOADER_FIRST_SECTOR
    RUN dd of=$IMAGE_FILE_PATH bs=$SECTOR_SIZE if=./u-boot.itb seek=$U_BOOT_FIRST_SECTOR
    RUN dd of=$IMAGE_FILE_PATH bs=$SECTOR_SIZE if=./boot-config.img seek=$BOOT_CONFIG_FIRST_SECTOR

    # Force any async writes to complete
    RUN sync --file-system $IMAGE_FILE_PATH

    SAVE ARTIFACT $IMAGE_FILE_PATH AS LOCAL ./firmware.img
