VERSION --arg-scope-and-set --try 0.7

build-spl-loader:
    FROM alpine/git:2.43.0
    WORKDIR /src
    RUN git clone --depth 1 --single-branch --branch master https://github.com/rockchip-linux/rkbin.git .
    RUN ./tools/boot_merger ./RKBOOT/RK3588MINIALL.ini
    LET OUTPUT_FILE=$(sed -n 's/PATH=\(.*\)/\1/p' ./RKBOOT/RK3588MINIALL.ini)
    SAVE ARTIFACT $OUTPUT_FILE AS LOCAL ./rk3588_spl_loader.bin

build-boot-config:
    FROM ubuntu:22.04
    WORKDIR /build

build-u-boot:
    ARG SOURCE_DATE_EPOCH

    FROM ubuntu:22.04
    WORKDIR /src
    
    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt update && apt install --no-install-recommends -y \
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

    LET NUM_MAKE_PROC=$(nproc --ignore=2)
    COPY . .

    RUN make -j$NUM_MAKE_PROC clean
    RUN make -j$NUM_MAKE_PROC blade3-rk3588_defconfig 

    IF [ -n "$SOURCE_DATE_EPOCH" ]
        BUILD_ARGS="SOURCE_DATE_EPOCH=$SOURCE_DATE_EPOCH"
    END
    RUN make -j$NUM_MAKE_PROC $BUILD_ARGS

    RUN --interactive bash

    SAVE ARTIFACT ./idbloader.img AS LOCAL ./idbloader.img
    SAVE ARTIFACT ./u-boot.itb AS LOCAL ./u-boot.itb

create-image:
    FROM --allow-privileged ubuntu:22.04    # Privliged required to setup loopback device
    WORKDIR /build

    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt update && apt install --no-install-recommends -y parted fdisk udev

    ARG SECTOR_SIZE=512
    LET MIN_SECT_COUNT=$(echo $(( 1 + 63 + 7104 + 512 + 384 + 128 + 64 + 8192 + 8192 + 8192 + 229376 + 33 )))   # Calculated from https://opensource.rock-chips.com/wiki_Partitions
    ARG IMAGE_SIZE_BYTES=$(echo $(( $SECTOR_SIZE * $MIN_SECT_COUNT )))
    # This needs to be all one step as cached values may not be valid when it is reran
    # This could theoretically be written to not use a loopback device, but copying files
    # would be a huge pain
    LET IMAGE_FILE_PATH="./firmware.img"
    RUN truncate --size=$IMAGE_SIZE_BYTES $IMAGE_FILE_PATH  # Create the 0'd out image file
    RUN parted --script "$IMAGE_FILE_PATH" \
            mklabel gpt \                       # Create the GUID partition table
            unit s \                            # Set the unit for the remainder of the command to "sectors"
            mkpart primary 64 2111 \            # Create the SPL/TPL partition that idbloader.img will be written to
            mkpart primary 16384 20479 \        # Create the U-Boot partition that u-boot.itb will be written to
            mkpart primary fat16 32768 163839 \ # Create the boot configuration partition that the device tree and boot script will be written to
            name 1 "idbloader" \                # Name the partitions
            name 2 "u-boot" \
            name 3 "boot-config" \
            set 3 boot on                       # Set the boot config partition GUID to "boot"
    RUN sfdisk --part-type "$IMAGE_FILE_PATH" 3 "Linux extended boot"  # Label the boot-config partition as a /boot partition
    RUN sfdisk --dump "$IMAGE_FILE_PATH"                               # Print the partition table
    RUN --privileged LOOPBACK_DEV=$(losetup --sizelimit=$IMAGE_SIZE_BYTES --sector-size=$SECTOR_SIZE --partscan --show -f "$IMAGE_FILE_PATH") && \ # Mount the file
        ( \ # If successfully mounted
            ( \
                ls -l /dev \
                ls -l "$LOOPBACK_DEV"p1 \
            ); \
            ( \
                EXITCODE=$? && \
                # Always cleanup loopback dev
                losetup -d "$LOOPBACK_DEV" ; \
                exit $EXITCODE \
            ) \
        )
    SAVE ARTIFACT $IMAGE_FILE_PATH AS LOCAL ./firmware.img

test-loopback-dev:
    FROM --allow-privileged ubuntu:22.04    # Privliged required to setup loopback device
    WORKDIR /build
    RUN --privileged truncate --size=64M firmware.img && \
        losetup --sizelimit=64MiB --sector-size=512 --show -f firmware.img && \
        echo "Sucessfully attached firmware.img to $LOOPBACK_DEV"
