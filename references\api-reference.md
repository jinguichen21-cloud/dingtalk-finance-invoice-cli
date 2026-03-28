# CLI Reference — 智能财务 开具发票

> 按需读取，SKILL.md 已覆盖核心用法。本文件提供完整命令参数和输出字段路径。

---

## dws finance customer list

**用途**: 按关键词模糊查询客户列表，用于获取购方抬头和税号。支持多轮查询：原始名称 → 括号变体 → 核心词，依次尝试直至有结果。
**使用场景**: 用户说了客户名称，从客户库查询对应信息时使用。
**区分**: 本命令用于查客户（销项开票购方）；查供应商（付款单场景）不属于本 Skill。

```bash
dws finance customer list \
  --keyword "<客户名称关键词>" \   # 可选，为空时全量分页返回
  --page-size <每页数量> \         # 建议 10
  --page-index <页码> \            # 从 1 开始
  -f json
```

**输出关键字段路径**:
```
list[].name     → --purchaser（购方抬头）传给 dws finance invoice issue
list[].taxNo    → --taxnum（购方税号）传给 dws finance invoice issue
```

---

## dws finance invoice issue

**用途**: 向购方提交开具数电发票请求（异步）。提交成功不代表开票完成，需轮询查询结果。
**使用场景**: 用户明确要开发票，且已收集完购方信息和商品信息时调用。
**区分**: 本命令只提交开票，不返回最终结果；最终结果用 `dws finance invoice issue-result` 查询。

```bash
dws finance invoice issue \
  --invoice-type <8|9> \          # 8=数电专票，9=数电普票
  --purchaser "<购方抬头>" \       # 公司全称或个人姓名，必填
  --taxnum "<购方税号>" \          # 数电专票必填，普票可省略
  --products '<JSON数组>' \        # 商品明细，见下方格式说明
  -y \                             # 跳过交互确认（Agent 模式必加）
  -f json
```

**products JSON 字段说明**:

| 字段 | 类型 | 说明 |
|------|------|------|
| `productName` | string | 商品名称，必须为税务标准名称，不得用口语 |
| `amountIncludeTax` | string | 含税金额，纯数字字符串，如 `"100.00"` |
| `taxSign` | string | 含税标识：`"1"`=含税（固定填 `"1"`），`"0"`=不含税 |
| `quantity` | string | 数量，用户未说明时默认 `"1"` |
| `unit` | string | 单位，选填 |
| `specs` | string | 规格，选填 |

> ⚠️ `taxSign` 必须显式填写 `"1"`，缺失时系统可能按不含税处理导致金额错误。

**输出关键字段路径**:
```
orderId    → 传给 dws finance invoice issue-result --order-id，用于轮询结果
```

---

## dws finance invoice issue-result

**用途**: 查询某笔开票请求的最终结果，需轮询至出现明确成功或失败。
**使用场景**: `dws finance invoice issue` 提交后立即调用，或用户询问"发票开好了吗"时。
**轮询策略**: 结果未出 → 等待后重试；出现明确结论 → 停止轮询。

```bash
dws finance invoice issue-result \
  --order-id "<orderId>" \    # 开票请求的订单号，从 invoice issue 输出中提取，必填
  -f json
```

**输出关键字段路径**:
```
status           → 开票状态（处理中/成功/失败）
invoiceNo        → 发票号码（成功时）
invoiceTime      → 开票时间（成功时）
amountIncludeTax → 含税总额（成功时）
invoiceType      → 发票类型（成功时）
invoiceUrl       → 发票链接（成功时），dingtalk:// 链接逐字原样完整输出，禁止任何修改
failReason       → 失败原因（失败时，展示给用户）
```
