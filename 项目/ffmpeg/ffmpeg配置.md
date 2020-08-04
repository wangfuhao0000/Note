#### 安装依赖

首先，安装一下所需要的依赖：

```
sudo apt-get install -y autoconf automake build-essential git libass-dev libfreetype6-dev libsdl2-dev libtheora-dev libtool libva-dev libvdpau-dev libvorbis-dev libxcb1-dev libxcb-shm0-dev libxcb-xfixes0-dev pkg-config texinfo wget zlib1g-dev 
```

```shell
sudo apt-get install -y nasam yasm cmake mercurial 
```

#### 安装

然后，将ffmpeg的源码下载下来：

```shell
git clone https://git.ffmpeg.org/ffmpeg.git ffmpeg
```

接着就进行安装，这里我们将ffmpeg安装到目录`/usr/local/ffmpeg`下。首先进入到ffmpeg目录下，依次执行下面的安装语句：

```shell
./configure --enable-shared  --prefix=/usr/local/ffmpeg
sudo make
sudo make install
```

这样我们就可以在目录`/usr/local/ffmpeg`下看到安装的内容了，包括目录`lib`、`bin`、`include`等。

#### 添加链接

安装后我们需要能然我们的程序被链接，通过编辑文件`/etc/ld.so.conf`如下，在末尾添加如下语句（需要管理员权限`sudo`）：

```shell
/usr/local/ffmpeg/lib
/usr/local/lib
/usr/lib
```

编辑完成后进行更新：

```shell
sudo ldconfig
```

要注意每次要重新安装ffmpeg（可能缺失某些包）

#### CMake测试

使用Cmake程序编译时进行测试，需要我们手动指定ffmpeg的**头文件路径**和**库文件目录**。它们分别是使用命令`include_directories`和`link_directories`实现的。而刚才我们安装完ffmpeg后：

- **它的头文件.h在`/usr/local/ffmpeg/include`下**
- **它的库文件.so在`/usr/local/ffmpeg/lib`下**

这里我直接提供一个`CMakeLists.txt`的demo，更加清晰一些：

```makefile
cmake_minimum_required(VERSION 2.8)
PROJECT(testFFMpeg)
SET(SRC_LIST 0_hello_world.c)  # 设置所有可执行的文件赋值给变量SRC_LIST
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
include_directories("/usr/local/ffmpeg/include")  # 头文件目录
link_directories("/usr/local/ffmpeg/lib")   # 库文件目录
ADD_EXECUTABLE(testFFMpegDemo ${SRC_LIST})
# libpostproc.so
target_link_libraries(testFFMpegDemo libavutil.so libavcodec.so libavformat.so libavdevice.so libavfilter.so libswscale.so)   # 链接库
```

具体CMake相关的指令可查看[此博文](https://blog.csdn.net/lanchunhui/article/details/57574867)或者[这一篇](https://www.cnblogs.com/binbinjx/p/5626916.html)。