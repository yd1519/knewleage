# 计算机图形学秋招复习文档（九）：HDR 与后处理

**复习范围：** HDR Rendering、Scene-Referred Linear RGB、Radiance / Luminance、浮点渲染目标、Exposure / EV / Exposure Compensation、自动曝光与 Metering、Log-Luminance Histogram、Temporal Adaptation、Pre-Exposure、Tone Mapping、Reinhard / ACES Fitted、Bloom、Color Grading、3D LUT、sRGB / Display Output、HDR Display (PQ/HLG)、后处理 Pass 组织、RenderGraph、OpenGL/GLSL、GPU 性能与 RenderDoc 调试。

**面试目标：** 能从场景 Radiance 解释 HDR 及动态范围，而不是只说“颜色超过 1”；能推导并实现基于平均 Log Luminance 或直方图的自动曝光；能区分曝光、Tone Mapping、Gamma/OETF、Color Grading 与 HDR 显示编码；理解 TAA History 的 Pre-Exposure 一致性；能在 OpenGLRenderer 中设计完整可调试、可扩展的后处理管线。

**前置知识：** [04-纹理与采样](04-纹理与采样.md)（sRGB 与数据纹理）、[05-光照模型与渲染方程](05-光照模型与渲染方程.md)（Radiance、Irradiance）、[06-PBR与IBL](06-PBR与IBL.md)（HDR Environment）、[08-抗锯齿技术](08-抗锯齿技术.md)（TAA、History 与 Pre-Exposure）。

**下一篇预告：** 10-全局光照与光线追踪（待编写）。

**排版约定：** 简短变量/公式用行内代码；独立推导使用 GitHub 支持的块级公式；Markdown 表格中不使用需要额外宏包的数学表达式。示例使用 OpenGL 4.5 / GLSL 450 Core、C++17/GLM；除特别说明外，默认使用 Linear RGB 和浮点 HDR Render Targets。

**重要统一约定：**

- `C_scene` 表示光照阶段得到的**线性场景 RGB**。它的绝对物理单位是否等于 W/(m²·sr) 取决于引擎的光度标定；我们不把任意 Shader RGB 数字冒充真实 Radiance。
- `E` 表示曝光乘法系数，`C_exposed=E*C_scene`。曝光加一挡（+1 EV 的**曝光补偿**）在本文示例中意味着 `E` 乘 2；而**相机曝光值 EV100 增加 1**（光圈/快门/ISO 意义）通常代表减少一挡进光，方向相反，须分清术语。
- `T` 表示 Tone Mapping 或相应的 Tone/Display Mapping 处理。示意流程 `C_display=T(E*C_scene)`，但真实 Color Pipeline 还要考虑 Gamut、白点、Output Transfer Function (OETF/EOTF)、Dithering 等。
- sRGB 示例假设线性 RGB 使用 **sRGB/Rec.709 色度原色和 D65 白点**；`0.2126/0.7152/0.0722` 是这一前提下的线性亮度权重，并非所有 RGB 工作色域的通用系数。
- `Y` 表示从线性 RGB 计算的相对亮度（Relative Luminance）；它和引擎美术强度单位、显示器绝对亮度 `cd/m²` 不是可以随意互换的。
- HDR 的“场景高动态范围”与 HDR10/PQ 等**显示传输标准**是不同层次。HDR Scene Buffer 很亮，不代表一定通过 HDR 显示接口输出。
- Bloom、Exposure、Tone Mapping、TAA 和 Color Grading 的顺序依管线设计而不同。本文会给出一个自洽的教学方案，并说明哪些环节可以变体，不宣称只有一种合法顺序。

---

## 一、HDR 是什么？为什么实时渲染需要它？

### 1. HDR（High Dynamic Range）的真正含义是什么？

动态范围描述信号中最暗和最亮的可表达或可观察量之间的比例。高动态范围渲染希望在 Lighting 和一部分后处理中保留远大于普通显示编码 `[0,1]` 能表达的场景亮度比例，而不是过早裁剪。

**HDR 不等于“浮点颜色大于 1 就算物理正确”**。是否准确还依赖光源与材质单位、颜色空间、曝光、Tone Mapping 以及输出设备。

### 2. 为什么场景颜色允许超过 1？

在 Linear Lighting 中：

- 多盏灯的贡献可以累加。
- 太阳、发光体、镜面高光与室内环境亮度可能相差极大。
- 将 `C_scene` Clamp 到 `[0,1]` 会过早丢失强光的相对差异。
- 后续曝光、Bloom、Tone Mapping 仍需要这些信息。

若光照过程中将 `(3,2,1)` 与 `(100,50,10)` 都 Clamp 成近似白色，后续再调低曝光也无法恢复它们之间原本的强度差。

### 3. LDR 和 HDR 的区别只是纹理格式吗？

不是。LDR/HDR 描述信号范围和编码；纹理格式决定如何储存它们，二者有关但并不等价。

| 概念 | 常见示例 | 关键问题 |
|---|---|---|
| LDR 颜色目标 | RGBA8_UNORM / sRGB8_ALPHA8 | 有限整数编码，通常不能无损保留 HDR 高亮 |
| HDR 颜色目标 | RGBA16F、RGBA32F 等 | 容纳较宽浮点数值范围；精度/带宽不同 |
| HDR 环境源 | EXR、HDR/RGBE 等 | 可以保存高动态范围照明 |
| Scene-Referred 数据 | Linear HDR Scene Color | 描述相对场景照明结果，等待显示映射 |
| Display-Referred 数据 | Tone/Display-Mapped RGB | 为特定显示亮度和色域做输出准备 |

### 4. 为什么不能直接在 Shader 中先 Clamp 再 Bloom？

Bloom 依赖高亮区域及其空间扩散，过早裁剪会丢掉高亮强度差异，导致强太阳与普通白墙难以区分。

即使 Bloom 可在某些特殊管线里使用已映射颜色实现，也必须明确 Threshold 和亮度输入范围；不能无条件把 LDR 后处理的阈值当作 HDR 物理亮度阈值。

### 5. HDR Render Target 的优点有哪些？

- 支持大量灯光叠加后的较大数值。
- 保留镜面高光、发光和环境亮度变化。
- 支持曝光调整而不必重新执行全部 Lighting。
- 可以作为 TAA、Bloom、Lens Effects、Tone Mapping 的共同输入。
- 适合现代线性光照与 PBR 的处理链。

它的代价包括显存、带宽、采样/混合支持和不同格式的范围/精度限制。

---

## 二、Linear RGB、sRGB 与光照计算

### 1. 为什么光照必须在 Linear RGB 中计算？

BRDF、光线叠加、透射与曝光近似按照线性物理量组合。如果直接对经过非线性显示编码的 sRGB 数字求和，通常不再对应正确的能量叠加。

例如在线性空间将两个强度 `0.25` 相加得到 `0.5`。如果先将它们编码成 sRGB 再相加，结果不能等价于对线性能量求和后编码。

### 2. sRGB 的编码是否等于直接做 Gamma 1/2.2？

不是严格相同。标准 sRGB OETF/编码具有近黑部分的线性段和非线性幂函数段。

若线性通道 `L∈[0,1]`，sRGB 编码为：

$$
\boxed{
S(L)=
\begin{cases}
12.92L, & L\le0.0031308,\\
1.055L^{1/2.4}-0.055, & L>0.0031308.
\end{cases}
}
$$

反向 sRGB Decode：

$$
\boxed{
L(S)=
\begin{cases}
S/12.92, & S\le0.04045,\\
\left(\frac{S+0.055}{1.055}\right)^{2.4}, & S>0.04045.
\end{cases}
}
$$

Gamma 2.2 是常见口语近似，不能完全替代标准 Piecewise sRGB 转换。上述函数定义的是单位化 SDR 颜色通道编码，不是 HDR10 PQ。

### 3. 为什么 BaseColor 纹理和 Roughness 纹理的 sRGB 设置不同？

BaseColor 通常存美术创作的 sRGB 编码颜色，参与 PBR 运算前需转换到 Linear。Roughness、Metallic、AO、Normal 属于数值/几何数据；对它们做 sRGB Decode 会扭曲原本的数据意义。

具体导入路径需检查 Texture SRGB Flag、Shader Sample 结果和资源格式；不能看一张纹理“有 RGB 通道”就推断其是颜色图。

### 4. OpenGL 的 sRGB Texture 是否会自动 Decode？

当 Texture Internal Format 是 `GL_SRGB8` 或 `GL_SRGB8_ALPHA8` 等支持 sRGB 采样解码的格式时，通常纹理采样会把颜色通道自动转换到线性数值，Alpha 通道不执行相同的颜色解码。

是否进行 Framebuffer Output sRGB Encoding 则是另一个阶段，受到 Framebuffer 色彩格式和 `GL_FRAMEBUFFER_SRGB` 等状态影响。

### 5. GL_FRAMEBUFFER_SRGB 为什么容易造成双重 Gamma？

如果最终 Shader 已经手动将线性颜色执行 sRGB Encode，又将结果写入启用自动 sRGB Encoding 的合适 Framebuffer Attachment，就可能**重复编码**，画面变得异常发白或对比不正确。

反过来，若 Shader 输出 Linear 但最终普通 UNORM 目标没有执行正确的 Output Encoding，也会呈现错误亮度。

推荐为整个 Pipeline 选定**唯一明确**的 Output Encoding 阶段。

### 6. Color Space 和 Transfer Function 是一回事吗？

不是。

- Color Primaries/Gamut：定义 RGB 原色及可表达的色度范围，例如 sRGB/Rec.709、Display P3、Rec.2020。
- Transfer Function：定义信号数值如何编码/解码，例如 sRGB 曲线、PQ、HLG。
- White Point：定义参考白点（如 D65）。
- Scene-Referred/Display-Referred：定义颜色信号对应场景还是最终显示。

即使两个信号都叫 RGB，它们也可能使用不同的 Primaries 和 Transfer，不能直接逐通道相加。

---

## 三、Luminance：平均亮度到底怎样算？

### 1. RGB 的平均值等于人眼感受到的亮度吗？

不等于。对**线性 sRGB/Rec.709 RGB**，常见相对亮度计算：

$$
\boxed{
Y=0.2126R+0.7152G+0.0722B.
}
$$

其中绿色系数较大反映对应色域/视觉加权下的贡献。该式不是对任意 Wide Gamut RGB 通用的矩阵行，也不是 CIE 光谱积分的直接替代。

### 2. 为什么先要从 sRGB 解码再计算 Y？

这些系数属于线性 RGB 到 XYZ 的转换关系。对编码后的 `R_s,G_s,B_s` 直接套线性亮度权重一般会给出错误的物理比例。

因此 HDR Scene Color Metering 通常在合适的线性 RGB 中进行；如果输入是另一色域，要先使用匹配其 Primaries 的转换或相应亮度系数。

### 3. 为什么自动曝光常取 Log-Luminance？

人眼对亮度的响应可以在很宽动态范围内感受相对比例，而曝光本身也以倍数（Stops）控制。

对正亮度 `Y>0`：

$$
\boxed{
\ell=\log_2 Y.
}
$$

取 Log 可把若干数量级的亮度范围转换成更适合统计和分桶的数值空间，还可避免少量极端亮点完全主导简单算术平均。

### 4. 为什么需要防止 log(0)？

Scene Color 中可能有真正的黑色背景或未覆盖像素；数学上 `log2(0)` 不存在，Shader 中可能产生负无穷或无效数值。

常见方案：

$$
\boxed{
\ell=\log_2(\max(Y,\varepsilon)).
}
$$

`epsilon` 是工程保护阈值。阈值选取会影响极暗区域是否被计入曝光统计；它不是无关紧要的数字。

### 5. 几何平均亮度与平均 Log-Luminance 有什么关系？

对 `N` 个正亮度值 `Y_i`：

$$
\frac1N\sum_{i=1}^{N}\log_2 Y_i
=\log_2\left(
\prod_{i=1}^{N}Y_i
\right)^{1/N}.
$$

因此：

$$
\boxed{
Y_{\mathrm{geo}}=
2^{\frac1N\sum_i \log_2 Y_i}.
}
$$

这是**几何平均**，不是算术平均 `(1/N)sum(Y_i)`。在 HDR 场景中两者可能相差很大。

### 6. 几何平均是否天然比算术平均更正确？

不是绝对。自动曝光还与测光区域、主体权重、Metering Mode、Histogram Clipping、Highlight Protection、美术意图有关。

几何平均对乘性亮度变化更自然，但若黑色区域非常多也可能把曝光拉得过亮；所以实际引擎常采用加权直方图、分位数或 Metering Mask。

---

## 四、Exposure：曝光和光照是什么关系？

### 1. 曝光在渲染器中做什么？

曝光把场景线性颜色统一乘一个比例，决定某个 Scene Luminance 经过 Tone Mapping 前落到什么亮度范围：

$$
\boxed{
C_{\mathrm{exposed}}=E\,C_{\mathrm{scene}}.
}
$$

简单曝光不是改变 PBR BRDF 或灯光真实能量，而是改变**虚拟相机如何把场景量映射到成像信号**。

### 2. Exposure Compensation 加一挡意味着什么？

本篇把曝光补偿 `EC` 定义为加亮的 Stops，因此：

$$
\boxed{
E_{\mathrm{adjusted}}=
E_{\mathrm{base}}\,2^{EC}.
}
$$

- `EC=+1`：曝光乘 2，画面通常更亮。
- `EC=-1`：曝光乘 0.5，画面通常更暗。
- `EC=+2`：曝光乘 4。

Tone Mapping 之后视觉明暗变化可能是非线性的，因此“曝光乘 2”不等于最终 sRGB 数字逐通道乘 2。

### 3. 什么是 EV100？

EV100 是在 ISO 100 标准化的曝光值表达。对给定光圈 f-number `N`、快门时间 `t`（秒）与 ISO：

$$
\boxed{
EV_{100}=
\log_2\left(
\frac{N^2}{t}\,\frac{100}{ISO}
\right).
}
$$

它描述相机曝光参数之间的关系；**EV100 越大通常意味着在同等场景亮度下曝光越少**。它与上段“正曝光补偿提高 E”的符号含义不同。

### 4. 为什么不能只用 EV100 就断言最终 HDR 像素值？

要从 EV100 和场景物理亮度得到数字相机的输出，还需要选择校准常数、参考反射率、ISO 模型、Metering/Exposure 策略和色彩管线。

不同引擎可能有 `1.2`、`78`、`K` 等来源不同的转换常量，它们是**特定标定/工作流的约定**，不能混成跨引擎唯一通用的物理公式。

### 5. 为什么曝光应作用于 Linear HDR Color？

线性乘法对应比例变化；若在 Tone Mapped/sRGB 编码颜色上直接乘以 2，已经经历非线性压缩，既可能剪裁也不再代表同样的场景曝光变化。

### 6. 为什么“把某盏灯亮度减半”和“全局曝光减半”不一样？

前者只改变该光源对场景某些部分的光照贡献；后者整体缩放场景 HDR 颜色，包括环境、发光、镜面高光等。

即便某些局部画面像素变暗程度类似，场景内各灯光的相对贡献、Bloom Threshold、暗部关系都会不同。

---

## 五、自动曝光（Auto Exposure）为什么需要 Metering？

### 1. 自动曝光解决什么问题？

从明亮室外走进暗室，如果曝光一直固定，暗室可能几乎全黑；从暗室看向太阳，画面可能过曝。

Auto Exposure 根据当前场景亮度统计决定目标曝光，让画面在变化的亮度条件下维持可用的观看范围。

### 2. 自动曝光最简单的目标曝光公式？

假设 `Y_{\mathrm{meter}}` 是经过合理测光的线性相对亮度，`K_{\mathrm{key}}>0` 是希望映射到 Tone Mapping 前的参考灰度/Key（例如教学常用的 0.18），则：

$$
\boxed{
E_{\mathrm{target}}=
\frac{K_{\mathrm{key}}}{
\max(Y_{\mathrm{meter}},\varepsilon)
}.
}
$$

对该简化模型，`Y_meter` 翻两倍时目标曝光减半。真实引擎可能使用另一个曝光校准、物理相机参数或 Tone Mapping 的中灰定位。

### 3. Middle Gray 为何常举 18%？

18% Gray 常作为摄影/图形学曝光教育中的参考中灰值，方便说明“平均场景亮度要落在什么相对位置”。

它不是所有渲染器必须使用的固定常数，也不是说最终显示 sRGB 灰度值一定等于 0.18；中灰在 Tone Mapping 后映射到哪一个显示值由整条输出曲线决定。

### 4. Basic Metering、Histogram Metering 有什么区别？

- **Basic/Mean Metering：** 对整个图像或指定区域取（加权）平均/Log 平均亮度。
- **Histogram Metering：** 对亮度分布做分桶，并根据分位数、剔除极端值、权重计算目标测光亮度。
- **Spot/Center-Weighted Metering：** 强调画面中心或主体区域。
- **Metering Mask：** 通过纹理/函数为不同屏幕位置赋权。

这些方法衡量的都是“给相机曝光看的亮度分布”，不是必须与图像最终 HDR 颜色统计完全一样。

### 5. 为什么整个屏幕的平均亮度可能不符合摄影意图？

如果画面绝大部分是黑暗天花板，只有中间一个角色，简单全屏平均可能把角色曝光过亮；如果窗户占了很大区域，平均亮度又可能压暗前景角色。

需要根据场景与镜头构图决定测光权重，例如 Center Weight、Spot、Player-Focused Mask、High/Low Percentile 等。

### 6. Metering Mask 的数学定义是什么？

对各样本亮度 `Y_i`、对应非负权重 `w_i`：

$$
\boxed{
\overline{\ell}_w=
\frac{\sum_i w_i\,\log_2(\max(Y_i,\varepsilon))}
{\sum_i w_i},
\qquad
Y_{\mathrm{meter}}=2^{\overline{\ell}_w}.
}
$$

要求总权重为正。权重可以来自图像中心距离、Mask Texture 或其它画面主体识别逻辑。

**注意：** 这种加权几何平均是一种明确的 Basic Metering 设计。Histogram 还可以结合亮度分位数截断和不同权重语义，并不都等价于上式。

### 7. 正确的权重 Histogram 与普通像素计数有什么区别？

普通直方图将每个有效 Sample 贡献 1 次 Count；若要按 Metering Mask 加权，必须让样本对桶贡献与 `w_i` 相称，或者先按接受概率重采样。

简单在 Build Histogram Pass 中忽略权重，而最后只将曝光目标乘某个中心权重常数，并不等同于空间加权测光。

---

## 六、HDR Log-Luminance Histogram：GPU 上怎样做？

### 1. 为什么用 Histogram 而不是逐像素 CPU 回读？

1080p/4K HDR Color 每帧产生大量像素。将整个 Color Buffer 同步回读 CPU，再逐像素统计，往往引入数据传输和 GPU/CPU Pipeline Stall。

GPU Compute 可在 GPU 上做部分统计，并将少量 Bucket/Exposure 数据留在 GPU。CPU 如要调试，也可采用异步小量 Readback，而不是同步读取整张 HDR Buffer。

### 2. 如何从 Log-Luminance 映射到 Histogram Bin？

预先设定 Log2 Luminance 范围 `[ell_min,ell_max]`，有 `B` 个 Bucket。对有效的亮度 `Y>0`：

$$
\ell=\log_2(\max(Y,\varepsilon)).
$$

$$
\boxed{
b=\mathrm{clamp}\left(
\left\lfloor
B\frac{\ell-\ell_{\min}}{\ell_{\max}-\ell_{\min}}
\right\rfloor,\ 0,\ B-1
\right).
}
$$

这相当于按**等宽 Log2 Luminance 档位**统计。真实实现可能将 0 号 Bin 专门保留给纯黑/无效 Sample；这种策略要与 Reduction Pass 保持一致。

### 3. Histogram Range 为什么不能任意取特别大？

当固定 B 个 Bin 时，Log-Luminance 区间越大，每个 Bin 代表的亮度范围越宽，低亮度细节的统计分辨率下降。

如果区间太窄，又会有大量像素被 Clamp 到第一/最后一桶，失去极暗/极亮变化信息。工程上可根据预期环境范围选择合理的 MinEV/MaxEV，并考虑动态调整直方图范围。

### 4. Histogram 的计算流程？

~~~text
Linear HDR Scene Color (sRGB/Rec709 primaries)
              |
              v
Compute luminance Y = dot(rgb, coefficients)
              |
              v
Log2(max(Y, epsilon))
              |
              v
Map to 0 .. BinCount-1
              |
              v
Per-Workgroup Shared Histogram
              |
              v
Merge to Global Histogram Buffer
              |
              v
Histogram Reduction / Percentile Metering
              |
              v
Target Exposure -> Temporal Adaptation
~~~

### 5. 为什么 Compute Shader 要用 Workgroup Shared Memory？

若所有 Pixel 都使用 Global AtomicAdd 写相同少量 Histogram Bins，热门 Bucket 可能有较大原子操作竞争。

可先让一个 Workgroup 使用 Shared Array 收集部分统计，再由少量线程合并到 Global Histogram Buffer，以减少全局 Atomic 争用。

### 6. 直方图是否可以直接取每个 Bin 的平均亮度？

若以 Histogram Bin 代表的 Log-Luminance 中点 `ell_b` 加权，所得的是近似平均 **Log-Luminance**：

$$
\overline{\ell}\approx
\frac{\sum_b h_b\,\ell_b}
{\sum_b h_b}.
$$

最后 `Y_meter=2^{meanLog}`。分桶过程已经丢失每个 Bin 内的精确亮度，结果是数值近似而非与逐像素全精度 Log Mean 完全相等。

### 7. 为什么 Percentile Clipping 常用于 Auto Exposure？

例如只选择累计直方图中的某个中间百分比区域，忽略极暗和极亮尾部，有机会减少“太阳一个亮点”或“画面大块黑色边框”对测光的影响。

**注意 Percentiles 对应排序后的亮度分布，而不是按值直接剪裁每个 RGB 通道。** 高/低 Percent 的区间设置是画质/摄影意图参数，不是无偏积分要求。

