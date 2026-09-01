 针对时域非独立噪声的引导值矫正策略（fpncorr） - AIISP - AXERA          {"@context":"http://schema.org","@type":"WebSite","url":"http://soc.aixin-chip.com","potentialAction":{"@type":"SearchAction","target":"http://soc.aixin-chip.com/search?q={search\_term\_string}","query-input":"required name=search\_term\_string"}}                                                            {"@context":"http://schema.org","@type":"QAPage","name":"针对时域非独立噪声的引导值矫正策略（fpncorr）","mainEntity":{"@type":"Question","name":"针对时域非独立噪声的引导值矫正策略（fpncorr）","text":"低质量sensor的模式噪声问题，与其在VST引导去噪路线下的解决方案\\n\\n问题分析\\n\\n背景\\n\\n基于sensor噪声的i.i.d.假设，为2DNR服务的噪声建模理论上可以直接迁移到3DNR，无需变更。\\n\\n然而，由于垃圾sensor难以去除的fpn和&hellip;","upvoteCount":4,"answerCount":0,"dateCreated":"2025-11-10T07:02:06.150Z","author":{"@type":"Person","name":"fenghansen"}}} 

[

## AXERA

](/)

# [针对时域非独立噪声的引导值矫正策略（fpncorr）](/t/topic/11053)

[AIISP](http://soc.aixin-chip.com/c/126-category/126) 

[噪声模型](http://soc.aixin-chip.com/tag/噪声模型)

[fenghansen](http://soc.aixin-chip.com/u/fenghansen)    2026年06月8日 08:08 #1

# 低质量sensor的模式噪声问题，与其在VST引导去噪路线下的解决方案

* * *

## 问题分析

### 背景

基于sensor噪声的i.i.d.假设，为2DNR服务的噪声建模理论上可以直接迁移到3DNR，无需变更。  
然而，由于垃圾sensor**难以去除的fpn**和**画蛇添足的内部去噪**，实际情况非常复杂。

![](//soc.aixin-chip.com/user_avatar/soc.aixin-chip.com/fenghansen/40/38562_2.png)[\[prototype\] 超低成本空域降噪](//soc.aixin-chip.com/t/topic/10536/3)

> 忽然既视感！怀疑某些坏厂商在sensor出raw前，在按行异步读出数据时做了横向的滤波，感觉形态有点像
> 
> ![f5be7a0e9b6a6b1c652285302e6dc4e2](//soc.aixin-chip.com/uploads/default/optimized/3X/5/5/557e8629f1309a693ea84ddb3a9c838fd53f7c5f_2_690x182.jpeg)
> 
> ![image](//soc.aixin-chip.com/uploads/default/optimized/3X/5/7/57e06a627570a8d7aababfc03003cc185b1e1446_2_690x181.png)
> 
> 正经sensor做FFT一般也就中心有个细竖线，随机行噪声
> 
> ![9eb19f7b7bf9b921092e5ae6206f0405](//soc.aixin-chip.com/uploads/default/optimized/3X/a/3/a308e7a4d03776f649f1ef8188be746bb53418e7_2_690x182.png)
> 
> ![db6ee78b79fa8aebd0d512e5a640308e](//soc.aixin-chip.com/uploads/default/optimized/3X/2/b/2bfe326505c23e4da084e581de7d97590bab9e81_2_690x277.jpeg)

SC系列常见问题回顾:

> #### 直接2D方案迁移3D的问题分析
> 
> 仅替换噪声模型，更清晰且静区模式消失，但mask边缘等区域有artifacts
> 
> ![image](//soc.aixin-chip.com/uploads/default/optimized/3X/2/c/2c7041d7a9aabd1d1dd1b21c7c9527125998b059_2_690x246.jpeg)
> 
> 基于不同噪声模型下的差异，我们可以推断是FPN导致的降噪力度错配
> 
> #### 3D去噪中的噪声模型问题
> 
> +   无融合（noisy3d）的RNN帧循环去噪模型无此问题，但信息损失严重
> +   本质上是noisy3d上非独立噪声形态OOD了，噪声模型调整力度解不掉
> +   需要在训练时构造多帧叠加过程，模拟真实的多帧非独立噪声
> 
> ![image](//soc.aixin-chip.com/uploads/default/optimized/3X/d/a/daf79fdcc392de600a308434da29a03210a84e43_2_690x161.png)  
> 中间图公式为 \\dfrac{std(\\sum\_n{noisy})}{\\sqrt{n}} ，独立噪声应为水平直线。右图为基于方差计算的版本， ρ^2 为方差比值。

#### FPN系数的计算

代码

```python
import numpy as np
import glob
from natsort import natsorted
import matplotlib.pyplot as plt
from tqdm import tqdm

paths = natsorted(glob.glob(f'{bias_dir}/49.6gain/*.raw'))
n = len(paths)

# 原代码逻辑分析：
# 1. 对前idx+1帧计算每个像素的时间平均：raws[:idx+1].mean(axis=0)
# 2. 对这个平均图像计算空间方差：raw.var()
# 3. 计算：(idx+1) * var / vars.mean()

# 我们需要：1) 累积每个像素的均值 2) 计算这些像素均值的空间方差

# 变量初始化
sum_single_frame_vars = 0.0
sum_single_frame_means = 0.0
sum_single_frame_vars_diff = 0.0
sum_single_frame_means_diff = 0.0

# 用于累积每个像素的均值
pixel_cumulative_sum = np.zeros((H, W), dtype=np.float64)  # 用float64提高精度
pixel_cumulative_sum_diff = np.zeros((H, W), dtype=np.float64)  # 用float64提高精度

xs = np.arange(n) + 1
ys = np.zeros(n)
xs_diff = np.arange(n//2)*2 + 1
ys_diff = np.zeros(n//2)

print("单遍扫描计算...")
for i, path in enumerate(tqdm(paths, desc='Processing frames')):
    # 读取并处理当前帧
    raw = np.fromfile(path, np.uint16).reshape(H, W)
    raw = (raw.astype(np.float32) - 4096) / (65535 - 4096)
    current_frame = raw * (wp - bl)
    if i % 2 == 0: prev_frame = current_frame
    # 计算当前帧的统计量
    current_mean = current_frame.mean()
    current_var = current_frame.var()
    sum_single_frame_means += current_mean
    sum_single_frame_vars += current_var
    # 累积每个像素的值
    pixel_cumulative_sum += current_frame
    # 计算当前像素平均值图像
    pixel_average_image = pixel_cumulative_sum / (i + 1)
    # 计算这个平均图像的空间方差
    avg_image_var = pixel_average_image.var()
    # 计算当前平均单帧方差
    avg_single_var = sum_single_frame_vars / (i + 1)
    # 计算 ys = (N * var_of_averages) / avg_single_frame_var
    ys[i] = ((i + 1) * avg_image_var) / avg_single_var

    # 计算差分帧相关统计量
    if i % 2 == 1:
        diff_frame = current_frame - prev_frame
        diff_mean = diff_frame.mean()
        diff_var = diff_frame.var()
        sum_single_frame_means_diff += diff_mean
        sum_single_frame_vars_diff += diff_var
        pixel_cumulative_sum_diff += diff_frame
        pixel_average_image_diff = pixel_cumulative_sum_diff / ((i + 1) // 2)
        avg_image_var_diff = pixel_average_image_diff.var()
        avg_single_var_diff = sum_single_frame_vars_diff / ((i + 1) // 2)
        ys_diff[i // 2] = (((i + 1) // 2) * avg_image_var_diff) / avg_single_var_diff

# 最终的平均单帧方差
avg_single_frame_var = sum_single_frame_vars / n
avg_single_frame_var_diff = sum_single_frame_vars_diff / (n // 2)

# 绘图
plt.figure(figsize=(5, 3))
plt.title('Real SC850 Noise (Dark Frame)')
plt.xlabel("nframes")
plt.ylabel("ρ2")
plt.grid('on')

plt.plot(xs, np.ones_like(xs), 'C1--', 
         label=f'Single Frame ({avg_single_frame_var:.2f})')

plt.scatter(xs, ys, color='C0', label='Real', s=10)

# 线性拟合
coeffs = np.polyfit(xs, ys, deg=1)
fit_line = np.poly1d(coeffs)
plt.plot(xs, fit_line(xs), 'r-', 
         label=f'Fit: y={coeffs[0]:.4f}x+{coeffs[1]:.4f}')

plt.legend()
plt.tight_layout()
plt.show()

print(f"平均单帧方差: {avg_single_frame_var:.4f}")
print(f"线性拟合结果: y = {coeffs[0]:.6f}x + {coeffs[1]:.6f}")
print(f"最终ys值: {ys[-1]:.6f}")

# 绘图
plt.figure(figsize=(5, 3))
plt.title('Real SC850 Noise (Diff Dark Frames, without FPN)')
plt.xlabel("nframes")
plt.ylabel("ρ2")
plt.grid('on')

plt.plot(xs_diff, np.ones_like(xs_diff), 'C1--', 
         label=f'Single Frame ({avg_single_frame_var:.2f})')

plt.scatter(xs_diff, ys_diff, color='C0', label='Real', s=10)

# 线性拟合
coeffs = np.polyfit(xs_diff, ys_diff, deg=1)
fit_line = np.poly1d(coeffs)
plt.plot(xs_diff, fit_line(xs_diff), 'r-', 
         label=f'Fit: y={coeffs[0]:.4f}x+{coeffs[1]:.4f}')

plt.legend()
plt.tight_layout()
plt.show()

print(f"平均单帧Diff方差: {avg_single_frame_var_diff:.4f}")
print(f"线性拟合结果: y = {coeffs[0]:.6f}x + {coeffs[1]:.6f}")
print(f"最终ys值: {ys_diff[-1]:.6f}")
```

#### 代码样例输出

```auto
单遍扫描计算...
Processing frames: 100%|██████████| 300/300 [00:55<00:00,  5.42it/s]
平均单帧方差: 11.9353
线性拟合结果: y = 0.135124x + 0.915521
最终ys值: 41.415128
平均单帧Diff方差: 20.6426
线性拟合结果: y = -0.000004x + 1.000389
最终ys值: 0.998898
```

![image](//soc.aixin-chip.com/uploads/default/original/3X/8/e/8e37f1afad98f389ba7f3392f74d0554029e5f05.png)  
![image](//soc.aixin-chip.com/uploads/default/original/3X/c/8/c8c5e62277b0f240ffb8f03adcc1835c171dbfa5.png)

### 相关前置工作

当时我们的实践还有所不足，对问题的认识还并不深刻，因此解决方案流于表面。

具体的，我们当时寄希望于单纯通过修正数据来解决问题，即噪声模型的视频化适配。

Video噪声模型适配方案分析:

> +   基于SFRN改
>     +   仅贴暗帧，实现非常简单，不过噪声多样性可能不足，影响解析力
>     +   由于噪声Aug手段有限，对于OOD噪声较敏感，易残留artifacts
> +   基于PMNNP改
>     +   基于PNNP做噪声拆解，记录FPN，建模band- / pixel-wise噪声
>     +   考虑到SC系列有局部模式噪声，以原始pixel-wise噪声为基准，做方差一致的PNNP与SFRN（pixel-wise）融合（α>0.7）
> 
> 𝑁=𝑁\_𝑝+𝑁\_{𝐹𝑃𝑁}+𝑁\_{𝐵𝐿𝐸}+𝑁\_{𝑟𝑜𝑤}+(𝛼𝑁\_{𝑆𝐹𝑅𝑁}+ \\sqrt{1−𝛼^2} 𝑁\_{𝑝𝑛𝑛𝑝} )+𝑁\_𝑞

上述解决方案存在两个局限性：

1.  **分解出来的真实pixel-wise噪声并非完全独立**
    +   用50帧均值作为dark shading其中本就包含部分pixel-wise噪声，会影响独立性
    +   SC系列的局部模式噪声可能带来额外的非独立性 Var(x)+Var(y)<Var(x+y)
2.  **上述解法只修正了train/test数据的OOD问题，却没有修正FPN导致的数据映射冲突**

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/4/d/4dded7f01fb3e3feb18592c72b8777d0b6d19509_2_690x103.png)

image1351×202 19.8 KB

](//soc.aixin-chip.com/uploads/default/original/3X/4/d/4dded7f01fb3e3feb18592c72b8777d0b6d19509.png "image")

当时寄希望于依靠一些NN训练策略（比如帧循环）来解决问题，虽然确实有收益但并不本质，反而把问题搞复杂了。以至于Debug了很长时间，怀疑这怀疑那，走了不少弯路。

在训练集上修改损失函数无法解决本质问题:

> **增加2D loss约束惩罚RNN后变差的区域**
> 
> +   2D loss会抑制RNN导致的错误累计，惩罚3D去噪后比2D去噪结果更糟糕的部分
> +   降低RNN网络对前一帧去噪结果的依赖性
> 
> **2D loss的问题**
> 
> +   2D loss可能开太大了，把前一帧ban掉（置0）竟然效果几乎一样，这样RNN就没有意义了，而且看起来细节还没有noisy3d直接去噪来的清晰，很难绷……
> +   In a word，仍需探索时域循环信息的合理利用方式，强调2d loss的训练方式下实质上真的会退化到原来的2D去噪方案

直到开始尝试SCUNet，对比gru64n+AWGN，才意识到“**并非小NN做不到，而是SNR引导值出了问题**”。

* * *

## 解决思路

+   之前尝试过给NN提供cntmap自己学映射，然而构造无约束，并学不到
+   无法彻底地定义SC的噪声模型，因此只能统一SNR引导值的统计意义σ

* * *

## 数学建模

理想方差： \\sigma\_{\\text{id}}^2 = \\sigma\_{\\text{ps}}^2 + \\sigma\_{\\text{gs}}^2 ， 实际方差： \\sigma\_{\\text{rl}}^2 = \\sigma\_{\\text{ps}}^2 + (\\sigma\_{\\text{gs}}^2 + \\sigma\_{\\text{fpn}}^2)  
其中 \\sigma\_{\\text{ps}}^2 是光子散粒噪声方差， \\sigma\_{\\text{gs}}^2 是高斯（信号无关）噪声方差， \\sigma\_{\\text{fpn}}^2 是固定模式（时不变）噪声方差

假设去噪模型映射合理，若欲使推理时引导值 \\sigma\_{\\text{SNR}} 匹配，则需 **fpncorr**（FPN校正系数）。

泊松噪声独立时， (\\sigma\_{\\text{gs}}^2 + \\sigma\_{\\text{fpn}}^2) 为读噪声的实际标定方差，可通过统计获得。 对于独立同分布（i.i.d.）噪声， N 帧平均后噪声为 \\bar{\\sigma} = \\frac{\\sigma}{N} ，而FPN（固定模式噪声）不随多帧平均变化。

因此，fpncorr的可以表示为

\\rho = \\sqrt{\\frac{\\bar{\\sigma}\_{\\text{rl}}^2}{\\bar{\\sigma}\_{\\text{id}}^2}} = \\sqrt{\\frac{\\left( \\frac{\\sigma\_{\\text{ps}}^2}{N} + \\left( \\frac{\\sigma\_{\\text{gs}}^2}{N} + \\sigma\_{\\text{fpn}}^2 \\right) \\right)}{\\left( \\frac{\\sigma\_{\\text{ps}}^2}{N} + \\frac{\\sigma\_{\\text{gs}}^2}{N} \\right)}} = \\sqrt{\\frac{\\sigma\_{\\text{ps}}^2 + (\\sigma\_{\\text{gs}}^2 + N\\sigma\_{\\text{fpn}}^2)}{\\sigma\_{\\text{ps}}^2 + \\sigma\_{\\text{gs}}^2}}

无光暗帧下的fpncorr系数标定代码（rou2=实际方差/理想方差）

```python
paths = natsorted(glob.glob(f'{bias_dir}/15.5gain/*.raw'))
n = len(paths)
mus, vars = [None]*n, [None]*n
raws = [None] * n
for i in tqdm(range(n)):
    raw = np.fromfile(paths[i], np.uint16).reshape(H, W)
    raw = (raw.astype(np.float32) - 4096) / (65535-4096)
    raws[i] = raw * (wp-bl)
    mus[i], vars[i] = raws[i].mean(), raws[i].var()
    # raws[i] = np.random.randn(*raw.shape) * vars[i]**0.5
    # print(mus[i], sigs[i])
raws = np.array(raws)
mus = np.array(mus)
vars = np.array(vars)
xs = np.arange(n) + 1
plt.figure(figsize=(5,3))
plt.title('Real OS04A10 Noise (Dark Frame)')
plt.xlabel("nframes")
plt.ylabel("rou2")
plt.grid('on')
plt.plot(xs, np.ones_like(xs), 'C1--', label=f'Single Frame ({vars.mean():.2f})')
ys = []
for idx in range(n):
    raw = raws[:idx+1].mean(axis=0)
    mu, var = raw.mean(), raw.var()
    ys.append((idx+1) * var / vars.mean())
    # print(f'{mu:.3f}->{mus[:n].mean():.3f}, {sig*n**0.5:.3f}->{sigs[:n].mean():.3f}')
plt.scatter(xs, ys, color='C0', label='Real')
print(np.polyfit(xs, ys, deg=1))
plt.legend()
plt.show()
```

### 一些关键点

+   **噪声模型相关解释**：
    +   (\\sigma\_{\\text{gs}}^2 + \\sigma\_{\\text{fpn}}^2) 定义为读噪声实际标定方差 \\sigma\_{\\text{read}}^2 ，因此多帧平均帧数 N 需从0开始计数（即单帧情况）。
    +   \\sigma\_{\\text{read}}^2 可通过统计直接获取，且能规避局部模式噪声的影响。
    +   \\sigma\_{\\text{ps}}^2 （光子散粒噪声方差）与信号相关，可通过GT信号合成精确值。
+   **实现上的困难**
    +   **推理阶段**：静区可利用前一帧去噪结果替代，但动区缺乏可靠数据支撑。
    +   **训练阶段**：直接使用GT合成噪声相当于“泄露答案”（平坦特征可能引导出GT图形的先验信息）。
+   **类似EM-VST的解决方案**
    +   **基于噪声图的计算方案**：鉴于 \\rho （FPN校正系数）的斜率较低，且 \\sigma\_{\\text{ps}}^2 对系统为非敏感系数，可通过噪声图直接计算 \\sigma\_{\\text{ps}}^2
+   **数据策略**：需剔除空间相关的数据增强（如锐化操作），避免引导值与实际噪声分布失配。

其中读噪声标定 \\sigma\_{\\text{read}}^2 = \\sigma\_{\\text{gs}}^2 + \\sigma\_{\\text{fpn}}^2 ，光子散粒噪声特性 \\sigma\_{\\text{ps}}^2 = KI\_{GT}

* * *

## 补充说明

+   如果带锐化Aug，引导值可能会自动绑到锐化上
+   修正fpn\_corr后，模型对力度控制响应灵敏，可以加力度抑制噪声，鲁棒性有所下降
+   后来发现是训练中见过的数据不够暗，增加LightAug后，问题修复，观感逼近大模型

 

  4赞

 

  [个人未来研究方向规划与展望（2026）](http://soc.aixin-chip.com/t/topic/11217)

[首页](/) [分类](/categories) [FAQ/指引](/guidelines) [服务条款](/tos) [隐私政策](/privacy)

采用 [Discourse](https://www.discourse.org)，启用 JavaScript 以获得最佳体验