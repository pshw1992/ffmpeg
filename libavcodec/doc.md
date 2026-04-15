# libavcodec 模块详细分析

## 一、模块概述

libavcodec 是 FFmpeg 的核心编解码库，负责音视频的编码和解码功能。它提供了对数百种编解码器的支持，包括视频编解码器（如 H.264、H.265、VP9、AV1）、音频编解码器（如 AAC、MP3、Opus）、字幕编解码器等。

### 1.1 核心职责
- 编解码器注册与管理
- 编解码 API 实现（send/receive 模式）
- 帧/包数据结构管理
- 硬件加速支持
- 比特流解析和处理
- 多线程解码支持

---

## 二、目录结构

```
libavcodec/
├── avcodec.h          # 主公共 API 头文件
├── codec.h            # AVCodec/FFCodec 结构定义
├── codec_internal.h   # 编解码器内部结构
├── avcodec_internal.h # 内部 API
├── internal.h         # 通用内部定义
├── packet.h           # AVPacket 结构定义
├── decode.c           # 解码核心实现
├── encode.c           # 编码核心实现
├── utils.c            # 工具函数
├── allcodecs.c        # 编解码器注册
├── codec_desc.c       # 编解码器描述符
├── Options.c          # 选项管理
│
├── h264/              # H.264 相关实现
├── hevc/              # H.265/HEVC 相关实现
├── av1/               # AV1 相关实现
├── mpeg4/             # MPEG-4 相关实现
├── mpeg2/             # MPEG-2 相关实现
├── aac/               # AAC 音频编解码器
├── ac3/               # AC3 音频编解码器
├── vpx/               # libvpx 包装器
├── openh264/          # OpenH264 包装器
├── vaapi/             # VAAPI 硬件加速
├── vdpau/             # VDPAU 硬件加速
├── cuvid/             # NVIDIA CUVID 硬件加速
├── qsv/               # Intel QSV 硬件加速
│
└── aarch64/           # ARM64 架构优化
    ├── arm/            # ARM 架构优化
    ├── x86/            # x86/x64 架构优化
    └── mips/           # MIPS 架构优化
```

---

## 三、核心数据结构

### 3.1 AVCodecContext - 编解码器上下文

这是 libavcodec 最核心的结构体，封装了编解码器的所有状态和配置。

```c
typedef struct AVCodecContext {
    /* 基本信息 */
    const AVClass *av_class;           // 日志类信息
    enum AVMediaType codec_type;       // 媒体类型 (视频/音频/字幕)
    const struct AVCodec  *codec;       // 关联的编解码器
    enum AVCodecID     codec_id;        // 编解码器 ID

    /* 编码参数 */
    int64_t bit_rate;                  // 目标码率
    int bit_rate_tolerance;            // 码率容忍度

    /* 视频参数 */
    int width, height;                 // 视频分辨率
    int gop_size;                      // GOP 大小 (关键帧间隔)
    enum AVPixelFormat pix_fmt;        // 像素格式

    /* 音频参数 */
    int sample_rate;                   // 采样率
    int channels;                      // 通道数
    enum AVSampleFormat sample_fmt;    // 采样格式
    AVChannelLayout ch_layout;         // 通道布局

    /* 时间基 */
    AVRational time_base;              // 时间基准
    int max_b_frames;                  // 最大 B 帧数

    /* 私有数据 */
    void *priv_data;                  // 编解码器私有数据
    struct AVCodecInternal *internal;  // libavcodec 内部数据
} AVCodecContext;
```

### 3.2 FFCodec - FFmpeg 编解码器结构

FFCodec 是 FFmpeg 内部使用的编解码器描述符，包含实际的回调函数指针。

```c
typedef struct FFCodec {
    AVCodec p;                         // 公共 AVCodec 结构

    unsigned caps_internal:24;          // 内部能力标志
    unsigned is_decoder:1;             // 是否为解码器

    int priv_data_size;                // 私有数据大小

    /* 回调函数类型 */
    unsigned cb_type:3;               // 回调类型

    union {
        /* 旧式解码回调 (deprecated) */
        int (*decode)(AVCodecContext *avctx, AVFrame *frame,
                      int *got_frame_ptr, AVPacket *avpkt);

        /* 新式 receive_frame 回调 */
        int (*receive_frame)(AVCodecContext *avctx, AVFrame *frame);

        /* 旧式编码回调 (deprecated) */
        int (*encode)(AVCodecContext *avctx, AVPacket *avpkt,
                      const AVFrame *frame, int *got_packet_ptr);

        /* 新式 receive_packet 回调 */
        int (*receive_packet)(AVCodecContext *avctx, AVPacket *avpkt);
    } cb;

    int (*init)(AVCodecContext *);     // 初始化函数
    int (*close)(AVCodecContext *);     // 关闭函数
    void (*flush)(AVCodecContext *);   // 刷新函数
} FFCodec;
```

### 3.3 AVPacket - 压缩数据包

AVPacket 用于存储压缩数据（编码后的视频帧或音频样本）。

```c
typedef struct AVPacket {
    AVBufferRef *buf;                 // 数据缓冲区引用
    int64_t pts;                       // 显示时间戳
    int64_t dts;                       // 解码时间戳
    uint8_t *data;                     // 数据指针
    int   size;                        // 数据大小
    int   stream_index;                // 流索引
    int   flags;                       // 标志 (KEY_FRAME 等)

    AVPacketSideData *side_data;      // 附加数据
    int side_data_elems;

    int64_t duration;                  // 持续时间
    int64_t pos;                       // 流中的位置
} AVPacket;
```

### 3.4 AVCodecInternal - 内部上下文

AVCodecInternal 是 AVCodecContext 的内部扩展，存储 libavcodec 库层面的内部状态。

```c
typedef struct AVCodecInternal {
    int is_copy;                       // 是否为帧线程副本
    int is_frame_mt;                   // 是否使用帧级多线程

    struct FramePool *pool;            // 帧池
    AVBSFContext *bsf;                 // 比特流滤镜

    AVPacket *in_pkt;                  // 输入数据包
    AVPacket *last_pkt_props;         // 上一个数据包的属性

    uint8_t *byte_buffer;              // 编码器字节缓冲区
    unsigned int byte_buffer_size;

    AVFrame *in_frame;                 // 输入帧
    AVFrame *recon_frame;              // 重构帧

    int draining;                      // 是否正在排空
    AVPacket *buffer_pkt;              // 缓冲的数据包
    AVFrame *buffer_frame;              // 缓冲的帧
    int draining_done;                 // 排空完成标志
} AVCodecInternal;
```

---

## 四、核心 API

### 4.1 解码 API

#### 解码流程

```
avcodec_open2()              → 打开解码器
avcodec_send_packet()        → 发送压缩数据
avcodec_receive_frame()      → 接收解码后的帧
avcodec_flush_buffers()      → 刷新缓冲区
```

#### 关键函数实现

**avcodec_send_packet()** - 将压缩数据包发送给解码器
```c
// 位于 decode.c
int attribute_align_arg avcodec_send_packet(AVCodecContext *avctx, const AVPacket *avpkt)
{
    // 1. 检查编解码器是否已打开
    // 2. 应用比特流滤镜 (如果配置了 BSF)
    // 3. 调用编解码器的 send_packet 或等效回调
    // 4. 返回 AVERROR(EAGAIN) 表示需要接收更多输出
}
```

**avcodec_receive_frame()** - 从解码器接收解码后的帧
```c
// 位于 decode.c
int attribute_align_arg avcodec_receive_frame(AVCodecContext *avctx,
                                              AVFrame *frame)
{
    // 1. 检查编解码器状态
    // 2. 调用 ff_decode_receive_frame() 获取帧
    // 3. 处理帧属性 (时间戳、帧类型等)
    // 4. 返回 AVERROR(EAGAIN) 表示需要更多输入
}
```

### 4.2 编码 API

#### 编码流程

```
avcodec_open2()              → 打开编码器
avcodec_send_frame()         → 发送未压缩帧
avcodec_receive_packet()     → 接收编码后的数据包
avcodec_flush_buffers()      → 刷新缓冲区
```

#### 关键函数实现

**avcodec_send_frame()** - 发送未压缩帧给编码器
```c
// 位于 encode.c
int attribute_align_arg avcodec_send_frame(AVCodecContext *avctx,
                                          const AVFrame *frame)
{
    // 1. 验证输入帧格式
    // 2. 处理延迟帧
    // 3. 调用编码器回调
}
```

**avcodec_receive_packet()** - 从编码器接收压缩数据包
```c
// 位于 encode.c
int attribute_align_arg avcodec_receive_packet(AVCodecContext *avctx,
                                              AVPacket *avpkt)
{
    // 1. 调用 ff_encode_receive_frame()
    // 2. 处理编码器输出
    // 3. 设置输出包的属性 (PTS/DTS/FLAGS)
}
```

### 4.3 编解码器查找与注册

**avcodec_find_decoder()** - 通过 ID 查找解码器
```c
const AVCodec *avcodec_find_decoder(enum AVCodecID id)
{
    void *opaque = NULL;
    const AVCodec *codec;
    // 遍历所有注册的解码器，找到匹配的 ID
    while ((codec = av_codec_iterate(&opaque))) {
        if (!av_codec_is_decoder(codec) && codec->id == id)
            return codec;
    }
    return NULL;
}
```

**avcodec_find_encoder_by_name()** - 通过名称查找编码器
```c
const AVCodec *avcodec_find_encoder_by_name(const char *name)
{
    void *opaque = NULL;
    const AVCodec *codec;
    while ((codec = av_codec_iterate(&opaque))) {
        if (av_codec_is_encoder(codec) && codec->name && !strcmp(codec->name, name))
            return codec;
    }
    return NULL;
}
```

---

## 五、编解码器注册机制

### 5.1 编解码器描述符

编解码器通过 AVCodecDescriptor 描述：

```c
typedef struct AVCodecDescriptor {
    const char *name;                 // 编解码器名称
    const char *long_name;            // 详细名称
    enum AVMediaType type;            // 媒体类型
    enum AVCodecID id;                // 编解码器 ID

    /* 能力标志 */
    int props;                        // 能力属性

    /* 代理的外部库名称 (如 "libx264") */
    const char *mime_types;           // MIME 类型列表

    /* 支持的配置文件 */
    const AVProfile *profiles;        // 支持的配置文件数组
} AVCodecDescriptor;
```

### 5.2 编解码器注册表

allcodecs.c 文件包含所有编解码器的注册声明：

```c
// 示例：注册 H.264 解码器
extern const FFCodec ff_h264_decoder;

// allcodecs.c 中的声明
const AVCodec* const avcodec_codec_list[] = {
    // 视频解码器
    &ff_h264_decoder,
    &ff_hevc_decoder,
    &ff_av1_decoder,
    &ff_vp9_decoder,
    // 音频解码器
    &ff_aac_decoder,
    &ff_mp3_decoder,
    &ff_opus_decoder,
    // ... 更多编解码器
    NULL
};
```

### 5.3 编解码器能力标志

```c
#define AV_CODEC_CAP_DR1                 (1 << 1)
// 编解码器使用 get_buffer() 分配缓冲区

#define AV_CODEC_CAP_DELAY               (1 << 5)
// 编解码器有延迟，需要在结束时刷新

#define AV_CODEC_CAP_FRAME_THREADS       (1 << 12)
// 支持帧级多线程

#define AV_CODEC_CAP_SLICE_THREADS       (1 << 13)
// 支持 slice 级多线程

#define AV_CODEC_CAP_VARIABLE_FRAME_SIZE (1 << 16)
// 音频编码器支持可变帧大小

#define AV_CODEC_CAP_HARDWARE            (1 << 18)
// 由硬件实现的编解码器

#define AV_CODEC_CAP_HYBRID              (1 << 19)
// 可能由硬件加速的编解码器
```

---

## 六、多线程解码

### 6.1 线程模型

libavcodec 支持两种多线程模型：

1. **帧级多线程 (Frame-level threading)**：多个线程同时解码不同的帧
2. **Slice 级多线程 (Slice-level threading)**：将单帧分割为多个 slice 并行解码

### 6.2 实现机制

```c
// thread.c 中的关键结构
typedef struct PerThreadContext {
    AVCodecContext *ctx;              // 线程自己的上下文
    AVPacket *pkt;                    // 待解码的数据包
    int thread_count;                 // 线程数
    pthread_t *threads;               // 线程数组
} PerThreadContext;
```

### 6.3 帧分发逻辑

```
输入 Packet
    ↓
ff_thread_get_packet()     → 从主线程获取数据包
    ↓
分发给工作线程
    ↓
各线程独立解码
    ↓
ff_thread_receive_frame()  → 收集解码结果
    ↓
输出 Frame (按 PTS 排序)
```

---

## 七、硬件加速

### 7.1 硬件配置接口

```c
typedef struct AVCodecHWConfig {
    enum AVPixelFormat pix_fmt;        // 硬件像素格式
    int methods;                       // 配置方法
    enum AVHWDeviceType device_type;  // 设备类型
} AVCodecHWConfig;

// 配置方法标志
#define AV_CODEC_HW_CONFIG_METHOD_HW_DEVICE_CTX   0x01
#define AV_CODEC_HW_CONFIG_METHOD_HW_FRAMES_CTX   0x02
#define AV_CODEC_HW_CONFIG_METHOD_INTERNAL         0x04
#define AV_CODEC_HW_CONFIG_METHOD_AD_HOC           0x08
```

### 7.2 支持的硬件加速

| 硬件加速 | 位置 | 说明 |
|----------|------|------|
| VAAPI | vaapi/* | Linux 视频加速 API |
| VDPAU | vdpau/* | NVIDIA 视频解码 API |
| CUVID | cuvid/* | NVIDIA GPU 解码 |
| QSV | qsv/* | Intel 快速视频同步 |
| VideoToolbox | videotoolbox.c | macOS 视频工具箱 |
| D3D11VA | d3d11va.c | Windows Direct3D 11 |
| OpenCL | opencl.c | 通用 GPU 计算 |

### 7.3 硬件上下文链

```
AVCodecContext.hw_device_ctx
    ↓
AVHWDeviceContext
    ↓
AVHWFramesContext (用于帧格式转换)
    ↓
硬件解码器/编码器
```

---

## 八、比特流滤镜 (Bitstream Filter)

### 8.1 BSF 架构

比特流滤镜用于在解码前/编码后处理码流数据。

```c
// bsf.c 中的核心结构
typedef struct AVBSFContext {
    const AVClass *av_class;
    const FFBitStreamFilter *filter;   // 滤镜实例
    AVCodecParameters *par_in;         // 输入参数
    AVCodecParameters *par_out;        // 输出参数

    void *priv_data;                  // 私有数据
} AVBSFContext;
```

### 8.2 内置 BSF

| 名称 | 功能 |
|------|------|
| h264_mp4toannexb | H.264: MP4 → Annex B 格式转换 |
| hevc_mp4toannexb | HEVC: MP4 → Annex B 格式转换 |
| aac_adtstoasc | AAC: ADTS → ASC 格式转换 |
| filter_units | 过滤指定类型的 NAL 单元 |
| remove_extradata | 移除码流中的 extradata |

### 8.3 使用示例

```c
// 为解码器配置 BSF
const char *bsfs = "h264_mp4toannexb";
av_bsf_list_parse_str(bsfs, &bsf);
avcodec_parameters_copy(bsf->par_in, codecpar);
av_bsf_init(bsf);

// 解码前处理
av_bsf_send_packet(bsf, pkt);
av_bsf_receive_packet(bsf, processed_pkt);
avcodec_send_packet(ctx, processed_pkt);
```

---

## 九、编解码器实现示例

### 9.1 简单解码器结构

```c
// 解码器初始化
static av_cold int decode_init(AVCodecContext *avctx)
{
    // 1. 分配私有上下文
    MyContext *c = av_mallocz(sizeof(*c));
    if (!c)
        return AVERROR(ENOMEM);

    // 2. 解析 extradata (如 SPS/PPS)
    avctx->extradata_size = ...;
    avctx->extradata = ...;

    // 3. 初始化解码器特定数据
    c->frame_count = 0;

    avctx->priv_data = c;
    return 0;
}

// 解码核心
static int decode_receive_frame(AVCodecContext *avctx, AVFrame *frame)
{
    MyContext *c = avctx->priv_data;

    // 1. 获取输入数据
    int ret = ff_decode_get_packet(avctx, avctx->internal->in_pkt);
    if (ret < 0)
        return ret;

    // 2. 解析压缩数据
    ret = parse_bitstream(avctx, avctx->internal->in_pkt->data,
                          avctx->internal->in_pkt->size, &c->slice);
    if (ret < 0)
        return ret;

    // 3. 解码为像素数据
    ret = decode_slice(avctx, c->slice, frame);
    if (ret < 0)
        return ret;

    // 4. 设置帧属性
    frame->pict_type = AV_PICTURE_TYPE_P;
    frame->pts = avctx->internal->in_pkt->pts;

    return 0;
}

// 刷新解码器
static void decode_flush(AVCodecContext *avctx)
{
    MyContext *c = avctx->priv_data;
    // 清除内部缓冲
    c->frame_count = 0;
}

// 编解码器描述符
const FFCodec ff_simple_decoder = {
    .p.name         = "simple_decoder",
    .p.type          = AVMEDIA_TYPE_VIDEO,
    .p.id            = AV_CODEC_ID_SIMPLE,
    .p.capabilities  = AV_CODEC_CAP_DR1 | AV_CODEC_CAP_DELAY,
    .priv_data_size = sizeof(MyContext),

    .init           = decode_init,
    .receive_frame  = decode_receive_frame,
    .flush          = decode_flush,

    .caps_internal  = FF_CODEC_CAP_SETS_PKT_DTS,
};
```

### 9.2 编解码器回调类型

```c
// 旧式回调 (deprecated)
FF_CODEC_DECODE_CB(decode_callback)       // decode()
FF_CODEC_ENCODE_CB(encode_callback)       // encode()

// 新式回调 (current)
FF_CODEC_RECEIVE_FRAME_CB(receive_frame_callback)  // 解码器
FF_CODEC_RECEIVE_PACKET_CB(receive_packet_callback) // 编码器
```

---

## 十、内存管理

### 10.1 帧缓冲区分配

```c
// AVCodecContext 中的缓冲区分配回调
int (*get_buffer2)(AVCodecContext *s, AVFrame *frame, int flags);

// 默认实现
int avcodec_default_get_buffer2(AVCodecContext *s, AVFrame *frame, int flags)
{
    // 1. 计算所需缓冲区大小
    int linesize[AV_NUM_DATA_POINTERS];
    av_image_fill_linesizes(linesize, s->pix_fmt, s->width);

    // 2. 分配缓冲区
    for (int i = 0; i < AV_NUM_DATA_POINTERS; i++) {
        frame->buf[i] = av_buffer_alloc(linesize[i] * s->height);
    }

    // 3. 设置 data 和 linesize
    av_image_fill_pointers(frame->data, s->pix_fmt, s->height,
                           frame->buf[0]->data, linesize);

    return 0;
}
```

### 10.2 引用计数

AVFrame 和 AVPacket 使用引用计数管理内存：

```c
// 增加引用计数
AVFrame *av_frame_ref(AVFrame *dst, const AVFrame *src)
{
    // 增加所有缓冲区的引用计数
    for (int i = 0; i < src->nb_extended_buf; i++)
        av_buffer_ref(src->extended_buf[i]);
}

// 减少引用计数
void av_frame_unref(AVFrame *frame)
{
    // 减少所有缓冲区的引用计数，必要时释放
    for (int i = 0; i < nb_buffers; i++)
        av_buffer_unref(&frame->buf[i]);
}
```

---

## 十一、常见模式与最佳实践

### 11.1 解码器错误处理

```c
int decode_receive_frame(AVCodecContext *avctx, AVFrame *frame)
{
    int ret;

    // 处理解码错误
    ret = decoder_process(avctx, frame);
    if (ret < 0) {
        // 清除损坏的输出帧
        av_frame_unref(frame);
        return ret;
    }

    // 检查输出帧有效性
    if (frame->width == 0 || frame->height == 0) {
        av_frame_unref(frame);
        return AVERROR_INVALIDDATA;
    }

    return 0;
}
```

### 11.2 时间戳处理

```c
// PTS/DTS 转换
frame->pts = avpkt->pts;
frame->pkt_dts = avpkt->dts;

// 计算 duration
if (avpkt->duration > 0)
    frame->duration = avpkt->duration;
else
    frame->duration = ff_guess_frame_duration(avctx, frame);
```

### 11.3 帧类型检测

```c
static enum AVPictureType get_pict_type(int slice_type)
{
    switch (slice_type) {
    case SLICE_TYPE_I: return AV_PICTURE_TYPE_I;
    case SLICE_TYPE_P: return AV_PICTURE_TYPE_P;
    case SLICE_TYPE_B: return AV_PICTURE_TYPE_B;
    default:            return AV_PICTURE_TYPE_NONE;
    }
}
```

---

## 十二、配置与选项

### 12.1 编解码器选项表

```c
// options_table.h
#define CODEC_FLAG_QSCALE       (1 << 1)   // 使用固定量化因子
#define CODEC_FLAG_4MV          (1 << 2)   // 每 MB 4 个运动矢量
#define CODEC_FLAG_GLOBAL_HEADER (1 << 22) // 全局头信息

#define CODEC_FLAG2_FAST        (1 << 0)  // 允许非标准加速
#define CODEC_FLAG2_NO_OUTPUT    (1 << 2)  // 跳过编码输出
```

### 12.2 选项设置示例

```c
// 设置编码质量
av_opt_set(avctx->priv_data, "crf", "23", 0);
av_opt_set_int(avctx, "b", 5000000, 0);  // 5 Mbps

// 启用多线程
avctx->thread_count = 8;
avctx->thread_type = FF_THREAD_SLICE | FF_THREAD_FRAME;
```

---

## 十三、文件清单

### 13.1 核心文件

| 文件 | 功能 |
|------|------|
| avcodec.h | 公共 API 头文件 |
| codec.h | AVCodec 结构定义 |
| codec_internal.h | FFCodec 内部结构 |
| avcodec_internal.h | 内部函数声明 |
| decode.c | 解码核心实现 |
| encode.c | 编码核心实现 |
| utils.c | 编解码器工具函数 |
| allcodecs.c | 编解码器注册表 |
| codec_desc.c | 编解码器描述符表 |
| packet.c | AVPacket 相关函数 |
| options.c | 选项管理 |

### 13.2 辅助模块

| 目录 | 功能 |
|------|------|
| h264/ | H.264 编解码器实现 |
| hevc/ | HEVC 编解码器实现 |
| mpeg2/ | MPEG-2 编解码器实现 |
| mpeg4/ | MPEG-4 编解码器实现 |
| aac/ | AAC 音频编解码器 |
| ac3/ | AC3 音频编解码器 |
| vaapi/ | VAAPI 硬件加速 |
| vdpau/ | VDPAU 硬件加速 |
| qsv/ | Intel QSV 硬件加速 |
| cuvid/ | NVIDIA CUVID 硬件加速 |

---

## 十四、总结

libavcodec 是 FFmpeg 最核心的模块，其设计特点包括：

1. **统一的 API 抽象**：通过 AVCodecContext 提供一致的编解码接口
2. **灵活的回调机制**：支持旧式 encode/decode 和新式 receive_frame/receive_packet 两种模式
3. **模块化设计**：每个编解码器独立实现，通过注册机制接入系统
4. **强大的硬件加速**：支持多种硬件加速接口
5. **完善的多线程支持**：帧级和 slice 级多线程并行解码
6. **精细的内存管理**：引用计数的帧和包管理

理解 libavcodec 的核心数据结构和 API 是掌握 FFmpeg 多媒体处理流程的关键。
