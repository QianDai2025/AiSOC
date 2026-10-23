# AiSOC 架构说明文档

AiSOC 是一个开源的、可自托管的 AI 安全运营中心（Security Operations Center）。它接收来自各种安全工具的遥测数据，对威胁进行检测和关联，并使用 AI 智能体（Agent）进行调查，其推理过程完全可审计，最后提出需要人类批准的响应措施。

本文档详细梳理了 AiSOC 的整体系统架构、各个智能体及核心模块是如何交互的，并提供了如何在本地配置环境并运行起来的指南。

---

## 1. 核心模块与系统架构

AiSOC 采用微服务架构设计，各服务之间职责清晰。以下是主要的核心服务模块：

* **Ingest（接收与标准化，`services/ingest`）**: 负责接收来自安全工具（通过轮询供应商 API 或直接推送到 API）的遥测数据，并将其映射为标准的 OCSF (Open Cybersecurity Schema Framework) 格式。它会为每个事件生成“活动投影（Activity Projection）”（参与者、动作、资源、位置、结果）。
* **Spine（事件骨干网，Kafka）**: 所有标准化的事件都会被发布到 Kafka 主题 `aisoc.raw_events` 中。这个边界解耦了生产者和消费者。
* **Fusion（检测与关联，`services/fusion`）**: 消费 Kafka 中的原始事件，运行检测规则（包含单事件规则和基于时间窗的规则）。当规则匹配时，系统结合租户自定义的调整策略（开启/关闭、严重程度下限、压制）判断是否应升级为告警。同时，它将相关告警与现有事件（Incident）关联在一起，写入 Postgres 和 Kafka `aisoc.alerts.fused` 主题。
* **Agents（AI 智能体调查，`services/agents`）**: 监听关联告警。AI 智能体会自动对告警进行分类，提供裁决（Verdict）、置信度和建议的操作。智能体通过 LLM 网关运行，其读取上下文、工具调用、引用以及得出的判定结论都会完整记录到**调查账本（Investigation Ledger）**中。
* **LLM 网关（`litellm` + `ollama`）**: 默认在 CORE 部署中使用 `litellm` 作为网关，并在本地运行 `ollama` 提供小参数量的指令微调模型（如 `llama3.2:3b-instruct-q4_K_M`）。这也确保了离线时的基础智能分析能力。
* **Actions / Playbooks（响应执行，`services/actions` & `services/agents`）**: 基于告警和 AI 的分类结果，可以触发自动化剧本（Playbook）。剧本包含多种类型的响应步骤，涉及到需要向外部厂商执行的变更操作必须经过权限校验并需要**人工批准**。暂停状态会持久化到数据库中。
* **API 与 Web 控制台**: `services/api` 提供核心的业务管理接口，`apps/web` 是基于 Next.js 的前端控制台。`services/realtime` 则通过 WebSocket 将告警实时推送到前端。
* **Threat Intel（威胁情报，`services/threatintel`）**: 抓取 CISA KEV 等公开漏洞目录和情报，供控制台展示并提供 IOC 和威胁主体的查询能力。

### 模块间的交互流转

各模块和智能体的交互通过异步消息（Kafka）和共享数据库协同完成。主要交互如下：
1. **连接器**收集日志并推送给 **Ingest** 服务。
2. **Ingest** 处理后将事件推入 **Kafka** (`aisoc.raw_events`)。
3. **Fusion** 消费事件，进行规则匹配和关联分析，产生告警推送到 **Kafka** (`aisoc.alerts.fused`) 和 **PostgreSQL**。
4. **Agents** 服务消费该告警，搜集上下文证据（如图谱、日志、情报），验证后将提示词发给 **LLM 网关**。LLM 返回判定和建议动作后，Agents 校验引用（Groundedness check），如果有效则记录到**调查账本**。
5. 当满足剧本触发条件时，剧本进入挂起（暂停）状态，向 SOC 分析师呈现**行动建议**。
6. 分析师审查 AI 智能体的记录，通过审批后，由 **Actions** 模块调用厂商 API 执行最终响应动作。

---

## 2. 数据存储

AiSOC 在不同场景下使用了多种数据存储介质，分为必须的（CORE）和可选的（Full 配置）：

### CORE 必选数据存储
* **PostgreSQL**: 存储所有关系型数据，包括告警、事件、案件、用户、租户、检测规则、审计日志以及**调查账本**。如果缺失，整个系统无法运行。
* **Redis**: 用于高吞吐量和需要过期策略的场景（如事件关联的时间窗、去重键和剧本运行状态）。
* **Kafka**: 作为事件骨干网。解耦生产者与消费者，缓冲激增流量。主题包括 `aisoc.raw_events` 和 `aisoc.alerts.fused`。
* **Qdrant**: 向量数据库。用于 IOC（失陷指标）和威胁主体的向量相似度匹配，并为威胁情报页面提供数据支持。

### Full 可选数据存储
* **ClickHouse**: 事件数据湖。用于对数亿条标准化遥测数据进行列式扫描与威胁狩猎（Hunting）。
* **Neo4j**: 实体图谱数据库。解决多跳遍历问题（例如“这个身份还接触了什么”），分析爆炸半径。
* **OpenSearch**: 用于对威胁情报进行全文和结构化搜索（供 `services/threatintel` 使用）。

---

## 3. 安全事件的完整生命周期

一个安全事件在系统中的流转过程如下：

1. **产生与摄取**: 厂商工具（如 EDR、云平台、身份认证服务）产生日志。AiSOC 的轮询连接器或者直接通过 HTTP API 接收到这些数据。此时它会验证请求携带的 Token 的权限范围。
2. **标准化**: `services/ingest` 根据该连接器的特定配置文件，将其解析为 OCSF 格式，提取五元组（Actor, Action, Resource, Location, Outcome）。
3. **入列与检测**: 事件写入 Kafka 后，被 `services/fusion` 消费。该服务运行包含数千条编译后 Sigma 规则的检测引擎。根据租户自己的微调（覆盖）设置，判断是否匹配。
4. **关联与产生告警**: 如果事件为严重告警或特定的安全发现，会被标记为 Promotable 并生成告警。关联引擎检查该告警是否属于某个已存在的事件（Incident），并合并或新建案件。随后该告警被写入 Postgres 并推送至 `aisoc.alerts.fused`。
5. **AI 智能体调查 (Agents Triage)**:
   * `services/agents` 捕获到告警。
   * 收集相关证据：同类告警、实体上下文以及过往的相同判定。
   * 发送给 LLM 前验证提示词，拒绝包含敏感或原始 OCSF 载荷的内容。
   * LLM 返回判决、置信度、被引用的证据及建议。
   * **校验依据 (Groundedness Check)**: 如果模型引用了证据中不存在的指标，判决会被降级并交由人类复核，而不会自动关闭。
   * 这一系列提示词、工具调用、引用和判定都被记入**调查账本 (Investigation Ledger)**。
6. **响应剧本与审批**:
   * 根据判定结果触发预置的 Playbook。剧本默认以**预览**模式运行。
   * 任何试图调用外部 API 的动作（如隔离机器）都会使剧本持久化暂停，并通知需要对应权限级别的分析师进行审批（审批者不能是发起者）。
   * 人工批准后，系统恢复剧本，由对应的控制服务执行变更并验证操作结果。

---

## 4. 本地环境配置与运行指南

AiSOC 提供了一键式的启动方式，能够非常快地在本地拉起完整架构进行测试。

### 环境要求
* Docker Compose v2
* 分配给 Docker 虚拟机的资源至少为 **8 GB 内存** 和 **20 GB 可用磁盘空间**
* `python3` (3.9+) 和 `bash`

你可以运行以下命令检查环境是否满足要求：
```bash
make doctor
```

### 快速启动 (CORE 配置)

通过以下命令拉取代码并启动服务：
```bash
git clone https://github.com/beenuar/AiSOC && cd AiSOC
make up
```

或者使用一步安装脚本：
* **Linux/macOS**: `curl -fsSL https://raw.githubusercontent.com/beenuar/AiSOC/main/install.sh | bash`
* **Windows**: `iwr -useb https://raw.githubusercontent.com/beenuar/AiSOC/main/install.ps1 | iex`

**`make up` 做了什么：**
1. 创建 `.env` 文件并生成必须的 15 个密钥。
2. 创建管理员用户，并在控制台打印在本地生成的随机密码（只显示一次，务必保存）。如果在后续忘记了，可以运行 `make bootstrap ARGS=--reset-password` 重置。
3. 检查端口。如果有端口占用（例如 5432 或 11434 已被占用），它会自动分配可用端口并修改 Web 访问地址。
4. 拉取所有需要的容器，包括自带的 `llama3.2:3b-instruct-q4_K_M` 本地大模型。

### 部署配置类型

| Profile | 命令 | 容器数量 | 内存需求 | 包含内容 |
|---|---|---|---|---|
| **core** | `make up` | 16 | ~8 GB | 完整的告警流水线，LLM 网关，本地模型，CISA 威胁情报源以及连接器。 |
| **full** | `make up-full` | 22 | ~12 GB | Core 的全部内容，加上事件数据湖(ClickHouse)、实体图谱(Neo4j)、全文检索(OpenSearch)和定时连接器。 |
| **demo** | `make up && make demo` | 16 | ~8 GB | Core 的全部内容，并自动加载用于演示的合成安全数据，方便直观查看。 |

### 验证部署 (冒烟测试)

系统启动并正常运行后，执行验证测试来证明整个流水线能够正常工作：
```bash
make smoke
```
该命令会通过 API 推送一条真实的事件，并跟随它穿过 Kafka、检测引擎、关联组件、Postgres，最后通过公开 API 读出对应的告警结果。每一阶段都会报告 PASS 还是 FAIL。

### 访问 Web 控制台
访问 `make up` 命令最后打印出的控制台地址，使用打印的管理员账号密码登录。
第一次登录没有任何数据时，会自动进入“设置向导”。你可以点击 **Load sample data (加载示例数据)**，系统会模拟真实连接器的数据摄取路径推送数据，供你观察告警分析、AI 调查裁决的过程。
