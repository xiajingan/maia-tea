# Tea 用户故事

> 演进顺序：T0 候选/检索 → T1 规则/模板建议 → T2 人工审核发送 → T3 低风险自动发送 → T4 知识增强。任何阶段先证明同一业务 reply key 不重复外发。

Tea Sprint 发现公共基础缺口时，从当前 Story/Task 向 `maia-seed` 写入统一 `dependency` Assignment；写入不启动 Seed Sprint、不预填版本。Tea 只在结果 `delivered` 后使用 Delivery 的精确 wheel version + SHA-256 并完成真实回复 Test。回复策略、审批和业务去重语义不得下沉 Seed，禁止 `latest`、Git/path 依赖。

| ID | 用户故事 | 验收标准 | 来源 | 状态 |
|---|---|---|---|---|
| TEAAPP-001 | 作为会话运营人员，我希望检索授权会话和消息。 | 支持账号/时间/关键词；Own 与 Use 明示；关联实体分别鉴权。 | TEA-001 | `draft` |
| TEAAPP-002 | 作为运营人员，我希望配置消息过滤规则。 | 规则版本化、可测试；重复/过期/不合规消息不进入生成；原因可查。 | TEA-002 | `draft` |
| TEAAPP-003 | 作为运营人员，我希望发布自动回复策略。 | 固定账号、意图、Content/生成器、Action/Workflow 与审批模式；发布版本不可修改。 | TEA-003 | `draft` |
| TEAAPP-004 | 作为用户，我希望生成、编辑、接受或拒绝回复建议。 | 只用最小脱敏上下文；记录模型/提示/策略版本；用户处理留痕。 | TEA-004 | `draft` |
| TEAAPP-005 | 作为系统，我希望经 Stem 幂等发送回复。 | 固定已发布执行版本；未知结果先对账；Tea 不直连 Celt。 | TEA-005/006 | `draft` |
| TEAAPP-006 | 作为运营人员，我希望查看自动回复结果和异常。 | Candidate→Draft→Approval→Task→回执可关联；失败分类和人工入口明确。 | TEA-005 | `draft` |
| TEAAPP-007 | 作为 Tea，我希望经版本化 Resolver 解析所有关联实体。 | 逐实体鉴权；记录版本/来源；缺少 KB/Content 时只降级人工。 | TEA-006 | `draft` |
| TEAAPP-008 | 作为安全负责人，我希望阻止越权或敏感上下文进入模型。 | Tea 用本地 AccessMetadata OwnerRef + Mud capability/effective-Own + Seed kernel 最终授权；关联实体逐所有者复验；客户端/模型权限事实无效，依赖失败关闭；凭据和敏感字段过滤，拦截有审计测试。 | FND-004/007～011/018 | `draft` |
| TEAAPP-009 | 作为发布负责人，我希望在 Test 验证真实回复闭环。 | 覆盖重复消息、规则过滤、审核、发送失败、对账、回执与无越权。 | FND-021 | `draft` |

## 产品化细化故事

| ID | 用户故事 | 验收标准 | 来源 | 状态 |
|---|---|---|---|---|
| TEAAPP-010 | 作为运营人员，我希望用样本预览策略命中。 | 展示命中/排除规则、priority/specificity 决策和预计审批模式；预览不创建 Candidate。 | TEA-002/003 | `draft` |
| TEAAPP-011 | 作为运营人员，我希望冲突规则确定性收敛。 | 相同消息得到相同 winner；无唯一 winner 转人工；发布前检测不可达/重叠规则。 | TEA-002 | `draft` |
| TEAAPP-012 | 作为审核人员，我希望并发编辑不会错批。 | draft version CAS；编辑 supersede 旧 Approval；claim 过期释放；审计确认人和内容摘要。 | TEA-003/004 | `draft` |
| TEAAPP-013 | 作为安全负责人，我希望源权限撤销使旧草稿失效。 | 提交前重验 Message/Content/Account/Action；失效转人工并说明；不发送缓存内容。 | FND-018/TEA-006 | `draft` |
| TEAAPP-014 | 作为平台，我希望模型慢或不可用时消息消费不堵塞。 | Candidate 先持久化；生成独立 Worker/配额；超时进入人工；消费水位可恢复。 | TEA-004 | `draft` |
| TEAAPP-015 | 作为质量人员，我希望评估建议质量而不自动放权。 | 采纳/编辑/拒绝指标关联版本；离线数据脱敏；质量提升不改变审批模式。 | TEA-004/005 | `draft` |
| TEAAPP-016 | 作为运营人员，我希望自动发送可随时降级。 | 按 tenant/policy kill switch；停止新提交、不伪取消已执行；降级到 review 并审计。 | TEA-003/005 | `draft` |
