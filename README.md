# mod_taering_stream

`mod_taering_stream` 是一个面向 FreeSWITCH 的实时双向音频推流模块，提供 `taering_stream` API，可将通话音频通过 `WebSocket` 推送到后端，并将后端返回的音频实时注入到 FreeSWITCH 通话链路中。

该模块适合语音机器人、实时 ASR、TTS 回放、全双工对话、打断控制等场景。

## 联系方式

- [商用授权请与我联系](mailto:taering@vip.qq.com)
- 个人主页：https://www.liaoxinghui.com


## 项目定位

- FreeSWITCH 双向实时音频推流
- WebSocket 实时通信，支持 `ws://` / `wss://`
- 后端可接入 ASR / LLM / TTS / 语音中台
- 支持低延迟 PCM 回推
- 支持机器人播放过程中的打断控制

## 核心能力

- 支持 `FreeSWITCH -> WebSocket` 实时上行推流
- 支持 `WebSocket -> FreeSWITCH` 实时下行回放
- 支持 `mono`、`mixed`、`stereo` 三种音频组织模式
- 支持二进制 `PCM16LE` 主链路回推
- 支持文本 JSON + `audio_b64` 兼容回推
- 支持 `pause` / `resume`
- 支持 `pause_stream` / `resume_stream`
- 支持 `pause_play` / `resume_play`
- 支持 `interrupt_play` 打断当前播放
- 支持播放状态事件通知与播放进度回报
- 支持队列限流、断线重连、运行指标导出
- 支持 XML 默认配置与命令行覆盖

## 典型架构

```text
FreeSWITCH
   |
   | taering_stream start
   v
mod_taering_stream
   |
   | WebSocket 双向通信
   v
业务后端 / AI 中台
   |
   | ASR / LLM / TTS / 打断判断
   v
返回 PCM 音频或控制消息
```

推荐主路径：

- 上行：FreeSWITCH 实时发送二进制 PCM 音频帧到后端
- 下行：后端将 TTS 结果转为单声道 `PCM16LE`，按小块二进制帧实时回推
- 打断：后端判断用户插话后，主动调用 `interrupt_play`




### 安装验证

```bash
fs_cli -x "load mod_taering_stream"
fs_cli -x "show api" | grep taering_stream
```

## 授权说明

模块支持授权部署模式，实际交付时通常包含以下文件：

- `mod_taering_stream.so`
- `mod_taering_stream.la`
- `mod_taering_stream.conf.xml`
- `mod_taering_stream.data`

如果你需要商用授权或定制部署支持，请通过下方联系方式联系我。

## API 用法

### 命令列表

```text
taering_stream version
taering_stream license_info
taering_stream stats
taering_stream <uuid> start [ws://|wss://url|-] [mono|mixed|stereo|-] [8k|16k|24k|48k|-] [metadata_json]
taering_stream <uuid> stop [metadata_json]
taering_stream <uuid> pause
taering_stream <uuid> resume
taering_stream <uuid> pause_stream
taering_stream <uuid> resume_stream
taering_stream <uuid> pause_play
taering_stream <uuid> resume_play
taering_stream <uuid> interrupt_play
taering_stream <uuid> send_text <json_or_text>
```

### 常用示例

全部参数显式指定：

```bash
fs_cli -x "taering_stream <uuid> start ws://127.0.0.1:9000/stream mono 16k"
```

仅传 UUID，其余走 XML 默认值：

```bash
fs_cli -x "taering_stream <uuid> start"
```

仅覆盖 URL：

```bash
fs_cli -x "taering_stream <uuid> start ws://127.0.0.1:9000/stream - -"
```

暂停 / 恢复：

```bash
fs_cli -x "taering_stream <uuid> pause"
fs_cli -x "taering_stream <uuid> resume"
```

仅暂停上行：

```bash
fs_cli -x "taering_stream <uuid> pause_stream"
fs_cli -x "taering_stream <uuid> resume_stream"
```

仅暂停下行：

```bash
fs_cli -x "taering_stream <uuid> pause_play"
fs_cli -x "taering_stream <uuid> resume_play"
```

打断当前播放：

```bash
fs_cli -x "taering_stream <uuid> interrupt_play"
```

## 后端协议说明

### 连接建立后的时序

1. FreeSWITCH 作为 WebSocket Client 连接后端地址。
2. 连接成功后，先发送一条文本 JSON 控制消息 `type=start`。
3. 之后持续发送二进制 PCM 音频帧。
4. 后端可随时回发二进制 PCM 音频帧给 FreeSWITCH。
5. 通话结束或主动停止时，模块发送 `type=stop`。

### FreeSWITCH -> 后端

首包控制消息示例：

```json
{"type":"start","uuid":"<uuid>","sample_rate":16000,"mix_type":"mono","metadata":{}}
```

停止消息示例：

```json
{"type":"stop","uuid":"<uuid>","sample_rate":16000,"mix_type":"mono"}
```

字段说明：

- `uuid`：当前通话 UUID
- `sample_rate`：当前会话采样率
- `mix_type`：音频组织模式，支持 `mono` / `mixed` / `stereo`
- `metadata`：启动时附带的业务参数

后续二进制帧约定：

- 编码：`PCM16LE`
- 字节序：小端
- 封装：裸 PCM，不带 WAV Header
- 采样率：以 `start.sample_rate` 为准
- 分片：通常按 `packet_ms` 配置切块

`mix_type` 说明：

- `mono`：单声道，适合实时 ASR
- `mixed`：双方混音后的单声道
- `stereo`：双声道交织音频，左声道为 `read`，右声道为 `write`

### 后端 -> FreeSWITCH

推荐生产主路径：直接回推 WebSocket 二进制 PCM。

要求如下：

- 编码：`PCM16LE`
- 单声道
- 裸 PCM
- 采样率尽量与 `start.sample_rate` 一致
- 建议按 `20ms` 或 `40ms` 小块流式返回

如果后端拿到的是其他 TTS 格式，推荐流程如下：

```text
TTS 输出 -> 后端解码/转码 -> PCM16LE -> WebSocket Binary -> FreeSWITCH
```

兼容路径也支持文本 JSON + `audio_b64`：

```json
{"uuid":"<uuid>","format":"raw","audio_b64":"..."}
```

或：

```json
{"uuid":"<uuid>","format":"wav","audio_b64":"..."}
{"uuid":"<uuid>","format":"mp3","audio_b64":"..."}
{"uuid":"<uuid>","format":"ogg","audio_b64":"..."}
```

说明：

- `raw` 表示解码后为裸 `PCM16LE`
- `wav/mp3/ogg` 属于兼容文件播放路径
- 高并发、低延迟场景不建议长期依赖 `audio_b64`

## 打断能力说明

模块支持在机器人播放过程中执行 `interrupt_play`，用于实现用户插话打断。

建议配置：

```xml
<param name="default_mix_type" value="stereo"/>
<param name="tx_buffer_ms" value="80"/>
<param name="rx_buffer_ms" value="80"/>
<param name="enable_barge_in" value="true"/>
```

说明：

- `stereo` 便于后端识别双方音频
- `80ms` 缓冲有助于降低打断尾音
- `enable_barge_in=true` 表示允许模块执行打断

需要注意：

- 模块只负责“允许打断”和“执行打断”
- 真正何时打断，仍然由后端判断
- 后端检测到用户抢话后，应主动发送 `interrupt_play`

推荐打断流程：

1. 后端回推 TTS PCM。
2. 后端进入播放保护期。
3. 在保护期内检测用户是否插话。
4. 一旦确认插话，调用 `taering_stream <uuid> interrupt_play`。
5. 模块清空当前 PCM 播放队列。
6. 模块通过 `taering_stream::play` 上报 `interrupted` 状态。
7. 后端退出保护期，继续处理用户语音。

## 事件通知

模块会发出以下事件：

- `taering_stream::connect`
- `taering_stream::disconnect`
- `taering_stream::json`
- `taering_stream::error`
- `taering_stream::play`

播放事件可用于回报：

- `queued`
- `start`
- `done`
- `interrupted`
- `error_*`

同时支持输出以下播放进度字段：

- `Play-State`
- `Played-Ms`
- `Played-Samples`
- `Playback-Generation`

## XML 配置项

配置文件：

```text
autoload_configs/mod_taering_stream.conf.xml
```

主要参数包括：

- `default_ws_url`
- `default_ws_urls`
- `default_mix_type`
- `default_sample_rate`
- `default_autoplay`
- `tx_queue_limit`
- `rx_queue_limit`
- `play_queue_limit`
- `packet_ms`
- `tx_buffer_ms`
- `rx_buffer_ms`
- `network_workers`
- `allow_text_audio`
- `enable_file_playback`
- `temp_dir`
- `enable_barge_in`
- `reconnect_base_ms`
- `reconnect_max_ms`
- `reconnect_max_attempts`

配置生效方式：

```bash
fs_cli -x "reloadxml"
fs_cli -x "reload mod_taering_stream"
```

## 运行指标

```bash
fs_cli -x "taering_stream stats"
```

常见输出包括：

- `active_sessions`
- `worker[i].sessions`
- `worker[i].tx_fill`
- `worker[i].rx_fill`
- `worker[i].tx_drop`
- `worker[i].rx_drop`
- `worker[i].reconnects`

## 常见问题

`load mod_taering_stream` 失败：

- 检查模块文件是否已安装到 FreeSWITCH 模块目录
- 检查依赖库是否完整

`show api` 看不到 `taering_stream`：

- 说明模块未成功加载，应查看 FreeSWITCH 控制台日志

启动后无下行回放：

- 检查后端返回的 `uuid` 是否与当前通话一致
- 检查回推音频格式是否正确
- 检查是否处于 `pause_play` 状态

打断无效：

- 检查 `enable_barge_in` 是否开启
- 检查后端是否真正发送了 `interrupt_play`
- 检查缓冲配置是否过大导致尾音明显


