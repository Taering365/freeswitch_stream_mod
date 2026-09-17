# mod_taering_stream

# 🚀 FreeSWITCH AI Real-time Streaming Module

> Real-time bidirectional audio streaming module for FreeSWITCH  
> Built for AI voice systems (ASR + LLM + TTS) with low latency & barge-in support

---

## 🔥 Overview

This module enables **real-time audio streaming** from FreeSWITCH to backend services and supports **injecting PCM audio back into calls**.

It is designed for building modern **AI voice systems**, including:

- 🤖 AI outbound calling systems
- 🎙️ Real-time voice bots
- 📞 Intelligent IVR systems
- 🧠 ASR + LLM + TTS pipelines

---

## ⚡ Key Features

- 🎧 **Real-time RTP audio streaming → WebSocket**
- 🔊 **PCM audio injection (no file playback)**
- ⚡ **Low latency audio pipeline**
- 🔁 **Bidirectional audio (full-duplex)**
- 🧠 **AI integration ready (ASR / LLM / TTS)**
- ✋ **Barge-in support (interrupt TTS playback)**
- 📈 Designed for **high concurrency**

---

## 🏗️ Architecture

```text```
Caller
  ↓
FreeSWITCH
  ↓
mod_taering_stream
  ↓ WebSocket
Backend (ASR → LLM → TTS)
  ↓ PCM audio
FreeSWITCH playback injection

`mod_taering_stream` 是一个面向 FreeSWITCH 的实时双向音频推流模块，提供 `taering_stream` API，可将通话音频通过 `WebSocket` 推送到后端，并将后端返回的音频实时注入到 FreeSWITCH 通话链路中。

该模块适合语音机器人、实时 ASR、TTS 回放、全双工对话、打断控制等场景。




## 联系方式

- 个人邮箱: [taering@vip.qq.com](mailto:taering@vip.qq.com)
- 个人邮箱: [taering4417@gmail.com](mailto:taering4417@gmail.com)
- 个人主页: [https://www.liaoxinghui.com](https://www.liaoxinghui.com)
  




# mod_taering_stream 0.54 商业接口与集成手册

本文档面向购买、部署和集成 `mod_taering_stream` 的客户，说明模块当前版本 `0.54` 的公开调用方式、XML 配置、字段、返回格式、Python 示例、媒体后端协议、授权与运维功能。


阅读索引：第 4～6 节为控制接口，第 7 节为完整 Dialplan 示例，第 8～9 节为媒体协议与事件，第 11 节为全部 XML 参数，第 12～13 节为授权与排障，第 14 节为进程内模块集成。

## 1. 产品边界

`mod_taering_stream` 是独立的 FreeSWITCH 双向音频流模块，只负责：

- 在指定 FreeSWITCH channel 上挂载和卸载 media bug；
- 将 FreeSWITCH 音频以 WebSocket 二进制 PCM 推送到媒体后端；
- 接收媒体后端返回的 PCM 并注入 FreeSWITCH；
- 暂停、恢复、停止、清空播放缓冲；
- WebSocket 重连、缓冲控制、丢帧计数和媒体事件。

模块不负责外呼、入呼、振铃、应答、桥接、挂机原因或其他呼叫状态。客户可以独立使用本模块，也可以由 `mod_fcc`、ESL 程序或其他呼叫控制系统调用。

## 2. 调用方式选择

| 方式 | 使用场景 | 是否需要 ESL | 推荐程度 |
|---|---|---:|---:|
| HTTP REST | 普通 Java/Python/Go 后端远程调用 | 否 | 推荐 |
| `taering_stream_json` + ESL | 已有 ESL 长连接或本机控制服务 | 是 | 推荐 |
| 传统 `taering_stream` + ESL | 兼容旧系统 | 是 | 兼容 |
| `taering_stream_json` 进程内调用 | FreeSWITCH 模块通过 API 注册表调用 | 否 | 推荐 |
| Dialplan application | 在 XML dialplan 内自动启动或停止 | 否 | 推荐 |

HTTP 和 ESL 最终调用同一个媒体会话核心，行为一致。FCC 运行在同一个 FreeSWITCH 进程时，建议通过 `switch_api_execute("taering_stream_json", ...)` 调用，不需要 ESL，也不需要经过 HTTP。

## 3. 基础概念

### 3.1 标识字段

| 字段 | 类型 | 最大长度 | 必填 | 说明 |
|---|---|---:|---:|---|
| `stream_id` | string | 63 | 模块生成 | 媒体会话 ID，格式类似 `stream_<uuid>`，创建后控制媒体的首选主键 |
| `channel_id` | string | 36 | 创建时必填 | FreeSWITCH channel UUID，不是 SIP Call-ID |
| `external_id` | string | 127 | 否 | 调用方的不透明关联 ID；FCC 可放 `call_id`，模块不解释其含义 |
| `request_id` | string | 127 | 建议填写 | 活动媒体会话范围内的启动幂等键，不跨模块重载或 FreeSWITCH 重启持久化 |

当前版本一个 `channel_id` 同时只允许一个活动媒体流。

### 3.2 媒体状态

| 状态 | 说明 |
|---|---|
| `connecting` | media bug 已挂载，正在连接媒体后端 |
| `streaming` | WebSocket 已连接，正在传输媒体 |
| `reconnecting` | 后端连接中断，正在按配置退避重连 |
| `stopping` | 已接受停止请求，正在释放连接、队列和 media bug |

这些只是媒体状态，不代表电话是否振铃、应答或挂机。

### 3.3 通用成功响应

查询或创建成功时：

```json
{
  "ok": true,
  "stream_id": "stream_550e8400-e29b-41d4-a716-446655440000",
  "channel_id": "8bb70000-1111-2222-3333-444455556666",
  "external_id": "call_01K...",
  "request_id": "media-20260827-001",
  "state": "connecting",
  "connected": false,
  "capture_paused": false,
  "playback_paused": false,
  "tx_drop_frames": 0,
  "rx_drop_frames": 0,
  "idempotent": false
}
```

| 字段 | 类型 | 说明 |
|---|---|---|
| `ok` | boolean | 是否成功 |
| `state` | string | 当前媒体状态 |
| `connected` | boolean | 媒体 WebSocket 当前是否连接 |
| `capture_paused` | boolean | FreeSWITCH → 后端上行采集是否暂停 |
| `playback_paused` | boolean | 后端 → FreeSWITCH 下行注入是否暂停 |
| `tx_drop_frames` | integer | 上行 ring 因容量不足丢弃的帧数 |
| `rx_drop_frames` | integer | 下行 ring 因容量不足丢弃的帧数 |
| `idempotent` | boolean | 是否由重复的活动 `request_id` 返回原会话 |

动作成功时：

```json
{"ok":true,"stream_id":"stream_...","action":"pause_capture","accepted":true}
```

### 3.4 通用失败响应

```json
{"ok":false,"code":"STREAM_NOT_FOUND","message":"stream_id or channel_id does not match an active stream"}
```

| 错误码 | 常见 HTTP 状态 | 说明 |
|---|---:|---|
| `UNAUTHORIZED` | 401 | HTTP Bearer Token 缺失或错误 |
| `INVALID_REQUEST` | 400 | 缺少 `action`、`channel_id` 或动作所需字段 |
| `BACKEND_NOT_CONFIGURED` | 400 | 请求和 XML 均未提供媒体后端 URL |
| `START_FAILED` | 400 | channel 不存在、media bug 挂载失败或授权/容量拒绝 |
| `STREAM_ALREADY_EXISTS` | 409 | channel 已有活动媒体流 |
| `IDEMPOTENCY_CONFLICT` | 409 | 同一活动 `request_id` 被用于另一个 channel |
| `STREAM_NOT_FOUND` | 404 | 找不到指定活动媒体流 |
| `UNKNOWN_ACTION` | 400 | action 不受支持 |
| `ACTION_FAILED` | 400 | 动作被拒绝，例如未启用 `enable_barge_in` 时执行打断 |
| `NOT_FOUND` | 404 | HTTP 路由不存在或请求体不是 JSON object |
| `INTERNAL_ERROR` | 500/400 | 内部创建或返回异常 |

## 4. HTTP REST API

### 4.1 HTTP 配置与鉴权

HTTP 默认关闭。配置文件：`conf/autoload_configs/mod_taering_stream.conf.xml`。

```xml
<param name="api_enabled" value="true"/>
<param name="api_listen_host" value="127.0.0.1"/>
<param name="api_listen_port" value="18081"/>
<param name="api_token" value="请替换为高强度随机Token"/>
```

HTTP 请求体上限为 65536 字节，超过上限当前实现会关闭连接，不保证返回 JSON 错误体；此限制独立于 WebSocket 的 `max_text_payload`。健康检查仅表示 HTTP 控制接口可用，不检查授权是否通过或媒体后端是否连通。示例中的 `build_id` 随构建变化。

所有 HTTP 请求，包括健康检查，都必须携带：

```http
Authorization: Bearer <api_token>
```

若 `api_enabled=true` 但 `api_token` 为空，模块拒绝加载。模块当前提供明文 HTTP，不直接提供 HTTPS。远程访问时必须置于受信任 TLS 反向代理和防火墙后面。

下面所有 Python 示例均使用：

```bash
pip install requests
```

公共初始化代码：

```python
import requests

BASE_URL = "http://127.0.0.1:18081/api/v1"
TOKEN = "replace-with-a-strong-random-token"
HEADERS = {
    "Authorization": f"Bearer {TOKEN}",
    "Content-Type": "application/json",
}
```

### 4.2 健康检查

```http
GET /api/v1/health
```

请求参数：无。

成功响应，HTTP 200：

```json
{"ok":true,"module":"taering_stream","version":"0.54","build_id":"commercial-generic-20260828"}
```

Python 示例：

```python
response = requests.get(f"{BASE_URL}/health", headers=HEADERS, timeout=3)
response.raise_for_status()
print(response.json())
```

### 4.3 创建媒体流

```http
POST /api/v1/streams
Content-Type: application/json
```

请求体：

```json
{
  "channel_id": "8bb70000-1111-2222-3333-444455556666",
  "external_id": "call_01K...",
  "request_id": "media-20260827-001",
  "ws_url": "ws://127.0.0.1:9000/stream",
  "mix_type": "stereo",
  "sample_rate": 16000
}
```

| 参数 | 类型 | 必填 | 默认值 | 说明 |
|---|---|---:|---|---|
| `channel_id` | string | 是 | 无 | 已存在的 FreeSWITCH channel UUID |
| `external_id` | string | 否 | `""` | 原样关联和回传，不解释业务含义 |
| `request_id` | string | 建议 | `""` | 活动会话级幂等键；重复相同键和 channel 返回原 stream |
| `ws_url` | string | 条件必填 | XML 地址池 | `ws://` 或 `wss://` 媒体后端地址；省略时轮询 XML `default_ws_url/default_ws_urls` |
| `mix_type` | string | 否 | XML 默认值 | `mono`、`mixed` 或 `stereo` |
| `sample_rate` | integer/string | 否 | XML 默认值 | 如 `8000`、`16000`、`24000`、`48000`，字符串也可写 `"16k"`；实际通道采样率优先，见第 8.7 节 |

成功响应为 HTTP 202，响应体使用“通用成功响应”。`state=connecting` 表示请求已接受，不保证后端 WebSocket 已经建立；调用方应继续查询 `connected/state`。

Python 示例：

```python
payload = {
    "channel_id": "8bb70000-1111-2222-3333-444455556666",
    "external_id": "call_01KABCDEF",
    "request_id": "media-20260827-001",
    "ws_url": "ws://127.0.0.1:9000/stream",
    "mix_type": "stereo",
    "sample_rate": 16000,
}
response = requests.post(f"{BASE_URL}/streams", headers=HEADERS, json=payload, timeout=5)
data = response.json()
if response.status_code != 202:
    raise RuntimeError(data)
stream_id = data["stream_id"]
print(stream_id, data["state"])
```

### 4.4 查询媒体流

```http
GET /api/v1/streams/{stream_id}
```

路径参数：

| 参数 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `stream_id` | string | 是 | 创建媒体流时返回的 ID |

成功响应为 HTTP 200，格式为“通用成功响应”；不存在返回 HTTP 404。

Python 示例：

```python
stream_id = "stream_550e8400-e29b-41d4-a716-446655440000"
response = requests.get(f"{BASE_URL}/streams/{stream_id}", headers=HEADERS, timeout=3)
data = response.json()
if response.status_code == 404:
    print("媒体流不存在或已经完成回收")
else:
    response.raise_for_status()
    print(data["state"], data["connected"], data["tx_drop_frames"])
```

### 4.5 媒体动作统一端点

```http
POST /api/v1/streams/{stream_id}/actions
Content-Type: application/json
```

请求体至少包含 `action`。以下六种动作用于日常媒体控制，成功返回 HTTP 200 和通用动作响应。此端点复用 JSON 分发器，也接受 `get`（返回会话对象）和 `stop`（返回动作响应）；常规查询和停止建议使用第 4.4、4.6 节的 GET/DELETE。

#### 4.5.1 pause_capture：暂停上行采集

停止采集新的 FreeSWITCH → 后端音频，不影响后端音频注入；已进入发送队列的音频不会因此清空，可能继续发出。

```python
response = requests.post(
    f"{BASE_URL}/streams/{stream_id}/actions",
    headers=HEADERS, json={"action": "pause_capture"}, timeout=3,
)
response.raise_for_status()
print(response.json())
```

#### 4.5.2 resume_capture：恢复上行采集

```python
response = requests.post(
    f"{BASE_URL}/streams/{stream_id}/actions",
    headers=HEADERS, json={"action": "resume_capture"}, timeout=3,
)
response.raise_for_status()
```

#### 4.5.3 pause_playback：暂停下行注入

暂停后端 → FreeSWITCH 的 PCM 注入及尚未开始的文件任务，不停止上行采集，也不停止接收后端音频。暂停期间下行仍入队，队列满会丢弃旧帧；恢复后消费保留的音频。已经进入文件播放函数的任务不会因此立即暂停。

```python
response = requests.post(
    f"{BASE_URL}/streams/{stream_id}/actions",
    headers=HEADERS, json={"action": "pause_playback"}, timeout=3,
)
response.raise_for_status()
```

#### 4.5.4 resume_playback：恢复下行注入

```python
response = requests.post(
    f"{BASE_URL}/streams/{stream_id}/actions",
    headers=HEADERS, json={"action": "resume_playback"}, timeout=3,
)
response.raise_for_status()
```

#### 4.5.5 interrupt_playback：打断并清空播放队列

立即清空当前下行 PCM ring 和尚未开始的兼容文件任务。必须在 XML 中配置 `enable_barge_in=true`，否则返回 `ACTION_FAILED`。不停止上行或 WebSocket，也不会撤销已经注入的音频、终止正在执行的文件播放，或阻止后端继续发送旧 TTS。业务后端应取消旧合成与发送任务，再清空播放队列。

```python
response = requests.post(
    f"{BASE_URL}/streams/{stream_id}/actions",
    headers=HEADERS, json={"action": "interrupt_playback"}, timeout=3,
)
if not response.ok:
    raise RuntimeError(response.json())
print("播放已打断", response.json())
```

#### 4.5.6 send_text：向媒体后端透传文本

请求：

```json
{"action":"send_text","text":"language=zh-CN"}
```

| 参数 | 类型 | 必填 | 说明 |
|---|---|---:|---|
| `action` | string | 是 | 固定为 `send_text` |
| `text` | string | 是 | 通过当前媒体 WebSocket 文本帧发送的内容；建议使用不含嵌套引号的简洁控制文本 |

```python
response = requests.post(
    f"{BASE_URL}/streams/{stream_id}/actions",
    headers=HEADERS,
    json={"action": "send_text", "text": "language=zh-CN"},
    timeout=3,
)
response.raise_for_status()
```

### 4.6 停止媒体流

```http
DELETE /api/v1/streams/{stream_id}
```

该接口异步停止媒体 WebSocket、播放线程并卸载 media bug，不会挂机。成功为 HTTP 200，返回 `{"ok":true,"stream_id":"stream_...","action":"stop","accepted":true}`；媒体资源完成回收后查询返回 404。

Python 示例：

```python
response = requests.delete(f"{BASE_URL}/streams/{stream_id}", headers=HEADERS, timeout=3)
if response.status_code == 404:
    print("媒体流已经不存在")
else:
    response.raise_for_status()
    print(response.json())
```

## 5. 通过 ESL 调用

### 5.1 准备 Python ESL 库

FreeSWITCH 源码构建环境通常在 `libs/esl` 提供 Python ESL 模块。安装方式随发行版而异。代码运行前需确保：

```python
import ESL
```

能够成功。ESL 连接参数取自 `event_socket.conf.xml`。生产环境不要开放无访问控制的 8021 端口。

### 5.2 公共 ESL 调用封装

```python
import json
import ESL

ESL_HOST = "127.0.0.1"
ESL_PORT = 8021
ESL_PASSWORD = "ClueCon"  # 必须替换为实际强密码

def connect_esl():
    con = ESL.ESLconnection(ESL_HOST, str(ESL_PORT), ESL_PASSWORD)
    if not con.connected():
        raise ConnectionError("无法连接 FreeSWITCH ESL")
    return con

def stream_json(con, payload):
    # separators 去掉空格，避免不同 ESL 客户端对命令参数做额外拆分。
    command = json.dumps(payload, ensure_ascii=False, separators=(",", ":"))
    event = con.api("taering_stream_json", command)
    if event is None:
        raise RuntimeError("FreeSWITCH 未返回结果")
    text = event.getBody().strip()
    data = json.loads(text)
    if not data.get("ok"):
        raise RuntimeError(data)
    return data
```

### 5.3 ESL 创建媒体流

入参、默认值和返回字段与 HTTP 创建接口一致；唯一额外字段是 `action=start`。

```python
con = connect_esl()
result = stream_json(con, {
    "action": "start",
    "channel_id": "8bb70000-1111-2222-3333-444455556666",
    "external_id": "call_01KABCDEF",
    "request_id": "media-20260827-001",
    "ws_url": "ws://127.0.0.1:9000/stream",
    "mix_type": "stereo",
    "sample_rate": 16000,
})
stream_id = result["stream_id"]
print(result)
```

### 5.4 ESL 查询媒体流

可传 `stream_id`，也可为旧系统传 `channel_id`。同时存在时优先使用 `stream_id`；非空 `stream_id` 查找失败时不会回退到 `channel_id`。

```python
result = stream_json(con, {"action": "get", "stream_id": stream_id})
print(result["state"], result["connected"])
```

### 5.5 ESL 暂停与恢复上行

```python
stream_json(con, {"action": "pause_capture", "stream_id": stream_id})
stream_json(con, {"action": "resume_capture", "stream_id": stream_id})
```

### 5.6 ESL 暂停与恢复下行

```python
stream_json(con, {"action": "pause_playback", "stream_id": stream_id})
stream_json(con, {"action": "resume_playback", "stream_id": stream_id})
```

### 5.7 ESL 打断播放

```python
stream_json(con, {"action": "interrupt_playback", "stream_id": stream_id})
```

### 5.8 ESL 发送文本

```python
stream_json(con, {
    "action": "send_text",
    "stream_id": stream_id,
    "text": "language=zh-CN",
})
```

### 5.9 ESL 停止媒体流

停止只释放媒体资源，不会执行 `uuid_kill`，不会改变 FCC 呼叫状态。

```python
stream_json(con, {"action": "stop", "stream_id": stream_id})
```

### 5.10 ESL 查询版本、指标和授权

```python
def legacy_api(con, command, args=""):
    event = con.api(command, args)
    if event is None:
        raise RuntimeError(f"{command} 没有返回结果")
    return event.getBody().strip()

print(legacy_api(con, "taering_stream", "version"))
print(legacy_api(con, "taering_stream", "stats"))
print(legacy_api(con, "taering_stream", "license"))
```

## 6. 传统 taering_stream API（ESL 兼容）

传统接口使用 FreeSWITCH UUID 作为控制主键，返回文本结果；操作命令通常以 `+OK`、`-ERR` 或 `-USAGE` 开头，`stats` 则为多行键值输出。新项目推荐使用结构化的 `taering_stream_json`。

| 命令 | 参数 | 说明 |
|---|---|---|
| `version` | 无 | 查询模块版本和能力 |
| `stats` | 无 | 查询活动会话、worker、缓冲、丢帧和授权摘要 |
| `license` | 无 | 查询授权状态、机器指纹和有效期 |
| `<uuid> start` | `[url|-] [mix|-] [rate|-] [metadata_json]` | 启动媒体流 |
| `<uuid> stop` | `[metadata_json]` | 停止媒体流 |
| `<uuid> pause` | 无 | 同时暂停上行和下行 |
| `<uuid> resume` | 无 | 同时恢复上行和下行 |
| `<uuid> pause_stream` | 无 | 仅暂停上行 |
| `<uuid> resume_stream` | 无 | 仅恢复上行 |
| `<uuid> pause_play` | 无 | 仅暂停下行 |
| `<uuid> resume_play` | 无 | 仅恢复下行 |
| `<uuid> interrupt_play` | 无 | 清空播放队列，需要启用 barge-in |
| `<uuid> send_text` | `<text>` | 透传文本；传统命令不适合包含空格的复杂文本 |

Python ESL 兼容示例：

```python
uuid = "8bb70000-1111-2222-3333-444455556666"
print(legacy_api(con, "taering_stream", f"{uuid} start ws://127.0.0.1:9000/stream stereo 16k"))
print(legacy_api(con, "taering_stream", f"{uuid} pause_stream"))
print(legacy_api(con, "taering_stream", f"{uuid} resume_stream"))
print(legacy_api(con, "taering_stream", f"{uuid} pause_play"))
print(legacy_api(con, "taering_stream", f"{uuid} resume_play"))
print(legacy_api(con, "taering_stream", f"{uuid} interrupt_play"))
print(legacy_api(con, "taering_stream", f"{uuid} send_text language=zh-CN"))
print(legacy_api(con, "taering_stream", f"{uuid} stop"))
```

## 7. Dialplan 调用

### 7.1 调用格式

模块作为 FreeSWITCH application 使用时自动取得当前 channel UUID。

```xml
<action application="taering_stream" data="start"/>
<action application="bridge" data="user/1000"/>
```

上述 `action` 写在实际命中的 Dialplan 路由中，即 `<extension>` 下的 `<condition>` 内。下面直接提供完整 XML、已有路由的插入示例、保存路径和重载验证命令。

完整格式：

```text
start [ws://|wss://url|-] [mono|mixed|stereo|-] [rate|-] [metadata_json]
stop [metadata_json]
clear
interrupt_play
```

`data` 为空时等同 `start`；还支持 `start mono 16k` 这样的省略 URL 简写。`clear` 与 `interrupt_play` 等效，均要求 `enable_barge_in=true`。这里的简写仅适用于 Dialplan；传统 CLI 覆盖混音、采样率而保留默认 URL 时，应写 `<uuid> start - mono 16k`。

执行结果写入 channel variable：

```text
taering_stream_result=+OK
taering_stream_result=+OK already started
taering_stream_result=-ERR
```

Python 生成 dialplan XML 示例：

```python
from xml.sax.saxutils import escape

backend = escape("ws://127.0.0.1:9000/stream", {'"': "&quot;"})
application_xml = (
    f'<action application="taering_stream" '
    f'data="start {backend} stereo 16k"/>'
)
print(application_xml)
```

### 7.2 完整呼叫路由 XML

模块配置与呼叫路由是两个文件：第 11 节的 `mod_taering_stream.conf.xml` 配置后端与媒体参数；本节的 Dialplan XML 决定哪个号码触发推流。以下完整示例适用于接听后桥接分机并接入媒体后端的场景，可直接复制保存。

将以下内容保存为 `${conf_dir}/dialplan/default/00_taering_stream_demo.xml`，前提是来话进入 `default` context，`9196` 未被其他路由抢先匹配，且分机 `1000` 已配置：

```xml
<include>
  <extension name="taering_stream_demo">
    <condition field="destination_number" expression="^9196$">
      <action application="answer"/>
      <action application="taering_stream" data="start"/>
      <action application="bridge" data="user/1000"/>
      <action application="taering_stream" data="stop"/>
    </condition>
  </extension>
</include>
```

`start` 立即返回，不负责持续保持通话；此例由 `bridge` 承担后续呼叫流程，不能当作机器人独立接待的完整业务路由。纯机器人接听需要由实际呼叫控制流程保持通道及媒体帧运行，并负责超时、转人工和挂机；仅写 `answer`、`start` 后就结束路由不能保证持续对话。通道挂断时模块也会自动清理媒体资源。

已有业务路由可在实际需要采集音频的位置加入 `start`，并在后续呼叫流程返回后按需执行 `stop`。例如，以下三个 action 放入已有路由的 `<condition>` 内，网关名 `my_gateway` 换成实际配置名称：

```xml
<action application="taering_stream" data="start"/>
<action application="bridge" data="sofia/gateway/my_gateway/${destination_number}"/>
<action application="taering_stream" data="stop"/>
```

仅覆盖混音和采样率、继续使用模块默认后端地址时：

```xml
<action application="taering_stream" data="start mixed 16k"/>
```

在标准 `/opt/freeswitch` 安装下，示例文件通常保存为 `/opt/freeswitch/etc/freeswitch/dialplan/default/00_taering_stream_demo.xml`。实际目录以 `fs_cli -x "global_getvar conf_dir"` 的输出为准；如果来话进入其他 context，应放到该 context 实际加载的路由位置。

修改 Dialplan 后：

```bash
fs_cli -x "reloadxml"
fs_cli -x "xml_locate dialplan"
```

检查输出中是否存在 `taering_stream_demo`。仅修改 Dialplan 不需要重载本模块；修改模块 XML 的生效方式见第 11.3 节。

### 7.3 客服业务 metadata

传统 CLI 和 Dialplan 支持把合法 JSON 原样加入后端 `start/stop` 控制帧的 `metadata` 字段；JSON 应紧凑且不含空格，XML 属性内的引号必须转义：

```xml
<action application="taering_stream"
        data="start - mono 16k {&quot;scene&quot;:&quot;customer_service&quot;,&quot;language&quot;:&quot;zh-CN&quot;}"/>
```

`scene`、`language` 是业务自定义示例，模块不解释其含义。HTTP/JSON 创建接口当前不读取 `metadata`，需要关联业务时使用 `external_id`，或通过 `send_text` 与后端约定控制消息。

## 8. 媒体后端 WebSocket 协议

### 8.1 连接角色

`mod_taering_stream` 是 WebSocket client，客户的 ASR/AI/TTS 服务是 WebSocket server。每个媒体流使用一条独立连接。

### 8.2 start 文本帧

连接成功后模块先发送：

```json
{
  "type": "start",
  "uuid": "8bb70000-1111-2222-3333-444455556666",
  "stream_id": "stream_550e8400-e29b-41d4-a716-446655440000",
  "external_id": "call_01KABCDEF",
  "sample_rate": 16000,
  "mix_type": "stereo"
}
```

传统 CLI/dialplan 启动时还可携带 `metadata` JSON。模块不再发送 `ringing`、`early_media`、`answered` 或 `hangup`；呼叫状态由 FCC/ESL 业务系统处理。

### 8.3 FreeSWITCH → 后端二进制帧

- 编码：有符号 `PCM16LE`；
- 文件头：无，裸 PCM；
- 采样率：`start.sample_rate`；
- 分片：默认约 20ms，由 `packet_ms` 控制；
- `mono`：read 方向单声道；
- `mixed`：read/write 混合成单声道；
- `stereo`：双声道交织，左=read、右=write。

上述声道约定适用于普通通道，loopback A 腿例外见第 8.7 节。16kHz/20ms 时，mono 通常为 640 bytes，stereo 通常为 1280 bytes。上行按媒体回调数据切分，尾片可以短于配置帧长，后端不要要求每条消息都恰好等于这个字节数。

Python WebSocket 后端示例：

```bash
pip install websockets
```

```python
import asyncio
import json
import websockets

async def media_handler(websocket):
    start = json.loads(await websocket.recv())
    assert start["type"] == "start"
    print("media started", start["stream_id"], start["sample_rate"])

    async for message in websocket:
        if isinstance(message, bytes):
            pcm16le = message
            # 将 pcm16le 送入流式 ASR。
            # 如需回放，可发送同采样率的单声道 PCM16LE：
            # await websocket.send(tts_pcm16le_chunk)
        else:
            try:
                control = json.loads(message)
            except json.JSONDecodeError:
                # send_text 也可能发送普通文本，例如 language=zh-CN。
                print("control text", message)
                continue
            if isinstance(control, dict) and control.get("type") == "stop":
                break
            # 其他 JSON 业务控制消息按后端约定处理。

async def main():
    async with websockets.serve(media_handler, "127.0.0.1", 9000):
        await asyncio.Future()

asyncio.run(main())
```

### 8.4 后端 → FreeSWITCH 二进制帧

生产推荐路径：发送单声道裸 `PCM16LE` WebSocket binary frame，采样率与 `start.sample_rate` 一致，建议每帧 20ms 或 40ms。

### 8.5 文本控制和兼容音频

清空播放队列：

```json
{"type":"clear","uuid":"<channel UUID>"}
```

兼容 base64 音频：

```json
{"uuid":"<channel UUID>","format":"raw","audio_b64":"..."}
```

按当前源码，开关规则如下：

| 下行内容 | 必需开关 | 处理方式 |
|---|---|---|
| WebSocket binary PCM | 无需开启兼容开关 | 直接进入下行 PCM ring |
| 文本 `audio_b64`，`format=raw` | `allow_text_audio=true` | 解码后进入 PCM ring |
| 文本 `audio_b64`，`format=wav/mp3/ogg` | `enable_file_playback=true` | 解码、临时落盘、排队播放；不受 `allow_text_audio` 限制 |
| `type=clear` 或 `type=interrupt_play` | `enable_barge_in=true` | 清空 PCM 与待播文件队列 |

`raw` 还接受 `pcm/pcm16/pcm16le` 别名，`base64/b64` 也按 base64 PCM 处理；省略或未知 `format` 当前会按 raw 路径处理，建议始终明确填写格式。文件播放依赖 FreeSWITCH 对应格式的解码支持及 `temp_dir` 写权限；正常播放结束后删除临时文件。文件开关在入队时检查，关闭它不代表收到文件消息时完全不会临时写盘。

文本消息的 `uuid` 若提供必须匹配当前连接的 channel UUID，否则触发 error；当前实现允许省略 `uuid`，接入时仍建议显式携带。`clear` 不依赖 `allow_text_audio`，但受打断开关限制；失败通过 `taering_stream::error` 通知，没有 WebSocket JSON 成功回执。未被识别为清空或音频的文本通过 `taering_stream::json` 事件透传。

高并发实时业务应使用二进制 PCM，不建议 base64 或文件落盘。

### 8.6 stop 文本帧

主动停止时模块尝试发送：

```json
{
  "type": "stop",
  "uuid": "<channel UUID>",
  "stream_id": "<stream ID>",
  "external_id": "<opaque ID>",
  "sample_rate": 16000,
  "mix_type": "stereo"
}
```

网络异常或 channel 被 FreeSWITCH 销毁时，连接可能直接关闭，后端不能依赖一定收到 stop。

### 8.7 实际采样率、分片与 loopback 方向

模块启动媒体流时读取 FreeSWITCH 通道的 `actual_samples_per_second`，有效时优先于请求和 XML 里的采样率。例如通道实际为 8000 Hz，即使请求 `16000`，后端 `start.sample_rate` 仍可能是 8000。模块没有独立的 PCM 重采样过程，后端必须以 `start` 帧为准；ASR/TTS 需要其他采样率时由后端转换。改变配置不代表改变已协商的电话编码。

普通通道及 loopback B 腿默认 read 采集、write 注入；检测到 `loopback_leg=A` 时自动改为 write 采集、read 注入。当前 A 腿路径直接上传采集帧，即使 `mix_type=stereo` 也不执行普通 read/write 双声道交织，应使用 `mono` 并验证实际音频方向。`other_loopback_leg_uuid` 用于记录关联信息，不会自动给另一条腿再创建媒体流。

普通通道 `mixed` 使用最近的 write 缓存与 read 混合；`stereo` 使用 read/write 交织，缺少可用 write 缓存时右声道补零。下行 PCM 叠加到当前媒体帧并做 16 位饱和限幅，属于音频注入，不替换整个呼叫流程，也不是回声消除功能。

下行以 `packet_ms` 为单位切分，不足一片时补零。建议每次发送完整分片或其整数倍，避免频繁短片引入补零静音。TX/RX ring 满时丢弃最旧帧以保留较新音频；不要把整段长 TTS 一次灌入小缓冲，应按播放节奏流式发送。

### 8.8 地址池、重连与后端恢复

未显式指定 URL 时，新建会话从 `default_ws_url/default_ws_urls` 合并的地址池轮询选择。地址池用于新会话分流，不提供后端健康检查；断线重连使用本会话原 URL，不自动切换到池内其他地址。

连接错误或关闭后按 `reconnect_base_ms` 指数退避，倍数最多 64，再受 `reconnect_max_ms` 限制。`reconnect_max_attempts=0` 表示不限连续尝试次数；成功建连后尝试计数归零，超过有限次数会把媒体会话置为停止并进入清理流程，不主动挂机。

重连成功后重新发送 `start`，沿用原媒体会话的 `stream_id` 和业务关联信息。后端应按断链重建媒体处理状态，不能把每次连接都当作新电话。断线期间不采集新的上行音频，模块不提供断线音频补录或可靠消息重放保证。

## 9. FreeSWITCH 自定义媒体事件

事件子类：

- `taering_stream::connect`
- `taering_stream::disconnect`
- `taering_stream::json`
- `taering_stream::error`
- `taering_stream::play`

可解析到活动会话时，事件包含 `Unique-ID`、`Stream-ID`、`External-ID`。播放事件还包含 `Track-ID`、`Play-State`、`Audio-Format`、`Played-Ms`、`Played-Samples`、`Playback-Generation`。

Python ESL 订阅示例：

```python
con = connect_esl()
con.events("plain", "CUSTOM taering_stream::connect taering_stream::disconnect taering_stream::error taering_stream::play")

while True:
    event = con.recvEvent()
    if event is None:
        break
    print({
        "subclass": event.getHeader("Event-Subclass"),
        "channel_id": event.getHeader("Unique-ID"),
        "stream_id": event.getHeader("Stream-ID"),
        "external_id": event.getHeader("External-ID"),
        "play_state": event.getHeader("Play-State"),
        "body": event.getBody(),
    })
```

### 9.1 事件含义与播放进度

| 子类 / `Play-State` | 触发含义 |
|---|---|
| `connect` | 包含连接请求、重连请求或实际建连通知；正文为 `websocket connected` 才表示本次已建立连接，仍建议结合查询状态 |
| `disconnect` | WebSocket 正常关闭等断链通知 |
| `error` | 连接异常、UUID 不匹配、载荷过大、队列满、播放开关拒绝等，具体原因在正文 |
| `json` | 后端未被音频/清空逻辑消费的文本，正文保留原文本 |
| `play / queued` | 兼容文件任务入队 |
| `play / start` | 兼容文件任务开始处理 |
| `play / done` | 文件播放函数成功返回 |
| `play / error_no_session` | 文件开始播放时找不到通道 |
| `play / error_playback` | 文件播放函数返回失败 |
| `play / interrupted` | 成功清空播放队列，携带被清空批次的 PCM 已注入进度 |

`Track-ID` 由模块为文件任务生成，不是后端传入的 TTS ID。PCM 打断事件的 `Track-ID` 为空，`Audio-Format=pcm`；`Played-Samples` 为该批次已注入样本数，`Played-Ms=Played-Samples*1000/sample_rate`。`Playback-Generation` 用于区分缓冲播放批次，不能等同业务轮次 ID。

当前 PCM 路径不会逐片发出 `queued/start/done` 或周期进度事件；打断时才通过 `interrupted` 报告进度。文件事件的进度字段当前为 0。不能仅靠这些事件准确判断每一句实时 TTS 是否完整播完，也不能把“已注入”解释为远端已确认听到。

## 10. 生产接入注意事项

- HTTP 默认只监听 `127.0.0.1`；不要把明文 HTTP 直接暴露公网。
- HTTP Token 和 ESL 密码是两套独立凭据，都必须使用高强度随机值。
- `request_id` 当前只保证活动会话范围内幂等，不是持久化业务幂等。
- 创建成功 HTTP 202 只表示已接受，应查询到 `connected=true/state=streaming` 后再判断媒体后端已连通。
- 停止媒体不会挂机；挂机由 FCC、dialplan 或 ESL 呼叫控制程序负责。
- `interrupt_playback` 的触发时机由业务后端决定，模块不自行做 VAD 或语义判断。
- `tx_drop_frames/rx_drop_frames` 持续增长表示后端、网络或播放消费不足，应检查缓冲和实时处理能力。
- 实时主路径使用 binary PCM；base64 和文件播放仅作为兼容方案。
- 客户升级模块后应完成 HTTP 鉴权、ESL 控制、真实 channel 挂载、双向 PCM、断线重连和打断回归测试。

## 11. 模块 XML 配置完整说明

### 11.1 文件位置、最小配置与完整配置

配置文件名为 `mod_taering_stream.conf.xml`。部署路径为 `${conf_dir}/autoload_configs/mod_taering_stream.conf.xml`；标准 `/opt/freeswitch` 安装通常对应 `/opt/freeswitch/etc/freeswitch/autoload_configs/mod_taering_stream.conf.xml`。先查询实际目录：

```bash
fs_cli -x "global_getvar conf_dir"
```

模块直接读取该目录下的 XML 文件。以下为实时客服媒体后端的最小配置示例，将 URL 换成实际服务地址；呼叫路由另按第 7 节配置：

```xml
<configuration name="mod_taering_stream.conf" description="mod_taering_stream">
  <settings>
    <param name="default_ws_url" value="ws://127.0.0.1:9000/stream"/>
    <param name="default_mix_type" value="mono"/>
    <param name="default_sample_rate" value="16000"/>
    <param name="default_autoplay" value="true"/>
    <param name="enable_barge_in" value="true"/>
    <param name="network_workers" value="4"/>
    <param name="packet_ms" value="20"/>
    <param name="tx_buffer_ms" value="80"/>
    <param name="rx_buffer_ms" value="80"/>
    <param name="allow_text_audio" value="false"/>
    <param name="enable_file_playback" value="false"/>
    <param name="api_enabled" value="false"/>
  </settings>
</configuration>
```

此例开启业务主动打断能力，需要后端自行判断客户插话并发送 `clear` 或调用打断接口；模块不会自动做 ASR、VAD、LLM 或 TTS。需要 HTTP 控制时，在同一 `<settings>` 中将 `api_enabled` 改为 `true` 并加入第 4.1 节的监听地址、端口和 Token，避免重复定义同名普通参数。

#### 全部 26 项的完整 XML

以下配置列出模块支持的全部参数，数值与随包配置一致，可整体保存为 `mod_taering_stream.conf.xml`。保存前将空的 `default_ws_url` 改成实际媒体后端地址，例如 `ws://127.0.0.1:9000/stream`；若每次启动都会传入 URL，也可以保持为空。

这是完整基线配置。上面的客服最小示例使用 `mono` 并开启打断；若采用下方完整配置开展同一场景，可将 `default_mix_type` 改为 `mono`、`enable_barge_in` 改为 `true`。两份配置选择一份作为文件内容，不要把两个 `<configuration>` 根节点拼到同一文件中。

```xml
<configuration name="mod_taering_stream.conf" description="mod_taering_stream">
  <settings>
    <!-- HTTP 控制接口：默认关闭，开启时必须填写非空 Token。 -->
    <param name="api_enabled" value="false" />
    <param name="api_listen_host" value="127.0.0.1" />
    <param name="api_listen_port" value="18081" />
    <param name="api_token" value="" />

    <!-- 媒体后端：必须填写实际 URL，或在每次启动请求中传入 URL。 -->
    <param name="default_ws_url" value="" />
    <param name="default_ws_urls" value="" />

    <!-- 音频模式、采样率与播放控制。 -->
    <param name="default_mix_type" value="stereo" />
    <param name="default_sample_rate" value="16000" />
    <param name="default_autoplay" value="true" />
    <param name="enable_barge_in" value="false" />

    <!-- 队列及载荷限制：字节值使用十进制整数。 -->
    <param name="tx_queue_limit" value="524288" />
    <param name="rx_queue_limit" value="524288" />
    <param name="play_queue_limit" value="4" />
    <param name="max_text_payload" value="2097152" />
    <param name="max_audio_payload" value="8388608" />

    <!-- 断线重连参数。 -->
    <param name="reconnect_base_ms" value="300" />
    <param name="reconnect_max_ms" value="5000" />
    <param name="reconnect_max_attempts" value="0" />

    <!-- 网络线程、分片和缓冲时长。 -->
    <param name="network_workers" value="4" />
    <param name="packet_ms" value="20" />
    <param name="tx_buffer_ms" value="80" />
    <param name="rx_buffer_ms" value="80" />

    <!-- 兼容音频路径与调试日志。 -->
    <param name="allow_text_audio" value="false" />
    <param name="enable_file_playback" value="false" />
    <param name="log_debug" value="false" />

    <!-- 临时文件目录：FreeSWITCH 运行用户必须可写。 -->
    <param name="temp_dir" value="/tmp/mod_taering_stream" />
  </settings>
</configuration>
```

URL 查询参数中的 `&` 在 XML 属性里必须写成 `&amp;`，双引号写成 `&quot;`。HTTP 参数、媒体参数都位于同一个 `<settings>` 中；Dialplan 的 `<action>` 则放到独立路由文件里。

### 11.2 全部配置项、默认值与范围

下表区分“源码默认值”（未配置该参数时）与“随包模板值”（仓库 XML 显式设置的值）。布尔值可写 `true/false`，也接受 `yes/no`、`on/off`、`1/0`。数值使用十进制，不写 `KB/MB` 单位；XML 采样率使用整数，`16k` 简写只用于启动接口。

| 参数 | 源码默认值 | 随包模板值 | 作用与范围 |
|---|---|---|---|
| `default_ws_url` | 空 | 空 | 默认 `ws://` 或 `wss://` URL，可重复多行并合并到地址池；启动请求提供 URL 时覆盖本次选择 |
| `default_ws_urls` | 空 | 空 | 多个 URL 以逗号或分号分隔，与单地址配置合并；URL 内不要有空格 |
| `default_mix_type` | `stereo` | `stereo` | `mono/mixed/stereo`，音频方向及 loopback 限制见第 8.7 节 |
| `default_sample_rate` | `16000` | `16000` | 8000～192000 Hz；启动时优先采用实际通道采样率 |
| `default_autoplay` | `true` | `true` | 是否默认允许下行注入/文件任务开始；为 false 时需显式恢复播放 |
| `enable_barge_in` | `false` | `false` | 是否允许 HTTP/JSON、CLI、Dialplan、WebSocket 的清空/打断操作 |
| `tx_queue_limit` | `1048576` | `524288` | 上行 ring 容量约束及文本发送队列字节上限，65536～268435456 |
| `rx_queue_limit` | `1048576` | `524288` | 下行 ring 容量约束，65536～268435456 字节 |
| `play_queue_limit` | `32` | `4` | 单会话待播文件任务上限，1～10000；不统计 PCM ring |
| `max_text_payload` | `2097152` | `2097152` | WebSocket 文本接收/发送载荷限制，1024～67108864 字节 |
| `max_audio_payload` | `8388608` | `8388608` | 进入上行二进制发送或下行 PCM 队列的音频块上限，1024～268435456 字节；当前文件落盘路径没有复用此项检查，文件消息仍受文本载荷上限约束 |
| `reconnect_base_ms` | `300` | `300` | 重连初始等待，10～60000 ms |
| `reconnect_max_ms` | `5000` | `5000` | 重连等待上限，10～600000 ms；小于 base 时提升到 base |
| `reconnect_max_attempts` | `0` | `0` | 0～1000000；0 为不限连续尝试次数，成功建连后计数归零 |
| `network_workers` | `0` | `4` | 0～8；0 按 CPU 自动选择，1～8 显式指定网络 worker 数 |
| `packet_ms` | `20` | `20` | ring 分片时长，10～60 ms；通常选 20 或 40 |
| `tx_buffer_ms` | `80` | `80` | 上行目标缓冲时长，20～10000 ms |
| `rx_buffer_ms` | `80` | `80` | 下行目标缓冲时长，20～10000 ms |
| `allow_text_audio` | `false` | `false` | 允许文本 base64 PCM；不控制 wav/mp3/ogg 文件路径，见第 8.5 节 |
| `enable_file_playback` | `true` | `false` | 允许兼容文件任务入队并按需启动播放线程 |
| `log_debug` | `false` | `false` | 建连、媒体方向、收发分片、注入和重连调试日志，日志量较大 |
| `api_enabled` | `false` | `false` | 开启独立 HTTP 服务；开启但 Token 为空时模块加载失败 |
| `api_listen_host` | `127.0.0.1` | `127.0.0.1` | HTTP 监听地址 |
| `api_listen_port` | `18081` | `18081` | HTTP 监听端口，1～65535 |
| `api_token` | 空 | 空 | HTTP Bearer Token；内部存储最多 255 字节，建议使用较短的高强度随机 ASCII Token |
| `temp_dir` | `/tmp/mod_taering_stream` | 同源码默认值 | 兼容文件播放临时目录，FreeSWITCH 运行用户需要写权限 |

ring 槽位按 `ceil(buffer_ms/packet_ms)` 计算，先限制在 4～2048 槽，再在可保留至少 4 槽时按队列字节上限缩减。因此目标缓冲时长并非精确的播放延迟，字节上限也不能绕过最小槽位约束。只增大 `tx_queue_limit/rx_queue_limit` 不会自动扩大由缓冲时长决定的容量。

多后端示例（放入 `<settings>`）：

```xml
<param name="default_ws_url" value="ws://10.0.0.11:9000/stream"/>
<param name="default_ws_url" value="ws://10.0.0.12:9000/stream"/>
<param name="default_ws_urls" value="ws://10.0.0.13:9000/stream;ws://10.0.0.14:9000/stream"/>
```

### 11.3 配置生效与验证

修改模块配置后，在可接受媒体会话中断的维护窗口执行：

```bash
fs_cli -x "reloadxml"
fs_cli -x "reload mod_taering_stream"
fs_cli -x "taering_stream version"
fs_cli -x "taering_stream stats"
```

模块在加载时读取配置，单独 `reloadxml` 不会刷新模块内存中的参数。模块重载包含卸载与重新加载，会清理已有媒体会话，不属于无损热更新。首次尚未加载时使用 `load mod_taering_stream`。需要随 FreeSWITCH 启动自动加载时，在现有 `autoload_configs/modules.conf.xml` 的 `<modules>` 内加入：

```xml
<load module="mod_taering_stream"/>
```

## 12. 商业授权与离线运行

### 12.1 客户侧行为

模块内置在线授权，客户无需另外填写授权 XML 或手工调用授权 API。模块自动发起检查；服务端签名经模块内置公钥验证后生效。

- 未授权、待审核、过期、禁用等不可用授权状态下，允许最多 1 路试用媒体流。
- 授权有效且缓存时间有效时进入不限并发模式；这表示授权不限路数，不代表机器性能无限。
- 后台审核通过后由自动轮询更新，不要求重启 FreeSWITCH。
- 网络不可达时，模块可使用有效的本地签名缓存；离线期限不超过授权到期时间与最近成功在线授权后 7 天中的较早者，实际以 `cache_expire_at` 为准。
- 授权失效不会强行中断已有媒体会话；新建流恢复为 1 路试用限制，活动数量未降到 1 路以下时无法再新增。

模块自动上报机器指纹、安装/运行实例标识、主机名、IP 与部署环境信息用于授权识别。客户诊断命令：

```bash
fs_cli -x "taering_stream license"
```

| 返回字段 | 说明 |
|---|---|
| `license_status` | `unauthorized/pending/authorized/expired/disabled/error/instance_conflict/client_upgrade_required/product_mismatch` |
| `license_mode` | 正常授权模式为 `unlimited`，降级模式为 `trial` |
| `max_concurrency` | 授权并发字段，正常不限并发为 `-1`，试用为 `1` |
| `active_concurrency` | 当前占用授权槽位数 |
| `fingerprint` | 机器指纹，供授权方定位客户机器 |
| `installation_id` | 持久化安装实例 ID |
| `runtime_id` | 当前运行实例 ID |
| `last_check_at` | 最近检查时间，Unix 秒时间戳 |
| `expire_at` | 授权到期时间，Unix 秒时间戳 |
| `cache_expire_at` | 离线缓存可用截止时间，Unix 秒时间戳 |
| `message` | 授权检查说明，排障时保留原文 |

`instance_conflict` 表示运行实例冲突，`client_upgrade_required` 表示服务要求升级客户端，`product_mismatch` 表示产品不匹配；应把版本和授权命令输出交给授权方核对。HTTP 创建流遇到授权容量拒绝通常返回 `START_FAILED`，不能仅凭这个错误认定 UUID 不存在。

### 12.2 缓存与安装标识

默认文件位置：

```text
${conf_dir}/autoload_configs/mod_taering_stream.license.cache.json
${conf_dir}/autoload_configs/mod_taering_stream.license.cache.json.installation_id
```

FreeSWITCH 运行用户需要能够创建和维护这些文件；它们是模块管理的签名缓存和身份文件，不是客户填写的配置模板。容器重建时应按部署方案保留安装标识；当前实现还会读取可用的 `/host/etc/machine-id` 作为宿主机身份来源。不要把同一安装标识复制为多个独立运行实例。

## 13. 版本、运行指标与排障

### 13.1 版本能力

`taering_stream version` 返回文本键值，包括 `module/version/build_id/interrupt_play/play_progress/barge_in/capture_route/license`。其中 `interrupt_play=enabled` 表示存在该能力，实际能否打断还要看 `barge_in=enabled`；`capture_route=loopback-aware` 表示支持第 8.7 节的方向识别。`play_progress=enabled` 的事件范围见第 9.1 节，不表示提供周期进度订阅。

### 13.2 stats 字段

`taering_stream stats` 是模块级和 worker 级文本统计，不是 HTTP JSON 端点：

| 字段 | 说明 |
|---|---|
| `active_sessions` | 当前仍在模块会话表中的媒体会话数 |
| `workers` | 实际网络 worker 数 |
| `text_audio/barge_in/file_playback` | 当前配置开关 |
| `packet_ms/tx_buffer_ms/rx_buffer_ms` | 当前分片和目标缓冲参数 |
| `licensed_active` | 当前授权槽位占用数，创建/回收期间可能与会话表数量略有差异 |
| `license_status/license_mode/license_max/license_expire_at` | 授权摘要 |
| `worker[n] sessions` | 该 worker 当前会话数 |
| `tx_fill/tx_slots`、`rx_fill/rx_slots` | 输出形式为 `tx_fill=已占用槽/总槽数`、`rx_fill=已占用槽/总槽数`，单位是帧槽而非字节 |
| `tx_drop/rx_drop` | 当前活动会话累计丢弃帧数的汇总 |
| `reconnects` | 当前活动会话重连调度次数的汇总 |

会话销毁后，其计数不再参与汇总；这些统计不是自进程启动以来永久递增的计数器。单会话状态和丢帧可通过 HTTP GET 或 JSON `get` 查询。

### 13.3 常见问题定位

| 现象 | 优先检查 |
|---|---|
| 模块加载失败 | 模块依赖、HTTP Token 是否为空、监听端口是否被占用以及 FreeSWITCH 日志 |
| `BACKEND_NOT_CONFIGURED` | 请求 URL 和 XML 默认地址池是否都为空；修改后是否重载模块 |
| `START_FAILED` | channel UUID 是否存在、授权并发是否已满、media bug 是否挂载失败 |
| `STREAM_ALREADY_EXISTS` | 同一通道已有活动流；活动幂等重试应复用原 `request_id` |
| HTTP 202 后没有音频 | 查询 `connected/state`，检查 WS 后端、实际通道媒体是否运行、采集暂停状态与 loopback 方向 |
| TTS 无声或速度异常 | 下行是否裸单声道 PCM16LE、是否匹配 `start.sample_rate`、是否暂停播放、通道是否有可注入媒体帧 |
| TTS 后半句丢失 | 后端是否突发发送整句，`rx_drop` 是否增长；按播放节奏返回并评估缓冲配置 |
| 打断失败或很快又响起 | `enable_barge_in` 是否已生效；旧 TTS 是否仍在发送；是否正在执行兼容文件播放 |
| 文件播报失败 | `enable_file_playback`、临时目录写权限、格式解码支持、文件队列和 error/play 事件 |
| 重连频繁 | 后端连接日志、网络、worker `reconnects` 与错误事件；地址池不会自动故障切换 |

需要详细诊断时设置 `log_debug=true` 并按第 11.3 节重载；排障完成后关闭，避免持续输出逐帧日志。

## 14. FreeSWITCH 进程内集成

`mod_fcc` 等同进程模块通过 FreeSWITCH API 注册表调用 `taering_stream_json`，复用第 5 节相同请求/响应契约：

```c
switch_stream_handle_t result = {0};
SWITCH_STANDARD_STREAM(result);
switch_api_execute("taering_stream_json",
                   "{\"action\":\"get\",\"channel_id\":\"<channel UUID>\"}",
                   NULL, &result);
/* 解析 result.data 中的 JSON，并处理 API 调用失败与 ok=false。 */
switch_safe_free(result.data);
```

调用方应先检查 API 是否注册；媒体模块不可用时，由调用方报告媒体能力不可用。不要直接链接 `mod_taering_stream.so` 内部符号，避免加载、升级和故障耦合。FCC 应把自己的 `call_id` 解析为 FreeSWITCH UUID 作为 `channel_id`，并把业务 `call_id` 放入 `external_id`。

本地 JSON 支持 `start/get/stop/pause_capture/resume_capture/pause_playback/resume_playback/interrupt_playback/send_text`。非 start 操作可用 `stream_id` 或 `channel_id`，优先 `stream_id`。相同活动 `request_id` 与 channel 重试只返回原会话，不应用新的 URL、采样率等参数；会话销毁后该键不再保留。

当前公开接口不提供媒体会话全量列表、运行时配置修改、自动外呼、自动转人工或呼叫生命周期订阅。这些由调用方的呼叫控制系统承担，不能把模块媒体状态当成电话状态。

