# Code Structure Notes

## Hi3519DV500 SDK 重点目录

```text
source/
├── mpp/
│   └── sample/
│       ├── vdec/
│       ├── common/
│       └── svp/
├── common/
├── interdrv/
├── bsp/
├── osal/
├── out/
└── scripts/
```

## VDEC Sample

重点文件：

```text
sample/vdec/sample_vdec.c
sample/common/sample_comm_vdec.c
sample/common/sample_comm_vpss.c
sample/common/sample_comm_sys.c
```

### `sample_vdec.c`

职责：

```text
组织 VDEC + VPSS + VO 的主流程
启动 VDEC
启动 VPSS
绑定 VDEC 到 VPSS
调用送码流函数
```

Notion 中记录的阅读点：

```text
sample_h265_vdec_vpss_vo()
sample_start_vdec(...)
sample_start_vpss(...)
sample_send_stream_to_vdec(..., "3840x2160_8bit.h265")
```

### `sample_comm_vdec.c`

职责：

```text
创建 VDEC 通道
启动 VDEC 通道
读取本地 H264/H265 码流
填充 ot_vdec_stream
调用 ss_mpi_vdec_send_stream
```

当前 sample 输入方式：

```text
fp_strm = fopen(path, "rb")
read_len = fread(...)
stream.addr = buf + start
stream.len = read_len
ss_mpi_vdec_send_stream(...)
```

未来 RTSP 输入替换点：

```c
while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == video_stream_index) {
        ot_vdec_stream stream = {0};

        stream.addr = pkt->data;
        stream.len = pkt->size;
        stream.end_of_frame = TD_TRUE;
        stream.end_of_stream = TD_FALSE;
        stream.need_display = TD_TRUE;

        ss_mpi_vdec_send_stream(vdec_chn, &stream, milli_sec);
    }

    av_packet_unref(pkt);
}
```

## VPSS 到 YOLO/NPU

Notion 中记录 SVP/NPU sample 的方向：

```text
sample/svp/svp_npu/sample_svp_npu/sample_svp_npu_process.c
```

关键理解：

```text
ss_mpi_vpss_get_chn_frame(..., &ext_frame, ...)
ss_mpi_vpss_get_chn_frame(..., &base_frame, ...)
sample_svp_npu_acl_frame_proc(...)
sample_common_svp_npu_update_input_data_buffer_info(...)
sample_common_svp_npu_model_execute(...)
```

这说明 YOLO/NPU 不是直接吃 VDEC 输出，而是从 VPSS 通道拿处理后的 `ot_video_frame_info`，更新为模型输入 buffer 后再执行模型。

## Pelco-D 控制代码方向

最小函数职责：

```text
输入: addr, cmd1, cmd2, pan_speed, tilt_speed
输出: 7 字节 Pelco-D 指令
校验: Byte2 + Byte3 + Byte4 + Byte5 + Byte6 的低 8 位
```

Notion 中记录的 C 示例：

```c
void pelco_d_build(uint8_t addr,
                   uint8_t cmd1,
                   uint8_t cmd2,
                   uint8_t pan_speed,
                   uint8_t tilt_speed,
                   uint8_t out[7])
{
    out[0] = 0xFF;
    out[1] = addr;
    out[2] = cmd1;
    out[3] = cmd2;
    out[4] = pan_speed;
    out[5] = tilt_speed;
    out[6] = (uint8_t)(out[1] + out[2] + out[3] + out[4] + out[5]);
}
```

示例输出：

```text
FF 01 00 02 20 00 23
```
