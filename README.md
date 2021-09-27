<h1>基于Jetso Nano的嵌入式环境配置</h1>
本文档包含了以下各项内容：

* [开发板参数](#1)
* [目标代码运行环境](#2)
* [开发板基本配置](#3)
    
    * [更换国内源](#3.1)
    * [系统重刷与系统盘迁移](#3.2)
    * [Jtop工具使用](#3.3)
    * [JetPack配置检查](#3.4)
    * [线刷JetPack4.4.1](#3.5)
    * [卡刷JetPack4.4.1](#3.6)
* [Jetson nano 上编译 Opencv3.4.2](#4)
* [Jetson nano 上编译 Onnxruntime](#5)
* [Jetson nano 上编译 目标检测代码](#6)
 
<h2 id='1'> 开发板参数</h2>

    * 边缘盒子：图为T100
    * 硬件型号：Jetson Nano B01 
    * 操作系统：Ubuntu18.04
    * CPU型号 ：Cortex-A57
    * CPU架构 ：ARMv8-A (aarch64)
    * 显卡型号：128-core NVIDIA Maxwell @ 921MHz CUDA10
    * 运行内存：4 GB 64-bit LPDDR4, 1600M
    * 存储空间：16GB eMMC (Nano 模块内置) + M.2 Key M NVMe 2280 (256G固态硬盘)
<h2 id='2'> 目标代码运行环境</h2>

    * Opencv 3.4.2
    * Onnxruntime 1.7.0
    * Cuda 10.2.89
    * Cudnn 8.0.0.180
    * TensorRT 7.1.3.0
<h2 id='3'> 开发板的基础配置 (参考T100使用说明)</h2>

<h3 id='3.1'> 1. 更换国内镜像源 (华为源)</h3>

>$ sudo wget -O /etc/apt/sources.list https://repo.huaweicloud.com/repository/conf/Ubuntu-Ports-bionic.listsudo 
>
>$ sudo apt-get update

##### ***注：如非必要不可以使用 sudo apt-get upgrade，直接升级内核会覆盖设备树，使用下面指令后才能使用upgrade指令，误用只能重刷系统*** 
>$ sudo rm /etc/apt/sources.list.d/nvidia-l4t-apt-source.list
<h3 id='3.2'> 2. Jetson nano 线刷系统 & 设置256G固态硬盘为系统</h3>

##### ***线刷的意思就是利用数据线，将系统刷入开发板上，有重要的文件请先备份出来！这里你需要准备的工具有：***
    * 下载好的L4T系统包
    * 一个装有Ubuntu系统的主机
    * 一根 micro USB 数据线
    * 两支牙签或者笔(尖尖的东西均可)  
##### ***更换系统盘是必须的，默认盘只有16G，必须进行系统盘的迁移***
##### 第一步：主机上下载对应版本的L4T系统包，[百度网盘](https://pan.baidu.com/s/1t5J_eIjLAogLQtA-Q6qDdw) 提取密码：d54d 这里以本开发板为例，需要下载的是 JetPack4.4.1_T100_T101 下面的 t100.v1.3 (这个是驱动包) 以及 Linux_for_Tegra.gz (这个是系统固件包)，然后解压系统包，将驱动拷贝到系统包下面：
>$ sudo tar zxvf Linux_for_Tergra.gz
>
>$ cp -r ./t100.v1.3/Linux_for_Tergra/* ./Linux_for_Tergra/
##### 第二步：让开发板进入恢复模式，数据线连接开发板和主机，开发板上电，先按住RECOVERY按键然后按住RESET复位键(都别松开)，然后释放RESET按键，两秒后释放RECOVERY按键进入恢复模式。这里可以使用 lsusb 指令来查看是否出现 Nvidia Corp. 设备进行验证
> $ lsusb
>
> Bus 003 Device 005: ID 0955:7c18 Nvidia Corp. <font color=green size=2>//出现类似类似语句说明连接成功 </font>
##### 第三步：主机中进入第一步准备的工具包进行系统的刷入

>$ cd Linux_for_Tegra
>
>$ sudo ./flash.sh jetson-nano-emmc mmcblk0p1 <font color=green size=2>//这一步会格式系统</font>

##### 第四步：等待刷机完成进入系统后，如果出现WiFi无法正常使用的情况(WIFI正常，跳过此步)，检查rt18821cu.ko是否存在，目录为：
>$ cd /lib/modules/4.9.140-tegra/kernel/drivers/net/wireless/realtek/rtl8821cu/
##### 如果不存在该文件夹或者文件夹下面没有 rt18821cu.ko 文件，可以在第一步网盘中的 USB WIFI 文件夹下载创建目录放入。然后进行模块的载入。  
>$ sudo modprobe cfg80211 <font color=green size=2>//载入模块 </font>
> 
>$ sudo depmod -a <font color=green size=2>//分析可载入模块的相依性 </font>
>
>$ sudo modprobe rtl8821cu
>
>$ sudo echo rtl8821cu >> /etc/modules <font color=green size=2>//将驱动添加到开机启动列 </font>
##### ***注：和 insmod相比 modprobe 可以解决 load module时的依赖关系***
##### ***注：安装完系统建议更换国内镜像源***
##### 第五步：设置256G固态硬盘为系统盘：点击左上角或者按下 win 按键搜索打开 disks ,可以看到一块固态硬盘，然后右键将其格式化为Ext4模式，然后装载运行，进入固态中打开终端运行下列指令：
>$ git clone https://github.com/jetsonhacks/rootOnNVMe.git
>
>$ cd rootOnNVMe
>
>$ ./copy-rootfs-ssd.sh
>
>$ ./setup-service.sh
>
>$ reboot
##### 重启完成系统安装与系统盘的迁移
<h3 id='3.3'>3. 检查安装Jtop工具 </h3>

##### 一款系统监控程序。使用指令 jtop 进行查看系统的cpu、gpu、内存等信息  
##### Jtop工具安装指令：
>$ sudo apt-get install python3-pip
>
>$ sudo pip3 install jetson-stats <font color=green size=2>//包含Jtop</font>
>
>$ sudo jtop
<h3 id='3.4'> 4. 检查系统JetPack配置</h3>

##### JetPack主要包含了Cuda、Cudnn、TensorRT、Opencv 等软件的安装，可以使用以下指令直接查看各个版本信息：
>$ jetson-release
##### 对于单独查看各个软件版本的指令为：  
##### 3.1 查看 Cuda 版本：
>$ nvcc --version
##### 3.2 查看 TensorRT 版本：
>$ python3

>>import tensorrt
>>
>>tensorrt.\_\_version\_\_ <font color=green size=2>//注意这里有两个下划线，保证返回的只有版本型号7.1.3.0</font>

##### ***以上如果 Cuda 或者 TensorRT 出现问题需要重新安装 JetPack 系列，如果担心卸载不干净可以尝试重新给Jetson Nano刷入系统，当然刷入系统后仍要重新刷入JetPack*** 
##### ***如果系统中没有 JetPack 或者 JetPack 中TensorRT 出现了问题，可以采用线刷或者卡刷的方式进行 JetPack 刷入，下面分别介绍这两种刷入方式***
<h3 id='3.5'> 5. 线刷 JetPack Manager<h3>

##### ***线刷就是在主机上下载好官方给出的 SDK Manager 然后用数据线在恢复模式下连接开发板，给开发板刷入 jetpack 的方式，刷入的过程受制于网络，国外网络刷入较慢，比较随缘。不过刷入后的工具比较齐全，还是比较建议线刷的。在决定线刷后你需要这些：***
    * 一台安装ubuntu18.04的主机(其他我没试过)
    * 官网免费注册nvidia账号
    * 主机安装SDK manager
    * 一根micro usb数据线
##### 附: 你可以从官网下载：[SDK Manager](https://developer.nvidia.com/sdkmanager_deb) 或者你也可以从：[百度网盘](https://pan.baidu.com/s/1HOLORIpblrhpmJvzweYfOg) 下载，提取密码：tw12
##### 第一步：下载 SDK Manager 安装包，然后安装到主机上
>$ sudo dpkg -i ./sdkmanager_1.6.0-8170_amd64.deb
##### 第二步： 开发板连接主机，打开SDK Manager软件。这里有几个需要注意的点：
    * 去掉勾选 Host Machine(不需要主机上安装)
    * Target Hardware 显示为Detected则表示设备已与ubuntu电脑主机连接(需要出现Detected，如果一直不出现可以试试进入恢复模式)
    * 首次连接弹出选择硬件的窗口选择Jetson Nano(不是kit版本的)
    * jetpack版本要选择和系统对应的版本，这里应该选择JetPack4.4.1
    * 不需要勾选DeepStream，因为我们用不到
##### 第三步：进入 STEP 02 这里你需要注意的有：
    * 不需要勾选Jetson OS, 因为不需要重刷系统
    * 勾选Jetson SDK Components 会下载包含的所有组件
    * Download folder 这个是下载到主机的路径，保证空间足够
    * 接受条款
    * 勾选Download now. Install later.把所有组件下载到主机，这里推荐这么做，方便后期网络问题多次刷入。
    * 下一步安装即可
##### 第四步： 安装完成会出现Finish的按钮，注意这里你只是离线下载了所有的组件，并没有自动给你刷到刷入到开发板上，因此需要完成安装后，重新打开软件，重复第二步，然后在第三步的时候去除Download now. Install later的选项，进行刷入即可。有几个需要注意的点：
    * 这里可能会出现一个Disk space的错误，说开发板空间不足，直接Skip即可。
    * 安装过程需要连网，会下载各种组件，并且会出现卡住的情况，原因是外网连接不流畅，可以选择等待，超时的话继续下载，刷入过程开发板通过数据线共享主机网络环境，卡住时也可以通过关闭打开网络来尝试恢复下载组件。
##### 经过很长时间，刷入完成，开机检查 JetPack 配置即可。
<h3 id='3.6'> 6. 卡刷JetPack4.4.1</h3>

##### ***卡刷的意思是将安装包上传到开发板上，然后直接在终端进行安装即可，卡刷比较简单，但是不保证安装完会遇到缺少组件的情况***
##### 附：JetPack4.4 卡刷包下载：[百度网盘]( https://pan.baidu.com/s/1XqLujhw5dc423WBGJsXcgg) 提取密码：x7ck
##### 第一步：安装cuda10.2 
>$ sudo dpkg -i /opt/nvidia/deb_repos/cuda-repo-l4t-10-2-local-10.2.89_1.0-1_arm64.deb
>
><font color=green size=2>//安装完成会提示pub key， 根据提示添加apt key(.pub)，例如:</font>
>
>$ sudo apt-key add /var/cuda-repo-10-2-local-10.2.89/7fa2af80.pub
>
>$ sudo apt-get -y update
>  
>$ sudo apt-get -y  install cuda-toolkit-10-2 
>
>$ nvcc -V
>
><font color=green size=2>//以上安装完成后，通过nvcc 查询不到，但可以搜索目录/usr/local 是否有cuda，此时可通过添加环境变量 ~/.bash.rc </font>
>
>$ sudo vi ~/.bashrc 
>
> export PATH=/usr/local/cuda-10.2/bin:$PATH
>
> export LD_LIBRARY_PATH=/usr/local/cuda-10.2/lib64:$LD_LIBRARY_PATH
>
>$ source ~/.bashrc
##### 第二步：安装cudnn8.0
><font color=green size=2>//依次安装文件夹下所有安装包</font>
>
>$ sudo dpkg -i libcudnn8_8.0.0.180-1+cuda10.2_arm64.deb
>
>$ sudo dpkg -i libcudnn8-dev_8.0.0.180-1+cuda10.2_arm64.deb
>
>$ sudo dpkg -i libcudnn8-doc_8.0.0.180-1+cuda10.2_arm64.deb
>
>$ sudo apt-get -y update 
>
><font color=green size=2>//以上安装完成后，通过jetson_release 查看安装后的版本信息</font>
>
> $ jetson_release 
##### 第三步：安装TensorRT7.1.3
><font color=green size=2>//依次安装文件夹下所有安装包</font>
>
>$ sudo dpkg -i libnvinfer7_7.1.3-1+cuda10.2_arm64.deb
> 
>$ sudo dpkg -i libnvinfer-dev_7.1.3-1+cuda10.2_arm64.deb 
>
>$ sudo dpkg -i libnvinfer-plugin7_7.1.3-1+cuda10.2_arm64.deb 
>
>$ sudo dpkg -i libnvinfer-plugin-dev_7.1.3-1+cuda10.2_arm64.deb 
> 
>$ sudo dpkg -i libnvonnxparsers7_7.1.3-1+cuda10.2_arm64.deb 
>
>$ sudo dpkg -i libnvonnxparsers-dev_7.1.3-1+cuda10.2_arm64.deb 
>
>$ sudo dpkg -i libnvparsers7_7.1.3-1+cuda10.2_arm64.deb
> 
>$ sudo dpkg -i libnvparsers-dev_7.1.3-1+cuda10.2_arm64.deb 
>
>$ sudo dpkg -i libnvinfer-bin_7.1.3-1+cuda10.2_arm64.deb 
>
>$ sudo dpkg -i libnvinfer-doc_7.1.3-1+cuda10.2_all.deb
>  
>$ sudo dpkg -i libnvinfer-samples_7.1.3-1+cuda10.2_all.deb
> 
>$ sudo dpkg -i tensorrt_7.1.3.0-1+cuda10.2_arm64.deb 
>
>$ sudo dpkg -i python-libnvinfer_7.1.3-1+cuda10.2_arm64.deb 
>
>$ sudo dpkg -i python-libnvinfer-dev_7.1.3-1+cuda10.2_arm64.deb 
>
>$ sudo dpkg -i python3-libnvinfer_7.1.3-1+cuda10.2_arm64.deb 
>
>$ sudo dpkg -i python3-libnvinfer-dev_7.1.3-1+cuda10.2_arm64.deb 
>
>$ sudo dpkg -i graphsurgeon-tf_7.1.3-1+cuda10.2_arm64.deb
> 
>$ sudo dpkg -i uff-converter-tf_7.1.3-1+cuda10.2_arm64.deb
>
>$ sudo apt-get -y update
>
><font color=green size=2>//以上安装完成后，通过jetson_release 查看安装后的版本信息,进入python3中调用tensorrt输出版本信息</font>
>
>$ jetson_release
>
>$ python3
>
>>import tensorrt
>>
>>tensorrt.\_\_version\_\_ <font color=green size=2>//这里两个下划线</font>
<h2 id='4'>Opencv 3.4.2在Jetson nano上的编译 </h2>

编译过程中需要注意的是：

    * FFmpeg的编译 (用于视频流的解码读取)
    * GTK组件的编译 (用于窗口的创建)
##### 源码包的下载：[Opencv 3.4.2](https://github.com/opencv/opencv/archive/refs/tags/3.4.2.tar.gz)
##### 编译包的下载：[百度网盘](https://pan.baidu.com/s/1FFTX5ZKtxmnSy0aUS0UXoA) 提取密码：1o16

##### 第一步：删除本地的opencv环境
>$ sudo apt-get purge libopencv*
>
>$ sudo apt autoremove
>
>$ sudo apt-get update
##### 第二步：下载opencv依赖的库
>sudo apt-get install build-essential libglew-dev libtiff5-dev zlib1g-dev libjpeg-dev libavcodec-dev libavformat-dev libavutil-dev libpostproc-dev libswscale-dev libeigen3-dev libtbb-dev libgtk2.0-dev pkg-config libpng-dev libatlas-base-dev gfortran
##### 第三步：cmake版本升级 需要下载源码包进行安装
>$ sudo apt-get remove cmake
>
>$ ln -sf  /home/cmake-3.21/bin/* /usr/bin/ <font color=green size=2>//cmake源码bin文件夹建立软链接</font>
##### 第四步：opencv编译
    在明确了需要编译进去的功能后，进入到opencv3.4.2源码目录下，创建build文件夹。
> $ cmake \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_PNG=ON \
    -DBUILD_JPEG=ON \
    -DBUILD_ZLIB=ON \
    -DWITH_FFMPEG=ON \
    -DWITH_GSTREAMER=ON \
    -DWITH_GTK=ON \
    -DBUILD_opencv_world=ON \
    -CMAKE_INSTLL_PREFIX=/usr/local/arm64  \
    ..

<font color=green size=2>//这里解释一下：这里上面的指令为一句话，opencv_world表示构建整合包，INSTALL_PREFIX为你的安装目录。命令是在build目录下进行的，最后一行的 . . 用于进入到上层来生成。</font>

>$ sudo make -j4 <font color=green size=2>//等待20min</font>
>
>$ sudo make install
##### 报错首先查看依赖问题，尝试安装依赖包解决
##### 第五步：opencv的移植
    在编译安装好opencv3.4.2后，你会在安装目录下生成四个文件夹。需要注意的是lib下面有opencv_world文件，说明编译时的-DBUILD_opencv_world选项有效。

##### 配置 pkg-config
    打开/etc/ld.so.conf文件，在最后一行添加刚才编译输出的库路径/usr/local/arm64/lib，再进行ldconfig等配置，具体过程如下：
>$ vim /etc/ld.so.conf <font color=green size=2>//将/usr/local/arm64/lib添加到文件末尾</font>
>
>$ sudo ldconfig -v
>
>$ export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/local/arm64/lib/pkgconfig 
>
><font color=green size=2>//注意，export只在当前终端有效，切换或者重新打开需要重新export</font> 
>
##### 修改 $(OPENCV)/lib/pkgconfig/opencv.pc ，注意核对一下pc文件中的目录是否准确，在移植的过程不会自动改变，需要你根据目录自行修改
>$ pkg-config --libs --cflags opencv <font color=green size=2>//用于核对opencv是否可以正常引用，正常情况会返回-I和-L各种依赖库路径类似下图。</font>
>
>-I/usr/local/arm64_opencv/include/opencv -I/usr/local/arm64_opencv/include -L/usr/local/arm64_opencv/lib -lopencv_world
##### 第六步：直接调用opencv
>g++ -o Test1 ./Test1.cpp \`pkg-config --cflags --libs opencv\`
><font color=green size=2>//注意这里的 \` 是键盘左上角数字键1左边的</font>
<h2 id='5'> Onnxruntime1.7.0 在Jetson nano上的编译</h2>

##### 第一步：获取Onnxruntime1.7.0的源码包。
##### Onnxruntime1.7.0 源码包：[百度网盘](https://pan.baidu.com/s/1vKsWAE34lPdYBLho5QlGsg) 提取密码：8wtk
    * 当然你也可以从Github上进行git，但是一定要保证能拉取到完整的源码包！因为源码里面很多内容是链接到了别的文件。如果你懒得查找可以直接从网盘获取。
##### 第二步：在Jetson Nano上编译Onnxruntime
    * 这里你首先需要安装好一些支持包
>$ sudo apt-get update
>
>$ sudo apt-get install -y build-essential curl libcurl4-openssl-dev libssl-dev wget python3-pip git tar 
>
>$ pip3 install --upgrade pip
>
>$ pip3 install --upgrade setuptools
>
>$ pip3 install --upgrade wheel
>
>$ pip3 install numpy
>
>$ sudo apt install -y --no-install-recommends build-essential software-properties-common libopenblas-dev libpython3.6-dev python3-pip python3-dev protobuf-compiler libprotoc-dev
>
><font color=green size=2>/*接下来进入到onnxruntime目录,执行安装程序，这里你需要修改的是tensorrt_home路径、cuda_home路径、cudnn_home路径，这里的--use_openmp意思是开发板上的多核并行编译\*/</font>
>
>$ ./build.sh --update --config Release --enable_pybind --build_shared_lib --build --build_wheel --use_openmp --use_tensorrt --tensorrt_home /usr/src/tensorrt --cuda_home /usr/local/cuda --cudnn_home /usr/lib/aarch64-linux-gnu
<font color=green size=2>
>
>//大概经过5h，你可以在开发板上编译完成</font>

##### 如果你不想等待这么长的时间，你可以直接下载我编译好的版本：[百度网盘](https://pan.baidu.com/s/1_NRd4dHbFXZqIQrkp3E7xQ) 提取密码：kcly

<h2 id='6'>目标检测源码在Jetson Nano上的编译运行</h2>

    目标检测源码目前有两个版本：
    * 第一个版本的源码包含了使用仅仅Cuda的模式(当前开发板下可以达到平均帧率为2.6帧)
    * 使用TensorRT加速的模式(当前开发板下能达到平均帧率为4.6帧率，但是每次运行都需要花费四五分钟来生成TensorRT的引擎)。
    * 第二个版本只有初次运行时生成一个TensorRT引擎，在之后的运行中只需要调用该引擎即可，平均帧率可以达到4.6帧率
##### 这里提供两个版本的源码包和引擎文件。[百度网盘](链接：https://pan.baidu.com/s/1N1yLAop6MngdMIdWeqNT4w) 提取码: tpl9 


##### 源码的编译过程你需要提前准备的是： 
##### 第一步：boost、atlas依赖库的下载：
>$ sudo apt-get install libboost-dev
>
>$ sudo apt-get install libatlas-base-dev
##### 第二步：CmakeLists文件修改
    这里你需要对照修改的是：
    * 头文件部分，对照objectdetection.cpp文件中的头文件进行修改
    * 库文件部分，对照下面列出来的所需要的依赖库的库名进行查找修改
    * 库名部分，在文件夹中完整的库名应该是lib---.so的形式，这里直接写库名即可
<font color=green size=2>这里给出一个版本一的CmakeLists参考</font>

```
# ------------------------------头文件部分----------------------------------------------
set(CUDA_INC_DIR //usr/local/cuda-10.2/targets/aarch64-linux/include)     
set(CUDNN_INC_DIR /usr/include)
set(OpenCV_INC_DIR /usr/local/arm64/include)
set(ORT_INC_DIR  /home/epic/onnxruntime/include/onnxruntime/core)
set(INC_DIR ${CUDA_INC_DIR} ${CUDNN_INC_DIR} ${OpenCV_INC_DIR} ${ORT_INC_DIR})
# ------------------------------------------------------------------------------------

# ----------------------------库文件部分------------------------------------------------
set(CUDA_LIB_DIR /usr/local/cuda-10.2/targets/aarch64-linux/lib)
set(CUDNN_LIB_DIR /usr/lib/aarch64-linux-gnu)
set(OpenCV_LIB_DIR /usr/local/arm64/lib)
set(ORT_LIB_DIR /home/epic/onnxruntime/build/Linux/Release)
set(LIB_DIR ${CUDA_LIB_DIR} ${CUDNN_LIB_DIR} ${OpenCV_LIB_DIR} ${ORT_LIB_DIR})
# ------------------------------------------------------------------------------------

# ----------------------------库文件需要链接的库名----------------------------------------
set(CUDA_LIBS  cudart curand cufft nppig)
set(CUDNN_LIBS cudnn cublas)
set(OpenCV_LIBS  opencv_world)
set(ORT_LIBS onnxruntime custom_op_library onnxruntime_providers_shared onnxruntime_providers_tensorrt)
set(LINK_LIBS ${CUDA_LIBS} ${CUDNN_LIBS} ${OpenCV_LIBS} ${ORT_LIBS} )
#-----------------------------------------------------------------------------
```
##### 第三步：修改main.cpp中的文件
    * optim_tn.onnx 的位置 你的引擎
    * tn_label.txt 的位置 你的引擎标签
    * video_path 的位置 你检测的视频位置
##### 第四步：分配虚拟内存
    Jetson nano只有4G的内存，在直接运行TensorRT加速模式时会出现内存不足，进程killed的中断，这里的解决办法是使用虚拟内存来扩充。下面给出一个简单的配置项目。
>$ git clone https://github.com/JetsonHacksNano/installSwapfile
>
>$ cd installSwapfile/
>
>$ sudo ./installSwapfile.sh
##### 第五步：编译运行源码
    在源码目录下创建build文件夹，进入build文件夹下使用指令编译运行：
>$ cmake ..
>
>$ make
>
>$ ./tnProject


 



