# 智能财务 — 开具发票 Skill（CLI 版）

## 功能

AI 辅助通过 `dws` CLI 向客户开具数电专票/普票，全程引导收集购方信息和商品明细，异步轮询开票结果。

## 支持场景

- 给客户开专票（数电增值税专用发票）
- 给客户开普票（数电增值税普通发票）
- 查询开票进度和结果

## 不支持场景

- 上传/识别进项发票 → 使用 finance-payment Skill
- 创建付款单/报销 → 使用 finance-payment Skill

## 前置条件

- 已安装并配置 `dws` CLI（`dws auth login` 完成认证）

## 关键设计决策

1. **异步轮询**: 开票为异步流程，提交后必须轮询 `dws finance invoice issue-result`
2. **含税金额**: 用户说的所有金额统一视为含税，products JSON 中 `taxSign` 固定填 `"1"`
3. **商品名称**: 口语名称须向用户确认标准名称后方可填入，不得直接使用
4. **行动优先**: 有客户信息时立即搜索，不阻塞等待其他字段；全流程最多一次用户追问
5. **dingtalk:// 链接**: 任何输出中的 `dingtalk://` 链接必须逐字原样展示，禁止任何修改

## 文件结构

```
skill-cli/
├── SKILL.md              # 主文件
├── package.json
├── README.md
├── references/
│   ├── api-reference.md  # 完整 CLI 命令参数 + 输出字段
│   └── error-codes.md    # 错误场景 + 轮询逻辑
└── tests/
    └── testcases.json    # 评测用例
```
