## ubuntu下安装caffe(cpu)

### 更换镜像源

```bash
sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup
sudo vim /etc/apt/sources.list  
```

```sh
## 将之前的注释掉，更换为
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-updates main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-backports main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-security main restricted universe multiverse
deb https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src https://mirrors.ustc.edu.cn/ubuntu/ bionic-proposed main restricted universe multiverse
```

```bash
## 然后执行
sudo apt-get update
```

### 安装依赖

```bash
sudo apt-get install libprotobuf-dev  
sudo apt-get install libleveldb-dev  
sudo apt-get install libsnappy-dev  
sudo apt-get install libopencv-dev  
sudo apt-get install libhdf5-serial-dev  
sudo apt-get install protobuf-compiler  
sudo apt-get install libgflags-dev  
sudo apt-get install libgoogle-glog-dev  
sudo apt-get install liblmdb-dev  
sudo apt-get install libatlas-base-dev  
sudo apt-get install --no-install-recommends libboost-all-dev
```

### 编译源码

#### 下载源码

```bash
sudo git clone git://github.com/BVLC/caffe.git
```

#### 编译caffe

```bash
cd caffe
sudo cp Makefile.config.example Makefile.config
sudo vim Makefile.config
```

```sh
CPU_ONLY：=1 ##去掉“#”号
INCLUDE_DIRS := $(PYTHON_INCLUDE) /usr/include/hdf5/serial/ 
#替换掉 INCLUDE_DIRS := $(PYTHON_INCLUDE) /usr/local/include

hdf5_serial_hl hdf5_serial ## 替换掉 hdf5_hl hdf5
```

#### 执行编译

```bash
sudo make all -j8
sudo make test -j8
sudo make runtest
## 当出错重新编译时，需要执行sudo make clean
```

![caffe_make](https://raw.githubusercontent.com/liurio/deep_learning/master/img/caffe_make.png)

#### 问题解决

```sh
## 会报错 cv::imread(cv::String const&, int) 等....

## 解决：在Makefile文件中 LIBRARIES += glog gflags protobuf boost_system boost_filesystem m 后添加如下
LIBRARIES += glog gflags protobuf boost_system boost_filesystem m opencv_core opencv_highgui opencv_imgproc opencv_imgcodecs
```

### 编译python接口

```bash
sudo apt-get install python-pip  
sudo apt-get install python-numpy
sudo apt-get install gfortran
cd python
for req in $(cat requirements.txt); do pip install $req; done
sudo pip install ipython==5.3.0
sudo pip install -r requirements.txt
sudo vim ~/.bashrc

## 在 ~/.bashrc 文件的末尾添加
export PYTHONPATH=~/caffe/python:$PYTHONPATH

source ~/.bashrc

cd ../ ## caffe根目录
sudo make pycaffe
```

### 验证

```bash
python
## 在python中输入import caffe
import caffe
## 如果没报错，则安装成功
```

