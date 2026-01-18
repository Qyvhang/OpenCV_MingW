# OpenCV 4.11.0 MinGW-13.1.0预编译版本

这是使用MinGW-13.1.0编译的OpenCV 4.11.0完整版本，包含了opencv_contrib额外模块。

## 编译环境

- **操作系统**: Windows 11
- **编译器**: MinGW-13.1.0 (GCC 13.1.0)
- **CMake版本**: 3.30.0

## 特性与支持

- ✅ 包含完整的OpenCV 4.11.0核心库
- ✅ 包含所有opencv_contrib扩展模块
- ✅ 使用world模式编译(单一DLL文件)
- ❌ 不支持CUDA（Windows平台MinGW限制，仅MSVC支持）
- ✅ 启用OpenCL支持
- ✅ 支持C++开发
- ✅ 启用DirectX支持 (MinGW-13.10完全兼容)
- ❌ 不包含Python绑定
- ❌ 不包含Java绑定

## 主要功能模块

- 核心图像处理 (core, imgproc)
- 视频处理与摄像头支持 (videoio)
- 特征检测 (features2d, xfeatures2d)
- 深度学习支持 (dnn) 
- 目标跟踪 (tracking)
- 人脸识别 (face)
- 二维码识别 (wechat_qrcode)
- 图像拼接 (stitching)
- 增强现实 (aruco)
- ...以及更多

## 如何使用

1. **下载与安装**
   - 下载本仓库并解压
   - 将`x64/mingw/bin`目录添加到系统PATH环境变量

2. **在C++项目中使用**
   - 包含头文件路径: `include`
   - 链接库文件: `x64/mingw/lib`
   - 链接主要DLL: `libopencv_world4110.dll`

3. **CMake配置示例**
   ```cmake
   find_package(OpenCV REQUIRED PATHS "你的解压路径/install")
   target_include_directories(你的项目名 PRIVATE ${OpenCV_INCLUDE_DIRS})
   target_link_libraries(你的项目名 PRIVATE ${OpenCV_LIBS})
   ```

4. **MinGW编译命令示例**
   ```
   g++ -std=c++11 your_program.cpp -o your_program.exe -I"你的解压路径/install/include" -L"你的解压路径/install/x64/mingw/lib" -lopencv_world4110
   ```

## 兼容性说明

- 此版本为MinGW-13.10编译，不兼容MSVC编译的项目
- 需要使用MinGW-13.10或更高版本
- 相比MinGW-8.10版本，此版本兼容DirectX功能并提供更好的Windows API支持
- 如果遇到DLL错误，请确保系统PATH包含MinGW-13.10的bin目录

## 关于 CUDA 支持的说明

由于 OpenCV 的架构限制，**Windows 平台上的 CUDA 加速仅支持 MSVC 编译器**，MinGW 编译的版本无法启用 CUDA。

这是因为：
1. NVIDIA CUDA Toolkit 在 Windows 上只官方支持 MSVC 编译器
2. CUDA 运行时库与 MinGW 的 ABI 不兼容
3. OpenCV 源码中明确限制了 MinGW 下的 CUDA 支持

### GPU 加速替代方案

如果您需要 GPU 加速，有以下选择：

1. **使用 OpenCL**（推荐）
   - 本版本已启用 OpenCL 支持
   - 支持更广泛的硬件（AMD、Intel、NVIDIA）
   - 可在支持的设备上提供 GPU 加速
   - 对于大多数应用场景已足够

2. **使用 MSVC 版本**
   - 如需 CUDA 支持，请使用 Visual Studio 编译的 OpenCV 版本
   - CUDA 在 NVIDIA GPU 上通常性能更好

3. **使用 CPU 优化**
   - MinGW 版本仍包含完整的功能
   - CPU 性能经过优化

## 编译选项

本版本使用以下CMake选项编译:

```
cmake -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Release ^
    -DOPENCV_EXTRA_MODULES_PATH=../opencv_contrib/modules ^
    -DWITH_OPENCL=ON ^
    -DWITH_QT=ON ^
    -DWITH_GTK=ON ^
    -DWITH_OPENGL=ON ^
    -DWITH_FFMPEG=ON ^
    -DWITH_GSTREAMER=ON ^
    -DBUILD_TESTS=ON ^
    -DBUILD_EXAMPLES=ON ^
    -DBUILD_opencv_world=ON ^
    -DWITH_DIRECTX=ON ^
    ../opencv
```

## 许可证

- OpenCV使用[3-clause BSD许可证](https://opencv.org/license/)
- 此预编译版本遵循原始OpenCV项目的授权条款

## 免责声明

本预编译版本仅供开发使用，不得用于商业用途。作者不对因使用此库引起的任何问题负责。