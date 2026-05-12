# Hi3519DV500 Stage 1

This repository records the first-stage study notes for a Hi3519DV500 embedded vision project.

Main goal:

```text
RTSP stream
-> FFmpeg demux H264/H265 stream
-> HiSilicon VDEC hardware decode
-> VPSS resize / format conversion
-> YOLO inference
-> Pelco-D PTZ control
```

Current stage:

- Read Hi3519DV500 SDK sample structure.
- Trace the VDEC sample call chain.
- Understand how RTSP packets can replace local `fread` input.
- Train YOLOv5 baseline models on `coco128` and Pascal VOC 2012.
- Prepare the model conversion path: `.pt -> .onnx -> .om`.

Documents:

- [Notion learning summary](docs/notion-learning-summary.md)
- [Project process](docs/project-process.md)
- [Code structure](docs/code-structure.md)
- [Debug and training notes](docs/debug-notes.md)

Note: the learning-summary document is summarized only from the Notion page hierarchy recorded by the author.
