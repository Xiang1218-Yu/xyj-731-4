# ProseMirror 核心源码分析

> 基于本仓库 `model / transform / state / view` 四个核心 workspace 的源码，系统解析 ProseMirror 富文本编辑器的文档模型设计原理。

本文档面向想要深入理解 ProseMirror 架构的工程师，所有结论都直接标注到源码位置（可点击跳转）。

## 目录

- [1. 整体架构总览](#1-整体架构总览)
- [2. 文档模型：Schema / Node / Mark](#2-文档模型schema--node--mark)
- [3. Transaction 事务机制与不可变性](#3-transaction-事务机制与不可变性)
- [4. Selection 选区系统](#4-selection-选区系统)
- [5. 核心模块依赖关系图](#5-核心模块依赖关系图)
- [6. 与 Slate / Quill 的架构对比](#6-与-slate--quill-的架构对比)
- [7. 延伸阅读](#7-延伸阅读)

详细专题拆解见同目录下：

- [document-model.md](./document-model.md) — Schema / Node / Mark 深度解析
- [transaction.md](./transaction.md) — Transaction / Step / 不可变性
- [selection.md](./selection.md) — Selection 选区系统
- [architecture.md](./architecture.md) — 模块依赖与整体设计

---

## 1. 整体架构总览

ProseMirror 不是一个"开箱即用的编辑器"，而是一组分层的、职责单一的构建块。四个核心包自底向上依赖：

```
┌──────────────────────────────────────────────────────────────┐
│                     prosemirror-view                           │
│  DOM 渲染 / 事件处理 / DOMObserver / ViewDesc 虚拟描述树        │
│  职责：把不可变的 EditorState 映射为可编辑 DOM，并把 DOM 事件   │
│        转换为 Transaction                                      │
└───────────────────────────┬──────────────────────────────────┘
                            │ 依赖
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                     prosemirror-state                          │
│  EditorState（不可变状态）/ Transaction / Selection / Plugin   │
│  职责：编辑器"当前状态"的权威来源，聚合 doc + selection +      │
│        storedMarks + 插件状态                                  │
└───────────────────────────┬──────────────────────────────────┘
                            │ 依赖
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                   prosemirror-transform                        │
│  Step（原子变更）/ StepMap / Mapping / Transform               │
│  职责：以可逆、可映射、可序列化的 Step 描述文档变更            │
│        （协同编辑 OT 的基础）                                  │
└───────────────────────────┬──────────────────────────────────┘
                            │ 依赖
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                     prosemirror-model                          │
│  Schema / Node / Mark / Fragment / Slice / ResolvedPos         │
│  职责：定义"文档是什么"——持久化（不可变）的树形数据结构        │
│        与其约束规则                                            │
└──────────────────────────────────────────────────────────────┘
```

**核心设计哲学：**

| 原则 | 含义 | 体现 |
|------|------|------|
| 不可变数据结构 | 文档、状态从不被原地修改，每次变更产生新对象并共享结构 | [Node](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/node.ts#L22-L38) 注释明确 "persistent data structures" |
| Schema 驱动 | 文档结构由 Schema 严格约束，非法结构无法构造 | [NodeType.checkContent](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L198-L201) |
| 变更即数据 | 每个变更是一个可序列化、可反转、可映射的 Step 对象 | [Step](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/step.ts#L16-L67) |
| 单向数据流 | DOM 事件 → Transaction → 新 State → 重渲染 DOM | [state.ts applyInner](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L171-L179) |

---

## 2. 文档模型：Schema / Node / Mark

### 2.1 三个核心概念

ProseMirror 的文档是一棵**树**。树的每个节点是 `Node`，节点上可以附着 `Mark`，而整个结构必须符合 `Schema` 的约束。

```
Schema（规则/工厂）
  ├── nodes: { doc, paragraph, heading, text, image, ... }  ── 每个是一个 NodeType
  └── marks: { strong, em, link, code, ... }                ── 每个是一个 MarkType
           │
           │ 创建/校验
           ▼
Document (Node: type=doc)
  └── content: Fragment
        ├── Node(type=paragraph)
        │     └── content: Fragment
        │           ├── TextNode("Hello ", marks=[])
        │           └── TextNode("world", marks=[strong, link])   ← Mark 附着在 Node 上
        └── Node(type=heading, attrs={level:1})
              └── content: Fragment [ TextNode("Title") ]
```

- **Schema**（[schema.ts:571-630](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L571-L630)）：文档的"类型系统"。持有所有 `NodeType` 和 `MarkType`，负责创建节点、校验内容、序列化/反序列化。
- **Node**（[node.ts:22-38](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/node.ts#L22-L38)）：树中的一个节点，持久化不可变。由 `type`、`attrs`、`content`（子节点 Fragment）、`marks` 四要素构成。
- **Mark**（[mark.ts:10-17](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/mark.ts#L10-L17)）：附着在（通常是内联）节点上的元信息，如加粗、链接。由 `type` + `attrs` 构成，同样不可变。

### 2.2 关键关系

**① Node 与 Mark 是"内容/装饰"关系，而非父子关系。** 这是 ProseMirror 区别于很多编辑器的核心：加粗不是包裹文本的 `<strong>` 节点，而是文本节点携带的一个 mark 标签。相邻且 mark 相同的文本会被自然合并。

**② NodeType/MarkType 每个 Schema 只实例化一次。** 见 [NodeType.compile](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L235-L245) 和 [MarkType.compile](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L314-L318)。因此可以用 `===` 直接比较类型。

**③ 内容合法性由 ContentMatch（内容表达式的有限状态机）保证。** Schema 构造时把 `content: "paragraph+"` 这样的表达式编译成状态机（`ContentMatch.parse` 调用见 [schema.ts:609-610](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L609-L610)，所在编译循环 [schema.ts:605-620](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L605-L620)），创建/替换节点时校验：

```ts
// model/src/schema.ts:187-193 —— 校验 Fragment 是否是该节点类型的合法内容
validContent(content: Fragment) {
  let result = this.contentMatch.matchFragment(content)
  if (!result || !result.validEnd) return false
  for (let i = 0; i < content.childCount; i++)
    if (!this.allowsMarks(content.child(i).marks)) return false
  return true
}
```

**④ Mark 集合是"有序、去重、可互斥"的不可变数组。** 见 [Mark.addToSet](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/mark.ts#L24-L45)：按 `MarkType.rank` 排序插入，遇到互斥 mark（如两个不同颜色）会替换。互斥关系在 Schema 构造时解析 [schema.ts:621-624](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L621-L624)。

> 详见 [document-model.md](./document-model.md)

---

## 3. Transaction 事务机制与不可变性

### 3.1 状态的不可变性

`EditorState` 明确是"持久化数据结构"——不会被更新，而是从旧 state 计算出新 state（[state.ts:83-95](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L83-L95)）。变更流程：

```
oldState ──(state.tr)──> Transaction ──(变更方法)──> 累积 steps
                                                        │
oldState.apply(tr) ─────────────────────────────────────┤
                                                        ▼
                                        applyInner: 逐字段 field.apply(tr,...)
                                                        │
                                                        ▼
                                        全新的 newState（旧 state 原封不动）
```

关键点：`applyInner` 会**新建**一个 `EditorState`，为每个字段调用 `apply` 计算新值，旧实例完全不被触碰（[state.ts:171-179](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L171-L179)）。并且校验 `tr.before.eq(this.doc)`，防止把事务应用到不匹配的文档上。

### 3.2 Transaction = Transform + 状态元信息

`Transaction` 继承自 `Transform`（[transaction.ts:42](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts#L42)）。分工：

- **Transform**（[transform.ts:28-41](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/transform.ts#L28-L41)）：只关心文档变更，维护 `steps[]`、`docs[]`（每步前的文档）、`mapping`。
- **Transaction**：在此之上叠加 selection、storedMarks、时间戳、metadata、scroll 意图等编辑器状态。

每一次 `addStep` 都是 append，旧文档保存在 `docs[]` 中，`doc` 指向最新版本（[transform.ts:89-94](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/transform.ts#L89-L94)）。

### 3.3 Step —— 原子、可逆、可映射

不可变性的基石是 `Step`：一个描述原子变更的对象（[step.ts:16-67](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/step.ts#L16-L67)）。它的 `apply(doc)` **返回新文档而非修改入参**：

```ts
// transform/src/replace_step.ts:28-40（节选，getMap/invert 已压缩为单行示意）
apply(doc: Node) {
  if (this.structure && contentBetween(doc, this.from, this.to))
    return StepResult.fail("Structure replace would overwrite content")
  return StepResult.fromReplace(doc, this.from, this.to, this.slice)  // 产生新 Node
}
getMap() { return new StepMap([this.from, this.to - this.from, this.slice.size]) }
invert(doc: Node) { /* 返回反向 Step —— 撤销的基础 */ }
```

Step 的四个能力构成整个体系的支柱：

| 方法 | 作用 | 用途 |
|------|------|------|
| `apply` | 应用到文档，返回新文档 | 执行变更 |
| `invert` | 生成反向 step | Undo/History |
| `map` | 通过 mapping 调整自身位置 | 协同编辑 rebase |
| `getMap` | 返回位置映射 StepMap | 位置追踪 |

### 3.4 位置映射（StepMap / Mapping）

文档一变，所有旧位置都可能失效。`StepMap` 用 `[start, oldSize, newSize]` 三元组描述每个变更块，`map(pos)` 把旧位置换算成新位置（[map.ts:98-116](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/map.ts#L98-L116)）。事务里的 selection 正是靠它自动跟随变更（见 §4.3）。

> 详见 [transaction.md](./transaction.md)

---

## 4. Selection 选区系统

### 4.1 抽象基类与三种实现

`Selection` 是抽象基类（[selection.ts:9-23](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L9-L23)），核心是两个**已解析位置** `$anchor`（锚点，不动）和 `$head`（头，随操作移动），派生出 `ranges`。

```
                    Selection (abstract)
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
  TextSelection     NodeSelection      AllSelection
  文本/光标选区      整节点选区          全文档选区
  ($cursor 判空)    (选中 image 等)     (Ctrl+A)
```

- **TextSelection**（[selection.ts:229-305](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L229-L305)）：经典文本选区，`$cursor` 非空即为折叠光标。
- **NodeSelection**（[selection.ts:325-376](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L325-L376)）：选中单个节点（如图片），`from/to` 恰好包住该节点。
- **AllSelection**（[selection.ts:399-420](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L399-L420)）：全选，能表达纯文本选区无法表达的边界情况。

### 4.2 ResolvedPos —— 选区定位的关键

选区不存"裸数字位置"，而是 `ResolvedPos`（[resolvedpos.ts:12-28](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/resolvedpos.ts#L12-L28)）：一个整数位置被"解析"成 `{path, depth, parentOffset}`，可以 O(1) 拿到"这个位置在第几层、父节点是谁、在父节点里第几个 index"。选区的所有语义（能否放光标、往哪走）都基于它计算。

### 4.3 选区如何随文档变更自动跟随

这是选区系统最精妙的部分。Transaction 的 `selection` 是 getter，惰性地把旧选区 `map` 到最新文档：

```ts
// state/src/transaction.ts:71-77
get selection(): Selection {
  if (this.curSelectionFor < this.steps.length) {
    // 把选区通过"尚未映射的那部分 steps"映射到新文档
    this.curSelection = this.curSelection.map(this.doc, this.mapping.slice(this.curSelectionFor))
    this.curSelectionFor = this.steps.length
  }
  return this.curSelection
}
```

各选区类型自行实现 `map`。例如 TextSelection 映射后若落点不在内联内容里，会退化为就近合法选区（[selection.ts:241-246](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L241-L246)）。`Selection.near/findFrom/atStart`（[selection.ts:118-151](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L118-L151)）负责寻找"最近的合法选区落点"，保证选区永远有效。

> 详见 [selection.md](./selection.md)

---

## 5. 核心模块依赖关系图

```
                          ┌─────────────────────────┐
                          │   prosemirror-view      │
                          │                         │
                          │  EditorView             │
                          │  ViewDesc (虚拟描述树)  │
                          │  DOMObserver            │
                          │  input / clipboard      │
                          │  decoration             │
                          └───────────┬─────────────┘
                                      │ import
              ┌───────────────────────┼───────────────────────┐
              ▼                       ▼                       ▼
   ┌──────────────────┐   ┌──────────────────────┐  (model 直接可用)
   │ prosemirror-state│   │ prosemirror-transform│
   │                  │   │                      │
   │ EditorState      │──▶│ Transform            │
   │ Transaction ─────┼───┤ Step / ReplaceStep   │
   │ Selection        │   │ StepMap / Mapping     │
   │ Plugin           │   │ structure / replace   │
   └────────┬─────────┘   └──────────┬───────────┘
            │                        │
            │ import                 │ import
            ▼                        ▼
   ┌────────────────────────────────────────────┐
   │            prosemirror-model                │
   │                                             │
   │  Schema ── NodeType / MarkType              │
   │  Node / TextNode / Fragment                 │
   │  Mark                                       │
   │  Slice / ReplaceError                       │
   │  ResolvedPos                                │
   │  ContentMatch (内容状态机)                  │
   │  DOMParser / DOMSerializer                  │
   └────────────────────────────────────────────┘
```

**依赖方向严格自上而下，无循环：**

| 模块 | 依赖 | 关键导入证据 |
|------|------|------|
| view | state, transform, model | `import {...} from "prosemirror-view"` 上游 |
| state | transform, model | [transaction.ts:1-2](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts#L1-L2) |
| transform | model | [transform.ts:1](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/transform.ts#L1) |
| model | 仅 orderedmap | [schema.ts:1](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L1) |

**模块内部依赖（model 内部）：**

```
schema.ts ──▶ node.ts ──▶ fragment.ts ──▶ mark.ts
    │            │              │
    │            ├──▶ replace.ts (Slice)
    │            └──▶ resolvedpos.ts
    └──▶ content.ts (ContentMatch)
```

> 详见 [architecture.md](./architecture.md)

---

## 6. 与 Slate / Quill 的架构对比

| 维度 | **ProseMirror** | **Slate** | **Quill** |
|------|-----------------|-----------|-----------|
| 文档模型 | 严格 Schema 约束的树；Node + Mark 分离 | JSON 树（无强制 schema，靠 normalize 规则约束） | Delta（扁平的 op 列表，非树） |
| 数据结构 | 持久化不可变（结构共享的树） | 不可变（Immer，操作产生新对象） | Delta 值对象（compose/transform 返回新 Delta，非结构共享树） |
| 变更表示 | Step（原子/可逆/可映射/可序列化） | Operation（9 种基础 op） | Delta op（insert/retain/delete） |
| 状态管理 | EditorState + Transaction 单向流 | React 受控组件 + Editor 对象 | 内部 model，命令式 API |
| Schema | 一等公民，编译成内容状态机强校验 | 无内建 schema，靠 `normalizeNode` | 无 schema，靠 formats 白名单 |
| 内联样式 | Mark（附着于节点，独立于结构） | Text 节点上的属性（leaf） | Delta 属性（attributes） |
| 位置系统 | 整数偏移 + ResolvedPos（可解析上下文） | Path + Offset（数组路径） | 单一整数索引 |
| 协同编辑 | 原生支持（Step 可 rebase，OT 基础） | 需第三方（如 slate-yjs） | Delta 天生适合 OT |
| 渲染层 | 自建 ViewDesc 虚拟树，框架无关 | 依赖 React | 自建，命令式 DOM |
| 定位 | 底层工具箱，需自行组装 | 偏底层，与 React 深度绑定 | 高层开箱即用 |

### 架构差异的本质

```
ProseMirror ── "文档是受 Schema 约束的不可变树，变更是可逆的原子 Step"
    → 强类型、可协同、可扩展；但学习曲线陡，需自己搭积木

Slate ────── "文档是可自定义的 JSON 树，深度拥抱 React"
    → 灵活、React 生态友好；但大文档性能与 schema 一致性需自己兜底

Quill ────── "文档是线性 Delta 序列，命令式 API"
    → 简单、上手快、协同天然；但表达复杂嵌套结构（如表格）能力弱
```

**一句话总结：**
- ProseMirror 用**树 + Schema + Step** 换来了结构正确性、可扩展性与协同能力；
- Slate 用**可插拔 JSON 树 + React** 换来了灵活性；
- Quill 用**线性 Delta** 换来了简单与协同友好，但牺牲了结构表达力。

---

## 7. 延伸阅读

- 各 workspace 自带的 `README.md`（如 [model/src/README.md](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/README.md)）为官方 API 参考。
- 专题文档：
  - [document-model.md](./document-model.md)
  - [transaction.md](./transaction.md)
  - [selection.md](./selection.md)
  - [architecture.md](./architecture.md)
