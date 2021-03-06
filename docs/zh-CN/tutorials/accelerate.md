简体中文 | [English](../../en/tutorials/accelerate.md)

- [简介](#简介)
- [更优的解码库Decord](#更优的解码库Decord)
- [多进程加速Dataloader](#多进程加速Dataloader)
- [数据预处理DALI](#数据预处理DALI)
- [预先解码存成图像](#预先解码存成图像)
- [Multigrid加速策略算法](#Multigrid加速策略算法)
- [分布式训练](#分布式训练)


# 简介

视频任务相比于图像任务的训练往往更加耗时，其原因主要有两点:
- 数据：视频解码耗时。mp4/mkv等视频文件都是经过encode后的压缩文件，通过需要经过解码和抽帧步骤才能得到原始的图像数据流，之后经过图像变换/增强操作才能将其喂入网络进行训练。如果视频帧数多，解码过程是极其耗时的。
- 模型：视频任务使用的模型通常有更大的参数量与计算量。为学习时序特征，视频模型一般会使用3D卷积核/(2+1)D/双流网络，这都会使得模型的参数量与计算量大大增加。

为了加速视频模型的训练，可以分别从模型角度和数据角度考虑。模型上可以通过op融合或混合精度训练的方式提升op运算效率。针对[TSM模型]()，我们实现了[temporal shift op]()，在节省显存的同时加速训练过程。而对于单机训练，视频模型的训练瓶颈大多是在数据预处理上，因此本教程主要介绍在数据处理上的一些加速经验，同时这些能力都已经集成进PaddleVideo中，欢迎试用~

# 更优的解码库Decord

视频在喂入网络之前，需要经过一系列的数据预处理操作得到数据流，这些操作通常包括:

- 解码: 将视频文件解码成数据流
- 抽帧: 从视频中抽取部分帧用于网络训练
- 数据增强：缩放、裁剪、随机反转、正则化

其中解码是最为耗时的。相较于传统的opencv或pyAV解码库，这里推荐使用性能更优的解码库[decord]()，其官方给出的性能数据如下图:

目前[SlowFast]()模型使用decord进行视频解码([源码]())，对单进程的速度提升有较大作用。


我们分别以opencv/decord为解码器，实现SlowFast模型数据预处理pipeline，然后随机从kinetics-400数据集中选取200条视频，计算各pipeline处理每条视频的平均时间，测试环境为:
```
GPU: v100，4卡*16G
CPU: Intel(R) Xeon(R) Gold 6148 CPU @ 2.40GHz
PaddlePaddle: 2.0.0-rc1
```

性能测试数据如下:

| 解码库 | 版本 | pipeline处理每条视频的平均时间/s | 加速比 |
| :------ | :-----: | :------: | :------: | 
| opencv | 4.2.0 | 0.20965035 | baseline |
| decord | 0.4.2 | 0.13788146 |  1.52x |

可以看到，单进程下decord相较于opencv解码库可以加速1.52倍。

# 多进程加速Dataloader

数据准备好后喂入网络进行训练，网络运算使用GPU并行加速，其运算速度是很快的。因此对于单个进程来说，速度瓶颈大多在数据处理部分，GPU大部分时间是在等待CPU完成数据预处理。
飞桨2.0使用[Dataloader]()进行数据加载，DataLoader支持单进程和多进程的数据加载方式，当 num_workers 大于0时，将使用多进程方式异步加载数据。多进程加速协作，可以overlap掉GPU大部分等待的时间，提升GPU利用率，显著加速训练过程。

我们分别设置num_workers为0或4，单卡batch_size统一设置为8，统计训练一个batch的平均耗时，测试环境同上。

性能测试数据对比如下:
| 卡数 | 单卡num_workers | batch_cost/s | ips | 加速比 |
| :------ | :-----: | :------: |:------: |:------: |
| 单卡 | 0 | 1.763 | 4.53887 | 单卡baseline |
| 单卡 | 4 | 0.578 | 13.83729 | 3.04x |
| 4卡 | 0 | 1.866 | 4.28733 | 多卡baseline |
| 4卡 | 4 | 0.615 | 13.00625 | 3.03x | 

其中ips = batch_size/batch_cost，即为训练一个instance(一个video)的平均耗时。

**结合使用decord和飞桨dataloader，加上在数据增强部分做一些细节优化，SlowFast模型训练速度增益为100%，详细数据可以参考[benchmark]()**。

# 数据预处理DALI

既然GPU等待CPU进行数据处理耗时，能否把数据处理放到GPU上呢？[NVIDIA DALI]()将数据预处理pipeline转移到GPU上执行，可以显著提升训练速度。针对视频文件，DALI提供`VideoReader`op进行解码抽帧操作，但目前其仅支持连续采样的方式进行抽帧。而视频领域常用的2D模型TSN或TSM，它们均采用分段采样方式，即把视频均匀分成N段segument，然后在每个segument内随机选取一帧，最后把选取的帧组合作为输入张量。为此，我们基于DALI进行了二次开发，实现了支持分段采样方式的`VideoReader`op。为方便用户使用，我们提供了配置好的docker运行环境，具体使用方法参考[TSN-DALI使用教程]()。

# 预先解码存成图像

这是一种简单直接的方法，既然视频解码耗时，那可以事先将视频解码好，存成图片，模型训练时直接读取图像即可。这种方法可以显著提升视频模型训练速度，在XXX上大约提速3倍。但它也有一个很明显的缺点，就是需要耗费大量的内存空间，以kinetics-400数据集为例，共包含24万个训练样本，mp4文件约130G，解码存成图像后，占用的内存空间约为2T，所以这种方法比较适用于较小规模的数据集，如ucf-101。PaddleVideo提供了[预先解码]()的脚本，并且TSN和TSM模型均支持直接使用frame格式的数据进行训练，详细参考[TSN]() [TSM]()。

# Multigrid加速策略算法

前述方法大多从工程的角度提升训练速度，在算法策略上，FAIR在CVPR 2020中提出了一种[Multigrid训练加速方案]()，它的基本思想如下: 

在图像分类任务中，若经过预处理后图像的高度和宽度分别为H和W，batch_size为N，则网络输入batch的Tensor形状为`[N, C, H, W]`，其中C等于3，指RGB三个通道。
对应到视频任务，由于增加了时序通道，输入batch的Tensor形状为`[N, C, T, H, W]`。
传统的训练策略中，每个batch的输入Tensor形状都是固定的，即都是`[N, C, T, H, W]`。若以高分辨的图像作为输入，即设置较大的`[T, H, W]`，则模型精度会高一些，但训练会更慢；若以低分辨的图像作为输入，即设置较小的`[T, H, W]`，则可以使用更大的batch size，训练更快，但模型精度会降低。在一个epoch中，能否让不同batch的输入Tensor的形状动态变化，既能提升训练速度，又能保证模型精度？

基于以上思想，FAIR在实验的基础上提出了Multigrid训练策略: 固定`N*C*T*H*W`的值，降低`T*H*W`时增大`N`的值，增大`T*H*W`时减小`N`的值。具体的有两种策略，如示意图所示：

Long cycle:
设完整训练需要N个epoch，将整个训练过程分4个阶段，每个阶段对应的输入tensor形状为:
```
[8N, T/4, H/sqrt(2), W/sqrt(2)], [4N, T/2, H/sqrt(2), W/sqrt(2)], [2N, T/2, H, W], [N, T, H, W]
```

Short cycle:
在Long cycle的基础上，Short-cycle让每个iter的输入Tensor形状都会发生变化，变化策略为:
```
[H/2, W/2], [H/sqrt(2), W/sqrt(2)], [H, W]
```

在PaddleVideo中，SlowFast模型结合multigrid训练策略，使用8卡v100/32G机器训练全量Kinetics-400数据集358个epoch只需要4.X天。


# 分布式训练 

目前尚未支持此方案，如有需求，欢迎联系我们~
