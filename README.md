# truenas scale 23.10 编译intel-gpu-i915-backports驱动 for dg1 驱动
## 安装依赖
>买的蓝戟的80eu的半高刀卡。
```
#开启开发者模式
install-dev-tools
# 安装依赖
sudo apt install dkms make debhelper devscripts build-essential flex bison mawk dh-dkms
```
## 拉取github代码
```
# 进入到你的zfs存储为止，存储源码
cd /mnt/app/scjtqs
# clone源码
git clone https://github.com/intel-gpu/intel-gpu-i915-backports -b backport/main
```
## 编译源码生成deb包并安装
```
# 进入源码目录
cd intel-gpu-i915-backports
# 开始编译
make i915dkmsdeb-pkg
# 编译完成后，找到deb包。deb包在上层目录里面
cd ../
ls
# 例如 intel-i915-dkms_1.23.9.11.231003.15+i1-1_all.deb
cp intel-i915-dkms_1.23.9.11.231003.15+i1-1_all.deb /tmp
cd /tmp
apt install ./intel-i915-dkms_1.23.9.11.231003.15+i1-1_all.deb
# 若无法安装，因为文件系统只读，用mount -o remount,rw /改正可读写。
```
## dg1下载新的固件
```
# 查看是否有报错
sudo dmesg |grep i915
# 如果遇到  *ERROR* GT0: GuC firmware i915/dg1_guc_70.9.1.bin: fetch failed -ENOENT 根据具体报错的版本，选择下载对应的文件
wget -P /lib/firmware/i915 https://github.com/intel-gpu/intel-gpu-firmware/raw/main/firmware/dg1_guc_70.9.1.bin
apt-get install  intel-media-va-driver-non-free
```
## 编译 intel-media-driver
```
# 安装 libva
sudo apt-get install git cmake pkg-config meson libdrm-dev automake libtool
git clone https://github.com/intel/libva.git
cd libva
./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
make -j`(nproc)`
sudo make install
# 安装 gmmlib (libigdgmm-dev)
git clone https://github.com/intel/gmmlib.git
cd gmmlib
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr/ -DCMAKE_INSTALL_LIBDIR=/usr/lib/x86_64-linux-gnu -DCMAKE_BUILD_TYPE=ReleaseInternal .. 
make -j`(nproc)`
sudo make install
# 安装 intel-media-driver
apt install autoconf libtool libdrm-dev xorg-dev  libx11-dev libgl1-mesa-glx libva-dev
git clone https://github.com/intel/media-driver
mkdir build_media
cd build_media
export ENABLE_PRODUCTION_KMD=ON
cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR=/usr/lib/x86_64-linux-gnu ../media-driver
make -j`$(nproc)` -e ENABLE_PRODUCTION_KMD=ON
sudo make install
```
## truenas scale 24.04 beta1出来了，不用这么麻烦了
```
# 开启dg1支持
vim /etc/modprobe.d/dkms.conf
# 添加两行行 
options i915 force_probe=4908
options i915 enable_guc=3

# 让后重启
reboot
# 重启后，显卡基本上可以工作了,显示器上有输出了。接下来安装vapp解码器
 apt install intel-media-va-driver-non-free  vainfo
 # 安装完成后，输入 vainfo查看解码能力
补充：24.04 beta1只能点亮强行驱动。但是编解码会报错。intel的backports驱动还不支持6.6内核下的编译，会报错。挺蛋疼的
```
