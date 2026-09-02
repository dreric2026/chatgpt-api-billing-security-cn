---
title: "2026年9月2日 ChatGPT Plus、Pro、Codex 项目指令教程：用 AGENTS.md 与 CI 管住 AI Agent"
description: "把 Codex 从临时聊天变成可审计开发流程：任务边界、AGENTS.md、测试门禁、成本记录，以及订阅充值失败时的降级方案。"
date: 2026-09-02
permalink: /guides/2026-09-02-chatgpt-plus-pro-codex-agents-ci/
---

# 2026年9月2日 ChatGPT Plus、Pro、Codex 项目指令教程：用 AGENTS.md 与 CI 管住 AI Agent

> ## [gptupcn.com｜ChatGPT Plus、Pro、Codex 订阅与续费入口](https://gptupcn.com)

> **适用人群**：在 GitHub 仓库中使用 Codex 或其他 AI Agent 改代码，希望把任务边界、测试、审计与 ChatGPT Plus / Pro 预算统一起来的开发者。  
> **核心搜索词**：Codex教程、AGENTS.md、AI Agent工作流、ChatGPT Plus充值、Pro充值、Codex充值、订阅续费、充值失败。  
> **更新时间**：2026年9月2日。Codex 的模型、可用入口、allowance、credits 与套餐关系会变化；使用前请查看当前官方文档、账号状态与仓库实际环境。

仓库刚接入 AI Agent 时，大家通常从一句“帮我修这个 bug”开始。

第一次改动很顺利，第二次 Agent 却顺手格式化了几百个无关文件，第三次因为测试环境缺失不断重试。团队开始怀疑是不是 Plus 不够、是不是应该 Pro充值。

可就算升级套餐，没有边界的任务仍会扫描更多、改动更多、重试更多。容量只会放大原有流程，不能替代工程治理。

另一个风险发生在订阅或登录短时异常：关键修复依赖单次 Agent 会话，任务上下文没有写进 Issue，也没有可运行的人工验收命令。一旦 Codex 暂时不可用，团队甚至不知道做到哪里。

本教程把治理写进仓库：用 `AGENTS.md` 描述规则，用任务契约限定范围，用 CI 做独立验收，用日志记录结果。套餐选择与充值只是容量层，工程门禁才是质量层。

## **先问一句：如果 Agent 明天不能继续，这次修改还能被人类接手吗？**

## 1. AGENTS.md 解决什么，不解决什么？

`AGENTS.md` 适合放仓库级或目录级的工作说明，例如构建命令、测试要求、禁改区域、代码风格和安全边界。它让 Agent 进入项目后先读取可执行规则，而不是每次依赖聊天中的临时记忆。

它不能替代权限控制、分支保护和 CI。把“不要提交密钥”写在文件里很重要，但仍要用密钥扫描和最小权限做技术约束。

也不要把整个公司手册复制进去。好的指令短、明确、可验证，并尽量指向仓库中已有脚本。

## 2. 一份可执行的 AGENTS.md 应包含哪些部分？

至少包含范围、准备、允许修改、禁止修改、测试、提交和停止条件。

```markdown
# AGENTS.md

## Scope
- Applies to the whole repository unless a deeper AGENTS.md overrides it.

## Setup
- Install: `npm ci`
- Unit tests: `npm test -- --runInBand`
- Lint: `npm run lint`

## Change boundaries
- Keep changes inside `src/` and `tests/` for ordinary bug fixes.
- Do not edit `.github/workflows/`, billing code, or secrets without an explicit task.
- Never print environment variables, tokens, customer data, or payment details.

## Acceptance
- Add or update a regression test.
- Run the smallest relevant test first, then the full required suite.
- Summarize changed files, commands, failures, and remaining risks.

## Stop conditions
- Stop after the same failure occurs twice.
- Stop if credentials, production access, or destructive migration is required.
- Ask for human review when the requested change exceeds the stated scope.
```

停止条件尤其重要。它能避免 Agent 在缺少依赖、权限或环境时用重试消耗时间和用量。

## 3. 任务契约为什么要和仓库指令分开？

`AGENTS.md` 是长期规则，任务契约是本次工作的临时目标。把两者分开，团队可以修改 Issue 而不频繁改仓库政策。

一份任务契约应该写：问题、允许范围、非目标、验收命令、风险和交付格式。例如“修复分页边界错误；只改 `src/pager.ts` 与相关测试；不改公共 API；新增零条记录和整页记录测试”。

范围越清楚，Agent 越少为了寻找目标而扫描无关上下文。对 Codex 用量而言，这比单纯追求更短提示更有意义。

## 4. Plus、Pro 与 Codex 应处在架构的哪一层？

可以把开发流程分为五层：身份、容量、任务、执行、验收。

| 层级 | 关键问题 | 主要控制 |
|---|---|---|
| 身份层 | 谁在运行、能访问什么 | ChatGPT账号、仓库权限、最小授权 |
| 容量层 | 高峰能否连续工作 | Plus / Pro、当前 allowance、合资格 credits |
| 任务层 | 改什么、不改什么 | Issue、任务契约、AGENTS.md |
| 执行层 | 如何检索、修改、测试 | 工具权限、停止条件、日志 |
| 验收层 | 结果是否可合并 | CI、评审、分支保护、回滚 |

ChatGPT Plus充值或 Pro充值只影响其中一部分，不能替代后面三层。Codex充值也必须先说明是账号套餐、额外 credits 还是 API 预算。

## 5. 怎样用 CI 独立验证 Agent 的结果？

无论代码来自人还是 Agent，合并条件都应一致。下面是简化的 GitHub Actions 工作流，使用只读权限、固定 Node 版本和最小测试步骤。

```yaml
name: agent-change-gate

on:
  pull_request:

permissions:
  contents: read

jobs:
  verify:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm test -- --runInBand
```

真实仓库还应固定依赖、限制外部动作权限、增加代码扫描，并为高风险目录设置 CODEOWNERS。CI 的角色不是“相信 Agent”，而是重放一组独立、可审计的验收。

## 6. 怎样避免 Agent 修改不该改的文件？

先用文字边界，再用技术边界。

文字边界在任务契约中列出允许路径；技术边界通过分支保护、代码所有者、只读凭证和 CI 检测实现。还可以在 CI 中检查 diff，如果普通任务改到了付款、身份或工作流文件，就要求安全负责人评审。

对于支付与订阅相关仓库，不要让 Agent 接触真实卡号、完整订单、客户身份或生产密钥。使用脱敏夹具、沙盒接口和合成数据完成测试。

## 7. 如何记录一次 Agent 任务，便于容量复盘？

在 Pull Request 模板中增加四个字段：任务类别、上下文规模、运行命令、验收结果。不要贴完整对话，只保留可复现事实。

```markdown
## AI task record
- Task class: bugfix
- Context band: medium
- Allowed paths: `src/pager.ts`, `tests/pager.test.ts`
- Commands run: `npm test -- pager`
- Retries: 1
- Result: accepted by CI
- Sensitive data exposed: no
```

连续一个月后，你可以比较哪些任务适合 Codex，哪些任务总要人工接管。套餐选择就从主观感受变成基于交付的判断。

## 8. Codex 受限时，应该升级还是优化？

先看限制发生在哪类任务。如果大量消耗来自范围模糊、相同失败重复、测试环境缺失或把整个仓库当上下文，先修流程。

如果已经有清晰边界、停止条件和 CI，仍然在高价值交付窗口频繁等待，再依据当前官方说明比较 Plus 与 Pro。实际用量会随模型、上下文、复杂度、推理与工具调用变化，不能用网上固定次数做永久承诺。

ChatGPT订阅与 API 账单分开。CI 自动化若使用 API key，应按 API 项目预算管理，而不是认为个人 Pro 可以覆盖所有流水线调用。

## 9. 充值失败时，仓库工作流如何降级？

首先不要连续跨渠道付款。核对目标账号、原购买渠道、是否已有待处理订单、订阅状态和 Codex 当前身份。已扣款未到账时保留去敏证据，等待合理同步并联系正确支持入口。

工程上准备三项降级：所有任务有 Issue 契约；Agent 输出进入分支而非直接主干；验收命令能由人类重跑。这样 ChatGPT Plus充值失败或账号会话失效时，任务只是切换执行者，不会失去状态。

任何第三方协助前，确认目标账号、套餐、实时费用、订阅周期、到账验证和异常规则。不要共享仓库写权限、长期会话、密码或完整支付信息。

## 10. 代码评审怎样区分“生成正确”和“工程正确”？

生成正确是代码看起来回答了问题；工程正确还包括边界测试、依赖影响、性能、安全、可维护性与回滚。

评审时先读任务契约，再看 diff，然后运行独立测试。对 Agent 的解释不应高于可重复证据。若它说“所有测试通过”，CI 才是最终依据。

高风险变更要求两人评审；涉及身份、支付、数据迁移和工作流权限时，不能因为改动行数少就降低等级。

## 11. 落地检查清单

- [ ] 根目录已有简洁的 `AGENTS.md`；
- [ ] 子目录特殊规则使用更深层指令覆盖；
- [ ] 每个 Agent 任务有范围与非目标；
- [ ] 同一错误两次后停止并报告；
- [ ] 敏感目录受 CODEOWNERS 或审批保护；
- [ ] CI 以最小权限运行；
- [ ] Agent 不接触生产密钥和真实支付数据；
- [ ] PR 记录命令、重试和验收结果；
- [ ] ChatGPT 身份任务与 API 自动化分账；
- [ ] 订阅异常时人类能接管并重放测试；
- [ ] 套餐、credits 和模型信息以实时官方页面核对；
- [ ] 合并前准备回滚方案。

## 12. FAQ

### Q1：AGENTS.md 越长越好吗？

不是。保持可执行、可验证，优先引用仓库脚本，避免复制百科式手册。

### Q2：有了 AGENTS.md 还需要提示词吗？

需要。长期规则放仓库，本次目标与范围放任务契约，两者各司其职。

### Q3：Codex 能直接提交到主分支吗？

不建议把高权限当便利。优先分支、Pull Request、CI 和人工评审。

### Q4：Plus 不够是不是只能 Pro充值？

先检查任务边界、上下文和重试。流程已经优化后，再比较高峰连续性的价值。

### Q5：Codex充值能替代 API 预算吗？

不能先假定。ChatGPT套餐、合资格 credits 与 API 项目是不同账本。

### Q6：如何防止 Agent 泄露密钥？

最小权限、密钥管理、脱敏夹具、扫描和 CI 门禁同时使用，不只依赖文字提醒。

### Q7：充值失败后是否应该换一个账号重试？

先核对原账号与原渠道。随意换账号会增加订阅归属和重复扣款问题。

### Q8：AI 生成的测试可靠吗？

可以作为候选，但要覆盖真实边界，并由独立 CI 重放。不要让同一解释同时充当结果和证据。

### Q9：团队需要保存完整 Agent 对话吗？

通常不需要。保存任务、命令、diff、测试和风险即可，敏感对话应遵循数据政策。

### Q10：什么时候必须人工停止 Agent？

需要生产凭证、破坏性迁移、范围明显扩大、相同错误重复或触及高风险目录时。

## 附录：一个适合放进 Issue 的 Agent 任务模板

Issue 不需要写成长文，但必须让陌生开发者能接手。标题描述用户可见行为，正文先写复现步骤和期望结果，再列允许路径、禁止范围、验收命令与回滚办法。涉及日期、套餐、模型或账号状态时，注明“以实时页面为准”，避免把临时事实固化成仓库真理。

Agent 开始前，由人类确认任务不需要生产凭证。执行中每完成一个阶段就留下简短状态：已复现、已补测试、已修复、全量测试通过。若中途受限，分支、diff 和命令记录足以让下一位继续。

评审者则从反方向验收：先运行失败用例证明问题存在，再运行修复后用例，最后检查无关文件和权限变化。不要只看 Agent 的总结，也不要因为 CI 绿色就忽略业务边界。

当订阅或登录异常时，这个模板就是降级协议。人类可以按相同步骤继续，不必重新追问会话历史；恢复后 Agent 也能从 Issue 与分支状态接续，而不是重新扫描整个仓库。

还可以给每个任务指定一位人工负责人。负责人不需要盯住每一步，但必须能判断范围是否扩大、测试是否可信、异常是否应停止。这样责任不会被“这是 Agent 做的”一句话稀释，仓库也不会把自动化误当成无人负责。

## 结论：容量决定能跑多远，门禁决定能不能合并

高质量 Codex 工作流把规则写进仓库，把目标写进 Issue，把证据交给 CI，把套餐放在容量层。这样无论 Plus、Pro、模型或使用规则如何变化，工程底座都不会跟着摇摆。

可记住的顺序是：**先身份、再范围；先测试、再合并；先减少无效重试，再为高价值容量付费。**

> **原创封面提示词**：16:9 GitHub 技术文档封面，暖米白背景，浅薄荷绿仓库文件树连接浅蓝紫 CI 流水线，中央一张写有 AGENTS.md 的白色卡片，淡杏橙盾牌表示门禁，标题“管住 AI Agent”，极简清爽，无暗黑霓虹、无商标。

> ## [gptupcn.com｜开通或续费前，请核对目标账号、套餐、实时费用、周期、到账及异常规则](https://gptupcn.com)

