# AuthBuddy — Universal Authorization Vulnerability Audit

基于大模型 Agent Team 的通用越权漏洞审计系统。方法论驱动，不区分技术栈、不维护规则、不靠推断。

## 核心能力

- **语言/框架无关** — 语义模型统一，不依赖关键词、框架名、规则库
- **完整调用链** — 不设固定深度，语义驱动终止
- **双向分析** — 正向信任传播 + 反向出参回溯
- **9 Agent 流水线** — 每个阶段有验证，缺失回补
- **8 种越权场景** — S1 BOLA, S2 身份冒用, S3 BFLA, S4 未授权访问, S5 Mass Assignment, S6 审批绕过, S7 出参泄露, S8 锚点失效
- **10 条信任传播规则** — R1-R10 方法论

## 快速开始

```
/authbuddy /api/users/getInfo           # 审计指定接口
/authbuddy com.example.UserController   # 按类名审计
```

## 架构

```
主 Agent (SKILL.md) — 纯调度器
  ├── A00-Recon       → recon.json      (前置侦查)
  ├── A01-Entry       → entry.json      (入参提取)
  ├── A02-CallChain   → callchain.json  (调用链构建)
  ├── A02b-Verify     → 验证报告         (调用链验证)
  ├── A03-Forward     → forward.json    (正向信任传播)
  ├── A03b-Verify     → 验证报告         (正向验证)
  ├── A04-Backward    → backward.json   (反向出参回溯)
  ├── A04b-Verify     → 验证报告         (反向验证)
  └── A05-Report      → report.md/json  (最终报告)
```

## 文件结构

```
authbuddy/
├── SKILL.md                    # 主编排器
├── README.md
├── agents/                     # 9 个 Agent 定义文件
├── reference/                  # 6 个方法论参考文件
├── schemas/                    # 9 个 JSON Schema
└── docs/
    └── spec.md                 # 完整设计规格书
```

## 核心原则

1. **信任链分析** — 越权 = 用户可控参数到达数据操作而未与认证锚点建立信任关系
2. **每端点独立** — 每个接口独立分析，不假设数据因其他接口写入而安全
3. **证据完整性** — 每个判断必须附带 file_ref + rule_ref + code_snippet
4. **不截断** — 不设固定深度限制，由语义终止条件驱动
5. **方法论优先** — 关键词/框架名仅作弱提示，授权结论由语义分析决定

## 借鉴项目

- [authscan-2](https://github.com/...) — R1-R10 信任传播规则、Layer 分层
- [code-audit](https://github.com/...) — 双轨审计、门控条件、防幻觉规则
- [VulSolver](https://github.com/...) — Interest/Sink 定义、路径探索+验证

## 版本

- **Current:** 1.0
- **Updated:** 2026-05-06
