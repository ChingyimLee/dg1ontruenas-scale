# truenas scale 24.04(linux-6.6) 驱动dg1并实现jellyfin转码
***声明：本过程参考了[```scjtqs大神的博文```](https://jose.scjtqs.com/truenas/2024-02-13-2011/truenas-scale-23-10-%E7%BC%96%E8%AF%91intel-gpu-i915-backports%E9%A9%B1%E5%8A%A8-for-dg1.html)，[```Harry Chen大神的博文```](https://harrychen.xyz/2024/04/01/accelerate-video-encoding-with-intel-dg1-on-linux-6.6/),[```Icarus_Radio大神的博文```](https://icarusradio.github.io/guides/ubuntu-dg1-jellyfin.html)（排名不分先后），我仅仅是做了一点微小的整合工作，目前在我的设备已成功点亮并解码成功，如有启发请到原各位作者处表达感谢***
>买的皮蛋熊大大推荐的的80eu的全高被动散热卡（据说兼容性最佳，实测ASRockRACK E3C246D4U2-2T with CC150打开Above4G，关闭CSM即可点亮）
## 1.前置准备
### 1.1. truenas scale 24.04 开启apt 开发者模式(Developer Mode)
truenas scale 从24.04 beta1开始，/usr/bin下的东西变成了只读的了。不再是 chmod +x /usr/bin/*来解开apt了

这里需要手动开启开发者模式：
```
install-dev-tools
```
用root用户，在shell下输入上面的命令开启

### 1.2.Encountered Read-only file system problem
~~如果你执行什么命令，出现了系统只读问题。执行下面命令：~~
```
zfs get readonly
```
~~查看哪些路径是只读的。需要把 on改成off~~
```
zfs set readonly=off [dataset]
```
~~例如 zfs set readonly=off boot-pool/ROOT/24.04-BETA.1~~
使用```mount -o remount,rw /```改读写权限即可。
## 2.安装依赖
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
## 3.拉取github代码
```
# 进入到你的zfs存储为止，存储源码，如：
cd /mnt/Applications/DG1
# clone源码
git clone https://github.com/intel-gpu/intel-gpu-i915-backports -b backport/main
```
## 4.编译源码生成deb包并安装
### 4.1.进入源码目录
```
cd intel-gpu-i915-backports
```
### 4.2开始编译
```
make i915dkmsdeb-pkg
```
### 4.3编译完成后，找到deb包。deb包在上层目录里面
```
cd ../
ls
# 例如 intel-i915-dkms_1.23.9.11.231003.15+i1-1_all.deb
cp intel-i915-dkms_1.23.9.11.231003.15+i1-1_all.deb /tmp
cd /tmp
apt install ./intel-i915-dkms_1.23.9.11.231003.15+i1-1_all.deb  # 若无法安装，因为文件系统只读，用mount -o remount,rw /改为可读写。此步安装后，需要重启以重新加载。
```
>***p.s. 以上3、4步，也许可以藉以添加intel官方ubuntu仓库拉取安装（节选自[```Icarus_Radio《Ubuntu 22.04 下修改驱动使 Intel DG1 可以在 Jellyfin 下解码》```](https://icarusradio.github.io/guides/ubuntu-dg1-jellyfin.html))***
>## (1)添加 Intel 官方 Ubuntu 仓库 
>1️⃣首先先添加 Intel 的 GPG 密钥：```wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | sudo gpg --dearmor --output /usr/share/keyrings/intel-graphics.gpg``` \
>2️⃣添加 LTS或Rolling仓库： \
添加 LTS 仓库: \
```echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu jammy/lts/2350 unified" | sudo tee /etc/apt/sources.list.d/intel-gpu-jammy.list``` \
添加 Rolling 仓库: \
```echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu jammy client" | sudo tee /etc/apt/sources.list.d/intel-gpu-jammy.list``` \
>3️⃣更新仓库： \
```apt update```
>## (2)安装 DKMS
>Icarus_Radio大神曾在chiphell回帖提到，“估计要降。虽然教程里说 DKMS 和内核版本不相关，但是我试过好几个版本不用他指定的内核都会报错。”，但实际上经我在6.6内核的truenas 24.4.2.2版本测试，安装编译对内核已经没有要求 \
```sudo apt install intel-i915-dkms intel-fw-gpu```
>安装完成后关闭机器。物理机装上 DG1，如果使用核显安装系统，请 BIOS 内选择仅独立显卡工作来关闭核显；如果使用亮机卡安装系统，请拆下亮机卡。请确保不要把 DG1 连上显示器，因为之前某个版本删除了图形输出能力，只支持硬件解码，所以连上也没有用。虚拟机上直通 DG1，并将显示关闭，理由同物理机，PVE 自带的控制台也是看不了显示输出的。
## 5.dg1下载新的固件
```
# 查看是否有报错
sudo dmesg |grep i915
# 如果遇到  *ERROR* GT0: GuC firmware i915/dg1_guc_70.25.0.bin: fetch failed -ENOENT 根据具体报错的版本，选择下载对应的文件
wget -P /lib/firmware/i915 https://github.com/intel-gpu/intel-gpu-firmware/raw/main/firmware/dg1_guc_70.25.0.bin
apt-get install  intel-media-va-driver-non-free
```
## 6.编译 intel-media-driver，需注意libva gmmlib intel-media-driver版本是否适配
先点开GitHub 仓库```https://github.com/intel/media-driver``` Releases 查看 Intel 发布的源代码。例如这里选择最新的 “Intel Media Driver 2024Q1 Release - 24.2.5” 版本，在发行说明里 Intel 标注了 Gmmlib 和 Libva 需求的版本，其中Gmmlib版本是```intel-gmmlib-22.3.20```，Libva版本是```2.22.0```
```
# 6.1.安装 libva
apt-get install git cmake pkg-config meson libdrm-dev automake libtool  #安装依赖
git clone --branch 2.22.0 https://github.com/intel/libva.git /root/libva  #克隆指定版本的libva
cd /root/libva
./autogen.sh --prefix=/usr --libdir=/usr/lib/x86_64-linux-gnu
make -j`(nproc)`
make install

# 6.2.构建 libva-utils
git clone https://github.com/intel/libva-utils.git
cd libva-utils
./autogen.sh # or ./autogen.sh --enable-tests to run unit test
make
sudo make install

# 6.3.安装 gmmlib (libigdgmm-dev)
git clone --branch intel-gmmlib-22.3.20 https://github.com/intel/gmmlib.git /root/gmmlib  #克隆指定版本的 gmmlib
cd /root/gmmlib
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/usr/ -DCMAKE_INSTALL_LIBDIR=/usr/lib/x86_64-linux-gnu -DCMAKE_BUILD_TYPE=ReleaseInternal .. 
make -j`(nproc)`
make install

# 6.4.安装 intel-media-driver
apt install autoconf libtool libdrm-dev xorg-dev  libx11-dev libgl1-mesa-glx libva-dev
git clone --branch intel-media-24.2.5 https://github.com/intel/media-driver.git /root/media-driver  #克隆指定版本的 intel-media-driver
cd /root/media-driver
mkdir build_media && cd build_media
export ENABLE_PRODUCTION_KMD=ON
cmake -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_INSTALL_LIBDIR=/usr/lib/x86_64-linux-gnu ..
make -j`$(nproc)` -e ENABLE_PRODUCTION_KMD=ON  #本条负载很大，若发生内存占满，请自行缩小线程数，如单线程```make -j1 -e ENABLE_PRODUCTION_KMD=ON```
make install
```
## 7.检查驱动是否正确安装
```
ls /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
```
如果该文件存在，说明 intel-media-driver 已正确安装，但大概率是驱动不了的，我经过几天尝试，最终还是借了icarusradio大神编译的iHD_drv_video.so驱动，地址[```https://github.com/Icarusradio/intel-media-driver-dg1/releases```](https://github.com/Icarusradio/intel-media-driver-dg1/releases)，具体操作是，使用scp sfp等工具，进入/usr/lib/x86_64-linux-gnu/dri/目录直接替换，替换后，输入vainfo，如果出现类似输出证明替换成功：
```
Trying display: drm
libva info: VA-API version 1.22.0
libva info: Trying to open /usr/lib/x86_64-linux-gnu/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_22
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.22 (libva 2.22.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 24.2.5 ()
vainfo: Supported profile and entrypoints
      VAProfileNone                   : VAEntrypointVideoProc
      VAProfileNone                   : VAEntrypointStats
      VAProfileMPEG2Simple            : VAEntrypointVLD
      VAProfileMPEG2Simple            : VAEntrypointEncSlice
      VAProfileMPEG2Main              : VAEntrypointVLD
      VAProfileMPEG2Main              : VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointFEI
      VAProfileH264Main               : VAEntrypointEncSliceLP
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointEncSlice
      VAProfileH264High               : VAEntrypointFEI
      VAProfileH264High               : VAEntrypointEncSliceLP
      VAProfileVC1Simple              : VAEntrypointVLD
      VAProfileVC1Main                : VAEntrypointVLD
      VAProfileVC1Advanced            : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointEncPicture
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSlice
      VAProfileH264ConstrainedBaseline: VAEntrypointFEI
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSliceLP
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointEncSlice
      VAProfileHEVCMain               : VAEntrypointFEI
      VAProfileHEVCMain               : VAEntrypointEncSliceLP
      VAProfileHEVCMain10             : VAEntrypointVLD
      VAProfileHEVCMain10             : VAEntrypointEncSlice
      VAProfileHEVCMain10             : VAEntrypointEncSliceLP
      VAProfileVP9Profile0            : VAEntrypointVLD
      VAProfileVP9Profile0            : VAEntrypointEncSliceLP
      VAProfileVP9Profile1            : VAEntrypointVLD
      VAProfileVP9Profile1            : VAEntrypointEncSliceLP
      VAProfileVP9Profile2            : VAEntrypointVLD
      VAProfileVP9Profile2            : VAEntrypointEncSliceLP
      VAProfileVP9Profile3            : VAEntrypointVLD
      VAProfileVP9Profile3            : VAEntrypointEncSliceLP
      VAProfileHEVCMain12             : VAEntrypointVLD
      VAProfileHEVCMain12             : VAEntrypointEncSlice
      VAProfileHEVCMain422_10         : VAEntrypointVLD
      VAProfileHEVCMain422_10         : VAEntrypointEncSlice
      VAProfileHEVCMain422_12         : VAEntrypointVLD
      VAProfileHEVCMain422_12         : VAEntrypointEncSlice
      VAProfileHEVCMain444            : VAEntrypointVLD
      VAProfileHEVCMain444            : VAEntrypointEncSliceLP
      VAProfileHEVCMain444_10         : VAEntrypointVLD
      VAProfileHEVCMain444_10         : VAEntrypointEncSliceLP
      VAProfileHEVCMain444_12         : VAEntrypointVLD
      VAProfileHEVCSccMain            : VAEntrypointVLD
      VAProfileHEVCSccMain            : VAEntrypointEncSliceLP
      VAProfileHEVCSccMain10          : VAEntrypointVLD
      VAProfileHEVCSccMain10          : VAEntrypointEncSliceLP
      VAProfileHEVCSccMain444         : VAEntrypointVLD
      VAProfileHEVCSccMain444         : VAEntrypointEncSliceLP
      VAProfileAV1Profile0            : VAEntrypointVLD
      VAProfileHEVCSccMain444_10      : VAEntrypointVLD
      VAProfileHEVCSccMain444_10      : VAEntrypointEncSliceLP
```
~~## 8.安装完成后，安装vainfo查看解码能力~~
```
apt install vainfo
```
经Harry Chen大神指点，apt源里的vainfo依赖libva，若使用apt命令安装，会拉回来一个debian编译的版本，所以此步略过，且上面6.4步骤构建libva-utils时，已编译vainfo，可直接输入 vainfo查看解码能力，如第7步输出即有编解码能力，此时emby可直接解码，但是不知为何，解码性能极弱，遂放弃，还是得选用jellyfin，但由于最好用的nyanmisaka大佬的jellyfin集成的iHD_drv_video驱动与dg1并不兼容，需要使用上步中icarusradio大佬编译的驱动进行替换。
## 9.TrueNAS-SCALE安装docker
在 TrueNAS SCALE 上，Docker 不像在其他 Linux 发行版上那样直接安装。TrueNAS SCALE 本身是基于 Debian 的，但它主要依赖 K3S 和 Helm 来管理容器。如果你真的需要在 TrueNAS SCALE 上使用 Docker，可以考虑以下方法：
### 9.1. 使用 Shell 安装 Docker，虽然不推荐，但如果你确实需要安装 Docker，可以尝试以下命令：
# 更新包列表
```
sudo apt update
```
# 安装必要的包
```
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
```
# 添加 Docker 的官方 GPG 密钥
```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
```
# 添加 Docker 的稳定版本仓库
```
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
```
# 再次更新包列表
```
sudo apt update
```
# 安装 Docker
```
sudo apt install -y docker-ce
```
### 9.2. 启用 Docker 服务
安装完成后，你可以启动 Docker 服务：
```
sudo systemctl start docker
sudo systemctl enable docker
```
### 9.3. 验证安装
运行以下命令检查 Docker 是否正常工作：
```
sudo docker --version
```
### 注意事项
1.不推荐: 这种方式可能会导致与 TrueNAS SCALE 的容器管理不兼容，因为它不使用 Docker 而是 K3S。
2.推荐使用 K3S: 建议使用 K3S 和 Helm 来管理你的容器，如之前提到的 Jellyfin 安装方法。

## 10.安装jellyfin并替换驱动
### 10.1.docker安装jellyfin（个人设置，请自行修改参数）
```
sudo docker run -d \
 --name=jellyfin \
 --mount type=bind,source=/mnt/Applications/Jellyfin/config,target=/config \
 --mount type=bind,source=/mnt/Applications/Jellyfin/cache,target=/cache \
 --mount type=bind,source=/mnt/TrueNAS/Movies,target=/media/movies \
 --mount type=bind,source=/mnt/TrueNAS/Dramas,target=/media/dramas \
 --user 1000:1000 \
 --group-add="110" \
 --net=host \
 --restart=unless-stopped \
 --device /dev/dri/renderD128:/dev/dri/renderD128 \
 -p 8097:8096 \  # 映射新端口
 nyanmisaka/jellyfin
```
### 10.2.替换定制iHD驱动(节选自[```https://icarusradio.github.io/guides/ubuntu-dg1-jellyfin.html```](https://icarusradio.github.io/guides/ubuntu-dg1-jellyfin.html))
安装完成 Jellyfin 后，输入指令进入 docker 内命令行：
```
sudo docker exec -it jellyfin /bin/bash
```
在命令行内输入指令测试 ffmpeg：
```
/usr/lib/jellyfin-ffmpeg/vainfo --display drm --device /dev/dri/renderD128
```
此时应该会报错，报错内容类似如下：
```
Trying display: drm
libva info: VA-API version 1.21.0
libva info: Trying to open /usr/lib/jellyfin-ffmpeg/lib/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_21
double free or corruption (!prev)
Aborted (core dumped)
```
因为我们还没有替换驱动文件，docker 内自带的文件不支持 DG1。这里我编译好了驱动文件，请去这个仓库下载。这个仓库我会尽量保持与 Intel 官方最新版本一致。
用以下指令将下载的 iHD_drv_video.so 替换 docker 内的：
```
sudo docker cp iHD_drv_video.so jellyfin:/usr/lib/jellyfin-ffmpeg/lib/dri/iHD_drv_video.so
```
此时再输入上述指令进入 docker 内命令行并执行测试 ffmpeg 指令，如果出现类似输出证明替换成功：
```
Trying display: drm
libva info: VA-API version 1.21.0
libva info: Trying to open /usr/lib/jellyfin-ffmpeg/lib/dri/iHD_drv_video.so
libva info: Found init function __vaDriverInit_1_20
libva info: va_openDriver() returns 0
vainfo: VA-API version: 1.21 (libva 2.21.0)
vainfo: Driver version: Intel iHD driver for Intel(R) Gen Graphics - 24.1.5 ()
vainfo: Supported profile and entrypoints
      VAProfileNone                   : VAEntrypointVideoProc
      VAProfileNone                   : VAEntrypointStats
      VAProfileMPEG2Simple            : VAEntrypointVLD
      VAProfileMPEG2Simple            : VAEntrypointEncSlice
      VAProfileMPEG2Main              : VAEntrypointVLD
      VAProfileMPEG2Main              : VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointVLD
      VAProfileH264Main               : VAEntrypointEncSlice
      VAProfileH264Main               : VAEntrypointFEI
      VAProfileH264Main               : VAEntrypointEncSliceLP
      VAProfileH264High               : VAEntrypointVLD
      VAProfileH264High               : VAEntrypointEncSlice
      VAProfileH264High               : VAEntrypointFEI
      VAProfileH264High               : VAEntrypointEncSliceLP
      VAProfileVC1Simple              : VAEntrypointVLD
      VAProfileVC1Main                : VAEntrypointVLD
      VAProfileVC1Advanced            : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointVLD
      VAProfileJPEGBaseline           : VAEntrypointEncPicture
      VAProfileH264ConstrainedBaseline: VAEntrypointVLD
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSlice
      VAProfileH264ConstrainedBaseline: VAEntrypointFEI
      VAProfileH264ConstrainedBaseline: VAEntrypointEncSliceLP
      VAProfileHEVCMain               : VAEntrypointVLD
      VAProfileHEVCMain               : VAEntrypointEncSlice
      VAProfileHEVCMain               : VAEntrypointFEI
      VAProfileHEVCMain               : VAEntrypointEncSliceLP
      VAProfileHEVCMain10             : VAEntrypointVLD
      VAProfileHEVCMain10             : VAEntrypointEncSlice
      VAProfileHEVCMain10             : VAEntrypointEncSliceLP
      VAProfileVP9Profile0            : VAEntrypointVLD
      VAProfileVP9Profile0            : VAEntrypointEncSliceLP
      VAProfileVP9Profile1            : VAEntrypointVLD
      VAProfileVP9Profile1            : VAEntrypointEncSliceLP
      VAProfileVP9Profile2            : VAEntrypointVLD
      VAProfileVP9Profile2            : VAEntrypointEncSliceLP
      VAProfileVP9Profile3            : VAEntrypointVLD
      VAProfileVP9Profile3            : VAEntrypointEncSliceLP
      VAProfileHEVCMain12             : VAEntrypointVLD
      VAProfileHEVCMain12             : VAEntrypointEncSlice
      VAProfileHEVCMain422_10         : VAEntrypointVLD
      VAProfileHEVCMain422_10         : VAEntrypointEncSlice
      VAProfileHEVCMain422_12         : VAEntrypointVLD
      VAProfileHEVCMain422_12         : VAEntrypointEncSlice
      VAProfileHEVCMain444            : VAEntrypointVLD
      VAProfileHEVCMain444            : VAEntrypointEncSliceLP
      VAProfileHEVCMain444_10         : VAEntrypointVLD
      VAProfileHEVCMain444_10         : VAEntrypointEncSliceLP
      VAProfileHEVCMain444_12         : VAEntrypointVLD
      VAProfileHEVCSccMain            : VAEntrypointVLD
      VAProfileHEVCSccMain            : VAEntrypointEncSliceLP
      VAProfileHEVCSccMain10          : VAEntrypointVLD
      VAProfileHEVCSccMain10          : VAEntrypointEncSliceLP
      VAProfileHEVCSccMain444         : VAEntrypointVLD
      VAProfileHEVCSccMain444         : VAEntrypointEncSliceLP
      VAProfileAV1Profile0            : VAEntrypointVLD
      VAProfileHEVCSccMain444_10      : VAEntrypointVLD
      VAProfileHEVCSccMain444_10      : VAEntrypointEncSliceLP
```
## 9.测试gpu负载
如果需要检测 GPU 的利用率，可以（在 Docker 外即可）安装 intel-gpu-tools 软件包。它提供了 intel_gpu_top 命令，可以实时查看 GPU 的各个组件的利用率。
