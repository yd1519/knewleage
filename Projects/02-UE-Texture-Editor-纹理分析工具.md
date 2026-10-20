# 项目深挖（二）：UE Texture Editor 纹理分析工具

**项目背景：** 在 Unreal Engine 5.6 Texture Editor 中扩展纹理分析能力。<br>
**功能分支：** [`feature/texture-editor-analysis-tools`](https://github.com/yd1519/UnrealEngine/tree/feature/texture-editor-analysis-tools)<br>
**功能范围：** Pixel Inspector、高倍率 Pixel Values、RGB(A) Histogram、False Color、Source Art / GPU Decoded 双数据源、Analysis Dock。<br>
**技术范围：** Unreal Editor、Slate、FCanvas、RHI、GPU Readback、Texture Source、异步任务、Shader、纹理格式与颜色空间。<br>
**代码规模：** 相对基础提交共 13 个功能提交，涉及 21 个文件，新增约 5000 行、删除约 30 行。该数字只用于理解改动范围，不代表有效工作量或个人手写量。<br>
**文档定位：** 这是一份对 AI 辅助实现代码的审读、复盘和面试准备文档，不把分支中的全部代码描述为个人独立手写。

---

## 一、先明确 AI 参与和个人贡献口径

### 1. 为什么必须把功能事实和个人贡献分开？

“这个分支实现了什么”和“这些代码是谁、以什么方式完成的”是两个不同问题。

- 功能事实可以根据代码客观描述：双数据源、GPU Readback、异步 Source Art、直方图和 False Color 确实存在。
- 个人贡献只能根据实际过程描述。如果主要代码由 AI 生成，就不能说成“我独立设计并手写了全部实现”。
- 即使代码由 AI 生成，需求拆解、约束输入、代码审查、集成、编译、对比验证和后续维护仍然可能是有效工作，但必须确实做过。

这篇文档可以帮助理解代码，却不会自动把所有技术决策变成个人原创贡献。

### 2. 推荐怎样定义这段经历？

如果实际情况是“你提出需求、让 AI 实现、持续反馈并完成验证”，可以定义为：

> AI 辅助的引擎工具开发与集成实践。我负责明确纹理分析需求和数据语义，借助 AI 生成并迭代实现，通过代码审读、模块编译和结果对比完成集成验收。

如果你只给过一次需求，之后没有检查代码、编译或验证，则不应直接使用上面的完整表述。更准确的是：

> 使用 AI 生成了 Texture Editor 纹理分析功能原型，目前正在补充代码审读、验证和技术复盘。

### 3. 文档里哪些措辞可以用？

| 情况 | 推荐措辞 | 不推荐措辞 |
|---|---|---|
| 描述客观功能 | “分支实现了”“当前代码采用” | 无需强调作者 |
| 自己提出需求并迭代 | “负责需求拆解、约束和验收” | “独立实现全部功能” |
| AI 生成主要代码 | “借助 AI 完成实现与集成” | “从零手写” |
| 自己读懂并验证代码 | “完成代码审读和关键链路验证” | “自主设计底层架构”，除非确实如此 |
| 只运行了模块编译 | “通过 TextureEditor 模块编译” | “完成充分测试” |
| 未测量效率 | “减少分析时切换外部工具” | “效率提升 XX%” |

### 4. 面试中要不要主动说 AI？

不用在自我介绍第一句话就强调，但当面试官问“你具体怎么实现”“代码是不是你写的”“AI 在其中做了什么”时，应该直接说明：

> 这套功能的主要代码是借助 AI 完成的，不是我逐行从零手写。我负责的是需求和约束输入、迭代反馈以及集成验证；之后我对关键链路做了代码复盘。下面我可以按 GPU Decoded、Source Art 和 Histogram 三条数据流具体说明。

这样的回答比模糊成“AI 只是补全”更可信。后续能否站住脚，取决于是否真正理解代码、能指出限制，并能独立定位问题。

---

## 二、这个工具解决什么问题？

### 1. 原始 Texture Editor 缺少什么？

纹理调试经常要回答这些问题：

- 鼠标所在位置的真实像素值是多少？
- 导入源图与平台构建、压缩、GPU 解码后的值是否一致？
- 高倍率观察时，能否直接看到每个 Texel 的数值？
- HDR 或数据纹理的整体数值分布如何？是否存在异常峰值、负值或极亮点？
- 纹理动态范围很大时，普通预览看不清的区域能否用伪彩色快速定位？

如果每个问题都需要导出纹理，再切换到 Tev、Photoshop 或脚本工具，分析路径会被打断。因此该分支尝试把常见纹理检查集中到 UE Texture Editor 内。

### 2. 最终版本提供了哪些能力？

| 功能 | 作用 | 数据来源 |
|---|---|---|
| Pixel Inspector | 查看鼠标位置、UV、Mip、格式和通道值 | GPU Decoded 或 Source Art |
| Pixel Values | 高倍率下直接在 Texel 内显示 RGB(A) | GPU Decoded 或 Source Art |
| Histogram | 显示 RGB(A) 分布及 Min/Mean/Max | GPU Decoded 或 Source Art |
| False Color | 将亮度映射为伪彩色，观察动态范围 | 当前预览 Shader |
| Analysis Dock | 集中放置分析开关和选项 | Editor Toolkit / Slate |

### 3. 为什么 False Color 不是第三种数据源？

False Color 是显示变换：它改变用户在 viewport 中看到的颜色，但不改变分析工具读取的底层值。

因此：

- Pixel Inspector 仍显示 Source Art 或 GPU Decoded 值。
- Histogram 仍统计选择的数据源。
- 切换 False Color 不应让数值和分布跟着变成 Turbo 色带的 RGB。

这个区分是整个工具最重要的语义边界之一。

---

## 三、功能如何逐步演进？

分支包含 13 个连续提交，主要演进顺序如下：

1. 增加初版 Pixel Inspector，异步读取 GPU 解码值。
2. 抽象 GPU Decoded / Source Art 数据源。
3. 增加高倍率 Pixel Values，并统一单点与区域采样。
4. 增加 Histogram viewport overlay。
5. 修正 Alpha 通道显示与 RGB 组合颜色。
6. 把分析数据源从单一功能设置扩展为公共设置。
7. 增加 Auto / Full Resolution 两种 Histogram 采样模式。
8. 让 Pixel Values 文本随屏幕 Texel 尺寸缩放。
9. 增加 False Color preview tonemap，并修正重复 Gamma 导致的发白。
10. 增加独立 Analysis Dock。
11. 将 Histogram 从 viewport overlay 迁移到 Analysis Dock 中的 Slate Widget。

最终状态应以分支头提交为准。仓库中的 Tev 对比文档仍描述早期 Histogram overlay 和 Small/Medium/Large 尺寸选项，这部分已经落后于最终代码。

---

## 四、整体架构与数据流

### 1. 核心模块怎样分工？

| 模块 | 职责 |
|---|---|
| `FTextureEditorToolkit` | 保存开关和模式，注册 Analysis Tab，构建 Slate 控件 |
| `FTextureEditorViewportClient` | 处理鼠标坐标、Pixel Inspector 和 Pixel Values 绘制 |
| `FTextureEditorPixelInspector` | 根据当前设置把采样请求路由到具体数据源 |
| GPU Decoded Provider | 将预览纹理绘制到临时 RT，并异步 Readback |
| Source Art Provider | 异步加载 `FTextureSource`，缓存并采样源数据 |
| `FTextureEditorHistogramAnalyzer` | 请求区域数据、异步构建 Histogram、缓存结果 |
| `STextureEditorHistogram` | 在 Analysis Dock 中绘制图表和统计值 |
| Preview Shader | 执行 Normal / False Color 显示变换 |

### 2. 三条主要数据流是什么？

#### Pixel Inspector

~~~text
Viewport 鼠标位置
    ↓
换算 UV、Preview Pixel、Mip、Layer、Slice
    ↓
FTextureEditorPixelInspector::Sample
    ↓
GPU Decoded Provider 或 Source Art Provider
    ↓
Ready / Pending / Unsupported / Failed
    ↓
格式化坐标、线性值、Stored Value、sRGB 等信息
~~~

#### Pixel Values

~~~text
计算当前 viewport 中可见的 Texel 区域
    ↓
检查 Texel 屏幕尺寸和最大可见数量
    ↓
SampleRegion
    ↓
得到区域 LinearValues
    ↓
按 Texel 绘制 RGB(A) 文本
~~~

#### Histogram

~~~text
读取当前数据源、Mip、Layer、Slice、通道和采样模式
    ↓
计算 Auto 或 Full Resolution Sample Size
    ↓
SampleRegion
    ↓
在线程池中构建 Bin、Min、Mean、Max
    ↓
缓存与 RequestId 对应的结果
    ↓
STextureEditorHistogram 在 Analysis Dock 中绘制
~~~

---

## 五、为什么要抽象 GPU Decoded 与 Source Art？

### 1. 两种数据代表什么？

| 数据源 | 含义 | 适合检查的问题 |
|---|---|---|
| GPU Decoded | 平台构建、压缩并由 GPU 格式解码后的线性结果 | 压缩误差、运行时格式、实际采样值 |
| Source Art | 导入源数据，在 Texture Build 和平台压缩之前 | 原始存储格式、Gamma、源图与构建结果差异 |

它们不是“精确”和“不精确”的关系，而是纹理生命周期中的两个阶段。

### 2. 统一请求里包含什么？

单点和区域请求都要携带能够定位当前预览内容的信息：

- Texture 与 Texture Resource。
- UV 或 Preview Rect。
- Preview Size。
- Mip Level。
- Layer Index。
- Slice Index。
- 是否为 Virtual Texture。
- 是否为单通道纹理。

统一 Result 则保存：

- 数据源类型。
- 像素坐标与图像尺寸。
- Mip、Layer、Slice。
- 线性 RGBA。
- Source Art 的原生 Stored Value、Raw Format 和 Gamma Space。
- 是否存在有效 Alpha 通道。

这样 Pixel Inspector、Pixel Values 和 Histogram 不需要分别实现两套完整取数逻辑。

### 3. 为什么需要状态枚举？

采样不是同步必定成功的函数，因此返回值区分：

- `Ready`：结果可用。
- `Pending`：加载、GPU Readback 或后台统计尚未完成。
- `Unsupported`：当前纹理类型或数据路径不支持。
- `Failed`：输入有效，但操作失败。

UI 可以显示 Loading、Calculating 或具体不支持原因，而不是把所有情况都当作零值。

### 4. 切换数据源时为什么要 Reset？

GPU Readback 和 Source Art Load 都可能跨多帧完成。如果从 GPU Decoded 切到 Source Art 后仍保留旧请求，晚到的 GPU 结果可能污染新界面。因此路由层检测数据源变化，重置旧 Provider 的缓存和 pending 状态，再把请求交给新 Provider。

---

## 六、GPU Decoded 路径怎样工作？

### 1. 为什么不直接从纹理显存读取一个像素？

GPU 纹理可能使用 BC、ASTC 等压缩格式，当前预览还涉及 Mip、Layer、Slice、单通道模式和资源类型。直接解释显存布局会把分析工具绑定到大量底层格式细节。

当前方案复用 Texture Editor 的预览绘制路径：

1. 创建临时高精度 Render Target。
2. 用 `FCanvasTileItem` 和 `FBatchedElementTexture2DPreviewParameters` 绘制目标像素或区域。
3. 由 Shader/GPU 完成纹理格式解码和对应 Mip、Layer、Slice 采样。
4. 使用 `FRHIGPUTextureReadback` 异步读回线性值。

### 2. 临时 Render Target 使用什么格式？

优先选择 `PF_A32B32G32R32F`；若平台不具备需要的 Texture2D 与 RenderTarget 能力，则回退到 `PF_FloatRGBA`。

使用浮点 RT 是为了避免先把 HDR 或负值压缩到普通 8-bit 显示范围。

### 3. 单点采样和区域采样有什么区别？

- 单点采样创建或复用 `1×1` RT，UV0 与 UV1 指向同一点。
- 区域采样创建与请求区域同尺寸的 RT，把对应 UV 矩形缩放绘制进去。
- 两者分别维护 Readback 状态，避免 Pixel Inspector 与 Pixel Values/Histogram 共用一个不匹配的结果。

### 4. 为什么要异步 Readback？

如果 Game Thread 立即等待 GPU 完成，会产生 CPU/GPU 同步点，鼠标移动时尤其容易卡住 Editor。

当前流程是：

~~~text
Game Thread 创建请求并 EnqueueCopy
    ↓
Render Thread 执行复制
    ↓
后续 Tick 中轮询 IsReady
    ↓
Lock Readback，转换 float/half-float
    ↓
通过原子状态发布完成结果
    ↓
Game Thread 下一次请求时消费结果
~~~

### 5. RequestId 解决什么问题？

RequestId 由资源指针、像素/区域、预览尺寸、Mip、Layer、Slice 和单通道状态等字段组合得到。

异步结果返回时，只有 ID 仍与当前请求一致才会被接受。例如鼠标已经移动到另一个 Texel，旧 Readback 即使完成，也不会显示成新位置的结果。

### 6. GPU 路径有哪些限制？

- 当前分析采样明确拒绝 Virtual Texture。
- 需要 SM5 和有效 Texture Resource。
- 结果表示预览路径下的 GPU Decoded 线性值，不等同于源文件字节。
- RequestId 使用普通 Hash，理论上存在碰撞可能；它是缓存一致性标识，不是安全 ID。

---

## 七、Source Art 路径怎样工作？

### 1. 为什么 Source Art 不能直接在每次鼠标移动时同步解压？

`Texture2D->Source` 可能需要读取和解压较大 Mip。若鼠标移动一次就同步执行，会阻塞 Editor。当前实现先复制一份脱离 UObject 生命周期的 Source Snapshot，再在线程池中调用 `GetMipImage`。

### 2. Source Art 缓存的 Key 是什么？

缓存依据包括：

- Texture 对象。
- `TextureSource` 的 Source ID。
- Mip Level。
- Layer Index。

Source ID 改变意味着源数据发生了变化，旧缓存不再有效。Slice 在当前 Source Art 路径中固定为 0。

### 3. Source Art 能提供哪些额外信息？

除了转成线性空间后的 `FLinearColor`，单点结果还会保留：

- Raw Image Format。
- Gamma Space。
- 原生 Stored Value。
- 存储分量数和类型，例如 UInt8、UInt16、Float16、Float32、RGBE8。

这使用户能够区分“文件中存了什么”和“转换到线性空间后是什么”。

### 4. 区域采样为什么又分同步和异步？

Source Art Mip 已经加载到内存后，小区域遍历成本很低。当前代码规定：

- 不超过 4096 个像素的区域同步构建结果，适合高倍率 Pixel Values。
- 更大的区域放到线程池中处理，适合 Histogram。

这样避免为了几十个可见 Texel 付出异步调度延迟，同时又防止全图统计阻塞 UI。

### 5. “放弃旧任务”是否真的取消了线程？

不是。旧 Future 会被移动到 Abandoned 数组，完成后再清理；代码通过 RequestId 和缓存 Key 拒绝过期结果，但后台工作本身通常不会被强制取消。

析构时会等待这些任务结束，以保证捕获数据的生命周期安全。这种做法简单可靠，但在极端大任务下可能让工具关闭过程等待一段时间。

### 6. Source Art 路径有哪些限制？

- 只支持具有有效 Source Data 的 `Texture2D`。
- 暂不支持 Virtual Texture Source Art。
- 暂不支持 Multi-block Source。
- 目标 Mip 或 Layer 不存在时返回 Unsupported。
- Source Art 坐标和 Preview 坐标可能不同，需要在 UI 中同时显示。

---

## 八、Pixel Inspector 的信息如何解释？

### 1. GPU Decoded 模式显示什么？

- Pixel 坐标与图像尺寸。
- UV。
- 当前 Mip 信息。
- GPU Linear RGB 或 RGBA。
- 转换后的 sRGB 数值或十六进制表示。

### 2. Source Art 模式显示什么？

- Preview Pixel 与 Preview Size。
- Source Art Pixel 与 Source Size。
- UV、Mip 和 Layer。
- Raw Format 与 Gamma Space。
- 原生 Stored Value。
- 转换后的 Linear RGB(A)。

### 3. 为什么两个坐标都要显示？

当前预览可能使用不同 Mip、缩放尺寸或 CPU Copy Texture。鼠标对应的是 viewport 中的 Preview Pixel，但 Source Art 真正采样的坐标由 UV 映射到源图尺寸得到。把两个坐标都显示出来，有助于排查 Mip 和尺寸映射错误。

### 4. 鼠标连续移动时为什么可能短暂显示 Pending？

GPU Readback 和 Source Art 加载都不是即时返回。每个新位置会形成新 RequestId；界面宁可显示 Reading/Loading，也不会把上一位置的值冒充成当前位置结果。

---

## 九、高倍率 Pixel Values 怎样兼顾可读性与性能？

### 1. 什么时候开始绘制？

当前代码要求屏幕上的单个 Texel 宽高都至少约 24 像素。小于该阈值时直接跳过，因为文本无法读清，而且一次会覆盖过多像素。

### 2. 为什么只采样可见区域？

放大纹理后，viewport 通常只显示整张图的一小部分。代码把屏幕可见矩形反算为 Preview Pixel Rect，只请求这些 Texel，避免为了显示几十个像素而读取整张纹理。

### 3. 为什么限制最多 4096 个可见 Texel？

即使每个值的采样已经完成，逐 Texel 格式化字符串、测量文字并提交 Canvas Draw Call 仍然有成本。超过上限时不绘制，可以防止极端窗口和缩放组合拖慢 Editor。

### 4. 文本怎样缩放？

字体高度根据当前屏幕 Texel 尺寸计算，并限制最大缩放倍数。代码还会测量每一行文本的实际宽高：如果 RGB(A) 文本放不进 Texel，就跳过该格，避免与相邻像素重叠。

### 5. Alpha 何时显示？

数据源报告纹理具有有效 Alpha 时绘制第四行；RGB-only 纹理保持三行。通道信息由纹理格式和 Source Art 内容共同判断，而不是无条件显示一个值为 1 的 Alpha。

---

## 十、Histogram 如何采样和构建？

### 1. Auto 和 Full Resolution 有什么区别？

- `Auto`：保持宽高比，将最长边限制在 512，同时保证总样本数不超过 `512×512`。
- `Full Resolution`：使用当前预览尺寸的完整分辨率。

Auto 面向交互响应，Full Resolution 面向需要精确当前预览统计的显式操作。大纹理使用 Full Resolution 会明显增加 RT、Readback、内存和统计成本。

### 2. 为什么 Histogram 不在 Slate 绘制时现场计算？

UI 的 `OnPaint` 应主要消费准备好的结果。最终实现把数据获取和统计放进 `FTextureEditorHistogramAnalyzer`：

- `Tick` 中检查配置并更新 Analyzer。
- SampleRegion 完成后，把 Bin 构建放到线程池。
- `OnPaint` 只读取有效结果并绘图。

这样 Histogram 的数据路径不再依赖 viewport canvas overlay。

### 3. Histogram 的通道如何确定？

- 单通道纹理使用 `L`。
- 普通纹理根据当前 RGBA Channel Mask 选择可见通道。
- 没有有效 Alpha 时不加入 A。
- RGB 同时存在时，在 Slate 图表中用组合颜色表达重叠关系。

### 4. 为什么使用 400 个 Bin？

400 是固定的表现与开销折中。更多 Bin 能显示更细分布，但会增加统计数组和绘图成本；更少 Bin 则会损失 HDR 分布细节。

它不是由纹理位深自动决定，也不意味着 8-bit 纹理的每个整数值恰好对应一个 Bin。

### 5. 为什么 x 轴使用 Symmetric Log？

HDR 或数据纹理可能同时包含：

- 很接近零的小值。
- 远大于 1 的高亮值。
- 负值。

普通线性轴容易被极端值拉伸；普通 `log(x)` 又不能处理负值。代码使用带小常数的对称对数变换：

$$
S(x)=
\begin{cases}
\ln(x+\epsilon)-\ln\epsilon, & x>0 \\
-\left(\ln(-x+\epsilon)-\ln\epsilon\right), & x\le 0
\end{cases}
$$

其中 `ε = 0.001`。它在零点两侧保持符号，并压缩大幅值范围。

### 6. Min、Mean、Max 是怎样计算的？

对当前启用通道的全部有限样本统一统计：

- 忽略 NaN 和 Infinity。
- Minimum 是所有有效通道样本的最小值。
- Maximum 是最大值。
- Mean 是所有有效通道样本的总和除以数量。

这不是每通道分别给出一组 Min/Mean/Max，而是当前显示通道的合并摘要。

### 7. 为什么还要按 Bin 宽度做归一化？

Symmetric Log 轴映射回原始值域后，每个 Bin 覆盖的原始数值宽度不同。如果只画原始计数，宽区间天然更容易收集更多样本。代码会除以对应原始值域 Bin Width，近似形成密度，再根据排除少量极端峰值后的高分位进行显示归一化。

这一步影响图表高度，但 `BinCounts` 和 `ChannelSampleCounts` 仍保留原始计数。

### 8. Histogram 统计的是可见颜色吗？

不是。当前默认语义是所选 Analysis Data Source 的 Raw/Linear Analysis Values：

- 不统计 False Color 后的 Turbo RGB。
- 透明像素中的隐藏 RGB 仍可能进入统计。
- RGB 默认没有按 Alpha 加权。
- 不执行 Tev 的完整 Inspection Color Space 或 Error Metric 语义。

这对排查透明边缘 RGB、Mip Bleeding 和压缩数据很有价值，但与“人眼当前看到的图像分布”不是同一个问题。

---

## 十一、False Color 如何工作？

### 1. 映射公式是什么？

Shader 先计算 RGB 平均值：

$$
L = \max\left(\frac{R+G+B}{3}, 0\right)
$$

再映射到色带位置：

$$
\boxed{
t = \frac{\log_2(L+0.03125)}{10}+0.5
}
$$

最后将 `t` 限制到 `[0,1]`，通过 Turbo 多项式近似得到显示颜色。

### 2. 为什么选择对数亮度？

线性映射在 HDR 范围中很难同时看清暗部和高亮。对数映射能把多个数量级压缩到有限色带，使异常亮点、过暗区域和动态范围分布更容易辨认。

### 3. 为什么 False Color 后要跳过普通 Gamma Correction？

Turbo 色带输出已经是希望显示的颜色。如果之后仍走普通 Texture Editor Gamma 分支，色带会再次被抬亮，表现为发白，与参考工具差异明显。

最终 Shader 使用互斥分支：

~~~text
if FalseColor:
    RGB = FalseColor(RGB)
else if Gamma != 1:
    RGB = ApplyGammaCorrection(RGB)
~~~

普通预览路径保持不变，False Color 则直接输出色带结果。

### 4. 与 Tev 为什么仍可能存在差异？

- Tev 可使用 Colormap Texture/LUT，当前 UE 使用 Turbo 多项式近似。
- 两者的背景、Alpha 合成和 Inspection Color Space 不完全相同。
- UE 输入还受当前 Mip、Layer、Slice、Texture Build 和 GPU Decode 影响。

因此更准确的名称是 Tev-like False Color，而不是“与 Tev 逐像素完全一致”。

---

## 十二、Analysis Dock 如何接入 UE Editor？

### 1. 为什么从菜单和 overlay 迁移到 Dock？

分析开关较多，如果全部放在 View Options 菜单中：

- 用户每次调整都要重复打开菜单。
- Viewport 显示设置和数据分析设置混在一起。
- Histogram overlay 会遮挡纹理内容，大小受 viewport 影响。

最终版本注册独立 `Analysis` Tab，与 Details、Oodle、Platforms 等面板并列。

### 2. 面板分成哪些区域？

- Preview Tonemap：Normal / False Color。
- Analysis Data Source：GPU Decoded / Source Art。
- Tools：Pixel Inspector、Pixel Values、Histogram。
- Histogram Sampling：Auto / Full Resolution。
- Histogram Widget：数据源、采样尺寸、图表及 Min/Mean/Max。

### 3. 为什么 Histogram 使用独立 Slate Widget？

`STextureEditorHistogram` 只持有 Toolkit 的弱引用和 Analyzer 指针：

- `Tick` 请求 Analyzer 更新数据。
- `OnPaint` 根据 Ready/Pending/Disabled 状态绘制。
- 面板宽高直接决定图表区域，不再需要 Small/Medium/Large overlay 尺寸。

这样分析逻辑与 UI 绘制职责更清晰。

---

## 十三、异步状态与生命周期怎样保证安全？

### 1. UObject 为什么需要引用收集？

临时 Render Target 由分析 Provider 持有，但 Provider 本身不是 UObject。它通过 `AddReferencedObjects` 把 RenderTarget 添加到 GC 引用集合，避免 GPU Readback 尚未结束时对象被回收。

### 2. GPU 结果如何跨线程发布？

Readback 对象使用线程安全 Shared Pointer，并用原子变量记录：

- 是否已安排 Poll。
- 是否完成。
- 是否成功。

Render Thread 写入结果后，以 Release 顺序发布完成状态；Game Thread 用 Acquire 读取，保证看到与完成标记对应的数据。

### 3. 为什么不能只看 Future 是否完成？

完成只表示某个旧任务结束，不表示它仍对应当前纹理和设置。代码还要比较 RequestId、Source ID、Mip、Layer 等 Key，才能决定是否接纳结果。

### 4. 销毁工具时为什么要等待 Abandoned Future？

后台 Lambda 捕获了 Source Snapshot、FImage 或其它数据。析构时等待它们结束，可以避免对象和任务生命周期交错导致悬空引用。代价是长任务可能让关闭工具等待。

---

## 十四、当前实现与 Tev 的语义差异

### 1. 产品定位不同

Tev 是独立图像查看和比较工具，更关注当前 Inspection Space、Channel Group、Error Metric 和可见解释。

UE Texture Editor 是资产编辑器，除了视觉检查，还要回答：

- Source Art 与平台构建结果是否一致？
- GPU 压缩和解码是否改变数值？
- 透明区域隐藏 RGB 是否可能引起边缘污染？
- 当前 Mip、Layer、Slice 的实际数据是什么？

因此该分支优先保留资产数据语义，而不是完全复制 Tev 的显示语义。

### 2. Histogram 的关键差异

| 维度 | 当前 UE 实现 | Tev 常见语义 |
|---|---|---|
| 数据来源 | GPU Decoded 或 Source Art | 当前 Image / Inspection 结果 |
| Alpha | 隐藏 RGB 仍参与，默认不按 Alpha 加权 | 可能 Unpremultiply 并按 Alpha 加权 |
| 颜色空间 | 数据源线性值 | Inspection Chroma / Transfer Function |
| False Color | 不影响 Histogram | 取决于当前检查模式 |
| 采样 | Auto 降采样或 Full Resolution | 更偏向当前图像统计 |

### 3. 是否应该完全改成 Tev 语义？

不建议直接替换默认行为。更合理的扩展是增加明确模式：

- Raw Values：保留当前资产调试语义。
- Alpha Weighted：按 Alpha 对 RGB Bin 加权。
- Visible Values：进一步考虑 Unpremultiply、颜色空间和显示变换。

不同模式必须明确命名，否则用户会误以为自己正在查看源数据，实际却已经被显示语义改变。

---

## 十五、性能与正确性分析

### 1. 哪些工作仍发生在主线程或 UI 路径？

- 计算鼠标和可见 Texel 范围。
- 创建请求和检查缓存状态。
- 小于等于 4096 像素的 Source Art 区域采样。
- Pixel Values 字符串格式化和 Canvas 绘制。
- Slate Histogram 图表绘制。

### 2. 哪些工作被异步化？

- GPU Texture Readback。
- Source Art Mip 加载。
- 大 Source Art 区域采样。
- Histogram Bin 和统计值构建。

### 3. Auto Sampling 为什么很重要？

对 8K 纹理做 Full Resolution Histogram 需要处理约 6700 万像素；即使后台统计不阻塞 Game Thread，RT 分配、GPU 绘制、Readback 数据量和内存占用仍然很高。Auto 将样本限制在约 26 万，可显著降低交互成本。

### 4. 还可以怎样优化？

- 对 Full Resolution 增加显式警告或任务进度。
- 复用不同尺寸的 RT，而不是尺寸变化时频繁重建。
- 对 Histogram 使用 GPU Compute 直接累加 Bin，避免全图 Readback。
- Pixel Values 按可见区域变化增量更新。
- 为高频鼠标移动增加节流或只保留最新请求。
- 给异步任务增加真正的取消令牌，而不只是丢弃结果。

---

## 十六、代码审读后发现的边界与技术债

### 1. 最终 Histogram 已迁移到 Dock，但旧 overlay 代码仍存在

分支头已经移除了 viewport 中对 `DrawHistogram` 的调用，但 `FTextureEditorViewportClient` 中仍保留：

- 旧 Histogram 常量。
- `DrawHistogram` 方法。
- Histogram State 成员和相关声明。

这些代码可能已经成为不可达的遗留实现。后续应确认没有命令或隐藏路径引用后删除，减少两套实现并存造成的维护误导。

### 2. 对比文档已经部分过期

仓库根目录的 `TextureEditor_Analysis_Tools_Tev_Comparison.md` 仍写着 Histogram viewport overlay 和 Small/Medium/Large 尺寸。最终提交已经改为 Dock Widget，并不再显示这些尺寸选项。

技术文档需要跟随最终架构更新，否则读者会根据旧行为理解当前代码。

### 3. Full Resolution 缺少强约束

Full Resolution 是显式选择，但当前设计主要依靠用户判断成本。超大 HDR 纹理可能产生较大的 GPU/CPU 数据和等待时间。可以增加：

- 像素数提示。
- 内存估算。
- 确认操作。
- 最大上限或分块统计。

### 4. 测试以编译验证为主

提交记录显示曾执行 TextureEditor 模块的 Windows 或 Mac 编译；但改动中没有看到新增的专用自动化测试文件。

因此可以说“完成模块编译验证”，不能据此说：

- 所有纹理格式都验证通过。
- 所有平台行为一致。
- 没有异步生命周期问题。
- 已建立完整自动化回归。

### 5. AI 生成代码为什么更需要这一章？

AI 很容易持续叠加功能，却不主动删除旧实现、同步文档或建立测试。这里残留的 overlay 代码和过期对比文档正是典型风险。

能够指出这些问题，说明复盘不只是把代码翻译成中文，而是在检查最终状态是否一致。

---

## 十七、应该怎样补齐验证？

### 1. 最小功能矩阵

| 维度 | 建议样本 |
|---|---|
| 纹理类型 | Texture2D、RenderTarget2D、单通道、RGBA、HDR |
| 数据源 | GPU Decoded、Source Art |
| 格式 | G8、BGRA8、RGBA16F、RGBA32F、BC 系列 |
| 颜色空间 | Linear、sRGB、Pow22 |
| 视图 | 不同 Mip、Layer、Slice、Channel Mask |
| Alpha | 无 Alpha、全 1、渐变、透明区隐藏 RGB |
| 尺寸 | 小图、非方形图、4K、8K |

### 2. Pixel Inspector 应验证什么？

- UV 到像素坐标的边界映射。
- 左上角、右下角和纹理外部。
- 快速移动鼠标时是否出现旧结果串位。
- Mip、Layer、Slice 切换后缓存是否失效。
- Source Art Stored Value 与 Linear Value 是否符合格式和 Gamma。

### 3. Histogram 应验证什么？

- 常量纹理是否只在一个区域出现峰值。
- 已知渐变的分布是否符合预期。
- 负值、零和大于 1 的 HDR 样本是否落入正确 Bin。
- NaN/Infinity 是否被忽略。
- Auto 与 Full Resolution 的 Min/Mean/Max 差异是否可解释。
- RGB Channel Mask 和 Alpha 检测是否正确。
- 数据源、Mip 或通道变化后旧异步结果是否被拒绝。

### 4. False Color 应验证什么？

- 与公式计算的关键亮度点对应。
- 切回 Normal 后原预览完全恢复。
- False Color 不改变 Pixel Inspector 和 Histogram。
- sRGB 与 Linear Texture 的输入语义明确。
- Alpha、Checkerboard 和背景合成符合预期。

### 5. 性能和生命周期应验证什么？

- 连续移动鼠标时 Editor 帧时间。
- Auto 与 Full Resolution 的耗时和内存。
- 快速切换纹理、Mip 和数据源。
- 任务尚未完成时关闭 Texture Editor。
- Texture Reimport 或 Recode 后缓存失效。

---

## 十八、简历应该怎么写？

### 1. 推荐的诚实版本

如果你确实负责需求、迭代、审读、编译和效果验证，可以写：

> 围绕 UE Texture Editor 纹理分析需求，借助 AI 完成 Pixel Inspector、高倍率 Pixel Values、RGB(A) Histogram 与 False Color 的原型开发和编辑器集成；负责需求拆解、数据语义约束、迭代反馈、模块编译与效果验收。

> 梳理 Source Art / GPU Decoded 两类分析语义，验证异步 Source Art 加载与 GPU Readback 取数路径，使 Inspector、Pixel Values 和 Histogram 能够复用统一采样接口，并区分底层分析值与 False Color 显示变换。

### 2. 如果目前还没有完成充分审读

应该先降级描述：

> 使用 AI 搭建 UE Texture Editor 纹理分析工具原型，覆盖 Pixel Inspector、Pixel Values、Histogram 和 False Color；正在通过源码审读、模块编译和对比测试补齐实现理解与验证。

在真正补齐前，不建议使用“设计并实现双数据源抽象”“解决线程生命周期”等强归因措辞。

### 3. 简历里必须写“AI”吗？

从诚信角度，关键不是机械地在每条后面加“AI”，而是不能制造个人独立实现的错误印象。

如果代码主要由 AI 生成，而你的职责确实是 AI 协作开发，写出“借助 AI 完成”更稳。特别是目标岗位明确关注 AI 工作流时，这反而能成为正面经历。

### 4. 哪些原简历措辞需要调整？

原本类似：

> 设计 Source Art / GPU Decoded 双数据源抽象，采用异步 Source Art 加载与采样、GPU Readback 获取纹理数据……

如果该设计并非你独立提出，应改为：

> 梳理并验证 Source Art / GPU Decoded 双数据源方案，借助 AI 完成异步 Source Art 加载与 GPU Readback 取数路径的集成……

“设计并实现”强调原创设计和直接编码；“梳理、约束、验证、集成”更符合 AI 主导生成而个人负责落地的情况。

---

## 十九、面试中怎样介绍？

### 1. 30 秒版本

> 这项工作是在 UE Texture Editor 里补充纹理分析能力，包括 Pixel Inspector、高倍率逐像素数值、Histogram 和 False Color，并区分 Source Art 与 GPU Decoded 两种数据。我主要通过 AI 协作完成代码实现，负责明确功能和数据语义、持续反馈、编译集成和结果检查。之后我重点复盘了 GPU Readback、Source Art 异步加载以及 Histogram 的缓存和过期结果处理。

### 2. 90 秒版本

> 这个工具要解决的是纹理调试时频繁切换外部工具的问题。最终在 Texture Editor 中加入了 Pixel Inspector、Pixel Values、Histogram 和 False Color，并单独建立了 Analysis Dock。
>
> 数据层分成 GPU Decoded 和 Source Art。GPU 路径复用预览 Shader，把目标 Mip、Layer、Slice 绘制到浮点 Render Target，再通过 FRHIGPUTextureReadback 异步读回；Source Art 路径复制 Texture Source Snapshot，在线程池中加载 Mip，并按 Source ID、Mip 和 Layer 缓存。单点和区域采样共用接口，Pixel Inspector、Pixel Values 和 Histogram 都从这里取数。
>
> Histogram 默认最多均匀采样到 512×512，也支持 Full Resolution；统计放在线程池，Slate 面板只绘制缓存结果。False Color 是独立的预览变换，不会改变分析值。
>
> 这套代码的主要实现是借助 AI 完成的，我不会把它说成全部独立手写。我的责任是需求和约束、迭代反馈、集成验证，以及把生成代码读懂并发现遗留 overlay 和文档过期等问题。

### 3. 面试官问“那你到底会不会写”时怎么回答？

不要辩解“AI 写也等于我写”。可以直接回答：

> 这个版本我没有逐行从零手写，所以我不会这样声称。我的优势是能把需求拆成可验证的数据语义和链路，也能对 AI 结果做代码审读和集成。为了证明不是只会生成，我可以具体解释 RequestId 如何防止旧 Readback 串位、Source Art 为什么要 CopyTornOff，以及当前还残留哪些需要清理的旧代码。如果岗位要求现场编码，我会按题目独立完成。

---

## 二十、高频面试问题与回答提纲

### 1. Source Art 与 GPU Decoded 的区别是什么？

> Source Art 是导入源数据，处于 Texture Build 和平台压缩之前；GPU Decoded 是运行平台纹理经过 GPU 格式解码后得到的线性值。前者适合检查源格式、Stored Value 和 Gamma，后者适合检查压缩及运行时实际采样结果。

### 2. 为什么 GPU Readback 会导致卡顿？当前怎样避免？

> CPU 若立即等待 GPU 完成，会形成同步点。当前先 Enqueue Copy，后续 Tick 轮询 Readback 是否 Ready，完成后才 Lock 数据。UI 在等待期间显示 Pending，不阻塞当前帧。

### 3. 为什么不直接调用同步的 ReadPixels？

> 同步读取会迫使 CPU 等 GPU；而且这里需要频繁响应鼠标和视图变化。FRHIGPUTextureReadback 更适合跨帧获取结果，并能显式管理 Render Thread 生命周期。

### 4. 如何防止鼠标移动后显示上一像素的数据？

> 请求由资源、坐标、尺寸、Mip、Layer、Slice 等字段生成 RequestId。异步结果完成后只有 ID 与当前请求一致才会被接受，否则丢弃。

### 5. Source Art 为什么要 `CopyTornOff`？

> 后台线程不应直接依赖随 UObject 状态变化的 Source 对象。复制一份脱离原对象生命周期的 Snapshot，再在线程池读取 Mip，可以降低异步访问和对象变化之间的风险。

### 6. Histogram 为什么使用 Symmetric Log？

> 数据可能同时有负值、零附近小值和大于 1 的 HDR 值。线性轴容易被极值拉伸，普通 Log 又不能处理负数；Symmetric Log 在零点两侧保持符号，并压缩数量级。

### 7. Histogram 为什么不统计 False Color？

> False Color 是显示变换，Histogram 是底层数据分析。若统计变换后的色带，只能得到 Turbo RGB 的分布，反而失去纹理原始数值信息。

### 8. Auto Histogram 是局部区域还是全图？

> 是当前预览全图的均匀降采样，不是只统计 viewport 局部。最长边和总样本数约束在 512 与 512×512 内。

### 9. 当前方案最大的性能风险是什么？

> Full Resolution Histogram 对大纹理需要很大的 GPU 绘制、Readback 和 CPU 统计数据。它虽然异步，但仍会消耗显存、带宽和内存。后续可以使用 Compute Shader 在 GPU 上直接构建 Bin，或对超大纹理分块统计。

### 10. 当前方案最大的工程问题是什么？

> 最终 Histogram 已迁移到 Dock，但 ViewportClient 中仍保留旧 overlay 的常量、状态和绘制方法；对比文档也没有同步。这说明功能迭代完成后缺少一次彻底的收尾重构和文档校验。

### 11. AI 在这项工作里犯过什么典型问题？

> 典型问题不是语法错误，而是迭代式叠加：新 Dock 已经替代旧 overlay，但旧代码没有全部删除，文档也保留旧行为。AI 能快速铺功能，但最终一致性、测试矩阵和删除无效路径仍需要人工负责。

### 12. 如果让你现在独立继续开发，你会先做什么？

> 我会先删除或确认旧 Histogram overlay 路径，更新对比文档，然后建立已知输入纹理的自动化或半自动回归，包括常量、渐变、HDR、负值和 Alpha 隐藏 RGB。完成基线后，再考虑 GPU Histogram 和 Alpha Weighted 模式。

---

## 二十一、容易说错的口径

| 不建议这样说 | 更准确的说法 |
|---|---|
| 我独立手写了全部 Texture Editor 扩展 | 主要代码借助 AI 完成，个人负责部分按实际说明 |
| Source Art 是未压缩纹理 | 它是导入源数据，位于平台构建和压缩之前，仍有自身 Raw Format 与 Gamma |
| GPU Decoded 就是屏幕最终颜色 | 它是预览采样路径得到的线性解码值，不等同于最终显示颜色 |
| False Color 是一种 Analysis Data Source | False Color 是 Preview Transform，分析数据源仍是 GPU Decoded 或 Source Art |
| 异步就完全没有性能成本 | 它避免主线程同步等待，但仍有 GPU、带宽、内存和后台计算成本 |
| Abandon 会取消后台任务 | 当前主要是丢弃过期结果，任务本身通常继续到完成 |
| Auto Histogram 统计当前可见区域 | Auto 对当前预览全图均匀降采样 |
| Histogram 与 Tev 完全一致 | 当前默认是 Raw Analysis Data 语义，Alpha 和颜色空间处理与 Tev 不同 |
| 通过模块编译就说明功能完全正确 | 编译只验证构建，不替代格式、平台、异步和数值测试 |
| 分支有 5000 行，所以工作量很大 | 行数包含 AI 生成和遗留代码，只用于描述改动范围 |

---

## 二十二、源码复习索引

### 1. 接口与工具状态

- `Engine/Source/Editor/TextureEditor/Public/Interfaces/ITextureEditorToolkit.h`
- `Engine/Source/Editor/TextureEditor/Private/TextureEditorToolkit.h`
- `Engine/Source/Editor/TextureEditor/Private/TextureEditorToolkit.cpp`

### 2. 数据源与异步采样

- `Engine/Source/Editor/TextureEditor/Private/Models/TextureEditorAnalysisDataSource.h`
- `Engine/Source/Editor/TextureEditor/Private/Models/TextureEditorAnalysisDataSource.cpp`

### 3. Pixel Inspector 与 Pixel Values

- `Engine/Source/Editor/TextureEditor/Private/Models/TextureEditorViewportClient.h`
- `Engine/Source/Editor/TextureEditor/Private/Models/TextureEditorViewportClient.cpp`
- `Engine/Source/Editor/TextureEditor/Private/Widgets/STextureEditorViewport.h`
- `Engine/Source/Editor/TextureEditor/Private/Widgets/STextureEditorViewport.cpp`

### 4. Histogram

- `Engine/Source/Editor/TextureEditor/Private/Models/TextureEditorHistogramAnalysis.h`
- `Engine/Source/Editor/TextureEditor/Private/Models/TextureEditorHistogramAnalysis.cpp`
- `Engine/Source/Editor/TextureEditor/Private/Widgets/STextureEditorHistogram.h`
- `Engine/Source/Editor/TextureEditor/Private/Widgets/STextureEditorHistogram.cpp`

### 5. False Color Preview

- `Engine/Source/Editor/UnrealEd/Public/Texture2DPreview.h`
- `Engine/Source/Editor/UnrealEd/Private/Texture2DPreview.cpp`
- `Engine/Shaders/Private/SimpleElementTexture2DPreviewPixelShader.usf`

### 6. 命令与菜单

- `Engine/Source/Editor/TextureEditor/Private/Models/TextureEditorCommands.h`
- `Engine/Source/Editor/TextureEditor/Private/Models/TextureEditorCommands.cpp`
- `Engine/Source/Editor/TextureEditor/Private/Menus/TextureEditorViewOptionsMenu.h`

---

## 二十三、面试前必须补齐的能力

- [ ] 能独立画出 GPU Decoded 从 Canvas 到 Readback 的完整线程流程。
- [ ] 能解释 Source Art Snapshot、Source ID 和缓存失效。
- [ ] 能说明 Pixel Inspector 与 Pixel Values 为什么分别使用单点和区域采样。
- [ ] 能解释 RequestId 如何拒绝过期异步结果。
- [ ] 能推导 False Color 的亮度映射，并说明为什么跳过普通 Gamma。
- [ ] 能解释 Symmetric Log、Bin Width Density 和显示归一化。
- [ ] 能区分 Raw Analysis Values 与 Visible/Tev Inspection Values。
- [ ] 能指出 Virtual Texture、Multi-block Source 和 Full Resolution 的限制。
- [ ] 能说明分支目前缺少哪些自动化测试。
- [ ] 能指出旧 Histogram overlay 和过期文档等技术债。
- [ ] 能在面试中如实说明 AI 的参与，不把生成代码包装成独立手写。
- [ ] 至少选择一个小改动，在不依赖 AI 直接给出完整补丁的情况下自行完成并验证。

当这些问题能够独立回答，并且可以亲自修改、编译和验证至少一条链路后，这段经历才真正从“AI 生成的功能”转化为“自己能够负责和维护的工程经验”。
