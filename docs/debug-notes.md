# Debug And Training Notes

## VDEC 编译检查

Notion 中记录的基础检查命令：

```bash
source ~/.profile
aarch64-v01c01-linux-gnu-gcc --version
cd /home/ian/hisi_sdk/.../sample/vdec
make clean
make
ls -lh sample_vdec
file sample_vdec
```

检查点：

```text
1. 交叉编译器能被 shell 找到
2. sample_vdec 能成功编译
3. 编译产物是 ARM/aarch64 目标程序
4. Makefile 能正确找到 SDK include/lib
```

## YOLOv5 COCO128 测试训练

训练配置：

```text
YOLOv5s + coco128 + 10 epochs + CPU
```

结果对比：

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

结论：

```text
训练流程正常，模型确实在学习。
coco128 只能证明流程能跑通，不能代表项目最终场景效果。
```

记录的产物：

```text
训练目录:
/home/ian/yolo_task2/runs/train/task2_coco128_yolov5s_10epochs

best.pt:
/home/ian/yolo_task2/runs/train/task2_coco128_yolov5s_10epochs/weights/best.pt

整理出的权重:
/home/ian/yolo_task2/exports/yolov5s_coco128_10epochs.pt

检测结果图:
/home/ian/yolo_task2/runs/detect/task2_10epochs_bus/bus.jpg
```

异常现象：

```text
bus 图检测输出了 3 persons, 2 buss, 1 snowboard。
```

理解：

```text
模型能检测，但还不稳定。训练轮数不是越多就一定自动变好，数据质量、类别、标注、验证集才是项目效果核心。
```

## YOLOv5 VOC 2012 训练

训练配置：

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

`person` 类：

```text
P ≈ 0.800
R ≈ 0.792
mAP50 ≈ 0.838
mAP50-95 ≈ 0.500
```

记录的产物：

```text
最终权重:
/home/ian/yolo_task2/exports/yolov5s_voc2012_20cls_3epochs.pt

完整训练目录:
/home/ian/yolo_task2/runs/train/task2_voc2012_yolov5s_3epochs

检测结果图:
/home/ian/yolo_task2/runs/detect/voc2012_3epochs_bus/bus.jpg
```

检测结果：

```text
1 bus, 3 persons, 1 pottedplant
```

阶段性判断：

```text
VOC 版本已经足够进入下一阶段：导出 ONNX，然后准备 ATC 转 OM。
```

## 指标解释

```text
P: 预测出来的框准不准，误检多不多。
R: 真实存在的目标找没找全，漏检多不多。
mAP50: IoU=0.5 宽松标准下的综合检测能力。
mAP50-95: IoU=0.50 到 0.95 严格标准下的综合检测能力。
```

## 模型转换检查点

推荐链路：

```text
best.pt
-> export.py
-> best.onnx
-> atc
-> best.om
```

导出 ONNX 时优先保证：

```text
固定输入尺寸
固定 batch=1
不要动态 shape
NMS 放到模型外
选择 ATC 兼容的 opset
```

部署检查点：

```text
1. ONNX 输入 shape 明确，例如 1x3x320x320 或 1x3x640x640
2. 输出是 raw predictions，而不是已经写死平台不兼容的 NMS
3. ATC 转换不报 unsupported op
4. 板端能加载 .om
5. VPSS 输出帧能预处理成模型输入格式
6. 后处理能输出 bbox / class / confidence
7. bbox 中心点能转成 Pelco-D 控制方向
```
