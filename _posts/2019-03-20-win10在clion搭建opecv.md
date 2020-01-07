## win10在clion搭建opencv开发环境

### 1 版本(我的环境)

- **clion** 2018.3.4
- **cmake** 3.12.4
- **TDM-GCC** (推荐这个，其他的都没有编译通过)
- **open-cv** 3.4

### 2 配置

打开cmake安装路径的bin下边的cmake-gui.exe

- 手动下载一个opencv_ ffmpeg_64.dll文件，放到opencv/sources/3rdparty/ffmpeg/目录下，下载地址： opencv_ffmpeg_64。

- opencv_ ffmpeg.dll，也需要放到opencv/sources/3rdparty/ffmpeg/目录下，下载地址：[opencv_ ffmpeg.dll](https://link.jianshu.com?t=http%3A%2F%2Fdownload.csdn.net%2Fdownload%2Fu010013521%2F9437481)。

#### 2.1 编译

- 选择需要编译的opencv源码，如图所示：

  ![cmake-1](https://raw.githubusercontent.com/liurio/deep_learning/master/img/cmake-1.jpg)

  会出现一片红，这时候是未编译的状态

- 去掉几个(不去会报错)
  - with_matlab
  - install_tests、build_tests
  - enable_precompiled_headers
- 如果一直出现红色，可一直点击configure，可提前把
- 没有错误时，点击generate

#### 2.2 安装

在选择的build_输出目录下，打开cmd进行编译

```bash
mingw32-make -j8 # 以8线程进行编译
#完成后，输出目录下的bin目录里会生成一些.dll和.exe文件，lib目录会生成一些.a文件。
mingw32-make install
# 输出目录下会多出install文件夹，
# 添加...\install\x86\mingw\bin 添加到path系统环境变量环境变量；
```

### 3 测试

![cmake-2](https://raw.githubusercontent.com/liurio/deep_learning/master/img/cmake-2.jpg)

- 新建一个工程，将opencv的build目录bin下的文件拷贝到工程的cmake-build-debug下(要不找不到dll文件)。

- 编辑cmakelist文件

```txt
cmake_minimum_required(VERSION 3.6.3)

project(opencv)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17")
# Where to find CMake modules and OpenCV  D:\note\opencv-3.4\opencv\build_clion
set(OpenCV_DIR "D:\\note\\opencv-3.4\\opencv\\build_clion\\install")
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/")
find_package(OpenCV REQUIRED)
include_directories(${OpenCV_INCLUDE_DIRS})
add_executable(opencv main.cpp)
# add libs you need
set(OpenCV_LIBS opencv_core opencv_imgproc opencv_highgui opencv_imgcodecs)
# linking
target_link_libraries(opencv ${OpenCV_LIBS})
```

- 写测试代码

```c++
#include "iostream"
#include <opencv2/core/core.hpp>
#include <opencv2/highgui/highgui.hpp>
using namespace std;
using namespace cv;

int main() {
	Mat img = imread("haha.jpg");
	if (img.empty()) {
		cout << "Error" << endl;
	return -1;
	}
	imshow("Lena", img);
	waitKey();
	return 0;
}
```

![cat](https://raw.githubusercontent.com/liurio/deep_learning/master/img/cat.jpg)





 

