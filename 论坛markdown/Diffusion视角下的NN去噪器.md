 Diffusion视角下的NN去噪器 - AIISP - AXERA          {"@context":"http://schema.org","@type":"WebSite","url":"http://soc.aixin-chip.com","potentialAction":{"@type":"SearchAction","target":"http://soc.aixin-chip.com/search?q={search\_term\_string}","query-input":"required name=search\_term\_string"}}                                                            {"@context":"http://schema.org","@type":"QAPage","name":"Diffusion视角下的NN去噪器","mainEntity":{"@type":"Question","name":"Diffusion视角下的NN去噪器","text":"数据映射拟合\\n\\n理论基础\\n\\n万能逼近定理：只需二个包含足够多神经元的隐层，多层前馈网络就能以任意精度逼近任意复杂度的连续函数。\\n\\n<a class=\\"lightbox\\" href=\\"//soc.aixin-chip.com/uploads/default/original/3X/e/c/ec81a726562b90f92e2611164c5147273b3353c3.jpeg\\" data-download-href=\\"//soc.aixin-chip.com/uploads/default/ec81a726562b90f92e2611164c5147273b3353c3\\" title=\\"image\\">\[image\]<\\/a>\\n\\n正面解读：只要搭的好喂得好训得好，神经网络就能拟合超困难的数据映射\\n\\n反向思考：模型、数据、训&hellip;","upvoteCount":2,"answerCount":0,"dateCreated":"2024-12-08T14:57:16.950Z","author":{"@type":"Person","name":"fenghansen"}}} 

[

## AXERA

](/)

# [Diffusion视角下的NN去噪器](/t/topic/10272)

[AIISP](http://soc.aixin-chip.com/c/126-category/126) 

[aiisp](http://soc.aixin-chip.com/tag/aiisp)

[fenghansen](http://soc.aixin-chip.com/u/fenghansen)    2024年12月13日 06:40 #1

# 数据映射拟合

## 理论基础

万能逼近定理：只需二个包含**足够多**神经元的隐层，多层前馈网络就能以**任意精度**逼近**任意复杂度**的连续函数。  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/e/c/ec81a726562b90f92e2611164c5147273b3353c3_2_690x265.jpeg)

image911×350 105 KB

](//soc.aixin-chip.com/uploads/default/original/3X/e/c/ec81a726562b90f92e2611164c5147273b3353c3.jpeg "image")

  
**正面解读**：只要**搭**的好**喂**得好**训**得好，神经网络就能拟合超困难的**数据映射**  
**反向思考**：**模型、数据、训练**都是难点，让**数据映射**表征**分布映射**是技术活儿

## 生成相关的核心问题：随机性从哪里来？

没有随机性，**数据映射**就无法等效**分布映射**，**泛化能力**听天由命  
如何构造**模型、数据、训练**框架来表征分布，是生成式算法的核心差异

### VAE：用均值锚定核心特征，用随机噪声建模分布差异

[变分自编码器（六）：从几何视角来理解VAE的尝试 - 科学空间|Scientific Spaces](https://kexue.fm/archives/7725)

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/9/b/9b8bbe2901c27503776b39f446bdb966e6c959c8_2_690x175.png)

image1860×474 137 KB

](//soc.aixin-chip.com/uploads/default/original/3X/9/b/9b8bbe2901c27503776b39f446bdb966e6c959c8.png "image")

### StyleGAN：模型内**分尺度**地构造**核心特征**和**随机噪声**到真实图片的映射

[![image](//soc.aixin-chip.com/uploads/default/original/3X/a/4/a4f93230c0f4d1611083114a3291ad95e842640f.jpeg)

image819×700 110 KB

](//soc.aixin-chip.com/uploads/default/original/3X/a/4/a4f93230c0f4d1611083114a3291ad95e842640f.jpeg "image")

### Diffusion：在数据和训练维度上拆分

[生成式 AI 研究聚焦：揭开基于扩散的模型的神秘面纱 - NVIDIA 技术博客](https://developer.nvidia.cn/zh-cn/blog/generative-ai-research-spotlight-demystifying-diffusion-based-models/)

+   转为去噪任务，为**随机**提供**确定性度量**
+   将随机性**分尺度地**建模到不同频率(timestep)
+   迭代去噪，**降低数据映射拟合难度**

[![735e694c7d6da273b6f28908de86d7392664779a](//soc.aixin-chip.com/uploads/default/optimized/3X/7/3/735e694c7d6da273b6f28908de86d7392664779a_2_625x500.gif)

735e694c7d6da273b6f28908de86d7392664779a1200×960 2.33 MB

](//soc.aixin-chip.com/uploads/default/original/3X/7/3/735e694c7d6da273b6f28908de86d7392664779a.gif "735e694c7d6da273b6f28908de86d7392664779a")

[![7f3a8310a43c8db94e57750576f9d92f253e3310](//soc.aixin-chip.com/uploads/default/optimized/3X/7/f/7f3a8310a43c8db94e57750576f9d92f253e3310_2_625x500.gif)

7f3a8310a43c8db94e57750576f9d92f253e33101200×960 1.1 MB

](//soc.aixin-chip.com/uploads/default/original/3X/7/f/7f3a8310a43c8db94e57750576f9d92f253e3310.gif "7f3a8310a43c8db94e57750576f9d92f253e3310")

# DDPM去噪

## DDPM回顾

1.  损失函数只依赖于 p(x\_t | x\_0)
2.  推理/采样（Sampling）过程只依赖于 p(x\_{t-1} | x\_t)

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/2/d/2df8c81e06a1477e869e3e4f41c197f8b7349f51_2_690x168.png)

image1772×432 260 KB

](//soc.aixin-chip.com/uploads/default/original/3X/2/d/2df8c81e06a1477e869e3e4f41c197f8b7349f51.png "image")

也就是说，尽管整个过程是以 p(x\_{t-1} | x\_t) 为出发点一步步往前推的，但是从结果上来看，压根儿就没 p(x\_t | x\_0) 的事。于是DDIM就把这项去掉了。

## DDIM

### 流程创新（SDE->ODE）

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/5/0/50833295fa2e116e65fcb93340962f668c6124e1_2_690x186.jpeg)

image1709×461 164 KB

](//soc.aixin-chip.com/uploads/default/original/3X/5/0/50833295fa2e116e65fcb93340962f668c6124e1.jpeg "image")

  
公式里的direction pointing to 𝑥\_𝑡相当于回帖部分原图  
\\sigma\_t=0 一般被认为是纯DDIM，$\\sigma\_t=\\sqrt{1-\\alpha\_t}$ 公式退化为DDPM

### 去噪视角

时间戳 t 可以直接计算出噪声强度 \\sqrt{\\frac{1-\\alpha\_t}{\\alpha\_t} } ，相当于给去噪力度  
先由 x\_t 和 t 估计一个粗糙的 \\hat{x}\_0 \\approx x\_0 ，再加噪到 x\_{t-1} ，迭代循环

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/1/4/14538cf46382a5bf62fecfe84d72861203a5c23d_2_690x181.jpeg)

image1466×385 166 KB

](//soc.aixin-chip.com/uploads/default/original/3X/1/4/14538cf46382a5bf62fecfe84d72861203a5c23d.jpeg "image")

反正每次都是在预测 $x\_0$，所以没必要迭代1000步，可以跳步

> [生成扩散模型漫谈（四）：DDIM = 高观点DDPM - 科学空间|Scientific Spaces](https://kexue.fm/archives/9181)  
> 
> [![image](//soc.aixin-chip.com/uploads/default/optimized/3X/3/7/37149857f4d1c5010b610aa5cbbfc1c37cc96b21_2_690x239.jpeg)
> 
> image1454×504 232 KB
> 
> ](//soc.aixin-chip.com/uploads/default/original/3X/3/7/37149857f4d1c5010b610aa5cbbfc1c37cc96b21.jpeg "image")

## 基于扩散模型的生成式图像去噪（李桐的DMID）

### 去噪视角的重解读

+   **时间戳t作为输入送入？**  
    这就是力度控制啊！
+   **为啥学噪声会更好？**  
    学习噪声非极暗去噪更好，前面极暗随便乱搞
+   **采样策略的改进？**  
    稳定方差/稳定信号再考虑采样策略。α=1/2的时候就是极暗了，均匀采样太浪费

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/7/9/796fbec7c99210b799c308e275c0984c32b69d95_2_690x466.jpeg)

image1123×760 218 KB

](//soc.aixin-chip.com/uploads/default/original/3X/7/9/796fbec7c99210b799c308e275c0984c32b69d95.jpeg "image")

### 具体方法

+   通过噪声先验激活扩散模型的降噪功能，可用于真实图像降噪
+   利用自适应嵌入进行方差稳定变换，将带噪图嵌入推理过程
+   利用自适应集成约束扩散模型的保真程度，避免过度生成

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/5/0/503ec50f9ca82c0a2c9ad0476ac89d4ce3dcceb2_2_690x175.jpeg)

image1920×488 211 KB

](//soc.aixin-chip.com/uploads/default/original/3X/5/0/503ec50f9ca82c0a2c9ad0476ac89d4ce3dcceb2.jpeg "image")

### 实验结果

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/b/2/b207342563132a0b91d37563d27f7c9e7da7dbdd_2_690x306.jpeg)

image1850×822 557 KB

](//soc.aixin-chip.com/uploads/default/original/3X/b/2/b207342563132a0b91d37563d27f7c9e7da7dbdd.jpeg "image")

# 去噪视角下的DDPM

## Score-based的视角

[![image](//soc.aixin-chip.com/uploads/default/original/3X/6/8/68d4b71c9b785f12d8dd9105f25c223b1b75e613.jpeg)

image867×295 124 KB

](//soc.aixin-chip.com/uploads/default/original/3X/6/8/68d4b71c9b785f12d8dd9105f25c223b1b75e613.jpeg "image")

计算score function，作为分布变换的梯度场，由nn力度去噪来实现  
时间戳t表示与数据分布的距离，作为参考条件可以极大地补充信息

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/5/1/5179f1247022b72f6b4360a147a9da70060bb753_2_690x204.jpeg)

image1976×585 360 KB

](//soc.aixin-chip.com/uploads/default/original/3X/5/1/5179f1247022b72f6b4360a147a9da70060bb753.jpeg "image")

## 核心思想

+   **简化理解**，无需过度关注数学推导，当做去噪任务理解就好
    +   难点1：接受“NN去噪”和“传统去噪”不一样，NN本身就具有生成能力
    +   难点2：辩证看待，DDPM是力度去噪的延拓，噪声强度越小结论越相似
+   **互相借鉴**，利用去噪中已有的结论改造DDPM，或者用DDPM改造去噪
    +   难点1：如何合理建模以链接两种范式——信噪比、频率
    +   难点2：基于去噪已有研究（尤其是视频去噪），延拓出新的生成式图像恢复方法
+   **解放思想**，关注数据映射拟合（nn去噪器）
    +   DDPM的复杂推导使人们过于关注结构设计，忽略了凭什么能学到score function
    +   更重要的是，之前的研究忽略了“注定学不好”时怎么取舍

## 典型案例

### Consistency Models（ICML 2021，Yang Song）

增强一致性约束，一步生成，一步去噪  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/1/3/13ea609d996fa55068997d39953481ac3c3ba4ff_2_690x294.jpeg)

image1789×764 816 KB

](//soc.aixin-chip.com/uploads/default/original/3X/1/3/13ea609d996fa55068997d39953481ac3c3ba4ff.jpeg "image")

翻了一下代码，除了系数变换，感觉核心做法就是让优化 x\_t \\rightarrow x\_0 的时候，让相邻帧t, t+1的去噪结果尽量一致，有点视频去噪中时域一致损失的那个意思，挺直观的。  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/4/4/44c3eb3bed781b78f61de3b3f4e0ec9f184154d3_2_661x500.png)

image879×664 162 KB

](//soc.aixin-chip.com/uploads/default/original/3X/4/4/44c3eb3bed781b78f61de3b3f4e0ec9f184154d3.png "image")

### ResShift（NIPS 2023，Chen Change Loy ）

#### 概述

+   只用15步就可以做到高质量图像恢复，果然有压缩空间
+   提供了一种创造性破坏的方式，可以避免对SNR节点判别的吹毛求疵

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/0/9/098063f5672df49a9de7359abb4eba5cdacb8a1d_2_690x217.jpeg)

image1884×595 234 KB

](//soc.aixin-chip.com/uploads/default/original/3X/0/9/098063f5672df49a9de7359abb4eba5cdacb8a1d.jpeg "image")

#### 方法

VE-SDE，均值项干净图 x\_0 噪图 $y\_0$，时间戳 \\eta\_t 从0到1，方差项k有上限

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/7/9/79ac3742d789f5223ba3a317778119d333555894_2_690x286.jpeg)

image1717×713 329 KB

](//soc.aixin-chip.com/uploads/default/original/3X/7/9/79ac3742d789f5223ba3a317778119d333555894.jpeg "image")

ResShift正向加噪流程可视化（k-噪声强度，p-曲率，T-步数）  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/b/c/bc2bcb61441ebb46b44841c02f16294ba82fe3f5_2_690x336.jpeg)

image1532×747 361 KB

](//soc.aixin-chip.com/uploads/default/original/3X/b/c/bc2bcb61441ebb46b44841c02f16294ba82fe3f5.jpeg "image")

#### 消融实验

+   步数越多越清晰
+   加噪策略不同以往
+   噪声跨度要适当，SNR不匹配的话会亏

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/4/e/4e33c01572e3230e7ae5d1781b960491d86e29d9_2_690x310.png)

image824×371 114 KB

](//soc.aixin-chip.com/uploads/default/original/3X/4/e/4e33c01572e3230e7ae5d1781b960491d86e29d9.png "image")

  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/7/5/75bc9ad975883305c711f97db01a8024b0b9e335_2_690x402.jpeg)

image1409×821 326 KB

](//soc.aixin-chip.com/uploads/default/original/3X/7/5/75bc9ad975883305c711f97db01a8024b0b9e335.jpeg "image")

### SinSR（CVPR 2024，Yufei Wang & Bihan Wen）

#### 概述

+   蒸馏ResShift，一步生成式超分，说明在low-level上还有压缩空间

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/f/d/fd57389e2f526684697b75e3bc3f7f9eb404623c_2_690x216.jpeg)

image1886×592 365 KB

](//soc.aixin-chip.com/uploads/default/original/3X/f/d/fd57389e2f526684697b75e3bc3f7f9eb404623c.jpeg "image")

#### 方法

DDIM化，保证映射稳定，再蒸馏学习DDIM的结果  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/8/e/8e14c521494c3e79b031c4966b7d0cc536d7d7b7_2_495x500.jpeg)

image1114×1125 381 KB

](//soc.aixin-chip.com/uploads/default/original/3X/8/e/8e14c521494c3e79b031c4966b7d0cc536d7d7b7.jpeg "image")

提供了一种蒸馏的具体方法（inverse loss 不太合理，删掉更好）  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/6/e/6e2b3da638ace6a9800ba7ebae8137a3e4635e5c_2_690x221.jpeg)

image1920×617 247 KB

](//soc.aixin-chip.com/uploads/default/original/3X/6/e/6e2b3da638ace6a9800ba7ebae8137a3e4635e5c.jpeg "image")

# DDPM视角下的NN去噪

## Motivation

从极暗去噪的视角回顾，从海量数据中归纳先验信息才是NN的核心能力

+   传统去噪方法大多在利用图像内的信息，但NN可以用本图外的数据先验
+   对于生成而言，最关键的是如何构造分布度量以有效训练

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/e/8/e8729d4e91a3e3e54c5cd2bafffae0ee753b88f8_2_690x221.jpeg)

image1885×605 604 KB

](//soc.aixin-chip.com/uploads/default/original/3X/e/8/e8729d4e91a3e3e54c5cd2bafffae0ee753b88f8.jpeg "image")

## YOND-p

力度去噪器=DDPM去噪器？  
天然是近似的！但是由于网络规模和训练策略的差异，artifacts严重  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/8/c/8c31cab262a66a15c82aa594c6f1dfa902f37bb5_2_690x224.jpeg)

image1926×627 743 KB

](//soc.aixin-chip.com/uploads/default/original/3X/8/c/8c31cab262a66a15c82aa594c6f1dfa902f37bb5.jpeg "image")

  

[![image](//soc.aixin-chip.com/uploads/default/original/3X/d/4/d4a5619f0f6b34d8ea0f50952a57c03632e01982.jpeg)

image1020×211 51.9 KB

](//soc.aixin-chip.com/uploads/default/original/3X/d/4/d4a5619f0f6b34d8ea0f50952a57c03632e01982.jpeg "image")

### YOND概述

+   **数据是实用Raw去噪的灵魂**
+   **现有的方法都有数据依赖性**
    +   配对真实数据：依赖大量相机数据
    +   噪声建模：依赖少量相机标定数据
    +   自监督：难以泛化到相机外数据
+   **我们的思路**
    +   摆脱数据采集、标定与训练过程
    +   一次训练，任意泛化（Raw）  
        可解释的噪声估计+魔改VST
    +   灵活可调，细节能加（YOND-p）

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/7/7/77a6f82aec946fc17a512669d84e106e62d8cb28_2_560x499.jpeg)

image1051×937 335 KB

](//soc.aixin-chip.com/uploads/default/original/3X/7/7/77a6f82aec946fc17a512669d84e106e62d8cb28.jpeg "image")

### 方法

#### 代码

```python
def ddim(x_T, model, nsr, name='ddim', epoch=10, eta=0.85, 
         sigma_t=0.8, sigma_corr=1.03, wb=None, ccm=None):
    x_t = x_T.clone() # 初始化，x_t从x_T开始
    t = nsr * sigma_corr # nsr是噪信比，σ=25时nsr=25/255，sigma_corr为额外矫正系数
    for i in tqdm(range(epoch)):
        x_0 = model(x_t, t).clamp(0,1) # 去噪
        # noise = torch.randn_like(x_0) * t # ddpm, 等效eta=1
        noise = (x_T - x_0) * (sigma_t**i) # ddimT, 指向最初的噪图，减少误差累计
        # noise = (raw_vst - raw_dn) # ddim, 等效eta=0
        noise = noise * eta + torch.randn_like(x_0) * t * (1-eta**2)**0.5 # ddpim
        x_t = x_0 + noise * sigma_t # x_0 to x_t
        # t = nsr * (sigma_t**(i+1)) * sigma_corr # ideal t
        t = noise.std() * sigma_t * sigma_corr # real-time t
    return x_0
```

#### 流程

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/8/d/8da816d665edc881be8bb3c0b5613bc4b0b02165_2_690x145.png)

image1461×308 34.4 KB

](//soc.aixin-chip.com/uploads/default/original/3X/8/d/8da816d665edc881be8bb3c0b5613bc4b0b02165.png "image")

  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/3/9/39f56c3ac1f0f41d657e878af9fd9c93efda12d7_2_690x258.jpeg)

image1924×720 674 KB

](//soc.aixin-chip.com/uploads/default/original/3X/3/9/39f56c3ac1f0f41d657e878af9fd9c93efda12d7.jpeg "image")

#### 效果

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/2/1/216460aa1eac43c3a76c9f44e8e5906d294b875c_2_690x414.jpeg)

image1756×1056 633 KB

](//soc.aixin-chip.com/uploads/default/original/3X/2/1/216460aa1eac43c3a76c9f44e8e5906d294b875c.jpeg "image")

### 消融实验

#### ddimT消融

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/7/b/7b1e74e577d56b3b3afa61e732acce0c9e20e2fe_2_690x145.png)

image1461×308 32.1 KB

](//soc.aixin-chip.com/uploads/default/original/3X/7/b/7b1e74e577d56b3b3afa61e732acce0c9e20e2fe.png "image")

  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/a/c/ac0838c5eebf06cecd2b0bcd88642672dc799930_2_690x258.jpeg)

image1924×720 678 KB

](//soc.aixin-chip.com/uploads/default/original/3X/a/c/ac0838c5eebf06cecd2b0bcd88642672dc799930.jpeg "image")

  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/2/8/28e5f0bd48c9c58ccec7dd422c9234194672e602_2_690x414.jpeg)

image1756×1056 637 KB

](//soc.aixin-chip.com/uploads/default/original/3X/2/8/28e5f0bd48c9c58ccec7dd422c9234194672e602.jpeg "image")

#### ddim消融

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/3/a/3a0abce51c24f5c988c1031683c2b2f2ce83eb39_2_690x145.png)

image1461×308 33.4 KB

](//soc.aixin-chip.com/uploads/default/original/3X/3/a/3a0abce51c24f5c988c1031683c2b2f2ce83eb39.png "image")

  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/a/7/a7f4931c4ced52c1a68a9b82185d78d85602cc86_2_690x258.jpeg)

image1924×720 675 KB

](//soc.aixin-chip.com/uploads/default/original/3X/a/7/a7f4931c4ced52c1a68a9b82185d78d85602cc86.jpeg "image")

  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/a/3/a34681566d3f26fac84b2b4b255e5545cb3af9a0_2_690x414.jpeg)

image1756×1056 656 KB

](//soc.aixin-chip.com/uploads/default/original/3X/a/3/a34681566d3f26fac84b2b4b255e5545cb3af9a0.jpeg "image")

#### ddpm消融

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/7/c/7c711c0c28aa3168204911ec85d85e29576b2892_2_690x145.png)

image1461×308 32.2 KB

](//soc.aixin-chip.com/uploads/default/original/3X/7/c/7c711c0c28aa3168204911ec85d85e29576b2892.png "image")

  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/5/1/513bc4cfa7e60edf54ae06df1152a6f33cd92bd5_2_690x258.jpeg)

image1924×720 674 KB

](//soc.aixin-chip.com/uploads/default/original/3X/5/1/513bc4cfa7e60edf54ae06df1152a6f33cd92bd5.jpeg "image")

  

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/5/8/5858b5d501ea02fdeafec200dc243337a61d9f3b_2_690x418.jpeg)

image1743×1056 618 KB

](//soc.aixin-chip.com/uploads/default/original/3X/5/8/5858b5d501ea02fdeafec200dc243337a61d9f3b.jpeg "image")

#### 模型消融

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/d/4/d458f0fe860ff6aaf6203470d6fedc8914f1fe74_2_690x336.jpeg)

image1054×514 375 KB

](//soc.aixin-chip.com/uploads/default/original/3X/d/4/d458f0fe860ff6aaf6203470d6fedc8914f1fe74.jpeg "image")

1.  **模型尺寸减小nf32**  
    直接去噪结果相近，迭代去噪差异巨大，生成能力明显降低，artifacts显著增多  
    
    [![image](//soc.aixin-chip.com/uploads/default/optimized/3X/b/9/b938e7f4595a38d180c3533cfbd3ec658f79894c_2_690x214.jpeg)
    
    image1623×505 193 KB
    
    ](//soc.aixin-chip.com/uploads/default/original/3X/b/9/b938e7f4595a38d180c3533cfbd3ec658f79894c.jpeg "image")
    
2.  **DIV2K only**  
    暗区效果明显变差，artifacts略有好转  
    
    [![image](//soc.aixin-chip.com/uploads/default/optimized/3X/2/f/2f85e4a8c146bc7b374c1886f7816fd851a8088f_2_690x214.jpeg)
    
    image1623×505 187 KB
    
    ](//soc.aixin-chip.com/uploads/default/original/3X/2/f/2f85e4a8c146bc7b374c1886f7816fd851a8088f.jpeg "image")
    
3.  **Clip模型**  
    高噪信息丢失严重，不适应迭代去噪流程，偏色会累计增强  
    
    [![image](//soc.aixin-chip.com/uploads/default/optimized/3X/6/b/6bfa6e41e29b3dc3b69924d3c380deebc6eee6ad_2_690x214.jpeg)
    
    image1623×505 251 KB
    
    ](//soc.aixin-chip.com/uploads/default/original/3X/6/b/6bfa6e41e29b3dc3b69924d3c380deebc6eee6ad.jpeg "image")
    
4.  **增加感知损失**  
    去噪风格明显变化，直接去噪artifacts，严重但细节更多，迭代误差累计不保真  
    
    [![image](//soc.aixin-chip.com/uploads/default/optimized/3X/f/2/f2729c375b78974ceb12a35785da1186ab07b4e3_2_690x214.jpeg)
    
    image1623×505 215 KB
    
    ](//soc.aixin-chip.com/uploads/default/original/3X/f/2/f2729c375b78974ceb12a35785da1186ab07b4e3.jpeg "image")
    
5.  **适当改变迭代策略**  
    原则上终点𝜎=5/255 ，迭代越多细节越多，artifacts也越多，“生成”味儿很强  
    
    [![image](//soc.aixin-chip.com/uploads/default/optimized/3X/0/a/0a319829c1355bc3055ef49ae20e2faa712c8f97_2_690x215.jpeg)
    
    image1623×506 208 KB
    
    ](//soc.aixin-chip.com/uploads/default/original/3X/0/a/0a319829c1355bc3055ef49ae20e2faa712c8f97.jpeg "image")
    

#### 总结

+   **蓝色-基础实验**  
    普通YOND细节有限迭代起来细节生成
+   **黄色-迭代消融**  
    步数少类sharpen步数多artifacts
+   **绿色-尺寸消融**  
    小模型回归效果类似迭代生成吃模型体量

[![image](//soc.aixin-chip.com/uploads/default/optimized/3X/2/9/29b9b93174f496df9f13e29b011daa6d0fc53a53_2_635x500.jpeg)

image1261×992 480 KB

](//soc.aixin-chip.com/uploads/default/original/3X/2/9/29b9b93174f496df9f13e29b011daa6d0fc53a53.jpeg "image")

 

  2赞

 

  [\[阅读\] Is Noise Conditioning Necessary for Denoising Generative Models? (Arxiv 2025)](http://soc.aixin-chip.com/t/topic/10460)

  [\[worklog\] 条件DDPM改造训练数据](http://soc.aixin-chip.com/t/topic/10994)

[首页](/) [分类](/categories) [FAQ/指引](/guidelines) [服务条款](/tos) [隐私政策](/privacy)

采用 [Discourse](https://www.discourse.org)，启用 JavaScript 以获得最佳体验