---
title: "npu编译器和runtime总结"
url: https://zhuanlan.zhihu.com/p/1946589866398844420
author: 王多鱼-加油
date: 2026-09-01
---

## 前言

我最近看了一套[npu编译器](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=npu%E7%BC%96%E8%AF%91%E5%99%A8&zhida_source=entity)源码，在介绍npu的编译器和runtime的总结开始前，我先抛出[tvm](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=tvm&zhida_source=entity)是怎么做的。我目前看来npu的编译器和runtime和tvm做的是相似的工作，区别在于tvm的源码谁都能看到，能否看懂完全赖你自己；npu的编译器和[runtime](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=3&q=runtime&zhida_source=entity)因为封闭、还很贵、还容易暴露国产npu厂家可能就是买来了一份方案，自己啥都没干，怕露馅等原因，你看不到细节，靠信息差吹牛逼。我目前感觉npu这玩意国内能做的也有限，年初失业那会，某npu公司技术第四面把我挂了的原因就应该是信息差。上面都是我个人的推测，如果有问题辛苦大家指正。

## 细节归纳

我目前理解，npu有两条主线。模型的[编译优化](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=%E7%BC%96%E8%AF%91%E4%BC%98%E5%8C%96&zhida_source=entity)是一条路线；模型的runtime是一条线。

模型的[Tiling](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=Tiling&zhida_source=entity)、 算子融合、 算子优化模式和[pass](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=pass&zhida_source=entity) 、并行流水线是为了产生优化后的模型文件。编译后的模型靠应用层的runtime lib把模型加载，内存申请，然后把模型信息给到驱动的runtime，然后给到[gpu](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=gpu&zhida_source=entity) firmware固件，然后去执行npu指令。

下面我先列出tvm，然后列出npu。

### [TVM](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=TVM&zhida_source=entity)

1）模型编译优化

TVM的IR可以适应pytorch onnx tensorflow等各种推理平台的模型及模型的IR，等于用一层变换IR或语法树对齐了不同推理平台来的模型计算图的表示方式。然后借助[ansor](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=ansor&zhida_source=entity)或meta-schedule完成了计算图的优化和融合。借助ffi能力可以让[cpp](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=cpp&zhida_source=entity)和python的算子互相调用，又可以使用mlir tensorrt cutlass等的算子实现。虽然tvm是python优先的，前端只需要一个build就能得到优化融合后的算子，但是后端的cpp却有不同硬件对应的codegen，是这些生成的目标平台的汇编指令或cpp文件来真正实现算子在目标target运行的，而加载这些汇编指令和cpp文件导出的lib的对象就是runtime模块。可以说runtime负责把上面生成的代码加载起来。这里其实很有意思，不同平台你得用不同平台的方式来加载，例如你生成的是[cuda](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=cuda&zhida_source=entity)代码，你要用cuda的那个动态加载lib的方式来执行；你生成的是[ptx指令](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=ptx%E6%8C%87%E4%BB%A4&zhida_source=entity)，你得编译链接得到cubin文件，然后用cuda的这个方式来加载运行。总之你[codegen](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=2&q=codegen&zhida_source=entity)对应的是一个或一堆函数，这些函数要能正确的访问目标硬件的驱动，进而能被gpu或npu执行指令。

2）模型的runtime加载

runtime要能正确加载tvm导出生成的lib，需要能正确识别这个lib，然后找到入口函数，入口函数进行device memory等资源的申请后，然后执行对应的cuda 函数或者其他设备的函数。因为你用的目标平台的运行api，所以你能在目标平台申请内存，指定设备，以及在目标平台上运行代码。triton能在nvidia的目标sm硬件上运行代码也是因为triton的mlir最后调用了[llvm](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=llvm&zhida_source=entity)的ptx对应的ir，生成了目标指令。即使tvm是自动codegen生成的代码，codegen其实本质和你调用目标平台的api是一致的。

总结就是，本质计算图这种网络结构是给人看的，最后执行的都是函数或指令，你codegen生成的是优化后的目标平台的指令或函数。因此你能被目标平台的驱动加载，进而执行。

### NPU

我目前看到这套npu的接口，感觉和地平线j6M的非常像，编译器实现因为没有地平线的源码就不得而知了。

1）模型的优化

我之前用地平线J6M的板子的时候，一个模型的ptq的流程大概如下。

![图片](https://pica.zhimg.com/v2-5c2645da4e1e178839fc303195bc43de.jpg)

每一步骤都是一个python脚本，第3步是模型转换加[ptq量化](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=ptq%E9%87%8F%E5%8C%96&zhida_source=entity)。我发现我看的这个npu的模型build也和J6M几乎一致。等于这些脚本都最后调用了Neural Network Accelerator Controller模块，这个模块分为前后端，前端是一些python代码。

这些python代码也会用onnxruntime pytorch的一些能力，比如make\_graph、make\_model接口，NetworkX ，借助这些能力，先对[onnx模型](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=onnx%E6%A8%A1%E5%9E%8B&zhida_source=entity)就行一些变换；然后对转换后的模型的算子进行**legalize操作；接着dipatch 计算图；然后对onnx模型量化（插入QDQ量化节点）；最后编译前面转换后的onnx模型为npu的模型格式，实现是调用backend的compile能力。我发现这个backend没有使用**pybind调用cpp函数，因为他的backend的优化是最核心的，没有源码。他是直接执行了这个prebuild好的backend bin。所以你改不了模型的tiling codegen 目标硬件的算子优化融合等。虽然他把他的底层算子lib提供给你了，但是在算子优化这里卡住了你。

**等于前面的基本模型优化和模型ptq量化是借助onnx模型来做的，后端的和芯片相关的算子优化及codegen是闭源的。美国人在七寸上卡住了你。即使你有驱动，有部分模型优化能力，但是在最关键的七寸，你没有办法。基于这个，你都连谈自定义算子融合的能力都没有。**

**\-------------------------**

**我又看了tvm的[计算图](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=5&q=%E8%AE%A1%E7%AE%97%E5%9B%BE&zhida_source=entity)优化和算子融合。我感觉这套代码，计算图融合这部分应该是有了，缺的是算子优化和codegen这部分。**

2）模型的runtime加载

国内的npu一般都只提供计算图的接口，我感觉是因为他们购买的方案就是这样的。为了支持这个接口需要提供lib和头文件。主要包括初始化、 反初始化、 加载模型 、获取模型结构 、填充输入接口数据、 推理运行、 获取输出数据、 已经内存相关的接口。其中内存的操作很有可能是通过/dev目录下的节点通过[ioctl](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=ioctl&zhida_source=entity)能力进行操作的；还会通过/sys目录获取一些运行状态和运行效率的信息。

模型的推理部分在应用层的runtime比tvm复杂很多。device manager、Stream manager graph manager 、submit、 schedule、 rescoure manager都是在这里实现的。等于驱动实现的单一了，主要功能在应用层实现，分层设计思想包括hal层。这些操作包括了很多个job，这些job通过ioctl接口吐给驱动的runtime程序，驱动的runtime还是维护内存池，以及分页的一些操作。最后job被submit给了 firmware固件，job对应的指令最后发送给了NPU真正的计算单元。另外我发现他的runtime还是分层设计的，把一些内存地址的映射的实现是基于驱动，但是管理放到了应用层。

另外需要提的一点，npu这玩意不是所有的算子都在npu上跑，有些算子他可以在cpu和dsp更合适。所以他在应用层的context概念比gpu上的大，[nvida](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=nvida&zhida_source=entity)平台你模型除了gpu就最多是dla（一般会直接把模型业务切割，不存在一个context的模型部分算子跑在gpu，部分算子跑在dla，关联关系是业务负责解决）。这个npu的加速算子是靠dsp来实现的。

  

## 总结：

1）tvm 和npu的 [Synopsys](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=1&q=Synopsys&zhida_source=entity) 实现的思路和主要功能是相同的，区别就是tvm是面向多个硬件， Synopsys 是面向自己的这套解决方案。  
2）tvm是完全开源的，虽然社区力量不够，但是完全开源；Synopsys 把算子优化、codegen部分闭源了，npu最重要的工作就是支持算子开发和算子优化，这么看，没有自定义算子的能力，新算子的优化估计还得单独收费。从这里看tvm和[mlir](https://zhida.zhihu.com/search?content_id=262601336&content_type=Article&match_order=3&q=mlir&zhida_source=entity)又有价值了。  
3）驱动部分的价值还没有应用层的runtime的context有价值。