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


---

## 十三、Bloom：为什么亮光会出现光晕？

### 1. Bloom 在模拟什么现象？

Bloom 是一种屏幕空间高亮光晕效果，用来近似光学系统中点扩散函数（PSF）、镜头内部散射、传感器/显示感知等因素让强亮区域向周围扩散的观感。

它**不是让物体真正照亮周围场景**，也不等于 Global Illumination。亮的 Emissive 面板可以产生 Bloom，但这不意味着附近墙壁会自动收到真实的反弹光。

### 2. Bloom 的典型后处理流程？

~~~text
Linear HDR Scene Color
           |
           v
Bright-Pass / Soft-Knee Prefilter
           |
           v
Half / Quarter Resolution Downsample Chain
           |
           v
Blur / Multi-Scale Filter
           |
           v
Upsample and Accumulate
           |
           v
Bloom + Scene HDR Composition
           |
           v
Exposure / Tone Mapping / Output Encoding
~~~

这里表示一种常见的**场景线性 HDR 中合成 Bloom，之后进行 Tone Mapping** 的设计。若引擎采用 Pre-Exposure，要先明确 Bloom Threshold 用哪个曝光尺度；不同游戏也可能采取其它顺序。

### 3. 为什么 Bloom 应使用线性 HDR 信号？

在场景 HDR 中，白墙和强烈灯光可能都“看起来很亮”，但后者的线性数值强度可能远大于前者。Bloom 可以利用这种差异决定哪些区域形成更强光晕。

若提前 Tone Map 到普通 SDR，极亮的多种光源可能都被压缩到接近同一显示数值，丢失光晕的亮度层次。

### 4. 什么是 Bright Pass / Threshold？

最简单的方法在亮度超过阈值 `T` 时保留高亮部分：

$$
\boxed{
B(C)=
\begin{cases}
0,&Y(C)\le T,\\
C,&Y(C)>T.
\end{cases}
}
$$

但是硬阈值在 `Y≈T` 附近不连续，镜头移动、TAA Jitter 或光源变化时可能导致 Bloom 闪烁。

### 5. Soft Knee 为什么比硬 Threshold 更稳定？

Soft Knee 在阈值附近设计一段平滑过渡，使 Bright Pass 响应连续。实现可以用 Smoothstep 或更合适的曲线，让接近阈值的像素逐渐贡献 Bloom，而不是突然全开/全关。

例如教学用的 Soft Weight：

$$
w(Y)=
\mathrm{smoothstep}(T-k,\ T+k,\ Y),
\quad k>0,
$$

再令 `B(C)=w(Y(C))*C`。这是简单平滑权重，并非某个商业引擎 Bloom Prefilter 的唯一标准算法。

### 6. 为什么 Bloom 常用多级 Downsample，而非直接大核 Blur？

在高分辨率图上进行大范围 Blur 成本很高。Downsample Pyramid 将不同尺度的模糊效果分散到低分辨率纹理上：

- 低 Mip 生成较局部的柔和光晕。
- 高 Mip 贡献更大范围、低频的光晕。
- Upsample 时把不同空间尺度组合。

这种方式更适合模拟层次丰富的亮光扩散，还能显著减少大半径 Filter 在全分辨率上的成本。

### 7. Bloom 是否必须进行独立 3×3 Gaussian Blur？

不是。可使用 Tent/Bilinear、Separable Gaussian、Dual Kawase、FFT 或其它卷积/滤波方法。实际画质和成本取决于 Kernel Shape、Halo Size、Texture Fetch、Memory Bandwidth 和目标光学风格。

### 8. Bloom 的 Energy Conservation 应怎样理解？

Bloom 作为视觉镜头/显示效果，可能从数学上对 HDR Scene Color 增加额外颜色，看起来让亮度增强。它不一定是严格能量守恒的真实光学 PSF 模拟。

若要模拟能量归一化的 PSF，需要考虑卷积核的能量分布、场景原图与散射部分的组合，而不应简单认定 `scene+bloom*intensity` 是精确的物理能量传输。

### 9. Bloom 与 Lens Flare 有什么区别？

Bloom 是高亮区域的相对局部/跨尺度扩散；Lens Flare 更多模拟强光在镜头内部反射、散射产生的条纹、光斑、鬼影形状，可能依赖光源位置与镜头参数。

它们常一起出现，但不能互相等同。

---

## 十四、GLSL：Soft-Knee Bloom 与多尺度合成

### 1. 用线性 Luminance 做 Bloom Prefilter

以下输入约定：`uSceneHDR` 存 Linear Scene Color，未 Tone Map；Threshold 与 Soft Knee 处于**相同线性场景/曝光尺度**，并且 Pixel UV 不跨越有效 Render Rect。

~~~glsl
#version 450 core

in vec2 vUV;
layout(location = 0) out vec4 outBloom;

uniform sampler2D uSceneHDR;
uniform float uBloomThreshold;
uniform float uBloomKneeWidth;

vec3 BloomPrefilter(vec3 sceneRGB)
{
    vec3 C = max(sceneRGB, vec3(0.0));
    float Y = dot(C, vec3(0.2126, 0.7152, 0.0722));

    float knee = max(uBloomKneeWidth, 1e-5);
    float weight = smoothstep(
        uBloomThreshold - knee,
        uBloomThreshold + knee, Y);

    return C * weight;
}

void main()
{
    vec3 scene = texture(uSceneHDR, vUV).rgb;
    outBloom = vec4(BloomPrefilter(scene), 1.0);
}
~~~

该函数只演示平滑高亮选取；完整 Bloom 一般还要低通/下采样和上采样。对 HDR 高亮做子采样时应考虑 Fireflies 和极亮区域的稳定性。

### 2. 为什么每一级 Bloom Downsample 要考虑正确的采样 Footprint？

Downsample 若只随机读取一个像素，可能跳过极亮的子像素高光或让它在不同帧突然出现。合理的空间滤波和采样设计有助于降低高频 Bright-Pass Flicker。

但过宽滤波会让小光源被抹得太淡；核形状和能量归一化需要符合艺术预期。

### 3. Bloom Upsample 合成该怎样理解？

若 `B_i` 是各层低分辨率 Bloom，经过 Upsample/Filter 回到同一空间和尺寸后可加权求和：

$$
\boxed{
B_{\mathrm{full}}=
\sum_i w_i\,\mathrm{Upsample}(B_i).
}
$$

最终一种常见合成：

$$
\boxed{
C_{\mathrm{combined}}=
C_{\mathrm{scene}}+s_B B_{\mathrm{full}}.
}
$$

其中 `s_B` 是艺术强度参数；`w_i` 和各 Mip 的过滤核会影响光晕半径与能量，需要避免所有 Mip 不加归一化地随意叠加导致强度依 Mip Count 改变。

### 4. 一个简单的 Bloom HDR 合成 Shader

~~~glsl
#version 450 core

in vec2 vUV;
layout(location = 0) out vec4 outColor;

uniform sampler2D uSceneHDR;
uniform sampler2D uBloomBlurHDR;
uniform float uBloomStrength;

void main()
{
    vec3 C = texture(uSceneHDR, vUV).rgb;
    vec3 B = texture(uBloomBlurHDR, vUV).rgb;

    outColor = vec4(C + B * uBloomStrength, 1.0);
}
~~~

输入 Bloom 必须已经与 Scene Color 使用相同的线性颜色空间和曝光尺度，且纹理采样坐标已按照分辨率/Viewport 转换正确设置。

### 5. 为什么不能把 Bloom 在 SRGB8 Buffer 上反复叠加还声称线性能量正确？

如果 Bloom 纹理已是 sRGB 编码，却按线性能量 `C+B` 直接相加，色彩计算通常不正确。应在适当 Linear Scene/Display-Linear 阶段组合，再在最终目标上做 Output Transfer Encoding。

### 6. Bloom 是否应该影响曝光 Metering？

取决于测光意图。若 Metering 希望代表场景中真实/输入照明，应通常在额外 Lens Bloom 前统计；否则已扩散的高光会改变更多像素的 Metering Weight，可能反馈到曝光系统。

某些特定美术系统可以有意让后处理后的亮区参与测光，但须明确这是另一种设计，而不是没有差别。

---

## 十五、Color Grading：LUT 处理的究竟是什么颜色？

### 1. Color Grading 是什么？

Color Grading 用于统一画面风格和色彩观感，可包含：

- White Balance（色温/色调）。
- Contrast、Saturation、Gamma、Gain。
- Shadows/Midtones/Highlights 调整。
- Curves、Color Wheels、Selective Color。
- Film Look、色彩风格与场景统一。

它与物理光照分工不同：PBR 描述入射与反射；Color Grading 负责最终成像的视觉风格。

### 2. Color Grading 一定在 Tone Mapping 之后吗？

不一定。某些 Color Grading 操作在 Scene-Referred HDR 下执行，有些在 Display-Referred SDR 下执行；也可以把多个 Color Transform Baking 到一个 LUT 中。

如果 LUT 是为 `[0,1]` 的 SDR Display-Referred RGB 制作，却拿它直接查询 `RGB>1` 的线性 HDR Scene Color，结果通常没有定义或被 Clip，造成高光和颜色损失。

### 3. 3D Color LUT 的输入输出是什么？

3D LUT 以三个颜色通道作为三维索引，将输入颜色映射到输出颜色：

$$
\boxed{
C_{\mathrm{out}}=
LUT_{3D}(C_{\mathrm{in}}).
}
$$

`Cin` 所属色域、Transfer、数值区间必须和 LUT 的作者/生成软件使用的 Domain 一致。

### 4. 3D LUT 为什么需要插值？

LUT 是离散采样网格，例如 `32³` 或 `64³` 个控制点。运行时大多数输入颜色位于网格之间，需要 Trilinear 或 Tetrahedral Interpolation 来估计连续变换。

LUT Resolution 越低，某些剧烈曲线/局部色彩调整可能出现 Banding；高分辨率则增加纹理储存和潜在采样成本。

### 5. 为什么 3D LUT 可以被 Texture3D 采样？

将 LUT 的 R/G/B 输入作为三维 UVW 坐标，使用 GPU 3D Texture 的 Linear Filtering，可获得 Trilinear 插值结果：

~~~glsl
uniform sampler3D uColorLUT;
uniform float uLUTSize; // e.g. 32.0

vec3 SampleLUT(vec3 color01)
{
    float size = uLUTSize;
    vec3 uvw = clamp(color01, 0.0, 1.0);
    uvw = (uvw * (size - 1.0) + 0.5) / size;
    return texture(uColorLUT, uvw).rgb;
}
~~~

这里的 UVW 半 Texel 转换适用于 LUT 以规则中心点离散存储的设计。**输入需要预先处于 LUT 定义的 [0,1] 域和色彩空间**；不能无条件把超范围 HDR 值 Clamp 后当成正确 HDR Grading。

### 6. 为什么 LUT 的伽马约定非常重要？

一个 LUT 若针对 sRGB 编码的 Display RGB 生成，在 Linear Scene RGB 上直接使用，颜色响应会完全不同。反过来，针对 Linear Display RGB 生成的 LUT 也不能拿去处理已经 sRGB 编码的值。

Color LUT 必须有明确 Input/Output Color Space Metadata，不能只传一张无语义 `RGBA8` 3D Texture。

### 7. 3D LUT 能代替所有 Color Grading 吗？

3D LUT 可以近似大量静态三通道颜色变换，但不天然适合：

- 强依赖场景局部空间位置的调整。
- 需要无限/极宽 HDR 动态范围且无合适 Log Encoding 的变换。
- 复杂时间自适应逻辑。
- 特定感知色彩/色域约束下的动态数学运算。

它是实用的预计算表达，不是所有颜色处理的完全替代。

---

## 十六、White Balance、对比度、饱和度与色域

### 1. White Balance 是做什么的？

White Balance 调整不同光源色温/色偏下中性色的呈现方式，通常依赖颜色空间转换、色适应（Chromatic Adaptation）或引擎的可控艺术近似。

它不简单等于给 `R` 通道乘 1.2、`B` 乘 0.8：这种简化能做艺术调整，但不等同于物理/色度学意义完整的白平衡算法。

### 2. Saturation 的一个线性 RGB 教学表达

对于线性 `C`、亮度 `Y=dot(C,w)`，定义 `s`：

$$
\boxed{
C_{\mathrm{sat}}=
Y\mathbf 1+s(C-Y\mathbf1).
}
$$

`s=0` 得到灰度；`s=1` 不变。该式是简单色彩插值，不是完全符合人眼感知或专业 Grading 系统的饱和度模型。

### 3. Contrast 与 Gamma 是同一种数学操作吗？

不是。Contrast 常围绕一个 Pivot 调整亮暗差异；Gamma/幂函数改变中间调响应。若用非线性 Power Curve 处理 HDR Linear Color，必须考虑输入非负和定义的参考亮度。

将所有调整都叫 Gamma 会掩盖它们在中灰、黑点与高光上的区别。

### 4. 什么是 Gamut Mapping？

场景/工作色域可能表示一些目标显示器无法表达的颜色。Gamut Mapping 负责将这些 Out-of-Gamut Colors 转为目标 Gamut 内更合理的颜色。

最简单的逐通道 Clamp 可能破坏色相/饱和度，现代显示管线往往需要更有感知依据的映射方法。

### 5. 为什么广色域 RGB 不能继续盲用 Rec.709 Luminance 权重？

不同 Primaries 到 XYZ 的矩阵不同，因此相对亮度 `Y` 的系数也不同。若当前工作空间为 ACEScg/AP1、Rec.2020 或其它线性广色域 RGB，则应使用该空间对应的 `RGB→XYZ Y` 转换或先转换到正确的测光空间。

**RGB 的线性性**和**RGB 属于哪个色域**是两个正交属性，不能混淆。

### 6. Color Grading 与 Display Calibration 有什么区别？

Color Grading 决定内容的创作风格与颜色变换；Display Calibration/Color Management 决定最终输出设备能否按照预期的亮度、白点、传输曲线和色域显示。

未经校准的显示器可能让画面风格看起来不同，但不意味着 Shader BRDF 错误；同样不能用 Color Grading 去代替正确的 Output Transfer Function。

---

## 十七、HDR Display：HDR Scene Buffer 与 HDR10 不是一回事

### 1. 什么是 SDR 输出？

普通 SDR 管线可使用特定的显示色域（常见 sRGB/Rec.709）、标准化的显示映射和相应 Transfer。为了让显示正确，需要明确是否输出 sRGB 编码数值及 Framebuffer 对其进行怎样的转换。

不能简单说“屏幕只有 0 到 1，所以就是 SDR”：所有数字显示编码都有有限码值范围，HDR10 的编码同样可归一化到 `[0,1]`，但数值背后的亮度语义完全不同。

### 2. 什么是 HDR10？

HDR10 是面向高动态范围显示的一组常见系统约定，通常包括：

- PQ (Perceptual Quantizer) 传输函数，ST 2084。
- Rec.2020 容器色域/信号表述。
- 10-bit 级别的编码支持与相应输出路径。
- 有关主显示和亮度的静态元数据（如兼容平台所要求的信息）。

这与“游戏内部使用 RGBA16F”不是同一个概念。HDR10 输出还需要系统、窗口、交换链/显示接口与硬件支持。

### 3. PQ 和 sRGB 的主要区别？

sRGB 编码面向 SDR 显示条件；PQ 是以**绝对显示亮度**为基础设计的非线性编码（ST 2084），在标准定义中覆盖到高绝对亮度范围。

**不能把 sRGB OETF 用在 HDR10 输出值上**；同样不能把 PQ 编码的 Display Data 直接视为可以用于 PBR 求和的线性 Radiance。

### 4. HLG（Hybrid Log-Gamma）是什么？

HLG 是另一类 HDR 传输方案，强调不同显示系统与广播等使用条件下的兼容和相对亮度映射特性。它与 PQ 的绝对亮度标定逻辑不同。

选择 PQ/HLG、SDR Backbuffer、平台 HDR Swapchain，取决于应用和显示生态，不由 Scene HDR Float Texture 格式自动决定。

### 5. HDR Output 需要什么额外工程处理？

至少要考虑：

- 系统/显示器的 HDR 能力检测。
- HDR Swapchain / Framebuffer Color Space。
- Target Peak Brightness 和 Paper White / UI Reference White。
- Tone/Output Mapping 到目标显示动态范围。
- Work Gamut 到输出 Gamut 变换。
- PQ/HLG/其他 Output Transfer Encoding。
- 元数据、显示器色彩管理及平台 API 路径。
- SDR 与 HDR 模式切换的历史资源、曝光、UI 合成差异。

### 6. 为什么 HDR UI 不能简单用传统 SDR 白色直接叠加？

HDR 场景中的亮度数值可能远大于 SDR UI 的 1.0 参考范围。如果没有明确 Paper White/Reference White 和 UI Composite Space，UI 可能过暗、过亮或在设备上不一致。

很多引擎对 SDR UI Layer 做专门的 HDR Composite/Scaling，使文本在宽亮度范围场景中仍可读。

### 7. “Linear HDR Render Target + ACES Fitted = HDR10 输出”这句话为什么错误？

它至少漏掉了 HDR10 的显示色域、绝对亮度映射、PQ 输出编码、交换链/平台支持与可能的元数据。ACES 风格曲线只是可能参与其中的一部分 Tone Mapping。


---

## 十八、后处理 Pass 顺序应该怎样设计？

### 1. 为什么不能把所有 Postprocess 当成随意交换顺序的滤镜？

很多运算不满足交换律。曝光与 Tone Mapping：

$$
T(E C)\ne E T(C).
$$

HDR Bloom 和 Tone Mapping：

$$
T(C+B)\ne T(C)+T(B).
$$

Color Grading 变换可能依赖输入的颜色空间与亮度范围，TAA 依赖 History 和一致的 Pre-Exposure，UI 需要独立的显示范围，因此 Pass 顺序与数据语义同样重要。

### 2. 一种自洽的 HDR 后处理方案是什么？

~~~text
Geometry / Shadow / PBR / Indirect Lighting
                     |
                     v
           Linear HDR Scene Color
                     |
           +---------+----------------------+
           |                                |
           v                                v
  Luminance Metering                 Bloom Prefilter
  / Histogram                         / Pyramid
           |                                |
           v                                v
   Target Exposure                  Bloom Reconstruct
           |                                |
           v                                |
 Exposure Adaptation                       |
           |                                |
           +--------------------+-----------+
                                |
                                v
               Scene HDR + Bloom Composition
                                |
                                v
         TAA / TAAU (if this is chosen order)
           [Velocity / History / Pre-Exposure]
                                |
                                v
             Exposure + Tone / Color Grading
                                |
                                v
            Gamut Mapping / Output Encoding
                                |
                                v
                    UI Composite / Present
~~~

**示意，不是唯一标准。** 有些引擎在 Bloom 之前运行 TAA/TAAU，有些在 TAA 后再生成/合成 Bloom；Metering 可能使用 TAA 后颜色或特殊降采样，最终 UI 合成可能在 HDR Output Transform 前于已知绝对亮度空间内进行。

### 3. 为什么 Metering 依赖的 Color Buffer 选择需要明确？

如果 Histogram 对 Current Pre-Exposed Scene Color 统计，却按原始 Scene-Linear 标定解释 Bin Range，就会统计错误。若选择 Bloom/Tone Map 后的颜色，则亮度分布可能受光晕或曲线压缩影响。

因此 Metering Pass 的输入、Pre-Exposure Scale 和 Histogram Range 必须是明确设计的 Frame Metadata。

### 4. TAA 在 Bloom 前还是后更好？

不同方案各有问题：

- TAA 在 Bloom 前：Bloom 使用已稳定的 HDR Color，有机会减少高光采样闪烁；但 Bloom 产生的结果不再经同一个 TAA History。
- Bloom 在 TAA 前：Bloom 也可能被时域平滑；但如果高亮扩散变化很大，History Clip/Reactive 需要避免光晕拖尾。
- 分离时域：对 Bloom 或亮点使用专门稳定策略，再与 TAA 结果合成。

必须按目标画质和渲染管线验证，不能简单断言某一种顺序绝对正确。

### 5. Exposure 应在 TAA 前应用还是后应用？

很多实时管线使用 Pre-Exposure 让 TAA 处理数值范围合适的 HDR Scene Buffer，并在最后应用剩余 Exposure Scale；还有其它方案可将曝光适配融入时域处理。

关键条件是**Current 和 History 同一数值语义**、曝光历史变化被正确转换、最终 Tone Mapper 收到设计好的 Exposed Color。

### 6. 为什么每个 Postprocess Pass 都需要知道颜色语义？

同一张 `RGBA16F` Texture 可能储存：

- Scene Linear HDR。
- Pre-Exposed Scene Linear HDR。
- Exposed HDR。
- Tone-Mapped Display-Linear SDR。
- 未编码/已编码的 Display RGB。
- 某种 Log/PQ 内部编码。

这些语义如果没有在资源命名、Shader Uniform 或 RenderGraph Metadata 中明确，极易出现重复 Gamma、错误曝光与色彩不一致。

---

## 十九、OpenGLRenderer：HDR Framebuffer 与最终输出实现

### 1. 怎样分配一个 HDR Scene Color Target？

下面使用 OpenGL 4.5 Direct State Access，创建一个带 `RGBA16F` Scene Color 与深度纹理的 FBO：

~~~cpp
GLuint CreateHDRFramebuffer(
    GLsizei width, GLsizei height,
    GLuint& outColor, GLuint& outDepth)
{
    glCreateTextures(GL_TEXTURE_2D, 1, &outColor);
    glTextureStorage2D(
        outColor, 1, GL_RGBA16F, width, height);
    glTextureParameteri(outColor, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTextureParameteri(outColor, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTextureParameteri(outColor, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTextureParameteri(outColor, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);

    glCreateTextures(GL_TEXTURE_2D, 1, &outDepth);
    glTextureStorage2D(
        outDepth, 1, GL_DEPTH_COMPONENT32F, width, height);
    glTextureParameteri(outDepth, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTextureParameteri(outDepth, GL_TEXTURE_MAG_FILTER, GL_NEAREST);

    GLuint fbo = 0;
    glCreateFramebuffers(1, &fbo);
    glNamedFramebufferTexture(
        fbo, GL_COLOR_ATTACHMENT0, outColor, 0);
    glNamedFramebufferTexture(
        fbo, GL_DEPTH_ATTACHMENT, outDepth, 0);

    GLenum status = glCheckNamedFramebufferStatus(
        fbo, GL_FRAMEBUFFER);
    if (status != GL_FRAMEBUFFER_COMPLETE)
    {
        // Production code logs the status and destroys resources.
        return 0;
    }

    return fbo;
}
~~~

**工程限制：** 为突出 HDR FBO 构造，本函数省略完整 RAII、失败回收、Resize、MSAA、FBO DrawBuffer 配置的所有替代路径，以及 OpenGL Error Handling。真正项目还需确认最大纹理尺寸、GL Context 与 Render Target 生命周期。

### 2. 为什么 Scene HDR FBO 要与 Final SDR Backbuffer 分离？

Scene HDR Buffer 中的 `RGB>1` 需要保留供 Bloom/Exposure/Tone Mapping 使用；默认 Backbuffer 往往由窗口系统设置特定 SDR/HDR 格式与 Output Encoding 约定，不能假设它可以安全保存同样的线性 HDR 数据。

先渲染到浮点离屏目标，再执行 Postprocess，最后才写 Present Target，能让每一步数据含义更清晰。

### 3. 最终 SDR 输出 Shader 的示意写法

假设：

- 当前 HDR Scene Texture 未经 Tone Mapping，存 Linear RGB。
- `uExposure` 是定义好的总体曝光系数。
- 输出目标使用**线性 UNORM 纹理/Framebuffer，且 GL_FRAMEBUFFER_SRGB 不会自动编码**。
- 因此 Shader 最后**手动执行一次 sRGB Encode**。

~~~glsl
#version 450 core

in vec2 vUV;
layout(location=0) out vec4 outColor;

uniform sampler2D uHDRScene;
uniform float uExposure;

vec3 ToneMapACESInspiredFit(vec3 exposedRGB)
{
    vec3 x = max(exposedRGB, vec3(0.0));
    vec3 a = x * (2.51 * x + 0.03);
    vec3 b = x * (2.43 * x + 0.59) + 0.14;
    return clamp(a / b, vec3(0.0), vec3(1.0));
}

vec3 EncodeSRGB(vec3 linearRGB)
{
    vec3 x = clamp(linearRGB, vec3(0.0), vec3(1.0));
    vec3 low = 12.92 * x;
    vec3 high = 1.055 * pow(x, vec3(1.0 / 2.4)) - 0.055;
    return mix(high, low,
               lessThanEqual(x, vec3(0.0031308)));
}

void main()
{
    vec3 scene = texture(uHDRScene, vUV).rgb;
    vec3 exposed = scene * uExposure;
    vec3 mappedLinear = ToneMapACESInspiredFit(exposed);
    vec3 encoded = EncodeSRGB(mappedLinear);
    outColor = vec4(encoded, 1.0);
}
~~~

**重要：** GLSL `mix(vec3,vec3,bvec3)` 是标准的逐通道布尔选择重载（在符合支持的 GLSL 版本中）。本示例只给出一种明确的手动编码流程；若目标使用可自动 sRGB 编码的附件并启用 `GL_FRAMEBUFFER_SRGB`，就应改为直接输出适当 Display-Linear RGB，不能再手动 Encode 一次。

### 4. 为什么无法对“默认 Framebuffer”统一给出一行正确的 Gamma 代码？

默认 Framebuffer 的实际 Color Format/Color Space、Window System、Swapchain 选择、`GL_FRAMEBUFFER_SRGB` 状态可能不同。

在一个 GL Window 上正确的手动编码策略，在另一个 SRGB-Capable/Auto-Encode Backbuffer 上可能双重编码。应通过平台/上下文配置和测试色阶确认，而不是只复制 Shader 里的 `pow(color,1/2.2)`。

### 5. 为什么 Display Target 如果是 HDR10 不能复用上面的最终 Shader？

因为它只输出 sRGB 编码的 SDR 范围颜色，没有 Wide Gamut、绝对亮度/Paper White 映射或 PQ Encoding。HDR10 输出需要独立的 Display Transform 与 Swapchain 路径。

### 6. 如果已经使用 Scene Pre-Exposure 怎么修改？

假设 Scene Buffer 存 `C_stored=P*C_scene`，而最终需要曝光 `E`，Tone Mapper 前应：

~~~glsl
vec3 exposed = storedScene * (exposure / preExposure);
~~~

前提是你的引擎 `preExposure` 的定义就是之前文档所述的乘法尺度 `P`，且 Bloom/History 已处于兼容缩放域。不能既在 Lighting Shader 应用 `E`，又在 Postprocess 无条件乘一遍 `E`。

---

## 二十、Exposure Metering Mask 的工程实践

### 1. 为什么游戏往往需要给 Auto Exposure 加“关注主体”的权重？

相机视野变化可能包含大面积天空、强烈灯光或黑暗墙壁，但玩家主要关注角色、准星附近或交互区域。

以屏幕位置为输入的 Metering Mask 能让曝光系统对主体区域更加敏感，并降低边缘极端亮度对曝光的干扰。

### 2. 一个简单的中心权重怎么写？

令屏幕 UV `u,v∈[0,1]`，以屏幕中心 `c=(0.5,0.5)` 为参考：

$$
r^2=(u-0.5)^2+(v-0.5)^2.
$$

可采用高斯式权重：

$$
\boxed{
w(u,v)=
w_{\min}+
(1-w_{\min})\exp\left(-\frac{r^2}{2\sigma^2}\right).
}
$$

其中 `sigma` 控制 Metering Region 的宽度，`w_min` 防止边缘完全失去权重。此公式忽略画面非方形 Aspect Correction；真实项目可对 UV 距离进行宽高比校正，或在 Pixel Space 计算。

### 3. GLSL 中怎样计算权重？

~~~glsl
float CenterMeteringWeight(vec2 uv, vec2 renderSizePx,
                           float radiusPixels, float minWeight)
{
    vec2 deltaPixels = (uv - vec2(0.5)) * renderSizePx;
    float sigma = max(radiusPixels, 1.0);
    float d2 = dot(deltaPixels, deltaPixels);

    return mix(
        clamp(minWeight, 0.0, 1.0),
        1.0,
        exp(-0.5 * d2 / (sigma * sigma))
    );
}
~~~

这使用 Pixel Space 距离，使不同 Aspect Ratio 下的中心区域可以用统一 Pixel Radius 表示。如果目标是归一化视角/屏幕占比，应该采用另一种明确的尺度。

### 4. 怎样把 Metering Mask 正确接进 Histogram？

如果一个像素的权重 `w_i∈[0,1]`，严格的 Weighted Histogram 应让该样本为相应 Bucket 贡献数值 `w_i` 而不是无条件 1。

`uint` Histogram 可以通过固定点编码（例如 `round(w_i*S)`）再 Atomically Add，但要考虑：

- 量化误差。
- 总 Count 上界与 Overflow。
- Reduction 用相同 Scale 解释权重。
- 部分像素为零权重时防止除零。
- Workgroup 内 Weighted Accumulation 的 Atomic 容量。

另一个方案是使用浮点累加/归并，但要确认目标 GPU 的原子浮点支持与精度/性能。**不要把非加权 uint Histogram 直接当成已经使用了 Metering Mask 的结果**。

### 5. 为什么角色 Mask 有时比单纯中心权重更合适？

玩家角色可能不在屏幕正中心；过场动画、第三人称偏移镜头也可能把关注对象放在构图侧边。

可以对角色屏幕包围区、Focus Point 或专门的 Metering Texture 赋权，不过要处理遮挡、角色突然离开屏幕、Mask 闪烁与镜头切换时的时间稳定性。

### 6. Metering Mask 与后处理 Vignette 是一回事吗？

不是：

- Metering Mask：影响亮度统计，从而影响**整张图**的相机曝光。
- Vignette：直接改变屏幕边缘的显示亮度/颜色，是成像风格或镜头衰减效果。

即使二者都以屏幕中心到边缘的距离构造函数，它们在 Pipeline 中的职责完全不同。

### 7. 为什么 Exposure Histogram 和 Texture Editor 的颜色直方图不能直接互换？

Texture Editor 的 Histogram 可能按 Source Art 的 `R/G/B/A` 值或解码纹理数据统计，而 Auto Exposure Histogram 通常按 HDR Scene 的**线性 Luminance、Log2 Bin、Metering 权重及 Percentile**统计。

数据输入、色彩空间、单位、Bin 映射和权重语义完全不同。GPU Compute/Reduction/Async Readback 的工程设计可复用，但算法定义不能直接照搬。

---

## 二十一、其它常见 Postprocess：Vignette、Grain、Dithering 与 Lens Effects

### 1. Vignette 是什么？

Vignette 是图像边缘比中心暗的视觉效果，可以模拟光学镜头/传感器成像衰减，也可以作为艺术手段突出中心主体。

它通常属于 Display/Color Postprocess，不应该影响物理光源照明的 BRDF 或场景 Shadow Visibility。

### 2. Film Grain 为什么会加入已经很干净的画面？

Grain 是有意设计的纹理噪声或胶片风格，使图像更有质感、改善部分量化/低频大块色阶观感。但它不能替代正确的 Dithering 或 Tone Mapping，过强 Grain 会降低 TAA 稳定性/压缩效率。

### 3. Dithering 与 Film Grain 有什么区别？

Dithering 有意加入受控的小幅噪声，以缓解量化产生的 Banding；Film Grain 更多是风格化效果，虽然二者都可能用到噪声。

对于最终 8-bit SDR 输出，在正确的 Output Encoding/Quantization 之前应用适当 Dither，有机会让平滑色阶更自然，但噪声幅度/统计和色彩空间必须匹配。

### 4. Chromatic Aberration 是什么？

色差是不同波长的光在光学系统中发生不同成像位置/倍率的现象。实时后处理通常以 RGB 通道之间的空间采样偏移进行风格化近似。

它不是 PBR 的色散/折射方程精确求解，且过量会损害清晰度和边缘视觉质量。

### 5. Depth of Field 与 Motion Blur 属于同一类 Blur 吗？

它们都可以通过 Screen-Space/Postprocess 重建，但成像机制不同：

- Depth of Field：有限镜头光圈导致不在焦平面上的点形成 Circle of Confusion，产生景深模糊。
- Motion Blur：有限快门时间对运动中的场景进行时间积分，产生运动方向模糊。

二者的过滤核、Depth/Occlusion 处理和采样时间维度不同，不能简单用一个 Gaussian Blur 代替。

### 6. 为什么这些效果都和 TAA 有关系？

TAA 利用 History 做时间重建；Film Grain、Noise、Motion Blur、DOF、Vignette 和 Camera Shake 可能改变颜色稳定性、运动估计或重投影有效性。

例如 Film Grain 通常不希望被 TAA 长时间平均掉；Motion Blur 应与真实运动及 TAA 的有意/无意历史混合区分。因此 Pass Order 和 History State 很关键。

### 7. 什么是 Lens Dirt？

Lens Dirt 将 Bloom/高亮分布与镜头污渍纹理组合，产生可见的眩光污渍艺术效果。它一般是对光晕信号的视觉调制，而不是光照系统中的几何遮挡。

---

## 二十二、性能优化：Bloom、Histogram、Tone Mapping 谁更贵？

### 1. 哪些后处理容易受 Memory Bandwidth 限制？

典型全屏 Pass 常涉及大量 Texture Read/Render Target Write；尤其 HDR 格式每像素字节数较多，多次 Fullscreen Ping-Pong、Bloom Mip 生成和 TAA History 会产生明显带宽与 Cache 压力。

但 Compute Histogram 也可能受 Atomic Contention 影响；Tone Mapping 可能有复杂 Color Math 与 3D LUT Fetch；不能无条件断言所有 Postprocess 都只受 Bandwidth 限制。

### 2. 常见 Render Target 每像素占多少字节？

以未压缩、单样本、忽略 Alignment/Compression/Metadata 为例：

| 格式 | 名义每像素字节数 | 典型用途 |
|---|---:|---|
| RGBA8 | 4 | SDR/某些后处理缓冲 |
| R16F | 2 | 单通道 HDR/亮度/曝光相关数据 |
| RG16F | 4 | 速度/双通道统计 |
| RGB16F | 6（名义） | 某些 RGB 浮点数据；格式/API 支持和布局需确认 |
| RGBA16F | 8 | HDR Scene Color、TAA History |
| RGBA32F | 16 | 高精度中间结果/参考测试 |

实际 GPU Allocation、Tile Memory、压缩以及多样本格式会改变真实内存/带宽，不能以表中名义字节数预测全部成本。

### 3. 1080p 的一张 RGBA16F 约占多少空间？

$$
1920\cdot1080\cdot8
=16\,588\,800\ \mathrm{bytes}.
$$

折算为二进制 MiB：

$$
\boxed{
\frac{16\,588\,800}{1024^2}
\approx15.82\ \mathrm{MiB}.
}
$$

这是**单张无多样本、无 Mip、无额外分配开销**的名义大小。两张 TAA History、Bloom Mips、HDR Ping-Pong 等会另外占空间。

### 4. 为什么 Histogram 适合先 Downsample？

自动曝光只需要稳定统计，通常无需每像素精确保存所有空间细节。合理降采样可减少 Image Load、Atomics、Global SSBO Write 和资源争用；但要验证极亮小光源、Metering Mask 和分位数是否被低分辨率样本错误稀释。

### 5. 为什么 Bloom Pyramid 通常比一次全分辨率大核 Blur 更高效？

低分辨率 Mip 的像素数按尺寸平方递减，大范围的低频 Blur 可在较小纹理上计算，减轻 Texture Fetch/Filtering 成本。

不过多 Pass Transition、Upsample 和额外资源仍有代价，应比较总 GPU 时间和最终光晕质量。

### 6. 怎样降低 Fullscreen Pass 数量？

- 将多个兼容的 Color Transform 合并进同一个 Pass。
- 将 Static Color Grading Operations Baking 到 3D LUT。
- 用 Compute/Tiled Pass 或 Pass Fusion 组合部分算子。
- 减少不必要的 HDR Full Resolution Intermediate Copies。
- 合理选择 Bloom Mip Count、分辨率、Filter Tap。
- 管理临时资源别名与 FrameGraph Lifetime。

**注意：** Pass Fusion 可能增加寄存器压力、Shader 复杂度与输入依赖，不保证一定更快；要 GPU Profiling 验证。

### 7. 为什么不能用 CPU Frame Time 判断后处理是否 GPU 瓶颈？

CPU 可能在提交 Draw/Dispatch 时与 GPU 异步执行；真正的 GPU 后处理成本需要 Timestamp Query/GPU Profiler 等测量。

在 OpenGL 中插入会同步等待 GPU 的 Query/Readback 也可能影响性能结果，应使用延迟读取、GPU Timestamp 或工具侧时间线分析。

### 8. HDR Float Format 和 Bloom 精度怎样取舍？

Half-Float `RGBA16F` 经常在范围/精度/带宽之间取得较好的平衡，但大 HDR 值和微小暗部数值可能带来 Precision/Underflow/Overflow 问题。

某些 Bloom Mips 可以使用较低精度/较小分辨率；而 Metering/Exposure Reduction 需要避免在极值场景中的数值异常和 Histogram Overflow。应按数据范围选格式，而不是什么都用 RGBA32F。

---

## 二十三、RenderDoc 与 HDR 后处理问题的诊断流程

### 1. 画面整体过曝，应该先查什么？

建议按阶段逐项显示：

1. Raw Linear Scene HDR Color：PBR/Light Strength 是否已经异常。
2. Luminance Heatmap：测光输入是否使用正确线性颜色和 Primaries。
3. Histogram：Bin Mapping、Clip Range、Mask、Percentile 是否合理。
4. Target Exposure：Middle Gray/Compensation 是否正确。
5. Adapted Exposure：是否方向反了或速度过快。
6. Pre-Exposure Ratio：是否重复乘/除或 History 不一致。
7. Tone Mapping Output：是否输入范围与曲线预期不符。
8. Final Encoding：是否重复 sRGB/Gamma 或使用错误 HDR Display 模式。

### 2. 为什么“模型 Albedo 是正确的，颜色仍发灰/发白”？

可能原因很多：

- BaseColor Texture sRGB Flag 错误。
- Lighting 阶段错误使用了编码后的颜色。
- 曝光目标/Key 设置不合理。
- Reinhard/Filmic 曲线压缩使对比度降低。
- Tone Map 后重复 sRGB Encode。
- Output Gamut/Display Calibration 不匹配。
- Color Grading LUT 输入 Domain 不对。

正确做法是查看**每个阶段具体数值**，而非仅调整 Albedo。

### 3. 为什么画面突然走进屋子时曝光不停抖动？

检查：

- Histogram Sampling 是否受低分辨率取样和小亮点影响。
- Percentile/Highlight Weight 是否稳定。
- Exposure Adaptation 是否使用 Delta Time 和正确的 Speed。
- Camera Cut/Resize 是否错误清空/重用 Exposure History。
- Pre-Exposure 与 Metering 是否形成错误反馈。
- Bloom/TAA 是否在错误的空间进入 Metering。

### 4. 为什么调 Bloom Threshold 时光晕有时完全没有？

确认：

- Prefilter 读取的是 Scene Linear HDR 还是已 Tone-Mapped SDR。
- Threshold 是否对应同一种数值/曝光尺度。
- 是否提前 Clamp 到 1.0。
- Bloom Bright Pass/Mip Chain 是否真正生成数据。
- Upsample/Compose Pass 是否绑定了正确纹理和强度。
- Bloom Render Target Filter/Mip 资源是否完整。

### 5. 为什么画面有不连续色带（Banding）？

可能来源：

- HDR→SDR/8-bit Quantization。
- 不合适的 Tone Curve 或 Grading LUT Resolution。
- Output Gamma/Transfer 不正确。
- 某个低精度中间 RT 过早量化。
- 缺少适当 Dithering。
- Bloom/Blur 的采样不足。

推荐用长渐变 Ramp Test，在 Linear HDR、Tone-Mapped Linear 和 Encoded Output 三个阶段分别查看。

### 6. 为什么 HDR 输出在不同显示器观感差别很大？

HDR Display 的最大亮度、黑位、色域、Tone Mapping 和系统显示设置可能不同。没有一致的 Paper White、Peak Brightness、Gamut Mapping 和 PQ/HLG Output Transform 就无法保证一致观感。

如果应用运行在 SDR Backbuffer，却以为自己已输出 HDR10，最终 HDR Scene Buffer 再亮也不能说明显示链路正确。

### 7. TAA 为什么会因为 Auto Exposure 出现 Ghosting？

如果 History 的 Pre-Exposure Scale 来自 `P_{t-1}`，当前 Color 使用 `P_t`，但 History 没做 Scale Ratio 校正，同一物理表面就会以不同数字亮度混合，形成残影或突然的亮度变化。

因此应结合第八篇的 Jitter/Velocity/History Reprojection 调试，同时可视化 History Before/After Exposure Conversion。

### 8. RenderDoc 重点看哪些资源？

| Debug 资源 | 用途 |
|---|---|
| SceneColor HDR | 检查 Lighting 是否保留高亮数值 |
| Linear Luminance | 检查空间/通道系数 |
| Histogram SSBO | 检查 Bin、黑色样本及分位数 |
| Exposure Buffer | Target/Current/Pre-Exposure |
| Bloom Prefilter | 检查 Threshold、Soft Knee |
| Bloom Pyramid | 检查 Downsample/Blur/Upsample |
| TAA History | 检查曝光一致性/残影 |
| Tone-Mapped Color | 检查 Curve 是否合理 |
| Grading LUT | 检查 LUT Domain、采样坐标 |
| Final Output | 检查 Display Encoding/Gamut |

### 9. 为什么 Texture Viewer 显示 HDR 很亮或很暗不能单独证明 Shader 错？

HDR Texture Viewer 通常自身具有显示范围、缩放或亮度映射控制；工具显示的预览可能不是原始 Float RGB 直接线性映射到显示器。

应查看具体像素的**原始数值**及格式、显示范围与所有后续变换，而不是只根据预览是否灰白判断算法错误。


---

## 二十四、秋招高频面试题与连续追问

### Q1：什么是 HDR Rendering？
在 Lighting/部分 Postprocess 中保留高动态范围 Scene Color，避免过早 Clamp/Quantize，并通过曝光和显示映射输出到目标设备。

### Q2：为什么 Scene Linear RGB 大于 1 不代表 BRDF 违反能量守恒？
BRDF 能量守恒限制入射能量被表面反射的比例；强烈入射照明仍能产生很大的出射 Radiance/HDR RGB。

### Q3：HDR Rendering 与 HDR10 Output 有什么区别？
前者是内部高动态范围计算与资源；后者还涉及目标显示峰值亮度、PQ、色域、输出接口/Swapchain 和相关元数据。

### Q4：为什么光照计算必须在 Linear RGB 中？
能量叠加与 BRDF/曝光比例要求线性运算；sRGB 编码数值含非线性 Transfer，不能直接解释为线性能量。

### Q5：sRGB Encoding 能否简单等于 Gamma 1/2.2？
只能作为某些场景的近似。标准 sRGB 是包含暗部线性段的分段曲线，严格编解码阈值和幂指数都不同。

### Q6：OpenGL sRGB Texture 与 GL_FRAMEBUFFER_SRGB 各负责什么？
前者通常在纹理采样时将 sRGB Color Channel Decode 到 Linear；后者在适当的 sRGB Framebuffer 写入时处理 Output Encoding。二者发生于不同阶段。

### Q7：为什么 BaseColor Texture 通常设置 sRGB，而 Roughness 不设置？
前者存编码颜色，后者存有确定数值意义的数据；对 Roughness 做 sRGB Decode 会扭曲材质参数。

### Q8：什么是 Luminance？常见 Rec.709 系数是多少？
相对亮度可由线性 Rec.709/sRGB RGB 近似为 `Y=0.2126R+0.7152G+0.0722B`。对其它 Primaries 需要对应转换。

### Q9：为什么不能直接对 sRGB 编码通道使用 Rec.709 亮度权重？
这组系数假设输入是 Linear RGB；编码颜色具有非线性响应，会造成测光比例错误。

### Q10：为什么 Auto Exposure 常在 Log2 Luminance 空间做统计？
场景亮度可能跨多个数量级，Log2 将亮度倍数转换为加法 Stops，便于大范围 Histogram/Metering。

### Q11：几何平均亮度和算术平均亮度有何区别？
几何平均 `Ygeo=2^(mean(log2(Y)))`，对乘法变化更自然；算术平均是 `sum(Y)/N`，极亮少数点可能产生不同的影响。

### Q12：为什么 Histogram 不能直接计算 log2(0)？
零的对数无定义，会造成负无穷或数值异常；应选择 Black Bin、Epsilon Clamp 或明确排除策略。

### Q13：什么是 Exposure Compensation 的一挡？
本篇正曝光补偿 `+1` 令最终曝光乘数 `E` 乘 2；但 EV100 的数值增大通常意味着相机进光减少，应区分符号。

### Q14：EV100 的曝光公式？
`EV100=log2((N²/t)*(100/ISO))`，其中 N 为光圈 f-number、t 为快门秒数，描述摄影参数关系。

### Q15：为什么没有标定时不应把 EV100 直接当成曝光乘数？
EV100 到数字 HDR 输出还需要摄影/引擎曝光校准约定、物理亮度标定与 Tone Mapping，不能直接用 2^EV100 当作所有引擎通用乘法。

### Q16：Middle Gray 为什么经常取 0.18？
18% Gray 是常见摄影/曝光参考约定，便于解释目标亮度定位；实际中灰显示值还受 Tone Curve、标定与输出变换影响。

### Q17：什么是 Basic Auto Exposure？
根据当前画面或指定区域的（加权）平均/Log Mean Luminance 计算目标曝光，再经时间适应得到实际曝光。

### Q18：Histogram Auto Exposure 相比全图平均有什么优势？
可按亮度分布分位数剔除/减小极亮或极暗尾部影响，也便于支持 Metering Mask 与更稳定的摄影控制。

### Q19：Metering Mask 怎样影响自动曝光？
它给屏幕不同位置的测光样本赋予权重，需在 Histogram/统计的累加阶段真正生效，而非事后任意修改曝光结果。

### Q20：为什么 HDR Histogram 通常按 Log Luminance 分桶？
Log2 档位与曝光 Stops 对应，可在有限 Bucket 中覆盖跨数量级的亮度范围。

### Q21：GPU Histogram 为什么可能遇到 Atomic 争用？
大量像素集中在有限几个亮度桶里，直接对 Global Bins 原子加会冲突；可先用 Workgroup Shared Histogram 汇总。

### Q22：Compute Shader 中为什么所有线程都要执行 Barrier？
在同一 Workgroup 的 Shared Memory 同步点，若仅部分 Invocation 提前 Return，剩余线程可能发生同步错误；越界线程也应到达 Barrier。

### Q23：Percentile Clipping 与 RGB Clamp 有什么区别？
前者按 Histogram 排序/累计质量排除亮度分布尾部，再对剩余亮度求统计值；后者直接限制颜色通道值，不是同一操作。

### Q24：Exposure Adaptation 为什么需要 Delta Time？
如果每帧使用相同固定插值因子，30 FPS 和 120 FPS 的每秒响应不同。指数平滑 `1-exp(-speed*dt)` 更接近按时间控制。

### Q25：为什么可以在 Log Exposure/EV 空间做平滑？
曝光本质按倍数/Stops 改变，Log2 平滑让相同 Stop 跨度具有更一致的时间响应，而非受当前线性乘数大小直接影响。

### Q26：为什么 Brighten/Darken Speed 名称容易混淆？
可能指场景亮度增加，或指曝光乘数增加，两者方向相反；应明确比较 `E_target` 和 `E_current` 的变化方向。

### Q27：什么是 Pre-Exposure？
在部分实时引擎中，Lighting/HDR Buffer 提前使用已知曝光相关尺度储存颜色，使浮点范围与时域过滤更合适；不自动等于最终曝光值。

### Q28：TAA History 为什么要补偿 Pre-Exposure Ratio？
若上帧和当前帧 History 的存储比例不同，必须转换到同一尺度后混合，否则会因曝光变化产生颜色残影或闪烁。

### Q29：若 SceneBuffer=P*SceneLinear，最终要应用曝光 E，该怎样处理？
Tone Mapping 前应乘 `E/P`，在已经应用过 Pre-Exposure 的 HDR Buffer 上不能再把 E 原样乘一遍。

### Q30：Tone Mapping 与 sRGB Gamma/Encoding 的区别？
Tone Mapping 压缩/重分配 HDR 动态范围；Output Transfer Encoding 把显示域颜色变为目标传输编码，二者不是一个概念。

### Q31：Reinhard Tone Mapping 公式是什么？
`T(x)=x/(1+x)`，对非负输入会平滑映射到 [0,1)；简单、易于测试，但可能降低高光对比或使画面偏灰。

### Q32：White-Point Reinhard 是否对任意大输入自动限制到 1？
不一定。常见扩展式 `x*(1+x/W²)/(1+x)` 在 `x=W` 输出 1，但 `x>W` 可能输出大于 1。

### Q33：ACES 与常见 ACES Fitted Shader 是否相同？
不是。ACES 是完整色彩管理体系；一些游戏 Shader 只使用受 ACES 启发的标量/逐通道 Tone Curve，不包括完整 RRT/ODT 与色域处理。

### Q34：为什么逐通道 Tone Mapping 可能改变 Hue？
R/G/B 经非线性压缩的相对比例不再恒定，强饱和高光容易发生色彩偏移，需要考虑 Luminance-Preserving/Gamut 方案。

### Q35：Bloom 和 GI 有什么区别？
Bloom 是 HDR 高亮在图像/镜头空间的扩散；GI 是光在场景几何间传播与反射，两者物理意义和实现层次不同。

### Q36：为什么 Bloom Bright-Pass 常需要 Soft Knee？
硬阈值在亮度临界点会突然开/关，易在运动和曝光变化时出现闪烁；Soft Knee 让响应平滑。

### Q37：为什么 Bloom 常用 Downsample/Upsample Pyramid？
大范围柔和光晕可在低分辨率下更高效生成，组合不同 Mip 的模糊尺度，也可减轻全分辨率大核滤波成本。

### Q38：Bloom 应该在 Tone Mapping 之前还是之后？
常见 HDR Pipeline 在 Tone Mapping 前合成，以保留亮光差异；其它布局存在，但需明确阈值、曝光尺度与颜色空间一致性。

### Q39：Color Grading 与 PBR 有什么不同？
PBR 描述场景照明和材质反射；Grading 改变最终成像的色彩风格。Grading 不能修复错误的物理光照。

### Q40：3D LUT 为什么要明确 Input Color Space/Domain？
LUT 以 RGB 值作坐标；若它为 SDR sRGB 编码颜色设计，却用于 HDR Linear RGB，将产生错误查找、Clamp 和色彩变化。

### Q41：为什么 3D LUT 要用 Trilinear/Tetrahedral Interpolation？
LUT 网格离散，实际 RGB 往往位于节点之间，需要插值估计连续的颜色变换。

### Q42：为什么普通 sRGB 输出不能直接当 HDR10？
HDR10 需要相应的 PQ、色域、亮度映射及 HDR 显示通路；普通 sRGB Encode 仅处理 SDR 颜色编码。

### Q43：PQ 与 HLG 是一回事吗？
不是。PQ 按绝对亮度范围设计；HLG 是另一类 HDR 传输设计，其相对亮度/显示系统理念不同。

### Q44：HDR UI 为什么需要 Paper White 或参考白色？
UI 数值应映射到可读、稳定的显示亮度尺度，不能把 SDR UI 的白色数值直接当成 HDR 场景中的统一物理白光。

### Q45：Film Grain 与 Dithering 的区别？
Grain 是风格化噪声；Dither 用受控噪声缓解量化 Banding，可有重叠实现但目标不同。

### Q46：为什么 TAA、Bloom、Exposure 的顺序重要？
TAA 重用历史并要求曝光尺度一致；Bloom 依赖高亮范围；Exposure Metering 的输入阶段决定统计分布。不同顺序数值和画质不等价。

### Q47：为什么一个 RGBA16F HDR Scene Texture 不等于全部 Postprocess 都正确？
仍需正确 Color Space、Histogram、曝光标定、TAA History、Tone/Gamut Map、Output Transfer、Swapchain Format 与 Shader Resource State。

### Q48：后处理性能主要受哪几类因素影响？
全屏 Read/Write Bandwidth、采样次数、Bloom Pyramid、Histogram Atomic Contention、3D LUT、临时资源和 Pass Synchronization；瓶颈须 GPU Profiling 判定。

### Q49：为什么不能每帧 glReadPixels 一整张 HDR 图做曝光？
同步 GPU Readback 会传大量数据、导致 GPU/CPU 等待。GPU Histogram/Reduction 通常更适合实时测光，CPU 需要时用异步小量 Readback。

### Q50：在 OpenGLRenderer 中如何排查“高光亮度随曝光突然跳动”？
先检查 Scene Linear HDR、Histogram、Exposure Adaptation、Pre-Exposure Ratio、Bloom Threshold、Tone Mapping，逐阶段固定/可视化数值，识别是场景光照、测光反馈还是时间域数值尺度错误。

---

## 二十五、动手验证题与参考答案

### 练习 1：sRGB 编解码数值题

给定一个线性通道 `L=0.18`，根据 sRGB 标准 Piecewise Curve 计算 `S(L)`。

由于 `0.18>0.0031308`：

$$
S(0.18)=1.055(0.18)^{1/2.4}-0.055
\approx0.46136.
$$

这说明 Linear Middle Gray 0.18 **不是** sRGB 编码值 0.18。用 CPU 和 GLSL 同时验证。

### 练习 2：计算线性 Rec.709 Luminance

`C=(2.0,1.0,0.5)`，假设线性 Rec.709 原色空间：

$$
Y=0.2126\cdot2.0+
0.7152\cdot1.0+
0.0722\cdot0.5.
$$

参考结果：

$$
\boxed{Y=1.1765.}
$$

其结果大于 1 并不违反什么限制，因为 Scene RGB 本来就是 HDR。

### 练习 3：算术平均和几何平均的差别

两个正亮度 `Y_1=1`、`Y_2=16`：

- 算术平均：`(1+16)/2=8.5`。
- 几何平均：

$$
Y_{\mathrm{geo}}=\sqrt{1\cdot16}=4.
$$

使用 `log2` 均值再反变换，检查结果也应为 4。

### 练习 4：曝光补偿 Stops

原始曝光乘数 `E=0.5`：

- `EC=+1`：`E'=1.0`。
- `EC=-1`：`E'=0.25`。
- `EC=+2`：`E'=2.0`。

解释为什么 “EV100 +1” 不能不加说明就用同一个加亮符号规则。

### 练习 5：Basic Exposure 的目标值

Metered Luminance 为 `Y=0.36`，Reference Key 为 `K=0.18`：

$$
E_{\mathrm{target}}=
\frac{0.18}{0.36}=0.5.
$$

若测光亮度减半变成 `0.18`，新目标曝光回到 `1.0`。

### 练习 6：Reinhard Tone Mapping 的两个输入

$$
T(1)=\frac{1}{2}=0.5,
\qquad
T(4)=\frac45=0.8.
$$

解释为何线性曝光倍增和 Tone-Mapped Output 不呈线性倍增。

### 练习 7：时间适应的指数平滑

已知 `E_current=1`、`E_target=4`、`s=2/s`、`dt=0.5s`，在线性 Exposure 空间：

$$
a=1-e^{-1}\approx0.6321206.
$$

$$
E_{\mathrm{new}}=1+a(4-1)\approx2.89636.
$$

再使用第九节的 C++ 代码在 Log2 Exposure Space 测试，比较两个算法的输出，说明它们不相等且都可以构成合理的时间适应方案。

### 练习 8：Pre-Exposure 与 TAA History

假设上一帧 `P_prev=2`，History Texture 颜色值为 `H_prevStored=(4,2,1)`；当前 `P_current=0.5`。

先转换 History 到当前 Pre-Exposure：

$$
H_{\mathrm{currentScale}}=
(4,2,1)\frac{0.5}{2}
=\boxed{(1,0.5,0.25)}.
$$

如果不转换便将其与当前 Color 混合，数值将相差 4 倍。

### 练习 9：对 256-Bin Histogram 验证分桶

设 `logMin=-8`、`logMax=8`、`B=256`。

对 `Y=1`：

$$
\ell=\log_2(1)=0,
\quad
b=\left\lfloor
256\frac{0-(-8)}{8-(-8)}
\right\rfloor
=128.
$$

在 GLSL 与 CPU Reference 中检查 Bucket 128 是否得到该样本。若输入为 `Y=2`，结果应落入 Bucket 144。

### 练习 10：Metering Mask 的加权几何平均

两个样本 `Y_1=1`、`Y_2=16`，权重 `w_1=3`、`w_2=1`：

$$
\overline{\ell}=
\frac{3\log_2(1)+\log_2(16)}{4}
=\frac{4}{4}=1.
$$

$$
Y_{\mathrm{weighted}}=2^1=2.
$$

它与等权几何平均 4 明显不同，说明 Metering Mask 需要进入统计过程。

### 练习 11：Bloom Threshold 与 Exposure Space

有一个 Scene Linear 亮度 `Y_scene=4`、Pre-Exposure `P=0.25`、Bloom Threshold `T_scene=2`。

- Scene Space：`4>2`，应参与 Bloom。
- Stored HDR：亮度为 `1`，若阈值也转换为 `T_stored=0.5`，仍应参与。
- 如果在 Stored HDR 上错误使用 `T=2`，该高亮会被忽略。

解释为什么 Bloom 与 Pre-Exposure 不一致会导致光晕突然出现/消失。

### 练习 12：RGBA16F 显存计算

一张 `1920×1080` 的单样本 RGBA16F 名义字节数：

$$
1920\cdot1080\cdot8=16\,588\,800\ \mathrm{bytes}.
$$

换算：

$$
\frac{16\,588\,800}{1024^2}\approx15.82\ \mathrm{MiB}.
$$

再计算同分辨率双 History Buffer 约 `31.64 MiB`，不包含压缩、额外 Mips 和分配开销。

### 练习 13：HDR 渐变、Tone Mapping 和 Banding

绘制 Linear HDR 梯度，例如从 `0` 到 `16`，分别：

1. 直接 Clamp 到 `[0,1]`。
2. 使用 Reinhard。
3. 使用 ACES-Inspired Fit。
4. 对每个输出分别执行正确 sRGB Encode。
5. 在 8-bit SDR Output 上开启/关闭 Dither。

检查高亮 Roll-Off、灰度分布、RGB Hue 变化与 Banding。

### 练习 14：Bloom 工程验证

用一个 `Emissive=10` 的灯带与 `Albedo≈1` 的白墙作为测试场景：

- 在 Linear HDR 观察两者亮度差。
- 对 Bloom Prefilter 分别使用 Hard Threshold 和 Soft Knee。
- 调整 Exposure Compensation 和 Pre-Exposure。
- 验证 Bloom Threshold 在同一尺度下对相同场景持续一致。
- 关闭 Bloom 并保留 Tone Mapping，确认发光材质的照明并未变成 GI。

### 练习 15：完整 OpenGL 后处理 Pass Profiling

固定相机与室内外亮度场景，分别记录 GPU 时间和中间 Buffer：

~~~text
HDR Scene Color
Histogram Build + Reduction
Exposure Adaptation
Bloom Downsample/Blur/Upsample
TAA / TAAU + History
Tone Mapping / Color Grading
Final Output Encoding
~~~

测量每 Pass GPU Time、Framebuffer 格式与尺寸、采样次数和 RenderDoc 中的实际颜色值。尝试单独降低 Bloom Resolution、Histogram Source Resolution，分析画质与耗时变化。

---

## 二十六、公式与高频结论速记

| 知识点 | 公式 / 结论 |
|---|---|
| HDR Scene Color | 线性场景颜色，允许大于 1，不能过早 Clamp |
| Linear Rec.709 Luminance | `Y=0.2126R+0.7152G+0.0722B` |
| Log Luminance | `ell=log2(max(Y,epsilon))` |
| Geometric Mean | `Ygeo=2^(mean(log2(Y)))` |
| Histogram Bin | `floor(B*(ell-minLog)/(maxLog-minLog))`，随后 Clamp |
| Key-Based Target Exposure | `Etarget=K/max(Ymeter,epsilon)` |
| Exposure Compensation | `Eadjusted=Ebase*2^EC` |
| EV100 | `log2((N²/t)*100/ISO)` |
| Exposure Color | `Cexposed=E*Cscene` |
| Exponential Adaptation | `a=1-exp(-speed*dt)` |
| Linear Exposure Update | `Enext=E+a*(Etarget-E)` |
| Log Exposure Update | `enext=e+a*(eTarget-e)` |
| Pre-Exposure Scene Storage | `Cstored=P*Cscene` |
| TAA History Conversion | `HcurrentScale=HprevStored*(Pcur/Pprev)` |
| Final Exposure after Pre-Exposure | `Cexposed=Cstored*(E/P)` |
| Reinhard | `T(x)=x/(1+x)` |
| ACES-Inspired Fit | `x*(2.51*x+0.03)/(x*(2.43*x+0.59)+0.14)` |
| sRGB Encode dark segment | `S=12.92*L`，小于等于 0.0031308 |
| sRGB Encode bright segment | `S=1.055*L^(1/2.4)-0.055` |
| Bloom Bright Pass | `B(C)=weight(luminance)*C` |
| Bloom Compose | `Ccombined=Cscene+strength*Bloom`（近似） |
| 3D Grading LUT | `Cout=LUT3D(Cin)`，注意 Domain |
| HDR10 | PQ / 宽色域 / HDR 显示系统，非单纯 FP16 |
| PQ vs sRGB | PQ 是 HDR 绝对亮度传输；sRGB 面向 SDR |
| RGBA16F | 名义 8 bytes / pixel（单样本） |
| Postprocess Profiling | Timestamp、Bandwith、Texture Access、Atomic |
| Histogram Workgroup | Shared Bins + Global Atomic Merge |
| Metering Mask | 在统计/桶累加时施加实际空间权重 |
| Tone Mapping != Output Encoding | 动态范围映射与目标传输编码不同 |

---

## 二十七、阅读完成后的自查清单

- [ ] 能从动态范围和场景颜色语义解释 HDR，而不是只说 RGB 可以大于 1。
- [ ] 能解释浮点 HDR Buffer 为什么不等于 HDR10 显示输出。
- [ ] 能分清 Scene-Referred Linear、Display-Linear 与 sRGB Encoded。
- [ ] 能正确配置 BaseColor sRGB Texture 与 Roughness/Metallic Data Texture。
- [ ] 能写出标准 sRGB Piecewise Encode/Decode，说明 Gamma 2.2 只是近似。
- [ ] 能区分 GL Texture sRGB Decode 与 Framebuffer sRGB Encode。
- [ ] 能针对 Rec.709 Linear RGB 计算 Luminance，并说明色域前提。
- [ ] 能推导 Geometric Mean 与 Mean Log2 Luminance 的关系。
- [ ] 能解释 Auto Exposure、Middle Gray 和 Exposure Compensation 的意义。
- [ ] 能区分加亮的 Exposure Compensation 与 EV100 数值的方向。
- [ ] 能写出 EV100 与光圈、快门、ISO 的关系。
- [ ] 能解释 Histogram 256-Bin Log Mapping 和黑色像素策略。
- [ ] 能写出 GPU Histogram Shared Memory/Atomic 合并流程。
- [ ] 能解释为何部分线程不能在 Workgroup Barrier 前提前 Return。
- [ ] 能从 Histogram Reduction 计算 Percentile / Metered Luminance。
- [ ] 能说明 Metering Mask 为什么必须进入加权统计阶段。
- [ ] 能推导线性与 Log2 Exposure Space 的时间适应公式。
- [ ] 能区分 Brighten/Darken Speed 对 Exposure 系数的影响。
- [ ] 能解释 Pre-Exposure 的原因、含义与 GPU 浮点范围关联。
- [ ] 能写出 TAA History 乘 `P_current/P_previous` 的条件和公式。
- [ ] 能解释为什么 Histogram 要还原 Scene Linear Luminance。
- [ ] 能推导 Reinhard Tone Mapping 并说明其限制。
- [ ] 能区分 ACES-Inspired Fitted Curve 与完整 ACES 色彩体系。
- [ ] 能解释 Tone Mapping 与 sRGB/PQ Output Encoding 不同。
- [ ] 能解释 Bloom、Lens Flare 和 GI 的区别。
- [ ] 能实现 Bloom Soft Knee、Downsample/Upsample Pyramid。
- [ ] 能解释 Bloom Threshold 与 Exposure/Pre-Exposure 的单位约定。
- [ ] 能解释 3D LUT 的 Domain、Interpolation 和适用阶段。
- [ ] 能说明 White Balance、Saturation 与 Gamut Mapping 的职责。
- [ ] 能解释 HDR10/PQ/HLG、HDR Swapchain 与 Paper White。
- [ ] 能搭建 OpenGL HDR FBO 与一次合法的 SDR Output Encoding。
- [ ] 能设计清晰的 Exposure/Bloom/TAA/ToneMap/ColorGrading Pass 依赖。
- [ ] 能区分 Dithering 和 Film Grain，理解 Color Banding 根源。
- [ ] 能用 GPU Timestamp/RenderDoc 分析后处理性能与颜色错误。
- [ ] 能针对 OpenGLRenderer 项目说明 Auto Exposure、Metering Mask、Bloom 与 HDR 输出的实现/测试思路。

**关联前一篇：** [08-抗锯齿技术](08-抗锯齿技术.md)。

**关联下一篇：** 10-全局光照与光线追踪（待编写）—— GI、Light Transport、路径追踪、BVH、MIS、降噪、实时 RTGI 与渲染架构。
