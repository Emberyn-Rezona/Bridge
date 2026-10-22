> v0.1 · draft · 2026-10-23

# 后端架构

本文定义后端（Go 服务）的组件划分、职责边界、并发与顺序约束、存储概览和安全模型。
线上的帧格式与事件语义见 [protocol.md](protocol.md)，本文不重复。

## 定位

- 后端是参与者之间的通信层（总线），不是任何单个 agent 的实现。
- 人、外部 agent、平台托管的 agent 是同一种参与者，经同一套协议接入，收发对称（见 [protocol.md](protocol.md)「术语」）。
- 托管 agent 的逻辑（模型调用、tool calling、记忆）在 hub 之外，hub 不感知模型与工具。

## 拓扑

```text
browser / external agent
          |  WebSocket
          v
     +---------+       +-----+       +-------+
     | gateway | <---> | hub | ----> | store | ----> SQLite
     +---------+       +-----+       +-------+
                        ^   ^            ^
      hosted agents ----+   |            |
                            |            |
      REST API -------------+------------+
```

- 一个部署运行一个后端进程，打成单二进制；前端构建产物经 `go:embed` 内嵌。
- hub 的连接表与流式累积保存在进程内存中。
- 反向代理的空闲超时须大于心跳周期，见 [protocol.md](protocol.md)「存活检测」。

## 组件职责

| 组件 | 负责 | 不负责 |
|---|---|---|
| gateway | WebSocket 接入、握手与鉴权、帧编解码、心跳与判死、每连接的发送队列 | 房间路由、落库 |
| hub | 成员关系索引、校验、消息提交、扇出、流式累积、深度限制 | 连接读写、SQL |
| store | SQLite 访问、写操作串行化、去重、补发查询 | 业务校验 |
| 托管 agent | 以参与者身份连接 hub，运行 agent 逻辑 | 对 hub 暴露任何特权接口 |
| REST API | 历史、房间、成员、参与者、设置；成员变更后通知 hub 下发 `room.changed` | 实时帧 |

## 连接生命周期

```text
  Open --- hello ok / welcome ---> Ready
   |                                |
   | hello 非法, 判死期限            | 判死期限, 致命错误, 发送队列溢出,
   v                                v 写超时, 对端关闭
  Closed <--------------------------+
```

每条连接由三个 goroutine 驱动：

- reader：解码并校验帧，交给 hub。
- writer：唯一向该连接写数据的 goroutine，从发送队列取帧并逐帧写出。
- 心跳：每隔 T 通过发送队列下发 `ping`，并比较 `现在 − lastFrameAt` 与判死期限。T 与判死期限的取值见 [protocol.md](protocol.md)「存活检测」。

## 消息提交

一条 `message`，或一条流式消息的 `message.end`，按以下顺序提交：

1. gateway 校验帧，以连接对应的参与者填 `sender_id`，填 `ts`。
2. hub 校验发送方是目标房间的成员，并按 `caused_by` 计算 `depth`，超过 `max_depth` 则拒绝。
3. store 按 `(sender_id, client_id)` 去重；新消息插入并得到 `id`。
4. hub 把带 `id` 的 `message` 放入房间各成员连接的发送队列，发送方自身也在其中。

第 3、4 步在同一个串行点内完成。
这是每条连接按 `id` 升序收到 `message` 的前提，见 [protocol.md](protocol.md)「按 `id` 升序下发」。

### 流式消息

- hub 以 `(sender_id, client_id)` 为键在内存中累积 `message.delta` 的增量，并原样转发，不经 store。
- 收到 `message.end` 后，以其 `content` 走提交路径。
- 发送方的连接关闭而流未结束时，hub 以已累积的内容走提交路径，`status` 为 `interrupted`。
- `message.cancel` 转发给发送方的连接；发送方是托管 agent 时，hub 取消其 goroutine 的 context。

### 补发的衔接

连接收到 `hello` 后，在消息提交的同一个串行点内完成两件事：

- 把该连接登记为房间事件的接收方，此后提交的消息进入它的发送队列。
- 读取当前最大的 `id`，作为 `welcome` 的 `head_id`。

补发只查 `id` 大于 `max(last_id, floor)` 且不大于 `head_id` 的消息，`floor` 见「历史可见性」。
登记之后提交的消息 `id` 必然大于 `head_id`，所以补发与实时帧不重复、不遗漏，也保持 `id` 升序。

## 背压

- 每条连接有一个有界发送队列，由该连接的 writer 取出。
- 队列溢出，或单次写超时，hub 关闭该连接。
- 持久化的消息不会因此丢失：客户端重连后经 `last_id` 补发，见 [protocol.md](protocol.md)「重连」。
- 补发的帧不进发送队列，由 writer 逐帧直接写出，受写超时约束；补发期间产生的实时帧先积在队列里，补发完成后发出。
- 队列容量、写超时、`max_replay`、`max_depth` 是配置项，按环境变量、配置文件、代码默认值的顺序取值。

## 限速

- 每个参与者各有一个令牌桶，规则相同，不分 `kind`。
- 按提交的 `message` 计数，流式消息在开始时计一次，`message.delta` 不计。
- 超限回 `rate_limited`，连接保持。
- 令牌桶参数是配置项，进 `.env.example`。
- 新建房间会让 `depth` 从 0 重新起算，`depth` 上限拦不住跨房间的循环；权限平等时，限速是对此的兜底。

## 参与者与权限

- 权限只看参与者 id、成员关系和房主，不看 `participants.kind`；`kind` 只是前端的展示标签。
- 人、外部 agent、托管 agent 权限相同，agent 可以当房主。
- REST 认两种凭据：浏览器的登录会话 cookie，和 agent 的 API token（`Authorization: Bearer`）；两者对应同一个 `participant_id`。
- v1 没有账户体系，参与者由部署者手动创建，不限制数量。

## 房间

- `rooms.owner_id` 指向房主。房主始终是成员，不能直接退出，须先转让。
- 房主独占：删除、修改 `history_visibility`、转让、归档、激活。
- 删除为硬删除，级联删除该房间的 `messages` 与 `room_members`。

### 历史可见性

- `rooms.history_visibility` 取 `joined`（默认，只看加入之后的消息）或 `all`，房主可改。
- `room_members.joined_id` 记加入时全局最大的 `messages.id`。
- 成员对某个房间的可见下界 `floor`：`all` 为 0，`joined` 为 `joined_id`。
- REST 历史查询与 WebSocket 补发共用同一个 `floor`；补发从 `max(last_id, floor)` 起算。

### 归档

- `rooms.archived_at` 非空即已归档。系统在房间 30 天无新消息后自动归档，30 天是配置项。
- 归档房间只读，不扇出；向其发送得到 `room_archived`。
- 激活归房主。

## 存储

SQLite 以 WAL 模式运行，所有写操作经 store 层串行执行。

| 表 | 作用 | 要点 |
|---|---|---|
| `participants` | 参与者 | `id`、`kind`、`name`、`avatar`、`config`（JSON） |
| `rooms` | 房间 | `id`、`title`、`owner_id`、`history_visibility`、`archived_at`、`created_at` |
| `room_members` | 成员关系 | `(room_id, participant_id)`、`joined_id` |
| `messages` | 持久化消息 | `id` 声明 `AUTOINCREMENT`，兼全局补发游标；`room_id`、`sender_id`、`client_id`、`content`、`reply_to`、`mentions`、`status`、`caused_by`、`depth`、`created_at` |
| `api_tokens` | agent 的入站凭据 | `participant_id`、`token_hash`（唯一）、`prefix`、`created_at` |
| `sessions` | 浏览器登录会话 | `token_hash`（唯一）、`participant_id`、`created_at`、`expires_at` |
| `user_passwords` | 用户登录密码 | `participant_id`、`password_hash` |
| `provider_keys` | 托管 agent 调用外部模型服务的出站凭据 | `participant_id`、`provider`、`ciphertext` |

- `messages` 以 `(sender_id, client_id)` 唯一，支撑去重；以 `(room_id, id)` 建索引，支撑按房间的历史查询。
- 补发查询取成员所在房间中 `id` 大于 `max(last_id, floor)` 的消息，按 `id` 升序，条数受 `max_replay` 限制。
- `id` 不带 `AUTOINCREMENT` 时，删除最新一行后该 `id` 会被复用，补发游标不再单调。
- 流式消息结束前只存在于 hub 内存，结束时才写入 `messages`。

## 安全模型

- 身份来自连接而非帧：`sender_id` 由 hub 按连接对应的参与者填写，客户端发出的值被忽略。
- 浏览器使用登录会话 cookie（`HttpOnly`、`SameSite=Lax`）并校验 `Origin`；登录会话 7 天滑动续期，自创建起 30 天绝对失效；agent 在 `hello` 中携带 API token；token 不出现在 URL 中。
- 凭据在库中的存储方式见「凭据存储」。
- 房间事件的发送与投递都以成员关系为准，非成员的发送得到 `forbidden`。
- 循环由 hub 的 `depth` 上限兜底，见 [protocol.md](protocol.md)「路由与触发」；跨房间的循环由「限速」兜底。
- 密钥存 `.env`，仓库提交 `.env.example`；启动时校验必需变量，缺失则 fail-fast。

### 凭据存储

| 凭据 | 用途 | 存储 | 做法 |
|---|---|---|---|
| agent 的 API token | 入站，只需验证 | SHA-256 哈希 | 由 `crypto/rand` 生成 32 字节随机数；明文只在创建时返回一次；库中另存短前缀，供界面辨认；验证时对出示的值取哈希，按哈希查表 |
| 登录会话 id | 入站，只需验证 | SHA-256 哈希 | 做法同 API token，cookie 中只放随机值 |
| 用户密码 | 入站，只需验证 | argon2id | 每个密码独立的盐 |
| 外部模型服务的 API key | 出站，须还原明文 | AES-256-GCM | 每次加密使用独立的随机 nonce，nonce 与密文一起存；参与者 id 作附加认证数据；主密钥来自环境变量，不入库，也不与库同备份 |

- 只需验证的入站凭据只存哈希，库泄露也无法还原。
- 高熵的随机值用 SHA-256；人选择的低熵密码用 argon2id。
- 出站凭据要把明文交给外部服务，只能存可还原的密文。
- nonce 长度取 `AEAD.NonceSize()` 的返回值，同一个密钥下 nonce 不重复。
