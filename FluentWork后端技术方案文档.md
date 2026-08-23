# FluentWork 后端技术方案设计文档

**版本**：V1.2　**日期**：2026年8月　**对应**：PRD V1.4 / 技术方案 V3.2 / Prompt 工程与语料库设计文档 V1.2　**评审角色**：后端技术负责人

> 本文档覆盖服务端全部设计：技术栈、服务架构、数据库、接口契约、核心链路时序、异步任务管线、订阅计费、稳定性与部署。前端与语音链路细节见技术方案 V3.2，Prompt 与调度算法见专项文档，本文只定义服务端如何承载它们。

---

## 一、技术栈选型

| 项 | 选型 | 理由 |
|---|---|---|
| 语言/框架 | Go 1.23 + Gin | 团队通用栈、协程模型天然匹配高并发长连接与 IO 密集场景、部署简单 |
| 语音网关 | Go + gorilla/websocket | 与业务服务同语言，降低维护成本 |
| 数据库 | MySQL 8.0 | 业务数据主体；InnoDB 行级锁足够当前并发 |
| 缓存/队列 | Redis 7（会话态、限流、调度队列快照） | 一器多用，MVP 不引入 Kafka/RocketMQ |
| 异步任务 | Redis Stream + 自研 Worker（消费组） | 任务量级小（日均千级），MQ 中间件后置 |
| 对象存储 | 火山引擎 TOS（录音、TTS 音频、每日一读音频） | 与语音供应商同云，内网传输快、免跨云流量费 |
| 部署 | 单云主机 + Docker Compose 起步，预留容器化 | 内测期规模无需 K8s |
| 配置中心 | DB 配置表 + Redis 缓存 + 管理后台 | Prompt 与调度参数热更新 |
| 可观测 | Prometheus + Grafana + Loki（日志）+ 自研业务埋点表 | 轻量组合，不引入重型 APM |

**语言决策说明**：AI 编排层若后续需要复杂文本处理，Python 有生态优势，但 MVP 阶段全部 AI 调用都是 HTTP API 编排，Go 完全胜任；保持单语言栈对 3-5 人团队的收益远大于生态差异。

---

## 二、服务架构：模块化单体 + 独立语音网关

```
                    ┌──────────────┐
   iOS App ──WSS──► │ 语音会话网关   │──► 火山引擎（端到端/ASR/TTS）
      │             │ voice-gateway │
      │ HTTPS       └──────┬───────┘
      ▼                    │ 内部 RPC（会话落库/命中检测触发）
┌─────────────┐            │
│  API 网关层   │◄───────────┘
│ (鉴权/限流/   │
│  埋点/计费)   │
└──────┬──────┘
       ▼
┌─────────────────────────────────────────┐
│  app-server（模块化单体，Go）              │
│  ├─ account   账号与鉴权                  │
│  ├─ material  素材                       │
│  ├─ session   练习会话与回顾               │
│  ├─ corpus    语料库与调度                 │
│  ├─ content   每日一读/话题卡              │
│  ├─ drill     闪测                       │
│  ├─ billing   订阅与内购校验（V1.1 启用，MVP 预留接口与表结构）   │
│  ├─ notify    推送                       │
│  └─ ai-worker 异步任务（评价/炼化/生成；发音评测 V1.1） │
└─────────────────────────────────────────┘
   MySQL / Redis / TOS / 火山引擎 AI API
```

**为什么语音网关独立部署**（重申并落到服务端视角）：
1. 长连接服务的发布不能打断进行中的会话——独立部署后可用连接 draining 滚动发布，单体按请求级服务随时重启；
2. 资源画像不同：网关是连接密集+低 CPU，单体是 IO 密集，分开伸缩；
3. 故障隔离：网关挂掉不影响语料库浏览等纯 HTTPS 功能。

**网关与单体的边界**：网关只做协议转换、会话状态机、流转发、旁路触发；一切持久化和业务判断（命中检测的实际执行、计费扣减）回调单体内部接口。网关无状态化设计——会话上下文存 Redis，网关实例可随意重启，客户端 3 秒内重连恢复会话。

---

## 三、数据库设计

### 3.1 表清单与关键字段

```sql
-- 账号（支持游客身份：演示主路径免注册，见 3.2 设计要点 5）
users(id, email NULL, phone NULL, device_id NULL, is_guest,
      pwd_hash NULL, status, created_at)
auth_tokens(id, user_id, refresh_token_hash, expires_at)

-- 素材
materials(id, user_id, source_type, raw_text, refine_json, status, deleted_at)

-- 会话与话轮
practice_sessions(id, user_id, material_id, scene_type, status,
                  duration_sec, review_json NULL, created_at)
utterances(id, session_id, seq, speaker, text, asr_confidence,
           audio_url, hit_block_ids JSON, pron_score NULL,
           pron_detail JSON NULL, created_at)
  -- pron_score / pron_detail 字段预留，随发音评测 V1.1 启用

-- 语料库（详见专项文档，此处列服务端视角补充）
phrase_blocks(id, user_id, intent_zh, expression_en, anchor_user_said,
              scene_tag, function_tag, state, success_streak,
              next_due_at, ease_factor, real_use_count,
              is_favorite, pinned_at NULL,          -- 收藏/置顶（PRD F3）
              source_session_id, deleted_at, created_at, updated_at)
  KEY idx_due(user_id, next_due_at), KEY idx_scene(user_id, scene_tag),
  KEY idx_fav(user_id, is_favorite, pinned_at),
  FULLTEXT KEY ft_expr(expression_en, intent_zh)

drill_records(id, user_id, block_id, drill_type, semantic_pass,
              pronunciation_score, response_ms, created_at)
  KEY idx_block(block_id, created_at), KEY idx_user(user_id, created_at)
  -- pronunciation_score 字段预留（V1.1），MVP 期恒为 NULL

-- 内容生成
daily_reads(id, user_id, gen_date, title, body, audio_url,
            used_block_ids JSON, source_refs JSON, read_score NULL,
            UNIQUE KEY uk_user_date(user_id, gen_date))
topic_cards(id, user_id, topic, scene_desc, block_ids JSON,
            opener_lines JSON, source_refs JSON, checkin_status,
            checkin_score NULL, created_at)

-- 订阅（表结构随 MVP 建立，商业化能力随 V1.1 上线）
subscriptions(id, user_id, plan, platform, status,
              expires_at, trial_used, original_tx_id, created_at, updated_at)
iap_receipts(id, user_id, platform, tx_id UNIQUE, payload, verify_result, created_at)

-- 配置与运营
prompt_configs(id, task_type, version, content, schema_json, status, gray_pct, created_at)
sched_configs(id, param_key, param_value, updated_at)   -- 调度参数热更新

-- 成本与埋点
ai_cost_logs(id, user_id, task_type, model, tokens_in, tokens_out,
             audio_sec, cost_fen, created_at)
  KEY idx_user_date(user_id, created_at)

-- 删除回执（3.2 级联删除的每步回执，支撑 72h SLA 与进度可查）
deletion_receipts(id, task_id, user_id, entity_type, entity_id,
                  step, status, vendor_cleaned, receipt_json, created_at)
  KEY idx_task(task_id, step)
```

### 3.2 设计要点

1. **软删除 + 级联任务**：`deleted_at` 软删除先行（误删可恢复 72h 宽限），级联硬删由 MQ 任务执行并写 `deletion_receipts` 回执表——对应 PRD 隐私承诺的工程落地；
2. **JSON 列的纪律**：只存"读多写少、无需检索"的结构（refine_json、review_json、hit_block_ids）；需要筛选的字段（scene_tag 等）必须是独立列；
3. **成本埋点独立表**：每次 AI 调用同步落一条 `ai_cost_logs`——这是单位经济模型的数据基础，W1 就上线，不允许后补；
4. **锚点字段的隐私边界（V1.2 定案）**：`anchor_user_said` 是从用户话轮提炼的表达片段，**定性为衍生数据而非敏感原始数据**（与素材原文/转录/录音不同级），明文落库并参与 FULLTEXT 检索——若加密则全文检索失效，权衡后不加密；代价是两件事必须做到：素材删除时锚点级联脱敏为"[已删除]"（对齐 PRD F4）；隐私政策中显性告知"练习内容的提炼片段会明文存储"。此口径不允许悬而未决，提审材料引用本条；
5. **游客身份与数据归并（V1.2 新增，对齐 PRD G4）**：首次用户以设备身份（`device_id`，`is_guest=1`）进入演示闭环，无需注册；业务表（materials/sessions/phrase_blocks）的 `user_id` 允许为游客 ID；用户首次"保存语料"触发注册后，`POST /account/merge` 把游客数据归并到正式账号——实现要点：以 `device_id` 幂等（重复调用不重复归并）、归并后游客记录置为已合并状态（不物理删除，留审计）、话术块的 `next_due_at` 与调度状态原样迁移；**W3 建 session 模块时必须带上，否则演示练习丢在游客态或被迫提前登录破坏主路径**。

---

## 四、接口契约（REST 核心清单）

约定：`/api/v1/` 前缀；鉴权 `Authorization: Bearer`；错误统一 `{code, message, request_id}`；分页 cursor 制（不用 offset，语料库数据会持续写入）。

| 模块 | 方法与路径 | 说明 |
|---|---|---|
| 账号 | POST /auth/email-code | 发送邮箱验证码（限流：同邮箱 1/min） |
| | POST /auth/sms-code | 发送手机短信验证码（国内主通道，限流：同号码 1/min；V1.3 新增） |
| | POST /auth/guest | 签发游客身份（device_id 绑定，免注册，支撑演示主路径；V1.4 新增） |
| | POST /auth/login | 验证码登录（邮箱/手机双通道），返回 access+refresh token |
| | POST /account/merge | 注册后将游客期数据归并到账号（幂等，以 device_id 去重；V1.4 新增） |
| 素材 | POST /materials | 创建素材，触发异步提炼，返回 material_id |
| | GET /materials/:id | 轮询提炼结果（refine_json ready 后可开始练习） |
| | DELETE /materials/:id | 单条素材删除（PRD A4/F4）：级联策略同账号级删除——衍生话术块可选级联删除或锚点脱敏保留，按用户选择 |
| 会话 | POST /sessions | 创建练习会话，返回 session_id + WSS 接入地址 + 临时票据 |
| | GET /sessions?page= | 历史列表（工作台） |
| | GET /sessions/:id/review | 回顾页数据（转录+评价+对照+发音，异步任务完成后就绪） |
| | POST /sessions/:id/abandon | 放弃练习（B6） |
| | POST /sessions/:id/messages | 文本降级对话（三级降级末级，PRD 可用性承诺的落地载体；V1.4 新增）：提交用户文本，返回 AI 文本回复；非流式，≤3s；仅网关判定降级后开放，语音可用时返回 409 |
| 话术块 | GET /corpus/blocks?scene=&func=&kw=&cursor= | 语料库列表 |
| | PUT /corpus/blocks/:id | 编辑（D2） |
| | POST /corpus/blocks/:id/favorite | 收藏/取消收藏、置顶/取消置顶（PRD F3；V1.4 新增） |
| | DELETE /corpus/blocks/:id | 删除 |
| | POST /corpus/blocks/batch-accept | 全部入库 |
| 闪测 | GET /drill/round | 取本轮题目（调度引擎取数，返回 ≤10 块） |
| | POST /drill/judge | 提交作答音频 → 语义判定 + 发音分（模块化 ASR 链路） |
| 每日一读 | GET /daily-reads/today | 今日文章（不存在则同步触发生成，轮询就绪） |
| | POST /daily-reads/:id/follow-read | 跟读评分提交 |
| 话题卡 | GET /topic-cards | 当前有效话题卡 |
| | POST /topic-cards/:id/checkin | 聊后打卡（H3） |
| 订阅（V1.1 启用） | POST /billing/verify | 提交 StoreKit 收据，服务端校验并激活 |
| | GET /billing/status | 订阅状态与权益；MVP 期恒返回免费权益，接口先行预留 |
| 隐私 | DELETE /account/data | 一键删除全部数据（异步级联，返回任务 ID 可查进度） |

**会话建立的两次握手**：`POST /sessions` 返回的是 WSS 地址 + 一次性票据（60s 有效），客户端持票据连网关——网关校验票据后才建立语音会话。API Key 与火山凭证全程不出服务端。

---

## 五、核心链路时序

### 5.1 实时对话链路（说的房间）

```
App                voice-gateway            app-server         火山引擎
 │ POST /sessions ──────────────────────► 建会话,发票据
 │ ◄──── wss_url + ticket ─────────────── │
 │── WSS connect(ticket) ─►│ 校验票据      │
 │                         │── session.start(素材提炼+角色Prompt+音色) ──►│
 │◄═════════════ Opus 音频流双向转发 ══════════════════════►│
 │                         │                                  │
 │  user.speech.end        │── 旁路:该话轮ASR文本 ──► B7命中检测 │
 │                         │   (800ms超时,失败跳过)  ◄── 命中?   │
 │                         │── 命中→badge下发+下轮system注入 ──►│
 │  interrupt(开口) ──────►│── interrupt ─────────────────────►│
 │  (本地立即停播)          │   丢弃过期序列号帧                  │
 │  session.end ──────────►│── 会话落库+触发异步评价任务 ────────►│
```

延迟责任划分：端上 VAD 与打断（客户端）、流转发开销（网关，目标 <30ms 附加延迟）、模型链路（火山）。网关埋点三段延迟分开发上报，定位劣化环节不靠猜。

### 5.2 会话结束后的异步管线（MQ，Redis Stream）

```
session.end → 投递 session.finished 事件
   ├─► worker: 评价+炼化（合并一次旗舰模型调用，JSON Schema 校验，失败重试 1 次）
   └─► 全部完成 → session.status=reviewed → APNs 静默推送刷新客户端
```

（发音评测 worker 随 V1.1 接入，作为第三条并行分支，接口与字段已预留。）

- 任务幂等：以 `session_id + task_type` 为幂等键，重复消费直接跳过；
- 失败策略：重试 1 次仍失败 → 会话可回看转录，评价区前端展示"重试"入口（对应 UX 错误态），不重试风暴；
- 超时 SLA：评价任务 P90 ≤ 15s，超时告警；
- **完成通知的兜底（V1.2）**：APNs 静默推送只做加速——APNs 对 content-available 推送有节流，客户端进入工作台/回顾页时对未就绪的 review 必须主动拉取兜底（见 iOS 文档 §六）。

### 5.3 闪测调度取数

```
GET /drill/round:
  1. 取 due 块：state IN (new,training,automated) AND next_due_at <= now ORDER BY next_due_at LIMIT 10
  2. 不足 10 → 补充：近 7 天新入库且未练过的块
  3. 仍不足 → 返回实际数量（前端空态逻辑处理）
POST /drill/judge:
  音频 → 火山流式 ASR(短音频接口) → 文本 → 小模型语义判定（发音评测并行分支随 V1.1 接入）
  → 写 drill_record → 更新调度(streak/next_due/state) → 返回 {pass, reason}
```

判定与调度更新在同一事务，避免"判成功但调度没走"的脏状态。

### 5.4 每日一读与话题卡的批量生成

- 每日一读：凌晨 02:00 批处理，遍历活跃用户（近 7 天有练习），按 C5 Prompt 生成并合成 TTS 音频落 TOS；`uk_user_date` 唯一键防重复生成；用户当天首次打开若未生成（新用户），走同步生成兜底（约 5-10s，前端轮询）；
- 话题卡：每周一凌晨生成，仅对 `phrase_blocks ≥ 阈值` 的用户；生成失败不影响已有卡片。

---

## 六、订阅计费服务端（V1.1 启用，本章为预先设计）

> **节奏说明**：商业化在 MVP 验证通过后随 V1.1 启动；billing 模块代码随 V1.1 实现，但表结构、接口契约与权益校验点位在 MVP 期预留，避免 V1.1 返工。本章设计在 V1.1 直接执行。

1. **校验流程**：客户端 StoreKit 2 交易 → `POST /billing/verify` 上传签名交易 → 服务端 App Store Server API 验证 → 写 `iap_receipts`（tx_id 唯一，防重放）→ 更新 `subscriptions`；
2. **状态机**：`trial → active → expired / refunded / revoked`，状态变更以 App Store Server Notifications V2 推送为准（服务端必须接 V2 通知，不能只靠客户端上报，否则退款/续费失败感知不到）；
3. **权益判断**：所有额度型权益（每日对话次数、闪测轮次、语料库上限）由服务端在业务入口校验，`billing/status` 只用于展示——**额度判断逻辑绝不下放客户端**；
4. **额度计数**：Redis `INCR` + 每日过期 key（如 `quota:chat:{uid}:{date}`），快速路径不查库。

---

## 七、稳定性与降级设计

| 场景 | 策略 |
|---|---|
| 火山端到端故障 | 网关三级降级：端到端 → 模块化编排 → 纯文本对话（TTS 停用，文字聊天保核心练习不中断）；切换自动+可人工 |
| 小模型判定超时 | 闪测判定 3s 超时按"待人工"处理（不判错，避免误伤调度） |
| 发音评测链路故障（V1.1） | 非核心链路，故障不影响转录与评价最小闭环（熔断原则已覆盖） |
| Redis 故障 | 额度计数降级为放行（宁可多给额度不可阻断付费用户）；调度取数降级直查 MySQL |
| 突发流量 | API 网关按用户限流；语音会话按账号并发=1 限制（同一时刻只允许一条练习会话） |
| 数据备份 | MySQL 每日全量 + binlog；TOS 跨区域复制；每月一次恢复演练 |

**熔断原则**：任何非核心链路（发音评测、话题卡、每日一读）故障，都不能影响"说 → 转录落库"这条最小闭环。核心链路定义到接口级别，写进 oncall 手册。

---

## 八、安全与隐私工程

1. 素材原文、转录、录音 URL 字段级 AES-GCM 加密落库，密钥走 KMS，密钥与数据分权限管理；
2. TOS 录音全部私有读 + 预签名 URL（15 分钟过期），客户端不持有永久链接；
3. 邮箱验证码 6 位、5 分钟有效、错误 5 次锁定；refresh token 旋转（rotation）+ 复用检测（reuse detection，发现复用全家踢下线）；
4. 删除级联任务留回执，72h 完成，进度可查（见 3.2）；
5. 火山引擎企业协议确认"API 数据不用于模型训练"——采购条款，技术侧留存确认记录。

---

## 九、测试与发布策略

- **单测重点**：调度引擎（状态迁移全覆盖）、额度校验、级联删除、计费状态机（V1.1 实施）——这几类是"错了就丢数据/丢钱"的模块，覆盖率 ≥ 90%；其余业务接口以集成测试为主；
- **契约测试**：网关与单体的内部接口、与客户端的 WSS 协议帧，用 schema 契约测试防静默变更；
- **灰度发布**：Prompt 变更走配置中心灰度（见专项文档）；服务端发布用按 UID 尾号灰度；
- **压测**：W8 前完成网关长连接压测（目标单实例 3000 并发连接）与核心接口压测；供应商并发配额压测需与火山商务确认测试窗口，避免触发对方风控。

---

## 十、排期对齐（服务端视角）

| 周期 | 服务端任务 |
|---|---|
| W1-W2 | 骨架工程（CI/日志/埋点/ai_cost_logs）、网关 POC + 火山联调、压测报告 |
| W3-W4 | account/material/session 模块、WSS 协议全量、管理后台雏形（Prompt 配置） |
| W5 | ai-worker 管线（评价/炼化/命中检测；发音评测分支 V1.1）、review 接口 |
| W6 | corpus/drill 模块、每日一读批处理（StoreKit 校验与 V2 通知随 V1.1 实施，接口预留） |
| W7 | 调度引擎、话题卡批处理、APNs、限流与降级开关 |
| W8 | 压测、oncall 手册、TestFlight 内测支持 |

---

*评审重点：第二章网关与单体边界（重连恢复设计）、3.2 成本埋点纪律、第五章四条时序、第六章计费 V2 通知。*
