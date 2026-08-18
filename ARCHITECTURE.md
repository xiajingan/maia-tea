# Tea 架构

> Tea 是可控自动回复业务应用；Mud 保存公共会话数据，Stem 执行外部发送。

## 1. 定位与边界

Tea 负责会话检索用例、候选消息过滤、ReplyPolicy 评估、上下文最小化、回复生成适配、人工审核及回执关联。它不摄取/拥有原始 Conversation/Message，不向终端直发命令，不把账号凭据或未授权消息发送给模型。

### Seed 依赖契约

Tea 通过正式、精确锁定的 Seed wheel 复用配置、上下文、错误/事件、通用状态原语、安全/加密/审计、可观测性及所需 Redis/OceanBase 技术 adapter。ReplyPolicy/Candidate/Draft/Approval/Submission、业务去重 key、Model/Repository/SQL/migration 留在 Tea。Tea Sprint 发现公共基础缺口时，从当前 Story/Task 向 Seed 写入 `dependency` Assignment；Seed 自主规划并以 `dependency-package` Delivery 返回精确 wheel version + SHA-256 后，Tea 更新锁文件并完成真实回复 Test。禁止 `latest`、Git/path 依赖、复制 Seed 实现或反向依赖。

## 2. 模型与流程

Tea 是 ReplyPolicy、ReplyCandidate、ReplyDraft、Approval 和 ReplyOutcome 的唯一写入者；Mud 只拥有 Conversation/Message。ReplyOutcome 仅保存 Stem Task/Execution/Proof 引用、来源事件版本和消费水位，不复制或裁决执行最终状态。

Tea 是其私有聚合 AccessMetadata 的 PEP 和最终授权者：从本库读取真实 OwnerRef/Creator/User/version，经服务身份调用 Mud capability/effective-Own，再以 Seed `access-scope-kernel.v1` 合并。Message/Content 与 Stem 引用仍分别调用其所有者；Sage/Iris/模型不得传授权事实。依赖不可用或版本未知时 fail-closed，P0 不缓存授权决定。

ReplyPolicy 遵循 `draft→validated→published→deprecated→archived`，发布版本不可原地修改，固定 Account、意图、Content/生成器、已发布 Action/Workflow 和审批模式（自动执行/先审后发/仅建议）。

- Candidate：`discovered→filtered/eligible→drafted→closed`，CAS 版本阻止并发重复评估。
- Draft：`drafting→pending_approval→approved/rejected/expired/superseded→submitted`；编辑产生新 version 并 supersede 旧稿。
- ReplyApproval：`pending→approved/rejected/expired→consumed`，绑定 draft/策略/参数摘要、确认人和过期时间，单次消费。
- Submission：`prepared→calling→succeeded/failed/outcome_unknown`，终态命令按 idempotency key 返回原结果。

流程为：从 Mud 查询/消费消息 → 逐个校验关联实体 → 规则过滤 → 构造脱敏上下文 → 生成/模板草稿 → 策略分支/ReplyApproval → 提交 Stem Task Command → 关联只读回执。Task Command 固定 Action/Workflow 版本、输入/目标/权限候选；Stem 重验并生成权威快照与 Todo，Tea 不定义隐式发送操作或流程节点。ReplyApproval 只批准回复内容；若 Action 风险策略需要外部副作用确认，Tea 通过 Stem Confirmation API 创建并获得绑定同一目标/版本/参数的 `confirmation_id`，由 Stem 在入队事务单次消费，不能用 ReplyApproval 替代。

自动回复业务唯一键为 `(tenant_id, message_id, reply_purpose)`，独立于 policy version；策略升级/事件重放只能产生新的评估记录，若该 reply key 已 submitted/succeeded 则不得再次外发，人工明确创建的新回复使用新的 purpose/command ID。每次提交先持久化稳定 submission/idempotency key，超时后向 Stem 对账，不盲重试。策略版本、上下文摘要和 Resolver 来源随结果保存。

## 3. 契约与可靠性

Mud API/事件是会话事实来源；生成器经窄 Provider 接口注入；Stem API 是发送唯一入口；Iris/Sage 只调用 Tea 应用 API。消息延迟、重复、乱序均不得产生重复回复。模型失败可降级为人工草稿，但不得静默改用未审批内容。

### 3.1 业务模块

| 模块 | 核心对象 | 责任 |
|---|---|---|
| Inbox | MessageCursor、Candidate | 消息消费、业务去重、候选生命周期 |
| Policy Studio | ReplyPolicy、IntentRule、Scope、ApprovalMode | 规则预览、冲突优先级、版本发布 |
| Context | ContextBundle、Citation、Redaction | 授权取数、窗口裁剪、知识引用、脱敏 |
| Generation | GeneratorPort、PromptVersion、DraftVersion | 模板/模型生成、结构化校验、成本预算 |
| Review | ReplyApproval、ReviewQueue | 编辑/接受/拒绝/超时和责任归因 |
| Delivery | Submission、StemConfirmationRef | 发送幂等、对账、执行引用 |
| Outcome | ReplyOutcome projection | 原消息→策略→草稿→审核→Task/Proof 追溯 |

### 3.2 核心业务流程

```text
message event → candidate dedupe → policy match/conflict resolution
→ authorization + context bundle → template/model draft
→ content/safety validation → auto | review | suggestion branch
→ Stem confirmation when required → Task Command → outcome projection
```

规则冲突按显式 priority、specificity、published_at 决定并记录原因；没有唯一命中时转人工，不随机选择。ContextBundle 记录每条信息的 owner/source/version/permission decision，生成后源权限撤销时禁止提交旧草稿。

### 3.3 对外契约与非功能

- Policy API：draft/validate/sample-preview/publish/deprecate，不原地改 published。
- Review API：claim/edit/approve/reject，使用 draft version CAS，防两人同时审批。
- Query API：candidate/draft/outcome/cursor freshness；消息正文仍按 Mud 权限实时读取。
- 生成 Provider 强制超时、token/cost budget、供应商数据政策和内容安全；提示注入内容不能成为系统指令。
- 消息洪峰按 tenant/account 分区和配额；慢模型不阻塞消费水位，Candidate 持久队列可恢复。

路线：T0 检索/候选 → T1 规则与模板建议 → T2 人工审核发送 → T3 低风险自动发送 → T4 知识增强/质量评估 → T5 多渠道策略；每一步先证明无重复回复和可人工接管。

## 4. 部署与观测

Python/FastAPI 服务与异步 Worker 独立伸缩，私有 Schema 经 Alembic 管理，OCI/Helm 部署。观测覆盖候选积压、过滤原因、生成延迟、审核等待、提交/回执失败、敏感信息拦截和重复抑制。

## 5. 迁移

迁移影响固定覆盖 Mud/旧会话模块中的策略/草稿/结果表、Sage/Iris API、Mud 消息事件、Tea 队列、Stem Command/Confirmation、Schema/Helm/Test。首个 migration manifest 必须证明源中是否存在对应数据：存在则以可重跑 Alembic/backfill 映射状态/权限，Conversation/Message 只留 Mud 引用；不存在则记录零数据基线。`maia-tea-v0.1.0` 完成消费者切流和数量/摘要/水位对账，`v0.2.0` 删除旧 API、事件消费者和表，不双写。
