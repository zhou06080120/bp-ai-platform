# 平台总体架构

> 血压计 AI 客服平台：三个独立微服务组成"检索 → 问答 → 自进化"闭环。

## 1. 系统全景

```mermaid
flowchart TB
    subgraph 用户侧
        U["用户 / 前端 Web UI"]
    end

    subgraph 服务层
        BPA["blood-pressure-agent<br/>ReAct 客服 Agent<br/>(SSE 流式回答)"]
        PIPER["blodd-answer-piper<br/>FAQ 语义检索引擎<br/>(两阶段混合检索)"]
        BPM["bp-monitor-service-agent<br/>技能自进化监控 Agent<br/>(评估 + Skill 迭代闭环)"]
    end

    subgraph 数据与模型
        KB["FAQ 知识库<br/>30 FAQ × 1045 变体"]
        BGE["BGE 嵌入模型<br/>(ONNX INT8 量化)"]
        CE["CrossEncoder<br/>精排模型"]
        LLM["LLM<br/>(通义千问 qwen-max)"]
        SKILLS["Skill 知识库<br/>(向量化存储)"]
    end

    U -- "POST /query/manual|fc" --> BPA
    U -- "POST /query" --> PIPER
    U -- "GET /stream/events (SSE)" --> BPM

    BPA -- "faq_search 工具<br/>(本地混合检索)" --> KB
    BPA -- "推理" --> LLM
    PIPER -- "召回: Dense + BM25 → RRF" --> KB
    PIPER -- "编码" --> BGE
    PIPER -- "精排" --> CE
    BPM -- "LLM-as-a-Judge + 语义嵌入<br/>双模态评估" --> LLM
    BPM -- "向量检索动态挂载" --> SKILLS
    BPM -. "评估结论驱动 Skill<br/>自动 patch / 回滚" .-> SKILLS
```

三个服务**独立部署、独立演进**（各自仓库、各自 CI），通过统一工程基建保持同款口径。

## 2. 各系统职责

| 系统 | 定位 | 核心技术 | 关键指标 |
|---|---|---|---|
| [blodd-answer-piper](https://github.com/zhou06080120/blodd-answer-piper) | FAQ 语义检索引擎 | Dense(BGE)+BM25 双路召回 → RRF 融合 → CrossEncoder 精排；ONNX INT8 量化 | Top1 94.91% / Top3 98.61%；p50 延迟 34.7→16.5ms（2.28×） |
| [blood-pressure-agent](https://github.com/zhou06080120/blood-pressure-agent) | 面向用户的 ReAct 客服 Agent | 手写 ReAct 与 Function Calling 双实现、SSE 流式、5 异构工具、语义缓存、决策状态机护栏 | 203 单测 / 覆盖率 87%；BM25 门禁 Top1 94.1% |
| [bp-monitor-service-agent](https://github.com/zhou06080120/bp-monitor-service-agent) | 技能自进化监控 Agent | LLM-as-a-Judge + 语义嵌入双模态评估、Skill 向量检索动态挂载、异步任务解耦 + SSE 进度 | 评估器判别力门禁双 100%（好答放行/坏答拦截） |

## 3. 服务间关系

当前为**松耦合三服务**（可独立运行）：

1. **用户问答主链路**：用户 → `blood-pressure-agent`（ReAct 循环：思考 → 工具调用 → 观察 → 收敛作答），`faq_search` 工具内置与 blodd 同款的混合检索逻辑（BM25 + 语义向量 + RRF），SSE 逐步推送思考过程；
2. **检索兜底链路**：轻量客户端 / 标注员可直接调 `blodd-answer-piper` 的 `/query`，两阶段检索返回 Top-K 候选与置信度，低于阈值（0.82）自动拦截并建议就医；
3. **质量闭环链路**：`bp-monitor-service-agent` 周期性拉取会话样本，用 LLM-as-a-Judge + 语义嵌入双模态校验回答质量，评估结论驱动 Skill 自动 patch / 回滚（带快照精确回退），前端经 SSE 实时观察迭代进度。

## 4. 三仓同款工程基建

| 基建模块 | 作用 | 引入阶段 |
|---|---|---|
| `src/rate_limit.py` | 按 IP 滑动窗口限流（零依赖，/health /metrics 豁免） | P1 |
| `src/observability.py` | JSON 结构化日志 + request_id 贯穿 + Prometheus /metrics + LLM 调用埋点 | P2 |
| `src/compliance.py` | 医疗免责声明 + 输入校验 + Prompt 注入拦截 | P3 |
| `evals/gate.py` | CI 评估门禁：检索命中率 / 评估器判别力回归即拦截 | P2 |
| CI（GitHub Actions） | lint → test(3.10/3.12) → eval-gate → docker，五 job 全绿才放行 | P0-P2 |

## 5. 部署拓扑

```
docker compose up --build   # 本仓库 docker-compose.yml

宿主机端口          服务
8001  ──────────►  bp-monitor-service-agent   (SSE 进度流 + 评估任务)
8002  ──────────►  blood-pressure-agent       (SSE 问答流)
8003  ──────────►  blodd-answer-piper         (同步 JSON 检索)
```

- 三个镜像均由各自仓库 Dockerfile 多阶段构建（依赖层 + 运行层分离，模型权重运行时挂载不入镜像）；
- 密钥统一走环境变量注入（`.env`，见 `.env.example`），代码与镜像零硬编码；
- 每个服务自带 `/health`（探针豁免限流）与 `/metrics`（Prometheus 抓取，Grafana 可直接对接）。

## 6. 医疗合规设计

- **免责声明**：三个服务的对外响应统一附带 `MEDICAL_DISCLAIMER`（仅供健康教育参考，不替代执业医师诊疗），前端页脚同步展示；
- **低置信拦截**：检索置信度低于阈值时不编造答案，改为固定兜底话术 + 就医建议；
- **输入安全**：长度上限（500 字）+ 空输入拒绝 + 控制字符剥离；LLM 生成链路叠加 Prompt 注入模式拦截（指令覆盖 / 角色劫持 / 系统提示探测）；
- **角色边界**：拒绝"扮演医生"类角色劫持——AI 始终以"健康教育参考"定位作答。

## 7. 平台安全与水平扩展（P4）

### 7.1 请求防护链（中间件洋葱模型）

三服务统一注册顺序（先注册在内层）与执行顺序（由外向内）：

```
请求 → observability（X-Request-ID + Prometheus 记账，401/403/429 全统计）
     → rate_limit（限流先于鉴权，防 API Key 暴力枚举）
     → auth（X-API-Key 双级校验：普通 Key / 管理 Key）
     → 路由（管理端点再叠加 require_admin 守卫 → 403）
```

- **双级 Key**：`BP_API_KEYS`（业务查询）/ `BP_ADMIN_KEYS`（管理端点，隐含普通权限），逗号分隔可配多把；两者均未配置 = 本地开发模式（不鉴权，零影响）；
- **管理端点清单**：bp-monitor 的 `/reset`、`/approvals/*`；blood-pressure-agent 的 `/cache/clear`；blodd-answer-piper 的 `POST /annotations/{rid}`、`POST /iterate/apply`、`/admin/promote`、`/admin/refresh`。

### 7.2 审计留痕（audit.py）

所有管理端点写追加式 JSONL 审计日志（`logs/audit.jsonl`，`BP_AUDIT_PATH` 可覆盖）：
每条记录 `{ts, actor, action, target, detail, request_id}`——`actor` 为 Key 的 SHA-256 前 8 位指纹（脱敏不落明文），`request_id` 与结构化日志、Prometheus 指标三方关联，形成完整追责链路。

### 7.3 Skill 变更审批 gate（bp-monitor）

`BP_APPROVAL_MODE=manual` 时，评估器通过的技能进化动作不直接落盘：

```
Reviewer 产出动作 → 红线扫描（合规拦截）→ request_approval()
     ├─ auto（默认）：直接 apply_skill_action 落盘（向后兼容）
     └─ manual：入 pending 队列 + approval_pending SSE 事件 → 不落盘
                  ↓
        GET /approvals（admin 查看）→ POST /approvals/{id}/approve（复用同一
        apply_skill_action 落盘 + 重建索引 + 审计）| reject（仅标记 + 审计）
```

审批端点与后台 Worker 共用同一落盘入口 `apply_skill_action`，保证两条路径行为一致。

### 7.4 LLM 网关与水平扩展（blood-pressure-agent）

- **LLM 网关**（`llm_gateway.py`）：全部 LLM 调用收敛到单入口 `chat()`——内部统一完成 `llm_call_timer` 埋点、显式重试（`BP_LLM_MAX_RETRIES`）与主备切换（`BP_LLM_FALLBACK_*` 三项全配才启用备链路）；
- **会话外置**（`conversation_store.py`）：会话存储抽象为 `ConversationStore`（get/put/append/clear/delete/list_sessions），内存实现默认、Redis 实现读 `BP_REDIS_URL`（连接失败自动回落内存并告警）；Redis 由 compose 的 `scale` profile 提供（`docker compose --profile scale up`），会话出进程后即可多副本水平扩展，其余两个服务本身无会话状态、天然可扩展。
