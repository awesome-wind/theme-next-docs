---
title: MediaSDK编码 
date: 2022-03-29 15:10:28
update: 2022-04-08
categories: [Codec]
tags: [QSV, Encoder, MediaSDK]
---

> 前置条件：下载并安装好[media sdk工程](https://www.intel.com/content/www/us/en/developer/tools/media-sdk/choose-download-client.html)

使用visual studio打开默认的工程sample_encode.sln后，在vs中可以看到整个工程的代码结构。程序的启动点位于sample_encode.cpp中的main函数，包含了编码器创建，编码过程以及销毁流程。

<!-- more -->

## main函数结构

查看代码逻辑，整体代码流程可分为以下五个部分

**解析外部传参**
```c++
// Parsing Input Stream working with presets
sts = ParseInputString(argv, (mfxU8)argc, &Params);
ModifyParamUsingPresets(Params);
```

**创建Pipeline**
```c++
pPipeline.reset(CreatePipeline(Params));
sts = pPipeline->Init(&Params);
// print current kind of pipeline info
pPipeline->PrintInfo(); 
```

**摄像头操作**
```c++
// using yuv file as input generally,
pPipeline->CaptureStartV4L2Pipeline();
pPipeline->CaptureStopV4L2Pipeline();
```
> if camera is wanted, macro ENABLE_V4L2_SUPPORT should be added to project

**编码过程**
```c++
sts = pPipeline->Run();
if(sts  == MX_ERR_DEVICE_LOST || sts == MFX_ERR_DEVICE_FAILED){
    sts = pPipeline->ResetDevice();
    sts = pPipeline->ResetMFXComponents(&Params);
}
```

**销毁逻辑**
```c++
pPipeline->Close();
```

## 传参解析

外部传递参数存在可选与必选参数，

#### 必选参数

| 编码标准 | 输入文件 | 输出文件 | 宽 | 高 |
| --- | --- | --- | --- | --- |
| h264/h265/jepg | -i xxx.yuv | -o xxx.bitstream | -w xxx | -h xxx |

`e.g. sample_encode.exe h264 -i input.yuv -o output.h264 –w 720 –h 480`

#### 可选参数

| 编码实现 | 编码格式 | 码控方式 | 会议模式 |
| --- | --- | --- | --- |
| -sw/-hw | -yuy2/-nv12/-rgb4 | -vbr/-cbr/-qvbr/-avbr/-cqp | -vcm |
| I帧间隔 | 参考帧数 | HEVC P帧参考帧数 | svc 设置 |
| -idr_internal | -x numRef | -num_active_P numRef | -AvcTemporalLayers "1 2 4 0 0 0 0 0" |
| slice 数量 | GOP size | B帧 | 其他常用 |
| -num_slice | -g size | -bref/-nobref(default) | -f/-b/-u/-q |

> HEVC中参考帧数与svc设置是互斥的，直接会导致编码器初始化失败.参数设置详见readme-encode_windows.pdf

#### 解析过程
默认设置使用硬编，无camera，参考帧数量为0，I420格式，而后通过匹配外部传入参数对pParams进行赋值,e.g.
```c++
if(0 == msdk_strcmp(strInput[i],MSDK_STRING("-sw")){
    pParams->bUseHWLib = false;
}
```

# 创建pipeline

#### pipeline reset
mediaSDK提供了三种pipeline模式
```c++
CRegionEncodingPipeline : 区域编码模式，不支持硬编
CUserPipeline：自定义编码pipeline，可设入插件，进行图像处理，demo给出的为旋转图像能力
CEncodingPipeline：普通编码模式
```


#### pipeline init
1. 依据参数与限制条件，选择最合适的硬件/软件实现
```c++
mfxInitParamlWrap initPar;
initPar.Version.Major = 1;
initPar.Version.Minor = 0;
initPar.GPUCopy = nParams->gpuCopy;
mfxStatus sts = GetImpl(*pParams,initPar.Implementation);
```

2. 使用获取到的编码器去初始化mfxsession
```c++
sts = m_mfxSession.InitEx(initPar);
```

3. 获取系统版本，检验特性支持的api版本，满足时加载编码器
如果有自定义的编码器，则加载自定义的,若没有指定编码器的path或者guid则检查mediasdk当前版本是否默认含有上述选择出的编解码器，有则加载，否则判断为不支持
```c++
mfxVersion version;
sts = MFXQueryVersion(m_mfxSession,&version);

// do some check by version
if (pParams->pluginParams.type == MFX_PLUGINLOAD_TYPE_FILE && msdk_strnlen(pParams->pluginParams.strPluginPath, sizeof(pParams->pluginParams.strPluginPath))){
    m_pUserModule.reset(new MFXVideoUSER(m_mfxSession));
    m_pPlugin.reset(LoadPlugin(MFX_PLUGINTYPE_VIDEO_ENCODE, m_mfxSession, pParams->pluginParams.pluginGuid, 1, pParams->pluginParams.strPluginPath, (mfxU32)msdk_strnlen(pParams->pluginParams.strPluginPath, sizeof(pParams->pluginParams.strPluginPath))));
    if (m_pPlugin.get() == NULL) sts = MFX_ERR_UNSUPPORTED;
} else {
    bool isDefaultPlugin = false;
    if (AreGuidsEqual(pParams->pluginParams.pluginGuid, MSDK_PLUGINGUID_NULL)) {
        mfxIMPL impl = pParams->bUseHWLib ? MFX_IMPL_HARDWARE : MFX_IMPL_SOFTWARE;
        pParams->pluginParams.pluginGuid = msdkGetPluginUID(impl, MSDK_VENCODE, pParams->CodecId);
        isDefaultPlugin = true;
    }
    if (!AreGuidsEqual(pParams->pluginParams.pluginGuid, MSDK_PLUGINGUID_NULL)) {
        m_pPlugin.reset(LoadPlugin(MFX_PLUGINTYPE_VIDEO_ENCODE, m_mfxSession, pParams->pluginParams.pluginGuid, 1));
        if (m_pPlugin.get() == NULL) sts = MFX_ERR_UNSUPPORTED;
    }
}
```

4. 创建编码器与初始化
```c++
m_pmfxENC = new MFXVideoENCODE(m_mfxSession); // create encoder

// if no camera, file input system is inited.
if(!isV4L2InputEnable){
    sts = m_FileReader.Init(pParams->InputFiles, nParams->FileInputFourCC, readerShift);
}
sts = InitFileWriters(pParams); // init output file

sts = CretaeAlloctor();  // memory init used by encoding
sts = InitMfxEncParams(pParams); // collect all kinds of encoder params 
sts = InitMfxVppParams(pParams); // init vpp if needed
sts = ResetMFXComponents(pParams); // init task pool and set params into encoder
InitV4L2Pipeline(pParams); // init camera if V4L2 is opened

``` 

# encode

创建for循环，循环进行以下流程的处理
```c++
读帧：LoadNextFrame(mfxFrameSurface1)
编码：m_mfxENC->EncodeFrameAsync(mfxEncodeCtrl,mfxFrameSurface1,&mfxBitstreamWrapper,&mfxSyncPoint)
写流：WriteNextFrame(&mfxBitstream，bool)
```

#### ExtendSurface创建
extendSurface包含了编码器的数据信息与控制信息，从文件中的读取的帧会按照特定格式存放于ExtendSurface.pSurface中，ExtendSurface.pCtrl则带有帧信息，有一定的控制作用，例如编I帧等
```c++
struct ExtendSurface {
    mfxFrameSurface1 *pSurface;  // 输入编码器的帧数据
    mfxEncodeCtrl    *pCtrl;  // 编码器控制参数
    mfxSyncPoint      Syncp;  //同步时机
}

ExtendedSurface encSurface,preencSurface,vppSurface = {};
m_nEncSurfIdx = GetFreeSurface(m_pEncSurfaces, m_EncResponse.NumFrameActual);
preencSurface.pSurface = vppSurface.pSurface = &m_pEncSurfaces[m_nEncSurfIdx];
```
#### sTask创建
sTask会创建任务池，承担两项任务职能
- 取得编码后的码流数据
- 将码流数据写入文件

每次需要执行代码时，便获取当前的空任务，获取时会将当前的任务队列中的码流写文件任务执行，编码完成后会将编码后的数据填入`pCurrentTask.mfxBS`中。
```c++
sts = GetFreeTask(&pCurrentTask);
    WriteNextFrame(mfxBitstream,bool) in GetFreeTask()
EncodeOneFrame(encSurface, pCurrentTask);
```