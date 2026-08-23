# FluentWork iOS App 端技术设计文档

**版本**：V1.2　**日期**：2026年8月　**对应**：技术方案 V3.2 第六章 / UI 设计文档 V1.2 / 团队分工文档 V1.1　**定位**：客户端实现详设

> 本文档是技术方案第六章（架构与选型层）与 UI 文档（交互层）之间的**模块内部设计层**，同时是 iOS 编程 Agent 的直接任务上下文。分工基线（已定案）：AudioEngine 与 SpeechSession 状态机由负责人手写，其余模块由 Agent 起草 + 负责人评审。本文档对两类代码一视同仁地定义接口契约——人写模块的接口冻结后，Agent 代码只依赖接口，不依赖实现细节。

---

## 一、客户端架构与模块边界

### 1.1 分层结构

```
Views（SwiftUI，纯声明式）
  └─ Store（@Observable，页面级状态，每页一个）
       └─ Service 层（可单测、无 UI 依赖）
            ├─ SpeechSessionClient  语音会话编排（状态机宿主）
            ├─ AudioEngine          音频采集/播放/打断（人写）
            ├─ APIClient            HTTPS 业务接口（async/await）
            ├─ SocketTransport      WSS 帧收发（无业务语义）
            ├─ CorpusStore          语料库本地缓存（SwiftData）
            └─ SyncEngine           后台同步（游标推进、冲突处理）
```

### 1.2 模块责任与归属

| 模块 | 职责 | 归属 | 依赖 |
|---|---|---|---|
| AudioEngine | 采集、Opus 编解码对接、播放、AEC 配置、打断停播、音频会话管理 | **人写** | 无（最底层） |
| SpeechSession 状态机 | 说的房间全部状态流转，唯一消费 AudioEngine 事件的模块 | **人写** | AudioEngine 接口 |
| SocketTransport | WSS 连接、帧编解码、序列号、重连 | Agent 起草 | 无 |
| SpeechSessionClient | 状态机 + Transport + 业务事件（徽章/转录）的组装 | 人写骨架，Agent 补全 | 上两者 |
| APIClient | REST 接口、票据获取、错误归一化 | Agent 起草 | 无 |
| CorpusStore / SyncEngine | SwiftData 读写、后台同步 | Agent 起草 | 无 |
| 全部 Views / Store | 页面与交互，严格按 UI 文档还原 | Agent 起草 | Store 只调 Service |

### 1.3 纪律（Agent 禁区清单）

1. Views 禁止直接引用 AudioEngine 与 SocketTransport——一切音频与连接状态只从 Store 读；
2. 状态机（`SpeechSessionReducer`）为纯函数实现（状态 × 事件 → 新状态 + 副作用清单），Agent 不得修改，只允许消费其输出；
3. 三方库白名单沿用技术方案 6.3，新增依赖一律打回；
4. 全 App 不引入震动反馈（UI 文档触觉纪律）；所有动效时长走 UI 文档档位，禁止自创弹簧参数。

---

## 二、SpeechSession 状态机详设（人写）

### 2.1 状态定义

```
idle ─► connecting ─► aiSpeaking ⇄ waitingUser ⇄ recording
                                        │
                                        ▼
                                    processing ─► aiSpeaking ...
任意状态 ─► degradedText（文本降级） / ended / failed
```

| 状态 | 含义 | UI 映射（UI 文档 4.2） |
|---|---|---|
| `idle` | 未进入房间 | —— |
| `connecting` | WSS 握手中（票据 60s 内） | 进入房间加载态 |
| `aiSpeaking` | AI 音频播放中 | 气泡波形动画 |
| `waitingUser` | 等待用户开口 | 说话键呼吸动效 |
| `recording` | 用户说话中（VAD 已触发或按住） | 波纹 + 转录浮层 |
| `processing` | 用户话音落下，等待 AI 首帧 | 环形进度"对方正在思考" |
| `degradedText` | 纯文本降级模式 | 状态条提示（UI 文档第五章） |
| `ended` / `failed` | 会话结束 / 不可恢复错误 | 退出确认 / 错误态 |

### 2.2 事件表

| 事件 | 来源 | 关键处理 |
|---|---|---|
| `sessionStartTap` | UI | idle → connecting，调 `POST /sessions` 取票据 |
| `socketReady` | Transport | connecting → aiSpeaking（AI 开场白） |
| `aiAudioEnd` | Transport | aiSpeaking → waitingUser |
| `vadSpeechStart` / `holdStart` | AudioEngine / UI | waitingUser → recording；**若当前 aiSpeaking：触发打断事件** |
| `vadSpeechEnd` / `holdEnd` | AudioEngine / UI | recording → processing |
| `aiFirstAudioChunk` | Transport | processing → aiSpeaking |
| `badgeHit` | Transport（feedback.badge） | **不进状态机**，旁路直达 Store（不阻塞任何状态流转） |
| `networkLost` | Transport | 任意 → 3 秒重连窗口；超时 → degradedText |
| `interruptedBySystem` | 系统（来电） | 挂起当前状态，恢复后回 waitingUser |
| `textMessageSent` / `textReplyReceived` | UI / APIClient | 仅 degradedText 内部循环（文本降级走 `POST /sessions/:id/messages`，见后端契约） |
| `endTap` | UI | 底部条确认（UI 文档 4.2）→ ended，触发 session.end |

### 2.3 实现与测试要求

- 实现为 `reduce(state, event) -> (state, [SideEffect])`，副作用（发送 interrupt、停播、上报）由 SpeechSessionClient 解释执行——状态机本身零 IO，可完全离线单测；
- **单测硬要求**：状态 × 事件矩阵全覆盖（含非法组合的忽略行为）、打断竞态序列（`vadSpeechStart` 与 `aiAudioEnd` 交错到达）、重连窗口内的重复 `socketReady` 幂等；
- 所有状态迁移发埋点（状态对 + 耗时），W1-W2 压测与内测 badcase 归档共用此数据。

---

## 三、AudioEngine 详设（人写）

### 3.1 音频图

```
采集：AVAudioEngine inputNode(16kHz 单声道 tap) → Opus 编码（火山 SDK 优先）→ Transport 上行
播放：Transport 下行 Opus → 解码 → AVAudioPlayerNode（流式 append，分段调度）
```

- `AVAudioSession`：category `.playAndRecord` + mode `.voiceChat`（启用 Voice Processing I/O 硬件 AEC）；采样率与 Opus 参数与火山要求严格对齐（技术方案坑清单 #3：接入第一天做音频回环自测）；
- 播放缓冲策略：低延迟优先，缓冲 ≤100ms；被丢弃帧的解码队列随打断清空。

### 3.2 打断实现（200ms 红线）

```
本地能量检测超阈值（或用户点按）
  → ① 立即 stopPlayer + 清空待播队列（本地，不等服务端）
  → ② 上报 vadSpeechStart → 状态机
  → ③ Transport 上行 interrupt 帧
```

①必须在音频线程同步完成，②③异步——顺序不可调换，这是 200ms 指标的实现保证（技术方案 3.2）。

### 3.3 系统干扰处理

| 场景 | 行为 |
|---|---|
| 来电 / CallKit 中断 | `AVAudioSession.interruption` → 状态机 `interruptedBySystem`；结束后自动恢复采集 |
| 路由切换（耳机拔出） | 不中断会话；外放路径下 AEC 仍生效（`.voiceChat` 保证），内测必测 |
| 切后台 | 说的房间：暂停会话并提示（语音对话无法后台进行）；每日一读：允许后台播放（独立播放器实例，`audio` background mode） |
| 麦克风权限被拒 | 非阻断说明条（UI 文档第五章），闪测/对话/跟读共用一次授权 |

### 3.4 可测性

AudioEngine 对外只暴露协议 `AudioEngineProtocol`（start/stop/playChunk/interruptNow/事件回调），单测用内存环回实现替代真实硬件；真实硬件行为靠内测清单覆盖（第七章）。

---

## 四、WSS 协议客户端处理（Agent 起草）

### 4.1 连接生命周期

1. `POST /sessions` → `wss_url` + 一次性票据（60s）；
2. `URLSessionWebSocketTask` 连接，首帧发送票据完成握手；
3. 心跳：30s ping；连续 2 次失败判 `networkLost`；
4. 重连：3 秒窗口内自动重连 + 携 `session_id` 恢复（后端网关无状态、上下文在 Redis，技术方案六章）；超时进入文本降级。

### 4.2 帧处理

- 控制帧（JSON，Codable）：`session.start` / `user.speech.start|end` / `ai.text.delta` / `ai.audio.chunk` / `interrupt` / `feedback.badge` / `session.end`——与后端契约测试共用同一 schema（防静默变更）；
- 音频帧：二进制 Opus，20ms 帧，每帧携带**服务端递增序列号**；
- **打断丢帧规则**：本地记录打断时刻的"当前最大序列号"，此后到达的、序列号 ≤ 该值的音频帧直接丢弃——解决"打断时 TTS 流仍在下发"的竞态（技术方案 2.4）；该逻辑独立成纯函数，单测覆盖边界（相等、回绕不存在、空队列）；
- `ai.text.delta` 流式拼接进当前 AI 气泡；`feedback.badge` 直达 Store 触发徽章动效（不经状态机）。

### 4.3 转录浮层衔接

用户话轮的本地即时转录（`user.speech.*` 期间的增量文本）显示于浮层；话轮结束事件到达后 200ms 内归位气泡、浮层淡出（UI 文档 4.2 衔接规则），归位动作由 Store 编排，Transport 不参与。

---

## 五、本地存储（SwiftData，Agent 起草）

### 5.1 模型

| 模型 | 内容 | 同步策略 |
|---|---|---|
| `CachedBlock` | 话术块镜像（含状态灯、调度时间） | 拉取增量（`updated_at` 游标）；服务端权威 |
| `SessionRecord` | 会话列表与回顾摘要（详情懒加载） | 同上 |
| `DailyReadCache` | 每日一读正文与音频本地路径 | 每日拉取 + 7 天保留 |
| `UserPrefs` | 音色偏好、通知开关、说话模式（自动/按住） | 本地写，登录态下上报 |

### 5.2 规则

- 读本地优先：语料库/历史/每日一读首屏永远来自本地，后台同步完成后静默刷新（弱网可浏览，PRD 9.2）；
- 冲突处理：客户端无写入冲突源（话术块编辑走服务端接口后回刷），本地只做镜像——**不做本地离线写**，MVP 不承担同步复杂度；
- 录音文件：本地临时目录，上传成功后标记，7 天后清理（技术方案 6.2）。

---

## 六、系统集成点（Agent 起草）

| 集成项 | 要点 |
|---|---|
| 麦克风权限 | 首次进入说的房间前申请；拒绝走非阻断说明条；三类用声场景共用结果 |
| 推送 | 闪测复习 / 每日一读 / 话题建议三类独立开关（对应设置页）；回顾生成完成走**静默推送**触发刷新——**静默推送只做加速**：APNs 对 content-available 有节流，客户端进入工作台/回顾页时对未就绪的 review 必须主动拉取兜底（与后端 5.2 对齐） |
| 锁屏与媒体控件 | 每日一读与 AI 朗读支持锁屏/控制中心媒体控件（MPNowPlayingInfoCenter + MPRemoteCommandCenter）：朗读场景用户会锁屏听，仅后台播放不够（UI 文档 4.6） |
| 深色主题 | 深色默认（UI 文档 2.1），跟随系统仅作反转；色值全部进 Design Tokens 常量，禁止页面内硬编码 |
| 动态字体 | 正文支持至 130%，布局用系统字体度量，不锁死行高像素值 |
| 无障碍 | VoiceOver 标签与徽章公告按 UI 文档第七章；所有纯视觉元素（波形/状态灯/徽章）必须有 accessibilityLabel |

---

## 七、客户端测试策略

| 层级 | 对象 | 标准 |
|---|---|---|
| 单测（最高优先） | SpeechSession 状态机、序列号丢帧、打断时序 | 状态矩阵全覆盖；打断竞态序列测试 |
| 单测 | AudioEngine（协议 mock）、CorpusStore 同步 | 关键路径覆盖；环回实现验证帧流转 |
| 快照测试 | 工作台/说的房间/回顾页/闪测四页 × 深浅主题 | 抽查制，防 UI 回归 |
| 契约测试 | WSS 帧与后端 schema 对齐 | 后端同步变更即红 |
| 内测人工清单 | 外放 AEC、打断 200ms 体感、来电恢复、切后台、弱网降级、耳机/外放切换 | W8 内测逐项签字 |

---

## 八、Agent 任务拆解清单（与团队分工文档第五章对齐）

**顺序纪律**：状态机与 AudioEngine 接口在 W3 前半周冻结；接口冻结前不下发依赖它们的 UI 任务。

| 任务单 | 内容 | 周次 | 依赖 | 验收要点 |
|---|---|---|---|---|
| C1 | SocketTransport + 帧编解码 + 序列号丢帧 | W3 | 后端 WSS schema | 契约测试通过；丢帧单测全绿 |
| C2 | APIClient 全量接口 + 票据流程 + 游客身份与归并（后端 G4） + 文本降级 messages 接口 | W3 | 后端 REST 契约 | 错误归一化单测；归并幂等用例 |
| C3 | 说的房间 UI（气泡流/说话键/浮层/徽章/素材栏） | W3-W4 | 状态机接口冻结 | 快照测试 + 状态机驱动的全状态走查 |
| C4 | 创建练习弹层 + 工作台 | W4 | C2 | UI 文档 4.1 还原；一句话默认 Tab |
| C5 | 回顾页（渐进加载 + 双栏对照 + 炼化卡） | W5 | C2 | 两层到达时序单测（骨架→内容） |
| C6 | CorpusStore / SyncEngine + 语料库页 | W6 | C2 | 弱网本地优先场景测试 |
| C7 | 每日一读页 + 跟读（不出分） | W6 | C6 | 后台播放配置验证 |
| C8 | 闪测卡流 + 判定等待态 + 结算页 | W7 | C2 | 判定等待/超时兜底交互走查 |
| C9 | 话题建议页 + 设置页 + 订阅占位（远程开关隐藏） | W7 | C2 | 远程开关行为测试 |
| C10 | 新手引导 + 空态/错误态全集 | W8 | 全部 | UI 文档第五章全场景走查 |

每个任务单按团队分工文档 4.1 的四要素格式下发（上下文链接/契约/验收/禁区）。

---

*评审重点：第二章状态机事件表与打断竞态、第三章打断三步顺序、第四章序列号丢帧规则、第八章任务顺序纪律。*
