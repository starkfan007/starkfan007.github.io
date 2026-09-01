 Hybrid VST：面向端侧极暗去噪的鲁棒VST方案 - AIISP - AXERA          {"@context":"http://schema.org","@type":"WebSite","url":"http://soc.aixin-chip.com","potentialAction":{"@type":"SearchAction","target":"http://soc.aixin-chip.com/search?q={search\_term\_string}","query-input":"required name=search\_term\_string"}}                                                            {"@context":"http://schema.org","@type":"QAPage","name":"Hybrid VST：面向端侧极暗去噪的鲁棒VST方案","mainEntity":{"@type":"Question","name":"Hybrid VST：面向端侧极暗去噪的鲁棒VST方案","text":"Hybrid VST：面向端侧极暗去噪的鲁棒VST方案\\n\\nIntroduction\\n\\n背景\\n\\n在很多端侧 ISP 去噪架构中，VST (Variance Stabilizing Transformation) 是算法得以高效运行的核心组件。VST&hellip;","upvoteCount":5,"answerCount":0,"dateCreated":"2026-02-12T14:43:17.098Z","author":{"@type":"Person","name":"fenghansen"}}} 

[

## AXERA

](/)

# [Hybrid VST：面向端侧极暗去噪的鲁棒VST方案](/t/topic/11277)

[AIISP](http://soc.aixin-chip.com/c/126-category/126) 

[noise-model](http://soc.aixin-chip.com/tag/noise-model)

[fenghansen](http://soc.aixin-chip.com/u/fenghansen)    2026年02月12日 14:54 #1

# Hybrid VST：面向端侧极暗去噪的鲁棒VST方案

## Introduction

### 背景

在很多端侧 ISP 去噪架构中，VST (Variance Stabilizing Transformation) 是算法得以高效运行的核心组件。VST利用了散粒噪声“均值=方差”的统计特性，通过结合泊松高斯噪声参数的加权开根，巧妙地破除了噪声的信号相关性，让困难的真实噪声去噪任务转化为了类高斯去噪任务，从而大大降低了Raw图像去噪的难度。

然而，在极暗条件下，信号强度往往只有个位数光子（< 5 e-）。此时，传统的 Anscombe 变换及其变体[GAT](https://ieeexplore.ieee.org/abstract/document/6212354)在零点附近存在强非线性，因此会暴露出严重的数值稳定性问题。尽管本人在[YOND](https://arxiv.org/abs/2506.03645)中提出的EM-VST能对此有极大的环节，但是该方案强依赖准确的噪声参数，这对于低端sensor和超极暗场景而言难以保证。故而，如何处理极暗场景始终是VST-based去噪方法的核心问题。

### 现有方法的局限性

1.  **极暗区的非线性失真（Bias & Color Shift）**：标准的 VST（如 [GAT](https://ieeexplore.ieee.org/abstract/document/6212354)）在接近 0 点处具有无穷大的导数。根据 Jensen 不等式 (E\[f(x)\] \\neq f(E\[x\]))，这种强非线性会导致变换后的信号均值发生显著偏移。在 ISP pipeline 中，这直接体现为暗部死黑（Black Crush）或严重的偏色（Hue Shift）。
2.  **EM-VST 的工程落地痛点**：我们之前的工作[EM-VST](https://arxiv.org/abs/2506.03645)提出了“期望匹配”的思想，理论上解决了 Bias 问题。但在实际工程落地中，Sensor 往往存在标定误差或温漂导致的 **BLE (Black Level Error)**。EM-VST 本质上是在利用噪声模型计算一个精密的修正量，这要求输入的 K 和 \\sigma 以及当前的 Black Level 必须极其精准。一旦出现参数失配，EM-VST 的修正逻辑可能会导致过度补偿，从而引发极暗处的偏色或模糊。
3.  **LUT 采样的不友好**：工程上 VST 通常以 LUT (Look-Up Table) 形式部署。传统 VST 曲线在低光区斜率极陡，曲率极大。为了保证插值精度，必须在低光区设置极高密度的采样点，这挤占了有限的 LUT 资源。

### 我们的 Motivation

为了解决上述问题，我们需要一种“既要又要”的方案：

+   **鲁棒性**：在极低光区放弃对“方差绝对稳定”的执念，转而追求**线性度**。线性变换不会导致噪图与干净图均值有偏（VST会），自然也对 Black Level 的漂移不敏感（线性增益不会像VST一样放大零点偏移）。
+   **数理性质**：在中高光区保留 VST 的特性，将信号相关噪声规范为方差稳定的类高斯噪声，压缩去噪图像的动态范围，有利于网络去噪。
+   **工程友好**：整体曲线 C^1 连续，且在最难处理的低光区是线性的，这对 LUT 采样极其友好（两点决定一条直线，插值零误差）。

* * *

## Related Work：从 Foi 到 EM-VST 再到 Hybrid VST

为了讲清楚 Hybrid VST 的定位，我们需要回顾一下Raw去噪相关的 VST 关键工作：

1.  **Post-Correction (Foi et al.)**

I\_{\\hat{\\sigma}}(\\hat{x}) =\\frac{1}{4}\\hat{x}^{2}+\\sqrt{\\frac{3}{32}}\\hat{x}^{-1} -\\frac{11}{8}\\hat{x}^{-2}+\\sqrt{\\frac{75}{128}}\\hat{x}^{-3}-\\frac{1}{8}- \\hat{\\sigma}^2

+   **思路**：先做标准 VST \\to 神经网络去噪 \\to 在逆变换阶段，假设去噪后的结果 \\hat{x} 是 GT，将Bias矫正融入IAT逆变换 I\_{\\hat{\\sigma}}(\\hat{x}) 。
+   **问题**：这是一种“亡羊补牢”。由于 VST 变换本身引入了 Bias，输入给神经网络的分布就是歪的（均值漂移），导致网络处理极暗信号时能力受限。且逆变换时依然存在计算误差。  
    
    [![image](//soc.aixin-chip.com/uploads/default/optimized/3X/7/a/7adc95ba2bac98fd7313f6ea1d9d26d2cfcb417b_2_690x254.jpeg)
    
    image1451×535 295 KB
    
    ](//soc.aixin-chip.com/uploads/default/original/3X/7/a/7adc95ba2bac98fd7313f6ea1d9d26d2cfcb417b.jpeg "image")
    

2.  **Pre-Correction (EM-VST, Previous Work)**

f\_{\\hat{\\sigma}}(z)= 2 \\sqrt{z+\\frac{3}{8}+\\hat{\\sigma}^2}, z = \\frac{y - \\mu}{K}, \\hat{\\sigma} = \\frac{\\sigma}{K}, \\mu = BLE = 0\\\\ \\epsilon\_x=e\_{\\hat{\\sigma}}(x)=E \\left(f\_{\\hat{\\sigma}}(z) \\mid x\\right) - f\_{\\hat{\\sigma}}(x)\\\\ \\epsilon\_y = e\_{\\hat{\\sigma}}(y) = e\_{\\hat{\\sigma}}(x+n) = e\_{\\hat{\\sigma}}(x) + e^{'}\_{\\hat{\\sigma}}(x) \\cdot n + \\mathcal{O}(n^2)\\\\ E(\\epsilon\_y) = E\\left(e\_{\\hat{\\sigma}}(x) + e^{'}\_{\\hat{\\sigma}}(x) \\cdot n + \\mathcal{O}(n^2)\\right) = E(\\epsilon\_x) + 0 + E(\\mathcal{O}(n^2)) \\approx E(\\epsilon\_x)

+   **思路**：直接在变换阶段解决问题。利用噪图 y 估算VST导致的偏差 \\epsilon\_x ，基于 E(\\epsilon\_y) \\approx E(\\epsilon\_x) 的近似，在去噪前用 E \\left(f\_{\\hat{\\sigma}}(z) \\mid x\\right) - \\epsilon\_y 提前矫正偏差，构造一个 E\[f(y)\] = f(x) 的变换。
+   **优势**：输入给网络的信号是无偏的，保证了色彩正确性。
+   **隐患**：成也萧何败萧何。它太依赖“用噪图估算 Bias”这一步的准确性了。如果标定的噪声模型与实际 Sensor 状态不符（如 BL 漂移），估算出的 Bias 就会出错，导致矫枉过正。  
    
    [![image](//soc.aixin-chip.com/uploads/default/optimized/3X/d/7/d7fa32061ad1e37960c1430d5a7d7344a16de66d_2_690x222.jpeg)
    
    image1517×490 127 KB
    
    ](//soc.aixin-chip.com/uploads/default/original/3X/d/7/d7fa32061ad1e37960c1430d5a7d7344a16de66d.jpeg "image")
    

3.  **Hybrid VST (Ours)**

+   **思路**：**避其锋芒**。既然极低光区的 Bias 修正如此困难且敏感，我们为何非要在那里做非线性变换？
+   **策略**：在低光区（如 < 20 e-），直接回退到**线性变换**。线性变换没有 Bias，不需要修正，因此绝对鲁棒。只在信号足够强（> 20 e-）、信噪比足够高时，才平滑过渡到 VST 变换。
+   **代价**：在极低光区放弃了“方差归一化”的数学完美性（噪声方差不再是 1，而是随信号变化），换取了工程上的绝对稳定和无偏。

* * *

## Method

### 1\. 核心变换公式

我们将变换函数 f(x) 定义为分段形式。关键约束在于分段点 x\_{th} 处的 **C^1 连续性**（值相等且导数相等）。

f(x) = \\begin{cases} ax + b, & x \\le x\_{th} \\\\ 2\\sqrt{\\frac{x}{K} + \\left(\\frac{3}{8} + \\frac{\\sigma^2}{K^2}\\right)}, & x > x\_{th} \\end{cases}

**参数定义**：

+   **常数项**： C = \\frac{3}{8} + \\frac{\\sigma^2}{K^2}
+   **切换阈值 (DN)**： x\_{th} = Th\_{photon} \\cdot K
+   **线性斜率**： a = \\frac{1}{K\\sqrt{\\frac{x\_{th}}{K} + C}}
+   **线性截距**： b = 2\\sqrt{\\frac{x\_{th}}{K} + C} - a \\cdot x\_{th}

### 2\. 平滑偏差修正 (Smooth Bias Correction)

虽然线性区天然无偏，但高光区的根号变换仍存在 Bias。为了防止在 x\_{th} 处出现 Bias 的跳变分层，我们利用 **Cosine 权重** 将 Bias 修正项温和地“挂载”上去。

B(x) = w(x) \\cdot \\text{Bias}\_{cf}(x)

+   **权重 w(x)**：在过渡区基于噪声标准差 \\sigma\_{total} 进行 Cosine 插值。在纯线性区 w=0 （完全不修正，绝对鲁棒），在 VST 区 w=1 （完全修正，方差稳定）。
+   **Bias项 \\text{Bias}\_{cf}(x)**：采用 Foi 的渐近线闭式解，计算高效。

\\text{Bias}\_{cf}(x) = 2\\sqrt{\\hat{y}} \\left( -\\frac{1}{8}m\_1 + \\frac{1}{16}m\_2 - \\frac{5}{128}m\_3 \\right)

其中各项中间变量定义为：

+   y = \\frac{x}{K} ， \\hat{y} = y + C
+   m\_1 = \\frac{y + \\sigma\_{norm}^2}{\\hat{y}^2}
+   m\_2 = \\frac{y}{\\hat{y}^3}
+   m\_3 = \\frac{y + 3(y + \\sigma\_{norm}^2)^2}{\\hat{y}^4}

### 3\. 逆变换公式 (Inverse Hybrid VST)

逆变换 f^{-1}(y) 同样遵循分段逻辑，用于将修正后的 VST 域信号还原：

f^{-1}(y) = \\begin{cases} \\frac{y - b}{a}, & y \\le y\_{th} \\\\ K \\left( \\left(\\frac{y}{2}\\right)^2 - C \\right), & y > y\_{th} \\end{cases}

其中 y\_{th} = f(x\_{th}) 为 VST 域对应的切换阈值。

* * *

## Code

```python
def get_vst_params(K, sigma, th_photon=20.0):
    """
    计算混合 VST 的分段几何参数与线性系数。
    支持 K 与 sigma 为标量、numpy 数组或 torch 张量。
    """
    c = 3/8 + (sigma/K)**2
    x_th = th_photon * K
    z_th = th_photon + c
    a = 1 / (K * z_th**0.5)
    y_th = 2 * z_th**0.5
    b = y_th - a * x_th
    return a, b, x_th, y_th, c

def hybrid_vst(x, K, sigma, th_photon=20.0):
    """
    前向混合 VST 变换：x(DN) -> z(VST Domain)。
    低光区(<=th_photon)采用线性插值，高光区采用 Anscombe 变换。
    """
    lib = torch if torch.is_tensor(x) else np
    a, b, x_th, _, c = get_vst_params(K, sigma, th_photon)
    
    y_linear = a * x + b
    y_vst = 2.0 * (x/K + c).clip(0, None)**0.5
    
    return lib.where(x <= x_th, y_linear, y_vst)

def hybrid_inverse_vst(y, K, sigma, th_photon=20.0):
    """
    逆向混合 VST 变换：z(VST Domain) -> x(DN)。
    根据 VST 域阈值 y_th 自动选择线性还原或平方还原。
    """
    lib = torch if torch.is_tensor(y) else np
    a, b, _, y_th, c = get_vst_params(K, sigma, th_photon)
    
    x_linear = (y - b) / a
    x_vst = K * ((y/2)**2 - c)
    
    return lib.where(y <= y_th, x_linear, x_vst)

def get_hybrid_bias_smooth(x, K, sigma, th_photon=20.0, blend_scale=1.0, tail_ratio=1.25):
    """
    计算带 Cosine 平滑过渡的 EM-VST 偏差修正图。
    
    权重采用 Cosine 插值实现从 0 (线性区) 到 闭式解 (VST区) 的平滑挂载。
    过渡带宽基于阈值处的总噪声标准差 (Total Noise Std) 动态确定。
    """
    lib = torch if torch.is_tensor(x) else np
    
    # 计算基于闭式解的高光偏差
    bias_high = close_form_bias(x.clip(0, None), sigGs=sigma, K=K)
    # 计算基于物理噪声水平的过渡起止点 (光子域)
    noise_std = ((sigma/K)**2 + th_photon)**0.5
    w_b = noise_std * blend_scale
    b_start = th_photon - w_b
    b_end = th_photon + w_b * tail_ratio
    
    # 计算归一化进度 alpha 并应用 Cosine 权重
    alpha = ((x/K - b_start) / (b_end - b_start)).clip(0, 1)
    
    # Cosine 权重: 0.5 - 0.5 * cos(pi * alpha)
    # 使用 np.pi 是安全的，torch 可兼容标量乘法；使用 lib.cos 确保后端一致
    weight = 0.5 - 0.5 * lib.cos(3.141592653589793 * alpha)
    
    return weight * bias_high

def close_form_bias(x, sigGs=25.853043, K=24.48128):
    y = x / K
    sigma = sigGs / K
    y_hat = y + 3/8 + sigma**2
    m1 = (y+sigma**2)/y_hat**2
    m2 = (y)/y_hat**3
    m3 = ((y)+3*(y+sigma**2)**2)/y_hat**4
    bias = 2*y_hat**0.5*(-1/8*m1+1/16*m2-5/128*m3)

    return bias
```

* * *

## Experiments

### 1\. 变换函数形态与分段逻辑

通过对 C^1 连续条件的约束，我们确定了 Hybrid VST 的几何形态。下图展示了在给定噪声参数（ K=12, \\sigma=18 ）及阈值（ 25 e^- ）下的变换曲线：

+   **线性区 (Input \\le x\_{th})**：变换表现为 y = ax + b 。在该区域，由于函数斜率恒定，均值期望保持线性传递。这意味着即便在传感器black level矫正不彻底、信号存在负值或微小偏移的情况下，变换也不会加剧偏移。
+   **VST 区 (Input \> x\_{th})**：随着信号增强，曲线平滑切入平方根域。此时通过预计算的参数 $a, b$，确保了分段点处的函数值与导数完全一致，规避了由于曲线不连续可能引入的异常。

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/1/8/183cb62e49d28faa6e50cd409d89232392e88170_2_690x427.png)

image860×533 89.5 KB

](//soc.aixin-chip.com/uploads/default/original/3X/1/8/183cb62e49d28faa6e50cd409d89232392e88170.png "image")

### 2\. 偏差函数（Bias）的数值验证

为了验证 `get_hybrid_bias_smooth` 函数生成的修正项是否贴合物理真实，我们引入数值积分器 `calculate_true_bias` 作为 Ground Truth。

+   **观察**：数值积分结果（橙色）证实，在线性区内偏差严格为 0，在VST区偏差较小（与VST相同）。
+   **平滑过渡**：在切换点附近，我们利用 Cosine 权重实现的闭式解挂载（蓝色）与积分值高度契合。在推理阶段，直接使用噪图 y 估算的平滑 Bias 修正量是足以高效消除非线性漂移。

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/4/6/46cb87051e2891df24eb03f774f052b7ddc67834_2_690x498.png)

image726×524 83.5 KB

](//soc.aixin-chip.com/uploads/default/original/3X/4/6/46cb87051e2891df24eb03f774f052b7ddc67834.png "image")

### 3\. 蒙特卡洛模拟：均值重建稳定性

我们通过 4 \\times 10^6 次蒙特卡洛采样，模拟了10bit ISO-6400信号经过“前向变换 -> 偏差修正 -> 逆变换”后的重建过程。

+   **重建误差 (Bias Value)**：实验结果显示，在全量程范围内（包含线性区与 VST 区），10bit ISO-6400重建信号与原始信号的差值（Reconstruction Bias）始终维持在 0 附近。最大的波动出现在非极暗区，偏移峰值仅0.2%（0.4/200）。
+   **结论**：这验证了 Hybrid VST 的核心设计预期：**在极暗区通过线性化彻底规避偏差，在非线性区通过平滑挂载实现期望匹配。** 这种结构设计使得算法对标定参数的微小扰动具有极强的容忍度，确保了极暗环境下画面的色彩重建精度。

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/1/b/1b8782980b5a2da90f0c522a7bc09986241414aa_2_690x426.png)

image829×513 108 KB

](//soc.aixin-chip.com/uploads/default/original/3X/1/b/1b8782980b5a2da90f0c522a7bc09986241414aa.png "image")

### 4\. HybridVST实战效果对比

暗区的细节更清晰、色彩更还原（本质上是因为去噪力度没错配）  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/c/b/cb973d0ce6395447555950d83ec11838a36d5043_2_690x174.jpeg)

image1839×465 103 KB

](//soc.aixin-chip.com/uploads/default/original/3X/c/b/cb973d0ce6395447555950d83ec11838a36d5043.jpeg "image")

  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/d/6/d6fdc508bd356f6b2f2b4bccbd854ace26e9acba_2_690x176.jpeg)

image1838×471 137 KB

](//soc.aixin-chip.com/uploads/default/original/3X/d/6/d6fdc508bd356f6b2f2b4bccbd854ace26e9acba.jpeg "image")

  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/a/1/a168138595afc88cb39dea859fc24ceea40b6331_2_690x174.jpeg)

image1839×465 192 KB

](//soc.aixin-chip.com/uploads/default/original/3X/a/1/a168138595afc88cb39dea859fc24ceea40b6331.jpeg "image")

* * *

## 总结与讨论

简单小结一下这一系列VST方案演进的核心逻辑：

+   **对原版 VST 的诉求**：我们希望排除噪声的信号相关性（方差稳定），同时压缩动态范围。但实际上低光区域处处是问题。
+   **EM-VST 的尝试**：试图在全域实现“期望匹配”。但在极暗信号（低 SNR）区，这种修正对噪声参数（尤其是 Black Level）的准确性有较高的要求。
+   **Hybrid VST 的权衡**：
    1.  **避开非线性陷阱**：在最容易出偏色问题的极暗区（线性区），由于没有高阶导数，计算结果天然无偏。
    2.  **放宽约束换取鲁棒性**：我们主动放弃了极暗区的“方差绝对稳定”（允许噪声随信号微弱变化），换取了对 Sensor 温漂和标定误差的绝对耐受力。
    3.  **衔接与平滑**：通过计算分段点处的衔接偏差，保证了亮暗过渡区的视觉平滑。

这种设计保证了 Hybrid VST 在具备鲁棒性、可控性的同时，能够兼顾亮暗两端的去噪能力。

虽然这种在低光区残留的“非单位方差”可能会对基于 SNR 引导的 Diffusion Model 拓展造成一定的理论负担，但在绝大多数端侧 CNN 判别式去噪模型中，Hybrid VST 展现出了显著更优的稳定性与易用性。

 

  5赞

 

[首页](/) [分类](/categories) [FAQ/指引](/guidelines) [服务条款](/tos) [隐私政策](/privacy)

采用 [Discourse](https://www.discourse.org)，启用 JavaScript 以获得最佳体验