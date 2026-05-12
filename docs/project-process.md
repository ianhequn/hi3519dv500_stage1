# Project Process

## 当前最小闭环

这个项目的最小可运行闭环不是一上来就写完整应用，而是分阶段打通：

```text
1. 读懂 SDK sample
2. 编译并运行 VDEC sample
3. 明确本地码流输入怎么送进 VDEC
4. 用 FFmpeg RTSP packet 替换本地 fread 输入
5. VDEC 输出绑定 VPSS
6. 从 VPSS 拿帧给 YOLO/NPU 推理
7. 解析检测框，计算目标中心偏差
8. 通过 Pelco-D 串口控制云台
```

## Stage 1 已完成的理解闭环

### 1. SDK 结构

重点目录：

```text
source/mpp/sample/vdec
source/mpp/sample/common
source/mpp/sample/svp
source/out/include
source/out/lib
source/out/ko
```

### 2. VDEC 输入链路

当前 sample 是：

```text
本地 H264/H265 文件
-> fopen
-> fread
-> ot_vdec_stream.addr / len
-> ss_mpi_vdec_send_stream
-> VDEC 硬件解码
```

项目目标是：

```text
RTSP
-> FFmpeg av_read_frame
-> AVPacket.data / AVPacket.size
-> ot_vdec_stream.addr / len
-> ss_mpi_vdec_send_stream
-> VDEC 硬件解码
```

### 3. VDEC / VPSS / YOLO 分工

```text
FFmpeg: 拉 RTSP 流，解封装，拿到压缩码流 packet
VDEC: 硬件解码 H264/H265，输出 YUV/NV12 图像帧
VPSS: 缩放、裁剪、格式转换，输出适合 AI 输入的帧
YOLO/NPU: 从 VPSS 输出帧构造模型输入，执行推理
后处理: 置信度过滤、NMS、坐标映射
Pelco-D: 根据目标中心偏差控制云台
```

## YOLO 训练路线

训练不是最终目的，训练的作用是得到能进入部署链路的模型产物。

```text
图片 + 标签
-> YOLOv5 训练
-> best.pt
-> export.py 导出 ONNX
-> ATC 转换 OM
-> 板端推理
```

当前已经用 `coco128` 和 `VOC 2012` 跑通训练流程，其中 VOC 2012 的 `person` 类结果更接近后续云台跟踪需求。

## 下一步建议

优先级从高到低：

```text
1. 固定一个 YOLOv5s VOC 权重，导出 ONNX
2. 固定输入尺寸、batch=1、关闭动态 shape
3. 用 ATC 尝试 ONNX -> OM
4. 在 PC 侧先验证 ONNX 输出 shape 和后处理
5. 在板端验证 OM 推理能跑
6. 再接入 VPSS 帧输入
7. 最后接 Pelco-D 云台闭环
```

不要现在就追求高 mAP。当前最有价值的是先把部署链路跑通。
