# ProseMirror 核心源码分析

> 基于本仓库 `model / transform / state / view` 四个核心 workspace 的源码，系统解析 ProseMirror 富文本编辑器文档模型的设计原理。所有结论均标注可点击的源码位置。

## 文档导航

| 文档 | 内容 |
|------|------|
| 本文件（根目录 README） | 总览、架构图、模块依赖、与 Slate/Quill 对比 |
| [docs/document-model.md](./docs/document-model.md) | Schema / Node / Mark 深度解析 |
| [docs/transaction.md](./docs/transaction.md) | Transaction / Step / 不可变性 |
| [docs/selection.md](./docs/selection.md) | Selection 选区系统 |
| [docs/architecture.md](./docs/architecture.md) | 模块依赖与整体设计 |

---

## 1. 分层架构图

ProseMirror 采用严格自底向上、无循环依赖的四层分层设计：

```
┌════════════════════════════════════════════════════════════════┐
│ 第 4 层  prosemirror-view    —— 展示与交互（唯一接触 DOM 的层）  │
│   EditorView · ViewDesc(虚拟描述树) · DOMObserver · input       │
├════════════════════════════════════════════════════════════════┤
│ 第 3 层  prosemirror-state   —— 状态编排（当前状态的权威来源）   │
│   EditorState · Transaction · Selection · Plugin                │
├════════════════════════════════════════════════════════════════┤
│ 第 2 层  prosemirror-transform —— 文档变更（原子·可逆·可映射）   │
│   Step · ReplaceStep · StepMap · Mapping · Transform            │
├════════════════════════════════════════════════════════════════┤
│ 第 1 层  prosemirror-model   —— 数据结构（文档是什么）           │
│   Schema · Node · Mark · Fragment · Slice · ResolvedPos         │
└════════════════════════════════════════════════════════════════┘
        依赖方向：上层 ──依赖──▶ 下层，反之绝不成立
```

| 层 | 包 | 职责 | 接触 DOM |
|----|----|------|:--------:|
| 1 | model | 定义不可变文档树及 Schema 约束 | 否 |
| 2 | transform | 用 Step 描述并应用文档变更 | 否 |
| 3 | state | 聚合 doc+selection+plugins，用 Transaction 演进 | 否 |
| 4 | view | 渲染 state 为可编辑 DOM，收集事件转 Transaction | 是 |

---

## 2. 模块依赖关系图（含内部文件）

```
                         ┌───────────────────────────────────────┐
                         │         prosemirror-view              │
                         │  index / viewdesc / domobserver       │
                         │  input / clipboard / decoration       │
                         └──────┬──────────────┬─────────────┬────┘
                    import      │      import   │    import   │
            ┌────────────────────┘              │             └──────────────┐
            ▼                                   ▼                            ▼
┌───────────────────────────┐   ┌───────────────────────────────┐   (直接使用 model)
│    prosemirror-state       │   │     prosemirror-transform     │
│  state.ts                  │   │  transform.ts                 │
│  transaction.ts (extends ──┼──▶│  step.ts / replace_step.ts    │
│     Transform)             │   │  mark_step.ts / attr_step.ts  │
│  selection.ts              │   │  structure.ts / replace.ts    │
│  plugin.ts                 │   │  map.ts (StepMap/Mapping)     │
└───────────┬───────────────┘   └──────────────┬────────────────┘
            │  import                           │  import
            └───────────────┬───────────────────┘
                            ▼
            ┌────────────────────────────────────────────┐
            │            prosemirror-model                │
            │  schema.ts  ──▶ node.ts ──▶ fragment.ts     │
            │      │             │            │           │
            │      │             ├──▶ mark.ts             │
            │      │             ├──▶ replace.ts (Slice)  │
            │      │             └──▶ resolvedpos.ts      │
            │      └──▶ content.ts (ContentMatch 状态机)  │
            │  from_dom.ts / to_dom.ts (DOM 桥接)         │
            └────────────────────────────────────────────┘
                            │
                            ▼
                    orderedmap (唯一外部依赖)
```

依赖证据（关键 import）：

| 关系 | 证据 |
|------|------|
| view → state, model | [view/src/index.ts:1-2](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/view/src/index.ts#L1-L2) |
| state → transform, model | [transaction.ts:1-2](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts#L1-L2) |
| state → view（仅类型） | [transaction.ts:3](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts#L3) `import {type EditorView}` — 纯类型导入，无运行时循环依赖 |
| transform → model | [transform.ts:1](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/transform.ts#L1) |
| model → orderedmap | [schema.ts:1](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L1) |

---

## 3. 运行时数据流图（单向闭环）

```
   ┌──────────────┐   DOM 事件(键盘/鼠标/输入法/粘贴)   ┌──────────────┐
   │   浏览器 DOM  │ ─────────────────────────────────▶ │  EditorView  │
   └──────▲───────┘                                     │  (view 层)   │
          │                                             └──────┬───────┘
          │ ⑤ 视图 diff 更新 DOM (ViewDesc 比对)               │ ① 事件 → Transaction
          │                                                   ▼
   ┌──────┴───────┐        ④ dispatchTransaction      ┌──────────────┐
   │  EditorView  │ ◀────────────────────────────────  │ Transaction  │
   │  update(new) │                                     │ (state 层)   │
   └──────────────┘                                     └──────┬───────┘
          ▲                                                    │ ② 累积 Step
          │ ③ state.apply(tr) → 全新不可变 newState             ▼
   ┌──────┴───────────────────────────────────────────┐ ┌───────────────┐
   │              EditorState (state 层)                │ │ Step/Transform │
   │  doc + selection + storedMarks + plugin fields    │ │ (transform 层) │
   └───────────────────────────────────────────────────┘ └──────┬────────┘
                                                                 │ apply(doc)→新 doc
                                                                 ▼
                                                          ┌──────────────┐
                                                          │  Node/Schema │
                                                          │  (model 层)  │
                                                          └──────────────┘
```

1. **view** 捕获 DOM 事件，生成 `Transaction`。
2. **transform/model**：事务累积 `Step`，每步 `apply` 产出新的不可变 `doc`。
3. **state**：`state.apply(tr)` 校验并产出**全新** `EditorState`（[state.ts:171-179](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L171-L179)）。
4. **view**：`dispatchTransaction` 拿到新 state 后 `view.update`。
5. **view** 用 ViewDesc 虚拟描述树 diff，最小化更新真实 DOM，回到步骤 1。

数据永远单向流动，DOM 只是 state 的一个"投影"。

---

## 4. 文档模型三要素关系图

```
             ┌──────────────────────────────────────────┐
             │                 Schema                     │
             │  (每 Schema 仅一份 NodeType/MarkType 实例) │
             └───────────────┬──────────────┬────────────┘
                   compile   │              │  compile
                   ┌─────────▼──┐        ┌──▼─────────┐
                   │  NodeType  │        │  MarkType  │
                   │ contentMatch│       │  excluded  │
                   │  markSet   │        │   rank     │
                   └─────┬──────┘        └─────┬──────┘
              create/check│  实例化              │ 实例化
                   ┌──────▼──────┐        ┌──────▼──────┐
                   │    Node     │  持有  │    Mark     │
                   │ type/attrs  │◀───────│ type/attrs  │
                   │ content     │ marks  └─────────────┘
                   │ marks       │
                   └──────┬──────┘
                          │ content
                   ┌──────▼──────┐
                   │  Fragment   │  ← 子 Node 序列
                   └─────────────┘
```

- **Schema 造 NodeType/MarkType；NodeType/MarkType 造 Node/Mark**（[NodeType.compile](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L235-L245)、[MarkType.compile](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L314-L318)）。
- **Node 通过 `content: Fragment` 组成树，通过 `marks` 附着装饰**（[node.ts:22-38](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/node.ts#L22-L38)）。
- **合法性双重校验**：结构靠 `contentMatch`（[schema.ts:198-201](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L198-L201)），装饰靠 `markSet` + `excluded`。

---

## 5. 核心概念速览

### 5.1 Schema / Node / Mark（要求 1）
- **Schema**（[schema.ts:571-630](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L571-L630)）是文档的"类型系统"，构造时把 `content` 表达式编译成 `ContentMatch` 状态机做强校验。
- **Node**（[node.ts:22-38](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/node.ts#L22-L38)）是持久化不可变树节点，`copy/mark/cut` 全部返回新对象、复用旧内容。
- **Mark**（[mark.ts:10-17](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/mark.ts#L10-L17)）与树结构**解耦**——文本节点携带有序去重的 mark 集合（[addToSet](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/mark.ts#L24-L45)），规避 HTML 嵌套歧义。

### 5.2 Transaction 与不可变性（要求 2）
- `EditorState` 是持久化数据结构，`applyInner` 始终**新建**实例且校验 `tr.before.eq(doc)`（[state.ts:171-179](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L171-L179)）。
- `Transaction extends Transform`（[transaction.ts:42](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts#L42)），变更以原子 `Step` 累积，`addStep` 纯追加保留历史文档（[transform.ts:89-94](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/transform.ts#L89-L94)）。
- `Step` 四能力 `apply/invert/map/toJSON`（[step.ts:16-46](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/step.ts#L16-L46)）支撑撤销与协同。

### 5.3 Selection 选区系统（要求 3）
- 基于 `ResolvedPos` 上下文（[resolvedpos.ts:12-52](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/resolvedpos.ts#L12-L52)），三种类型 `TextSelection`/`NodeSelection`/`AllSelection`。
- `Selection.near` 兜底（[selection.ts:135-137](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L135-L137)）+ 事务里惰性 `map`（[transaction.ts:71-77](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts#L71-L77)）保证选区永远有效并自动跟随变更。

---

## 6. 与 Slate / Quill 的架构对比（要求 5）

| 维度 | **ProseMirror** | **Slate** | **Quill** |
|------|-----------------|-----------|-----------|
| 文档模型 | 严格 Schema 约束的树；Node + Mark 分离 | JSON 树（无强制 schema，靠 normalize） | Delta（扁平 op 列表，非树） |
| 数据结构 | 持久化不可变（结构共享的树） | 不可变（操作产生新对象） | Delta 值对象（compose/transform 返回新 Delta，非结构共享树） |
| 变更表示 | Step（原子/可逆/可映射/可序列化） | Operation（9 种基础 op） | Delta op（insert/retain/delete） |
| 状态管理 | EditorState + Transaction 单向流 | React 受控组件 + Editor 对象 | 内部 model，命令式 API |
| Schema | 一等公民，编译成内容状态机强校验 | 无内建 schema，靠 `normalizeNode` | 无 schema，靠 formats 白名单 |
| 内联样式 | Mark（附着于节点，独立于结构） | Text 节点属性（leaf） | Delta 属性（attributes） |
| 位置系统 | 整数偏移 + ResolvedPos（可解析上下文） | Path + Offset（数组路径） | 单一整数索引 |
| 协同编辑 | 原生支持（Step 可 rebase，OT 基础） | 需第三方（如 slate-yjs） | Delta 天生适合 OT |
| 渲染层 | 自建 ViewDesc 虚拟树，框架无关 | 依赖 React | 自建，命令式 DOM |
| 定位 | 底层工具箱，需自行组装 | 偏底层，与 React 深度绑定 | 高层开箱即用 |

**一句话总结：**
- ProseMirror 用**树 + Schema + 可逆 Step** 换来结构正确性、可扩展性与协同能力；
- Slate 用**可插拔 JSON 树 + React** 换来灵活性；
- Quill 用**线性 Delta** 换来简单与协同友好，但牺牲了结构表达力。

---

## 7. 延伸阅读

- 各 workspace 自带的 `src/README.md`（如 [model/src/README.md](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/README.md)）为官方 API 参考。
- 深入专题：[docs/](./docs) 目录下四篇文档。
