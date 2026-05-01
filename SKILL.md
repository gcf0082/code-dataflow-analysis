---
name: code-dataflow-analysis
description: 以一个函数为入口，使用数据流跟踪与调用链分析方法，梳理外部输入参数从源头（source）到关键操作点（sink，如文件操作、命令执行、网络操作、数据库操作等）的完整传播路径，并如实记录路径上经历的关键校验与关键转换。**仅做事实性梳理与结构化输出，不做安全风险判定**，结论留给人工。语言无关，支持 Java、Python、Go、Rust、JavaScript/TypeScript 等主流语言。任何时候用户给出一个函数（包括"分析这个函数"、"看一下这个 handler 的入参怎么用的"、"跟一下 xxx 函数的污点流"、"这个接口的参数最后到哪里去了"、"看一下用户传进来的字段会走到哪些 IO 操作"），或者请求 taint analysis / data flow / call chain / sink 分析，都应触发本 skill。
---

# 通用代码数据流与调用链分析

## 基本原则

1. **语言无关**。
2. **只陈述事实，不下安全结论**；是否构成风险由人工判断。
3. **完备性 > 简洁性**：多 source / 多 sink / 多路径全部列出。
4. **不确定的诚实标注**：用 `[unresolved]` / `[external]` / `[dynamic]` 等标签，不要瞎猜。

---

## 核心方法论

整体流程：**定位入口 → 识别 source / sink → 前向传播跟踪 → 相关 source 合并 → 路径汇总输出**。

### Step 1 — Sources

入口函数 `F` 的形参，加上 `F` 调用链上读到的外部数据（环境、命令行、stdin、HTTP/RPC 请求、配置、文件读入、IPC、反序列化对象、视上下文的 DB/RPC 返回值等）。每个 source 记：名称、类型、引入位置（文件:行）、引入方式。

### Step 2 — Sinks

按行为类型分类，常见类别：

| 类别 | 行为示例 |
|---|---|
| 文件 | 读 / 写 / 删 / 移 / 改权限 / 链接 / 解压 |
| 命令/进程 | shell exec / spawn / 加载动态库 / `eval` |
| 网络 | HTTP/TCP/UDP 发起 / 监听 |
| 数据库 / 存储 | SQL/NoSQL 查询、写入持久层、缓存写入 |
| 序列化 | JSON / YAML / XML / pickle 等解析 |
| 鉴权 / 权限 | 身份判定、签发 token、变更 ACL |
| 模板 / 渲染 | 模板渲染、HTML/SQL 拼接输出 |
| 日志 / 出口 | 写日志、上报指标（仅外发时记录） |

**所有命中的 sink 都要列出**，与本次 source 无关的在条目末尾注明理由（`*(无污点：<理由>)*`），便于读者确认你考察过它。

**默认影响等级**（每个 sink 必填）：按 sink 操作类型的固有属性打标，**不参考链路校验/转换**，**不按上下文覆盖**。

| 影响 | 类型 |
|---|---|
| 高 | 命令执行 / `eval` / 动态加载；不可信反序列化；文件写、删、改权限、改名、链接、解压；DB 写；外发网络；鉴权变更；模板渲染输出 |
| 中 | 文件读/移；DB 查询；内部 RPC；缓存写；外发日志/指标 |
| 低 | 可信反序列化；配置读；本地日志；鉴权读；网络监听 |

### Step 3 — 前向传播跟踪

从每个 source 沿数据流走向 sink；途中保持以下处理：

- **跨函数**：调用项目内函数时进入实现继续跟，记录调用栈；外部库依据可查文档判断（透传 / 校验 / 转换 / 终止），不确定标 `[external]`。
- **派生不丢污点**：编码、转义、规范化、截断、字段读取、容器存取后仍带污点。
- **控制流**：分支会影响传播则分支处理，不做完整路径敏感展开。
- **终止**：命中 sink、值被丢弃、被抛错/return 吞掉、进入不可达的外部、循环收敛后即停止。

跟踪时在节点上记录两类信息：

#### (a) 关键校验

对污点变量做判断且改变控制流的操作：记位置、表达式、通过/未通过分支的去向。**不评价校验是否充分。**

#### (b) 关键转换

改变值形态但不丢污点：记操作类型 + 引入的常量部分。**到达 sink 时的最终表达式要尽量还原**（见模板）。

#### (c) 业务意图（每条 flow 必填）

每条 flow 用 1–3 句自然语言说明在业务上做什么、关键校验/转换在该业务里扮演什么角色；多 source 时分别说明各自角色。**不评价是否充分或安全。**

### Step 4 — 相关 Source 合并

业务上紧密相关、沿同一调用链流向同一组 sink 的多个 source，合并到同一条 flow。**判断准则：拆开后是否会反复重述同一段链路？是 → 合并；否 → 各自独立。** 合并后的 flow 头用集合写法 `Flow F1: {S1, S2} → K1`。

### Step 5 — 方法核对记录

走入**任何**名字+签名暗示了明确契约的方法（utility 类的校验/规范化/编解码/解析是常见来源，但不限于此——领域方法、服务方法、内部 helper 同样适用），完整核对、行为与契约相符时，追加一条到 `methods-verified.md`（每核对完一个立即落盘）。下次分析撞到同样方法可直接引用，不必重做。

行为不符或未完整核对的不进此表——前者放 Open Questions 或在 flow 步骤里陈述，后者属于未完成。

---

## 输出形态

**输出一个目录，不是单个文件。** 目录名建议 `<入口函数名>-dataflow/`（用户已指定路径则尊重用户）。

- **简单**（1–2 条 flow、链路短）：目录里一个 `report.md`，所有内容内联。
- **复杂**（多 flow / 长链路 / 多 Open Questions）：

  ```
  <function>-dataflow/
  ├── index.md          # 入口、Sources、Sinks、Flow 索引、Unreached、Open Questions
  ├── flow-F1.md        # 单条 Flow 完整描述，自包含
  ├── flow-F2.md
  ├── methods-verified.md # 可选：方法核对记录（追加增量）
  └── ...
  ```

简单情形下 `methods-verified.md` 也单独成文件，不内联到 `report.md`——固定命名便于跨任务复用。

每分析完一条 flow / 一个独立小模块 / 一次方法核对就**立即落盘**，不要囤到最后。

每个 `flow-Fx.md` **自包含**：重述本条涉及的 source / sink 摘要、调用链、步骤、校验、转换、最终表达式，不依赖 `index.md` 也能独立看懂。多条 flow 共用同一段代码时各自展开，让 flow 之间互不干扰。

---

## 输出格式（每个文件的结构必须严格遵守，示例使用 Java）

### `index.md`（或简单情形下的 `report.md`）

```markdown
# 数据流报告：UserFileService.deleteUserFile

## 入口
- 函数：`com.example.UserFileService#deleteUserFile(String userId, String name, User user)`
- 位置：`src/main/java/com/example/UserFileService.java:23`
- 语言：java

## 输入源（Sources）
- S1：参数 `userId`（类型=String，来源=方法形参）
- S2：参数 `name`（类型=String，来源=方法形参）
- S3：环境变量 `APP_BASE_DIR`，读取于 `src/main/java/com/example/Config.java:14`（来源=环境）
- S4：HTTP 请求体字段 `payload.note`，引入于 `src/main/java/com/example/FileHandler.java:31`（来源=HTTP 请求体）

## 命中的关键操作点（Sinks）
- K1：文件删除 [影响=高] — `Files.delete(target)` 位于 `src/main/java/com/example/PathBuilder.java:18`
- K2：命令执行 [影响=高] — `Runtime.getRuntime().exec(cmd)` 位于 `src/main/java/com/example/Runner.java:42`
- K3：文件写入 [影响=高] — `Files.write(logPath, bytes)` 位于 `src/main/java/com/example/AuditLog.java:27`  *(无污点：`logPath` 由常量 `LOG_DIR` 与当前进程 PID 拼接，没有 source 流入)*

## 传播路径索引
- F1：{S1, S2} → K1（文件删除）— 详见 `flow-F1.md`
- F2：S4 → K2（命令执行）— 详见 `flow-F2.md`

> 简单情形（单文件 `report.md`）：把上面索引替换为各 flow 的完整章节内联展开（结构见下方 flow 文件模板的"业务意图"以下部分）。

## 未流出的输入源（Unreached Sources）
- S3（`APP_BASE_DIR`）：仅作为常量前缀使用，未传播到任何被跟踪的 sink 类型。

## 待确认问题（Open Questions）
- `PathBuilder#buildAndRemove` 内部调用 `com.thirdparty.fs.Cleaner.drop(target)`（来自 fs-helper 1.4）；`drop` 的行为未解析，此处当作终止 sink K1 处理。请确认 `drop` 是否还有额外校验。
- `FileHandler.java:31` 处的分支取决于运行时配置 `cfg.strictMode`；上述路径已枚举两个分支。
```

### `flow-Fx.md`

```markdown
# Flow F1：{S1, S2} → K1（文件删除）

## 摘要
- Source S1：参数 `userId`（String），引入于 `UserFileService.java:23`
- Source S2：参数 `name`（String），引入于 `UserFileService.java:23`
- Sink K1：`Files.delete(target)` 位于 `PathBuilder.java:18`，类别=文件删除，影响=高
- 调用链：`UserFileService#deleteUserFile` → `PathBuilder#buildAndRemove`
- 合并理由：`userId` 与 `name` 共同决定要删的文件位置，沿同一条调用链流向同一个 sink，拆开会反复重述同一段链路。

## 业务意图
该方法负责按 `userId` 定位用户在数据目录下的子目录，再删除其中名为 `name` 的文件。链路上 `name.contains("..")` 用于阻止跨目录引用；`Paths.get(BASE_DIR, userId, name)` 把最终路径锚定在常量基目录 `/var/data` 之下；`userId` 直接参与路径拼接，路径上未观察到对它的校验。

## 步骤
1. `UserFileService.java:24` — `if (name.contains("..")) throw new IllegalArgumentException(...)`（校验 `name`：包含 ".." 时抛出 IllegalArgumentException，命中 → 抛错；未命中 → 继续。`userId` 在该处未参与校验。）
2. `UserFileService.java:27` — `PathBuilder.buildAndRemove(userId, name)`（跨方法调用，`userId` 与 `name` 同时跟随到形参 `u`、`n`）
3. `PathBuilder.java:17` — `Path target = Paths.get(BASE_DIR, u, n)`（转换：与常量 `BASE_DIR="/var/data"` 三段拼接）
4. `PathBuilder.java:18` — `Files.delete(target)`（命中 sink K1）

## 路径上的校验
- V1 @ `UserFileService.java:24` — `name.contains("..")` 命中即抛 IllegalArgumentException（仅作用于 S2=`name`；S1=`userId` 路径上未观察到校验）

## 路径上的转换
- T1 @ `PathBuilder.java:17` — 路径拼接：`Paths.get(<常量 "/var/data">, <带污点 userId>, <带污点 name>)`

## 到达 sink 时的最终表达式
`Files.delete(Paths.get("/var/data", userId, name))`，其中 `name` 前置经过 V1，`userId` 未经过校验。

## 备注
无。
```

### `methods-verified.md`

```markdown
# 方法核对记录

> 此文件追加增量；每条记录一个被核对过、实现与名称契约相符的方法（utility / 领域 / 服务 / helper 皆可）。
> 后续分析撞到同样方法时可直接引用本文件，无需重新核对。

## com.example.IpUtils#isValidIpv4(String s)
- 位置：`src/main/java/com/example/IpUtils.java:12`
- 预期行为：校验输入是否为四段、每段 0–255 的 IPv4 字面量字符串
- 实际逻辑：按 `.` 切分后长度=4；每段 `Integer.parseInt` 后判断 0–255；非数字段抛 `NumberFormatException` 被 catch 后返回 false
- 边界：`null` → 提前返回 false；空串 → false；前置零如 `"01.0.0.0"` 按数字解析仍接受
- 结论：符合预期，可作 IPv4 字面量校验使用
```

五项字段固定；"结论"用"符合预期，…"句式开头便于跨任务检索。

---

## 风格与禁忌

- **不要**给修复建议。
- **不要**在事实里夹议论；提示统一放在 `Open Questions` 中以问题形式表述。
- **不要**用模糊量词（"很多"、"大量"、"看起来"）。具体到 `文件:行` 和表达式。
