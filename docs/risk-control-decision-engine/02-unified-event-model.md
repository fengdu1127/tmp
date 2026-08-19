# 风控决策引擎系统设计 · 统一事件模型

> 上篇：[01-overall-architecture.md](./01-overall-architecture.md)（总体架构）。本文展开接入层的核心契约——统一事件模型。
>
> 设计目标：**任何业务方接入风控，只需要做一次字段映射，引擎侧不为新业务改代码。**

## 1. 设计原则

1. **分层：通用骨架 + 领域扩展**。不为所有业务设计一张大宽表——支付、登录、营销的字段差异太大，大宽表最终会变成几百个可空字段的垃圾场。正确做法是：一层所有事件共有的骨架（envelope），一层按事件类型定义的领域字段（payload），schema 分开管理。
2. **主体（Entity）是一等公民**。风控的本质是围绕"主体"累积风险画像：用户、设备、IP、卡、商户、收货地址……事件模型必须显式声明"这个事件涉及哪些主体、各扮演什么角色"，特征平台才能按主体维度做聚合。
3. **只传事实，不传结论**。事件里放"发生了什么"（金额、IP、时间），不放业务方自己的判断（"疑似风险用户=true"）。业务方的结论如果有价值，作为独立的信号字段单独建模，口径由风控侧定义。
4. **不可变 + 可版本化**。事件一旦上报不可修改；schema 变更走版本管理，向后兼容（只加可选字段，不改语义、不删字段）。

## 2. 结构设计

整体分四段：**envelope（信封）→ entities（主体）→ context（环境上下文）→ payload（领域数据）**。

```jsonc
{
  // ── 1. Envelope：所有事件必填的骨架 ──
  "event_id": "evt_7f3a...",            // 全局唯一，幂等去重用（业务方生成，格式约束）
  "event_type": "payment.card_charge",  // 分层命名：<域>.<动作>，注册制，不允许自由发挥
  "event_version": "1.2",               // 对应 payload schema 的版本
  "occurred_at": 1755590400123,         // 业务发生时间（毫秒 UTC）
  "received_at": 1755590400156,         // 风控收到时间（引擎侧填，用于测时钟偏移）
  "source": {
    "app_id": "checkout-service",       // 上报方身份（鉴权后填入，不信任请求体）
    "channel": "ios_app",               // 渠道：web / ios / android / api / pos
    "trace_id": "..."                   // 打通全链路排查
  },
  "mode": "sync",                       // sync=要决策结果 / async=仅记录（如事后补报）

  // ── 2. Entities：这个事件涉及的主体及其角色 ──
  "entities": [
    { "type": "user",     "id": "u_1001",         "role": "actor" },        // 操作发起者
    { "type": "device",   "id": "dfp_ab3f...",    "role": "actor_device" },
    { "type": "card",     "id": "card_tok_9x...", "role": "instrument" },   // 支付工具
    { "type": "merchant", "id": "m_2002",         "role": "counterparty" }
  ],

  // ── 3. Context：环境上下文（跨事件类型通用） ──
  "context": {
    "ip": "203.0.113.7",
    "user_agent": "...",
    "device_fingerprint_raw": { /* 指纹 SDK 原始采集，透传给指纹服务解析 */ },
    "geo": { "lat": 31.23, "lng": 121.47, "accuracy_m": 50 },
    "session_id": "sess_..."
  },

  // ── 4. Payload：按 event_type 定义的领域字段 ──
  "payload": {
    "amount": { "value": 128800, "currency": "CNY", "exponent": 2 },  // 最小货币单位整数，禁止浮点
    "order_id": "ord_...",
    "card_bin": "622202",
    "is_first_purchase": false
  }
}
```

### 2.1 Envelope

- `event_type` 用 `域.动作` 两级命名（`account.login`、`account.register`、`payment.card_charge`、`withdrawal.request`、`marketing.coupon_claim`），**注册制**：新事件类型必须在 schema registry 登记后才能上报，未注册类型直接拒收。这是防止事件模型腐化的第一道闸。
- `occurred_at` 和 `received_at` 分开存：客户端时钟不可信，聚合特征一律用 `received_at`，`occurred_at` 仅作参考；两者偏差本身就是一个不错的风险特征。
- `event_id` 由业务方生成并保证幂等——重试场景下同一 `event_id` 必须返回同一决策结果，而不是重新决策（否则限频类规则会把重试算成多次行为）。

### 2.2 Entities（设计重心）

用「**类型 + ID + 角色**」的数组，而不是固定字段（`user_id`、`merchant_id`……）。原因：

- 同一类型可以出现多次（转账事件有两个 user：`actor` 和 `counterparty`）；
- 新增主体类型（比如加"收货地址"实体）不需要改骨架；
- 特征平台可以按 `(entity_type, entity_id)` 统一做聚合——"该 IP 近1小时事件数"和"该卡近1小时事件数"是同一套计算框架。

约束与规范：

- 实体类型和角色都是受控枚举，注册管理。常用角色：`actor`（发起者）、`counterparty`（对手方）、`instrument`（支付工具）、`beneficiary`（受益方）、`actor_device`。
- 敏感标识（卡号、手机号、邮箱）**入模前先标准化 + tokenize/哈希**：引擎和特征存储只见 token，原文只留在有权限管控的映射服务里。这同时解决合规问题和"同一卡号不同格式算成两个主体"的问题。

### 2.3 Context

- 通用环境信息放这里而不是 payload，因为几乎所有事件都有、且特征口径统一（"IP 维度聚合"不应该关心事件类型）。
- 设备指纹分两个字段：SDK 原始采集包（透传给指纹服务）和解析后的 `device_fingerprint_id`（作为 entity）。接入层调指纹服务完成解析，业务方不参与。

### 2.4 Payload

- 每个 `event_type` 一份 JSON Schema，登记在 registry 里，接入层按 schema 强校验（类型、必填、枚举、金额格式），**校验失败在接入层就拒绝**，不让脏数据进引擎——否则脏数据会污染特征，排查成本极高。
- 金额一律「最小货币单位的整数 + 币种 + 精度」，禁止浮点；跨币种聚合特征由特征平台统一换汇，不在事件里做。
- 字段命名规范化（snake_case、布尔用 `is_/has_` 前缀、时间统一毫秒 UTC），registry 里每个字段带中文描述和负责人，这些元数据后续直接喂给规则编辑器做自动补全。

## 3. Schema Registry 与演进管理

Registry 管三类对象：

| 对象 | 内容 |
|---|---|
| 事件类型 | envelope 约束 + payload schema + 版本历史 |
| 实体类型 / 角色枚举 | 受控枚举及其语义定义 |
| 字段字典 | 跨事件复用的公共字段定义（如 `amount`、`ip`），避免每个 payload 重复定义且口径漂移 |

版本规则：

- 加可选字段 → 小版本，向后兼容，老接入方无感；
- 改类型/语义/删字段 → 大版本，新旧并行一段时间，引擎按 `event_version` 路由校验；
- **永远不复用字段名表达新语义**。线上可能有大量规则引用着老字段，语义悄悄变化会让这些规则无声地判错——不报错、只默默做错误决定，是风控最可怕的事故类型。

治理流程：每个事件类型有 owner（业务方）和 reviewer（风控方），schema 变更走 PR 式审批；上线后 registry 自动生成接入文档和各语言的 SDK/DTO。

## 4. 两个容易踩的坑

1. **不要让"决策请求"和"事件"变成两套模型**。同步决策请求就是一个 `mode=sync` 的事件；异步补报、批量导入历史数据用同一模型 `mode=async`。一套模型贯穿同步决策、实时特征计算、离线回测——这正是"实时/离线口径一致"的前提：回测时重放的就是当初的事件本身。
2. **实体关系不要塞进事件里**。"设备关联了多少账户"这类图关系是特征平台从事件流中**推导**出来的，事件只忠实记录"本次事件中这几个实体同时出现"。事件负责事实，图引擎负责关系。

## 5. 接入流程

一个新业务的接入流程最终收敛成：

```
在 registry 登记事件类型
  → 定义 payload schema
  → 映射实体和上下文字段
  → 联调校验
  → 影子上报观察数据质量
  → 转正
```

全程不需要引擎侧发版。
