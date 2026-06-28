# Variance / EVSM 软阴影实现机理与静/动 moment 合并

- **Type**: Synthesis (aggregated from multiple sources)
- **Fetched**: 2026-06-28
- **Project**: urp-csm-cache-evsm
- **Relevant to**: Q4（Variance/EVSM 实现：moment 编码、Chebyshev、漏光抑制、EVSM 翘曲、预滤波、精度）、Q5（★核心★ 静/动 EVSM 分开存、光照阶段合并）
- **Sources drawn from**:
  1. [NVIDIA GPU Gems 3, Ch.8 — Summed-Area Variance Shadow Maps (Andrew Lauritzen)](https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-8-summed-area-variance-shadow-maps)
  2. [TheRealMJP/Shadows — VSM.hlsl (GitHub raw, MIT)](https://raw.githubusercontent.com/TheRealMJP/Shadows/master/Shadows/VSM.hlsl)
  3. [TheRealMJP — "Shadow Maps" blog](https://therealmjp.github.io/posts/shadow-maps/)
  4. [TheRealMJP — "Shadow Sample Update" blog](https://therealmjp.github.io/posts/shadow-sample-update/)
  5. [Annen et al. — "Exponential Shadow Maps" (Graphics Interface 2008)](https://jankautz.com/publications/esm_gi08.pdf)
  6. [Peters & Klein — "Moment Shadow Mapping" (I3D 2015)](https://momentsingraphics.de/Media/I3D2015/MomentShadowMapping.pdf)
  7. [Peters & Klein — "Beyond Hard Shadows" (I3D 2016)](https://momentsingraphics.de/I3D2016.html)
  8. [Insomniac Games — "CSM Scrolling" (SIGGRAPH 2012)](https://advances.realtimerendering.com/s2012/insomniac/Acton-CSM_Scrolling(Siggraph2012).pdf)（缓存侧机制背景）
  9. [szaboeme/layered_variance_shadow_maps (GitHub, LVSM 实现)](https://github.com/szaboeme/layered_variance_shadow_maps)
  10. [Martin Cap — EVSM project (per Donnelly/Lauritzen & Lauritzen/McCool)](https://martincap.io/projects/evsm/)
  11. [Unity Discussions — RequestShadowMapRendering for static/dynamic objects separately](https://discussions.unity.com/t/requestshadowmaprendering-for-static-dynamic-objects-separately/861966)

> [n] 引用对应研究流内部编号（见文末 Sources，与上列基本一一对应）。

---

# Q4 — Variance / Exponential Variance Shadow Maps 实现机理

## 1.1 VSM 原理：矩 + Chebyshev 不等式

标准 depth shadow map 存单值深度，做"比较再滤波"，但硬件只能"先滤波再比较"——把若干深度平均后再做一次二值比较结果是错的，所以普通 shadow map 不能预滤波 [1]。VSM 改为**存深度分布的前两阶矩**：每个 texel 写 (depth, depth²)，滤波后恢复 [1][3]:

```
M1 = E[x]       (一阶矩 = 深度均值)
M2 = E[x²]      (二阶矩 = 深度平方的均值)
```

由两阶矩直接得均值与方差 [1][3]:

```
μ  = E[x] = M1
σ² = E[x²] − E[x]² = M2 − M1²
```

设当前着色片元在光空间的深度为 t。要估的是"滤波核内有多少比例的遮挡者深度 ≥ t（即遮挡 t）"。**单边（one-tailed）Chebyshev 不等式**给出这个比例的上界 [1][3]:

```
对 t > μ：  P(x ≥ t) ≤ p_max(t) = σ² / ( σ² + (t − μ)² )
对 t ≤ μ：  p_max = 1   (片元比平均遮挡者还近，判为全亮)
```

这里 `p_max` 被当作**可见性/PCF 结果的近似**。关键性质：Donnelly & Lauritzen 证明，对**单个平面遮挡者 + 单个平面接收面**的理想情形，该不等式取等号，即 p_max 此时精确等于真实 PCF 结果 [1][3]。t ≤ μ 时返回 1，天然避免了亮区的自阴影（等价于一种软 bias）[3]。

MJP 的参考实现把上述写成一个函数（教科书级标准实现）[2]:

```hlsl
float ChebyshevUpperBound(float2 moments, float mean, float minVariance, float lbr) {
    float variance = moments.y - moments.x*moments.x;   // σ² = M2 − M1²
    variance = max(variance, minVariance);              // 方差下限，挡 fp 噪声
    float d = mean - moments.x;                         // t − μ
    float pMax = variance / (variance + d*d);           // Chebyshev
    pMax = ReduceLightBleeding(pMax, lbr);              // 漏光抑制
    return (mean <= moments.x ? 1.0 : pMax);            // 单边
}
```

**为什么 VSM 能预滤波而 depth map 不能**：矩 (M1, M2) 是**线性量**，对它们做区域平均，得到的就是合并后那块区域深度分布的矩。所以 mipmap / 三线性 / 各向异性 / MSAA(在 shadow pass) / 可分离 box blur / SAT 全部合法 [1]。代价是每 texel 要存 2 个高精度通道而非 1 个 [3]。

**精度细节（VSM）** [1]：
- 不要用投影后非线性 z 当深度度量（方差不稳），用**到光源距离**（点/聚光）或**到光平面距离**（方向光，线性）。
- 写 M2 时用屏幕空间偏导补偿一个 texel 内的深度斜率（把 texel 建模为半像素标准差的高斯）：
  ```
  M2 = depth² + 0.25·(∂depth/∂x)² + 0.25·(∂depth/∂y)²
  ```
  面与光平面平行时退化为 depth²。要 clamp 偏导项，避免遮挡者近乎侧视时像素闪烁 [1]。

## 1.2 漏光（light bleeding）成因与抑制

**成因**：Chebyshev 只给上界，不是真值。当一个滤波核内同时含**近遮挡者**和**远接收面**、二者深度差很大时，分布方差 σ² 很大，分母里 (t−μ)² 相对变小，p_max 被高估——本应全黑的远接收面上漏进了光。最严重的情形是"遮挡者-接收面距离 a"与"接收面-接收面距离 b"之比 a/b 大时 [1][3]。根因是：真实区域可见性是个**阶跃函数**（N 个遮挡者就有 N 段），两阶矩信息不足以重建，不采更多样本无法根除 [1]。

**抑制手段**：
1. **方差下限** `σ² = max(σ², minVariance)`：纯数值噪声防护，不解决物理漏光，但必须有 [2]。
2. **Light-bleeding reduction（线性重映射 clamp）**：全遮挡区 t>μ 时 p_max<1，把 [0, amount] 这段尾巴压到 0 再线性拉回 [1][2]:
   ```
   Linstep(a,b,v)        = saturate((v−a)/(b−a))
   ReduceLightBleeding   = Linstep(amount, 1, pMax)
   ```
   `amount` 与场景尺度无关，但调太高会把真实半影压暗（over-darkening）。MJP 用 0.25 量级 [4]。
3. **ESM 指数偏置思路**（见 1.4）：换一种对深度做指数翘曲的表示，从根上压制漏光。

## 1.3 ESM（Exponential Shadow Maps）——EVSM 的另一半基因

ESM 把二值阴影测试 f(d,z)（d=接收点到光距离，z=shadow map 里最近遮挡者深度，恒有 d ≥ z）用指数逼近 [5]:

```
f(d,z) = lim_{c→∞} e^{−c(d−z)}  ≈  e^{−c(d−z)} = e^{−cd} · e^{cz}     (c 大正常数)
```

关键是**可分离**：`e^{−cd}` 只依赖接收点、`e^{cz}` 只依赖遮挡者深度。于是只需把 `e^{cz}` 写进 shadow map 并预滤波，采样时乘上运行期才知道的 `e^{−cd}` [5]:

```
filtered s_f(x) = e^{−c·d(x)} · ( w ⊛ e^{c·z(p)} )
```

- **c 怎么取**：c 越大、阶跃逼近越陡、漏光越少；但 c 受浮点精度上限约束。Annen 实测 **c = 80 对 fp32 最优**（比 16 阶 CSM 还准），不受精度问题困扰 [5]。
- **失效情形**：当 d−z < 0（斜面、深度不连续、滤波核跨边界）指数不收敛到 1 而爆炸式增长。无滤波时 clamp 到 1 即可；有滤波时需检测失败像素并 fallback 到 2×2 PCF（Z-Max 分类或阈值分类，典型只有 ~3–9% 屏幕像素需特殊处理）[5]。ESM 几乎不需要 polygon offset、对 shadow acne 不敏感 [5]。
- ESM 在接触阴影/全黑区质量高，但在**接收面边界不稳定**；VSM 反之（边界好、全黑区高方差时漏光）。把两者结合就是 EVSM [3][5]。

## 1.4 EVSM（Exponential Variance Shadow Maps）

思路：**先对深度做指数翘曲，再对翘曲后的值跑 VSM（存矩 + Chebyshev）**。把深度 z 先归一化到 [−1,1]，做两个翘曲 [2]:

```
warp+(z) =  exp(+c_pos · z)     // 正指数翘曲
warp−(z) = −exp(−c_neg · z)     // 负指数翘曲（注意整体取负号）
```

MJP 的标准实现（canonical）[2]:

```hlsl
float2 GetEVSMExponents(float posExp, float negExp, uint fmt) {
    const float maxExponent = (fmt == SMFormat16Bit) ? 5.54 : 42.0;  // 防 fp 溢出
    return min(float2(posExp, negExp), maxExponent);
}
float2 WarpDepth(float depth, float2 exponents) {
    depth = 2.0*depth - 1.0;                  // [0,1] → [−1,1]
    float pos =  exp( exponents.x * depth);
    float neg = -exp(-exponents.y * depth);
    return float2(pos, neg);
}
```

- **EVSM2**：只存正翘曲的 (warp+, warp+²)，2 通道 fp32。
- **EVSM4**：正、负翘曲各存一组 VSM = 4 通道 fp32（128 bit/texel）。

**采样阶段（核心）**——对正、负两路各跑一次 Chebyshev，然后**取最小可见性** [2]:

```hlsl
float2 warpedDepth = WarpDepth(shadowPos.z, exponents);
// minVariance 用翘曲的导数缩放（链式法则）：d(warp)/dz = exp·c
float2 depthScale  = VSMBias * 0.01 * exponents * warpedDepth;
float2 minVariance = depthScale * depthScale;

// EVSM4:
float posContrib = ChebyshevUpperBound(occluder.xz, warpedDepth.x, minVariance.x, lbr);
float negContrib = ChebyshevUpperBound(occluder.yw, warpedDepth.y, minVariance.y, lbr);
return min(posContrib, negContrib);          // ← 两路取 min

// EVSM2: 只用 posContrib
```

**为什么指数翘曲同时减漏光 + 改精度**：
- 指数函数把深度差**放大**。漏光发生在 (t−μ) 较小、被方差"淹没"时；指数翘曲后原本接近的两个深度被拉开，Chebyshev 分母里 (warp(t)−warp(μ))² 变大，p_max 被压低——漏光减少。等价于把 ESM 那种"陡阶跃"的好处注入 VSM 方差框架 [3][5]。
- **正翘曲 exp(+c·z)** 对"接收面在遮挡者之后"那侧给出紧的界；**负翘曲 −exp(−c·z)** 对另一侧给紧的界。两路各是真实可见性的**保守上界**，取 min 仍是合法保守界，却同时吃到两侧紧度——这就是 EVSM4 比 EVSM2 漏光更少的原因 [2][3]。
- MSM 论文明确：EVSM 算的是 visibility 的**一个 lower bound，但不是 sharp（最紧）的**——这是它相对 MSM 的理论短板 [6]。

**c 常数与精度**：c 越大越逼近真实阴影测试、漏光越少，但 `exp(c·z)` 会撑爆浮点动态范围。**EVSM 必须用 fp32**：fp32 下 c 上限约 **42**；fp16（EVSM16）下只能到约 **5.54**，翘曲因子还得额外 clamp（旧版 MJP sample clamp 到 10.0 会让 fp16 溢出，是个 bug）[2][4]。fp16 EVSM 漏光更重、需要更大 bias。

**精度选型小结**：

| 表示 | 通道/精度 | bit/texel | 备注 |
|---|---|---|---|
| VSM | RG16 / RG32F | 32 / 64 | 最轻；16-bit 用优化量化可显著降伪影 [6] |
| EVSM2 | 2×fp32 | 64 | 单正翘曲 |
| EVSM4 | 4×fp32 | 128 | 正负翘曲，质量最好，带宽重 [4][6] |
| MSM (Hamburger 4) | 4×16-bit(量化) / 4×fp32 | 64 / 128 | 见 1.5 |

- VSM/EVSM 用 **R16G16 vs R32G32F**：16-bit 存矩会让 σ²=M2−M1² 出现灾难性抵消（相近大数相减），方差噪声大、漏光重，需更大 minVariance 和 bias；32-bit 才能撑住指数翘曲动态范围，所以 **EVSM 实践上必须 fp32** [4][6]。
- **VSM/EVSM bias**：VSM 与 32-bit EVSM 用 VSMBias≈0.01（实际乘 0.0001）就够；16-bit EVSM 要到 0.5（实际 0.005）[4]。
- **级联接缝**：每级 cascade 是独立 EVSM RT，深度范围不同 → **翘曲常数 c 与方差下限要随该级深度范围归一化**（WarpDepth 先把深度映射到 [−1,1] 即为此）；接缝处做 cascade 间 dither/blend，且两级 LBR/bias 一致，否则缝处亮度跳变。

## 1.5（旁证）Moment Shadow Mapping（Peters & Klein 2015）

MSM 把 VSM 推广到**四阶幂矩** b = (E[z], E[z²], E[z³], E[z⁴])，4 通道 [6]。把"已知 m 个矩、求 CDF 在 z_f 处最紧 lower bound"形式化为**广义矩问题**（Hamburger / Hausdorff），VSM 是 m=2 特例，Chebyshev 被矩问题替代 [6]。

**Hamburger 4MSM 重建算法**（I=ℝ，主推）[6]:

```
输入: 滤波后矩 b∈ℝ⁴, 片元深度 z_f, 偏置 ε (如 3e-5)
1. 偏置:  b := (1−ε)·b + ε·(0.5,0.5,0.5,0.5)ᵀ        // 防奇异
2. Cholesky 解 3×3 线性系统 B·c = (1, z_f, z_f²)ᵀ:
       | 1   b1  b2 |
   B = | b1  b2  b3 |   ,  B 对称半正定 → B = L·D·Lᵀ
       | b2  b3  b4 |
3. 解二次方程 c3·z² + c2·z + c1 = 0 → 根 z2 ≤ z3
4. 若 z_f ≤ z2:  G = 0
   否则若 z_f ≤ z3: G = (z_f·z3 − b1·(z_f+z3) + b2) / ((z3−z2)·(z_f−z2))
   否则:           G = 1 − (z2·z3 − b1·(z2+z3) + b2) / ((z_f−z2)·(z_f−z3))
```

- **偏置 ε**：fp32 全程 ε=2e-6；16-bit 优化量化需更强 ε=3e-5（略增漏光）[6]。
- **优化量化**：用差分熵分析得到把 4 个矩重组的线性变换，使 16-bit 整数（64 bit/texel）可用，省一半带宽 [6]。
- **MSM vs EVSM**：MSM 是**最紧 lower bound**（4 矩、理论最优），漏光比 EVSM 少；EVSM 的界不 sharp [6]。但 MSM 重建要解 Cholesky + 二次方程，比 EVSM 一个除法贵；带宽上 16-bit MSM(64bit) = 32-bit VSM，省于 EVSM4(128bit)。MSAA 对 MSM/VSM/EVSM 都适用、对 PCF 不适用 [6]。Hamburger 的独特优点：论文证明它是唯一"重建结果与所选深度区间端点无关"的技术 [6]。

---

# Q5 — ★核心难题★ 静/动 EVSM 分开存、光照阶段合并

> **2026-06-28 关键确认（主 agent 补搜）**：用 DuckDuckGo HTML 专门检索"combine/merge static dynamic variance/EVSM/moment shadow map"，**确认业界无任何已发表技术**描述如何在光照阶段合并"分开存储的静/动 Variance/EVSM/moment 阴影图"。最接近的命中是：(a) Unreal「VSM Separate Static Caching」——但那是 **Virtual** Shadow Maps（页式缓存深度），不是 **Variance**；(b) LVSM/EVSM 的分层是为降漏光，不为静/动合成；(c) Unity 一个讨论帖 `RequestShadowMapRendering for static/dynamic objects separately` 也是标准 depth 图。**因此 Q5 的合并方案是本项目的原创工程设计**，下文区分"文献可推导"与"纯推导"。

> 背景前提：标准 depth 缓存可"静态缓存 + 动态每帧 `min(depth)` 合并"，因为深度 min 有物理意义（取最近遮挡者）。但 **EVSM 的矩对 min() 没有物理意义**：对 (warp, warp²) 取 min 会破坏方差结构（一阶矩与二阶矩各取了不同处的 min，不再来自同一分布，σ²=M2−M1² 可能变负或失真）。所以方向是：**静态 EVSM 与动态 EVSM 各存一套 RT，在采样阶段合并两路的"可见性"而不是合并矩。**

## 1.1 合并发生在可见性空间，不在矩空间

能合并的是**各自跑完 Chebyshev 后的可见性标量** V∈[0,1]：

```
V_static  = ChebyshevUpperBound( EVSM_static  采样, warp(t), minVar, lbr )
V_dynamic = ChebyshevUpperBound( EVSM_dynamic 采样, warp(t), minVar, lbr )
V_final   = combine(V_static, V_dynamic)
```

对 EVSM4，每一路本身就是 `min(posContrib, negContrib)`，所以静/动合并是在两个已各自 min 过的标量之间再合并。

## 1.2 取 `min(V_static, V_dynamic)` 是否物理正确？

**结论：min 是"被任一路遮挡即遮挡"的保守合并，方向正确，但在半影区会偏暗（不是真实复合可见性）。**

**(a) 为什么 min 合理——这些技术给的都是"never-too-dark"的上界。** MSM 论文反复强调：VSM/ESM/EVSM/LVSM 计算的都是真实可见性的 **lower bound on shadow intensity**（= upper bound on visibility V）[6]。两路各是各自遮挡集合的可见性上界。最终物理可见性 = 被静态集合**且**被动态集合都不遮挡才可见。在"硬可见性"（0/1）语义下，复合可见性 = AND，对 0/1 而言 AND ≡ **min**。所以 `min(V_static, V_dynamic)` 是把两路硬可见性 AND 的连续化，且因每路都是上界，min 后仍是合法上界（不漏光放大）。EVSM4 内部 `min(posContrib, negContrib)` 用的就是同一逻辑 [2]——EVSM4 本身就是"两个独立 Chebyshev 估计取 min"的现成范例。

**(b) min 的误差：半影"相乘 vs 取最小"。** 真实软阴影里，若片元同时处在静态半影 V_s 与动态半影 V_d，且两遮挡者在光源圆盘上遮挡的是**不同方向子立体角**，正确复合可见性应是**相乘** `V_s · V_d`（独立部分遮挡叠加）。而 `min(V_s, V_d) ≥ V_s·V_d`，所以 min 在双半影重叠区**偏亮**（漏光方向）。反过来，若两遮挡者遮挡**同一方向子立体角**（一前一后），正确结果是 `min`（后面被前面完全包含），此时 min 精确。
- 取舍：min 在"一前一后"几何下精确、并排双半影下偏亮一点；乘法 V_s·V_d 在并排时精确、前后重叠时**过黑**（double-counting）。两者都非普适正确，因为单凭两路标量无法知道两遮挡者在光圆盘的方位关系——这是 VSM 类"区域可见性是阶跃函数、两阶矩信息不足"局限的延伸 [1]。
- **推荐 min**：方向"宁可微漏光不可过黑"（过黑会产生假接触阴影），与所有 VSM 类"never-too-dark"哲学一致 [6]；且 min 在最常见的"静态地面/墙 + 动态角色站前面"（典型一前一后）下就是精确的。

> 标注：「两路各是 upper bound、min 后仍是 upper bound」有 [6] 支撑，EVSM4 用 min 合并两个 Chebyshev 有 [2] 代码支撑；「并排该乘、前后该 min」是基于独立部分遮挡叠加的**合理推导**，无专门"静/动 EVSM 合并"文献（见上方确认）。

## 1.3 更"正确"的做法：分层 / 概率合并

- **Layered Variance Shadow Maps (LVSM, Lauritzen & McCool 2008)**：把深度区间**智能划分成若干层**，每层独立 VSM 且只约束在该层深度区间内 [6]。每层内方差被显著压低 → 从根上消漏光。MSM 指出"若忽略层间重叠，LVSM 求解的就是矩问题"，即用层数换精度 [6]。**对静/动方案的启示**：静态远、动态近本身就是天然"按深度分层"；若让静态那路覆盖远层、动态那路覆盖近层，并在查找时**按片元深度落在哪层取对应路**（而非无条件 min），就更接近 LVSM 的正确性——代价是边界层选择逻辑和层间过渡。（开源实现可参考 szaboeme/layered_variance_shadow_maps [9]、Martin Cap EVSM [10]。）
- **按概率合并（乘法）**：当能假定两遮挡集合在光圆盘方位**独立**时，`V = V_static · V_dynamic` 更接近真实复合可见性（独立遮挡事件联合）。多遮挡叠加的通用处理（Fourier opacity / deep shadow 一脉）也是"沿光线累乘透过率"。
- **多遮挡者叠加的文献立场**：GPU Gems 3 明确，N 个遮挡者真实区域可见性是 N 段阶跃函数，任何只存少量矩的方法都无法精确重建、不采更多样本无法根除——所以 min 与乘法都是启发式近似 [1]。MSM 的"4 矩"更好是因为能区分约 2 个 Dirac 分布（≈2 个不同深度面），对"双遮挡面"重建可达精确 [6]——暗示**若把静/动合并成单套 MSM（4 矩）反而能更好处理"双面"**，但 MSM 的矩同样不能 min 合并，仍要分两套矩各自重建再 min/乘。

## 1.4 分开存的额外成本与质量收益

- **成本**：两套 RT（静态 + 动态，EVSM4 各 128 bit/texel）、两次 blur、采样阶段两次 SampleGrad + 两次 Chebyshev → 带宽和采样翻倍。这是 EVSM 路线相对"标准 depth 缓存只一套、min 合并"的主要劣势。
- **收益**：静态那路可整段缓存、只在相机/光移动时重画（配合 Insomniac 式 CSM scrolling 滚动复用，只重画进入视野的边缘 [8]）；动态那路每帧只画动态物，几何量小。
- **动态那路能否更便宜**：可以。
  - **更低分辨率**：动态 EVSM 可用静态一半分辨率（动态阴影边缘多在运动，对走样不敏感；min 会自然吃到较粗动态遮挡）。
  - **是否需要 blur**：动态那路**仍需 blur**——EVSM 软阴影质量来自预滤波，不 blur 会让动态阴影边缘比静态硬、风格不一致。但可用更小的 blur 核。
  - 也可动态路只存 EVSM2（正翘曲，64bit）省一半，质量损失主要在动态另一侧漏光，通常可接受。

## 1.5 两路深度范围不一致（静态远、动态近）对 c 与方差下限的影响

- **翘曲常数 c**：`WarpDepth` 先把各自路深度归一化到 [−1,1] 再 exp。**两路必须各自用自己的深度范围归一化**，否则同一物理深度在两路映射到不同 warp 值，min 时不可比。
  - 工程做法：让两路共享**同一套全局深度→[−1,1] 映射**（用整个 cascade 的 near/far），保证 warp(t) 在两路一致、可直接 min。
- **方差下限 minVariance**：`minVariance = (VSMBias·c·warp(t))²` 随 warp 导数缩放 [2]。深度范围不同 → warp 量级不同 → 两路 minVariance 量级不同，需各自计算，不能共用常数；否则远路 minVariance 偏小（噪声漏光）或近路偏大（半影过暗）。

## 1.6 漏光与接缝在"两路合并"时是否被放大

- **漏光**：min 合并**不放大**单路已有漏光（取更黑的，方向是抑制）。风险是 1.2(b) 的"双半影并排偏亮"——温和、固有，可用各路自己的 LBR 先压再 min。两路 LBR `amount` 应一致。
- **接缝**：
  - **静/动接缝**：动态物脚下"接触阴影"处两路深度范围交叠，若归一化/bias 不一致，min 切换会亮度跳变或漏光带。对策：两路共用归一化与 bias、接触区小范围 dither。
  - **级联接缝**：每级各有静/动两套 EVSM，共 2×N 张图；cascade 过渡 blend 时要同时 blend 两路**可见性结果**（在 V 空间 blend，不在矩空间），相邻级 c/LBR/bias 一致。

## 1.7 可行方案总结（实现指引）

```
渲染阶段:
  静态 EVSM[cascade]  ← 缓存, 仅静态物体, 相机/光稳定时滚动复用(CSM scrolling)
  动态 EVSM[cascade]  ← 每帧, 仅动态物体, 可半分辨率 + EVSM2
  两路各自: 写 (warp+, warp+², warp−, warp−²) → blur(可分离高斯) → mipmap
  两路共用同一深度→[−1,1] 归一化 与 同一 c(pos/neg)

光照采样阶段(每片元):
  Vs = EVSM4_sample(静态)  // 内部 min(posContrib,negContrib) + LBR
  Vd = EVSM4_sample(动态)
  V  = min(Vs, Vd)         // 默认: 保守, "一前一后"精确; 与 EVSM4 内部同逻辑
  // 可选(若几何上静/动遮挡多为并排独立): V = Vs * Vd
  // 进阶(LVSM 思路): 按片元深度落在静态层还是动态层选取对应路
  级联过渡: 在 V 空间对相邻 cascade 的 V blend
```

精度：两路均 fp32（EVSM 必须），c_pos≤42 / c_neg 取 5 量级，LBR amount≈0.25，VSMBias≈0.01 [2][4]。

---

## Sources

[1] **GPU Gems 3, Ch.8 — Summed-Area Variance Shadow Maps (Lauritzen)** — https://developer.nvidia.com/gpugems/gpugems3/part-ii-light-and-shadows/chapter-8-summed-area-variance-shadow-maps — VSM 矩/μ/σ²、单边 Chebyshev、单平面取等、M2 偏导补偿、漏光成因(a/b)与 linstep LBR、线性度量、可预滤波原理、SAT 精度、N 遮挡者阶跃不可根除漏光。
[2] **TheRealMJP/Shadows — VSM.hlsl & Mesh.hlsl (MIT)** — https://raw.githubusercontent.com/TheRealMJP/Shadows/master/Shadows/VSM.hlsl — ChebyshevUpperBound 全码、ReduceLightBleeding=Linstep、GetEVSMExponents(fp16→5.54/fp32→42)、WarpDepth、EVSM4=min(pos,neg)、minVariance=(VSMBias·c·warp)²。
[3] **TheRealMJP — Shadow Maps blog** — https://therealmjp.github.io/posts/shadow-maps/ — VSM 均值/方差估遮挡、漏光为最大问题、EVSM=VSM+指数翘曲需高精度浮点、正指数 42/负 5、各模式存储开销表。
[4] **TheRealMJP — Shadow Sample Update blog** — https://therealmjp.github.io/posts/shadow-sample-update/ — 32bit EVSM 最高质量(128bit/texel)、EVSM16 伪影、翘曲 clamp 到 10 致 fp16 溢出修正、LBR 0.25、bias 取值。
[5] **Annen et al. — Exponential Shadow Maps (GI 2008)** — https://jankautz.com/publications/esm_gi08.pdf — f=e^{−c(d−z)}=e^{−cd}·e^{cz} 可分离、c=80 对 fp32 最优、d−z≥0 假设违反时 clamp/PCF fallback（~3–9%像素）、几乎免 polygon offset。
[6] **Peters & Klein — Moment Shadow Mapping (I3D 2015)** — https://momentsingraphics.de/Media/I3D2015/MomentShadowMapping.pdf — 4 幂矩、广义矩问题、Hamburger 4MSM 全算法、偏置 ε、优化量化(16bit 64bit/texel)、**EVSM lower bound 非 sharp**、**LVSM 解矩问题**、4 矩≈2 Dirac、MSAA 适用性、运行期 16bit MSM≈32bit VSM。
[7] **Peters & Klein — Beyond Hard Shadows (I3D 2016)** — https://momentsingraphics.de/I3D2016.html — MSM 扩展到单次散射/软阴影/半透明遮挡者(替代 Fourier opacity)。仅取 landing 摘要。
[8] **Insomniac "CSM Scrolling" (SIGGRAPH 2012)** — https://advances.realtimerendering.com/s2012/insomniac/Acton-CSM_Scrolling(Siggraph2012).pdf — 整个世界视作 mega shadow texture、缓存上帧静态、滚动复用、只重画变更几何——为"静态 EVSM 缓存 + 动态每帧重画"提供缓存侧机制背景。
[9] **szaboeme/layered_variance_shadow_maps (GitHub)** — https://github.com/szaboeme/layered_variance_shadow_maps — LVSM 开源实现，分层降漏光（与静/动合成无关，但分层思路可借鉴）。
[10] **Martin Cap — EVSM project** — https://martincap.io/projects/evsm/ — 按 Donnelly/Lauritzen 与 Lauritzen/McCool 的 VSM+EVSM 实现，无静/动合成。
[11] **Unity Discussions — RequestShadowMapRendering for static/dynamic separately** — https://discussions.unity.com/t/requestshadowmaprendering-for-static-dynamic-objects-separately/861966 — 标准 depth 图的静/动缓存拆分讨论（非 variance）。

## Gaps（研究流 B + 主 agent 确认）

- **有文献直接支撑**：Q4 全部核心机理（VSM/Chebyshev/漏光/LBR/预滤波/精度 [1]；EVSM 翘曲与 EVSM4 min 公式与常数 [2][3][4]；ESM 可分离与 c=80 [5]；MSM 四矩/Hamburger/量化/与 EVSM-LVSM 关系 [6]）均经原文核实。
- **合理推导（非直接文献）**：Q5「静/动 EVSM 分开存、采样阶段合并」**经 2026-06-28 补搜确认无任何专门文献**。`min(V_static,V_dynamic)` 正确性论证从两条文献事实推导（各路是保守上界 [6]；EVSM4 自身用 min 合并两个 Chebyshev [2]）；「并排该乘、前后该 min」「动态路可半分辨率/EVSM2/仍需 blur」「两路共用归一化与 c」「接缝在 V 空间 blend」「LVSM 式按深度分层选路」均为工程推导建议，落地需自测。
- **未能取到一手**：原始 VSM 论文(Donnelly & Lauritzen I3D 2006，punkuser.net DNS 失败) 与 LVSM 原文(Lauritzen & McCool 2008) 全文未直取；内容经 GPU Gems 3 ch.8（Lauritzen 本人执笔）[1] 与 MSM 论文转述 [6] 间接覆盖。WebSearch 工具本次全程 400 不可用，检索靠 WebFetch + DuckDuckGo HTML 完成。
