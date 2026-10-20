# 计算机图形学秋招复习文档（六）：PBR 与 IBL

**复习范围：** 微表面理论、Cook–Torrance BRDF、Fresnel、GGX、Smith 几何项、Metallic–Roughness、Disney Principled BRDF、直接光、HDR 环境贴图、Diffuse/Specular IBL、GGX 重要性采样、Split-Sum、BRDF LUT、GLSL 工程实践及性能优化。

**面试目标：** 能手写和解释 Cook–Torrance 的 D/F/G；独立推导 GGX 采样 PDF 与 BRDF LUT；解释为什么要同时生成 Irradiance Cubemap、Prefilter Cubemap、BRDF LUT；能够在 OpenGLRenderer 中实现并验证一条完整 PBR + IBL 管线。

**前置知识：** [01-数学基础与空间变换](01-数学基础与空间变换.md)、[04-纹理与采样](04-纹理与采样.md)、[05-光照模型与渲染方程](05-光照模型与渲染方程.md)。

**排版兼容说明：** 正文简短表达式使用行内代码，较复杂推导使用独立的数学公式块；表格中不嵌入 LaTeX，避免不支持行内公式的 Markdown 阅读器显示原始代码。

**本篇统一约定：**

- `N`：宏观单位法线；`V`：从表面点指向观察者；`L`：从表面点指向光源的单位方向；`H=normalize(V+L)`：Half Vector。所有方向都在同一空间。
- `NoV=max(dot(N,V),0)`、`NoL=max(dot(N,L),0)`、`NoH=max(dot(N,H),0)`、`VoH=max(dot(V,H),0)`。
- 材质输入粗糙度 `r=roughness`，本篇采用常见的 GGX 参数映射 `alpha=r²`。因此 GGX NDF 中的 `alpha²` 等于 `r⁴`。不同论文或 Shader 的命名可能不同。
- `F0` 指法线正入射时的镜面反射率；本篇常用介质近似 `F0=0.04`，金属 `F0≈baseColor`。
- 所有光照运算使用 Linear RGB 和 HDR Radiance；Albedo 颜色纹理应正确解码 sRGB，Metallic/Roughness/Normal 等数据纹理通常不进行 sRGB 解码。
- 微表面模型以**不透明、单次散射、各向同性 GGX** 为基础；多次散射、透明、Clear Coat、Sheen、次表面和各向异性是进一步拓展。
- 对点光/方向光，使用对应的离散光源贡献；对环境光，按连续方向的 Radiance 积分。不能无视 Radiometry 单位差别。

---

## 一、什么是 PBR？

### 1. 为什么 PBR 不等于“使用 Metallic/Roughness 纹理”？

PBR 是一组面向物理一致性的材质与光照设计原则。核心包括能量守恒、互易性、具有较明确含义的材质参数、统一的线性光照计算和适当的光传输模型。

实际实时 PBR 往往包含经验修正、预积分、稀疏采样及近似，因此“Physically Based”不意味着整个渲染器在数学上完全无偏。

### 2. PBR 和传统 Blinn–Phong 的核心差别？

传统 Blinn–Phong 的经验高光指数和强度可被任意调节，Diffuse/Specular 分配未必守恒。PBR Microfacet BRDF 使用 NDF、Fresnel、Geometry 共同约束高光方向分布及能量比例。

同样的 PBR 材质放到不同环境时通常更容易保持视觉与参数一致性，但前提是输入光照、材质空间、颜色空间及曝光正确。

### 3. PBR 的能量守恒条件是什么？

对被动反射材质，固定一个入射方向：

$$
\int_{\Omega^+}
f_r(\omega_i,\omega_o)
(\mathbf N\cdot\omega_o)\,\mathrm d\omega_o
\le1.
$$

这是**半球积分约束**，不是要求每个方向的 BRDF 数值都不超过 1。若包括透射还应计算其它能量通道。

### 4. Direct Lighting、IBL、GI 与 PBR 分别负责什么？

- **BRDF/PBR：** 表面怎样对入射光进行反射/散射。
- **Direct Lighting：** 显式光源到当前表面的贡献。
- **IBL（Image-Based Lighting）：** 让环境图中的方向 Radiance 参与材质反射积分。
- **GI：** 包括其他表面反弹、传播的间接光贡献。

IBL 不是独立于 PBR 的材质模型，而是 BRDF 的一种照明输入来源。

---

## 二、微表面理论与 Half Vector

### 1. 什么是 Microfacet Model？

假设宏观表面由许多微小、近似镜面的微平面构成：

- 平滑表面的微法线集中，高光尖锐。
- 粗糙表面的微法线方向分布很广，高光分散。
- 不同微面之间可能互相遮蔽（Masking/Shadowing）。

宏观材质的 BRDF 可从微面分布和统计可见性推导。

### 2. Half Vector 为什么是参与镜面反射的微面法线？

在本篇 `L,V` 都从表面向外指的约定下，镜面反射的微面法线平分它们之间的夹角：

$$
\boxed{
\mathbf H=\frac{\mathbf L+\mathbf V}{\|\mathbf L+\mathbf V\|}.
}
$$

当 `L≈-V` 时分母趋近零，工程实现需要提前排除退化方向，而不是无条件 Normalize。

### 3. NDF 是什么？

Normal Distribution Function `D(H)` 描述单位宏观投影面积上具有某个微法线朝向的微面面积密度。常见归一化条件：

$$
\boxed{
\int_{\Omega^+}D(\mathbf H)
(\mathbf N\cdot\mathbf H)\,\mathrm d\omega_h=1.
}
$$

因此**按 NDF 采样微法线 H**时，常用立体角 PDF：

$$
p_H(\mathbf H)=D(\mathbf H)(\mathbf N\cdot\mathbf H).
$$

面试易错：`D` 本身一般不是直接归一化的方向 PDF，不能漏掉 `NoH`。

### 4. Roughness 是几何顶点起伏幅度吗？

不是。材质 Roughness 控制微表面法线统计分布，并不直接改变宏观网格顶点；Normal Map、Displacement 和 Roughness 各有自己的作用与尺度。

### 5. 为什么 Roughness=0 时普通 GGX 数值实现不可靠？

理想镜面极限应趋向 Delta 分布，而普通浮点实现的 NDF 分母可能极小甚至产生数值异常。实时 Shader 往往对 Roughness 取一个小的下限，这是工程近似，不能等同于真正的完美镜面。

---

## 三、Cook–Torrance BRDF 的 D、F、G

### 1. Cook–Torrance 镜面项怎么写？

$$
\boxed{
f_{\mathrm{spec}}(\mathbf L,\mathbf V)=
\frac{
D(\mathbf N,\mathbf H)\,
F(\mathbf V,\mathbf H)\,
G(\mathbf N,\mathbf V,\mathbf L)
}{
4(\mathbf N\cdot\mathbf V)(\mathbf N\cdot\mathbf L)
}.
}
$$

定义在有效的上半球；数值实现需防止分母过小。

| 项 | 名称 | 回答的问题 |
|---|---|---|
| D | Normal Distribution Function | 有多少微面能以 H 方向把 L 反射到 V |
| F | Fresnel | 单个微面在该角度会反射多少光 |
| G | Geometry/Visibility | 参与反射的微面是否被其他微面遮蔽 |
| 分母 | 4 NoV NoL | 微表面几何和变量变换的归一化 |

### 2. 为什么不能只用 D 当镜面高光？

D 只有微法线方向分布，没有 Fresnel 的角度相关反射比例和微面的 Masking/Shadowing；简单将 NDF 当作完整 BRDF 会得到错误能量和掠射行为。

### 3. 为什么分母有 4 NoV NoL？

它来自微法线面积统计向宏观 BRDF 的转换，包含观察与入射方向的投影几何及镜面反射变量映射，不是美术调节用的魔法常量。

### 4. G 与场景 Shadow Map 是一回事吗？

不是。G 是微观几何遮蔽，属于 BRDF；Shadow Map/RT Shadow 是宏观光路可见性，属于 Light Transport。直接光照可能同时乘两者，但不能互相替代。

### 5. 为什么单次散射 Cook–Torrance 对粗糙金属可能偏暗？

部分光在微面间被遮挡后仍可能多次反射再离开。单次散射 GGX 没有完整建模此项。高质量材质可使用 Multiple-Scattering Energy Compensation，不能声称基础 DFG 公式已经完整模拟所有多次微反射。

---

## 四、Fresnel 与 F0

### 1. 为什么玻璃侧看比正面看反射更明显？

Fresnel 描述界面反射比例随入射角变化。常见介质在掠射角的反射趋近强烈，因此即使透明介质正面反射很弱，斜角也可能有明显高光。

### 2. 常见电介质的 F0 如何由折射率求得？

对无吸收、非磁性且折射率可用实数表示的两个介质，在法线入射时：

$$
\boxed{
F_0=\left(\frac{\eta_1-\eta_2}{\eta_1+\eta_2}\right)^2.
}
$$

空气到折射率约 1.5 的介质：

$$
F_0=\left(\frac{1-1.5}{1+1.5}\right)^2=0.04.
$$

因此 0.04 是常见介质的工作流近似，不是所有非金属的严格常数。

### 3. Schlick Fresnel 的公式是什么？

令 `c=clamp(VdotH,0,1)`：

$$
\boxed{
F(c)=F_0+(1-F_0)(1-c)^5.
}
$$

`c=1` 时 `F=F0`；`c=0` 时 `F=1`。RGB 实现中 `F0` 可以是向量，逐通道计算。

### 4. 为什么要用 VdotH，而不是 NdotV？

微面反射界面的真实法线是 `H`，入射相对于该微面法线的角度由 `V·H`（镜面时也等于 `L·H`）描述。`N·V` 反映宏观视角，不能无条件替代。

### 5. 为什么金属 F0 可以有颜色？

金属反射由复折射率（包含吸收相关参数）描述，不同波长的 Fresnel 响应不同。工作流中用 Base Color 近似金属 F0，但并不等同于完整测量/重建金属的复折射率。

### 6. Fresnel 掠射趋向 1，是否代表粗糙金属必定产生白色边缘？

不能这样断言。宏观 BRDF 仍被微面 NDF、Masking-Shadowing、环境入射方向亮度和多次散射共同影响。

---

## 五、GGX / Trowbridge–Reitz 分布

### 1. GGX 为什么比传统 Phong 高光更适合很多真实材质？

GGX 有相对显著的 Long Tail，能表示离镜面反射中心较远的方向仍有贡献。许多材质高光的长尾行为用 GGX 更易拟合。

### 2. 各向同性 GGX NDF 的公式？

定义 `r=roughness`、`alpha=r²`：

$$
\boxed{
D_{\mathrm{GGX}}(N,H)=
\frac{\alpha^2}{
\pi\big[(NoH)^2(\alpha^2-1)+1\big]^2
}.
}
$$

对 `NoH>0` 使用，并对数值分母采取保护。

### 3. GGX 代码中 a、a2 分别是什么？

~~~glsl
float a = roughness * roughness; // alpha
float a2 = a * a;                // alpha squared
float denom = NdotH * NdotH * (a2 - 1.0) + 1.0;
float D = a2 / (PI * denom * denom);
~~~

`a2` 实际等于 `r⁴`，而非 `r²`；确认每段 Shader 的 alpha/roughness 约定，防止重复映射导致高光宽度错误。

### 4. GGX 的归一化是什么？

$$
\int_{\Omega^+}
D_{\mathrm{GGX}}(H)\,NoH\,\mathrm d\omega_h=1.
$$

这也解释采样 Half Vector 的 PDF `pH=D*NoH`。

### 5. Roughness 对 F0 有影响吗？

在此工作流的基础模型里，Roughness 控制 NDF/G 的形状，F0 主要由材质折射率/金属颜色决定；**Roughness 不应直接被当成正入射反射率**。

更高阶的多次散射或分层模型可能在不同粗糙度下表现出有效反射的变化，但不意味着 F0 的基础定义改变。

### 6. 什么是 Anisotropic GGX？

微法线在 T/B 两个切线方向的分布宽度不同，常用于拉丝金属等。各向同性 GGX 仅由 `NoH` 和单个 alpha 控制；各向异性还需要 `TdotH`、`BdotH` 等量。

---

## 六、Smith Masking-Shadowing 与 Schlick–GGX

### 1. Smith G1 的常见准确 GGX 形式是什么？

对 `NoX>0`、`alpha=r²`：

$$
\boxed{
G_1(N,X)=
\frac{2\,NoX}{
NoX+\sqrt{\alpha^2+(1-\alpha^2)(NoX)^2}
}.
}
$$

`NoX=0` 时按有效边界处理为 0，数值实现应避免除零。

### 2. 一个常见的可分离 Smith Geometry 是什么？

$$
G(N,V,L)=G_1(N,V)\,G_1(N,L).
$$

这是常见的可分离 Smith 近似。还存在 Height-Correlated Smith 等相关版本，不能将所有 G 模型混称完全一致。

### 3. 为什么实时 Shader 常使用 Schlick–GGX 近似？

为了减少运算量，一种常见形式：

$$
G_{\mathrm{SchlickGGX}}(NoX)=
\frac{NoX}{NoX(1-k)+k}.
$$

经典直接光照近似使用 `k=(r+1)²/8`；常见 IBL 讨论会用 `k=alpha/2`（其中 `alpha=r²`）。这些 `k` 是不同使用背景的**近似选择**，不是同一个精确 G1 等式。

### 4. 是否可以 Direct 用一种 G，BRDF LUT 用另一种 G？

可以进行明确设计的近似，但必须理解会改变能量和角度响应。比较 Direct 与 IBL 的外观一致性时，首先核对 LUT 预计算时的 G 模型与运行时声明的模型。

### 5. 为什么掠射角 G 经常更小？

观察或入射方向越接近宏观表面切线，某些微面更容易被其他微面挡住，因此单次散射的 Masking/Shadowing 贡献下降。

### 6. 为什么不可以把宏观阴影可见性 Vshadow 直接塞到 G 中？

G 的定义基于材质内部微几何统计，Vshadow 则由场景中的其他物体遮挡决定；两者建模尺度与算法完全不同。

---

## 七、Metallic–Roughness 材质工作流

### 1. Base Color、Metallic、Roughness 的作用？

| 属性 | 电介质 metallic=0 | 金属 metallic=1 |
|---|---|---|
| Base Color | 通常表示漫反射反照率 | 通常近似镜面 F0 颜色 |
| F0 | 典型约 0.04 | 有色，可来自 Base Color |
| Diffuse | 通常存在 | 典型理想化近似为零 |
| Roughness | 控制微法线分布 | 同样控制微法线分布 |

### 2. 为什么塑料也有镜面高光？

空气与塑料之间存在折射率差，因此产生 Fresnel 反射。`metallic=0` 并不意味着 `specular=0`。

### 3. F0 怎样由 Metallic 计算？

$$
\boxed{
F_0=\mathrm{mix}(0.04,C_{\mathrm{base}},m).
}
$$

~~~glsl
vec3 F0 = mix(vec3(0.04), baseColor, metallic);
~~~

这是简化的材质工作流，而不是对任意复折射率材料的严格物理混合。

### 4. 为什么 Diffuse 系数要乘 (1-F)(1-metallic)？

对于介质，入射光的一部分被 Fresnel 镜面反射，剩下的部分才可能进入介质形成漫反射。一个常见实时近似：

$$
k_D=(1-F)(1-m).
$$

它不是完整多层/多次散射下的精确能量账本，但比 Diffuse 与 Specular 完全无约束相加更合理。

### 5. 合并的 Direct BRDF 怎么写？

$$
\boxed{
f_r=
(1-m)(1-F)\frac{C_{\mathrm{base}}}{\pi}
+
\frac{DFG}{4(NoV)(NoL)}.
}
$$

对点光源，示意贡献：

$$
L_o^{\mathrm{point}}=
f_r\,\frac{I}{r^2}\,(NoL)\,V_{\mathrm{shadow}}.
$$

`I` 是理想点光源 Radiant Intensity，`Vshadow` 是宏观阴影可见性。

### 6. Metallic=0.5 是真实材料的独立物理属性吗？

通常是材质混合/过滤与工作流连续值近似，而非某种纯材质有严格定义的“半金属程度”。例如金属被灰尘覆盖的纹理像素可使用中间 Metallic 作为近似混合。

### 7. 为什么金属没有良好环境反射时会显得黑？

典型金属的 Diffuse≈0，如果直接光又不照亮当前可见镜面方向，而环境 Specular 缺失，那么该方向的反射贡献接近零。因此 **PBR 不能只做 Direct Light 就希望所有金属材质自然好看**。

---

## 八、完整的直接光 GGX GLSL 代码

### 1. D、F、G 的示例实现

本实现使用 `alpha=r²`、Schlick Fresnel、直接光 Schlick–GGX `k=(r+1)²/8`。出于数值稳定性限制了最小 Roughness；这不是完美镜面处理，也没有额外多次散射补偿。

~~~glsl
const float PI = 3.14159265358979323846;

float DistributionGGX(float NdotH, float roughness)
{
    float r = clamp(roughness, 0.04, 1.0);
    float alpha = r * r;
    float a2 = alpha * alpha;
    float d = NdotH * NdotH * (a2 - 1.0) + 1.0;
    return a2 / max(PI * d * d, 1e-7);
}

vec3 FresnelSchlick(float cosTheta, vec3 F0)
{
    float c = clamp(cosTheta, 0.0, 1.0);
    return F0 + (vec3(1.0) - F0) * pow(1.0 - c, 5.0);
}

float GeometrySchlickGGX(float NdotX, float roughness)
{
    float r = clamp(roughness, 0.04, 1.0);
    float k = (r + 1.0) * (r + 1.0) / 8.0;
    return NdotX / max(NdotX * (1.0 - k) + k, 1e-6);
}

float GeometrySmith(float NdotV, float NdotL, float roughness)
{
    return GeometrySchlickGGX(NdotV, roughness) *
           GeometrySchlickGGX(NdotL, roughness);
}
~~~

### 2. 点光源 Direct Contribution 如何组合？

~~~glsl
vec3 EvaluateCookTorrance(
    vec3 N, vec3 V, vec3 L,
    vec3 baseColor, float metallic, float roughness,
    vec3 lightFactor, float shadowVisibility)
{
    N = normalize(N);
    V = normalize(V);
    L = normalize(L);

    float NoV = max(dot(N, V), 0.0);
    float NoL = max(dot(N, L), 0.0);
    if (NoV <= 0.0 || NoL <= 0.0) return vec3(0.0);

    vec3 VH = V + L;
    if (dot(VH, VH) <= 1e-8) return vec3(0.0);
    vec3 H = normalize(VH);

    float NoH = max(dot(N, H), 0.0);
    float VoH = max(dot(V, H), 0.0);

    vec3 F0 = mix(vec3(0.04), baseColor, metallic);
    vec3 F = FresnelSchlick(VoH, F0);
    float D = DistributionGGX(NoH, roughness);
    float G = GeometrySmith(NoV, NoL, roughness);

    vec3 specular = (D * G * F) /
                    max(4.0 * NoV * NoL, 1e-5);
    vec3 kD = (vec3(1.0) - F) * (1.0 - metallic);
    vec3 diffuse = kD * baseColor / PI;

    return (diffuse + specular) * lightFactor *
           NoL * shadowVisibility;
}
~~~

对理想点光，`lightFactor=I/r²`；对方向光，应输入按引擎定义的方向照明强度，而不是直接套用同一个距离衰减函数。

### 3. 为什么不能在 Specular 上额外再乘一次 F？

因为 `specular=D*F*G/(4 NoV NoL)` 已包含 F。若再次乘 F，会错误降低 Specular 贡献。对 G 和几何分母同样不能任意重复应用。

### 4. 为什么要先检查 NoV 和 NoL？

Cook–Torrance 的该反射形式仅针对有效上半球。仅对分母加一个 Epsilon 而继续处理背面或退化方向，会生成不符合模型物理定义的高光。

### 5. 这段 Shader 是严格能量守恒的完整 PBR 吗？

不是。它是常见实时单次散射近似；Schlick–GGX Geometry、Fresnel 近似、Metallic 混合、漫反射能量分配、Roughness Clamp、缺失多次散射均影响精确性。面试时应说明模型假设，而不是宣称“DFG 一定完全物理正确”。
