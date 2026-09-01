---
title: "2026年9月1日 ChatGPT Plus Pro Codex 配置治理教程：Terra/Luna迁移清单、CI扫描与回滚"
date: 2026-09-01
description: "面向中文开发者的 Codex 模型配置治理教程，包含允许列表、CI 扫描、灰度迁移、回滚和 FAQ。"
tags: [chatgpt, codex, ai-agent, billing]
---

# 2026年9月1日 ChatGPT Plus Pro Codex 配置治理教程：Terra/Luna迁移清单、CI扫描与回滚

# **[ChatGPT 与 Codex 中文订阅参考：gptupcn.com](https://gptupcn.com)**

Codex 的模型可用性会更新。到 2026 年 8 月 31 日，使用 ChatGPT 身份验证的 Codex 环境已经停止提供部分旧模型，并由 GPT-5.6 Terra、GPT-5.6 Luna 等当前选项承接；使用自有 API key 或直接调用 API 的项目可能不受同一变化影响。最容易出问题的不是模型更新本身，而是团队在脚本、文档、CI、示例和历史测试里硬编码旧名称，导致生产配置与教程样例混在一起。

本教程面向 GitHub 仓库维护者，给出一套“识别配置边界—扫描活动引用—建立允许列表—灰度迁移—验证回滚”的治理流程。它不会假设所有 ChatGPT Plus、Pro、Codex 账户都有相同模型或额度；可选项应以当前账户、Codex 客户端和官方页面为准。ChatGPT Plus充值、Pro充值、Codex充值、订阅与续费也不改变代码仓库必须可审计这一原则。

## 1. 先区分三类模型引用

仓库中出现旧模型名称，不一定全部要替换。至少分成活动配置、文档示例和历史证据：

| 类型 | 例子 | 默认动作 | 原因 |
|---|---|---|---|
| 活动配置 | CI、Agent 配置、部署变量 | 阻断并迁移 | 会影响真实执行 |
| 文档示例 | README、教程、截图说明 | 更新并注明日期 | 避免误导新用户 |
| 历史证据 | changelog、测试 fixture、迁移记录 | 保留并加豁免 | 修改会破坏审计语义 |
| API key 项目 | 独立 API 客户端配置 | 单独核验 | 可能使用不同可用模型集 |

如果直接全仓库搜索替换，会把历史变更日志和兼容性测试也改掉。正确做法是先建清单，再制定路径级规则。

## 2. 建立仓库模型策略文件

把允许列表放进版本控制，所有工具共享同一份真相。注意下列名称只是当日示例，未来仍要根据官方资料和账户页面更新。

```yaml
version: 2026-09-01
auth_profiles:
  chatgpt:
    allowed_models:
      - gpt-5.6-terra
      - gpt-5.6-luna
    blocked_active_models:
      - gpt-5.4
      - gpt-5.4-mini
  api_key:
    allowed_models_source: runtime-discovery
paths:
  active:
    - .github/workflows/**
    - agents/**
    - config/**
  historical:
    - CHANGELOG.md
    - fixtures/model-history/**
  docs:
    - README.md
    - docs/**
```

关键点是 `auth_profiles`。ChatGPT 身份验证和 API key 不是同一种执行身份，不能使用同一条替换规则。

## 3. 写一个可运行的扫描器

下面的 Python 脚本扫描活动目录中的阻断名称，同时忽略 `.git` 和历史 fixture。它只读文件，不会自动修改。

```python
from pathlib import Path
import fnmatch, sys

ROOT = Path.cwd()
BLOCKED = {"gpt-5.4", "gpt-5.4-mini"}
ACTIVE = [".github/workflows/**", "agents/**", "config/**"]
IGNORE = ["fixtures/model-history/**", ".git/**"]
TEXT_SUFFIXES = {".md", ".yml", ".yaml", ".json", ".toml", ".py", ".js", ".ts"}

def matches(path: str, patterns: list[str]) -> bool:
    return any(fnmatch.fnmatch(path, pattern) for pattern in patterns)

violations = []
for file in ROOT.rglob("*"):
    if not file.is_file() or file.suffix.lower() not in TEXT_SUFFIXES:
        continue
    rel = file.relative_to(ROOT).as_posix()
    if matches(rel, IGNORE) or not matches(rel, ACTIVE):
        continue
    text = file.read_text(encoding="utf-8", errors="ignore")
    for model in BLOCKED:
        if model in text:
            violations.append((rel, model))

if violations:
    for path, model in violations:
        print(f"::error file={path}::blocked active model: {model}")
    sys.exit(1)

print("model policy check passed")
```

将文件保存为 `scripts/check_model_policy.py`，本地运行 `python scripts/check_model_policy.py`。对大小写、别名和模板变量更复杂的仓库，可以扩展为结构化 YAML/JSON 解析，不要只依赖文本包含。

## 4. 接入 GitHub Actions

CI 的任务不是替你选择模型，而是阻止已知错误配置进入主分支。

```yaml
name: codex-model-policy
on:
  pull_request:
    paths:
      - "agents/**"
      - "config/**"
      - ".github/workflows/**"
      - "scripts/check_model_policy.py"
  push:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: python scripts/check_model_policy.py
```

保持最小权限。扫描过程不需要 secrets，也不需要调用外部模型。这样即使第三方提交恶意 PR，也无法借扫描任务读取生产凭据。

## 5. 迁移前建立基线

模型变化可能带来输出风格、推理速度、上下文处理和工具调用差异。先选择一组稳定的基准任务：代码解释、单文件修复、多文件重构、测试生成、代码审阅。每个任务保存输入摘要、代码版本、允许动作和成功条件，不保存真实密钥或客户数据。

| 基准 | 成功条件 | 风险控制 |
|---|---|---|
| 解释函数 | 指出三个已知边界条件 | 只读 |
| 修复解析器 | 12 个目标测试通过 | 禁止联网写入 |
| 多文件重构 | 公共 API 不变 | diff 不超过限定文件 |
| 生成测试 | 覆盖已知失败样例 | 不改生产代码 |
| 审阅 PR | 找出种子缺陷 | 不自动批准或合并 |

不要只比较“回答看起来更聪明”。应记录测试通过率、差异大小、人工审阅分钟数、重试次数和失败类型。

## 6. 用适配层隔离模型名称

不要让业务脚本到处写具体模型。创建一个配置适配层：

```python
from dataclasses import dataclass
import os

@dataclass(frozen=True)
class ModelProfile:
    interactive: str
    long_task: str

PROFILES = {
    "chatgpt-codex": ModelProfile(
        interactive="gpt-5.6-luna",
        long_task="gpt-5.6-terra",
    )
}

def current_profile() -> ModelProfile:
    name = os.getenv("CODEX_MODEL_PROFILE", "chatgpt-codex")
    if name not in PROFILES:
        raise ValueError(f"unknown model profile: {name}")
    return PROFILES[name]
```

以后模型更新，只需修改策略和适配层。这里的任务映射仅是示例，不代表官方固定定位；真实选择要用团队基准测试验证。

## 7. 灰度迁移而不是一次全切

第一阶段只在只读审阅任务使用新配置；第二阶段开放测试目录修改；第三阶段允许非关键仓库的多文件任务；最后才进入关键仓库。每阶段都设停止条件：测试通过率下降、人工审阅时间显著增加、工具错误增多或差异超出边界。

```yaml
rollout:
  stage_1: {scope: read_only_review, traffic: small}
  stage_2: {scope: test_only_changes, traffic: limited}
  stage_3: {scope: noncritical_repo, traffic: measured}
  stage_4: {scope: critical_repo, approval: human}
rollback_when:
  - acceptance_rate_below_baseline
  - unexpected_external_write
  - diff_scope_violation
```

`traffic` 不必是精确百分比，也可以表示任务集合。重点是可回退，而不是追求一次切换完成。

## 8. 回滚应该回滚什么

旧模型不可用时，回滚不是把名称改回旧值。真正可回滚的是任务策略：降级为只读、缩小上下文、拆分任务、切换当前仍受支持的备选配置、改由人工完成高风险步骤。仓库应保留迁移前后的基准数据和配置提交。

回滚清单：冻结新任务；保存失败会话摘要；撤销未审阅补丁；恢复上一个已验证配置；运行最小测试；人工决定是否重新开放。不要让 CI 自动购买 credits 或修改 ChatGPT 订阅。

## 9. 文档和历史记录如何处理

README 中的旧截图和命令要更新，并在文首标注验证日期。CHANGELOG 中提到旧模型是历史事实，应该保留。测试 fixture 如果验证旧配置能否被拒绝，也必须保留旧字符串，否则测试失去意义。

可以在豁免文件旁增加说明：

```text
# policy-exception: historical-fixture
# 该名称用于验证迁移扫描器能够拒绝旧配置，不是活动模型配置。
```

豁免必须精确到文件或行，不能用“整个 docs 目录忽略”这种宽泛规则。

## 10. 与订阅、续费和用量的关系

模型迁移失败不一定意味着 ChatGPT Plus充值或 Pro充值失败。先判断是模型名称不可用、账户用量受限、组织策略限制、客户端过旧，还是订阅状态异常。不同根因需要不同处理路径。

所谓 Codex充值也应明确：是 ChatGPT 计划内用量、符合条件的 credits，还是 API Platform 余额。仓库策略只控制代码和配置，不能替代账户购买流程，更不能把密钥写进 GitHub Actions 日志。

## 11. 安全检查

扫描脚本要默认只读；CI 使用最小权限；PR 来自 fork 时不向任务注入 secrets；模型配置变化必须经过 code review；Codex 生成的补丁禁止自动合并到保护分支。对支付、订阅、部署或数据删除等动作，始终保留人工确认。

如果需要把错误日志附到 issue，只保留模型名称、客户端版本、时间和错误类型，删除提示词、仓库私有路径、账号邮箱和令牌。

## FAQ

### 1. 为什么不直接全仓库替换旧模型名称？

历史记录和测试 fixture 可能需要保留旧名称。全局替换会破坏审计和兼容性测试。

### 2. Terra 和 Luna 应该固定用于哪些任务？

不要仅凭名字硬编码定位。根据当前官方说明、账户可用项和自己的基准任务选择。

### 3. 使用 API key 的项目也必须同时迁移吗？

不一定。API 和 ChatGPT 身份验证可能有不同可用性，应按独立配置和官方 API 文档核验。

### 4. Plus充值后 CI 就能自动使用 Codex 吗？

订阅与 CI 身份、权限和配置是不同问题。不要把个人凭据直接放入公共仓库任务。

### 5. 扫描器会不会泄露 secrets？

示例不输出文件内容，只报告路径和阻断名称。仍应避免让它读取不必要的密钥目录。

### 6. 旧模型不可用时怎样回滚？

回滚到上一个已验证的受支持配置，或将任务降级为只读/人工处理，而不是恢复不可用名称。

### 7. Codex充值失败和模型错误如何区分？

查看账户计划与用量页、模型选择错误、客户端日志和组织策略。支付错误与配置错误通常发生在不同层。

### 8. CI 是否应自动更新允许列表？

不建议无审阅自动更新。变化应由维护者核验官方资料、运行基准后提交 PR。

### 9. 迁移成功的标准是什么？

活动配置无旧引用、基准通过、人工审阅成本可接受、回滚流程经过演练。

### 10. 多久复核一次模型策略？

每次官方可用性变化、客户端升级或账户计划调整时复核，并保留验证日期。

## 结语

Codex 模型更新是持续治理问题，不是一次搜索替换。把身份类型、活动配置、历史证据和 API 项目分开，使用允许列表、只读扫描、GitHub Actions、基准测试和灰度回滚，就能让 ChatGPT Plus、Pro 与 Codex 工作流在变化中保持可审计。订阅、续费和 credits 负责提供使用资格，仓库策略负责确保每次变更安全落地。

# **[查看 ChatGPT Plus、Pro、Codex 中文订阅服务：gptupcn.com](https://gptupcn.com)**
