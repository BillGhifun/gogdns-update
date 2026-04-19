OpenWrt ipk
请使用脚本来打包新版二进制程序或使用提供的打包好的文件(可能更新不及时)

```sh
#!/bin/bash

# --- 基础配置 ---
PKG_NAME="gogdns"
PKG_VERSION="0.6.88_20260317-141449_Beta"
BUILD_DIR="ipk_build"

# --- 1. 架构映射函数 ---
# 负责将 Go 的编译后缀 (amd64, arm 等) 或系统自带的架构 (x86_64, aarch64 等) 转换为 OpenWrt 标准命名
get_arch_info() {
    local input=$1
    case $input in
        amd64|x86_64)
            PKG_ARCH="x86_64"
            BINARY_SUFFIX="amd64"
            ;;
        386|i386|i686)
            PKG_ARCH="i386_pentium4"
            BINARY_SUFFIX="386"
            ;;
        arm64|aarch64)
            PKG_ARCH="aarch64_generic"
            BINARY_SUFFIX="arm64"
            ;;
        arm|armv7l|armv6l)
            PKG_ARCH="arm_cortex-a7_neon-vfpv4"
            BINARY_SUFFIX="arm"
            ;;
        *)
            PKG_ARCH=$input
            BINARY_SUFFIX=$input
            ;;
    esac
}

# --- 2. 核心打包函数 ---
build_ipk() {
    local target_arch=$1
    local bin_suffix=$2
    local target_binary="${PKG_NAME}-linux-${bin_suffix}"
    local output_name="${PKG_NAME}_${PKG_VERSION}_${target_arch}.ipk"

    echo ">>> 开始构建: $output_name"

    if [ ! -f "./$target_binary" ]; then
        echo "    [跳过] 未找到对应的二进制文件: $target_binary"
        return 1
    fi

    # 清理并创建目录
    rm -rf $BUILD_DIR
    mkdir -p $BUILD_DIR/control
    mkdir -p $BUILD_DIR/data/etc/gogdns
    mkdir -p $BUILD_DIR/data/etc/init.d

    # 生成 control
    cat <<EOF > $BUILD_DIR/control/control
Package: $PKG_NAME
Version: $PKG_VERSION
Architecture: $target_arch
Maintainer: BillGhifun
Depends: libc, ca-bundle
Section: net
Priority: optional
Description: GOGDNS Server for $target_arch
EOF

    # 生成 postinst
    cat <<EOF > $BUILD_DIR/control/postinst
#!/bin/sh
ln -sf /etc/gogdns/gogdns /usr/bin/gogdns
chmod +x /etc/gogdns/gogdns
chmod +x /etc/init.d/gogdns
mkdir -p /etc/gogdns/workstation
exit 0
EOF
    chmod 755 $BUILD_DIR/control/postinst

    # 放置二进制文件 (统一重命名为 gogdns)
    cp "./$target_binary" "$BUILD_DIR/data/etc/gogdns/gogdns"
    chmod +x "$BUILD_DIR/data/etc/gogdns/gogdns"

    # 生成启动脚本
    cat <<EOF > $BUILD_DIR/data/etc/init.d/gogdns
#!/bin/sh /etc/rc.common
START=99
USE_PROCD=1
start_service() {
    mkdir -p /etc/gogdns
    procd_open_instance
    procd_set_param command /etc/gogdns/gogdns
    procd_set_param chdir /etc/gogdns
    procd_set_param env GOGDNS_BASE_PATH=/etc/gogdns
    procd_set_param respawn
    procd_set_param stdout 1
    procd_set_param stderr 1
    procd_close_instance
}
stop_service() {
    killall gogdns
}
EOF
    chmod 755 $BUILD_DIR/data/etc/init.d/gogdns

    # 打包
    echo "2.0" > $BUILD_DIR/debian-binary
    (cd $BUILD_DIR/control && tar --numeric-owner -czf ../control.tar.gz .)
    (cd $BUILD_DIR/data && tar --numeric-owner -czf ../data.tar.gz .)
    (cd $BUILD_DIR && tar --numeric-owner -czf ../$output_name debian-binary control.tar.gz data.tar.gz)

    # 清理
    rm -rf $BUILD_DIR
    echo "    [成功] 输出文件: $output_name"
    echo "-----------------------------------"
}

# --- 3. 运行逻辑处理 ---
if [ $# -eq 0 ]; then
    # 无参数：自动检测当前系统架构
    RAW_ARCH=$(uname -m)
    echo "模式: 自动检测 ($RAW_ARCH)"
    get_arch_info "$RAW_ARCH"
    build_ipk "$PKG_ARCH" "$BINARY_SUFFIX"

elif [ "$1" == "all" ]; then
    # 参数为 all：打包所有支持的架构
    echo "模式: 构建所有架构"
    # 这里定义你想批量打包的 Go 编译后缀列表
    for suffix in amd64 386 arm64 arm; do
        get_arch_info "$suffix"
        build_ipk "$PKG_ARCH" "$BINARY_SUFFIX"
    done

else
    # 带有特定参数：如 ./build.sh arm64
    echo "模式: 指定架构 ($1)"
    get_arch_info "$1"
    build_ipk "$PKG_ARCH" "$BINARY_SUFFIX"
fi
```
