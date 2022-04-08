---
layout: post
title: SVC介绍与MSDK实现时域SVC
date: 2022-04-07 16:29:53
categories: [H264,HEVC]
tags: [MediaSDK,Encoder]
---

SVC(Scaled Video Coding)最早作为H.264标准的一个扩展最初由JVT在2004年开始制定，并于2007年7月获得ITU批准，HEVC现均已支持SVC。H.264 SVC以H.264 AVC视频编解码器标准为基础，利用了AVC编解码器的各种高效算法工具，在编码产生的编码视频时间上（帧率）、空间上（分辨率）可扩展，并且是在视频质量方面可扩展的，可产生不同帧速率、分辨率或质量等级的解码视频。

<!-- more -->

## 概述
SVC，可分层视频编解码，分为时域可适性（*Temporal Scalability*），空域可适性（*spatial Scalability*）和质量可适性（*quality Scalability*），是一种能将视频流分割为多个分辨率、质量和帧率层的技术，一般存在一个基本层和一个或以上的增强层，为大多数视频会议设备所采用H264/HEVC视频编解码标准的扩展。空间可扩展性与质量可扩展性会显著增加解码器的复杂度以及显著降低编码效率，因此并未被广泛支持，然而时域可适性却可以带来显著的编码效率提升,因此而得到广泛使用。

#### 时域可适性
对于混合编码模式，可使用运动补偿编码方式，使高层Layer参考低层Layer或者同层layer即可产生时域可适性。可根据网络与解码设备状况从高层layer依次丢帧至基本层，高层layer解码依赖于底层layer。

4层时域结构，包含I帧与B帧，为1：1：2：4结构
![Temporal SVC](/images/temporal_b_frames.jpg)

3层时域结构，包含I帧与B帧，为1：2：4结构
![NonDyadic Temporal SVC](/images/temporal_nodyadic_b_frames.jpg)

4层IPPP低延时结构，包含I帧与P帧，为1：1；2：4结构
![IPPP temporal SVC](/images/temporal_ippp_frames.jpg)

#### 空域可适性
空域可扩展性的实现基于多层layer编码方式，每层对应各自的分辨率，一般低层layer分辨率较低，每帧参考于同层帧或者其依赖标识层帧（低层layer）。每层均会采用帧内预测与帧间预测，帧内预测时，高层layer参考低层layer，并使用上采样提升分辨率，同层参考时则使用帧间预测。解码时并非是高层layer解码依赖底层layer，只会使用特定layer层，因此，可结合时域与空域可适性。
![spatial SVC](/images/spatial_scalability.jpg)

#### 质量可适性
质量可适性是一种特殊的空域可适性，每层layer的分辨率相同且不会采用上采样。
![quality SVC](/images/quality_scalability.jpg)

## MediaSDK Temporal SVC
MediaSDK的时域SVC设置，在mfxVideoParam参数集中加入mfxExtAvcTemporalLayers，并设置BaseLayerPid与每层的帧数分布，MediaSDK最多支持8层结构。`typedef mfxExtAvcTemporalLayers mfxExtHEVCTemporalLayers`表明下述时域SVC设置同样适用与HEVC。

```c++
using MfxVideoParamsWrapper = ExtBufHolder<mfxVideoParam>;
MfxVideoParamsWrapper m_mfxEncParams;

auto avcTemporalLayers = m_mfxEncParams.AddExtBuffer<mfxExtAvcTemporalLayers>();

avcTemporalLayers->BaseLayerPID = 0;  //base layer 优先级
std::vector<mfxU16> layers{1, 2, 4, 0, 0, 0, 0, 0}; // 分层结构，1-3层帧数比为1：1：2
for (int i = 0; i < 8; i++) {
    avcTemporalLayers->Layer[i].Scale = layers[i];
}

```


## 参考文档

http://ip.hhi.de/imagecom_G1/assets/pdfs/Overview_SVC_IEEE07.pdf

