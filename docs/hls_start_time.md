# HLS Start Time 功能文档

## 概述

`hls_start_time` 是一个新增的 HLS 选项，允许用户从指定时间点开始加载 HLS 视频，而不是传统的"先下载起始 segment、探测格式、再 seek 到目标位置"的方式。这是一个性能优化，特别适用于长视频场景。

## 用法

```bash
ffplay -hls_start_time 1800 "http://example.com/stream.m3u8"
```

上述命令会从 30 分钟处开始播放，跳过前 30 分钟的 segment 下载和处理。

## 涉及文件

| 文件                | 改动说明                                         |
| ------------------- | ------------------------------------------------ |
| `libavformat/hls.c` | 核心实现：AVOption 定义、segment 选择、seek 修复 |
| `fftools/ffplay.c`  | 入口：新增 `-hls_start_time` 命令行选项          |

## 改动详解

### 1. AVOption 定义 (`libavformat/hls.c`)

```c
int64_t start_time;  // HLSContext 结构体字段

{ "start_time", "start time offset in microseconds",
    OFFSET(start_time), AV_OPT_TYPE_INT64,
    {.i64 = AV_NOPTS_VALUE}, INT64_MIN, INT64_MAX, FLAGS },
```

- 单位：微秒（AV_TIME_BASE）
- 默认值：`AV_NOPTS_VALUE`（未设置）
- 通过 ffplay 的 `format_opts` 注入为 `"start_time"` 键

### 2. ffplay 入口 (`fftools/ffplay.c`)

新增变量和选项定义：

```c
static int64_t hls_start_time = AV_NOPTS_VALUE;

{ "hls_start_time", OPT_TYPE_TIME, 0, { &hls_start_time },
  "set HLS start time offset in seconds", "pos" },
```

在 `avformat_open_input` 前注入到 `format_opts`，之后清除：

```c
if (hls_start_time != AV_NOPTS_VALUE) {
    av_dict_set_int(&format_opts, "start_time", hls_start_time, 0);
}
// ... avformat_open_input ...
if (hls_start_time != AV_NOPTS_VALUE)
    av_dict_set(&format_opts, "start_time", NULL, AV_DICT_MATCH_CASE);
```

### 3. Segment 选择 (`select_cur_seq_no`)

```c
if (c->start_time != AV_NOPTS_VALUE) {
    find_timestamp_in_playlist(c, pls, c->start_time, &seq_no, NULL);
    return seq_no;
}
```

当 `start_time` 有效时，通过 `find_timestamp_in_playlist()` 二分查找定位到包含目标时间的 segment，直接从该 segment 开始播放，而非从 segment 0 开始。

### 4. hls_read_header 初始化

#### 4.1 从目标 segment 打开

原代码使用 `pls->segments[0]` 进行格式探测和 sub-demuxer 打开，改为 `current_segment(pls)`，确保打开的是 `select_cur_seq_no` 选定的起始 segment。

#### 4.2 初始化 seek 状态

```c
if (c->start_time != AV_NOPTS_VALUE) {
    c->cur_timestamp = c->start_time;
    c->first_timestamp = 0;
    for (i = 0; i < c->n_playlists; i++) {
        struct playlist *pls = c->playlists[i];
        pls->seek_timestamp = c->start_time;
        pls->seek_flags = AVSEEK_FLAG_ANY;
        pls->seek_stream_index = -1;
    }
    s->start_time = 0;
}
```

**各字段作用：**

- `c->cur_timestamp = c->start_time`：设置当前播放位置，供 `recheck_discard_flags()` 中"catch up"逻辑使用
- `c->first_timestamp = 0`：强制 segment 搜索的基准时间为 0。原代码设为 `AV_NOPTS_VALUE`，会在第一个包到达时被设为该包的 DTS（约 1800s），导致 `find_timestamp_in_playlist` 的搜索起点偏高
- `pls->seek_timestamp = c->start_time`：设置包过滤阈值，丢弃 start_time 之前的包
- `s->start_time = 0`：覆盖格式级 start_time，防止上层（ffplay）将 seek 目标 clamp 到第一个包的 DTS

### 5. hls_read_packet 中的 start_time 覆盖

```c
if (c->start_time != AV_NOPTS_VALUE) {
    for (int si = 0; si < pls->n_main_streams; si++)
        pls->main_streams[si]->start_time = 0;
}
```

**原理：** 每个包输出时强制 `st->start_time = 0`。

**解决的问题：** 不设置时，`demux.c` 的 `update_stream_timings()` 会用第一个包的 DTS（约 1800s）设置 `st->start_time`，进而 `ic->start_time = 1800s`。ffplay 的方向键 seek 处理中有 clamp 逻辑：

```c
// ffplay.c:3552
if (pos < ic->start_time / AV_TIME_BASE)
    pos = ic->start_time / AV_TIME_BASE;
```

这会导致无法 seek 到 start_time 之前的位置。强制 `st->start_time = 0` 使 `ic->start_time = 0`，解除 clamp 限制。

**时间戳保持绝对值：** 包的 PTS/DTS 不做偏移，保持原始绝对时间（如 1800s），这样时间轴显示和 seek 定位都使用绝对时间，逻辑一致。

### 6. hls_read_seek 中的 PTS wrap 修复

这是最关键的修复，解决"多次前向 seek 后 seek 完全失效"的问题。

#### 6.1 问题现象

```
hls_read_seek: BOUNDARY FAIL duration=5491000000 < seek_ts(97173716977) - first_ts(0)
```

seek 目标变成 97173 秒（约 27 小时），而视频只有 5491 秒（约 91 分钟）。

#### 6.2 根因：PTS wraparound 误检测

mpegts 使用 33 位 PTS/DTS（90kHz 时钟），最大值 2^33 = 8589934592（约 95444 秒）。FFmpeg 的 `update_wrap_reference()`（`demux.c:477`）会在第一个包到达时设置 wrap 参考点：

```c
pts_wrap_reference = first_dts - 60_seconds;
pts_wrap_behavior  = AV_PTS_WRAP_ADD_OFFSET;
```

之后每个包经过 `wrap_timestamp()`（`demux.c:53`）：

```c
if (timestamp < pts_wrap_reference)
    return timestamp + (1ULL << 33);  // 加 8589934592
```

**无 hls_start_time 时：** 从 segment 0 开始，第一个包 DTS ≈ 0，`pts_wrap_reference ≈ -5400000`（负数），正常 DTS 永远不会小于这个负数，不会触发假 wraparound。

**有 hls_start_time 时：** 从 ~1800s 的 segment 开始，第一个包 DTS ≈ 162000000（1800s × 90000），`pts_wrap_reference ≈ 156600000`（1740s）。当 seek 到一个 DTS 低于 1740s 的 segment 时，`wrap_timestamp()` 错误地给 DTS 加上 2^33，导致 DTS 跳变到 ~97173s。

#### 6.3 修复

在 `hls_read_seek()` 中，`ff_read_frame_flush()` 之后重置 `pts_wrap_reference`，使其在下一个包到达时重新计算：

**Sub-demuxer 层（mpegts streams）：**

```c
if (pls->ctx) {
    ff_read_frame_flush(pls->ctx);
    if (c->start_time != AV_NOPTS_VALUE) {
        for (unsigned k = 0; k < pls->ctx->nb_streams; k++) {
            FFStream *const sti = ffstream(pls->ctx->streams[k]);
            sti->pts_wrap_reference = AV_NOPTS_VALUE;
            sti->pts_wrap_behavior  = AV_PTS_WRAP_IGNORE;
        }
    }
}
```

**HLS 层（main streams）：**

```c
if (c->start_time != AV_NOPTS_VALUE) {
    for (i = 0; i < s->nb_streams; i++) {
        FFStream *const sti = ffstream(s->streams[i]);
        sti->pts_wrap_reference = AV_NOPTS_VALUE;
        sti->pts_wrap_behavior  = AV_PTS_WRAP_IGNORE;
    }
}
```

**为什么需要两层重置：**

- `set_stream_info_from_input_stream()`（`hls.c:2066`）将 mpegts 的 `pts_wrap_bits=33` 复制到 HLS 层 stream
- `update_wrap_reference()` 在 `pts_wrap_reference != AV_NOPTS_VALUE` 时直接返回，不会重算
- sub-demuxer 层防止 mpegts 的 `wrap_timestamp()` 破坏 DTS
- HLS 层防止 `read_frame_internal()` → `update_timestamps()` → `wrap_timestamp()` 再次破坏

**性能考虑：** 两处重置都包裹在 `if (c->start_time != AV_NOPTS_VALUE)` 中，无 hls_start_time 时零开销。

## 数据流总结

```
ffplay -hls_start_time 1800
  │
  ├─ format_opts["start_time"] = 1800000000 (µs)
  │
  ├─ avformat_open_input → hls_read_header
  │    ├─ parse_playlist → 获取所有 segment
  │    ├─ select_cur_seq_no → find_timestamp_in_playlist(1800s) → 定位到 segment N
  │    ├─ 打开 segment N（而非 segment 0）进行格式探测
  │    ├─ c->first_timestamp = 0
  │    ├─ c->cur_timestamp = 1800s
  │    ├─ pls->seek_timestamp = 1800s（过滤 1800s 之前的包）
  │    └─ s->start_time = 0
  │
  ├─ hls_read_packet
  │    ├─ 每个 包输出时强制 st->start_time = 0
  │    └─ c->cur_timestamp 更新为包的 DTS（绝对时间）
  │
  └─ hls_read_seek（用户 seek 时）
       ├─ ff_read_frame_flush(pls->ctx)  ← 不重置 pts_wrap_reference
       ├─ 重置 sub-demuxer 的 pts_wrap_reference（仅 start_time 有效时）
       ├─ 重置 HLS 层的 pts_wrap_reference（仅 start_time 有效时）
       └─ c->cur_timestamp = seek_timestamp
```

## 为什么不影响无 hls_start_time 的场景

所有新增代码都由 `c->start_time != AV_NOPTS_VALUE` 条件守护：

| 改动点                           | 条件                              | 无 start_time 时行为 |
| -------------------------------- | --------------------------------- | -------------------- |
| select_cur_seq_no                | `c->start_time != AV_NOPTS_VALUE` | 走原有逻辑           |
| hls_read_header 初始化           | `c->start_time != AV_NOPTS_VALUE` | 不执行               |
| hls_read_packet st->start_time=0 | `c->start_time != AV_NOPTS_VALUE` | 不执行               |
| hls_read_seek sub-demuxer 重置   | `c->start_time != AV_NOPTS_VALUE` | 不执行               |
| hls_read_seek HLS 层重置         | `c->start_time != AV_NOPTS_VALUE` | 不执行               |
