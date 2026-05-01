---
name: code-dataflow-analysis
description: 以一个函数为入口，使用数据流跟踪与调用链分析方法，梳理外部输入参数从源头（source）到关键操作点（sink，如文件操作、命令执行、网络操作、数据库操作等）的完整传播路径，并如实记录路径上经历的关键校验与关键转换。**仅做事实性梳理与结构化输出，不做安全风险判定**，结论留给人工。语言无关，支持 Java、Python、Go、Rust、JavaScript/TypeScript 等主流语言。任何时候用户给出一个函数（包括"分析这个函数"、"看一下这个 handler 的入参怎么用的"、"跟一下 xxx 函数的污点流"、"这个接口的参数最后到哪里去了"、"看一下用户传进来的字段会走到哪些 IO 操作"），或者请求 taint analysis / data flow / call chain / sink 分析，都应触发本 skill。
---

# 通用代码数据流与调用链分析

## 适用范围与基本原则

1. **语言无关**。方法论适用于任何语言；遇到具体语言时只需把"赋值/调用/字段访问/集合元素/字符串拼接/反序列化"等通用概念映射到该语言的语法即可。SKILL.md 不内置语言专属规则。
2. **只陈述事实，不下安全结论**。报告中可以写"该路径上未发现对 `path` 的规范化或路径遍历过滤"，**不要写**"存在路径穿越漏洞"。是否构成风险，由阅读报告的人结合上下文判断。
3. **完备性 > 简洁性**。若某外部输入流向多个 sink，全部列出；若有多条路径到同一 sink，都列出（按可达性合并明显冗余的分支即可）。
4. **不可达的不写**。如果某条路径明显被前置条件 short-circuit 掉（例如 `if False:` / 死代码 / 提前 return），如实标注并跳过。
5. **遇到不确定的诚实标注**。看不到实现的库调用、动态分派、反射、跨进程/跨服务调用，使用 `[unresolved]` / `[external]` / `[dynamic]` 等标签，不要瞎猜。

---

## 核心方法论

整体流程分四步：**定位入口 → 识别 source / sink → 前向传播跟踪 → 路径汇总输出**。每一步只做必要工作，避免分析无关代码。

### Step 1 — 入口与外部输入识别（Sources）

对入口函数 `F`：

- **Source 集合**包括：
  - `F` 的形参（除非明显是常量/枚举/已被框架完全约束的类型）。
  - 在 `F` 或其下游调用中读取的**外部数据**：环境变量、命令行参数、stdin、配置文件读入、HTTP/RPC 请求体或 query/header/cookie、消息队列消息、文件读入的不可信内容、远端 RPC 返回值、数据库返回值（视场景）、反序列化对象等。
- 对每个 source，记录：名称、类型（如能识别）、引入位置（文件:行）、引入方式（参数 / 读环境 / 读请求 / 读文件 …）。

> 注意：是否把"DB 返回值"或"内部 RPC"算 source 取决于上下文；不确定时默认按 source 处理，并在备注里注明来源（如"来源为 DB 字段 X，是否可信请人工判断"）。

### Step 2 — Sink 分类（关键操作点）

按**行为类型**分类，不区分语言。常见 sink 类别（非穷举）：

| 类别 | 行为示例 |
|---|---|
| 文件操作 | 打开、读、写、追加、删除、移动、改名、改权限、创建符号链接、解压到磁盘 |
| 命令/进程 | 执行 shell 命令、spawn 子进程、加载动态库、`eval` / 动态执行代码 |
| 网络 | 发起 HTTP/TCP/UDP 请求、连接到地址、发送数据、监听端口 |
| 数据库 / 存储 | 执行 SQL / NoSQL 查询、构造查询字符串、写入持久层、缓存写入 |
| 序列化 / 反序列化 | 解析 JSON/YAML/XML/pickle/二进制协议 |
| 鉴权 / 权限 | 判定身份、设置 session、签发 token、变更 ACL |
| 模板 / 渲染 | 模板引擎渲染、HTML/SQL 拼接输出 |
| 日志 / 出口 | 写日志、上报指标（仅在涉及外发时记录） |

碰到一个调用，先问"它的副作用属于哪一类"。属于上表的就标为 sink。**只关心被 source 数据触达的 sink**——与污点无关的 sink 可以不写。

### Step 3 — 前向传播跟踪（Forward Data Flow + Call Chain）

从每个 source 出发，沿**赋值、传参、返回值、字段写入、集合存入、字符串/字节拼接、格式化、编码解码、解析、拷贝**等动作向前推进，直到到达 sink 或终止。

需要持续跟踪的传播形式：

- **直接传播**：`y = x`、`return x`、`obj.f = x`、`list.append(x)`、`map[k] = x`。
- **派生传播**：`y = f"prefix/{x}"`、`y = x + "_suffix"`、`y = json.dumps({"k": x})`、`y = base64.encode(x)`、`y = x.strip()` 等——派生值仍带污点。
- **跨函数传播（调用链分析）**：把 source 作为实参传给函数 `g(...)`，要进入 `g` 的函数体继续跟踪，记录调用栈。`g` 内部的形参成为新的污点变量。
  - 若 `g` 是项目内函数：进入实现继续跟。
  - 若 `g` 是标准库 / 三方库 / 框架方法：根据其**有据可查**的语义判定（透传 / 校验 / 转换 / 终止）。无把握时标 `[external: 行为未知]`。
  - 跨方法时记录调用栈深度和位置，便于阅读者还原路径。
- **容器/字段传播**：污点存入对象字段、集合元素、闭包捕获后，从该容器/字段被读出时**继续带污点**。
- **控制流敏感性（轻量）**：能识别的提前返回、`if/else` 分支要分支处理，但不必过度展开复杂的路径敏感分析；以"该路径在某条件下成立"标注即可。

跟踪时**记录两类节点**：

#### (a) 关键校验（Validations / Gates）

任何对当前污点变量进行**判断并据此改变控制流**的操作。如实记录：
- 校验位置（文件:行）、校验主体表达式（伪代码即可）。
- 校验**通过**和**未通过**分支各自的去向（继续传播 / 抛错 / 提前 return / 走默认值 / …）。
- **不要**评价校验是否充分。

常见校验形态：类型/空值检查、长度/范围检查、正则匹配、`startswith`/`endswith`/`contains`、白名单/黑名单匹配、规范化后比较（如 `realpath` 后 `startswith(base)`）、签名/校验和验证、鉴权检查、解析成功与否的判定。

#### (b) 关键转换（Transformations）

改变值形态但**不改变**污点状态的操作。需要记录的是：操作类型 + 是否引入常量/受控前后缀。例如：

- 拼接：`base_dir + "/" + user_name`（注明常量部分）。
- 编码/转义：`url_encode(x)`、`html_escape(x)`、`shell_quote(x)`、`json.dumps`、`base64`。
- 解析：`json.loads(x)` 后取字段——污点跟随到字段上。
- 规范化：`os.path.realpath(x)`、`Path(x).resolve()`、`strings.ToLower(x)`、`unicode.NFC`。
- 截断/替换：`x[:100]`、`x.replace("..", "")`。

转换本身不是校验，但读者会非常关心"sink 处的最终形态是什么"，**所以最终写到 sink 的表达式要尽量还原**（见输出样例）。

### Step 4 — 终止条件

一条传播路径在以下情况终止：
- 到达一个 sink → 记录为命中。
- 数据被丢弃（赋值后未再使用）。
- 数据被一个**总是抛错或终止**的分支吞掉。
- 进入 `[external]` 且无法继续 → 记录终点为该外部调用，标注未跟。
- 出现循环/递归 → 收敛后停止，避免无限展开（标注一次）。

---

## 输出格式（必须严格遵守）

对每次分析都输出**两部分**：先一段精简的人类可读 Markdown，再附一段等价的 JSON（便于工具处理）。两者信息一致。

### Markdown 部分

```markdown
# Dataflow Report: <FunctionName>

## Entry
- Function: `pkg.module.FunctionName(arg1, arg2, ...)`
- Location: `path/to/file.ext:LN`
- Language: <java | python | go | rust | js | ts | ...>

## Sources
- S1: param `arg1` (type=<...>, kind=parameter)
- S2: env `APP_BASE_DIR` read at `path/to/file.ext:LN` (kind=env)
- S3: HTTP body field `payload.name` at `path/to/handler.ext:LN` (kind=http-body)
  ...

## Sinks Reached
- K1: file.delete — `os.remove(target)` at `path/to/util.ext:LN`
- K2: command.exec — `exec.Command("sh", "-c", cmd)` at `path/to/runner.ext:LN`
  ...

## Flows

### Flow F1: S1 → K1 (file.delete)
- Call chain: `FunctionName` → `buildPath` → `removeIfExists`
- Steps:
  1. `path/to/file.ext:LN` — `name = arg1` (assignment)
  2. `path/to/file.ext:LN` — `if ".." in name: raise` (validation: reject containing "..", failure → raise)
  3. `path/to/file.ext:LN` — `buildPath(name)` (call, taint follows into param `n`)
  4. `path/to/util.ext:LN` — `target = base_dir + "/" + n` (transform: concat with constant `base_dir`)
  5. `path/to/util.ext:LN` — `removeIfExists(target)` (call)
  6. `path/to/util.ext:LN` — `os.remove(target)` (sink K1)
- Validations on path:
  - V1 @ file.ext:LN — `".." in name` rejects → raise
- Transformations on path:
  - T1 @ util.ext:LN — concat: `<const base_dir> + "/" + <tainted name>`
- Final expression at sink: `os.remove(<const base_dir> + "/" + arg1)` after V1
- Notes: none

### Flow F2: S3 → K2 (command.exec)
... (同上结构)

## Unreached Sources
- S2 (`APP_BASE_DIR`): used as constant prefix only; not propagated to any tracked sink type.

## Open Questions
- `removeIfExists` calls `fs_helper.Drop(target)` from third-party crate `xfs v0.4`; behavior of `Drop` not resolved — treated as terminal sink K1 here. Please confirm whether `Drop` performs additional checks.
- Branch at `handler.ext:LN` depends on runtime config `cfg.strict_mode`; both branches enumerated above.
```

### JSON 部分

```json
{
  "entry": {
    "function": "pkg.module.FunctionName",
    "signature": "FunctionName(arg1, arg2)",
    "location": "path/to/file.ext:LN",
    "language": "go"
  },
  "sources": [
    {"id": "S1", "kind": "parameter", "name": "arg1", "type": "string", "location": "path/to/file.ext:LN"},
    {"id": "S3", "kind": "http-body", "name": "payload.name", "type": "string", "location": "path/to/handler.ext:LN"}
  ],
  "sinks": [
    {"id": "K1", "category": "file.delete", "call": "os.remove(target)", "location": "path/to/util.ext:LN"},
    {"id": "K2", "category": "command.exec", "call": "exec.Command(\"sh\",\"-c\",cmd)", "location": "path/to/runner.ext:LN"}
  ],
  "flows": [
    {
      "id": "F1",
      "source": "S1",
      "sink": "K1",
      "call_chain": ["FunctionName", "buildPath", "removeIfExists"],
      "steps": [
        {"location": "file.ext:LN", "kind": "assign", "expr": "name = arg1"},
        {"location": "file.ext:LN", "kind": "validation", "expr": "\"..\" in name", "on_fail": "raise", "on_pass": "continue"},
        {"location": "file.ext:LN", "kind": "call", "expr": "buildPath(name)", "taint_into": "n"},
        {"location": "util.ext:LN", "kind": "transform", "op": "concat", "expr": "target = base_dir + \"/\" + n", "constants": ["base_dir", "/"]},
        {"location": "util.ext:LN", "kind": "call", "expr": "removeIfExists(target)"},
        {"location": "util.ext:LN", "kind": "sink", "sink": "K1", "expr": "os.remove(target)"}
      ],
      "validations": [
        {"id": "V1", "location": "file.ext:LN", "predicate": "\"..\" in name", "on_fail": "raise"}
      ],
      "transformations": [
        {"id": "T1", "location": "util.ext:LN", "op": "concat", "constants": ["base_dir", "/"]}
      ],
      "final_expr_at_sink": "os.remove(base_dir + \"/\" + arg1) [after V1]"
    }
  ],
  "unreached_sources": ["S2"],
  "open_questions": [
    "removeIfExists -> third-party fs_helper.Drop semantics not resolved",
    "branch on cfg.strict_mode enumerated both sides"
  ]
}
```

---

## 风格与禁忌

- **不要**给修复建议。这是另一项工作。
- **不要**在事实里夹议论；如果有提示，统一放在 `Open Questions` 中并以问题形式表述。
- **不要**用模糊量词（"很多"、"大量"、"看起来"）。具体到 `文件:行` 和表达式。
