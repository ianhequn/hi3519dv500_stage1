# Notion Learning Summary

Source parent page:

```text
基于hi3519dv500的嵌入式开发
```

Reading path preserved from Notion:

```text
基于hi3519dv500的嵌入式开发
├── 文件整理
├── vdec sample流程
├── pelco-d云台协议
├── vs code阅读
├── 第一次的命令终端是什么
├── yolo训练
│   ├── yolo的版本的更迭
│   │   └── 多尺度网格 / anchor / IoU / NMS 理解记录
│   ├── .pt是什么
│   ├── 实践过程（测试版）
│   │   └── 这些参数意义的理解
│   └── 实例训练（voc版）
└── 为什么要使用文件转换
    └── 怎么传播的
```

## 父页面目标

Notion 中记录的项目主线是把网络摄像头视频流接入 Hi3519DV500，并形成从视频解码、图像处理、目标检测到云台控制的闭环：

```text
RTSP 视频流
-> FFmpeg 拿到 H264/H265 码流
-> VDEC 硬件解码成 YUV
-> VPSS 缩放/格式转换
-> YOLO 推理
-> Pelco-D 控制云台
```

## 文件整理

Notion 中把 SDK 目录重点定位到：

```text
source/
├── mpp/        媒体处理平台，VDEC/VPSS/VENC/VI/VO 等
├── common/     系统公共组件
├── interdrv/   板端驱动相关
├── bsp/        板级支持包
├── osal/       OS 抽象层
├── out/        编译输出
└── scripts/    编译脚本
```

其中 `out` 的职责被记录为：

```text
include/   SDK 头文件
lib/       已编译好的库，比如 libss_mpi.a
ko/        内核模块
obj/       中间文件
```

在 sample 目录里，当前重点是：

```text
vdec/      视频解码，第一重点
common/    公共封装，第一重点
svp/       AI 推理相关，后面看
vgs/       图像处理辅助，后面可能用
region/    画框、叠加区域，后面可能用
```

## VDEC Sample 流程

Notion 中记录的 VDEC sample 调用链：

```text
main()
-> sample_h265_vdec_vpss_vo()
-> sample_start_vdec()
-> sample_comm_vdec_start()
-> sample_send_stream_to_vdec()
-> sample_comm_vdec_start_send_stream()
-> pthread_create(... sample_comm_vdec_send_stream ...)
-> sample_comm_vdec_send_stream()
-> fopen(码流文件)
-> sample_comm_vdec_send_stream_process()
-> fread(读取 H265/H264 数据)
-> sample_comm_vdec_handle_send_stream()
-> sample_comm_vdec_send_stream_proc()
-> ss_mpi_vdec_send_stream()
-> VDEC 硬件解码
```

这个记录说明当前 sample 的输入是本地 H264/H265 文件，后续要把 `fread` 输入替换成 FFmpeg 从 RTSP 中拿到的压缩码流。

## VS Code 阅读记录

Notion 中明确区分了当前 sample 和未来改造目标：

```text
现在 sample:
本地 H264/H265 文件
-> fread 读压缩码流
-> 填 ot_vdec_stream
-> ss_mpi_vdec_send_stream 送进 VDEC
-> VDEC 解码
-> 绑定到 VPSS
-> VPSS 输出处理后的图像帧
-> SVP/NPU sample 从 VPSS 取帧做 YOLO/检测推理

未来:
RTSP
-> FFmpeg av_read_frame 得到 AVPacket
-> AVPacket.data / size 填到 ot_vdec_stream
-> ss_mpi_vdec_send_stream
-> 后面 VDEC/VPSS/YOLO 链路尽量复用
```

Notion 中记录的关键替换关系：

```text
现在:
fread 读到的 buf -> stream.addr
read_len         -> stream.len

未来:
AVPacket.data    -> stream.addr
AVPacket.size    -> stream.len
```

并总结了核心理解：

```text
FFmpeg 不负责解码，只负责 RTSP 拉流和解封装；
VDEC 负责硬件解码；
VPSS 负责把解码后的图像变成合适尺寸/格式；
YOLO/NPU 从 VPSS 输出帧拿输入 buffer 做推理。
```

## 第一次命令终端记录

Notion 中记录的命令含义：

```text
source ~/.profile
加载环境变量，让系统能找到海思交叉编译器。

aarch64-v01c01-linux-gnu-gcc --version
检查交叉编译器是否安装成功。

cd /home/ian/hisi_sdk/.../sample/vdec
进入 VDEC 例程目录。

make clean
清理上一次编译生成的中间文件。

make
根据 Makefile 编译当前例程。

ls -lh sample_vdec
查看编译产物是否存在、大小是多少。
```

Notion 中还记录了一段阶段性自我理解：

```text
Linux 和 C/C++ 我不是单独刷语法，而是结合项目补。Linux 方面我已经能完成 SDK 解压、环境变量配置、依赖安装和 sample 编译。C/C++ 方面我已经能沿着 sample 的调用链看代码，理解本地码流怎么通过 fopen/fread 读入，如何填入 ot_vdec_stream，再调用 ss_mpi_vdec_send_stream 送入 VDEC。后面会继续重点补线程、内存、文件 IO、Makefile 和 FFmpeg API。
```

## Pelco-D 云台协议

Notion 中记录的控制链路：

```text
YOLO 检测目标
-> 得到目标框 bbox
-> 计算目标中心点
-> 和画面中心比较
-> 得到偏差 dx / dy
-> 生成 Pelco-D 指令
-> 串口发送给云台
-> 云台转动
```

Pelco-D 一帧通常是 7 字节：

```text
Byte1  Byte2  Byte3  Byte4  Byte5    Byte6    Byte7
同步字  地址   命令1  命令2  水平速度  垂直速度  校验和
```

校验和记录为：

```text
Byte2 + Byte3 + Byte4 + Byte5 + Byte6，只取低 8 位
```

## YOLO 训练

Notion 中选择 YOLOv5 的理由是：理解成本低、资料多、`.pt -> onnx -> om` 链路更常见。整体任务链路：

```text
数据集
-> 训练 YOLO
-> 得到 .pt
-> 导出 .onnx
-> 转成 .om
-> 放到海思/昇腾推理环境里跑
```

### YOLO 版本理解

Notion 中把 YOLOv1、YOLOv5、YOLOv8 的定位区分为：

```text
YOLOv1: 理解 YOLO 原理，单尺度网格预测，结构简单
YOLOv5: 工程训练/部署常用，多尺度，anchor-based，生态成熟
YOLOv8: 更现代的工程版本，anchor-free，decoupled head，接口统一
```

其中 YOLOv5 的结构被理解为：

```text
Image
-> Backbone 提特征
-> Neck 多尺度融合
-> Detect Head 三个尺度输出
-> NMS
```

### 多尺度、Anchor、IoU、NMS 理解

Notion 中对 YOLOv5 多尺度检测的理解是：

```text
80 x 80   stride 8    适合小目标
40 x 40   stride 16   适合中目标
20 x 20   stride 32   适合大目标
```

更准确的表述被记录为：

```text
不是“网格本身有大有小”，而是特征图分辨率不同，每个网格对应原图区域大小不同。
```

YOLOv5 每个 anchor 预测：

```text
x, y, w, h
objectness
class probabilities
```

推理后处理流程：

```text
模型输出大量候选框
-> 置信度过滤
-> 类别分数计算
-> NMS 去重
-> 输出最终检测框
```

### .pt 理解

Notion 中记录：

```text
.pt 是 PyTorch 模型权重文件。
.pt = 训练好的 YOLO 模型参数
best.pt 是验证集效果最好的模型
last.pt 是最后一轮训练结束时的模型
```

并指出 `.pt` 主要给 PyTorch 用，不一定能直接在海思板子上跑。

### COCO128 测试训练

Notion 中记录的测试训练：

```text
模型：YOLOv5s
数据集：coco128
epochs：10
设备：CPU
```

指标变化：

```text
1 epoch:
P = 0.460
R = 0.493
mAP50 = 0.454
mAP50-95 = 0.266

10 epochs:
P = 0.686
R = 0.523
mAP50 = 0.593
mAP50-95 = 0.344
```

Notion 中的结论：

```text
模型确实在学习，mAP50 从 0.454 提到 0.593，说明训练流程正常。但这不是项目最终准确率，因为 coco128 是通用小数据集，不是目标场景数据。
```

### VOC 训练

Notion 中记录的 VOC 训练配置：

```text
数据集：Pascal VOC 2012
类别数：20 类
训练图：5717 张
验证图：5823 张
模型：YOLOv5s
输入尺寸：320
epochs：3
设备：CPU
```

三轮指标：

```text
epoch 0:
P = 0.471
R = 0.414
mAP50 = 0.407
mAP50-95 = 0.209

epoch 1:
P = 0.631
R = 0.585
mAP50 = 0.609
mAP50-95 = 0.306

epoch 2:
P = 0.704
R = 0.630
mAP50 = 0.679
mAP50-95 = 0.395
```

Notion 中记录 `person` 类表现：

```text
P ≈ 0.800
R ≈ 0.792
mAP50 ≈ 0.838
mAP50-95 ≈ 0.500
```

Notion 中的项目意义判断：

```text
这很适合后面做“摄像头检测人 -> 算框中心 -> 控制云台跟踪”。
```

### 指标理解

Notion 中记录：

```text
P: Precision，预测出来的框里有多少是真的，P 高说明误检少。
R: Recall，真实存在的目标里有多少被模型找出来，R 高说明漏检少。
mAP50: IoU=0.5 标准下，各类别 AP 的平均值。
mAP50-95: IoU=0.50 到 0.95 多个阈值下的平均，更严格。
```

## 文件转换

Notion 中记录的模型转换链路：

```text
best.pt
-> best.onnx
-> best.om
```

每一步含义：

```text
.pt: PyTorch 训练权重
.onnx: 通用模型交换格式
.om: 海思/昇腾推理用的离线模型格式
```

Notion 中对部署阶段的理解：

```text
VPSS 输出图像
-> 预处理成 1x3x640x640
-> .om 推理
-> raw boxes
-> NMS
-> 检测结果
```

常见转换风险：

```text
ONNX opset 不兼容
某些算子 ATC 不支持
输入 shape 没写死
模型里有动态 shape
SiLU / Focus / NMS 等算子不兼容
后处理放进模型导致转换失败
```

下一步在 Notion 中记录为：导出 ONNX，然后准备 ATC 转 OM。
