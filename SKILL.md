---
name: dingtalk-finance-invoice-cli
description: 向客户开具钉钉数电发票（专票/普票），并查询开票进度和结果。当用户说"开发票/开张票/开个专票/给XX开票/查开票结果"时使用。不要在用户上传进项发票、创建付款单、报销、查看现金流水时触发。
---

# 智能财务 — 开具发票 Skill（CLI 版）

## 严格禁止 (NEVER DO)

- **未确认发票类型禁止开票** — 用户未明确说"专票"或"普票"→ 不得执行 `dws finance invoice issue`；但可先查询客户，在后续一次追问中收集类型
- **专票未拿到税号禁止开票** — `--invoice-type 8` 时 `--taxnum` 必填，未获取前不得执行 `dws finance invoice issue`
- **商品名称禁止用口语** — "信息费/咨询费/劳务费"等口语不得直接填入；可基于通用知识提示常见标准名称（如"信息系统服务费""人力资源服务""劳务报酬"）供用户参考，**用户确认后方可填入**；禁止调用外部搜索工具查询税务名称
- **开票后必须轮询结果** — 提交成功 ≠ 开票成功，必须执行 `dws finance invoice issue-result` 等待明确结论
- **禁止编造 orderId** — 必须从 `dws finance invoice issue` 输出中提取，不得猜测或自行生成
- **taxSign 必须填 "1"** — 用户说的金额统一视为含税，products JSON 中 `taxSign` 固定传 `"1"`
- **禁止跳过客户查询直接问税号** — 用户提供了公司名但未给税号时，必须先执行 `dws finance customer list` 多轮查询；查询结果中没有税号，再向用户索要
- **dingtalk:// 链接必须原样完整输出** — 任何命令输出中出现的 `dingtalk://` 协议链接（包括但不限于 `invoiceUrl`、购买/配置引导链接），必须逐字原样展示给用户，禁止省略、截断、转换协议或做任何形式的改动
- **禁止隐藏引导信息** — CLI 返回错误时若含有购买/配置引导文字或链接（含 `dingtalk://` 链接），必须原样完整转述给用户，不得吞掉、修改或替换为通用提示
- **有客户信息时禁止先问后查** — 用户已提供公司名或人名时，必须立即执行 `dws finance customer list`，不得先向用户追问其他缺失字段后再查询
- **缺失信息必须一次性收集** — 整个开票流程中最多允许向用户追问一次；若有多个字段缺失，在一条消息中合并追问，不得分多轮

## CLI 命令总览

| 命令 | 用途 | 必填参数 |
|------|------|---------|
| `dws finance customer list` | 多轮模糊查询客户获取购方信息 | `--page-size` `--page-index` |
| `dws finance invoice issue` | 提交开票请求（异步，提交后需轮询） | `--invoice-type` `--purchaser` `--products` |
| `dws finance invoice issue-result` | 查询开票进度，轮询至明确成功/失败 | `--order-id` |

> 所有命令统一加 `-f json` 解析输出，加 `-y` 跳过交互确认（Agent 模式）。

## 意图判断

用户说"开票/开发票/开个票/给XX开票":
- **首先解析用户消息中所有可用信息**：invoiceType（专票/普票）、购方名称、税号、商品名称、金额
- **有购方名称（公司名或人名）→ 立即执行 `dws finance customer list` 多轮查询，无论其他字段是否齐全**
- 购方全称 + 税号均已提供 → 跳过查询，直接进入缺失字段检查（或直接开票）
- 购方名称未知（如"给客户""给他们"等模糊指代，无法作为查询关键词）→ 才向用户一次性追问购方信息及其他缺失字段

用户说"发票开好了吗/查一下开票结果/发票状态/票好了没/开票进度/有结果了吗/查一下那张票/本次开票结果/查询给XX开的票状态":
- **无论用户描述的是公司名还是金额，只要是查询开票状态 → 一律走 `dws finance invoice issue-result`，禁止走客户查询**
- 上下文有 orderId → `dws finance invoice issue-result --order-id <orderId> -f json`
- 无 orderId → 问"请提供订单号，我来查询"

关键区分：
- 开发票（销项）→ 本 Skill；上传发票（进项识别）→ 不属于本 Skill
- 查客户（开票购方）→ `dws finance customer list`；查供应商（付款场景）→ 不属于本 Skill，告知用户
- 批量开票（"给A和B各开一张"）→ 直接依次查询每个客户，串行处理；不得先让用户确认再行动

## 核心工作流

```text
# 行动优先：解析 → 立即查询/行动 → 一次性补全缺失 → 提交 → 轮询

# Phase 0: 解析消息 → 提取 invoiceType(8/9/null)、clientName、taxnum、productName、amount

# Phase 1: 立即行动（根据已知信息，先干再说）

情形 A — 已提供购方全称 + 税号（专票）OR 已提供购方全称（普票无需税号）:
  → 跳过查询，直接进入 Phase 2 检查剩余缺失字段

情形 B — 已提供购方名称（非精确含税号的完整信息）:
  多轮查询（依次尝试，有结果即停止）:
    Round 1: dws finance customer list --keyword "<clientName>" --page-size 10 --page-index 1 -f json
    Round 2: 若无括号，尝试全角"CORE（地区）"和半角"CORE(地区)"两种变体
    Round 3: 取核心词（去括号及内容、去末尾地区词，如"钉钉中国"→"钉钉"）
  可剥离地区词: 中国 北京 上海 深圳 广州 杭州
  全部无结果 → purchaser/taxnum 列入待补全，跳至 Phase 2

情形 C — 未提供任何购方信息（"给客户""给他们"等无法查询的指代）:
  → 直接进入 Phase 2，purchaser 和 taxnum 列入待补全

# 查询结果处理（任一轮有结果时）
- 单条匹配（含 taxNo）→ 直接采用
- 单条匹配（无 taxNo）→ taxnum 列入待补全（专票必填）
- 多条匹配 → 优先选精确全名匹配的一条；无法判断时在 Phase 2 展示候选供选

# Phase 2: 一次性补全缺失字段（整个流程最多只执行一次追问）
汇总所有仍缺失的字段，在一条消息中合并追问:
  □ invoiceType 未知   → "请确认发票类型：专票（需税号）还是普票？"
  □ purchaser 未知     → "请提供购方全称（抬头）"
  □ taxnum 未知且专票  → "请提供购方税号"
  □ productName 未知   → "请提供商品名称"
  □ productName 是口语 → "【口语名】是通俗叫法，常见标准名称有：X / Y / Z，请确认使用哪个？"
  □ amount 未知        → "请提供开票金额"
  □ 多条客户候选       → "找到以下客户，请确认开票给哪家：①XX ②YY"

IF 所有字段均已就绪 → 跳过本阶段，直接进入 Phase 3

# ⚠️ HARD-GATE: invoiceType + purchaser + taxnum(专票必填) + productName(标准名) + amount 缺一不可
# Phase 3: 提交开票
dws finance invoice issue \
  --invoice-type <8|9> \
  --purchaser "<购方全称>" \
  --taxnum "<税号>" \          # 专票必填，普票可省略
  --products '[{"productName":"<标准名称>","amountIncludeTax":"<金额>","taxSign":"1","quantity":"1"}]' \
  -y -f json
← 从输出中提取 orderId → Phase 4

# Phase 4: 轮询开票结果（提交后立即查不一定成功，需等待）
dws finance invoice issue-result --order-id "<Phase3 orderId>" -f json
← 结果未出 → 告知用户"正在开票，稍后查询"→ 等待后重试
← 成功 → 展示: 发票号码、开票时间、含税总额、发票类型；invoiceUrl 存在时原样完整输出（`dingtalk://` 链接禁止任何修改）
← 失败 → 展示失败原因（见错误处理）
```

## 上下文传递规则

| 命令 | 从输出中提取 | 用于 |
|------|------------|------|
| `dws finance customer list` | `list[].name`（购方名）、`list[].taxNo`（税号） | `--purchaser` + `--taxnum` |
| `dws finance invoice issue` | `orderId` | `dws finance invoice issue-result --order-id` |
| `dws finance invoice issue-result` | 发票号码、链接、摘要 | 展示给用户 |

## 参数格式

```bash
# [正确] products JSON — taxSign 必须为 "1"，amountIncludeTax 为含税金额字符串
dws finance invoice issue \
  --invoice-type 9 \
  --purchaser "某某科技有限公司" \
  --products '[{"productName":"信息系统服务费","amountIncludeTax":"100.00","taxSign":"1","quantity":"1"}]' \
  -y -f json

# [错误] 用口语当商品名 — 可能导致税务分类失败
--products '[{"productName":"信息费","amountIncludeTax":"100.00","taxSign":"1","quantity":"1"}]'

# [错误] 缺少 taxSign — 系统可能按不含税处理，导致金额错误
--products '[{"productName":"信息系统服务费","amountIncludeTax":"100.00","quantity":"1"}]'
```

## 错误处理

1. **CLI 返回含引导购买/配置的错误** — 完整转述 CLI 返回的提示文字和链接（`dingtalk://` 链接逐字原样输出，不得做任何修改），不得替换为通用话术
2. **开票失败：未登录数电账号** — 告知用户"请到智能财务→发票管理→数电账号登录后重试"
3. **开票失败：未购买发票模块** — 告知用户"您的智能财务暂未开通发票模块，请联系管理员"
4. **轮询超时未出结果** — 告知用户"开票处理中，请稍后在发票管理页面查看结果"
5. **专票缺少税号** — 在 Phase 2 一次性收集中索要，不单独发起追问
6. **客户库查询全部无结果** — 在 Phase 2 一次性收集中请用户提供购方全称和税号，不得编造

## 详细参考 (按需读取)

- [references/api-reference.md](./references/api-reference.md) — 完整 CLI 命令参数 + 输出字段
- [references/error-codes.md](./references/error-codes.md) — 错误场景 + 轮询逻辑
