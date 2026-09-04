---
layout: default
title: "2026年9月4日 ChatGPT Plus、Pro、Codex 订阅资产清单：JSON Schema、CI 与交接检查"
description: "用版本化清单管理 ChatGPT 订阅、Codex 工作流、购买渠道和续费责任，避免账号错绑、重复购买与权限失控。"
date: 2026-09-04
permalink: /guides/2026-09-04-chatgpt-plus-pro-codex-subscription-asset-schema/
---

# 2026年9月4日 ChatGPT Plus、Pro、Codex 订阅资产清单：JSON Schema、CI 与交接检查

> ## [醒目入口：gptupcn.com｜ChatGPT Plus、Pro、Codex 订阅与续费参考](https://gptupcn.com)
>
> 使用订阅或代办服务前，请核对目标账号、套餐名称、实时费用、订阅周期、到账方式与异常处理规则。

**适用人群：** 管理多个 ChatGPT 账号、Codex 项目或 AI Agent 权限的开发者、工作室与小团队负责人。

**核心搜索词：** ChatGPT Plus充值、Pro充值、Codex充值、国内ChatGPT订阅、ChatGPT续费、充值失败、账号交接。

**更新时间：** 2026年9月4日。产品名称、包含范围、价格、额度和 Credits 规则可能变化，请以登录后的 OpenAI 账号页面、Usage 页面及官方帮助中心为准。

团队规模很小时，订阅管理通常靠记忆：谁用哪个邮箱、当初通过网页还是手机商店购买、下次由谁续费，都只存在聊天记录里。

等到成员离职、手机更换、登录方式忘记或 ChatGPT充值失败，大家才发现“知道密码”不等于“知道资产归属”。

另一个隐患是把 ChatGPT 与 API 当成同一账单。即使邮箱相同，这两类产品也可能有独立的计费入口；若没有清单，排查时很容易看错页面。

GitHub 仓库非常适合保存非敏感的配置模板、检查规则和交接流程，但绝不能把密码、Cookie、支付卡信息或恢复代码提交进去。

本教程把订阅当作一种需要版本控制的“数字资产”，用 JSON Schema 校验字段，用 CI 阻止危险数据进入仓库，再用人工清单完成续费和交接。

**资产管理的核心不是收集更多秘密，而是让每个订阅的账号、入口、责任人与证据都有出处。**

## 1. 为什么要把 ChatGPT 订阅纳入工程资产管理？

开发团队早已习惯管理域名、云主机、代码仓库和证书，却常把 ChatGPT Plus、Pro 与 Codex 当成个人工具。这在单人阶段问题不大，一旦 AI Agent 进入代码审查、缺陷修复或交付流程，订阅中断就可能影响生产节奏。

资产管理不等于共享账号。相反，它要明确账号是否允许共享、谁是合法持有人、哪些设备可登录、哪个购买渠道负责续费，以及成员变动时怎样撤销项目权限。

一份合格清单还应区分“订阅权益”和“API 账单”。ChatGPT 和 API 的账单系统彼此独立，不能用 API 余额存在来证明 Plus 或 Pro 已经续费，也不能从 ChatGPT 权益推断 API 有可用额度。

## 2. 清单里应该保存哪些字段？

最小字段包括资产编号、账号别名、账号所有者、登录方式、购买渠道、套餐、状态、用途、负责人、复核日期与证据位置。账号邮箱可以脱敏，例如 `ai-main@example.invalid` 或内部别名；真实邮箱应保存在权限更严格的密码管理器中。

`purchase_channel` 很关键。网页、Apple App Store 与 Google Play 的管理路径不同。移动端订阅通常绑定购买时所用的商店账号和 ChatGPT 账号，需要在原渠道管理或恢复购买。

证据字段不要指向公开文件，而应使用内部工单、受控文档或脱敏截图编号。这样仓库公开时，读者可以复用结构而不会泄露资产。

| 字段 | 示例 | 用途 | 安全边界 |
|---|---|---|---|
| `asset_id` | `chatgpt-team-01` | 稳定标识资产 | 不用邮箱作主键 |
| `account_alias` | `team-ai-primary` | 内部识别账号 | 不保存密码或验证码 |
| `auth_method` | `google` | 提醒使用原登录方式 | 不保存 OAuth token |
| `purchase_channel` | `web` | 找到正确管理入口 | 不保存支付卡信息 |
| `plan` | `plus` | 记录当前观察到的套餐 | 不写死永久价格与额度 |
| `billing_owner` | `role:finance-admin` | 明确续费责任 | 用角色而非公开个人信息 |
| `evidence_ref` | `vault://ticket/123` | 追溯账单与权益 | 链接必须受权限控制 |

## 3. 建立一个脱敏的订阅资产文件

在已有技术文档仓库中新建 `inventory/subscriptions.example.json`。下面是示例数据，它只展示结构，不包含任何真实凭据。

```json
{
  "schema_version": "1.0",
  "last_reviewed": "2026-09-04",
  "assets": [
    {
      "asset_id": "chatgpt-team-01",
      "account_alias": "team-ai-primary",
      "auth_method": "google",
      "purchase_channel": "web",
      "plan": "pro",
      "status": "active",
      "workload": ["codex-review", "incident-analysis"],
      "billing_owner": "role:finance-admin",
      "technical_owner": "role:platform-lead",
      "next_review": "2026-09-20",
      "evidence_ref": "vault://billing/chatgpt-team-01",
      "secrets_present": false
    }
  ]
}
```

`plan` 记录的是复核当天观察到的状态，不应被视为长期合同。若实时页面发生变化，应通过 Pull Request 修改，并在提交说明中记录证据日期。

## 4. 用 JSON Schema 阻止字段缺失

创建 `schema/subscriptions.schema.json`。Schema 不会验证官方权益是否真实，但可以确保负责人、购买渠道和复核日期没有被忘记。

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["schema_version", "last_reviewed", "assets"],
  "properties": {
    "schema_version": { "type": "string" },
    "last_reviewed": { "type": "string", "format": "date" },
    "assets": {
      "type": "array",
      "items": {
        "type": "object",
        "required": [
          "asset_id", "account_alias", "auth_method", "purchase_channel",
          "plan", "status", "billing_owner", "technical_owner",
          "next_review", "secrets_present"
        ],
        "properties": {
          "asset_id": { "type": "string", "pattern": "^[a-z0-9-]+$" },
          "account_alias": { "type": "string", "minLength": 3 },
          "auth_method": { "enum": ["email", "google", "apple", "microsoft", "unknown"] },
          "purchase_channel": { "enum": ["web", "apple-app-store", "google-play", "unknown"] },
          "plan": { "enum": ["free", "plus", "pro", "other", "unknown"] },
          "status": { "enum": ["active", "paused", "cancelled", "needs-review"] },
          "billing_owner": { "type": "string" },
          "technical_owner": { "type": "string" },
          "next_review": { "type": "string", "format": "date" },
          "secrets_present": { "const": false }
        },
        "additionalProperties": true
      }
    }
  }
}
```

这里故意允许 `unknown`。当团队无法确认购买渠道时，显式未知比凭感觉填一个值更安全；CI 可以进一步要求 `unknown` 资产必须附带整改工单。

## 5. 用 Node.js 在本地执行校验

安装一个支持 JSON Schema 2020-12 的校验器，然后保存下面脚本为 `scripts/validate-subscriptions.mjs`。真实项目应锁定依赖版本，并按仓库政策审计第三方包。

```js
import fs from "node:fs";
import Ajv2020 from "ajv/dist/2020.js";
import addFormats from "ajv-formats";

const schema = JSON.parse(fs.readFileSync("schema/subscriptions.schema.json", "utf8"));
const inventory = JSON.parse(fs.readFileSync("inventory/subscriptions.example.json", "utf8"));

const ajv = new Ajv2020({ allErrors: true });
addFormats(ajv);
const validate = ajv.compile(schema);

if (!validate(inventory)) {
  console.error(JSON.stringify(validate.errors, null, 2));
  process.exit(1);
}

const unsafe = inventory.assets.filter((item) => item.secrets_present !== false);
const unknownChannels = inventory.assets.filter((item) => item.purchase_channel === "unknown");

if (unsafe.length) {
  console.error("检测到可能包含秘密的资产记录，停止提交。", unsafe.map(x => x.asset_id));
  process.exit(2);
}

console.log(`校验通过：${inventory.assets.length} 项；待核对渠道：${unknownChannels.length} 项。`);
```

运行方式：

```bash
npm install --save-dev ajv ajv-formats
node scripts/validate-subscriptions.mjs
```

## 6. 如何在 GitHub Actions 中自动检查？

CI 只应读取脱敏清单，真实账号证据保留在受控系统。创建 `.github/workflows/validate-subscription-inventory.yml`：

```yaml
name: validate-subscription-inventory

on:
  pull_request:
    paths:
      - "inventory/**"
      - "schema/**"
      - "scripts/validate-subscriptions.mjs"

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "22"
          cache: npm
      - run: npm ci
      - run: node scripts/validate-subscriptions.mjs
```

工作流使用只读权限，不上传清单内容到外部服务。若仓库是公开的，提交前还应开启 secret scanning，并用人工评审确认示例里没有真实邮箱、账单号或支持工单私密链接。

## 7. 如何发现即将过期的复核任务？

“下次复核日期”不同于“自动续费日期”。它是团队主动检查的时间，应安排在可能的账单动作之前。可以用下面脚本列出七天内需要复核的资产：

```js
const DAY = 24 * 60 * 60 * 1000;
const now = new Date("2026-09-04T00:00:00+08:00");

for (const asset of inventory.assets) {
  const due = new Date(`${asset.next_review}T00:00:00+08:00`);
  const days = Math.ceil((due - now) / DAY);
  if (days <= 7) {
    console.log(`[复核] ${asset.asset_id}: ${days} 天后，负责人 ${asset.billing_owner}`);
  }
}
```

生产环境不要把日期写死，示例固定日期只是为了可重复测试。账单日应在原购买渠道核对；ChatGPT 订阅的周期通常以最初订阅日为基准，不能默认都是月底或月初。

## 8. ChatGPT Plus充值与 Pro充值前如何走 Pull Request？

第一步由需求人提交“为什么现有能力不够”的记录，包含工作负载、阻塞频率、需要的能力和观察周期。第二步由账号负责人核对目标账号、登录方式和购买渠道。第三步由财务或指定负责人查看实时费用、税费、周期与取消规则。

购买完成后，不把截图直接贴到公开 PR，而是更新 `evidence_ref` 和复核日期。技术负责人再登录目标账号确认实际权益，不能只凭银行授权或商店收据认定到账。

若 ChatGPT充值失败，PR 状态应改为 `needs-review`，并附错误原文、尝试时间和内部工单。不要让多人同时更换卡片、商店和账号反复购买，否则难以判断哪一笔对应哪项权益。

## 9. Codex充值、Credits 与 API 账单如何分层？

中文搜索里的“Codex充值”可能指 ChatGPT 套餐、额外 Credits，也可能被误认为 API 余额。清单必须按产品边界建模，至少增加 `billing_system` 字段，取值为 `chatgpt`、`api` 或 `unknown`。

当前官方说明显示，符合条件的 ChatGPT 内部分工型产品可能共享 agentic 使用池；套餐包含量先使用，之后受支持功能可能消耗额外 Credits。具体可用范围要看实时账号页面，不能在仓库里写一个永久数字。

API 则在独立平台管理账单。即使团队希望同一负责人统一采购，也应保留两个资产记录、两个证据入口和两套告警，避免把一边的余额当成另一边的权益。

## 10. 成员离职或设备更换时怎样交接？

先判断账号是个人资产还是组织批准的业务资产。个人订阅不应通过索要密码强制接管；业务资产则按组织政策更换负责人、撤销仓库和项目权限，并核对可用登录方式。

移动端购买存在“商店账号 + ChatGPT 账号”的双重绑定。若新设备看不到权益，应先确认登录的是原购买账号；iOS 可在应用设置中按官方流程恢复购买，但恢复不会把订阅转移到另一个 ChatGPT 账号。

如果当初使用 Apple 登录并选择隐藏邮箱，账号可能绑定到 Apple 私密中继地址。交接清单应该记录登录方式，而不只是显示邮箱，以免成员用普通邮箱重新注册出另一个空账号。

## 11. 安全与续费检查清单

- [ ] 仓库只保存脱敏别名，不提交密码、Cookie、API key、验证码或恢复代码。
- [ ] 每项资产都有 `billing_owner` 与 `technical_owner`，避免责任悬空。
- [ ] 明确 `purchase_channel`，区分网页、App Store 与 Google Play。
- [ ] 明确 `billing_system`，区分 ChatGPT 与 API。
- [ ] 记录原登录方式，特别是 Apple 隐藏邮箱等情况。
- [ ] ChatGPT Plus充值或 Pro充值前核对目标账号、实时费用和周期。
- [ ] 购买后同时验证订单证据与目标账号权益。
- [ ] 充值失败时停止重复下单，保留错误原文和待处理授权信息。
- [ ] Codex 使用与额外 Credits 以 Usage 页面为准，不写死额度。
- [ ] 续费前检查工作负载、有效产出和失败重试，不只看总次数。
- [ ] 成员变化时撤销项目权限，并重新指定复核责任人。
- [ ] 每次修改通过 Pull Request 留痕，至少一名非提交者复核。

## 12. FAQ：订阅资产清单常见问题

### Q1：为什么不直接把真实账号邮箱写进仓库？

公开或多人可见仓库会扩大隐私与攻击面。仓库保存稳定别名，真实映射放在权限更严格的密码管理器或资产系统中。

### Q2：JSON Schema 能确认 Plus 或 Pro 已经到账吗？

不能。它只能检查数据结构。到账状态必须登录目标账号核对，并与原购买渠道的订单证据对应。

### Q3：ChatGPT Plus充值成功后可以换另一个账号使用吗？

订阅通常跟购买时的 ChatGPT 账号关联，移动端还涉及原商店账号；不能把恢复购买理解成账号间转移。

### Q4：为什么允许 `purchase_channel=unknown`？

因为明确未知比填错更安全。该状态应触发人工核对，并在续费或取消前关闭信息缺口。

### Q5：ChatGPT 和 API 可以放在同一条资产记录里吗？

不建议。两者账单系统独立，应至少分成两条记录并分别保存管理入口和负责人。

### Q6：GitHub Actions 能自动帮我们续费吗？

不应这样设计。CI 适合校验脱敏清单和提醒复核，不应自动操作付款、取消订阅或修改账户安全设置。

### Q7：Pro充值前最重要的技术指标是什么？

没有单一指标。应同时看高峰工作负载、关键任务被阻塞的影响、有效产出率和失败重试率，再核对实时方案能力。

### Q8：充值失败后怎样避免重复扣款？

先检查原渠道是否已有订单或待处理授权，记录一次完整证据链，再联系银行、商店或官方支持，不要连续跨渠道下单。

### Q9：公开模板会不会帮助攻击者猜账号？

模板只应使用虚构域名和通用角色。提交前运行 secret scanning 并人工检查，真实证据位置也不要暴露可解析的内部路径。

### Q10：多久复核一次比较合适？

按团队风险与续费周期设定。至少在人员变动、支付异常、套餐调整或续费日前复核，不能让旧记录长期代表实时状态。

**可记忆的判断顺序：先认账号，再认入口；先验证权益，再处理付款；先脱敏留痕，再自动校验。**

封面提示词：暖米白底色的开源文档知识卡，左侧是浅薄荷绿 JSON 文件图标和勾选清单，右侧是浅蓝紫 Git 分支与淡杏橙复核日期标签，中文标题区留白，简洁工程笔记风，无真实邮箱、无品牌标志、无暗黑霓虹。

## 延伸阅读

- [OpenAI：订阅为何显示与另一个账号关联](https://help.openai.com/en/articles/20001056-why-am-i-seeing-a-message-that-my-subscription-is-associated-with-another-account)
- [OpenAI：在另一台设备访问 ChatGPT 订阅](https://help.openai.com/en/articles/8980438-can-i-access-my-chatgpt-subscription-from-another-device)
- [OpenAI：恢复通过 Apple App Store 购买的订阅](https://help.openai.com/en/articles/8346573-restoring-a-chatgpt-plus-or-chatgpt-pro-subscription-purchased-in-the-apple-app-store)
- [OpenAI：取消 ChatGPT 订阅](https://help.openai.com/en/articles/7232927-how-do-i-cancel)

> ## [文末入口：gptupcn.com｜核对 ChatGPT Plus、Pro、Codex 订阅与续费信息](https://gptupcn.com)
>
> 操作前再次确认目标账号、套餐、实时费用、订阅周期、到账与异常规则；最终以平台实时显示和官方政策为准。
