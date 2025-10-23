## 一、下载
[github链接](https://github.com/opencv/opencv/releases)(自己选择版本，例如opencv-3.4.15)
## 二、编译
```shell
sudo apt-get install build-essential cmake git libgtk2.0-dev pkg-config libavcodec-dev libavformat-dev libswscale-dev
cd opencv-3.4.15
mkdir build && cd build
mkdir installed
cmake -D CMAKE_BUILD_TYPE=Release -D CMAKE_INSTALL_PREFIX=~/opencv-3.4.11/build/installed -D WITH_GTK=ON -D WITH_CUDA=OFF -D BUILD_DOCS=OFF -D BUILD_EXAMPLES=OFF -D BUILD_TESTS=OFF -D BUILD_PERF_TESTS=OFF ..
make -j8
make install
```
> ❗**注意：**`-DCMAKE_INSTALL_PREFIX=~/opencv-3.4.15/build/installed` 设置安装路径
> opencv3 的 `OpenCVConfig.cmake` 是在 `~/opencv-3.4.15/build/installed` 下，而 opencv4 的在 `~/opencv-4.5.2/build/installed/lib/cmake/opencv4`。
## 三、链接
CMake 配置：
```shell
set(OpenCV_DIR /home/ubuntu/opencv-3.4.15/build/installed/share/OpenCV)
find_package(OpenCV_DIR REQUIRED) 
message("OpenCV version  is ： ${OpenCV_VERSION}")
```
## 四、替代系统库（可选，但不建议）
```shell
export PKG_CONFIG_PATH=~/opencv-4.5.2/build/installed/lib/pkgconfig
export LD_LIBRARY_PATH=~/opencv-4.5.2/build/installed/lib
```
查看版本：
```shell
source ~/.bashrc 
pkg-config --modversion opencv
```
## 五、无法找到链接
```shell
sudo vim /etc/ld.so.conf.d/nankel.conf
```
然后将 OpenCV 的 `lib` 路径加入里面，例如：
```shell
~/opencv-3.4.11/build/installed/lib/
```
然后执行更新：
```shell
sudo ldconfig
```