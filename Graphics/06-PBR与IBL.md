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

---

## 九、Disney Principled BRDF 与常见材质扩展

### 1. Disney BRDF 与 Cook–Torrance 有什么关系？

Disney 提出的 Principled BRDF/BSDF 系列模型以艺术家容易理解的参数表达多种材质外观，其中一些反射分量采用微表面模型，并扩展了 Diffuse、Sheen、Clear Coat、Anisotropy 等特性。

**不能把 Disney BRDF 等同于 GGX Cook–Torrance。** 基础 GGX 镜面项只是完整 Principled 模型的重要组成部分，不同渲染器/版本的 Disney 材质实现还可能不同。

### 2. Disney 为什么不只使用最简单的 Lambert Diffuse？

真实介质的漫反射在掠射方向可能有不同的角度响应，单纯常数 `albedo/π` 不一定符合所有测量材料。

一种经典 Disney Diffuse Approximation 使用：

$$
F_{D90}=0.5+2r\,(\mathbf L\cdot\mathbf H)^2.
$$

$$
F_D=
\big[1+(F_{D90}-1)(1-NoL)^5\big]
\big[1+(F_{D90}-1)(1-NoV)^5\big].
$$

相应 Diffuse 为：

$$
\boxed{
f_{\mathrm{DisneyDiffuse}}=
\frac{C_{\mathrm{base}}}{\pi}\,F_D.
}
$$

它描述特定经验拟合的方向响应，**不是所有方向/粗糙度下严格能量守恒的唯一漫反射公式**。实际引擎会采用归一化、能量补偿和其它模型的不同组合。

### 3. Clear Coat 用于什么材料？

Clear Coat 近似基底表面上方额外的一层透明镜面涂层，例如汽车漆、清漆和某些塑料涂层。

它通常具有自身的 Fresnel、Roughness 和反射分布；不能直接“在 Base Specular 上多加一份白色高光”还声称不增加能量。

### 4. Sheen 和 Anisotropy 用于什么？

- **Sheen：** 常用于布料、织物等在掠射角出现的柔和纤维散射外观。
- **Anisotropy：** 用两个不同切线方向的微面分布表示拉丝金属、缎面等方向性高光。
- **Subsurface：** 描述进入介质后发生的空间散射；完整 BSSRDF 与单点 BRDF 的建模范围不同。

### 5. 为什么真实引擎的 Disney 材质不必和论文逐行相同？

引擎会根据性能、移动平台、已有灯光结构、贴图通道、Layered Material、可维护性，以及多次散射补偿方案进行取舍。

面试中应先讲清楚**你真正实现的是哪些分量**，例如 GGX + Schlick Fresnel + Lambert/Disney Diffuse，不要把“参考了 Disney Principled”概括成完整实现了所有 Principled Lobes。

---

## 十、IBL 的总体思路

### 1. IBL 如何进入渲染方程？

环境贴图提供来自各个方向的入射 Radiance：

$$
L_o^{\mathrm{IBL}}=
\int_{\Omega^+}
f_r(N,L,V)\,
L_{\mathrm{env}}(L)\,
NoL\,\mathrm d\omega_l.
$$

其中 `Lenv(L)` 通过环境图在方向 `L` 上采样，材质 BRDF 负责根据入射方向和视角分配反射贡献。

### 2. 为什么不能只用一个环境平均色模拟 IBL？

环境照明具有方向性：室内窗户、夜景灯带、天空亮度与地面反射分布不同。

- Diffuse 因 Lambert 的宽半球积分而相对低频。
- Specular 对 Roughness、反射方向与 Fresnel 敏感，可能需要保留环境中的高频亮点。

只用一个全局平均色会丢掉方向信息和材质粗糙度的差异。

### 3. 实时 PBR IBL 为什么经常需要三类预计算资源？

| 资源 | 预计算的内容 | Runtime 主要输入 |
|---|---|---|
| Irradiance Cubemap | Diffuse 的半球 Cosine 加权环境积分 | 法线方向 N |
| Prefiltered Environment Cubemap | 按镜面分布和 Roughness 平滑后的环境 Radiance | 反射方向 R、Roughness/LOD |
| BRDF Integration LUT | Fresnel/Geometry 和 View Angle/Roughness 相关的积分因子 | NoV、Roughness |

实际还需要**原始 HDR Environment Cubemap** 作为天空/背景、预计算输入或特殊材质采样资源。三类预计算资源也可存到别的表示形式，如 SH、Octahedral Map 或 Probe Atlas。

### 4. 为什么 Diffuse 和 Specular IBL 必须分开处理？

对于 Lambert，BRDF 是常数 `albedo/π`，因此可以先对环境进行 Cosine Convolution。

对于 GGX Specular，BRDF 含 `D、F、G`，显著依赖方向 `V` 和材质 Roughness，不能简单复用 Diffuse Irradiance Cubemap。

### 5. IBL 一定等于完整 GI 吗？

不是。普通全局静态环境贴图无法精确知道当前表面点附近的遮挡、反射位置或多跳散射结果。

Reflection Probe、Local Probe Parallax Correction、Bent Normal、AO、Screen-Space Reflection、Ray Tracing 等可进一步补充位置变化与遮挡，但都需要理解各自近似范围。

---

## 十一、Diffuse IBL：Irradiance Convolution

### 1. Lambert 的环境漫反射怎么推导？

根据 `f_d=albedo/π`：

$$
L_o^{\mathrm{diffuse}}(N)=
\frac{C_{\mathrm{base}}}{\pi}
\int_{\Omega^+}
L_{\mathrm{env}}(L)(N\cdot L)
\,\mathrm d\omega_l.
$$

把仅与环境和法线方向有关的积分定义为 Irradiance：

$$
\boxed{
E(N)=\int_{\Omega^+}
L_{\mathrm{env}}(L)\,(N\cdot L)
\,\mathrm d\omega_l.
}
$$

则：

$$
\boxed{
L_o^{\mathrm{diffuse}}=
\frac{C_{\mathrm{base}}}{\pi}\,E(N).
}
$$

如果材质包含 Metallic，则按工作流为漫反射再乘 `1-metallic` 及相应 Fresnel 能量分配因子。

### 2. 为什么 Irradiance Map 通常可以使用较低分辨率？

Diffuse Cosine Convolution 是宽角度的低频积分，即使原环境纹理有尖锐亮点，积分到某法线方向后一般更平滑。

因此 Irradiance Cubemap 分辨率通常可显著低于原始 Environment Cubemap，具体尺寸仍需由照明频率和误差要求决定。

### 3. Cosine-Weighted Monte Carlo 如何计算 E(N)？

对半球使用 `p(L)=NoL/π`：

$$
\widehat E(N)=
\frac1M\sum_{k=1}^{M}
\frac{L_{\mathrm{env}}(L_k)(NoL_k)}
{NoL_k/\pi}.
$$

因此：

$$
\boxed{
\widehat E(N)=
\frac{\pi}{M}
\sum_{k=1}^{M}L_{\mathrm{env}}(L_k).
}
$$

样本必须真的是 Cosine-Weighted。若采用 Uniform Hemisphere，则 PDF 是 `1/(2π)`，权重不会直接化简为上式。

### 4. 为什么有的引擎采样 Irradiance 后不再除以 π？

**因为 LUT/Cubemap 的存储定义不同。**

- 若 Cubemap 存的是物理定义的 `E(N)`，运行时 Lambert 要乘 `albedo/π`。
- 若预计算时已经存入 `E(N)/π`，运行时可以直接乘 `albedo`。

两种约定都可用，**但不能同时在预计算和 Shader 中各除一次 π**；也不能完全漏掉归一化后依靠曝光弥补。

### 5. Constant Environment 的 Diffuse IBL 应该是什么？

若 `Lenv(L)=L0` 为常量，则：

$$
E(N)=L_0\int_{\Omega^+}\cos\theta\,\mathrm d\omega
=\pi L_0.
$$

因此：

$$
L_o^{\mathrm{diffuse}}=
\frac{albedo}{\pi}\,\pi L_0
=\boxed{albedo\,L_0}.
$$

这是非常适合验证 Irradiance Precompute 和 `1/π` 是否放对位置的单元测试。

### 6. 一段明确存储约定的 Diffuse IBL GLSL

~~~glsl
uniform samplerCube uIrradianceMap; // stores E(N), not E(N)/PI

vec3 SampleDiffuseIBL(
    vec3 N, vec3 baseColor, float metallic, vec3 kD)
{
    vec3 irradiance = texture(uIrradianceMap, normalize(N)).rgb;
    return kD * baseColor * irradiance / PI;
}
~~~

`kD` 是另外求得的材质漫反射权重，通常包含 `1-metallic`。若你的 Irradiance Map 存的是 `E/π`，则应删去这里的除以 `PI`。

---

## 十二、Specular IBL 与 Split-Sum Approximation

### 1. Specular IBL 原始积分是什么？

$$
L_o^{\mathrm{spec}}=
\int_{\Omega^+}
L_{\mathrm{env}}(L)\,
\frac{D(H)F(V,H)G(N,V,L)}{4NoV NoL}
\,NoL\,\mathrm d\omega_l.
$$

化简可以看出它同时依赖：

- `Lenv(L)`：环境中不同方向的亮度。
- `N,V`：表面法线与观察方向。
- `roughness`：NDF/Geometry 分布。
- `F0`：Fresnel 颜色。

运行时逐 Fragment 对环境全部方向精确积分过于昂贵。

### 2. Split-Sum 的核心近似是什么？

常见实时引擎采用如下**近似的积分分离**：

$$
\boxed{
L_o^{\mathrm{spec}}(N,V,r,F_0)
\;\approx\;
L_{\mathrm{prefilter}}(R,r)\,
\Big[F_0 A(NoV,r)+B(NoV,r)\Big].
}
$$

其中：

- `R=reflect(-V,N)`，是宏观法线镜面反射方向。
- `Lprefilter(R,r)` 来自预滤波的环境图，Roughness 控制采样的模糊级别。
- `A、B` 来自预计算的二维 BRDF LUT，以 `NoV` 和 `r` 为坐标。

**非常重要：** 积分的乘积不等于积分的乘积；Split-Sum 使用了**分离/预滤波的工程近似**，尤其预滤波环境部分常假设 `N=V=R` 来减小维度，并不完全正确描述掠射视角所有环境高光变化。

### 3. 为什么要从五个变量降到三类资源？

直接建立 `N,V,roughness,F0,environment` 的完整高维积分表不可行。把：

1. 环境对镜面高光方向和 Roughness 的影响近似放入 Prefilter Cubemap。
2. Fresnel/Geometry/视角/粗糙度影响放入可复用的 2D LUT。

就能把运行时积分降为少量 Texture Sample、F0 的线性组合及简单乘法。

### 4. 为什么 Prefilter Env 用 R，而不是 N？

理想镜面（`roughness→0`）从当前观察方向 `V` 对应的反射方向是：

~~~glsl
vec3 R = reflect(-V, N);
~~~

环境高光最主要围绕该反射方向分布。因此 Prefilter Cubemap 通常按 `R` 查找，随着 Roughness 增大使用更宽分布的预积分结果。

Diffuse Irradiance 则主要按 `N` 查找。这是 PBR/IBL 的高频面试区别。

### 5. 为什么 Prefilter Map 不是“随便 Gaussian Blur 环境图”？

微表面高光的方向权重由 GGX NDF/BRDF 相关分布定义。常用预滤波根据 GGX 重要性采样生成方向，并非简单屏幕空间高斯核。

预积分在不同 Roughness、方向与 Mip 中近似计算不同的环境角度平均，其权重与 PDF 必须匹配相应数学定义。

### 6. 为什么称为 Split-Sum 而不是完全精确的环境卷积？

由于真实入射环境可能有极窄亮点、方向分布和 BRDF 的多变量耦合，强行将它们分离会丢失相关性。此外常见预滤波采用 `N=V` 的近似，会在掠射角、高粗糙度和强方向环境中出现误差。

---

## 十三、GGX Importance Sampling 与 PDF

### 1. 为什么 Sample Half Vector H 而不是直接均匀 Sample L？

GGX BRDF 的镜面贡献主要集中在满足反射条件的微法线方向。先按 NDF 分布采样 `H`，再由 `H` 反射入射方向 `-V` 得到 `L`，通常比均匀随机选择光照方向更有效。

### 2. GGX Half Vector 的常见采样参数化怎么写？

使用 `u1,u2` 为独立均匀随机数，`alpha=r²`（`0<alpha≤1`）：

$$
\phi=2\pi u_1,
$$

$$
\boxed{
\cos\theta_h=
\sqrt{
\frac{1-u_2}{
1+(\alpha^2-1)u_2
}
}.
}
$$

随后在局部 `+Z` 方向：

$$
H_{\mathrm{local}}=
(\sin\theta_h\cos\phi,\,
\sin\theta_h\sin\phi,\,
\cos\theta_h).
$$

用局部 TBN/ONB 将采样结果转换到世界空间 `H`。

### 3. 从 H 计算 L 的方向关系是什么？

~~~glsl
vec3 L = reflect(-V, H);
~~~

然后检查 `NoL>0`。采出的某些 H 可能反射到宏观表面的下半球，该样本不会对普通反射半球积分作出贡献。

### 4. GGX H Sampling 的 PDF 是多少？

如果按 `D(H)NoH` 采样微法线：

$$
\boxed{
p_H(H)=D(H)\,NoH.
}
$$

但 BRDF 积分是对**方向 L** 进行的，需要使用变量变换的 Jacobian。

### 5. 为什么 PDF_L 中出现 4 VoH？

对固定 `V`，镜面反射 `H→L` 的立体角变量变换：

$$
\mathrm d\omega_l
=4(\mathbf V\cdot\mathbf H)\,\mathrm d\omega_h.
$$

因此：

$$
\boxed{
p_L(L)=
\frac{p_H(H)}{4VoH}
=
\frac{D(H)\,NoH}{4VoH}.
}
$$

这里应保证 `VoH>0`。漏掉这一步，会把 Half Vector 的 PDF 错误当成 Light Direction 的 PDF，使 Monte Carlo 积分出现系统性错误。

### 6. 一段 GLSL 重要性采样函数

~~~glsl
vec3 ImportanceSampleGGX(vec2 Xi, vec3 N, float roughness)
{
    float alpha = max(roughness * roughness, 1e-4);
    float a2 = alpha * alpha;

    float phi = 2.0 * PI * Xi.x;
    float cosTheta = sqrt(
        (1.0 - Xi.y) / (1.0 + (a2 - 1.0) * Xi.y)
    );
    float sinTheta = sqrt(max(0.0, 1.0 - cosTheta * cosTheta));

    vec3 Ht = vec3(cos(phi) * sinTheta,
                   sin(phi) * sinTheta, cosTheta);

    vec3 up = abs(N.z) < 0.999 ?
        vec3(0.0, 0.0, 1.0) : vec3(1.0, 0.0, 0.0);

    vec3 T = normalize(cross(up, N));
    vec3 B = cross(N, T);
    return normalize(T * Ht.x + B * Ht.y + N * Ht.z);
}
~~~

输入 `N` 应为非零、单位方向，且采样器已保证两个随机数落在 `[0,1)` 内。

### 7. VNDF Sampling 比普通 NDF Sampling 好在哪里？

可见法线分布（Visible Normal Distribution Function，VNDF）会考虑从当前 `V` 方向可见的微法线，通常能减少采样到无效/被遮蔽微面的浪费。

**注意：** VNDF 的采样方向分布及 PDF 与简单 `D NoH` 不一样；如果改用 VNDF Sampling，必须同步修改权重和 PDF，不能仅替换采样函数。

---

## 十四、Prefiltered Environment Cubemap

### 1. 怎样从 HDR Environment 生成 Specular Prefilter Map？

典型离线/启动预处理步骤：

1. 将 HDR Equirectangular Environment 转换到 Cubemap（如果原本是经纬图）。
2. 针对每个输出 Mip Level 选择对应 Roughness。
3. 对 Cubemap 每个输出 Texel 的方向 `R` 做 GGX 重要性采样。
4. 得到采样方向 `L`，从原始 HDR Cubemap 读取 Radiance。
5. 按预滤波定义累计权重、归一化后写入该 Mip。

**关键约定：** 常见 Split-Sum Prefilter 生成时将当前方向当作 `N`，并近似设置 `V=N`，因此所有角度相关的 BRDF 变化不会完全体现在 Prefilter Map 内。

### 2. 为什么 Prefilter Map 使用 Mipmap 表示 Roughness？

Roughness 越大，所需积分方向通常越宽，适合放在更粗的 Mip Level。Runtime 可以用：

~~~glsl
float lod = roughness * float(uMaxPrefilterMip);
vec3 prefiltered = textureLod(uPrefilterMap, R, lod).rgb;
~~~

**注意：** `lod=roughness*maxMip` 是常见工程映射，严格结果取决于你在预计算时每一级 Roughness 的定义；如果预滤波使用非线性映射，应同步修改运行时 LOD。

### 3. 预滤波采样 PDF 为什么还会影响输入 Environment 的 Mip 选择？

输入原始 HDR Cubemap 也可能有 Mip，用来防止随机少量样本读取极窄 HDR 亮点时产生方差或欠采样。

若输入 Cubemap Face 分辨率为 `resolution`，一种常见概念近似：

$$
\omega_{\mathrm{texel}}\approx
\frac{4\pi}{6\,resolution^2}.
$$

方向采样的平均立体角近似：

$$
\omega_{\mathrm{sample}}\approx
\frac{1}{M\,p_L(L)}.
$$

因此一种启发式输入采样 LOD：

$$
\boxed{
LOD_{\mathrm{source}}\approx
\frac12\log_2
\left(
\frac{\omega_{\mathrm{sample}}}
{\omega_{\mathrm{texel}}}
\right).
}
$$

这里的公式是以纹理 Level 的二维面积变化来对应立体角比例；它是**重要性采样环境预滤波的常见近似**，不是对任意 Cubemap Texel 精确立体角的通用解析式。

### 4. 为什么 Prefilter 不能只 Blur 最终屏幕颜色？

屏幕模糊不随真实环境方向与几何反射方向自然变化，物体表面转动、摄像机移动时容易失去反射一致性。

Prefilter Env 是在环境**方向域**上进行与 BRDF 相关的预积分，区别于简单的屏幕 Space Blur。

### 5. 为什么用有限样本预滤波 HDR 环境可能出现 Fireflies？

HDR 环境可能有极亮而很窄的区域，低概率命中样本会带来高权重贡献；样本太少、PDF 或源 LOD 估计不正确均可能导致噪点或闪烁。

增加样本、正确的 PDF/权重、输入环境预滤波、合理的源 Mip 选择可以改善；随意 Clamping 非常亮的样本可能造成偏差。

---

## 十五、BRDF Integration LUT：为何只需要两通道？

### 1. Fresnel Schlick 怎样帮助分离 F0？

Schlick 公式：

$$
F=F_0+(1-F_0)(1-VoH)^5.
$$

定义 `Fc=(1-VoH)^5`，重排：

$$
\boxed{
F=F_0(1-F_c)+F_c.
}
$$

对单次 GGX 镜面环境积分，使用这种分解可以将**F0 的线性系数**与**与 F0 无关的偏置项**分别积分。

### 2. LUT 中 A 和 B 到底积分了什么？

使用上面**按 NDF 采样 H 并转换成 L**的 Monte Carlo 估计，镜面积分的每样本几何权重可以化简成：

$$
G_{\mathrm{vis}}=
\frac{G(N,V,L)\,VoH}
{NoH\,NoV}.
$$

对满足 `NoL>0` 的采样：

$$
\boxed{
A(NoV,r)=
\mathbb E[(1-F_c)\,G_{\mathrm{vis}}],
}
$$

$$
\boxed{
B(NoV,r)=
\mathbb E[F_c\,G_{\mathrm{vis}}].
}
$$

因此预积分项为：

$$
\boxed{
F_0 A(NoV,r)+B(NoV,r).
}
$$

**使用前提：** 这里的期望对应特定的 GGX Half Vector 采样及 Geometry 模型。若采用 VNDF Sampling、不同 G、Multi-Scattering 模型或另一套近似定义，LUT 的生成公式应相应改变。

### 3. 为什么是二维 LUT？

对给定 BRDF 模型，以上系数主要随：

- 宏观视角 `NoV`。
- Roughness `r`。

变化。`F0` 已通过线性重排被分离，因此不需要再为每一种材质颜色生成单独 LUT。

LUT 常用 RG 通道分别存 `A、B`，纹理尺寸如 `256×256` 仅是工程常见选择，不是算法定理。

### 4. BRDF LUT 的积分 Shader 核心示例

下面代码与本篇普通 `D NoH` 的 NDF Sampling 及 `G1·G1` 或相应一致的 G 函数搭配。示例中的 `GeometrySmith` 需明确使用你决定的 LUT 几何项；**不建议误用第八节针对 Direct Light 的 Schlick k 参数，还同时声称 LUT 是精确 GGX Smith 积分**。

~~~glsl
vec2 IntegrateBRDF(float NdotV, float roughness, uint sampleCount)
{
    vec3 N = vec3(0.0, 0.0, 1.0);
    vec3 V = vec3(sqrt(max(0.0, 1.0 - NdotV*NdotV)),
                  0.0, NdotV);

    float A = 0.0;
    float B = 0.0;

    for (uint i = 0u; i < sampleCount; ++i)
    {
        vec2 Xi = Hammersley(i, sampleCount);
        vec3 H = ImportanceSampleGGX(Xi, N, roughness);
        vec3 L = normalize(reflect(-V, H));

        float NoL = max(L.z, 0.0);
        float NoH = max(H.z, 0.0);
        float VoH = max(dot(V, H), 0.0);

        if (NoL > 0.0 && NoH > 0.0 && NdotV > 0.0)
        {
            float G = GeometrySmithForLUT(NdotV, NoL, roughness);
            float Gvis = G * VoH / (NoH * NdotV);
            float Fc = pow(1.0 - VoH, 5.0);

            A += (1.0 - Fc) * Gvis;
            B += Fc * Gvis;
        }
    }

    return vec2(A, B) / float(sampleCount);
}
~~~

**依赖说明：** `Hammersley`、`GeometrySmithForLUT` 是后文单独解释的接口，这段核心示例不是一个无需其它声明即可编译的完整 Shader。对 `NoV≈0` 的边界要单独处理，避免除零。

### 5. LUT A/B 在 Metallic 与 Dielectric 上如何复用？

LUT 只依赖 `NoV`、Roughness 和所采用的几何/采样模型。用不同的 `F0` 向量乘 A 再加 B，就能为介质和金属得到不同的镜面颜色权重。

### 6. 为什么 LUT 的 UV 方向常出 Bug？

有些实现纹理横轴为 `NoV`、纵轴为 Roughness，有些写入或纹理坐标原点使用不同约定。若误把 `(NoV,roughness)` 颠倒，材质会在斜视角和粗糙度变化时出现奇怪的能量分布。

建议在 RenderDoc Texture Viewer 中观察 LUT 两个通道，再使用固定输入与 CPU Reference Test 校验。

---

## 十六、Hammersley、Radical Inverse 与预计算采样

### 1. 为什么预积分常使用 Hammersley Sequence？

纯伪随机序列简单，但会在有限样本下产生随机聚集。Hammersley 等低差异（Low-Discrepancy）序列更均匀地覆盖二维单位方形，常在预计算中降低噪点。

这不等于对所有积分“无偏性自动更强”，也不意味着所有场景里无条件超过随机数；它改变的是采样分布的均匀性和误差表现。

### 2. Hammersley 的一个常见二维定义？

对整数 `i=0..N-1`：

$$
\boxed{
\mathbf X_i=
\left(\frac{i}{N},\ \phi_2(i)\right).
}
$$

`φ2` 为以 2 为底的 Radical Inverse，等价于把二进制位反转后映射到 `[0,1)`。

### 3. GLSL 中怎样计算 Hammersley？

~~~glsl
float RadicalInverseVdC(uint bits)
{
    bits = (bits << 16u) | (bits >> 16u);
    bits = ((bits & 0x55555555u) << 1u) |
           ((bits & 0xAAAAAAAAu) >> 1u);
    bits = ((bits & 0x33333333u) << 2u) |
           ((bits & 0xCCCCCCCCu) >> 2u);
    bits = ((bits & 0x0F0F0F0Fu) << 4u) |
           ((bits & 0xF0F0F0F0u) >> 4u);
    bits = ((bits & 0x00FF00FFu) << 8u) |
           ((bits & 0xFF00FF00u) >> 8u);

    return float(bits) * 2.3283064365386963e-10;
}

vec2 Hammersley(uint i, uint N)
{
    return vec2(float(i) / float(N), RadicalInverseVdC(i));
}
~~~

要求 `N>0`。如果着色平台具有支持位反转的内建函数，可根据目标 API 优化。

### 4. 为什么低差异预计算结果也可能有可见条纹？

同一确定性 Sequence、有限样本、亮环境热点、采样不足及预滤波核近似可能产生结构化误差。可增加样本、适当随机扰动/旋转采样序列或使用其它低差异采样策略。

