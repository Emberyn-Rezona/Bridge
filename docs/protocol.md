# WebSocket 协议

本文定义前端、外部 agent 与 hub 之间的 WebSocket 帧格式与事件语义。
REST 接口（历史、房间、设置、参与者）不在本文范围内。

## 术语

| 术语 | 含义 |
|---|---|
| hub | 后端中转所有帧、分配消息 id、落库的组件 |
| 参与者（participant） | 经本协议接入 hub 的身份：用户、外部 agent、平台托管的 agent |
| 房间（room） | 消息的路由单位，参与者经成员关系加入房间 |
| 帧（frame） | 一条 WebSocket 文本消息，内容为一个 JSON 对象 |

所有参与者使用同一套帧，收发对称。
平台托管的 agent 由后端 goroutine 代为连接 hub，与外部 agent 无区别。

## 连接

- 端点为 `/ws`，每帧是一个 UTF-8 JSON 对象。
- 连接建立后，客户端首帧必须是 `hello`，hub 以 `welcome` 响应。
- 在 `welcome` 之前收到其他帧，hub 回 `error`（`unauthorized`）并关闭连接。
- 浏览器使用登录会话 cookie，hub 校验 `Origin`。
- agent 在 `hello` 的 `payload.token` 携带 API token。
- token 不出现在 URL 中。

```text
client                          hub
  |--------- hello ------------->|
  |<-------- welcome ------------|
  |<-------- message (补发) -----|   id > last_id 的持久化消息，按 id 升序
  |<-------- message / typing ---|
```

## 信封

所有帧共用同一信封，字段是否出现取决于方向与事件类型。

```json
{
  "type": "message",
  "id": 1042,
  "client_id": "01JADQ5Z8K3V7N2X4M6R9T0WBC",
  "room_id": "r_general",
  "sender_id": "p_alice",
  "ts": 1760000000000,
  "caused_by": 1041,
  "depth": 1,
  "payload": { "content": "收到", "reply_to": 1040, "mentions": ["p_bob"] }
}
```

| 字段 | 类型 | 填写方 | 说明 |
|---|---|---|---|
| `type` | string | 发送方 | 事件类型，见「事件」 |
| `id` | integer | hub | 持久化消息的全局单调序号，兼作补发游标；只出现在 `message` 帧 |
| `client_id` | string | 发送方 | 发送方生成，在该发送方范围内唯一，推荐 ULID；hub 原样回传 |
| `room_id` | string | 发送方 | 目标房间；`hello`、`welcome`、`presence` 不带此字段 |
| `sender_id` | string | hub | 发送方的参与者 id；客户端发出的值被忽略 |
| `ts` | integer | hub | Unix 毫秒时间戳 |
| `caused_by` | integer | 发送方 | 触发本消息的消息 id，可选 |
| `depth` | integer | hub | 无 `caused_by` 时为 `0`，否则为被引用消息的 `depth` + 1 |
| `payload` | object | 发送方 | 内容随 `type` 而定 |

`id` 以外的实体 id（参与者、房间）是不透明字符串，不假设格式。

## 事件

| type | 方向 | 持久化 | 作用 |
|---|---|---|---|
| `hello` | 客户端 → hub | 否 | 握手，声明协议版本与补发游标 |
| `welcome` | hub → 客户端 | 否 | 握手响应 |
| `message` | 双向 | 是 | 一条完整消息；流式消息的终态也经由它下发 |
| `message.delta` | 双向 | 否 | 流式消息的增量文本 |
| `message.end` | 客户端 → hub | 否 | 发送方宣告流式消息结束 |
| `message.cancel` | 双向 | 否 | 请求流式消息的发送方停止生成 |
| `typing` | 双向 | 否 | 输入中状态 |
| `presence` | hub → 客户端 | 否 | 参与者上下线 |
| `room.changed` | hub → 客户端 | 否 | 房间的成员或属性发生变化 |
| `ping` | hub → 客户端 | 否 | 存活探测 |
| `pong` | 客户端 → hub | 否 | 对 `ping` 的响应 |
| `error` | hub → 客户端 | 否 | 协议错误 |

### `hello`

| `payload` 字段 | 类型 | 说明 |
|---|---|---|
| `version` | integer | 协议版本号，不兼容变更时递增；本文定义的版本为 `1` |
| `token` | string | 仅 agent 使用 |
| `last_id` | integer | 客户端已处理的最大消息 `id`；首次连接为 `0` |

```json
{
  "type": "hello",
  "payload": { "version": 1, "token": "<YOUR_AGENT_TOKEN>", "last_id": 1041 }
}
```

hub 不支持该版本时回 `error`（`unsupported_version`）并关闭连接。

### `welcome`

| `payload` 字段 | 类型 | 说明 |
|---|---|---|
| `version` | integer | hub 采用的协议版本号 |
| `participant_id` | string | 本连接对应的参与者 id |
| `head_id` | integer | 补发结束时的游标 |
| `resume` | string | `replayed` 或 `reset` |

- `replayed`：hub 把 `id` 大于 `last_id` 的消息按 `id` 升序以 `message` 帧补发。收到 `id` 等于 `head_id` 的帧即补发结束；无消息可补时 `head_id` 等于 `last_id`。
- `reset`：缺口超过 hub 的补发上限，hub 不补发。客户端经 REST 拉取历史，并以 `head_id` 作为新游标。

### `message`

| `payload` 字段 | 类型 | 填写方 | 说明 |
|---|---|---|---|
| `content` | string | 发送方 | 文本内容，必填 |
| `reply_to` | integer | 发送方 | 被引用消息的 `id`，仅用于展示 |
| `mentions` | string[] | 发送方 | 被 @ 的参与者 id |
| `status` | string | hub | `completed`、`cancelled`、`error`、`interrupted`；缺省为 `completed` |

- 客户端发出的 `message` 带 `client_id`，不带 `id`。
- hub 落库后向房间所有成员（含发送方）下发带 `id` 的 `message`；发送方以回传帧中的 `client_id` 对应自己的待确认消息。
- hub 对每条连接按 `id` 升序下发 `message` 帧，实时与补发一致；`last_id` 是「已处理的最大 `id`」依赖这一顺序。
- 同一发送方重复发送相同 `client_id`，hub 不重复落库，回传已落库的那条。
- `reply_to` 是展示用的引用关系，与 `caused_by` 无关。

### 流式消息

流式消息由发送方经 `message.delta` 逐段发出，以 `message.end` 收尾，终态由 hub 以 `message` 下发。

```text
producer                        hub                       其他成员
  |-- message.delta (seq 0) ---->|-- message.delta -------->|
  |-- message.delta (seq 1) ---->|-- message.delta -------->|
  |-- message.end -------------->|  落库，分配 id
  |<--- message (id, status) ----|--- message (id, status) ->|
```

`message.delta`：

| `payload` 字段 | 类型 | 说明 |
|---|---|---|
| `seq` | integer | 从 `0` 起，每帧加 `1` |
| `text` | string | 增量文本 |

- 同一条流式消息的所有帧使用同一个 `client_id`。
- `caused_by` 只在 `seq` 为 `0` 的帧上带，hub 在此时校验 `depth`。
- `seq` 供接收方检测丢帧；终态内容以随后的 `message` 为准。
- 接收方遇到未见过的 `client_id` 时新建临时气泡，不要求先收到 `seq` 为 `0` 的帧。

`message.end`：

| `payload` 字段 | 类型 | 说明 |
|---|---|---|
| `status` | string | `completed`、`cancelled`、`error` |
| `content` | string | 完整内容，与 hub 累积的增量不一致时以它为准 |
| `reply_to` | integer | 同 `message` |
| `mentions` | string[] | 同 `message` |

- hub 向客户端不下发 `message.end`，终态一律是 `message`，补发与实时走同一路径。
- 发送方断线而未发 `message.end` 时，hub 以已累积的内容落库，`status` 为 `interrupted`。

`message.cancel`：

| `payload` 字段 | 类型 | 说明 |
|---|---|---|
| `sender_id` | string | 流式消息发送方的参与者 id |
| `client_id` | string | 流式消息的 `client_id` |

- 客户端发出后，hub 转发给该流式消息的发送方。
- 发送方停止生成，并以 `status` 为 `cancelled`、`content` 为已生成内容发出 `message.end`。
- 发送方是平台托管的 agent 时，hub 直接取消其 goroutine。

### `typing`

| `payload` 字段 | 类型 | 说明 |
|---|---|---|
| `active` | boolean | `true` 开始输入，`false` 停止 |

### `presence`

| `payload` 字段 | 类型 | 说明 |
|---|---|---|
| `online` | boolean | 信封的 `sender_id` 所指参与者是否在线 |

hub 只向与该参与者共处至少一个房间的成员下发。

### `room.changed`

- 信封带 `room_id`，`payload` 为空对象。
- 房间成员或属性经 REST 变更后，hub 向受影响的在线成员下发。
- 客户端据此经 REST 重新拉取该房间。

### `ping`

- 信封带 `ts`，`payload` 为空对象。
- 下发周期与判死规则见「存活检测」。

### `pong`

| `payload` 字段 | 类型 | 说明 |
|---|---|---|
| `ts` | integer | 原样回显所响应的 `ping` 的 `ts` |

hub 以收到 `pong` 的时刻减去 `payload.ts` 得到该连接的 RTT，两个时刻来自 hub 的同一个时钟。

### `error`

| `payload` 字段 | 类型 | 说明 |
|---|---|---|
| `code` | string | 见下表 |
| `message` | string | 供人阅读的说明 |
| `ref` | string \| null | 引起错误的帧的 `client_id`，无则为 `null` |

| `code` | 触发 | 连接 |
|---|---|---|
| `bad_frame` | 非 JSON、缺必填字段、未知 `type` | 保持 |
| `unsupported_version` | `hello` 的版本 hub 不支持 | 关闭 |
| `unauthorized` | 鉴权失败，或在 `welcome` 之前发送其他帧 | 关闭 |
| `forbidden` | 发送方不是目标房间的成员 | 保持 |
| `room_not_found` | `room_id` 不存在 | 保持 |
| `depth_exceeded` | `depth` 超过 hub 的 `max_depth` | 保持 |

## 路由与触发

- hub 向房间所有在线成员投递该房间的全部事件，不替成员决定谁该响应。
- agent 依据 `mentions` 自行决定是否回复，hub 不主动调用任何参与者。
- agent 回复触发它的消息时，在 `caused_by` 填该消息的 `id`。
- hub 据此计算 `depth`；超过 `max_depth` 的消息或流被拒绝，并回 `error`（`depth_exceeded`）。
- 用户经界面发出的消息不带 `caused_by`，`depth` 为 `0`。

## 存活检测

连接是否存活只由「是否持续收到对端的帧」判定，两端使用同一套规则。

| 项 | 取值 |
|---|---|
| 心跳周期 T | 26 秒 |
| 传输容忍 δ | 13 秒 |
| 判死期限 | 3T + δ = 91 秒 |

- hub 在 `welcome` 之后对每条连接每隔 T 下发一个 `ping`，每条连接独立计时。
- 任何帧都刷新对端的「最近收帧时刻」`lastFrameAt`，不限于 `ping` 与 `pong`。
- `lastFrameAt` 的初值是 WebSocket 打开的时刻，握手阶段同样适用判死期限。
- 距 `lastFrameAt` 超过判死期限，即判定对端已死。
- hub 判定客户端已死时，关闭该连接。
- 客户端判定 hub 已死时，关闭连接并按「重连」重连。
- 判定时比较 `现在 − lastFrameAt` 与期限，不累计计时器触发次数；浏览器会节流后台标签页的定时器，标签页回到前台时立即比较一次。
- T 须小于链路上各反向代理的空闲超时，例如 nginx 的 `proxy_read_timeout` 默认 60 秒。

## 重连

- 客户端断线后重新连接，`hello.last_id` 填已处理的最大 `id`。
- 补发只含持久化的 `message`；断线期间完成的流式消息以终态 `message` 到达。
- 重连时仍在进行的流式消息不补发 `message.delta`；其后续增量按 `client_id` 归入临时气泡，终态由 `message` 校正。
- 「已连接」以收到 `welcome` 为准；WebSocket 已打开但尚无 `welcome` 的连接视为未连上。
- 收到 `error`（`unauthorized` 或 `unsupported_version`）后 hub 关闭连接，客户端不自动重连。
- 其余原因的断开，客户端按指数退避加随机重连：第 `n` 次尝试前等待 `random(0, min(cap, base × 2^n))`，`n` 从 `0` 起，`base` 与 `cap` 是客户端的可配置常量。
- `n` 为连续失败的次数：尝试在收到 `welcome` 之前失败，`n` 加一；连接收到 `welcome` 后持续不少于 455 秒（5 × 判死期限）再断开，`n` 清零；不足 455 秒就断开，`n` 同样加一。
