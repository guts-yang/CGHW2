# 三角形栅格化与超采样反走样实现

这是一个基于计算机图形学原理实现的三角形栅格化渲染器，支持2x2超采样反走样功能。

## 功能实现

### 核心功能
- 三角形栅格化渲染
- 2x2超采样反走样（Super-sampling Anti-aliasing）
- 深度缓冲（Z-buffer）可见性判断
- MVP变换矩阵（模型、视图、透视投影）
<img width="855" height="535" alt="image" src="https://github.com/user-attachments/assets/19715793-03a1-4916-ae6a-9558bcd32fa8" />

### 算法特性
- 基于bounding box的高效像素遍历
- 向量叉积法判断点是否在三角形内
- 重心坐标插值计算颜色和深度
- 超采样样本平均化处理边缘锯齿

## 项目结构

```
├── HW2/                  # 源代码目录
│   ├── main.cpp          # 主程序入口
│   ├── rasterizer.cpp    # 光栅化器实现
│   ├── rasterizer.hpp    # 光栅化器头文件
│   ├── Triangle.cpp      # 三角形类实现
│   ├── Triangle.hpp      # 三角形类头文件
│   └── global.hpp        # 全局定义
├── include/              # 依赖库头文件
│   ├── Eigen/            # Eigen数学库
│   └── opencv2/          # OpenCV库
├── lib/                  # 库文件
│   └── opencv_world4120.lib  # OpenCV库文件
└── bin/                  # 可执行文件和运行时库
    └── opencv_world4120.dll  # OpenCV运行时库
```

## 编译指南

### 前提条件
- C++17兼容的编译器（推荐GCC 8+或MSVC 2019+）
- Eigen库（项目中已包含）
- OpenCV 4.1.2库（项目中已包含）

### 使用Visual Studio编译
1. 打开`HW.sln`解决方案文件
2. 设置为Release配置和x64平台
3. 编译项目

### 使用命令行编译
```bash
# Windows环境下使用MinGW
cd d:/coding/202330451061HW2
g++ -std=c++17 -I./include -I./include/opencv2 -I./include/Eigen ./HW2/main.cpp ./HW2/rasterizer.cpp ./HW2/Triangle.cpp -o ./build/HW2.exe -L./lib -lopencv_world4120 -lgdi32
```

## 运行说明
1. 确保`opencv_world4120.dll`与可执行文件在同一目录下
2. 运行编译生成的`HW2.exe`文件
3. 程序会显示一个包含两个彩色三角形的窗口
4. 按ESC键退出程序

## 算法说明

### 三角形栅格化流程
1. 计算三角形的bounding box，确定需要遍历的像素范围
2. 遍历bounding box内的每个像素
3. 对每个像素执行2x2超采样，生成4个采样点
4. 判断每个采样点是否在三角形内
5. 对内部采样点计算重心坐标和插值深度
6. 执行深度测试，更新样本颜色和深度缓冲区
7. 最后，将所有样本平均化，得到最终像素颜色

### 超采样反走样原理
超采样通过在每个像素内生成多个采样点并对结果进行平均，可以有效减少锯齿边缘，提高图像质量。本实现中采用2x2超采样，即在每个像素内均匀分布4个采样点，计算每个采样点的颜色后取平均值作为最终像素颜色。

## 关键代码说明

### 1. 超采样实现
在`rasterizer.hpp`中定义了超采样缓冲区：
```cpp
std::vector<std::vector<Eigen::Vector3f>> sample_color_buf;  // 存储每个像素的2x2样本颜色
std::vector<std::vector<float>> sample_depth_buf;  // 存储每个像素的2x2样本深度
const int sample_rate = 2;  // 采样率
```

### 2. 三角形内点判断
使用向量叉积法判断点是否在三角形内：
```cpp
static bool insideTriangle(int x, int y, const Vector3f* _v)
{
    // 计算点到三角形边的向量叉积，判断符号是否一致
    float cross1 = ab.x() * ap.y() - ab.y() * ap.x();
    float cross2 = bc.x() * bp.y() - bc.y() * bp.x();
    float cross3 = ca.x() * cp.y() - ca.y() * cp.x();
    
    // 所有叉积符号相同，则点在三角形内
    return (cross1 >= 0 && cross2 >= 0 && cross3 >= 0) ||
        (cross1 <= 0 && cross2 <= 0 && cross3 <= 0);
}
```

### 3. 样本解析函数
将超采样样本平均化到帧缓冲区：
```cpp
void rst::rasterizer::resolve_to_frame_buffer() {
    // 对每个像素，计算其sample_rate x sample_rate个样本的平均颜色
    for (int x = 0; x < width; ++x) {
        for (int y = 0; y < height; ++y) {
            // 计算所有样本的平均颜色
            Eigen::Vector3f avg_color(0, 0, 0);
            int valid_samples = 0;
            
            // 遍历所有样本点
            for (int sx = 0; sx < sample_rate; ++sx) {
                for (int sy = 0; sy < sample_rate; ++sy) {
                    // 只考虑深度测试通过的样本
                    if (sample_depth_buf[pixel_index][sample_index] < std::numeric_limits<float>::infinity()) {
                        avg_color += sample_color_buf[pixel_index][sample_index];
                        valid_samples++;
                    }
                }
            }
            
            // 计算平均颜色
            if (valid_samples > 0) {
                avg_color /= valid_samples;
                frame_buf[pixel_index] = avg_color;
            }
        }
    }
}
```

## 注意事项
- 程序使用的是右手坐标系
- 透视投影矩阵已正确实现，符合计算机图形学标准
- 模型矩阵设置为单位矩阵，不进行额外的模型变换
- 深度值范围为[0.1, 50]，符合OpenGL等图形API的常见设置

## 依赖库
- [Eigen](http://eigen.tuxfamily.org/)：用于矩阵运算
- [OpenCV](https://opencv.org/)：用于图像显示和保存
