VERSION --arg-scope-and-set --try 0.7

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

    SAVE ARTIFACT ./idbloader.img AS LOCAL ./idbloader.img
    SAVE ARTIFACT ./u-boot.itb AS LOCAL ./u-boot.itb

create-image:
    FROM --allow-privileged ubuntu:22.04    # Privliged required to setup loopback device
    WORKDIR /build

    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt update && apt install --no-install-recommends -y parted udev

    ARG SECTOR_SIZE=512
    LET MIN_SECT_COUNT=$(echo $(( 1 + 63 + 7104 + 512 + 384 + 128 + 64 + 8192 + 8192 + 8192 + 229376 + 33 )))   # Calculated from https://opensource.rock-chips.com/wiki_Partitions
    ARG IMAGE_SIZE_BYTES=$(echo $(( $SECTOR_SIZE * $MIN_SECT_COUNT )))
    # This needs to be all one step as cached values may not be valid when it is reran
    # This could theoretically be written to not use a loopback device, but copying files
    # would be a huge pain
    LET IMAGE_FILE_PATH="./firmware.img"
    RUN --privileged truncate --size=$IMAGE_SIZE_BYTES $IMAGE_FILE_PATH && \    # Create an empty image file
        # Existing loop devices don't work well in a container in Earthly for some reason, so create a new one
        LOOPBACK_DEV_ID=$(( $(cat /sys/module/loop/parameters/max_loop) + 50)) && \
        LOOPBACK_DEV="/dev/loop$LOOPBACK_DEV_ID" && \
        mknod "$LOOPBACK_DEV" b 7 "$LOOPBACK_DEV_ID" && \
        losetup --sizelimit=$IMAGE_SIZE_BYTES --sector-size=$SECTOR_SIZE "$LOOPBACK_DEV" "$IMAGE_FILE_PATH" && \ # Mount the file
        ( \ # If successfully mounted
            ( \
                dd if=/dev/zero of=$LOOPBACK_DEV count=4096 bs=$SECTOR_SIZE && \ # Blank out the first 4096 sectors. For some reason parted thinks that there is a GPT table here already although it's all 0s
                # If this fails, see https://unix.stackexchange.com/a/87189. If on WSL2, update the command line using these instructions: https://learn.microsoft.com/en-us/windows/wsl/wsl-config#wslconfig
                parted --script "$LOOPBACK_DEV" \
                    mklabel gpt \                   # Create the GUID partition table
                    unit s \                        # Set the unit for the remainder of the command to "sectors"
                    mkpart primary 64 2111 \        # Create the SPL/TPL partition that idbloader.img will be written to
                    mkpart primary 16384 20479 \    # Create the U-Boot partition that u-boot.itb will be written to
                    mkpart primary 32768 163839 \   # Create the boot configuration partition that the device tree and boot script will be written to
                    print \                         # Print the created table
            ); \
            ( \
                # Always cleanup loopback dev
                losetup -d "$LOOPBACK_DEV" ; \
                rm "$LOOPBACK_DEV" \
            ) \
        )

test-loopback-dev:
    FROM --allow-privileged ubuntu:22.04    # Privliged required to setup loopback device
    WORKDIR /build
    RUN --privileged truncate --size=64M firmware.img && \
        mknod /dev/loop123 b 7 0 && \
        losetup --sizelimit=64MiB --sector-size=512 /dev/loop123 firmware.img && \
        echo "Sucessfully attached firmware.img to $LOOPBACK_DEV"
