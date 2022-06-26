# 跨平台编译qemu, 以riscv64为例

## 方案一:使用qemu-system-riscv64模拟完整的riscv环境
- 从[https://cdimage.ubuntu.com/releases/22.04/release/](https://cdimage.ubuntu.com/releases/22.04/release/ "https://cdimage.ubuntu.com/releases/22.04/release/")下载riscv的image,然后使用qemu启动,之后正常编译


## 方案二:debootstrap+chroot
- 采用debootstrap构建riscv的rootfs, 然后使用chroot进入riscv虚拟环境进行编译

1. 安装qemu, 使得chroot后可以运行riscv的程序
```bash
sudo apt install qemu-user-static
```
2. 构建rootfs并编译qemu
```bash
export ROOTFS=<rootfs_path>
sudo debootstrap --arch=riscv64 --foreign jammy $ROOTFS https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports
sudo chroot $ROOTFS
/debootstrap/debootstrap --second-stage
#此时apt的source.list只包含main下的包,需要修改/etc/apt/souce.list, 请参考https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu-ports/
apt update
apt install ibglib2.0-dev pkg-config make gcc ninja-build meson wget
wget https://download.qemu.org/qemu-7.0.0.tar.xz
```
3. 编译
```bash
../configure --target-list=x86_64-linux-user,i386-linux-user --disable-werror
```
- rootfs可以采用跨平台docker来代替

## 方案三:riscv64-linux-gnu toochains + rootfs
- x86上提供了riscv64-linux-gnu工具链, 但是qemu依赖的glib和其它的库没有简单的riscv64的跨平台工具链,所以采用riscv64-linux-gnu-gcc来编译,但是链接到上一步rootfs里面的依赖库

1. 安装riscv64-linux-gnu toochains和pkg-config和相关工具
```bash
sudo apt install gcc-riscv64-linux-gnu pkg-config make ninja-build meson
```
2. 由于meson检测host的问题,需要在meson.build中做如下修改
```
diff --git a/meson.build b/meson.build
-cpu = host_machine.cpu_family()
+cpu = 'riscv'
```
3. 设置跨平台编译的环境变量
```bash
export PKG_CONFIG_LIBDIR=$ROOTFS/usr/lib/riscv64-linux-gnu/pkgconfig/
export PKG_CONFIG_SYSROOT_DIR=$ROOTFS
export LD_LIBRARY_PATH=$ROOTFS/usr/lib/riscv64-linux-gnu/
```
4. 编译
```bash
../configure --target-list=x86_64-linux-user,i386-linux-user --disable-werror --cc=riscv64-linux-gnu-gcc --cpu=riscv --static
```
## 总结
- 严格来说,方案一和二不算跨平台编译,只是借助qemu-system或qemu-user搭建了riscv系统的虚拟环境
- 以上方案, 构建环境复杂度递增, 编译时间,递减. 不过熟悉之后还是方案三使用起来最为方便.

## TODO
- 方案三的情况下只安装必需的依赖包,精简rootfs, 减少对debootstrap的依赖
- 方案三静态编译较为容易,动态编译仍需要修改参数
- ?跨平台编译qemu所需要的依赖库(glib等), 试了一下有点复杂
- meson.build对cpu的修改不够优雅,应该使其能够自动识别
- [nix](https://nixos.org/ "nix") cross build static executable
