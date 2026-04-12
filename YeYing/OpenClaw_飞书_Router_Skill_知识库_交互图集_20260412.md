# OpenClaw × Feishu × Router 交互图集（含 Skill/知识库扩展）

> 依据文档：
> - `OpenClaw_007_独立飞书读全群仅AT回复复现手册_20260403.md`
> - `OpenClaw_008_独立飞书机器人Router客户端拦截攻坚复盘_20260407.md`
>
> 目标：统一说明 OpenClaw 与 Feishu、Router 及未来 Skill/知识库插件的交互边界与执行链路。

---

## 1) 用例图（Mermaid 兼容表示）

```mermaid
flowchart LR
    A1[群成员]
    A2[机器人管理员]
    A3[运维/平台管理员]
    A4[外部能力提供方\nSkill/知识库]

    subgraph SYS[OpenClaw 独立实例]
      U1[接收群消息与媒体]
      U2[未提及机器人仅记录历史与媒体缓存]
      U3[提及机器人触发回复]
      U4[调用 Router 推理]
      U5[执行 Skill 工具调用]
      U6[检索知识库上下文]
      U7[返回群回复/结果]
      U8[配置 mention gate 与白名单]
      U9[查看日志并诊断故障]
    end

    A1 --> U1
    U1 --> U2
    A1 --> U3
    U3 --> U4
    U3 --> U5
    U3 --> U6
    U4 --> U7
    U5 --> U7
    U6 --> U7
    A2 --> U8
    A2 --> U9
    A3 --> U9
    A4 --> U5
    A4 --> U6
```

---

## 2) 架构图（当前 + 未来扩展）

```mermaid
flowchart LR
    subgraph FEI[Feishu 域]
      FU[群成员]
      FP[Feishu 平台\n事件/消息/媒体下载]
      FU <--> FP
    end

    subgraph OC[OpenClaw 独立实例]
      GW[openclaw-gateway]
      PLG[Feishu Plugin<br/>local-plugins/feishu]
      POL[Reply Policy<br/>requireMention + allowlist]
      AG[Agent Orchestrator<br/>main-larkbot]
      CTX[Session/History Store\nstate/agents/*]
      MED[Media Cache\nstate/media/*]
      BUS[Plugin/Tool Bus]
      SK[Skill Runtime<br/>未来扩展]
      KB[Knowledge Plugin<br/>未来扩展]
      OBS[日志与诊断\ngateway.out]

      GW <--> PLG
      PLG --> POL
      POL --> AG
      AG <--> CTX
      AG <--> MED
      AG <--> BUS
      BUS <--> SK
      BUS <--> KB
      GW --> OBS
      AG --> OBS
    end

    subgraph RT[模型与路由域]
      RP[Router Provider\nopenai-responses]
      RAPI[Router API\n/v1/responses]
      M1[gpt-5.4 / gpt-5.3-codex]
      RP --> RAPI --> M1
    end

    FP --> PLG
    PLG --> FP
    AG <--> RP
```

---

## 3) 序列图（核心消息链路）

```mermaid
sequenceDiagram
    autonumber
    participant U as 群成员
    participant F as Feishu
    participant P as Feishu Plugin
    participant O as OpenClaw Agent
    participant S as Session/Media Store
    participant K as Skill/知识库插件(可选)
    participant R as Router /v1/responses
    participant M as 模型(gpt-5.x)

    U->>F: 发送文本/图片/文件
    F->>P: 事件推送
    P->>O: 标准化消息
    O->>S: 写入历史与媒体引用

    alt 未@机器人
      O-->>P: 命中 requireMention gate
      P-->>F: 不回复（仅记录）
      F-->>U: 群内静默
    else @机器人
      O->>S: 读取 pending 媒体/引用消息
      opt 需要外部能力
        O->>K: Skill 调用 / 知识检索
        K-->>O: 工具结果/检索片段
      end
      O->>R: 请求推理(含headers与上下文)
      R->>M: 转发模型请求
      M-->>R: 推理结果
      R-->>O: 响应内容
      O-->>P: 生成回复
      P-->>F: 发送消息
      F-->>U: 返回答案
    end
```

---

## 4) 流程图（消息处理与故障分流）

```mermaid
flowchart TD
    A[收到 Feishu 事件] --> B{事件是否合法\n群白名单内?}
    B -- 否 --> B1[丢弃并记审计日志] --> Z[结束]
    B -- 是 --> C[解析消息体\n文本/图片/文件/引用]
    C --> D[写入 history + pending media]
    D --> E{是否 @ 机器人?}

    E -- 否 --> E1[命中 only-@ 策略\n不回复] --> Z

    E -- 是 --> F[拼装上下文\n消费 pending + 引用媒体]
    F --> G{是否需要 Skill/知识库?}
    G -- 是 --> G1[执行工具调用/检索] --> H
    G -- 否 --> H[调用 Router /v1/responses]

    H --> I{Router 返回状态}
    I -- 200 --> J[生成回复并发回 Feishu] --> Z
    I -- 400 Client not allowed --> K[检查 provider.headers\n与客户端签名]
    I -- 401/403 --> L[检查 apiKey/权限/配置文件是否串号]
    I -- 503 --> M[标记上游波动\n重试或降级提示]
    K --> N[双写校验 openclaw.json + state/models.json] --> O[重启实例后回归测试] --> Z
    L --> O
    M --> O
```

---

## 5) 交互原则（从两份复盘抽象）

1. OpenClaw 是编排中枢：负责策略决策、上下文管理、工具编排、模型调用。
2. Feishu 插件是 I/O 入口：负责收发消息与媒体，不负责最终推理决策。
3. Router 是模型路由与策略边界：`/models` 可用不代表 `/responses` 可用，必须以推理接口验证。
4. Skill/知识库插件应挂在 Tool Bus：避免和 Feishu/Router 强耦合，便于后续增删扩展。
5. “读全群仅@回复”本质是策略分流：未@走“记录路径”，@走“推理路径”。
