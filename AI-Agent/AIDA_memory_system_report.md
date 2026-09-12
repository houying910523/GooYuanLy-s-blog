# AIDA 记忆系统设计与运行全景报告

## 1. 报告目标

本文整合对 AIDA 当前记忆系统的代码分析，说明以下问题：

- AIDA 的记忆系统整体是如何设计的。
- 一次请求内，记忆是如何被初始化、注入、使用和收尾的。
- 历史会话是如何被提取成记忆、如何被融合、如何被更新和删除的。
- `/memory/memory_index.md`、`/memory/memories/*.md`、`/memory/sessions/*/*.md` 等产物分别承担什么职责。
- Agent 在运行时如何检索这些记忆，以及哪些文件允许被最终回答引用。

本文基于仓库当前实现，重点参考以下模块：

- `aida/core/agent_memory.py`
- `aida/memory/mode.py`
- `aida/memory/v2/runtime_factory.py`
- `aida/memory/v2/database_store.py`
- `aida/memory/v2/maintenance.py`
- `aida/memory/v2/transcript.py`
- `aida/memory/v2/management.py`
- `aida/memory/v2/management_tools.py`
- `aida/memory/v2/citations.py`
- `aida/memory/memory_rules/memory_v2_rules.md`

## 2. 一句话结论

AIDA 的记忆系统不是单个模块，而是一套分层生态：

- 前台请求层：决定当前请求是否启用记忆、给模型注入哪些记忆、在回合结束时提交 transcript。
- 存储层：把 transcript、记忆快照、用户长期记忆、显式记忆 note、citation 事件分别落到数据库、对象存储和 memory sandbox 文件树。
- 后台维护层：周期性扫描需要处理的记忆 scope，把历史 transcript 提炼为 session 级记忆，再融合成主题级记忆和总索引。
- 检索使用层：模型通过注入的规则和 `/memory` 文件树做轻量 memory pass，优先读索引，再读主题记忆，必要时下钻到 session 级摘要。

这套系统里，AIDA 主要负责接线、存储、调度和约束；真正的“提炼记忆”和“融合记忆”算法主体由 `bytedance.agent_mem` SDK 执行，AIDA 为其提供运行时和基础设施。

## 3. 模式与分层

### 3.1 运行模式

`aida/memory/mode.py` 定义了四种模式：

- `disabled`
- `single_file`
- `multi_file`
- `full`

可将其理解为：

| 模式 | 含义 |
| --- | --- |
| `disabled` | 不启用记忆系统 |
| `single_file` | 只启用用户维护的根 `MEMORY.md` |
| `multi_file` | 只启用 Agent Mem V2 多文件记忆 |
| `full` | 同时启用 `MEMORY.md` 与 Agent Mem V2 |

模式不是硬编码在 agent 里，而是通过 `(user, agent)` 维度配置 `long_term_memory_mode` 解析出来，并受入口类型、全局开关、cron 限制等约束。

### 3.2 逻辑分层

从职责上看，当前记忆系统可以分为五层：

1. 模式解析层  
   决定本次请求是否允许使用记忆，以及使用哪种模式。

2. 请求运行时层  
   为本次请求创建 memory bundle，准备 rules prompt，注册 memory attachment。

3. 记忆文件与数据库层  
   负责 transcript、phase1 artifact、session start snapshot、integrated notes、citation event、`MEMORY.md` 等对象的持久化。

4. 后台维护层  
   周期性领取待处理的 memory scope，执行 Phase 1 和 Phase 2。

5. 模型使用层  
   通过注入规则指导模型在 `/memory` 文件树内进行检索、下钻和引用。

## 4. 记忆数据模型与文件生态

### 4.1 几类核心数据

当前系统里最重要的几类数据不是同一种东西：

| 数据 | 作用 |
| --- | --- |
| `MEMORY.md` | 产品级、用户显式维护的长期记忆 |
| `notes/*.md` | 用户新增、修改、删除的显式记忆指令增量 |
| `memories/*.md` | 融合后的主题级正式记忆文件 |
| `sessions/<session-id>/*.md` | 单个历史会话的摘要或证据片段 |
| `memory_index.md` | 当前 scope 的主题索引和导航页 |
| `transcripts/<session-id>/transcript.md` | 完整原始历史对话与工具轨迹 |

### 4.2 这些文件的关系

它们的上下游关系大致如下：

```text
当前会话原始消息
    -> canonical transcript
    -> transcript 入库
    -> Phase 1 提炼
         -> session 级 distilled memory
    -> Phase 2 融合
         -> memories/*.md
         -> memory_index.md

用户显式编辑
    -> MEMORY.md            （产品长期记忆）
    -> notes/*.md           （增量记忆指令）
    -> 后台 maintenance
         -> 融入 memories/*.md / memory_index.md
```

### 4.3 各文件的职责区别

#### `MEMORY.md`

- 这是用户显式维护的长期偏好、稳定约束、长期事实。
- 它不依赖 transcript 生成。
- 读取时会做模板噪音剔除，避免未填写模板污染 prompt。
- `single_file` 模式下只注入它。
- `full` 模式下它会作为 session start 的一个输入交给 SDK。

它是“用户长期记忆”，不是“自动提炼快照”。

#### `notes/*.md`

- 这是用户显式新增或修改的记忆增量。
- 它不是最终的正式记忆快照，而是待融合输入。
- 用户通过 `memory-management` skill 新增、更新、删除的主要就是这类文件。
- 对 note 的变更会触发 maintenance signal，但不会立即同步改写 `memories/*.md`。
- 它不是 `bytedance.agent_mem` SDK 自动提炼出来的产物，而是 AIDA 管理链路直接写出的显式输入层。

它是“用户记忆指令的增量层”。

#### `memories/*.md`

- 这是正式的主题级记忆文件。
- 模型检索时优先搜索并打开它们。
- 它们是 transcript、历史 session 摘要和用户 note 融合后的结果。
- 最终回答允许对它们做 memory citation。

它是“正式可检索记忆正文”。

#### `sessions/<session-id>/*.md`

- 这是单个历史会话的摘要与证据片段。
- 粒度比 `memories/*.md` 更细。
- 通常只有在主题记忆指向某个历史会话、或需要更具体依据时才下钻读取。
- 最终回答允许对它们做 memory citation。

它是“主题记忆背后的会话级依据”。

#### `memory_index.md`

- 它是整个 memory scope 的索引页、关键词表、导航页。
- 它会在 session start 被自动注入。
- 模型规则要求优先利用注入的 index 内容判断要不要做 memory pass。
- 除非注入块标记为 `truncated="true"`，否则不应重复打开它。
- 它不应作为最终回答的 citation 来源。

它是“目录和路由页”，不是证据正文。

#### `transcripts/<session-id>/transcript.md`

- 这是原始历史会话与工具轨迹的完整形态。
- 它是最后一级证据。
- 只有当 `memories/*.md` 与 `sessions/*.md` 仍不足以支撑判断时，才应按具体关键词精确打开少量 transcript。

它是“原始材料”，不是常规 memory pass 的第一层。

### 4.4 `MEMORY.md` 与 `notes/*.md` 的来源

这两类文件都不是“历史会话自动提炼后直接生成”的同一层产物，但它们的来源不同：

#### `MEMORY.md` 的来源

`MEMORY.md` 是产品级用户长期记忆，不来自 SDK 自动提炼。

它的来源有三种：

1. 首次通过管理接口读取且文件不存在时，系统懒创建一个本地化模板。
2. 用户后续通过管理接口提交新内容时，系统直接覆盖写入。
3. 用户执行清空操作时，系统把它写成空字符串。

因此，`MEMORY.md` 的内容来源本质上是：

- 初始化模板
- 用户手工维护

而不是 transcript、Phase 1 摘要或 Phase 2 融合结果。

#### `notes/*.md` 的来源

`notes/*.md` 是显式记忆编辑链路的输出，不是 SDK 提炼输出。

主链路是：

```text
用户显式要求记住/修改/撤回
    -> memory-management skill
    -> MemEditSkillTool
    -> MemoryV2ManagementRepository.create_entry / edit_entry
    -> 写入 agent_view/notes/*.md
```

也就是说，`notes/*.md` 是 AIDA 自己的管理逻辑写出的“待融合输入层”。

#### `notes/*.md` 的生成输入

新建 note 时，工具层输入包括：

- `path`：形如 `/memory/notes/<slug>.md`
- `old_string`：必须为空
- `new_string`：note 正文自然语言内容
- `create_file`：必须为 `true`
- `goal`：本次记忆变更的目的说明

系统还会自动补充：

- `session_id`
- `updated_at_utc`

更新已有 note 时，工具层输入包括：

- `path`：现有 `/memory/notes/<filename>.md`
- `old_string`：当前正文中唯一命中的精确旧片段
- `new_string`：替换后的新片段
- `create_file`：为 `false`
- `goal`

系统同样自动补充：

- `session_id`
- `updated_at_utc`

此外，更新场景还隐含依赖一个输入：

- 旧 note 的当前正文内容

因为 repository 需要先读取旧文件，再在正文中做唯一精确替换。

#### `notes/*.md` 的最终文件形态

写入 repository 后，最终 note 文件会被规范化为：

```md
---
session_id: <系统写入>
updated_at_utc: <系统写入>
---

<自然语言正文>
```

其中：

- frontmatter 是系统元数据，不是用户正文的一部分
- 搜索、读取、编辑都以正文为准
- 如果用户自己在输入里携带 frontmatter，repository 会丢弃它并重建标准 frontmatter

因此，准确的数据方向是：

```text
显式用户输入
    -> AIDA 写入 notes/*.md
    -> SDK 读取 notes/*.md 并参与融合
    -> 产出 memories/*.md / memory_index.md
```

## 5. 请求内生命周期

### 5.1 初始化：模式解析与 runtime 创建

在 `BaseAgent` 初始化阶段，如果当前请求的 `memory_mode` 不是 `disabled`，AIDA 会尝试创建 `_memory_v2_bundle`。

但真正会创建 V2 bundle 的只有：

- `multi_file`
- `full`

`single_file` 不创建 V2 bundle，因为它只依赖 `MEMORY.md`。

bundle 中包含：

- `MemoryRuntime`
- `AidaMemoryV2DatabaseStore`
- sandbox transport
- V2 file storage
- user long-term memory provider
- layout 与 identity

这是本次请求独占的运行时容器。

### 5.2 首次 prompt 前：注册 memory session state

在 `AidaAgent.setup_react_loop` 早期，会调用 `_init_memory_session_state()`，再进一步调用 `initialize_memory_session()`。

这么做的目的是：

- 在第一次 prompt 渲染前就把本次请求的 memory rules 和 memory attachment 准备好；
- 使 PromptAssembler 在拼装上下文时能看到正确的记忆配置。

### 5.3 `single_file` 路径

`single_file` 路径最简单：

1. 构造当前 `(user_id, agent_id)` 的 `MemoryV2ScopeIdentity`
2. 读取根 `MEMORY.md`
3. 去掉模板占位噪音
4. 渲染成 `auto_injected_long_term_memory` 块
5. 作为一条 meta/resource message 注册到上下文

它不会调用 SDK 的 `handle_before_session_start()`，也不会启用多文件快照。

### 5.4 `multi_file/full` 路径

这条路径有两个动作：

1. 把 `memory_v2_rules.md` 的内容挂到 `agent._memory_system_prompt`
2. 注册一个懒加载 attachment builder

这个 attachment 不是启动时就立刻展开，而是在上下文 slot 重建时才真正读取 memory。

### 5.5 session-start 注入

当 attachment 真正构造时：

- 如果当前模式是 `full`，先读取用户长期记忆 `MEMORY.md`
- 再把这份长期记忆传给 `bundle.runtime.handle_before_session_start()`
- SDK 返回 `context_blocks`
- AIDA 用 `render_injected_memory_context()` 把这些 block 渲染成可注入的上下文字符串

当前注入体系支持的 block 包括：

- `auto_injected_long_term_memory`
- `auto_injected_memory_index`
- `auto_injected_memory_notes`
- `auto_injected_retracted_notes`

从职责上说：

- SDK 决定注入什么
- AIDA 决定怎么把它渲染成对模型可读的提示文案

## 6. AIDA 如何提取记忆

### 6.1 原始输入不是 step，而是 canonical transcript

记忆提取的原始材料来自当前请求结束时的 raw messages。

在回合结束前，AIDA 会调用 `build_memory_v2_transcript()` 把当前上下文消息投影成 Agent Mem 可接受的 transcript JSON。

这个投影做了几件关键事情：

- 只保留 `user` / `assistant` / `tool`
- 删除 host injection
- 删除 `final_output_prepare` 管道消息
- 删除旧式文本 citation block
- 尽力保留 tool call ID、turn ID、tool result 状态

因此，进入记忆系统的不是“原始 prompt 拼装结果全文”，而是一份尽量 canonical 的对话投影。

### 6.2 transcript 提交

构造好的 transcript 会通过 `bundle.runtime.submit_transcript()` 提交。

底层 `AidaMemoryV2DatabaseStore.upsert_transcript()` 会：

1. 把 transcript JSON 存到 blob store
2. 在 RDS 中按 `(scope, session_id)` upsert 一条 session 行
3. 若 transcript 内容有变化，则递增 `transcript_revision`
4. 把 scope state 标为 `transcript_changed`
5. 让后台 scheduler 后续能发现这个 scope 需要 maintenance

这一步不直接生成 `memories/*.md`，只是把“可供提炼的原料”放进系统。

### 6.3 Phase 1：从 transcript 提炼 session 记忆

后台 `MemoryV2MaintenanceService` 扫描到 due scope 后，会运行 Phase 1。

Phase 1 的职责是：

- 领取已经稳定的 pending session
- 读取对应 transcript
- 调用 `AGENT_MEMORY_PHASE1` 对应的模型
- 为每个 session 产出两份结构化结果：
  - `raw_memory`
  - `session_summary`

随后通过 `complete_phase1()` 写入 phase1 artifact。

从代码契约看，这两份数据可以理解为：

- `raw_memory`：更像供后续融合使用的“记忆原始提炼结果”
- `session_summary`：更像可供人或模型阅读的 session 级摘要

### 6.4 Phase 2：融合成正式记忆

Phase 2 由 SDK `MemoryMaintenanceRunner` 驱动，AIDA 负责运行环境和持久化接口。

从 AIDA 侧能确认的融合输入有：

- Phase 1 产出的 distilled sessions
- 当前 `notes/*.md`
- 当前 `MEMORY.md` 长期记忆
- 已有 `memory_index.md`
- 已有 integrated notes baseline

融合完成后，系统会更新：

- `memory_index.md`
- `memories/*.md`
- `selected_for_phase2`
- `integrated_note_snapshot`
- `session_start_snapshot`

也就是说，正式的主题记忆和总索引不是直接从 transcript 一步生成，而是要经过：

```text
transcript -> Phase 1 distilled session -> Phase 2 consolidation
```

## 7. AIDA 如何管理记忆

### 7.1 `MEMORY.md` 管理

`MEMORY.md` 的管理是直接读写型：

- `get_memory_md()`：读取；不存在时可懒创建模板
- `put_memory_md()`：覆盖写
- `clear_memory_md()`：写空

它的特点是：

- 面向用户长期偏好与稳定约束
- 不经过 note 融合链路
- 运行时读取是非创建式，只有管理接口会懒创建模板

### 7.2 note 的新增

显式“记住某件事”通常不是直接改 `memories/*.md`，而是创建 `notes/*.md`。

`MemEditSkillTool` 在 `create_file=true` 时会：

- 要求路径形如 `/memory/notes/<slug>.md`
- 自动生成 UTC 时间前缀
- 实际创建一个物理 note 文件
- 立即调用 `_signal_maintenance(reason="note_created")`

因此，“新增记忆”的第一步是创建 note，不是直接写正式记忆。

### 7.3 note 的更新

更新 note 采用“精确替换”策略：

- 必须给出 `old_string`
- `old_string` 必须在当前 note 中唯一命中
- 0 次命中报错
- 多次命中也报错

这个约束是为了防止模型对用户记忆做模糊修改。

更新成功后会发送 `_signal_maintenance(reason="note_changed")`。

### 7.4 note 的删除

note 删除也不是“删完就结束”。

删除成功后，系统需要让后台知道：

- 某条已存在的显式记忆被撤回了
- 未来的 consolidation 需要重新融合

因此删除 note 后也会触发 maintenance signal。即使用户是通过 Bash 执行 `rm -- /memory/notes/...`，运行时 hook 也会尝试补发 `note_deleted` 信号。

### 7.5 融合不是同步写回

note 的创建、编辑、删除都不会立即重写 `memories/*.md`。

AIDA 的行为是：

1. 修改 note 层
2. 标记 scope 需要 maintenance
3. 等后台 worker 执行 Phase 2 consolidation
4. 再把正式记忆与总索引更新出来

这是一个显式的异步融合设计。

### 7.6 integrated note snapshot 的作用

数据库里有一份 `integrated_note_snapshot`。

它可以理解为：

- “上一次已经融合进正式记忆的 note 指纹基线”

有了这份基线，系统就能比较：

- 当前 `notes/*.md` 与已融合基线相比哪些是新增的
- 哪些内容发生了变化
- 哪些历史 note 已被删除

这也是 `auto_injected_retracted_notes` 能成立的基础之一。

### 7.7 删除后的撤回语义

当某条已经融入过正式记忆的 note 被删除时，正式快照不一定会在同一时刻立刻完全重写。

因此系统需要一个过渡态：

- 旧的 snapshot 里可能还残留这条记忆
- 但当前用户已经明确撤回它了

为了解决这个问题，session-start 注入里预留了 `auto_injected_retracted_notes`。

这说明系统设计上承认：

- 正式记忆快照是异步收敛的
- 撤回语义需要先通过注入层及时覆盖，再等待下一次 consolidation 真正重写快照

## 8. 检索是如何进行的

### 8.1 检索入口不是工具链特殊 API，而是 `/memory` 文件树

对模型来说，memory 不是通过某个专用 RPC 检索的，而是通过普通文件工具访问 `/memory` 和 `/transcripts`：

- `/memory`
- `/transcripts`

这些路径不是本地真实目录，而是通过 memory sandbox transport 映射出来的 scoped 文件系统视图。

### 8.2 memory pass 的推荐路径

`memory_v2_rules.md` 明确约束了 memory pass 的步骤：

1. 先看自动注入的 `<auto_injected_memory_index>`
2. 从中抽取与当前问题相关的关键词
3. 用 `rg` 搜 `/memory/memories/*.md`
4. 打开相关 `memories/*.md`
5. 只有在主题记忆明确指向具体会话、且确实需要更细证据时，才打开 1-2 个 `/memory/sessions/<session-id>/<name>.md`
6. 只有在更高层文件仍不足以支撑结论时，才去查 `/transcripts/.../transcript.md`

这意味着记忆检索是分层下钻式的，而不是一上来就扫 transcript。

### 8.3 为什么 `memory_index.md`、`memories/*.md`、`sessions/*.md` 要分层

这三层各自解决不同问题：

| 层级 | 解决的问题 |
| --- | --- |
| `memory_index.md` | “当前 scope 大概记住了哪些主题，值不值得搜？” |
| `memories/*.md` | “围绕某个主题，系统已经整理好的正式记忆是什么？” |
| `sessions/*.md` | “这条主题记忆背后，具体是哪次历史会话、当时发生了什么？” |

分层的好处是：

- 常见请求不需要下钻太深
- agent 不会被 transcript 噪音淹没
- citation 也能更精确地落在真正使用过的正式记忆文件上

## 9. 产物之间的区别

### 9.1 `memory_index.md`、`memories/*.md`、`sessions/*.md`

这三类文件最容易混淆，其实职责非常明确：

| 文件 | 本质 | 典型用途 | 能否做 citation |
| --- | --- | --- | --- |
| `memory_index.md` | 总索引、关键词地图、导航页 | 判断要不要做 memory pass，定位主题 | 否 |
| `memories/*.md` | 主题级正式记忆 | 主要检索入口 | 是 |
| `sessions/*.md` | 单会话摘要与证据片段 | 按需下钻、补充证据 | 是 |

可以这样理解：

- `memory_index.md` 像目录
- `memories/*.md` 像按主题整理后的知识条目
- `sessions/*.md` 像知识条目背后的案例与出处

### 9.2 `notes/*.md` 与 `memories/*.md`

这两类也容易混淆，但它们不是同一层：

| 文件 | 语义 |
| --- | --- |
| `notes/*.md` | 用户显式编辑的增量记忆指令 |
| `memories/*.md` | 系统融合后的正式主题记忆 |

前者是“待融合输入”，后者是“融合产物”。

### 9.3 `MEMORY.md` 与 `notes/*.md`

这两者也不同：

| 文件 | 适合存什么 |
| --- | --- |
| `MEMORY.md` | 稳定、长期、整体性的偏好与约束 |
| `notes/*.md` | 某次新增、修改、撤回的显式记忆增量 |

`MEMORY.md` 更像“个人长期说明书”，`notes/*.md` 更像“增量补丁”。

## 10. citation 与“哪些记忆真正被用到”

记忆系统不只负责“给模型看什么”，还负责“记录最终回答到底用了什么”。

当前做法是：

- `final_output_prepare` 在符合条件时暴露一个结构化参数 `memory_citations`
- 模型若用了正式 memory 文件，需要把用到的 `memories/*.md` 或 `sessions/*.md` 上报进这个参数
- AIDA 在 `capture_memory_citations()` 中校验这些路径与 `source_session_ids`
- 回合结束时调用 `bundle.runtime.handle_response()` 进行 citation accounting

这一步会把 usage 反写到数据库：

- 记录 citation event
- 记录 citation source
- 给被引用 session 增加 `citation_count`
- 刷新 `last_used_at`

因此，系统不仅知道“记忆存在”，还知道“哪些历史记忆真正影响了最新回答”。

## 11. 存储与隔离设计

### 11.1 scope

记忆 scope 不是 session 级，而是 `(user_id, agent_id)` 级。

这意味着：

- 同一个用户在同一个 agent 下的多次会话会累积到同一个记忆空间
- 记忆是跨 session 演化的

### 11.2 前台与后台使用不同 sandbox 生命周期

前台请求：

- 优先复用当前 session sandbox
- 只释放本请求线程的 transport 引用

后台 maintenance：

- 为某个 memory scope 申请独占 sandbox
- 完成后销毁当前 scope sandbox

这么做是为了：

- 不让后台 consolidation 干扰前台请求
- 保持请求内读取与后台改写的生命周期边界清晰

### 11.3 降级路径

如果 file storage 部分不可用，runtime 可以退化为 `transcript_only_bundle`：

- `/memory` 正式文件能力不可用
- 但 transcript 依然可以继续提交

这说明系统设计优先保证“记忆原始材料不丢”，而不是“一旦 file storage 出问题就彻底停摆”。

## 12. 一个完整生态周期

下面用时间顺序描述一次完整记忆生态周期：

```text
用户发起请求
    -> 解析 memory mode
    -> 创建 memory runtime bundle（仅 multi_file/full）
    -> 注入 memory rules
    -> session start 注入
         -> 长期记忆
         -> memory index
         -> pending notes / retracted notes
    -> 模型运行，必要时读取 /memory
    -> 最终回答前可提交 memory citations
    -> 回合结束，提交 canonical transcript
    -> scope 标记为 transcript_changed

后台 scheduler 扫描 due scope
    -> 领取 dispatch lease
    -> Phase 1：把 transcript 提炼为 raw_memory + session_summary
    -> Phase 2：融合 distilled sessions、notes、长期记忆、现有 snapshot
    -> 更新 memories/*.md、memory_index.md、session_start_snapshot
    -> 下轮请求再注入更新后的记忆
```

如果用户中途显式说“记住这个”“把这个偏好改掉”“忘掉刚才那条”：

```text
显式编辑请求
    -> 写 MEMORY.md 或 notes/*.md
    -> signal_maintenance
    -> 后台下一轮融合
    -> 正式 memory snapshot 更新
```

## 13. 需要特别注意的实现事实

### 13.1 `aida/adaptor/memory_es.py` 不是当前主链路核心

仓库里有一个 `memory_es.py`，提供通用 ES 与 embedding 能力。

但从当前 AIDA 主记忆链路看：

- 它不是 `core/agent_memory -> runtime_factory -> maintenance` 这条链路的核心组件
- 它更像一个通用检索底座，被别的历史或旁支能力复用

因此当前主记忆系统的核心不在 ES，而在：

- Agent Mem runtime
- database store
- sandbox file storage
- maintenance scheduler

### 13.2 AIDA 自己没有实现全部融合算法

代码里最关键的边界是：

- AIDA 提供 transcript、file storage、database store、scheduler、prompt rules
- SDK 负责从 transcript 提炼和融合出正式记忆

因此如果要继续研究“Phase 2 到底如何挑选会话、如何合并 note、如何切分 memories 文件”，需要继续阅读 `bytedance.agent_mem` SDK 源码，而不是只看 AIDA 仓库。

## 14. 总结

AIDA 当前的记忆系统是一个明确分层、异步收敛、可追踪引用的系统。

它不是简单把历史对话丢给模型，而是把记忆拆成了几层：

- `MEMORY.md` 负责稳定长期偏好
- `notes/*.md` 负责显式增量记忆
- transcript 负责提供原始会话材料
- Phase 1 负责把会话提炼成 session 级记忆
- Phase 2 负责把会话级记忆与显式记忆融合成正式记忆快照
- `memory_index.md`、`memories/*.md`、`sessions/*.md` 负责分层检索
- citation accounting 负责记录“最终答案到底用了哪些记忆”

从设计上看，这套体系的关键特征是：

- 请求内读取与后台维护分离
- 用户显式记忆与系统自动提炼记忆分离
- 正式快照与待融合增量分离
- 检索层与持久化层分离
- 使用记录与记忆内容本身分离

如果只用一句话概括：

AIDA 把“记忆”设计成了一个跨请求持续演化的知识生态，而不是一段静态 prompt 文本。
