# Knowledge Base

用于整理和积累 C++、计算机图形学与游戏引擎开发相关的学习文档、面试八股和复习笔记。

## 目录

### C++

| 文档 | 内容 |
|---|---|
| [01 - C++ 语言基础](C%2B%2B/01-%E8%AF%AD%E8%A8%80%E5%9F%BA%E7%A1%80.md) | 基本类型、作用域与存储期、指针、引用、const、数组、函数、类型转换、初始化、运算符与未定义行为 |
| [02 - 类与对象模型](C%2B%2B/02-%E7%B1%BB%E4%B8%8E%E5%AF%B9%E8%B1%A1%E6%A8%A1%E5%9E%8B.md) | 构造与析构、拷贝与移动、特殊成员函数、继承、多态、虚函数表、对象切片、虚继承、RTTI |
| [03 - 资源管理与异常](C%2B%2B/03-%E8%B5%84%E6%BA%90%E7%AE%A1%E7%90%86%E4%B8%8E%E5%BC%82%E5%B8%B8.md) | new/delete、RAII、智能指针、资源所有权、异常处理、异常安全、noexcept、Rule of Three/Five/Zero |
| [04 - 模板与泛型编程](C%2B%2B/04-%E6%A8%A1%E6%9D%BF%E4%B8%8E%E6%B3%9B%E5%9E%8B%E7%BC%96%E7%A8%8B.md) | 函数与类模板、模板推导、特化与偏特化、参数包、完美转发、SFINAE、类型萃取、CRTP、Concepts |
| [05 - STL 标准模板库](C%2B%2B/05-STL.md) | vector、string、deque、list、map、unordered_map、容器适配器、迭代器、算法、lambda、复杂度与迭代器失效 |
| [06 - 现代 C++ 特性](C%2B%2B/06-%E7%8E%B0%E4%BB%A3C%2B%2B.md) | 移动语义、auto/decltype、lambda、constexpr、结构化绑定、optional/variant/string_view、Concepts、Ranges、C++20 特性 |
| [07 - 编译、链接与程序运行机制](C%2B%2B/07-%E7%BC%96%E8%AF%91%E9%93%BE%E6%8E%A5%E4%B8%8E%E7%A8%8B%E5%BA%8F%E8%BF%90%E8%A1%8C%E6%9C%BA%E5%88%B6.md) | 编译四阶段、头文件、ODR、static/extern/inline、符号与链接、静态/动态库、ABI、程序内存布局、对齐与优化 |
| [08 - 多线程与内存模型](C%2B%2B/08-%E5%A4%9A%E7%BA%BF%E7%A8%8B%E4%B8%8E%E5%86%85%E5%AD%98%E6%A8%A1%E5%9E%8B.md) | std::thread、mutex、条件变量、死锁、atomic、CAS、内存序、happens-before、future/async、线程池、jthread |

### Graphics：计算机图形学

| 文档 | 内容 |
|---|---|
| [01 - 数学基础与空间变换](Graphics/01-%E6%95%B0%E5%AD%A6%E5%9F%BA%E7%A1%80%E4%B8%8E%E7%A9%BA%E9%97%B4%E5%8F%98%E6%8D%A2.md) | 向量与矩阵、齐次坐标、MVP、正交/透视投影、法线逆转置、NDC、深度精度、Reversed-Z、四元数、常用几何计算 |
| [02 - 渲染管线与光栅化](Graphics/02-%E6%B8%B2%E6%9F%93%E7%AE%A1%E7%BA%BF%E4%B8%8E%E5%85%89%E6%A0%85%E5%8C%96.md) | 顶点与图元处理、齐次裁剪、背面剔除、边函数、重心坐标、透视校正插值、深度、MSAA、软光栅实现及性能优化 |
| [03 - 深度测试与透明渲染](Graphics/03-%E6%B7%B1%E5%BA%A6%E6%B5%8B%E8%AF%95%E4%B8%8E%E9%80%8F%E6%98%8E%E6%B8%B2%E6%9F%93.md) | 深度精度与 Reversed-Z、Depth Bias、Early-Z/Hi-Z、Stencil、透明排序、Straight/Premultiplied Alpha、OIT、OpenGL/RenderDoc |
| [04 - 纹理与采样](Graphics/04-%E7%BA%B9%E7%90%86%E4%B8%8E%E9%87%87%E6%A0%B7.md) | UV/采样过滤、Mipmap/LOD/各向异性、sRGB 与数据贴图、BC/ASTC、Texture Source/GPU 资源、GPU Readback 与 Texture Streaming |
| [05 - 光照模型与渲染方程](Graphics/05-%E5%85%89%E7%85%A7%E6%A8%A1%E5%9E%8B%E4%B8%8E%E6%B8%B2%E6%9F%93%E6%96%B9%E7%A8%8B.md) | Radiometry、立体角、Radiance、Lambert/Phong/Blinn-Phong、BRDF/Rendering Equation、Monte Carlo、重要性采样、MIS 与 GLSL 实现 |
| [06 - PBR 与 IBL](Graphics/06-PBR%E4%B8%8EIBL.md) | 微表面 Cook–Torrance、Fresnel、GGX、Smith、Metallic/Roughness、Disney BRDF、Diffuse/Specular IBL、Split-Sum、BRDF LUT、GLSL 与 RenderDoc |
| [07 - 阴影技术](Graphics/07-%E9%98%B4%E5%BD%B1%E6%8A%80%E6%9C%AF.md) | Shadow Mapping、Depth/Slope/Normal Bias、PCF/PCSS、CSM、点光 Cubemap、VSM/EVSM、Virtual Shadow Maps、Contact/RT Shadow 与性能调试 |
| [08 - 抗锯齿技术](Graphics/08-%E6%8A%97%E9%94%AF%E9%BD%BF%E6%8A%80%E6%9C%AF.md) | Sampling/Aliasing、SSAA/MSAA/A2C、FXAA/SMAA、TAA Jitter/Velocity/History、Disocclusion/Ghosting、TAAU/Dynamic Resolution 与 RenderDoc |
| [09 - HDR 与后处理](Graphics/09-HDR%E4%B8%8E%E5%90%8E%E5%A4%84%E7%90%86.md) | Linear HDR/sRGB、Luminance、EV/自动曝光、Histogram/Metering Mask、Pre-Exposure、Reinhard/ACES Fitted、Bloom、Color Grading LUT、HDR10/PQ 与 RenderDoc |
| [10 - 全局光照与光线追踪](Graphics/10-%E5%85%A8%E5%B1%80%E5%85%89%E7%85%A7%E4%B8%8E%E5%85%89%E7%BA%BF%E8%BF%BD%E8%B8%AA.md) | GI、渲染方程、路径追踪、NEE/MIS、Russian Roulette、Ray–Triangle/AABB、BVH/SAH、TLAS/BLAS、DXR/Vulkan RT、DDGI、SSGI、ReSTIR、SVGF 与性能调试 |

### 后续规划

- **C++ 后续专题：** 面试代码题、标准库实现分析、C++ 性能与并发专题深化。
- **计算机图形学后续专题：** 当前 01–10 已完成；后续可深入光谱渲染、体积渲染、神经渲染与高级采样。
- **游戏引擎：** 引擎架构、资源管理、场景与实体系统、动画、物理、渲染、工具链与性能优化。

## 复习方式

1. 先阅读对应主题文档。
2. 阅读完成后，通过选择题、代码分析题、纠错题和模拟面试追问检验掌握程度。
3. 根据答题情况补充薄弱知识点，并持续更新文档。
