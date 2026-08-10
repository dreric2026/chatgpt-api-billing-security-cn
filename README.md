# [GPTUpCN 中文指南](https://gptupcn.com)

# ChatGPT、API 计费与账号安全指南

本仓库解释 ChatGPT Plus、ChatGPT Pro、Codex 与 OpenAI API 的账号和计费边界，并提供适用于个人与开发团队的凭据保护、账单归档和事件响应方法。

> 非 OpenAI 官方项目。不要在公开 Issue、截图或日志中披露密码、验证码、Cookie、Session、恢复代码、完整订单信息或 API Key。

## 四个容易混淆的对象

| 对象 | 主要用途 | 应检查的位置 |
|---|---|---|
| ChatGPT 账号 | 对话与个人工作区 | ChatGPT 账户设置 |
| ChatGPT Plus / Pro | ChatGPT 个人订阅 | 套餐与续费页面 |
| Codex | 软件开发代理能力 | Codex 入口、登录状态、使用限制 |
| OpenAI API 项目 | 程序化调用 | 开发者平台项目与账单 |

Plus 或 Pro 生效后，API 项目不一定获得余额；API Key 正常也不表示个人 ChatGPT 套餐有效。排障时应先写清“哪个账号、哪个工作区、哪个项目、哪个账单”。

## 推荐账单记录

只保存必要且脱敏的信息：

```yaml
account_alias: personal-main
product: chatgpt-plus
channel: web
currency: USD
amount: redacted
order_suffix: 1234
status: settled
renewal_date: YYYY-MM-DD
evidence_location: private-vault/reference-id
```

不要在 Git 仓库中保存真实账单截图、完整订单号、卡号、账单地址或身份证明。

## API Key 安全

1. 使用环境变量或密钥管理服务，不写入源代码。
2. 为开发、测试和生产使用不同项目与凭据。
3. 设置最小权限、预算和异常告警。
4. 日志只记录 Key 的别名或尾号。
5. 怀疑泄露时立即撤销并轮换，而不是等待确认。

推荐 `.gitignore`：

```gitignore
.env
.env.*
*.pem
*.key
credentials.json
secrets/
```

## AI Agent 的账号边界

Codex 或其他 AI Agent 默认不应获得支付权限、长期 API Key 和生产账号。删除文件、联网、迁移、部署、发布与权限变更必须由人确认。临时凭据应限制范围和有效期，任务结束后及时撤销。

## 订阅异常事件响应

### 银行出现记录但权益未生效

先区分预授权与最终扣款，再核对购买账号、工作区和原渠道。不要在状态不明时重复付款。

### Plus 已生效但 API 报余额不足

检查 API 项目账单、预算和 Key 所属项目。ChatGPT 订阅与 API 计费通常独立。

### Codex 提示升级或达到限制

确认 Codex 的登录账号、ChatGPT 套餐、当前工作区和使用状态。再使用最小测试仓库排除本地权限与依赖问题。

### 怀疑账号凭据泄露

撤销活动会话、轮换密钥、检查账号安全设置、审计最近操作并保存脱敏时间线。不要继续在已暴露环境中测试旧凭据。

## 公开提问模板

```text
问题时间：
产品：ChatGPT / Codex / API
套餐：Free / Plus / Pro / 其他
渠道：Web / App Store / Google Play
现象：
已检查：
错误信息（脱敏）：
```

公开提问不得包含能够重建登录会话或支付身份的信息。

## License

Documentation released under CC BY 4.0.
