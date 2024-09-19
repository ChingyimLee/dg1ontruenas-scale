# truenas scale 24.04(linux-6.6) 编译intel-gpu-i915-backports驱动 for dg1 驱动
***免责声明：本过程基于```https://jose.scjtqs.com/truenas/2024-02-13-2011/truenas-scale-23-10-%E7%BC%96%E8%AF%91intel-gpu-i915-backports%E9%A9%B1%E5%8A%A8-for-dg1.html```改编，并参考了```https://harrychen.xyz/2024/04/01/accelerate-video-encoding-with-intel-dg1-on-linux-6.6/```,如有启发请到原作者处表达感谢***
>买的皮蛋熊大大推荐的的80eu的全高被动散热卡（据说兼容性最佳，实测ASRockRACK E3C246D4U2-2T with CC150打开Above4G，关闭CSM即可点亮）
## 一、前置准备
### 1.truenas scale 24.04 开启apt 开发者模式(Developer Mode)
truenas scale 从24.04 beta1开始，/usr/bin下的东西变成了只读的了。不再是 chmod +x /usr/bin/*来解开apt了

这里需要手动开启开发者模式：
```
install-dev-tools
```
用root用户，在shell下输入上面的命令开启

~~### 2.Encountered Read-only file system problem, unable to create anything~~
~~如果你执行什么命令，出现了系统只读问题。执行下面命令：~~
```
zfs get readonly
```
~~查看哪些路径是只读的。需要把 on改成off~~
```
zfs set readonly=off [dataset]
```
~~例如 zfs set readonly=off boot-pool/ROOT/24.04-BETA.1~~
~~
## 二、安装依赖
```
#开启开发者模式
install-dev-tools
# 安装依赖
apt install dkms make debhelper devscripts build-essential flex bison mawk dh-dkms
```
安装完以上依赖后，```/etc/modprobe.d```路径内应该创建了dkms.conf文件，需要添加以下两行，作用是强制使用驱动
```
# 开启dg1支持
vim /etc/modprobe.d/dkms.conf
# 添加两行
options i915 force_probe=4908
options i915 enable_guc=3
# 然后重启
reboot  #务必重启，否则无法加载模块
```
## 三、拉取github代码
```
# 进入到你的zfs存储为止，存储源码，如：
cd /mnt/Applications/DG1
# clone源码
git clone https://github.com/intel-gpu/intel-gpu-i915-backports -b backport/main
```
## 四、编译源码生成deb包并安装
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
## 五、dg1下载新的固件
```
# 查看是否有报错
sudo dmesg |grep i915
# 如果遇到  *ERROR* GT0: GuC firmware i915/dg1_guc_70.25.0.bin: fetch failed -ENOENT 根据具体报错的版本，选择下载对应的文件
wget -P /lib/firmware/i915 https://github.com/intel-gpu/intel-gpu-firmware/raw/main/firmware/dg1_guc_70.25.0.bin
apt-get install  intel-media-va-driver-non-free
```
## 六、编译 intel-media-driver，需注意libva gmmlib intel-media-driver版本是否适配：
先点开GitHub 仓库
```https://github.com/intel/media-driver``` Releases 查看 Intel 发布的源代码。这里选择最新的 “Intel Media Driver 2024Q1 Release - 24.2.5” 版本，在发行说明里 Intel 标注了 Gmmlib 和 Libva 需求的版本
例如，24.2.5版本```intel-media-driver```推荐使用的Gmmlib版本是```intel-gmmlib-22.3.20```，Libva版本是```2.22.0```
```
# 1.安装 libva
apt-get install git cmake pkg-config meson libdrm-dev automake libtool  #安装依赖
git clone --branch 2.14.0 https://github.com/intel/libva.git /root/libva  #克隆指定版本的libva
cd /root/libva
./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
make -j`(nproc)`
make install

# 2.安装 gmmlib (libigdgmm-dev)
git clone --branch intel-gmmlib-22.3.20 https://github.com/intel/gmmlib.git /root/gmmlib  #克隆指定版本的 gmmlib
cd /root/gmmlib
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr/ -DCMAKE_INSTALL_LIBDIR=/usr/lib/x86_64-linux-gnu -DCMAKE_BUILD_TYPE=ReleaseInternal .. 
make -j`(nproc)`
make install

# 3.安装 intel-media-driver
apt install autoconf libtool libdrm-dev xorg-dev  libx11-dev libgl1-mesa-glx libva-dev
git clone --branch intel-media-24.2.5 https://github.com/intel/media-driver.git /root/media-driver  #克隆指定版本的 intel-media-driver
cd /root/media-driver
mkdir build_media && cd build_media
export ENABLE_PRODUCTION_KMD=ON
cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR=/usr/lib/x86_64-linux-gnu ..
make -j`$(nproc)` -e ENABLE_PRODUCTION_KMD=ON  #本条负载很大，若发生内存占满，请自行缩小线程数，如单线程```make -j1 -e ENABLE_PRODUCTION_KMD=ON```
make install
```
# 检查驱动是否正确安装
```
ls /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
```
如果该文件存在，说明 intel-media-driver 已正确安装
## 七、安装完成后，输入 vainfo查看解码能力
```
apt install vainfo
```
安装完成后，输入 vainfo查看解码能力
## 八、测试gpu负载
如果需要检测 GPU 的利用率，可以（在 Docker 外即可）安装 intel-gpu-tools 软件包。它提供了 intel_gpu_top 命令，可以实时查看 GPU 的各个组件的利用率。不过在我的测试中，无论有多少负载，GPU 的 Video 组件利用率都显示得很高，似乎没有太多参考的价值。
