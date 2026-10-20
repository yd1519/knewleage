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


---

## 七、GLSL Compute：构建 256-Bin HDR Histogram

### 1. 这段 Compute 示例采用什么输入和单位约定？

- 输入为已经光照完成的**线性 HDR Scene Color**，其 RGB Primaries 为 sRGB/Rec.709。
- 输入纹理使用 `RGBA16F`，像素颜色允许大于 1。
- `uMinLog2Y` 和 `uMaxLog2Y` 表示统计区间的 Log2 亮度坐标。
- 所有 Pixel 均计入 Histogram（含暗像素，经 Epsilon 保护）；暂不加入 Mask 与 Percentile。
- 全局 Histogram 使用绑定为 SSBO 的 256 个 `uint` Bin，**每帧必须在 Dispatch 前清零**。

### 2. 一个可验证的 Histogram Compute Shader

~~~glsl
#version 450 core

layout(local_size_x = 16, local_size_y = 16, local_size_z = 1) in;

layout(binding = 0, rgba16f) uniform readonly image2D uSceneHDR;

layout(std430, binding = 1) buffer HistogramBuffer
{
    uint bins[256];
};

uniform float uMinLog2Y;
uniform float uMaxLog2Y;

shared uint sharedBins[256];

void main()
{
    uint lane = gl_LocalInvocationIndex;
    sharedBins[lane] = 0u;
    barrier();

    ivec2 pixel = ivec2(gl_GlobalInvocationID.xy);
    ivec2 sizePx = imageSize(uSceneHDR);

    if (pixel.x < sizePx.x && pixel.y < sizePx.y)
    {
        vec3 rgb = max(imageLoad(uSceneHDR, pixel).rgb, vec3(0.0));
        float Y = dot(rgb, vec3(0.2126, 0.7152, 0.0722));

        // Production input should reject/repair NaN and Inf upstream.
        float logY = log2(max(Y, 1e-6));
        float span = max(uMaxLog2Y - uMinLog2Y, 1e-6);

        float normalized = (logY - uMinLog2Y) / span;
        uint bin = uint(clamp(
            floor(normalized * 256.0), 0.0, 255.0));

        atomicAdd(sharedBins[bin], 1u);
    }

    barrier();
    atomicAdd(bins[lane], sharedBins[lane]);
}
~~~

这里 `16×16=256` 个 Workgroup Threads 与 Bin Count 相等，因此每个 Thread 负责初始化和合并一个 Bin。若改为其它 Workgroup Size，必须修改 Initialization / Merge Loop，不能直接沿用该实现。

### 3. 为什么边缘 Workgroup 不能在第一个 barrier 前提前 return？

例如贴图不是 16 的整数倍时，某些线程的 GlobalInvocationID 越界。若仅部分 Workgroup Threads 在执行 `barrier()` 前 `return`，其余线程可能永远等待 Barrier 或触发未定义执行行为。

因此示例中只对实际像素处理做条件判断，但**所有 Workgroup Invocations 都执行两个 barrier**。

### 4. Shared Histogram 和 Global Histogram 为什么都需要 AtomicAdd？

- 同一 Workgroup 内多个像素可能落入同一个 Bin，更新共享 Bin 需要原子加。
- 多个 Workgroups 也可能写同一个 Global Bin，合并共享统计时仍需全局原子加。

如果不做同步/原子更新，最终 Count 可能随机丢失，曝光在帧间出现不稳定变化。

### 5. OpenGL 侧应该怎样调度和同步？

~~~cpp
// Requirements:
//  - histogramSSBO has at least 256 * sizeof(uint32_t) bytes.
//  - hdrTexture is a complete GL_RGBA16F texture.
//  - histogramProgram matches the GLSL buffer/image bindings.

const GLuint zero = 0;
glClearNamedBufferData(
    histogramSSBO, GL_R32UI, GL_RED_INTEGER, GL_UNSIGNED_INT, &zero);

glUseProgram(histogramProgram);
glBindBufferBase(GL_SHADER_STORAGE_BUFFER, 1, histogramSSBO);

glBindImageTexture(
    0, hdrTexture, 0, GL_FALSE, 0, GL_READ_ONLY, GL_RGBA16F);

GLuint groupX = (renderWidth + 15u) / 16u;
GLuint groupY = (renderHeight + 15u) / 16u;

glDispatchCompute(groupX, groupY, 1u);

// Histogram Reduction reads the SSBO written by this pass.
glMemoryBarrier(GL_SHADER_STORAGE_BARRIER_BIT);
~~~

其他依赖需依完整 GPU Command Stream 决定：例如此前对 HDR Texture 的 Render Pass 写入、后续 Histogram 输出采样和最终曝光 Uniform/SSBO 使用，都要满足相应 API 的资源可见性要求。

### 6. 为什么 GPU Histogram 不一定要跑全分辨率？

自动曝光一般不需要逐像素精确统计每个 HDR Sample。可以先将 HDR Scene Color 低通/下采样，再在小图上构建 Histogram，以减少 Atomics 和 Texture/Memory 访问。

但采样图的空间覆盖与 Weight 必须代表原场景的 Metering 意图；若低分辨率前置采样不合理，极亮小灯光或局部主体可能丢失，导致曝光偏差。

---

## 八、Histogram Reduction、Percentile 与 Metering Mask

### 1. 怎样从 256 个 Bin 得到一个测光亮度？

对统计 `h_b` 和每 Bin 代表的 Log2 中心值 `ell_b`：

$$
\ell_b=\ell_{\min}
+\left(b+\frac12\right)
\frac{\ell_{\max}-\ell_{\min}}{B}.
$$

$$
\boxed{
Y_{\mathrm{meter}}\approx
2^{
\frac{\sum_b h_b \ell_b}
{\sum_b h_b}
}.
}
$$

这是依据桶中点近似的 Log-Mean。边界 Clamping 会让超出统计范围的亮度失真。

### 2. Low/High Percentile Clipping 应怎样理解？

例如选择累计像素统计质量的 5% 到 95% 区间：

1. 从暗到亮扫描 Histogram，计算 Cumulative Count。
2. 排除最暗 5% 和最亮 5% 的统计质量。
3. 对剩余 Bin（含部分边界 Bin）计算加权 Log2 均值。
4. 将其转换回线性 Metered Luminance。
5. 根据 Key 和曝光补偿计算 Target Exposure。

分位数的 **5%/95%** 是示例，不是引擎或相机必须采用的统一标准；不同摄影意图可能更强调高光保护或暗部细节。

### 3. 一个易于验证的 CPU 参考 Reduction

~~~cpp
#include <array>
#include <algorithm>
#include <cmath>
#include <cstdint>

float MeterLogLuminanceFromHistogram(
    const std::array<std::uint32_t, 256>& bins,
    float minLog2, float maxLog2,
    float lowPercent, float highPercent)
{
    double total = 0.0;
    for (std::uint32_t n : bins)
        total += static_cast<double>(n);

    if (total <= 0.0)
        return 0.0f; // Example fallback: log2(1.0)

    lowPercent = std::clamp(lowPercent, 0.0f, 1.0f);
    highPercent = std::clamp(highPercent, lowPercent, 1.0f);

    const double lowRank = total * lowPercent;
    const double highRank = total * highPercent;

    double cursor = 0.0;
    double weightSum = 0.0;
    double logWeightedSum = 0.0;

    for (int b = 0; b < 256; ++b)
    {
        const double next = cursor + bins[b];
        const double selected =
            std::max(0.0, std::min(next, highRank) -
                              std::max(cursor, lowRank));

        const double logCenter =
            minLog2 + (static_cast<double>(b) + 0.5) *
            (maxLog2 - minLog2) / 256.0;

        weightSum += selected;
        logWeightedSum += selected * logCenter;
        cursor = next;
    }

    return weightSum > 0.0
        ? static_cast<float>(logWeightedSum / weightSum)
        : 0.0f;
}
~~~

**注意：** 这是 CPU Reference Test，便于用已知 Histogram 对照 GPU Reduction；实际实时环境通常会在 GPU 上完成该计算，避免每帧同步读取 SSBO。

### 4. 为什么边界 Bin 应允许部分权重？

如果将低于某 Rank 的 Bin 整桶丢弃，尤其 Histogram Bins 较少时，Percentile Threshold 的小幅变化可能导致大量样本突然被加入/排除，产生曝光跳变。

在边界 Bin 中按 Count 的一部分参与权重，通常更平滑，但依旧是“桶内亮度均匀/代表值”的数值近似。

### 5. Metering Mask 应当在哪一步参与？

若希望 Mask 影响直方图的空间权重：

- Histogram Build 阶段：对 Pixel 赋权并累加到某 Bin，可能使用固定点权重 Atomic 或特殊归并。
- 或 Precompute Weighted Luminance 统计，再做对应归一化。
- 若只需要中心/点测光，也可直接在空间样本选择上施加权重。

不应只在 Histogram Reduction 后简单乘一个像素位置权重并声称等价于加权统计。

### 6. 如何在 HDR 亮度中防止 Histogram 被火花和太阳干扰？

可以组合高低分位数、中心/主体 Metering Mask、Highlight/Shadow Bias、时间稳定性策略。

但把最亮像素无条件忽略也有代价：例如真正应对强背光时，曝光可能不愿降低，导致大面积高光过曝。Measure Policy 是摄影与美术决策，不能只用“去除离群值”概括。

### 7. 为什么有些引擎的 Histogram 0 Bin 代表 Black？

这是**特定实现约定**：将 `Y≤epsilon` 的样本放入特殊 Bin，统计时排除或单独处理，避免黑色边框/天空无数据把 Log Mean 拉得太暗。

本篇第七节示例没有 Black-Special Bin，而是所有输入 Clamp 后直接映射；**不要将那段示例与另一个“忽略零桶” Reduction 直接拼接**，否则会错误跳过真正的暗部样本。

---

## 九、时间适应：为什么曝光不能每帧立即跳到目标？

### 1. Eye Adaptation / Exposure Adaptation 的目标是什么？

相机从暗室转到室外时，自动曝光如果立即改变许多 Stops，画面会突然跳亮/跳暗。时间适应让曝光在一定时间内平滑靠近目标，产生更稳定的视觉和摄影效果。

### 2. 一个与帧率相对独立的指数平滑公式

设当前曝光 `E_t`、目标 `E_target`、本帧时间步长 `Delta t`、适应速率 `s>0`：

$$
\boxed{
a=1-e^{-s\Delta t},
\qquad
E_{t+1}=E_t+a(E_{\mathrm{target}}-E_t).
}
$$

`a` 随 Frame Delta Time 调整；在目标不变且采样足够细时，能获得比固定每帧插值系数更一致的时间响应。

### 3. 为什么很多实现选择在 EV/Log2 曝光空间平滑？

曝光以倍数改变，常希望“每秒变化多少 Stops”更可控。设：

$$
e_t=\log_2(E_t),\qquad
e_{\mathrm{target}}=\log_2(E_{\mathrm{target}}).
$$

在 Log2 Space 平滑：

$$
\boxed{
e_{t+1}=e_t+
(1-e^{-s\Delta t})(e_{\mathrm{target}}-e_t),
\qquad
E_{t+1}=2^{e_{t+1}}.
}
$$

相比在线性 Exposure 值上插值，该方法对从 0.1→1 和 1→10 这类相同 Stops 跨度的变化更一致。

### 4. Brighten Speed 与 Darken Speed 应如何定义？

方向命名容易引起混淆，建议以**最终曝光系数 E 的变化**明确：

- `E_target>E_current`：画面需要加亮（常发生于进入较暗环境），选用 `speedBrighten`。
- `E_target<E_current`：画面需要变暗（常发生于进入亮环境），选用 `speedDarken`。

有的引擎以“真实场景亮度增加/减少”命名速度，术语方向可能反过来。**不要只根据变量名猜哪个 Speed 生效。**

### 5. 示例：平滑对亮度目标的变化

固定速率 `s=2/s`，`Delta t=0.5s`：

$$
a=1-e^{-1}\approx0.63212.
$$

线性空间从 `E=1` 朝 `E_target=4` 更新一帧：

$$
E_{\mathrm{next}}
=1+0.63212(4-1)
\approx2.89636.
$$

若使用 Log2 Space 插值，结果不一样；**比较数学结果时务必先确认平滑的是 Exposure 还是 Log Exposure**。

### 6. 一个 CPU 端曝光状态更新示例

~~~cpp
#include <algorithm>
#include <cmath>

struct ExposureState
{
    float exposure = 1.0f;
    bool valid = false;
};

void UpdateExposure(
    ExposureState& state,
    float targetExposure,
    float deltaTime,
    float speedBrighten,
    float speedDarken)
{
    targetExposure = std::max(targetExposure, 1e-6f);

    if (!state.valid)
    {
        state.exposure = targetExposure;
        state.valid = true;
        return;
    }

    deltaTime = std::max(deltaTime, 0.0f);
    float curStops = std::log2(std::max(state.exposure, 1e-6f));
    float targetStops = std::log2(targetExposure);

    float speed = targetStops > curStops
        ? speedBrighten : speedDarken;
    speed = std::max(speed, 0.0f);

    float factor = 1.0f - std::exp(-speed * deltaTime);
    state.exposure = std::exp2(
        curStops + factor * (targetStops - curStops));
}
~~~

CPU 示例依赖 `targetExposure` 由 GPU Histogram 或 Metering 系统得到；如果每帧同步 Readback 才能执行更新，可能引入 GPU Stall。实时系统通常考虑 GPU Exposure Buffer 或异步滞后一帧的更新。

### 7. 为什么 Camera Cut 时要考虑 Exposure History？

如果镜头突然从亮室切到完全不同的暗室，继续按之前的 Exposure History 慢慢适应可能是有意的电影化效果，也可能导致几秒看不清内容。

可以重置曝光到新目标、加快过渡或按镜头指令覆盖 Auto Exposure。选择取决于相机系统和艺术需求；**不应无条件假设所有 Camera Cut 必须立即重置曝光**。

---

## 十、Pre-Exposure：为什么引擎要提前缩放 HDR？

### 1. 什么是 Pre-Exposure？

在某些现代引擎中，Lighting/Scene HDR 阶段就使用一个上一帧或稳定预估得到的曝光相关系数缩放场景颜色，让 Buffer 数值范围更适于 Half-Float、时域过滤及后处理。

它和最终 Camera Exposure 的概念相关，但**Pre-Exposure 可能使用上一帧值/特定尺度定义**，并不自动等于“当前帧自动曝光目标”。

### 2. 为什么会引入 Pre-Exposure？

- 避免光照数值特别大时 `RGBA16F` 很早丢失精度或溢出。
- 在非常暗的场景中改善某些相对精度问题。
- 让 TAA/后处理在更合适的数值尺度工作。
- 将曝光变化与多个 Render Pass 统一管理。

Pre-Exposure 不是让低精度格式 magically 具备无限动态范围；其策略和资源格式必须一起设计。

### 3. Pre-Exposure 和真正的曝光可写成什么关系？

教学上先设场景线性颜色 `C_scene`，Pre-Exposure 缩放 `P_t>0`：

$$
\boxed{
C_t^{\mathrm{stored}}=P_t C_t^{\mathrm{scene}}.
}
$$

假设最终想使用相机曝光 `E_t`：

$$
\boxed{
C_t^{\mathrm{exposed}}=
\frac{E_t}{P_t}C_t^{\mathrm{stored}}.
}
$$

这两个式子自洽，表达了“Buffer 中存储值”与“最终曝光意义”之间的尺度转换。具体引擎可能把 `P` 定义为曝光倒数或采用其它方式，必须以真实约定为准。

### 4. 为什么 TAA History 要做 Pre-Exposure Ratio 转换？

假设本篇保存的上一帧 History 为：

$$
H_{t-1}^{\mathrm{stored}}
=P_{t-1}C_{t-1}^{\mathrm{scene}}.
$$

现在要在当前帧 `P_t` 的存储尺度下混合历史，就必须：

$$
\boxed{
H_{t-1}^{\mathrm{currentScale}}
=
H_{t-1}^{\mathrm{stored}}
\frac{P_t}{P_{t-1}}.
}
$$

如果忽略这个比例，曝光或 Pre-Exposure 快速变化时，即使画面光照不变，TAA History 也可能突然变亮/变暗、拖尾或闪烁。

### 5. 为什么 Metering 不能把 Pre-Exposed RGB 直接当作 Scene Brightness？

如果 `C_stored=P_t*C_scene`，则存储颜色亮度也被 `P_t` 缩放。若 Histogram 想得到真正的**Scene-Linear Luminance**，应先除以当前 `P_t`（或等价在 Log2 Space 减去 `log2(P_t)`）。

否则曝光统计反馈的对象已经包含旧曝光缩放，可能引入不希望的动态耦合，甚至使曝光行为非常难以解释。

### 6. TAA 和 Bloom 应使用哪种 Exposure Space？

必须在资源元数据上标明：

- 当前 HDR Scene Color 是否 Pre-Exposed。
- 上一帧 History 采用哪一帧的缩放。
- Bloom Threshold 使用 Scene Units、Pre-Exposed Units 还是 Exposed Units。
- Tone Mapper 输入的 Exposure Multiplier 是否已补偿 Pre-Exposure。

一致的定义比特定数值约定更重要。即使画面大致看上去正确，不一致的尺度也会导致随曝光变化的伪影。

---

## 十一、Tone Mapping：HDR 如何映射到可显示范围？

### 1. 为什么有了曝光还需要 Tone Mapping？

曝光只是把整体场景亮度乘一个标量，不会减少无限/很宽的动态范围。

Tone Mapping 将场景或曝光后的高范围信号通过非线性映射放入目标显示范围，尽量保留暗部、灰度、中间调和高光的视觉层次。

### 2. Tone Mapping 与 Gamma/sRGB Encoding 的区别？

- **Tone Mapping：** 重新分配动态范围和对比度，决定 Highlight Roll-Off、Midtone、Black/White Behavior。
- **Output Encoding：** 将已经面向目标显示的线性颜色编码到相应的 Transfer Function，例如 SDR sRGB。
- **Gamut Mapping：** 将色度控制在显示色域内，避免不可表示颜色出现不受控裁剪。

三者可能在某些 Color Pipeline 实现中融合，但概念上不能等同。

### 3. 最简单的 Reinhard Tone Mapping 公式

对非负线性曝光颜色 `x≥0`：

$$
\boxed{
T_{\mathrm{Reinhard}}(x)=\frac{x}{1+x}.
}
$$

当 `x=0` 时输出 0；`x→∞` 时趋近 1。可以逐 RGB 通道使用，也可以围绕 Luminance 构建不同变体。

### 4. 为什么 Reinhard 有时会让画面显得发灰？

简单 `x/(1+x)` 在较低值也进行压缩，亮部较柔和，可能缺乏某些影视化曲线的对比度与肩部风格。逐通道映射还可能改变高亮饱和颜色的色相/饱和度。

它易于理解，不一定是现代游戏最佳艺术选择。

### 5. 曝光两倍时 Reinhard 输出也两倍吗？

不是：

$$
T(1)=\frac12=0.5,\qquad
T(2)=\frac23\approx0.6667.
$$

曝光从 1 倍到 2 倍，输入翻倍，Tone-Mapped 结果从 0.5 到约 0.6667，而不是 1.0。

### 6. 什么是 Tone Curve 的 Toe、Linear Section 与 Shoulder？

- **Toe：** 暗部压缩/抬升方式，影响 Black Crush 与 Shadow Detail。
- **Midtone/近线性区：** 决定中间调对比度、灰度定位。
- **Shoulder：** 亮部平滑压缩到显示上限，降低生硬的 Highlight Clipping。

实际 Filmic Curve 会结合色彩、饱和度和白点策略；Toe/Shoulder 只是解释其亮度曲线的常用术语。

### 7. 什么是 Extended / White-Point Reinhard？

一种常见的白点参数化曲线：

$$
\boxed{
T(x)=
\frac{x\left(1+\frac{x}{W^2}\right)}{1+x}.
}
$$

其中 `W>0` 是选定 White Point，`T(W)=1`。**该函数对 `x>W` 可能输出大于 1**，最终仍需按照目标显示编码处理；不要错误宣称它对任意非负输入自动被界定在 [0,1]。

---

## 十二、Filmic / ACES Fitted 与色彩保真

### 1. 什么是 ACES？

ACES（Academy Color Encoding System）是一套涉及场景颜色编码、色彩转换、参考 Rendering Transform、输出设备变换等内容的色彩管理体系，不是单一的 `vec3 toneMap(vec3 x)` 函数。

在游戏 Shader 中常见所谓的“ACES Tone Mapping”，很多时候指**某个曲线拟合或艺术近似**，不能将它与完整 ACES Reference Color Pipeline 混为一谈。

### 2. 常见的 ACES-Inspired 拟合曲线是什么？

一类被广泛引用的标量拟合：

$$
\boxed{
T_{\mathrm{fit}}(x)=
\frac{x(2.51x+0.03)}
{x(2.43x+0.59)+0.14}.
}
$$

典型 Shader 对 RGB 通道分别计算后 Clamp 到 [0,1]。它以 ACES 风格的高光压缩为目标，但**不是完整 ACES RRT/ODT**；颜色原色空间、矩阵处理、色相行为和输出映射都有更复杂的细节。

### 3. 为什么简单 RGB 逐通道 Tone Mapping 会出现色相变化？

若 `R,G,B` 不相等，非线性函数对各通道的压缩比例不同。例如很亮的红色可能先接近输出上限，而绿蓝仍继续增长，使颜色饱和度和色相受到影响。

可以采用 Luminance-Preserving Tone Mapping、色度校正、Gamut Mapping 或更完整的颜色渲染变换，但这些都有各自的艺术/数值取舍。

### 4. ACES Fitted 比 Reinhard 一定好吗？

不一定。曲线形状、输入 Exposure、色域、Hue Preservation、Display Target、游戏美术风格都会改变对比结果。

面试中建议说清“我实现的是 Reinhard、ACES-Inspired Fit、还是完整 ACES Color Pipeline”，并描述实际 Shader 的数学形式及视觉区别。

### 5. 一组便于对比的 GLSL 函数

~~~glsl
vec3 ToneMapReinhard(vec3 linearExposed)
{
    vec3 x = max(linearExposed, vec3(0.0));
    return x / (vec3(1.0) + x);
}

vec3 ToneMapACESInspiredFit(vec3 linearExposed)
{
    vec3 x = max(linearExposed, vec3(0.0));
    vec3 a = x * (2.51 * x + 0.03);
    vec3 b = x * (2.43 * x + 0.59) + 0.14;
    return clamp(a / b, vec3(0.0), vec3(1.0));
}
~~~

这里要求输入已经经过适当的**曝光缩放**，输出仍应根据目标 Framebuffer 处理最终 Display Encoding。不能直接把此结果认定为 HDR10 的 PQ 编码信号。

### 6. 为什么 Tone Mapping 顺序不能替代 PBR 能量守恒？

Tone Mapper 能把错误超亮的图像视觉上压回 [0,1]，但它不会修复 BRDF 漏掉 `1/π`、非法光源单位、IBL 多重计算或错误 Gamma 空间加法。

应先保证 Lighting 的数学定义正确，再使用 Exposure/Tone Mapping 控制显示动态范围。

