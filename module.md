# FFmpeg 项目框架分析

## 一、整体架构概述

FFmpeg 是一个跨平台的多媒体处理框架，采用模块化设计，由多个动态库组成。其核心架构遵循 **解复用(Demuxer) → 解码(Decoder) → 滤镜(Filter) → 编码(Encoder) → 复用(Muxer)** 的数据流处理模式。

### 核心库关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           fftools (命令行工具)                          │
│                    ffmpeg.c | ffplay.c | ffprobe.c                     │
└─────────────────────────────────────────────────────────────────────────┘
         │                    │                    │                    │
         ▼                    ▼                    ▼                    ▼
┌─────────────┐      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│libavformat │      │ libavcodec  │      │ libavfilter │      │  libavutil  │
│  (容器封装) │◄────►│  (编解码)   │◄────►│   (滤镜)    │      │  (公共工具) │
└─────────────┘      └─────────────┘      └─────────────┘      └─────────────┘
         │                    │                    │                    │
         ▼                    ▼                    ▼                    ▼
┌─────────────┐      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│libavdevice │      │ libswscale  │      │libswresample│      │   compat    │
│ (设备输入输出)│      │ (图像缩放)  │      │  (音频重采样)│      │  (兼容性层) │
└─────────────┘      └─────────────┘      └─────────────┘      └─────────────┘
```

---

## 二、libavutil - 公共工具库

**路径**: `libavutil/`

### 2.1 核心功能
- 内存管理函数
- 字符串操作
- 数学运算（随机数、有理数）
- 帧数据抽象 (AVFrame)
- 包数据抽象 (AVPacket)
- 通道布局处理
- 像素格式/采样格式定义
- 日志系统
- 选项系统
- CPU 特性检测

### 2.2 关键源文件及角色

| 文件 | 作用 |
|------|------|
| `avutil.h` | 主头文件，定义库版本和基础类型 |
| `buffer.h` / `buffer.c` | 缓冲区管理 API |
| `frame.h` | AVFrame 定义（未压缩的音频/视频数据抽象） |
| `packet.h` | AVPacket 定义（压缩数据包抽象） |
| `channel_layout.h` | 音频通道布局处理 |
| `pixfmt.h` | 像素格式定义 |
| `samplefmt.h` | 音频采样格式定义 |
| `rational.h` | 有理数类型 (AVRational) |
| `dict.h` | 字典/键值对 API |
| `log.h` | 日志系统 |
| `opt.h` | 选项/属性系统 |
| `common.h` | 公共宏定义 |
| `mem.h` | 内存管理函数 |
| `imgutils.h` | 图像处理工具 |
| `cpu.h` / `cpu.c` | CPU 特性检测 |

### 2.3 核心数据结构

**AVFrame** - 未压缩的多媒体数据帧
```c
typedef struct AVFrame {
    uint8_t *data[AV_NUM_DATA_POINTERS];     // 图像/音频数据指针
    int linesize[AV_NUM_DATA_POINTERS];       // 每行字节数（视频）
    uint8_t **extended_data;                  // 扩展数据指针
    int width, height;                        // 视频尺寸
    int nb_samples;                           // 音频样本数
    int format;                               // 像素/采样格式
    int key_frame;                            // 是否关键帧
    enum AVPictureType pict_type;             // 帧类型
    int64_t pts;                              // 显示时间戳
    int64_t pkt_pts;                          // 包中的 PTS
    int64_t pkt_dts;                          // 包中的 DTS
    AVBufferRef *buf[AV_NUM_DATA_POINTERS];   // 数据缓冲区引用计数
    // ... 更多字段
} AVFrame;
```

**AVPacket** - 压缩数据包
```c
typedef struct AVPacket {
    AVBufferRef *buf;              // 数据缓冲区
    int64_t pts;                   // 显示时间戳
    int64_t dts;                   // 解码时间戳
    uint8_t *data;                 // 数据指针
    int size;                      // 数据大小
    int stream_index;              // 流索引
    int flags;                     // 标志
    // ... 更多字段
} AVPacket;
```

---

## 三、libavcodec - 编解码器库

**路径**: `libavcodec/`

### 3.1 核心功能
- 编码/解码抽象层
- 编解码器注册与管理
- 硬件加速支持 (VAAPI, VDPAU, CUVID, QSV 等)
- 码流解析
- 比特流滤镜 (Bitstream Filter)

### 3.2 关键源文件及角色

| 文件 | 作用 |
|------|------|
| `avcodec.h` | 公共 API 头文件，定义 AVCodecContext 等核心结构 |
| `avcodec.c` | AVCodecContext 成员函数实现 |
| `codec.h` | 编解码器接口定义 (AVCodec, FFCodec) |
| `codec_desc.c` | 编解码器描述符表 |
| `allcodecs.c` | 所有编解码器的 extern 声明和注册 |
| `decode.c` | 解码核心实现 (avcodec_send_packet / avcodec_receive_frame) |
| `encode.c` | 编码核心实现 (avcodec_send_frame / avcodec_receive_packet) |
| `utils.c` | 编解码器工具函数 |
| `options.c` | AVCodecContext 选项表 |
| `internal.h` | 内部 API |
| `bitstream.c` / `bitstream.h` | 比特流读取工具 |
| `bitstream_filters.c` | 比特流滤镜 |
| `bsf.c` / `bsf.h` | 比特流滤镜抽象 |
| `parser.c` / `parsers.c` | 码流解析器 |
| `profiles.c` / `profiles.h` | 编解码器配置文件 |
| `hwaccel_internal.h` | 硬件加速内部接口 |
| `hwaccels.h` | 硬件加速器列表 |
| `frame_thread_encoder.c` | 帧级多线程编码 |

### 3.3 核心数据结构

**AVCodecContext** - 编解码器上下文
```c
typedef struct AVCodecContext {
    const AVClass *av_class;           // 类信息
    enum AVMediaType codec_type;       // 媒体类型
    const struct AVCodec *codec;       // 关联的编码器/解码器
    enum AVCodecID codec_id;           // 编解码器 ID
    
    int bit_rate;                      // 目标码率
    int frame_number;                  // 帧编号
    
    // 视频参数
    int width, height;                 // 视频尺寸
    int gop_size;                      // GOP 大小
    enum AVPixelFormat pix_fmt;        // 像素格式
    
    // 音频参数
    int sample_rate;                   // 采样率
    int channels;                      // 通道数
    enum AVSampleFormat sample_fmt;    // 采样格式
    
    // 时间基
    AVRational time_base;
    
    // 私有数据
    void *priv_data;                  // 私有数据指针
} AVCodecContext;
```

**FFCodec** - FFmpeg 编解码器结构
```c
typedef struct FFCodec {
    const AVClass *p;
    enum AVMediaType type;             // 媒体类型
    enum AVCodecID id;                 // 编解码器 ID
    int priv_data_size;                // 私有数据大小
    int (*init)(AVCodecContext *);
    int (*send_frame)(AVCodecContext *, const AVFrame *);
    int (*send_packet)(AVCodecContext *, const AVPacket *);
    int (*receive_frame)(AVCodecContext *, AVFrame *);
    int (*receive_packet)(AVCodecContext *, AVPacket *);
    int (*close)(AVCodecContext *);
    void (*flush)(AVCodecContext *);
} FFCodec;
```

### 3.4 解码/编码 API 流程

**解码流程**:
```
avcodec_open2()           → 打开解码器
avcodec_send_packet()     → 发送压缩数据
avcodec_receive_frame()   → 接收解码后的帧
avcodec_flush_buffers()   → 刷新缓冲区
```

**编码流程**:
```
avcodec_open2()           → 打开编码器
avcodec_send_frame()      → 发送未压缩帧
avcodec_receive_packet()  → 接收压缩包
avcodec_flush_buffers()   → 刷新缓冲区
```

---

## 四、libavformat - 容器封装库

**路径**: `libavformat/`

### 4.1 核心功能
- 多媒体容器格式封装/解封装
- I/O 上下文管理
- 流管理
- 格式探测
- 网络协议支持 (HTTP, RTMP, SFTP 等)

### 4.2 关键源文件及角色

| 文件 | 作用 |
|------|------|
| `avformat.h` | 主 API 头文件 |
| `avformat.c` | 格式上下文管理 |
| `utils.c` | 工具函数 |
| `mux.c` | 复用核心实现 |
| `demux.c` | 解复用核心实现 |
| `avio.h` | I/O 抽象层 |
| `avio.c` | 缓冲 I/O 实现 |
| `internal.h` | 库内部 API |
| `allformats.c` | 所有格式的 extern 声明 |
| `options_table.h` | 选项表 |
| `network.h` | 网络支持 |
| `protocols.h` | I/O 协议定义 |

### 4.3 核心数据结构

**AVFormatContext** - 封装上下文
```c
typedef struct AVFormatContext {
    const AVClass *av_class;           // 类信息
    struct AVInputFormat *iformat;     // 输入格式
    struct AVOutputFormat *oformat;    // 输出格式
    void *priv_data;                  // 格式私有数据
    AVIOContext *pb;                  // I/O 上下文
    unsigned int nb_streams;          // 流数量
    AVStream **streams;                // 流数组
    char *url;                        // 文件 URL
    int64_t duration;                 // 时长
    int64_t bit_rate;                 // 码率
    int64_t start_time;               // 开始时间
    int64_t filesize;                 // 文件大小
} AVFormatContext;
```

**AVStream** - 媒体流
```c
typedef struct AVStream {
    int index;                        // 流索引
    int id;                           // 流 ID
    AVCodecContext *codec;            // 编解码上下文 (已废弃)
    int64_t start_time;              // 开始时间
    int64_t duration;                // 持续时间
    AVRational time_base;            // 时间基
    AVRational r_frame_rate;         // 帧率
    AVDictionary *metadata;         // 元数据
} AVStream;
```

### 4.4 解复用/复用 API 流程

**解复用流程**:
```
avformat_open_input()        → 打开输入
avformat_find_stream_info() → 查找流信息
av_read_frame()             → 读取数据包
av_seek_frame()             → 跳转
avformat_close_input()      → 关闭输入
```

**复用流程**:
```
avformat_alloc_output_context2() → 分配输出上下文
avformat_new_stream()            → 创建新流
avio_open2()                      → 打开输出
avformat_write_header()          → 写头
av_write_frame() / av_interleaved_write_frame() → 写数据包
av_write_trailer()               → 写尾
```

---

## 五、libavfilter - 滤镜库

**路径**: `libavfilter/`

### 5.1 核心功能
- 音频/视频滤镜图构建
- 帧级数据处理
- 格式协商
- 滤镜注册与管理

### 5.2 关键源文件及角色

| 文件 | 作用 |
|------|------|
| `avfilter.h` | 主 API 头文件 |
| `avfilter.c` | 滤镜注册与迭代 |
| `graph.c` | 滤镜图管理 |
| `buffersrc.c` | 视频源滤镜 (输入) |
| `buffersink.c` | 视频汇滤镜 (输出) |
| `abuffersrc.c` | 音频源滤镜 |
| `abuffersink.c` | 音频汇滤镜 |
| `framepool.c` | 帧池管理 |
| `formats.c` | 格式协商 |
| `graphs.c` | 滤镜图构建器 |

### 5.3 核心数据结构

**AVFilter** - 滤镜定义
```c
typedef struct AVFilter {
    const char *name;                // 滤镜名称
    const char *description;         // 描述
    const AVFilterPad *inputs;       // 输入垫
    const AVFilterPad *outputs;      // 输出垫
    int priv_data_size;              // 私有数据大小
    int (*init)(AVFilterContext *);
    int (*init_state)(AVFilterGraph *, const char *args);
    void (*uninit)(AVFilterContext *);
    int (*process_command)(AVFilterContext *, const char *cmd, const char *arg,
                          char *res, int res_len, int flags);
} AVFilter;
```

**AVFilterGraph** - 滤镜图
```c
typedef struct AVFilterGraph {
    unsigned int nb_filters;         // 滤镜数量
    AVFilterContext **filters;       // 滤镜数组
    char *scale_sws_opts;            // 缩放选项
} AVFilterGraph;
```

**AVFilterLink** - 滤镜连接
```c
typedef struct AVFilterLink {
    AVFilterContext *src;            // 源滤镜上下文
    int srcpad;                      // 源垫索引
    AVFilterContext *dst;            // 目标滤镜上下文
    int dstpad;                      // 目标垫索引
    enum AVMediaType type;           // 媒体类型
    int w, h;                        // 视频尺寸
    enum AVPixelFormat format;       // 像素格式
} AVFilterLink;
```

### 5.4 滤镜图构建流程

```
avfilter_graph_alloc()           → 分配滤镜图
avfilter_graph_parse2() / avfilter_graph_parse_ptr() → 解析滤镜字符串
avfilter_graph_config()          → 配置滤镜图
avfilter_graph_free()            → 释放滤镜图
```

---

## 六、libavdevice - 设备库

**路径**: `libavdevice/`

### 6.1 核心功能
- 输入/输出设备支持
- 屏幕捕获
- 音频采集
- 跨平台设备抽象

### 6.2 关键源文件及角色

| 文件 | 作用 |
|------|------|
| `avdevice.h` | 主 API 头文件 |
| `avdevice.c` | 设备注册 |
| `alldevices.c` | 所有设备声明 |
| `alsa.c` / `alsa_dec.c` / `alsa_enc.c` | ALSA 音频设备 (Linux) |
| `jack.c` | JACK 音频服务器 |
| `fbdev_dec.c` / `fbdev_enc.c` | Framebuffer 设备 |
| `gdigrab.c` | GDI 捕获 (Windows) |
| `dshow.c` | DirectShow 设备 (Windows) |
| `avfoundation.m` | AVFoundation 设备 (macOS) |
| `android_camera.c` | Android 相机 |
| `lavfi.c` | Libavfilter 虚拟设备 |
| `v4l2.c` / `v4l2-common.c` | Video4Linux2 设备 |
| `opencl.c` | OpenCL 设备 |
| `SDL/` | SDL 输出设备 |

---

## 七、libswresample - 音频重采样库

**路径**: `libswresample/`

### 7.1 核心功能
- 音频重采样
- 采样格式转换
- 通道布局转换
- 混音

### 7.2 关键源文件及角色

| 文件 | 作用 |
|------|------|
| `swresample.h` | 主 API 头文件 |
| `swresample.c` | 核心实现 |
| `resample.c` | 重采样算法 |
| `rematrix.c` | 通道矩阵重映射 |
| `audioconvert.c` | 音频格式转换 |
| `options.c` | 选项管理 |

### 7.3 核心数据结构

**SwrContext** - 重采样上下文
```c
typedef struct SwrContext {
    // 输入参数
    AVChannelLayout in_ch_layout;    // 输入通道布局
    enum AVSampleFormat in_sample_fmt; // 输入采样格式
    int in_sample_rate;              // 输入采样率
    
    // 输出参数
    AVChannelLayout out_ch_layout;   // 输出通道布局
    enum AVSampleFormat out_sample_fmt; // 输出采样格式
    int out_sample_rate;             // 输出采样率
} SwrContext;
```

### 7.4 API 流程
```
swr_alloc() / swr_alloc_set_opts() → 分配重采样上下文
swr_init()                         → 初始化
swr_convert()                      → 执行重采样
swr_free()                          → 释放
```

---

## 八、libswscale - 图像缩放库

**路径**: `libswscale/`

### 8.1 核心功能
- 图像缩放/拉伸
- 像素格式转换
- 颜色空间转换
- 色域转换

### 8.2 关键源文件及角色

| 文件 | 作用 |
|------|------|
| `swscale.h` | 主 API 头文件 |
| `swscale.c` | 公共 API 实现 |
| `input.c` | 输入处理 |
| `output.c` | 输出处理 |
| `rgb2rgb.c` / `rgb2rgb_template.c` | RGB 转换 |
| `format.c` | 格式检测 |
| `graph.c` | 缩放图构建 |
| `ops*.c` | 缩放操作实现 |
| `swscale_internal.h` | 内部结构 |
| `utils.c` | 工具函数 |

### 8.3 核心数据结构

**SwsContext** - 缩放上下文
```c
typedef struct SwsContext {
    // 源参数
    int srcW, srcH;                  // 源尺寸
    enum AVPixelFormat srcFormat;    // 源格式
    
    // 目标参数
    int dstW, dstH;                  // 目标尺寸
    enum AVPixelFormat dstFormat;    // 目标格式
    
    // 标志
    int flags;                       // 缩放算法标志
} SwsContext;
```

### 8.4 API 流程
```
sws_getContext()    → 创建缩放上下文
sws_scale()         → 执行缩放
sws_freeContext()   → 释放上下文
```

---

## 九、fftools - 命令行工具

**路径**: `fftools/`

### 9.1 核心文件

| 文件 | 作用 |
|------|------|
| `ffmpeg.c` | 主转换程序入口 |
| `ffmpeg.h` | ffmpeg 主头文件 |
| `ffmpeg_opt.c` | 选项解析 |
| `ffmpeg_dec.c` | 解码逻辑 |
| `ffmpeg_enc.c` | 编码逻辑 |
| `ffmpeg_demux.c` | 解复用逻辑 |
| `ffmpeg_mux.c` | 复用逻辑 |
| `ffmpeg_mux_init.c` | 复用初始化 |
| `ffmpeg_filter.c` | 滤镜处理 |
| `ffmpeg_sched.c` | 调度逻辑 |
| `ffmpeg_hw.c` | 硬件加速支持 |
| `cmdutils.c` / `cmdutils.h` | 命令行工具函数库 |
| `opt_common.c` | 通用选项处理 |
| `ffplay.c` | 播放器 |
| `ffprobe.c` | 信息探测工具 |
| `sync_queue.c` / `sync_queue.h` | 同步队列 |

### 9.2 ffmpeg 工作流程

```
1. 解析输入输出文件/流
2. 打开输入文件 (avformat_open_input)
3. 查找流信息 (avformat_find_stream_info)
4. 初始化解码器
5. 初始化滤镜图
6. 初始化编码器
7. 打开输出文件
8. 循环处理:
   - 读取输入帧 (av_read_frame)
   - 解码 (avcodec_send_packet / avcodec_receive_frame)
   - 滤镜处理 (av_buffersrc_add_frame / av_buffersink_get_frame)
   - 编码 (avcodec_send_frame / avcodec_receive_packet)
   - 写输出 (av_interleaved_write_frame)
9. 写输出文件尾 (av_write_trailer)
10. 清理资源
```

---

## 十、构建系统

**路径**: `ffbuild/` 和 `configure`

### 10.1 关键文件

| 文件 | 作用 |
|------|------|
| `configure` | 配置脚本 (生成 config.mak 和 config.h) |
| `ffbuild/common.mak` | 通用 Make 规则 |
| `ffbuild/library.mak` | 库构建规则 |
| `ffbuild/arch.mak` | 架构特定规则 |
| `ffbuild/bin2c.c` | 二进制转 C 工具 |
| `ffbuild/version.sh` | 版本生成脚本 |
| `ffbuild/pkgconfig_generate.sh` | pkg-config 生成 |

### 10.2 构建流程

```
./configure [选项]  → 生成 config.mak 和 config.h
make                → 使用 Makefile 构建
make install        → 安装
```

---

## 十一、组件间交互关系

### 11.1 典型媒体处理流程

```
                    ┌──────────────────────────────────────────────┐
                    │                   ffmpeg/ffplay               │
                    └──────────────────────────────────────────────┘
                                      │
                                      ▼
                    ┌──────────────────────────────────────────────┐
                    │                 libavformat                  │
                    │  (avformat_open_input / av_read_frame)        │
                    └──────────────────────────────────────────────┘
                                      │
                                      ▼ AVPacket
                    ┌──────────────────────────────────────────────┐
                    │                 libavcodec                   │
                    │  (avcodec_send_packet / avcodec_receive_frame)│
                    └──────────────────────────────────────────────┘
                                      │
                                      ▼ AVFrame
                    ┌──────────────────────────────────────────────┐
                    │                 libavfilter                 │
                    │  (av_buffersrc_add_frame / av_buffersink_get_frame)│
                    └──────────────────────────────────────────────┘
                                      │
                                      ▼ AVFrame
                    ┌──────────────────────────────────────────────┐
                    │              libswscale/libswresample        │
                    │  (sws_scale / swr_convert)                   │
                    └──────────────────────────────────────────────┘
                                      │
                                      ▼ AVFrame
                    ┌──────────────────────────────────────────────┐
                    │                 libavcodec                   │
                    │  (avcodec_send_frame / avcodec_receive_packet)│
                    └──────────────────────────────────────────────┘
                                      │
                                      ▼ AVPacket
                    ┌──────────────────────────────────────────────┐
                    │                 libavformat                   │
                    │  (av_interleaved_write_frame / av_write_trailer)│
                    └──────────────────────────────────────────────┘
```

### 11.2 数据流中的核心结构

| 阶段 | 数据结构 | 位置 |
|------|----------|------|
| 输入/容器 | AVPacket | libavcodec/packet.h |
| 解码后 | AVFrame | libavutil/frame.h |
| 滤镜处理 | AVFrame | libavutil/frame.h |
| 缩放/重采样 | AVFrame | libavutil/frame.h |
| 编码后 | AVPacket | libavcodec/packet.h |
| 输出/容器 | AVFormatContext | libavformat/avformat.h |

---

## 十二、关键设计模式总结

### 12.1 注册模式
通过 `*_register_all()` 函数在库初始化时注册所有组件，如 `avcodec_register_all()`、`avfilter_register_all()` 等。

### 12.2 抽象迭代器
如 `av_codec_iterate()`、`avfilter_iterate()` 用于遍历编解码器、滤镜等组件。

### 12.3 上下文模式
AVCodecContext、AVFormatContext、AVFilterGraph 等作为操作句柄，封装了组件的所有状态和配置。

### 12.4 发送/接收模式
编解码器采用 send/receive 模式：
- 解码：`avcodec_send_packet()` + `avcodec_receive_frame()`
- 编码：`avcodec_send_frame()` + `avcodec_receive_packet()`

### 12.5 引用计数
AVBuffer 和 AVFrame 使用引用计数管理内存，通过 `av_buffer_ref()` 和 `av_buffer_unref()` 操作。

### 12.6 选项系统
统一的 AVOption API 用于配置各种组件，可通过 `av_opt_set()` 和 `av_opt_get()` 访问。

### 12.7 回调机制
用于 I/O 中断、过滤链事件等场景，如 `AVIOInterruptCB`。

---

## 十三、目录结构总览

```
FFmpeg/
├── compat/              # 兼容性层（Windows pthread, cuda, openssl 等）
├── doc/                 # 文档
│   ├── examples/        # API 使用示例
│   └── *.texi           # 文档源文件
├── ffbuild/             # 构建系统
│   ├── common.mak      # 通用 Make 规则
│   ├── library.mak     # 库构建规则
│   └── arch.mak        # 架构规则
├── fftools/             # 命令行工具
│   ├── ffmpeg.c        # 转换工具
│   ├── ffplay.c        # 播放器
│   └── ffprobe.c       # 探测工具
├── libavcodec/         # 编解码器库
├── libavdevice/        # 设备库
├── libavfilter/        # 滤镜库
├── libavformat/        # 容器封装库
├── libavutil/          # 公共工具库
├── libswresample/      # 音频重采样库
├── libswscale/         # 图像缩放库
├── presets/            # 编码预设文件
├── tests/              # 测试
│   ├── fate/           # FATE 测试定义
│   └── *.c             # 测试工具源码
└── tools/              # 辅助工具
    ├── qt-faststart.c  # MP4 faststart 工具
    ├── veimage.c       # 图像提取工具
    └── ...
```

---

## 十四、总结

FFmpeg 采用高度模块化的设计，7 个核心库各司其职：

| 库 | 职责 |
|----|------|
| libavutil | 基础数据结构和工具函数 |
| libavcodec | 音视频编解码 |
| libavformat | 容器格式封装/解封装 |
| libavfilter | 滤镜链处理 |
| libavdevice | 输入输出设备 |
| libswresample | 音频重采样和格式转换 |
| libswscale | 图像缩放和格式转换 |

这种设计使得各个库可以独立使用，也可以组合成一个完整的多媒体处理流水线。fftools 提供了命令行层面的整合，让用户可以方便地进行多媒体处理操作。
