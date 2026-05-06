# AuthBuddy — 通用越权漏洞审计系统 设计规格书

**版本:** 1.0
**日期:** 2026-05-06
**状态:** 已确认

---

## 目录

1. [项目目标与定位](#1-项目目标与定位)
2. [核心设计原则](#2-核心设计原则)
3. [整体架构](#3-整体架构)
4. [主 Agent 编排器](#4-主-agent-编排器)
5. [A00-前置侦查](#5-a00-前置侦查)
6. [A01-入参提取](#6-a01-入参提取)
7. [A02-调用链构建](#7-a02-调用链构建)
8. [A02b-调用链验证](#8-a02b-调用链验证)
9. [A03-正向信任传播](#9-a03-正向信任传播)
10. [A03b-正向分析验证](#10-a03b-正向分析验证)
11. [A04-反向出参回溯](#11-a04-反向出参回溯)
12. [A04b-反向分析验证](#12-a04b-反向分析验证)
13. [A05-报告生成](#13-a05-报告生成)
14. [方法论参考文件](#14-方法论参考文件)
15. [JSON Schema 体系](#15-json-schema-体系)
16. [证据完整性原则](#16-证据完整性原则)
17. [项目文件结构](#17-项目文件结构)

---

## 1. 项目目标与定位

### 1.1 核心目标

| 目标 | 描述 |
|------|------|
| **目标 A** | 不区分技术栈、不维护规则、通用性强 |
| **目标 B** | 超高准确率、完整调用链、不靠推断 |

### 1.2 定位

- 以**单接口审计**为绝对中心
- **方法论优先**，不依赖技术栈/语言/规则维护
- 不能为了上下文省事而牺牲准确率
- 必须允许复杂调用链持续推进，直到可信终点
- 中间产物必须是「足够下一阶段直接使用的证据」，而不是模糊摘要
- 不靠固定深度截断
- 不靠模糊摘要脑补
- 多阶段产物必须足够完整，供下一阶段可信消费

### 1.3 正确认知

```
关键词 / 样式 / known pattern
→ 帮助快速定位候选点
候选点
→ 再通过语义分析确认
语义分析
→ 才能下结论
```

而不是反过来：看到某关键词就认为是某种鉴权模式再直接下判断。

### 1.4 不做的事情

- 不做全项目批量扫描（单接口是设计重心，批量是后续扩展）
- 不做可利用性验证（渗透测试范畴）
- 不生成 PoC（但提供完整证据链）
- 不自动修改代码
- 不把 SQL 当唯一授权面
- 不以关键词搜索为主
- 不把框架名当核心

---

## 2. 核心设计原则

### 2.1 方法论优先原则

所有判断基于语义分析，不基于关键词匹配或框架名。关键词/框架信息仅作为可选弱提示，不作为主体结构。

### 2.2 证据完整性原则

每个 agent 输出的字段值和原因字段必须满足两个「不能」：

1. **不能留空模糊词** — 禁止 `"可能"、"似乎"、"大概"、"待确认"`。不确定就标 `"unresolved"` 并写明缺少什么证据
2. **不能缺关联证据** — 每个判断字段必须附带三要素：
   - 代码位置: `file` + `line` 引用
   - 语义依据: 引用的规则编号或方法论条款
   - 实际代码片段: `code_snippet` — 从文件中读到的关键行（1-3 行足够）

### 2.3 产物文件强制原则

每个子 agent 必须将结果写入指定文件。不允许以对话文本代替文件输出。主 agent 通过 `Read` 工具校验文件存在且非空后才认为任务完成。

### 2.4 不截断原则

- 不设固定深度限制
- 调用链追到可信终点（项目外代码、纯静态无副作用方法、已访问节点（环检测）、无逻辑 getter/setter、无子调用的 leaf）才停
- 中间产物不压缩、不摘要、不脑补

### 2.5 越权本质定义

> **越权漏洞的本质是：用户的入参没有和用户不可控参数（可信锚点）产生直接或间接关联关系。用户的出参没有和可信锚点产生直接或间接关联关系。**

### 2.6 关键场景认知

- **没有必须先检测权限后执行的说法** — 只要最终返回给用户时校验了即可
- **super/parent 调用必须展开** — 拦截器中 `super.formatParam()` 场景，身份参数覆盖可能发生在父类方法中
- **审批流是鉴权** — 用户A操作用户B数据，发起审批流视为延迟授权
- **冗余纵深防御不误判** — 拦截器1完备时，不能因拦截器2不完备而判漏洞
- **越权不只发生在 SQL** — 文件读写、SSRF、网络访问、RPC 等场景同样存在

---

## 3. 整体架构

### 3.1 架构概览

```
┌─────────────────────────────────────────────────────────┐
│                    主 Agent (SKILL.md)                    │
│  纯调度器：任务分发 → 收集产物 → 交叉校验 → 分发回补      │
│  → 最终裁决 → 生成报告                                    │
└──┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬─────┘
   │      │      │      │      │      │      │      │
   ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
 [A00]  [A01]  [A02]  [A02b] [A03]  [A03b] [A04] [A04b] [A05]
 Recon  Entry  Call   Call   Fwd    Fwd    Bwd    Bwd   Report
        Analysis Chain Verify Trust  Verify Output Verify
```

### 3.2 流水线执行顺序

| 步骤 | Agent | 输入 | 输出 | 验证 |
|------|-------|------|------|------|
| 0 | A00-Recon | 接口路径/类名 | `recon.json` | 无（主 agent 复核 Gate 1） |
| 1 | A01-Entry | `recon.json` | `entry.json` | 无（结构层，主 agent 复核） |
| 2 | A02-CallChain | `recon.json` + `entry.json` | `callchain.json` | A02b → 回补 |
| 3 | A03-Forward | `recon.json` + `entry.json` + `callchain.json` | `forward.json` | A03b → 回补 |
| 4 | A04-Backward | `recon.json` + `callchain.json` + `forward.json` | `backward.json` | A04b → 回补 |
| 5 | A05-Report | 以上全部产物 | `report.md` + `report.json` | 无（终态） |

### 3.3 回补机制

1. 验证 agent 发现缺失 → 输出 `*-verify.json`，其中 `missing_items` 列出缺失项
2. 主 agent 检查缺失项，决定是否需要启动新的子 agent 回补
3. 回补结果合并进原 JSON，再交给后续阶段

### 3.4 主 agent 角色

**纯调度型**。主 agent 只分配任务、收集结果、做最终裁决，不亲自阅读代码。所有代码分析全由子 agent 完成。

### 3.5 交付形态

Claude Code Skill 集合。由主 SKILL.md + 9 个 agent 定义文件 + 6 个参考文件 + 9 个 JSON Schema 组成。

---

## 4. 主 Agent 编排器

### 4.1 执行状态机

```
START
  │
  ▼
[A00: Recon] ──Gate1── [A01: Entry] ──► [A02: CallChain]
                                            │
                                            ▼
                                       [A02b: Verify]
                                            │
                                         Gate2
                                      ┌──pass──┐
                                      │        │ fail
                                      ▼        ▼
                                 [A03: Fwd]  回补A02
                                      │
                                      ▼
                                 [A03b: Verify]
                                      │
                                    Gate3
                                 ┌──pass──┐
                                 │        │ fail
                                 ▼        ▼
                            [A04: Bwd]  回补A03
                                 │
                                 ▼
                            [A04b: Verify]
                                 │
                               Gate4
                            ┌──pass──┐
                            │        │ fail
                            ▼        ▼
                       [A05: Report] 回补A04
                            │
                            ▼
                          DONE
```

### 4.2 门控条件

#### Gate 1 (recon 完整性)

- `entry_file` + `entry_function` 非空且文件存在
- `auth_anchors` 至少有一个候选
- `interceptor_chain` 已列出（空数组也是合法状态，表示无拦截器）
- `framework.language` 已识别

#### Gate 2 (callchain 完整性)

- `callchain-verify.json` 的 `overall_pass` 为 true，或 `missing_nodes` 中无 `severity: high`
- 调用链至少从入口函数连到 1 个 datasink 或 1 个 leaf 节点
- `edges` 非空

#### Gate 3 (forward 完整性)

- `forward-verify.json` 的 `overall_pass` 为 true，或无 high severity
- `trusted_pool` 覆盖了所有 recon 中的 `auth_anchors`
- 所有 `resource_identifier` + `identity` 角色的参数都有 `final_trust_status`

#### Gate 4 (backward 完整性)

- `backward-verify.json` 的 `overall_pass` 为 true，或无 high severity
- 所有 `at_risk` 判定的输出字段都有 `risk_reason`

### 4.3 回补决策逻辑

| 验证产物 severity | 决策 |
|-------------------|------|
| `high` | **必须**回补，启动新子 agent，传入缺失项清单 |
| `medium` + 涉及 `resource_identifier`/`identity` 参数 | 回补 |
| `medium` + 其他角色参数 | 视情况，可推迟到报告阶段记录为 limitation |
| `low` | 不回补，记入最终报告的 `limitations` 字段 |

### 4.4 任务分发格式

每个子 agent 启动时必须明确给出：

```markdown
[AGENT_TASK]
Agent: {Agent ID}
Definition: agents/{agent-file}.md
Inputs: {列出所有输入文件路径}
Output: {必须产出的文件路径}
Schema: schemas/{schema-file}.json
Special Instructions: {验证抽查策略 | 回补缺失项清单 | 其他}
```

### 4.5 任务完成检查

```markdown
[TASK_CHECK: {Agent ID}]
1. {output-file} 存在? → YES/NO
2. {output-file} 非空? → YES/NO
3. 通过 → 进入下一步
   未通过 → 重试，附上失败原因
```

---

## 5. A00-前置侦查

### 5.1 职责

输入接口标识，输出该接口的完整上下文。不做任何安全判断。只做代码定位、信息提取、结构识别。

### 5.2 输入

- 接口路径（如 `/api/orders/{id}`）或 类名.方法名
- 项目根目录

### 5.3 输出 Schema

```json
{
  "endpoint": {
    "identifier": "string (接口标识)",
    "http_method": "string (GET/POST/PUT/DELETE/PATCH)",
    "entry_file": "string (入口文件完整路径)",
    "entry_function": "string (入口函数完整签名)",
    "entry_line": "number (入口函数起始行号)",
    "entry_line_end": "number (入口函数结束行号)",
    "code_snippet": "string (入口函数签名+注解关键行，1-5行)"
  },
  "framework": {
    "name": "string (框架名，如 Spring Boot/Django/Express)",
    "version_hint": "string (版本提示)",
    "language": "string (编程语言)"
  },
  "auth_context": {
    "auth_anchors": [
      {
        "name": "string (锚点变量名，如 currentUserId)",
        "source_type": "string (session/token/internal_sdk/config/rpc_context/jwt_claim)",
        "acquisition_chain": ["string (获取锚点的调用链，每步一个表达式)"],
        "acquisition_file": "string (锚点获取代码所在文件)",
        "acquisition_line": "number",
        "code_snippet": "string (锚点获取的关键代码，1-3行)",
        "credibility_checklist": {
          "has_noLogin_fallback": "boolean",
          "has_user_input_fallback": "boolean",
          "has_gray_toggle": "boolean",
          "has_conditional_bypass": "boolean",
          "credibility_notes": "string (如有风险，描述具体风险)"
        }
      }
    ],
    "global_filters": [
      {
        "class": "string (拦截器/中间件/过滤器类名)",
        "file": "string (文件路径)",
        "line": "number",
        "method": "string (方法名)",
        "role": "string (session_enforcement/param_override/token_verify/rbac_check/audit_log)",
        "code_snippet": "string (关键行，1-3行)"
      }
    ],
    "annotations_on_endpoint": ["string (方法上的注解/装饰器)"],
    "annotations_on_class": ["string (类上的注解/装饰器)"],
    "auth_framework_notes": "string (补充说明认证框架的特殊机制)"
  },
  "interceptor_chain": [
    {
      "order": "number (执行顺序，1开始)",
      "class": "string",
      "file": "string",
      "line": "number",
      "method": "string",
      "has_super_call": "boolean",
      "super_class": "string|null (如果 has_super_call，父类名)",
      "super_file": "string|null (父类文件路径)",
      "super_method": "string|null (父类方法名)",
      "super_line": "number|null (父类方法行号)",
      "role_hint": "string (此拦截器的角色描述)",
      "code_snippet": "string (拦截器方法签名和 super 调用关键行)"
    }
  ],
  "route_config": {
    "source": "string (路由配置来源: 注解/配置文件/路由文件)",
    "file": "string (路由配置文件路径)",
    "line": "number",
    "pattern": "string (路由匹配模式)"
  }
}
```

### 5.4 工作流程

1. 使用 Glob/Grep 定位入口函数所在的文件
2. 读取入口函数代码，提取注解/装饰器、函数签名
3. 识别框架类型（从依赖文件、注解风格、目录结构等判断）
4. 搜索拦截器/中间件配置（搜索框架特定的配置方式）
5. 分析每个拦截器/中间件的代码，特别关注：
   - 是否有 `super.xxx()` 调用（父类方法调用）
   - 如果有，记录父类位置
6. 识别 auth anchor 的获取代码：
   - 搜索 session/token/JWT/内部 SDK/DRM/mist 的获取调用
   - 追踪获取链（例如 SecurityContext → Authentication → Principal → getUserId）
   - 评估 anchor 可信度（R10 检查清单）
7. 输出 `recon.json`

### 5.5 特殊关注点

- **super 调用**: 拦截器/中间件中的 `super.xxx()`，必须定位到父类方法并记录
- **接口认证注解**: 如 `@PreAuthorize`, `@PermitAll`, `@Anonymous`, `@noLogin` 等
- **框架特定配置**: Spring Security Config、Django settings MIDDLEWARE、Express app.use 等
- **多级继承**: 如果父类还有父类调用，需要追到底

### 5.6 输出要求

- 必须写入文件: `recon.json`
- 格式: JSON
- Schema: `schemas/recon.schema.json`
- 所有字段必须附带 `code_snippet` 证据
- 不允许以对话文本代替文件输出

---

## 6. A01-入参提取

### 6.1 职责

从入口函数提取所有入参，展开到基础类型（String、int、long、boolean 等），做语义角色分类。不做安全判断。

### 6.2 输入

- `recon.json`
- 入口函数源码

### 6.3 输出 Schema

```json
{
  "entry_function": "string (完整函数签名)",
  "parameters": [
    {
      "name": "string (参数名，嵌套参数用点号连接，如 request.userId)",
      "primitive_type": "string (基础类型: String/int/long/boolean/double/float/List/Map/File/InputStream)",
      "source": "string (path_variable/query_param/header/cookie/body_field/multipart_form/session_derived/token_derived/internal_context)",
      "source_annotation": "string (参数来源注解，如 @PathVariable/@RequestParam/@RequestBody/request.args.get)",
      "semantic_role": "string (见 6.4 角色分类)",
      "pii": "boolean (是否包含个人身份信息)",
      "nullable": "boolean",
      "default_value": "string|null",
      "entry_file": "string (参数声明所在的文件)",
      "entry_line": "number (参数声明行号)",
      "code_snippet": "string (参数声明的代码片段)"
    }
  ],
  "unexpanded_parameters": [
    {
      "name": "string (参数名)",
      "primitive_type": "string (复杂类型)",
      "reason_not_expanded": "string (原因: 嵌套层级过深/类型定义在外部依赖/循环引用)",
      "note": "string (建议后续 agent 在哪里获取展开信息)"
    }
  ],
  "entry_signature_code_snippet": "string (入口函数完整签名的代码片段)"
}
```

### 6.4 入参语义角色分类

#### 资源定位类（决定访问 WHAT）

| 角色 | 含义 | 风险权重 | 示例 |
|------|------|----------|------|
| `resource_identifier` | 直接定位被操作资源的 ID | **极高** | orderId, fileId, documentId |
| `relationship_context` | 限定资源归属范围的上下文 ID | **高** | tenantId, projectId, groupId, parentId, deptId |
| `scope_range` | 范围边界值 | 中 | dateFrom, dateTo, amountMin, amountMax |

#### 主体类（决定 WHO）

| 角色 | 含义 | 风险权重 | 示例 |
|------|------|----------|------|
| `identity` | 声称的操作者身份标识 | **极高** | userId, operatorId, assigneeId |
| `auth_credential` | 认证凭证本身 | **极高** | token, apiKey, signature, password |

#### 操作类（决定做什么）

| 角色 | 含义 | 风险权重 | 示例 |
|------|------|----------|------|
| `action` | 操作指令 | 中 | action=delete, operation=approve, cmd=submit |
| `data_payload` | 要写入/更新的业务数据 | **高** | body 中的 name, price, status 等业务字段 |
| `file_content` | 文件内容/文件引用 | **高** | multipart file, base64Content, fileUrl |

#### 查询修饰类（决定结果如何呈现）

| 角色 | 含义 | 风险权重 | 示例 |
|------|------|----------|------|
| `filter` | 过滤/筛选条件 | 中 | status, keyword, category, tag |
| `pagination` | 分页 | 低 | page, offset, limit, size, cursor |
| `sorting` | 排序 | 低 | sortBy, sortOrder, orderBy, direction |

#### 基础设施类

| 角色 | 含义 | 风险权重 | 示例 |
|------|------|----------|------|
| `callback_url` | 回调/跳转/重定向 URL | **高** | redirectUrl, callbackUrl, webhook |
| `metadata` | 不参与业务逻辑的元数据 | 无 | traceId, locale, timezone, requestId |
| `configuration` | 行为配置/开关 | 低 | timeout, dryRun, debug, async |

### 6.5 展开原则

- 展开到基础类型（String/int/long/boolean/double/float/List/Map/File/InputStream）为止
- 复杂对象（如自定义 DTO）展开到字段级别
- 标注展开来源（参数名的点号路径）
- 嵌套集合展开元素类型
- 无法展开的复杂类型记录到 `unexpanded_parameters`

### 6.6 输出要求

- 必须写入文件: `entry.json`
- 格式: JSON
- Schema: `schemas/entry.schema.json`
- 每个参数必须附带 `code_snippet` 证据
- 不允许以对话文本代替文件输出

---

## 7. A02-调用链构建

### 7.1 职责

从入口函数开始，DFS 构建完整调用图。只做拓扑结构，不做参数语义。不追踪参数信任状态、不判断安全性。

### 7.2 输入

- `recon.json`
- `entry.json`
- 项目源码

### 7.3 输出 Schema

```json
{
  "entry_node_id": "string (入口节点 ID)",
  "nodes": {
    "N001": {
      "function": "string (完整函数签名，含类名.方法名)",
      "file": "string (文件绝对路径)",
      "line_start": "number",
      "line_end": "number",
      "signature": "string (返回类型 方法名(参数类型 参数名, ...))",
      "annotations": ["string (函数上的注解/装饰器)"],
      "call_type_markers": ["string (super_call/interface_dispatch/lambda/reflection)"],
      "override_of": "string|null (如果是重写方法，指向父类/接口方法签名)",
      "parent_class": "string (所属类名，含包名)",
      "is_project_code": "boolean (true=项目代码，false=外部依赖/标准库)",
      "termination": "string|null (external/pure_static/recursive/accessor/leaf/null)",
      "code_snippet": "string (函数签名关键行，1-3行)"
    }
  },
  "edges": [
    {
      "caller_node_id": "string",
      "call_site_line": "number (调用发生的行号)",
      "caller_branch_context": "string (if/else/try/catch/finally/loop/switch/unconditional)",
      "callee_node_id": "string",
      "callee_expression": "string (完整的调用表达式)",
      "argument_mappings": [
        {
          "to_param": "string (被调用函数的参数名)",
          "from_expression": "string (调用方传入的表达式)"
        }
      ],
      "code_snippet": "string (调用行的代码片段)"
    }
  ],
  "unresolved_calls": [
    {
      "caller_node_id": "string",
      "call_site_line": "number",
      "expression": "string (未解析的调用表达式)",
      "reason": "string (interface_dispatch_unresolvable/reflection_runtime_only/lambda_unresolvable/dynamic_proxy/deferred_binding)",
      "candidate_implementations": ["string (可能的实现)"],
      "why_unresolvable": "string (为什么无法在静态分析中确定目标)",
      "code_snippet": "string"
    }
  ]
}
```

### 7.4 终止条件

| 条件 | 标记 | 说明 |
|------|------|------|
| 项目外部代码 | `external` | 标准库、第三方依赖 |
| 纯静态方法无副作用 | `pure_static` | 如 `StringUtils.isEmpty()`、`Math.max()` |
| 已访问节点（环） | `recursive` | 检测到循环调用 |
| 无逻辑 getter/setter | `accessor` | 仅返回/设置字段，无其他逻辑 |
| 无子调用的叶节点 | `leaf` | 函数体内无任何调用 |

**重要：** `accessor` 标记要谨慎使用。即使看似简单的 getter，如果返回值包含身份字段（如 `order.getOwnerId()`），不应轻易截断。仅在确认返回的是纯数据容器字段时才标记为 accessor。

### 7.5 `call_type_markers` 说明

这些标记用于提醒下游 agent 此处值得额外关注：

| 标记 | 含义 | 下游影响 |
|------|------|----------|
| `super_call` | 调用了 super/parent 方法 | 正向分析需追到父类方法，可能有身份参数覆盖 |
| `interface_dispatch` | 通过接口/抽象类调用 | 可能存在多个实现，需确认具体实现 |
| `lambda` | 包含 lambda 表达式 | 闭包可能捕获外层变量，参数池不限于参数列表 |
| `reflection` | 疑似反射调用 | 路径不稳定，需保守评估 |

空数组表示普通函数调用，无特殊关注点。

### 7.6 工作流程

1. 从 `entry_function` 节点开始
2. 读取函数源码
3. 识别所有子调用（函数调用、方法调用、super 调用、lambda）
4. 对每个子调用：
   a. 判断是否项目内代码
   b. 如果是接口/抽象调用，尝试定位具体实现
   c. 如果是 super 调用，定位父类方法
   d. 如果无法解析，记录到 `unresolved_calls`
5. 对每个项目内子调用，递归执行步骤 2-4
6. 检查终止条件
7. 构建边关系，记录实参映射和分支上下文

### 7.7 输出要求

- 必须写入文件: `callchain.json`
- 格式: JSON
- Schema: `schemas/callchain.schema.json`
- 每个节点和边必须附带 `code_snippet` 证据
- 不允许以对话文本代替文件输出
- **不设固定深度限制**，由终止条件语义驱动

---

## 8. A02b-调用链验证

### 8.1 职责

审计 `callchain.json`，标注缺失/错误。不补做，只出验证报告。

### 8.2 输入

- `callchain.json`
- `recon.json`
- 项目源码

### 8.3 输出 Schema

```json
{
  "overall_pass": "boolean",
  "missing_nodes": [
    {
      "caller_node_id": "string",
      "call_site_line": "number",
      "expression": "string (遗漏的调用表达式)",
      "target_function": "string (应添加的函数节点)",
      "target_file": "string (目标函数所在文件)",
      "target_line": "number (目标函数起始行号)",
      "severity": "string (high/medium/low)",
      "reason": "string (为什么遗漏会影响分析结论)",
      "code_snippet": "string (遗漏调用的代码片段)"
    }
  ],
  "incorrect_edges": [
    {
      "edge": "string (caller→callee 标识)",
      "issue": "string (具体问题)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "premature_terminations": [
    {
      "node_id": "string",
      "termination": "string (当前截断原因)",
      "issue": "string (为什么不应截断)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "review_notes": ["string (其他审核意见)"]
}
```

### 8.4 检查清单

1. **节点覆盖检查** — 按入口层、中间层、叶子层各抽查 2-3 个节点，验证函数签名、行号正确
2. **边完整性检查** — 对每个节点的调用，确认没有遗漏
3. **终止合理性检查** — 是否误判为 accessor（getter 可能包含身份字段如 `getOwnerId()`）
4. **super 展开检查** — `super_call` 标记的节点，确认父类方法已纳入
5. **接口实现完整性** — `interface_dispatch` 标记的节点，确认有具体实现
6. **缺失检测** — 源码中存在但 JSON 中遗漏的调用

### 8.5 severity 定义

| 级别 | 含义 | 处理 |
|------|------|------|
| `high` | 缺失可能改变授权结论 | 必须回补 |
| `medium` | 缺失影响分析完整性（涉及 identity/resource 参数） | 建议回补 |
| `low` | 信息性偏差，不影响结论 | 记录到 limitations |

### 8.6 输出要求

- 必须写入文件: `callchain-verify.json`
- 格式: JSON
- Schema: `schemas/callchain-verify.schema.json`
- 所有问题必须附带 `code_snippet` 证据
- 不允许以对话文本代替文件输出

---

## 9. A03-正向信任传播

### 9.1 职责

采用**C 方案（先建可信地图再评估参数）**：第一阶段追踪所有 auth anchor 的传播链，构建「可信数据池」；第二阶段追踪每个入参是否经过可信数据池到达 datasink。

不依赖关键词、不依赖框架名、不依赖规则库。

### 9.2 输入

- `recon.json`
- `entry.json`
- `callchain.json`
- `reference/trust-propagation-rules.md`
- `reference/datasink-patterns.md`

### 9.3 输出 Schema

```json
{
  "trusted_pool": [
    {
      "pool_id": "string",
      "origin_anchor": "string (来自 recon.auth_anchors 的 name)",
      "propagation_chain": [
        {
          "step": "number",
          "node_id": "string (在调用链中的节点 ID)",
          "line": "number",
          "anchor_usage": "string (assignment/query_constraint/guard_condition/post_filter/approval_initiation/param_override/rpc_identity_propagation)",
          "expression": "string (anchor 在此步骤的使用表达式)",
          "trust_effect": "string (信任效果描述，引用 R1-R10 规则编号)",
          "pool_members_added": ["string (此次传播新增的可信池成员，用 类型/变量名.字段名 格式)"],
          "trust_rule_applied": "string (应用的信任传播规则: R1-R10)",
          "code_snippet": "string (关键代码，1-3行)",
          "file": "string (所在文件)"
        }
      ]
    }
  ],
  "parameter_analysis": [
    {
      "param": "string (参数名，与 entry.json 中的 name 对应)",
      "semantic_role": "string (来自 entry.json)",
      "propagation_path": [
        {
          "step": "number",
          "node_id": "string",
          "line": "number",
          "usage": "string (pass_to_sub_call/assignment/query_constraint/condition_check/return_output/transform/closure_capture)",
          "expression": "string (参数在此步骤的使用表达式)",
          "at_this_point": "string (trusted/untrusted/unresolved/output_only_safe)",
          "trust_rule": "string|null (如果状态发生了变化，引用的规则编号)",
          "code_snippet": "string (关键代码，1-3行)",
          "file": "string"
        }
      ],
      "datasinks_reached": [
        {
          "node_id": "string",
          "line": "number",
          "type": "string (db_read/db_write/file_read/file_write/rpc_call/network_connect/approval_flow/mass_assignment/auth_decision)",
          "expression": "string (datasink 表达式)",
          "auth_bound": "boolean",
          "bound_anchor": "string|null (绑定的 anchor 名)",
          "bound_trust_rule": "string|null (绑定的信任规则)",
          "code_snippet": "string",
          "file": "string"
        }
      ],
      "final_trust_status": "string (trusted/at_risk/output_only_safe/unresolved)",
      "trust_chain": "string (完整的信任链路描述，从 anchor 到 final_trust_status)",
      "unresolved_reason": "string|null (如果 final_trust_status 是 unresolved，写清楚缺少什么信息)"
    }
  ],
  "at_risk_parameters": ["string (存在越权风险的参数名列表)"],
  "trusted_parameters": ["string (可信的参数名列表)"],
  "output_only_safe_parameters": ["string (仅输出回显的参数名列表)"]
}
```

### 9.4 Datasink 类型定义（不限于 SQL）

| Datasink 类型 | 检测模式 | 授权面 |
|--------------|----------|--------|
| `db_read` | ORM查询/原生SQL/SQL Builder SELECT | 读取他人数据 |
| `db_write` | INSERT/UPDATE/DELETE/MERGE | 修改/删除他人数据 |
| `file_read` | File.read/Files.newInputStream/fopen/readFileSync | 读取他人文件 |
| `file_write` | File.write/FileOutputStream/fwrite/writeFileSync | 覆盖/写入他人文件 |
| `rpc_call` | HTTP client/RPC/gRPC/Dubbo/Feign | 以他人身份请求外部服务 |
| `approval_flow` | 审批发起/工单创建/工作流启动 | 跳过授权门槛（延迟授权） |
| `network_connect` | URL.openConnection/socket/requests.get | SSRF/网络访问 |
| `mass_assignment` | setProperties/bind/save(entity)/bulkUpdate | 越权修改敏感字段 |
| `auth_decision` | hasPermission/canAccess/isOwner/checkRole | 授权判定本身的数据源 |

### 9.5 第一阶段：构建可信数据池

从 `recon.json` 中的每个 auth_anchor 出发：

1. 在调用链中找到 anchor 的获取点（assignment 节点）
2. 追踪 anchor 值在调用链中的所有出现位置
3. 在每个位置判断 anchor 的使用类型：
   - **query_constraint**: anchor 出现在 WHERE 子句等约束中 → 应用 R1+R2
   - **guard_condition**: anchor 出现在 if 条件中进行归属判断 → 应用 R4
   - **post_filter**: anchor 用于后置过滤结果集 → 应用 R5
   - **param_override**: anchor 用于覆盖入参 → 应用 R1（被覆盖的参数获得信任）
   - **approval_initiation**: anchor 用于创建审批流 → 延迟授权
   - 等等
4. 记录每次传播新增的可信池成员（具体到变量名.字段名）

### 9.6 第二阶段：追踪参数

对 `entry.json` 中的每个参数（按风险权重从高到低）：

1. 在调用链中找到参数的首次使用位置
2. 追踪参数在调用链中的传播（函数参数传递、赋值、变换）
3. 在每个使用点判断参数是否与可信池成员建立了关联：
   - 参数与 anchor 出现在同一 WHERE 约束中 → R1 trusted
   - 参数被 anchor 覆盖（param_override） → R1 trusted
   - 参数被 guard 校验且通过 → R4 trusted
   - 参数变换（toString, split 等） → R7 trust 不变
   - 参数到达 datasink 且未绑 anchor → R8 at_risk
4. 记录参数到达的每个 datasink
5. 综合判断最终信任状态

### 9.7 多参数到达同一 datasink

如果多个参数共同到达同一个 datasink（如 `WHERE a1=? AND a2=? AND a3=?`），每个参数独立判断信任状态。其中一个参数是 trusted 不自动让其他参数也 trusted — 除非它们在同一个约束中绑定了同一个 anchor。

### 9.8 信任传播规则（R1-R10 概览）

详见 `reference/trust-propagation-rules.md`。

| 规则 | 名称 | 核心判断 |
|------|------|----------|
| R1 | 直接关联 | 参数与 anchor 在同一约束中 |
| R2 | 派生信任 | anchor 约束的查询结果字段全部可信 |
| R3 | 传递信任 | 使用已信任数据作为约束时，输出继承可信 |
| R4 | 条件守卫 | guard(参数 vs anchor) 通过后，参数可信 |
| R5 | 后置认证读取 | 读取后可关联 anchor，信任可追溯 |
| R6 | 后置认证写入 | 写入后才关联 anchor，风险不可逆 |
| R7 | 变换中性 | toString/split/parseInt 不改信任状态 |
| R8 | 无关联 | 参数到达 datasink 无任何 anchor 绑定 |
| R9 | 存储身份重验证 | DB 中的身份字段需显式比较当前用户 |
| R10 | 锚点可信度 | anchor 本身是否可被用户控制 |

### 9.9 输出要求

- 必须写入文件: `forward.json`
- 格式: JSON
- Schema: `schemas/forward.schema.json`
- 每个判断必须附带证据三要素：`file` + `rule_ref` + `code_snippet`
- 不允许以对话文本代替文件输出

---

## 10. A03b-正向分析验证

### 10.1 职责

审计 `forward.json` 的信任传播判断。重点查「看起来安全其实不安全」的漏判。

### 10.2 输入

- `forward.json`
- `callchain.json`
- `recon.json`
- 项目源码

### 10.3 输出 Schema

```json
{
  "overall_pass": "boolean",
  "missed_anchors": [
    {
      "anchor_name": "string",
      "issue": "string (该 anchor 在 forward 中无任何传播追踪)",
      "severity": "string"
    }
  ],
  "false_trust_assignments": [
    {
      "param": "string",
      "current_judgment": "string (当前判定)",
      "reason_current": "string (当前依据)",
      "issue": "string (为什么判定有误)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "missed_datasinks": [
    {
      "node_id": "string",
      "expression": "string",
      "type": "string",
      "note": "string (为什么 A03 遗漏了这个 datasink)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "super_chain_gaps": [
    {
      "node_id": "string",
      "issue": "string (super 调用链路中的 anchor 传播缺失)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "transitive_trust_misuse": [
    {
      "node_id": "string",
      "current_judgment": "R3 传递信任",
      "issue": "string (为什么此处的 R3 应用不当，如将展示数据当权限条件)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "redundant_defense_false_positives": [
    {
      "param": "string",
      "issue": "string (把冗余纵深防御误判为漏洞)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "approval_flow_missed": [
    {
      "node_id": "string",
      "issue": "string (审批流未被识别为授权机制)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "review_notes": ["string"]
}
```

### 10.4 检查清单

1. **锚点遗漏** — 所有 recon 中的 auth_anchor 是否都在 trusted_pool 中有传播链
2. **假可信判断** — 抽查 trusted 参数：
   - 「来自数据库」不等于安全（除非查询绑了 anchor）
   - 「来自 SDK/RPC」不等于安全（除非该 SDK 有 auth 约束）
   - 「是查询结果字段」不等于安全（除非父查询有 anchor 约束）
3. **遗漏 datasink** — 文件操作、网络调用、审批流是否被识别为 datasink
4. **super/父类遗漏** — A03 是否正确追踪了 super 链路的 anchor 传播
5. **间接可信误判** — R3 传递信任是否被滥用。只有后续查询使用该字段作为权限条件时才继承可信，仅作为展示数据不继承
6. **冗余防御误判** — 是否把「拦截器1已覆盖，拦截器2不完整」误判为漏洞
7. **审批流遗漏** — 审批流创建是否被识别为延迟授权
8. **身份参数覆盖** — 是否确认了拦截器中 identity 参数被 auth anchor 覆盖

### 10.5 输出要求

- 必须写入文件: `forward-verify.json`
- 格式: JSON
- Schema: `schemas/forward-verify.schema.json`
- 所有问题必须附带 `code_snippet` 证据
- 不允许以对话文本代替文件输出

---

## 11. A04-反向出参回溯

### 11.1 职责

从最终返回值的每个字段出发，回溯其数据来源，判断是否经过可信数据池或 auth anchor 绑定。不只看「字段来自哪个查询」，还要看「形成这个字段的整条链路上有无 auth 约束」。

### 11.2 输入

- `recon.json`
- `callchain.json`
- `forward.json`
- `reference/output-semantics.md`

### 11.3 输出 Schema

```json
{
  "output_fields": [
    {
      "field_path": "string (输出字段路径，用点号连接嵌套，如 result.data.order.orderId)",
      "primitive_type": "string (字段的基础类型)",
      "source_node_id": "string (来源在调用链中的节点 ID)",
      "source_expression": "string (来源表达式)",
      "source_kind": "string (db_query_result_field/db_query_result/rpc_response_field/computed/static_constant/literal/user_input_echo/stored_context_field)",
      "source_query_node_id": "string|null (如果来自查询，查询所在的节点 ID)",
      "source_query_bound_to_anchor": "boolean|null",
      "source_query_bound_anchors": ["string (具体哪个 anchor)"],
      "computation_path": [
        {
          "step": "number",
          "node_id": "string",
          "line": "number",
          "expression": "string",
          "depends_on": ["string (依赖的变量/字段名)"],
          "code_snippet": "string",
          "file": "string"
        }
      ],
      "deep_analysis": {
        "applicable": "boolean (是否需要深度分析，布尔/枚举/computed 类型为 true)",
        "judgment_basis": "string (对于布尔/枚举返回值，描述该值是如何计算出来的)",
        "auth_bound": "boolean (计算的链路上是否绑定了 auth anchor)",
        "trust_rule": "string|null"
      },
      "final_judgment": "string (trusted/at_risk/output_only_safe/static_safe/unresolved)",
      "trust_chain": "string (完整的信任链路描述)",
      "risk_reason": "string|null (如果 at_risk，详细描述风险原因和证据)",
      "file": "string (关键证据文件)",
      "code_snippet": "string (关键证据代码)"
    }
  ],
  "fields_at_risk": ["string (存在越权风险的输出字段路径列表)"],
  "fields_output_only_safe": ["string (仅回显/无风险字段列表)"],
  "fields_static_safe": ["string (静态安全字段列表)"],
  "fields_trusted": ["string (可信字段列表)"],
  "fields_unresolved": ["string (无法确定安全的字段列表)"]
}
```

### 11.4 反向追溯终止判断矩阵

```
回退到一个值 V，按以下条件依次判断：

1. V 直接来自 recon 中的 auth_anchor
   → ✅ TRUSTED

2. V 在正向分析的 trusted_pool 中
   → ✅ TRUSTED (rule_ref: forward.trusted_pool[pool_id])

3. V 来自一个「绑了 anchor 的查询/操作」的返回字段
   → ✅ TRUSTED (rule_ref: R1+R2)

4. V 来自一个「绑了 anchor 的 guard」之后允许通过的数据
   → ✅ TRUSTED (rule_ref: R4)

5. V 来自审批流创建/审批流状态查询（延迟授权已覆盖）
   → ✅ TRUSTED (rule_ref: approval_flow)

6. V 是入参的直接回显（未经过任何 datasink，或仅经过 R7 变换）
   → ⚠️ OUTPUT_ONLY_SAFE

7. V 来自静态常量（不可被用户写入的配置/枚举/字面量）
   → ⚠️ STATIC_SAFE

8. V 来自一个「未绑 anchor 的查询」结果
   → ❌ AT_RISK (rule_ref: R8)

9. V 来自一个「未绑 anchor 的判断逻辑」得出的 true/false
   → ❌ AT_RISK (rule_ref: R8, 需展开 deep_analysis)

10. V 来自用户可写入的缓存/DB/全局变量/可写配置
    → ❌ AT_RISK

11. V 来自外部 RPC/HTTP 调用且未做身份传递
    → ❌ AT_RISK
```

### 11.5 布尔/枚举返回值的深度分析

当输出字段类型为 boolean 或 enum 时，必须展开 `deep_analysis`：

1. 找到返回值的形成逻辑（if 条件、三元表达式、函数返回值）
2. 追踪形成逻辑中参与的变量和函数调用
3. 判断这些变量/函数是否与 auth anchor 有直接或间接关联
4. 如果有，标注 trusted；如果没有，标注 at_risk

例如：
- `result.hasPermission = checkAccess(userId, resourceId)` → 展开 `checkAccess` → 发现内部查询了权限表 WHERE userId = currentUserId → trusted
- `result.hasPermission = (order.status == "ACTIVE")` → status 来自未绑 anchor 的查询 → at_risk

### 11.6 审批流识别

- 搜索审批相关的关键字（approval、workflow、ticket、审批、工单）
- 检查审批流创建时的参数是否绑了 auth anchor
- 审批流状态查询/回调是否绑了 auth anchor
- 如果审批流已覆盖，即使直接操作的查询未绑 anchor，也视为延迟授权有效

### 11.7 输出要求

- 必须写入文件: `backward.json`
- 格式: JSON
- Schema: `schemas/backward.schema.json`
- 每个判断必须附带证据三要素：`file` + `rule_ref` + `code_snippet`
- 所有 `at_risk` 判定必须填写 `risk_reason`
- 不允许以对话文本代替文件输出

---

## 12. A04b-反向分析验证

### 12.1 职责

审计 `backward.json`，重点查输出字段来源是否能被伪造、误判为安全。

### 12.2 输入

- `backward.json`
- `forward.json`
- `callchain.json`
- `recon.json`
- 项目源码

### 12.3 输出 Schema

```json
{
  "overall_pass": "boolean",
  "output_field_coverage_issues": [
    {
      "issue": "string (返回值中有字段未被 A04 覆盖)",
      "field_path": "string",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "false_safe_judgments": [
    {
      "field_path": "string",
      "current_judgment": "string (当前判定)",
      "reason_current": "string (当前依据)",
      "issue": "string (为什么判定有误 — 如 static_safe 实际来自可写配置)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "boolean_deep_analysis_gaps": [
    {
      "field_path": "string",
      "issue": "string (布尔/枚举字段的 deep_analysis 不够深入)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "approval_flow_missed": [
    {
      "field_path": "string",
      "issue": "string (审批流相关输出未被识别)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "multi_source_field_issues": [
    {
      "field_path": "string",
      "issue": "string (输出字段来自多个分支/来源，A04 只覆盖了部分)",
      "severity": "string",
      "code_snippet": "string"
    }
  ],
  "review_notes": ["string"]
}
```

### 12.4 检查清单

1. **输出字段覆盖** — A04 是否覆盖了返回值的所有字段（包括嵌套对象内部字段、集合元素类型字段）
2. **假安全判断** — 字段标了 `static_safe`，但检查发现来自可写配置/缓存/用户可控 DB 字段
3. **布尔值深追** — `computed` 类型的布尔字段，`deep_analysis` 是否追到真正的判断条件和数据源
4. **审批流遗漏** — 输出字段涉及审批流状态，但 A04 未识别为授权机制
5. **多源字段** — 一个输出字段可能来自多个分支（如 `field = condition ? queryA : queryB`），A04 是否覆盖了所有分支
6. **信息泄露** — 输出字段是否包含了其他用户的身份信息/数据（通过追踪字段来源的上游判断）

### 12.5 输出要求

- 必须写入文件: `backward-verify.json`
- 格式: JSON
- Schema: `schemas/backward-verify.schema.json`
- 所有问题必须附带 `code_snippet` 证据
- 不允许以对话文本代替文件输出

---

## 13. A05-报告生成

### 13.1 职责

聚合全部产物（recon → entry → callchain → forward → backward → 全部 verify），按越权场景分类给出结论。生成人工阅读的 `report.md` 和程序化消费的 `report.json`。

### 13.2 输入

- 所有阶段的产物 JSON
- `reference/scenario-taxonomy.md`

### 13.3 越权场景分类（S1-S8）

| 场景 ID | 场景名 | 触发条件 |
|---------|--------|----------|
| S1 | BOLA (Broken Object Level Authorization) | `resource_identifier` / `relationship_context` 角色的参数，正向分析判定为 `at_risk`，且到达了 datasink |
| S2 | 身份冒用 (Identity Impersonation) | `identity` 角色的参数未被 auth anchor 覆盖，且被用于 datasink 约束 |
| S3 | BFLA (Broken Function Level Authorization) | 低权限用户可访问高权限功能（结合 endpoint 注解和锚点覆盖范围判断） |
| S4 | 未授权访问 (Unauthenticated Access) | 无需认证即可访问需登录接口（R10 锚点失效触发） |
| S5 | Mass Assignment | 敏感字段可被用户传入并写入 datasink，无过滤机制 |
| S6 | 延迟授权绕过 (Approval Bypass) | 审批流不完整或被绕过，直接操作缺少 auth 绑定 |
| S7 | 出参信息泄露 (Output Data Leak) | 返回值包含未授权数据，或布尔/枚举可推导授权状态（信息预言） |
| S8 | 信任锚点失效 (Anchor Compromise) | auth anchor 可被用户控制（R10 触发），所有基于此 anchor 的信任判断失效 |

### 13.4 report.md 结构

```markdown
# 越权审计报告 — {endpoint_identifier}

## 1. 审计概要
- **审计对象**: {endpoint}
- **框架/语言**: {framework/language}
- **可信锚点**: {auth_anchors 列表}
- **入参数量**: {N}
- **出参字段数量**: {M}
- **审计时间**: {timestamp}

## 2. 发现的越权场景

### {S1-BOLA}: {简述}
- **严重度**: {severity}
- **风险参数**: {参数列表}
- **完整信任链路**: 
  ```
  {文本树形图}
  ```
- **证据**:
  - 文件: {file}:{line}
  - 代码: 
    ```{language}
    {关键代码}
    ```
  - 规则依据: {rule_ref}
- **攻击场景**: {描述攻击者如何利用此漏洞}
- **修复建议**: {具体修复代码或方案}

### {S2-身份冒用}: {简述}
- ... (同上)

## 3. 参数风险矩阵
| 参数 | 角色 | 信任状态 | 到达的 Datasink | 规则依据 |
|------|------|----------|----------------|----------|
| orderId | resource_identifier | trusted | db_read (N002) | R1 |
| targetUserId | identity | at_risk | db_read (N008) | R8 |

## 4. 可信链路图
```
{完整的文本树形图，标注 anchor → trusted → at_risk 路径}
```

## 5. 修复建议汇总
1. {S1}: {修复方案}
2. {S2}: {修复方案}

## 6. 审计局限性
- {未覆盖的部分}
- {unresolved 的项}
- {severity: low 的验证发现}
```

### 13.5 report.json 结构

```json
{
  "endpoint": "string",
  "audit_timestamp": "string",
  "audit_summary": {
    "framework": {},
    "auth_anchors": [],
    "total_params": "number",
    "total_output_fields": "number",
    "scenarios_found_count": "number",
    "scenarios_checked": ["S1","S2","S3","S4","S5","S6","S7","S8"]
  },
  "scenarios_found": [
    {
      "scenario_id": "string (S1-S8)",
      "scenario_name": "string",
      "severity": "string (critical/high/medium/low/info)",
      "risk_parameters": ["string"],
      "risk_output_fields": ["string"],
      "evidence_chain": {
        "recon_ref": "string (引用 recon.json 中相关项)",
        "forward_ref": "string (引用 forward.json 中相关项)",
        "backward_ref": "string (引用 backward.json 中相关项)"
      },
      "trust_chain_visual": "string (文本树形图)",
      "attack_scenario": "string (攻击者利用路径描述)",
      "fix_suggestion": "string (修复方案)",
      "fix_code_snippet": "string (修复代码示例)"
    }
  ],
  "scenarios_not_found": ["string (检查但未触发的场景)"],
  "parameters_matrix": [
    {
      "param": "string",
      "semantic_role": "string",
      "final_trust_status": "string",
      "datasinks_reached": [{"type": "string", "auth_bound": "boolean"}],
      "trust_chain": "string"
    }
  ],
  "output_fields_matrix": [
    {
      "field_path": "string",
      "final_judgment": "string",
      "source_kind": "string",
      "trust_chain": "string"
    }
  ],
  "limitations": ["string (审计局限性说明)"],
  "cross_stage_validation_notes": ["string (交叉验证发现的问题)"]
}
```

### 13.6 主 agent 最终裁决逻辑

1. 收集所有阶段产物（包括验证报告）
2. 逐场景检查 S1-S8 的触发条件
3. 对每个触发的场景：
   a. 从 forward.json 提取风险参数和链路
   b. 从 backward.json 提取风险输出字段和链路
   c. 交叉验证：正向和反向的结论是否一致
   d. 组装证据链
4. 生成 report.md 和 report.json
5. 不触发任何场景 → 报告为「未发现越权漏洞」
6. 存在 unresolved → 在 limitations 中说明

### 13.7 输出要求

- 必须写入文件: `report.md` + `report.json`
- 格式: Markdown + JSON
- Schema: `schemas/report.schema.json`
- 报告中所有判断必须引用来源产物的具体位置
- 不允许以对话文本代替文件输出

---

## 14. 方法论参考文件

### 14.1 trust-propagation-rules.md

详细的 R1-R10 信任传播规则，包含每条规则的触发条件、信任结果、代码示例（多语言）、边界条件。这是所有判断的基石。

### 14.2 datasink-patterns.md

datasink 的识别模式，按类型分类（数据库/文件/网络/RPC/审批流/mass assignment/auth decision），每种类型列出不同语言/框架的典型代码模式。

### 14.3 auth-anchor-recognition.md

可信锚点的识别方法论，包括：
- 锚点候选源（session/token/JWT/内部SDK/DRM/mist/RPC上下文）
- 锚点可信度判断（R10 详细检查清单）
- 锚点传播追踪方法
- 常见锚点命名约定（跨语言参考）

### 14.4 output-semantics.md

出参语义判断方法论，包括：
- 11 种终止状态判断矩阵的详细说明
- 布尔/枚举返回值深度分析模板
- 多源字段处理策略
- 信息预言（oracle）风险识别

### 14.5 scenario-taxonomy.md

越权场景分类体系 S1-S8，每个场景的：
- 详细触发条件（带代码示例）
- 与其他场景的边界区分
- 严重度判定标准
- 误报排除指引

### 14.6 evidence-requirements.md

证据三要素规范：
- `file_ref`: 文件路径 + 行号格式要求
- `rule_ref`: 规则引用格式（R1-R10, S1-S8, 终止判断编号）
- `code_snippet`: 代码片段格式要求（1-3行，不截断关键表达式）
- 各阶段产物中 reason/trust_chain/risk_reason 字段的证据标准

---

## 15. JSON Schema 体系

### 15.1 设计原则

- 每个阶段的输出 JSON 有对应的 JSON Schema
- Schema 定义必需字段（required）、字段类型（type）、枚举值（enum）
- 所有 reason/judgment/notes 类型字段必须存在
- code_snippet 字段为必需字段

### 15.2 Schema 文件列表

| Schema 文件 | 对应产物 | 关键约束 |
|------------|----------|----------|
| `recon.schema.json` | recon.json | entry_file 非空，auth_anchors 至少一项，credibility_checklist 全部字段 |
| `entry.schema.json` | entry.json | 参数 primitive_type/source/semantic_role 为必填枚举 |
| `callchain.schema.json` | callchain.json | node 含 line_start/line_end/signature，edge 含 argument_mappings |
| `callchain-verify.schema.json` | callchain-verify.json | severity 为 high/medium/low 枚举 |
| `forward.schema.json` | forward.json | trusted_pool 和 parameter_analysis 的 propagation_chain 含 trust_rule 引用 |
| `forward-verify.schema.json` | forward-verify.json | 所有检查类别数组存在 |
| `backward.schema.json` | backward.json | output_fields 的 final_judgment 为枚举，at_risk 必须有 risk_reason |
| `backward-verify.schema.json` | backward-verify.json | 检查类别覆盖 |
| `report.schema.json` | report.json | scenarios_found 含 evidence_chain，fix_suggestion 非空 |

### 15.3 Schema 示例结构

以 `recon.schema.json` 为例：

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["endpoint", "framework", "auth_context", "interceptor_chain"],
  "properties": {
    "endpoint": {
      "type": "object",
      "required": ["identifier", "entry_file", "entry_function", "entry_line", "code_snippet"],
      "properties": {
        "identifier": {"type": "string", "minLength": 1},
        "entry_file": {"type": "string", "minLength": 1},
        "code_snippet": {"type": "string", "minLength": 1}
      }
    },
    "auth_context": {
      "type": "object",
      "required": ["auth_anchors", "global_filters"],
      "properties": {
        "auth_anchors": {
          "type": "array",
          "minItems": 1,
          "items": {
            "required": ["name", "source_type", "credibility_checklist", "code_snippet"]
          }
        }
      }
    }
  }
}
```

---

## 16. 证据完整性原则

### 16.1 原则声明

> 每个 agent 输出的字段值和原因字段必须满足两个「不能」：
> 1. 不能留空模糊词
> 2. 不能缺关联证据

### 16.2 证据三要素

每个判断字段（`reason`, `trust_chain`, `risk_reason`, `final_judgment`, `final_trust_status`, `trust_effect`, `issue`, `note` 等）必须附带：

| 要素 | 字段名 | 说明 |
|------|--------|------|
| 代码位置 | `file` + `line` | 判断依据所在的文件和行号 |
| 规则引用 | `rule_ref` / `trust_rule` | 引用的方法论规则编号（R1-R10, S1-S8, 终止判断 1-11） |
| 代码片段 | `code_snippet` | 从源码中读取的关键行（1-3行），直接展示判断依据 |

### 16.3 模糊词禁止列表

以下词汇和类似表达禁止出现在任何产物的判断字段中：
- "可能"、"也许"、"大概"、"似乎"
- "should be"、"might be"、"probably"、"seems like"
- "待确认"、"需进一步分析"（除非紧跟具体缺少什么信息）
- "一般"、"通常"、"大部分情况"

替代做法：如果确实不确定，标注 `"unresolved"` 并在对应 `*_reason` / `unresolved_reason` 字段中写明：
- 具体缺少什么信息
- 为什么当前信息不足以做出判断
- 建议如何获取缺失的信息

### 16.4 违规处理

主 agent 在每个 Gate 检查产物时，扫描 reason/trust_chain/risk_reason 等字段：
- 发现模糊词 → 判定为质量不合格，打回重做
- 发现 code_snippet 为空 → 判定为证据不足，打回重做
- 发现 rule_ref 缺失 → 判定为方法论偏差，打回重做

---

## 17. 项目文件结构

```
authbuddy/
├── SKILL.md                              # 主 agent：编排器、状态机、门控、回补决策
├── README.md                             # 项目说明
├── agents/
│   ├── agent-00-recon.md                # 前置侦查：prompt、工作流程、输出要求
│   ├── agent-01-entry.md                # 入参提取：语义角色分类、展开原则
│   ├── agent-02-callchain.md            # 调用链构建：拓扑规则、终止条件
│   ├── agent-02b-callchain-verify.md    # 调用链验证：审计检查清单
│   ├── agent-03-forward.md              # 正向信任传播：可信池构建、参数追踪
│   ├── agent-03b-forward-verify.md      # 正向验证：假可信/漏判检测
│   ├── agent-04-backward.md             # 反向出参回溯：终止判断矩阵、深度分析
│   ├── agent-04b-backward-verify.md     # 反向验证：假安全/多源遗漏检测
│   └── agent-05-report.md              # 报告生成：S1-S8 场景分类、report.md/json
├── reference/
│   ├── trust-propagation-rules.md       # R1-R10 信任传播规则（多语言示例）
│   ├── datasink-patterns.md             # 资源访问点识别模式
│   ├── auth-anchor-recognition.md       # 可信锚点识别方法论
│   ├── output-semantics.md              # 出参语义判断方法论
│   ├── scenario-taxonomy.md             # 越权场景分类体系 S1-S8
│   └── evidence-requirements.md         # 证据三要素规范
├── schemas/
│   ├── recon.schema.json
│   ├── entry.schema.json
│   ├── callchain.schema.json
│   ├── callchain-verify.schema.json
│   ├── forward.schema.json
│   ├── forward-verify.schema.json
│   ├── backward.schema.json
│   ├── backward-verify.schema.json
│   └── report.schema.json
└── docs/
    └── spec.md                           # 本设计规格书
```

---

## 附录 A：设计决策记录

| # | 决策 | 结论 | 依据 |
|---|------|------|------|
| 1 | 主 agent 角色 | 纯调度型，不读代码 | 减少主 agent 上下文负担，子 agent 产出物供裁决 |
| 2 | 流水线阶段 | 5阶段 + 3验证，串行 | 有数据依赖，必须前一步完成才能后一步 |
| 3 | Recon 定位 | 独立第0步，子 agent 执行 | 上下文构建独立，不混入入参分析 |
| 4 | 调用链粒度 | 纯拓扑 + 实参映射 + 注解 + call_type_markers | 分离关注点，减少调用链 agent 上下文压力 |
| 5 | 验证机制 | 审计模式（B），发现缺失→回补原始 agent | 省 token，保持一致性 |
| 6 | 正向分析方式 | C 方案：先建可信池→再评估参数 | 防止遗漏间接信任传导 |
| 7 | 反向分析终止 | 11 种语义驱动终止判断 | 不按类型截断，按语义状态判断 |
| 8 | 最终裁决 | 按越权场景 S1-S8 出报告 | 场景驱动，而非参数驱动 |
| 9 | 产物强制 | 每个 agent 必须写文件 | 防止以对话代替文件输出 |
| 10 | 证据要求 | file_ref + rule_ref + code_snippet | 证据完整性，不靠模糊推断 |
| 11 | 交付形态 | Claude Code Skill 集合（方案3） | 模块化但不过度拆分 |
| 12 | 深度控制 | 语义终止条件，不设固定深度 | 防止复杂调用链被截断 |

## 附录 B：与参考项目的关系

| 借鉴来源 | 借鉴内容 | AuthBuddy 中的体现 |
|----------|----------|-------------------|
| authscan-2 | R1-R10 信任传播规则 | reference/trust-propagation-rules.md |
| authscan-2 | Layer 0-3 分层分析 | 正向+反向两阶段分析 |
| authscan-2 | 每端点独立原则 | 单接口审计定位 |
| authscan-2 | 自学习协议 | SKILL.md 中 extended-knowledge 机制 |
| code-audit-main | 执行控制器 + 门控 | SKILL.md 状态机 + Gate 1-4 |
| code-audit-main | 防幻觉规则 | 证据完整性原则 |
| code-audit-main | 双轨审计模型 | sink-driven (datasink 识别) + control-driven (授权点识别) |
| code-audit-main | Multi-Agent 并行 | 9 个子 agent 流水线 |
| llm-code-analysis (VulSolver) | Interest/Sink 定义 | interest=项目内函数, sink=datasink/授权关键点 |
| llm-code-analysis (VulSolver) | 路径探索+验证两阶段 | 调用链构建(A02) + 验证(A02b)，正向分析(A03) + 验证(A03b) |
| llm-code-analysis (VulSolver) | 结构化 JSON 输出 | 全部产物为 JSON + JSON Schema 验证 |
| llm-code-analysis (VulSolver) | 直接下一跳 | 调用链边只记录直接下一跳 |
