# FFmpeg 深入学习路线图

## 前言

FFmpeg 是多媒体处理领域的核心框架，学习它需要对整个多媒体处理流程有清晰的认识。本路线图按照由浅入深、由全局到细节的顺序组织，建议按阶段逐步学习。

---

## 第一阶段：基础概念 (1-2周)

### 1.1 多媒体基础理论

在开始阅读 FFmpeg 代码之前，需要掌握以下基础知识：

**必须掌握：**
- [ ] 音视频编码原理（帧类型、I/P/B 帧、GOP）
- [ ] 常见音视频编码格式（H.264/H.265/AV1/AAC/MP3/Opus）
- [ ] 容器格式概念（MP4/MKV/TS/WebM）
- [ ] PTS/DTS 时间戳概念
- [ ] 像素格式（YUV/RGB）和采样格式
- [ ] 码率控制策略（CBR/VBR/CRF）

**参考资料：**
- 《数字音视频技术》- 了解编解码基础
- FFmpeg 官方文档：https://ffmpeg.org/documentation.html
- FFmpeg Wiki：https://trac.ffmpeg.org/wiki

### 1.2 FFmpeg 整体架构

阅读 `module.md`（已在上一任务中生成），理解：
- 7个核心库的作用和关系
- Demuxer → Decoder → Filter → Encoder → Muxer 的数据流
- fftools 命令行工具的角色

**重点理解：**
```
输入文件 → libavformat (解复用) → AVPacket
                                    ↓
                           libavcodec (解码) → AVFrame
                                    ↓
                           libavfilter (滤镜处理) → AVFrame
                                    ↓
                           libavcodec (编码) → AVPacket
                                    ↓
                           libavformat (复用) → 输出文件
```

---

## 第二阶段：核心数据结构 (1周)

### 2.1 必须掌握的核心结构

这些是 FFmpeg 整个框架的基石，必须深入理解：

**AVFrame（未压缩数据帧）**
```
位置：libavutil/frame.h
```
- `data[]` - 数据平面指针
- `linesize[]` - 每行字节数
- `width/height` - 视频尺寸
- `nb_samples` - 音频样本数
- `format` - 像素/采样格式
- `pts/dts` - 时间戳
- `buf[]` - 引用计数缓冲区

**AVPacket（压缩数据包）**
```
位置：libavcodec/packet.h
```
- `data/size` - 数据指针和大小
- `pts/dts` - 时间戳
- `flags` - 关键帧标志
- `side_data` - 附加数据

**AVCodecContext（编解码器上下文）**
```
位置：libavcodec/avcodec.h (第439行起)
```
- 编解码器配置和状态
- 视频/音频参数
- 时间基设置

**AVFormatContext（封装上下文）**
```
位置：libavformat/avformat.h
```
- 输入/输出格式信息
- 流管理
- I/O 上下文

### 2.2 学习建议

**必读代码文件：**
```bash
# 按顺序阅读以下文件
libavutil/frame.h           # AVFrame 定义
libavcodec/packet.h         # AVPacket 定义
libavcodec/avcodec.h        # AVCodecContext 定义 (重点看第439行开始)
libavutil/rational.h         # AVRational 时间基
libavutil/pixfmt.h          # 像素格式枚举
libavutil/samplefmt.h        # 采样格式枚举
```

---

## 第三阶段：libavutil 公共工具库 (3-5天)

### 3.1 核心功能模块

libavutil 是所有库的基础，学习时重点关注：

**内存管理 (mem.h)**
```c
位置：libavutil/mem.h, mem.c
```
- `av_malloc()` / `av_realloc()` / `av_free()`
- `av_mallocz()` - 分配并清零
- `av_dynmalloc()` - 动态分配
- 引用计数机制 `av_buffer_ref()` / `av_buffer_unref()`

**缓冲区管理 (buffer.h)**
```c
位置：libavutil/buffer.c
```
- `AVBuffer` / `AVBufferRef` 引用计数结构
- 缓冲区池管理

**选项系统 (opt.h)**
```c
位置：libavutil/opt.c
```
- `av_opt_set()` / `av_opt_get()`
- `av_opt_find()` / `av_opt_next()`
- FFmpeg 统一配置系统的基础

### 3.2 重点文件

```bash
libavutil/mem.c          # 内存管理实现
libavutil/buffer.c      # 缓冲区引用计数
libavutil/opt.c          # 选项系统实现
libavutil/log.c          # 日志系统
libavutil/error.c        # 错误处理
```

---

## 第四阶段：libavcodec 编解码器库 (1-2周) ⭐⭐⭐

**这是 FFmpeg 最核心的库，是学习的重点！**

### 4.1 核心架构

```
libavcodec/
├── avcodec.h           # 公共 API
├── codec.h             # AVCodec 结构定义
├── codec_internal.h     # FFCodec 内部结构
├── avcodec_internal.h   # 内部 API
├── decode.c            # 解码核心 (★★★★★)
├── encode.c            # 编码核心 (★★★★★)
├── utils.c             # 工具函数
├── allcodecs.c         # 编解码器注册
├── packet.c            # Packet 处理
└── [codec]/            # 各编解码器实现
```

### 4.2 Send/Receive API 模式 ⭐⭐⭐⭐⭐

**这是 libavcodec 最核心的 API 模式，必须彻底理解！**

```c
// 解码流程
avcodec_open2()                    // 打开解码器
avcodec_send_packet()              // 发送压缩数据
avcodec_receive_frame()             // 接收解码帧
avcodec_flush_buffers()             // 刷新（seek 时）

// 编码流程
avcodec_open2()                    // 打开编码器
avcodec_send_frame()               // 发送原始帧
avcodec_receive_packet()           // 接收编码包
avcodec_flush_buffers()             // 刷新
```

**必须阅读的代码：**
```
libavcodec/decode.c  # 重点！理解 send_packet/receive_frame 机制
libavcodec/encode.c  # 重点！理解 send_frame/receive_packet 机制
libavcodec/avcodec.h # 第92-179行：API 使用说明（注释非常详细）
```

### 4.3 解码器实现流程

以一个简单解码器为例，理解实现模式：

```c
// 参考：libavcodec/utils.c 中的通用逻辑

// 1. 打开解码器
avcodec_open2():
    → ff_decode_preinit()      // 预初始化
    → codec->init()            // 调用解码器自身的 init

// 2. 发送数据包
avcodec_send_packet():
    → extract_packet_props()    // 提取包属性
    → decode_bsfs_init()       // 初始化比特流滤镜
    → codec->send_packet()     // 或 receive_frame()

// 3. 接收帧
avcodec_receive_frame():
    → ff_decode_receive_frame()
    → codec->receive_frame()   // 或旧的 decode() 回调
```

### 4.4 FFCodec 结构 - 编解码器的核心

```c
// libavcodec/codec_internal.h 第127行起

typedef struct FFCodec {
    AVCodec p;                     // 公共结构

    unsigned is_decoder:1;         // 是解码器还是编码器

    int priv_data_size;           // 私有数据大小

    // 核心回调函数
    union {
        int (*decode)(...);       // 旧式回调
        int (*receive_frame)(...); // 新式回调（推荐）
    } cb;

    int (*init)(...);             // 初始化
    int (*close)(...);            // 关闭
    void (*flush)(...);           // 刷新
} FFCodec;
```

### 4.5 多线程解码

```
位置：libavcodec/thread.c
```
- 帧级多线程：多个线程同时解码不同帧
- Slice 级多线程：一帧分割多个 slice 并行处理
- `ff_thread_init()` / `ff_thread_free()`
- `ff_thread_receive_frame()` - 线程同步获取帧

### 4.6 硬件加速

```
硬件加速实现位置：
libavcodec/vaapi/     - Linux VAAPI
libavcodec/vdpau/     - NVIDIA VDPAU
libavcodec/cuvid/     - NVIDIA CUVID
libavcodec/qsv/       - Intel QSV
libavcodec/videotoolbox.c - macOS
libavcodec/d3d11va/   - Windows D3D11
```

**学习重点：**
```c
// libavcodec/hwaccel_internal.h
// 理解硬件上下文链：
AVCodecContext.hw_device_ctx → AVHWDeviceContext → AVHWFramesContext
```

### 4.7 比特流滤镜 (BSF)

```
位置：libavcodec/bsf.c
```
- 解码前处理码流格式转换
- 内置 BSF：`h264_mp4toannexb`, `hevc_mp4toannexb`, `aac_adtstoasc`

---

## 第五阶段：libavformat 容器封装库 (3-5天)

### 5.1 核心功能

```
libavformat/
├── avformat.h        # 主 API
├── avformat.c        # 格式上下文管理
├── demux.c           # 解复用核心 (★★★★★)
├── mux.c             # 复用核心 (★★★★★)
├── avio.h/avio.c     # I/O 抽象层 (重要！)
├── protocols.c       # 协议处理
└── [format]/         # 各容器格式实现
```

### 5.2 解复用流程 ⭐⭐⭐⭐⭐

```c
// 解复用流程
avformat_open_input()           // 打开输入
avformat_find_stream_info()    // 探测流信息
av_read_frame()                // 读取数据包
av_seek_frame()                // 跳转
avformat_close_input()         // 关闭

// 复用流程
avformat_alloc_output_context2() // 创建输出上下文
avformat_new_stream()           // 创建流
avio_open2()                    // 打开输出
avformat_write_header()         // 写头
av_interleaved_write_frame()   // 写帧
av_write_trailer()              // 写尾
```

### 5.3 I/O 抽象层 (AVIOContext) ⭐⭐⭐

```
位置：libavformat/avio.h, avio.c
```

这是 FFmpeg I/O 的核心，理解它就能理解 FFmpeg 如何支持各种输入源：

```c
typedef struct AVIOContext {
    unsigned char *buffer;      // 缓冲区
    int buffer_size;            // 缓冲区大小
    unsigned char *buf_ptr;     // 当前指针
    unsigned char *buf_end;     // 缓冲区结束

    // 回调函数
    int (*read_packet)(void *opaque, uint8_t *buf, int buf_size);
    int (*write_packet)(void *opaque, uint8_t *buf, int buf_size);
    int64_t (*seek)(void *opaque, int64_t pos, int whence);

    void *opaque;               // 用户数据
} AVIOContext;
```

### 5.4 协议层

FFmpeg 通过协议抽象支持各种输入输出：

```
位置：libavformat/protocols.c
```
- `file://` - 本地文件
- `http://` / `https://` - HTTP
- `rtmp://` - RTMP 流
- `udp://` - UDP 流
- `pipe://` - 管道

---

## 第六阶段：libavfilter 滤镜库 (3-5天)

### 6.1 滤镜图架构

```
位置：libavfilter/
├── avfilter.h         # 主 API
├── graph.c            # 滤镜图管理 (★★★★★)
├── buffersrc.c        # 视频源滤镜
├── buffersink.c       # 视频汇滤镜
├── abuffersrc.c       # 音频源滤镜
├── abuffersink.c      # 音频汇滤镜
└── [filters]/         # 各滤镜实现
```

### 6.2 核心概念

**AVFilter** - 滤镜定义
```c
typedef struct AVFilter {
    const char *name;           // 滤镜名称
    const AVFilterPad *inputs;  // 输入垫
    const AVFilterPad *outputs; // 输出垫
} AVFilter;
```

**AVFilterGraph** - 滤镜图
```c
typedef struct AVFilterGraph {
    unsigned int nb_filters;     // 滤镜数量
    AVFilterContext **filters;  // 滤镜数组
} AVFilterGraph;
```

### 6.3 滤镜使用流程 ⭐⭐⭐

```c
// 创建滤镜图
avfilter_graph_alloc()

// 解析滤镜链字符串
avfilter_graph_parse2("video=scale=1920:1080[v];[v]fps=30[s]", ...)

// 配置
avfilter_graph_config()

// 发送帧
av_buffersrc_add_frame(src_ctx, frame)

// 获取帧
av_buffersink_get_frame(sink_ctx, frame)

// 释放
avfilter_graph_free()
```

---

## 第七阶段：fftools 命令行工具 (2-3天)

### 7.1 ffmpeg.c 架构

```
位置：fftools/ffmpeg.c
```
这是 FFmpeg 最复杂的文件之一，将所有组件整合在一起：

```c
// 核心文件
ffmpeg.c          # 主程序
ffmpeg.h          # 主头文件
ffmpeg_opt.c      # 选项解析
ffmpeg_demux.c    # 解复用逻辑
ffmpeg_mux.c      # 复用逻辑
ffmpeg_dec.c      # 解码逻辑
ffmpeg_enc.c      # 编码逻辑
ffmpeg_filter.c   # 滤镜逻辑
ffmpeg_hw.c       # 硬件加速
ffmpeg_sched.c    # 调度逻辑
```

### 7.2 处理流程

```c
// ffmpeg.c 中的主循环
main()
    → parse_options()           // 解析命令行
    → open_input_file()         // 打开输入
    → open_output_file()         // 打开输出
    → init_complex_filters()     // 初始化滤镜

    // 主循环
    while (1) {
        if (ifile->eof) {
            // 输入结束，进入排空模式
            drain_encoder();
        } else {
            // 读取、解码、编码、写输出
            process_input();
        }
    }
```

---

## 第八阶段：深入特定领域 (持续学习)

### 8.1 如果你想深入视频编解码

**推荐重点学习：**
```bash
libavcodec/h264/     # H.264 实现
libavcodec/hevc/     # H.265/HEVC 实现
libavcodec/av1/      # AV1 实现
libavcodec/vp9/      # VP9 实现
libavcodec/mpeg4/    # MPEG-4 Part 2 实现
```

**学习建议：**
1. 先看 `h264dec.c` / `hevcdec.c` 等顶层文件
2. 理解码流结构（NAL 单元、SPS/PPS/VCL）
3. 理解帧类型和参考机制

### 8.2 如果你想深入音频编解码

```bash
libavcodec/aac/       # AAC 实现 (很复杂！)
libavcodec/ac3/       # AC3 实现
libavcodec/mpegaudio.c # MP3 实现
libavcodec/opusenc.c / opusdec.c  # Opus
```

### 8.3 如果你想学习硬件加速

```bash
libavcodec/vaapi/     # VAAPI (Linux)
libavcodec/cuvid/     # CUVID (NVIDIA)
libavcodec/qsv/       # QSV (Intel)
libavcodec/videotoolbox.c  # VideoToolbox (macOS/iOS)
libavcodec/d3d11va/   # D3D11VA (Windows)
```

### 8.4 如果你想学习容器格式

```bash
libavformat/mov.c     # MP4/MOV 格式
libavformat/matroska.c # MKV 格式
libavformat/mpegts.c  # TS 格式
libavformat/rm.c      # RM 格式
libavformat/avi.c     # AVI 格式
```

---

## 第九阶段：实践项目建议

### 9.1 入门项目

**项目1：最小播放器**
```c
// 使用 libavformat + libavcodec 实现最简视频播放流程
// 目标：理解 Demux → Decode → Display 的基本流程
```

**项目2：转码工具**
```c
// 实现 MP4 转 TS 的简单转码器
// 目标：理解 Encode → Mux 的流程
```

**项目3：添加水印滤镜**
```c
// 使用 libavfilter 添加文字水印
// 目标：理解滤镜图构建和数据流
```

### 9.2 进阶项目

- 实现一个简单的 H.264 解码器
- 实现一个自定义的容器格式 muxer/demuxer
- 实现硬件加速的转码流程

---

## 推荐学习顺序总结

```
┌─────────────────────────────────────────────────────────────────┐
│  第1步：多媒体基础理论（必须先了解概念）                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  第2步：核心数据结构                                              │
│         AVFrame / AVPacket / AVCodecContext / AVFormatContext   │
│         位置：libavutil/frame.h, libavcodec/avcodec.h            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  第3步：libavcodec 核心 API (★★★★★最核心★★★★★)                    │
│         send_packet/receive_frame 模式                          │
│         位置：libavcodec/decode.c, encode.c                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  第4步：libavformat 解复用/复用                                   │
│         avformat_open_input / av_read_frame                      │
│         位置：libavformat/demux.c, mux.c                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  第5步：libavfilter 滤镜系统                                     │
│         位置：libavfilter/graph.c                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  第6步：fftools/ffmpeg.c 整合流程                                │
│         理解所有组件如何在一起工作                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  第7步：深入特定领域（根据兴趣选择）                               │
│         视频编解码 / 音频编解码 / 硬件加速 / 容器格式              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 重点代码文件清单

### 必须精通 (⭐⭐⭐⭐⭐)
```bash
libavcodec/decode.c      # 解码核心 API 实现
libavcodec/encode.c      # 编码核心 API 实现
libavformat/demux.c      # 解复用核心实现
libavformat/mux.c        # 复用核心实现
```

### 应该熟悉 (⭐⭐⭐⭐)
```bash
libavcodec/avcodec.h     # AVCodecContext 定义和 API 文档
libavformat/avformat.h   # AVFormatContext 定义
libavutil/frame.h        # AVFrame 定义
libavfilter/graph.c      # 滤镜图管理
fftools/ffmpeg.c         # 命令行工具核心
```

### 需要了解 (⭐⭐⭐)
```bash
libavcodec/utils.c       # 编解码器工具函数
libavcodec/options.c    # 选项管理
libavformat/avio.c       # I/O 抽象层
libavutil/opt.c          # 选项系统实现
```

---

## 辅助资源

### 官方文档
- FFmpeg 官网：https://ffmpeg.org
- FFmpeg 文档：https://ffmpeg.org/documentation.html
- FFmpeg Wiki：https://trac.ffmpeg.org/wiki

### 代码学习
- `doc/examples/` 目录包含大量 API 使用示例
- `doc/APIchanges` 记录 API 变更历史
- `Changelog` 了解版本变化

### 调试技巧
```bash
# 启用 debug 日志
ffmpeg -v debug -i input.mp4 output.mp4

# 查看编解码器信息
ffmpeg -codecs
ffmpeg -decoders
ffmpeg -encoders

# 导出 GOP 信息
ffmpeg -i input.mp4 -frame_to_time_code_output dump.txt
```

---

## 总结

FFmpeg 是一个庞大而复杂的系统，学习它需要耐心和系统的方法。记住以下关键点：

1. **从基础开始**：先理解多媒体概念，再看代码
2. **核心是编解码**：libavcodec 的 send/receive 模式是重中之重
3. **数据流是关键**：理解 AVPacket → AVFrame → AVPacket 的转换
4. **实践出真知**：一定要动手写代码，光看是不够的
5. **善用官方示例**：`doc/examples/` 是最好的学习资源

祝学习顺利！
