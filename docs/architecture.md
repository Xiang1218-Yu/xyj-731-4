# 核心模块依赖关系与整体架构

> 对应源码：`model/`、`transform/`、`state/`、`view/` 四个 workspace

本篇给出 ProseMirror 核心模块的分层、依赖方向与协作方式的全景。

---

## 1. 四层分层架构

ProseMirror 采用严格自底向上、无循环依赖的分层设计：

```
┌════════════════════════════════════════════════════════════════┐
│ 第 4 层  prosemirror-view    —— 展示与交互（唯一接触 DOM 的层）  │
├════════════════════════════════════════════════════════════════┤
│ 第 3 层  prosemirror-state   —— 状态编排（当前状态的权威来源）   │
├════════════════════════════════════════════════════════════════┤
│ 第 2 层  prosemirror-transform —— 文档变更（原子、可逆、可映射） │
├════════════════════════════════════════════════════════════════┤
│ 第 1 层  prosemirror-model   —— 数据结构（文档是什么）           │
└════════════════════════════════════════════════════════════════┘
     依赖方向：上层 ──依赖──▶ 下层，反之绝不成立
```

每层的单一职责：

| 层 | 包 | 职责 | 是否接触 DOM |
|----|----|----|----|
| 1 | model | 定义不可变文档树及其 Schema 约束 | 否（仅 DOMParser/Serializer 桥接） |
| 2 | transform | 用 Step 描述并应用文档变更 | 否 |
| 3 | state | 聚合 doc+selection+plugins，用 Transaction 演进 | 否 |
| 4 | view | 渲染 state 为可编辑 DOM，收集 DOM 事件转 Transaction | 是 |

---

## 2. 模块依赖图（含内部文件）

```
                         ┌───────────────────────────────────────┐
                         │         prosemirror-view              │
                         │  index / viewdesc / domobserver       │
                         │  input / clipboard / decoration       │
                         │  domchange / domcoords / selection    │
                         └──────┬──────────────┬─────────────┬────┘
                    import      │      import   │    import   │
            ┌────────────────────┘              │             └──────────────┐
            ▼                                   ▼                            ▼
┌───────────────────────────┐   ┌───────────────────────────────┐   (直接使用 model)
│    prosemirror-state       │   │     prosemirror-transform     │
│  state.ts   ──────────────┐│   │  transform.ts                 │
│  transaction.ts (extends ─┼┼──▶│  step.ts / replace_step.ts    │
│     Transform)            ││   │  mark_step.ts / attr_step.ts  │
│  selection.ts             ││   │  structure.ts / replace.ts    │
│  plugin.ts                ││   │  map.ts (StepMap/Mapping)     │
└───────────┬───────────────┘│   └──────────────┬────────────────┘
            │  import         │ import           │  import
            └─────────────────┴──────────────────┘
                              ▼
            ┌────────────────────────────────────────────┐
            │            prosemirror-model                │
            │                                             │
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
| state → transform, model | [transaction.ts:1-2](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts#L1-L2) `import {Transform, Step} from "prosemirror-transform"`、`from "prosemirror-model"` |
| state → view（仅类型） | [transaction.ts:3](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts#L3) `import {type EditorView}` —— 仅类型引用，不构成运行时依赖 |
| transform → model | [transform.ts:1](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/transform.ts#L1)、[step.ts:1](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/step.ts#L1) |
| model → orderedmap | [schema.ts:1](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L1) |
| state 内部 selection → transform | [selection.ts:1-2](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L1-L2) |

> 注：`transaction.ts` 对 `EditorView` 只是 `import {type ...}`（TypeScript 纯类型导入，编译后消失），因此 **state 与 view 之间没有运行时循环依赖**，分层仍然严格单向。

---

## 3. 运行时数据流（单向闭环）

```
   ┌──────────────┐   DOM 事件(键盘/鼠标/输入法/粘贴)   ┌──────────────┐
   │   浏览器 DOM  │ ─────────────────────────────────▶ │  EditorView  │
   └──────▲───────┘                                     │  (view 层)   │
          │                                             └──────┬───────┘
          │ ⑤ 视图 diff 更新 DOM                                │ ① 事件 → 生成 Transaction
          │   (ViewDesc 比对)                                   ▼
   ┌──────┴───────┐         ④ dispatchTransaction     ┌──────────────┐
   │  EditorView  │ ◀────────────────────────────────  │ Transaction  │
   │  update(new) │                                     │ (state 层)   │
   └──────────────┘                                     └──────┬───────┘
          ▲                                                    │ ② 累积 Step
          │ ③ state.apply(tr) 产生全新不可变 newState           ▼
   ┌──────┴───────────────────────────────────────────┐ ┌──────────────┐
   │              EditorState (state 层)                │ │  Step/Transform│
   │  doc + selection + storedMarks + plugin fields    │ │ (transform 层) │
   └───────────────────────────────────────────────────┘ └──────┬───────┘
                                                                 │ apply(doc)→新 doc
                                                                 ▼
                                                          ┌──────────────┐
                                                          │  Node/Schema │
                                                          │  (model 层)  │
                                                          └──────────────┘
```

闭环解读：
1. **view** 捕获 DOM 事件，生成 `Transaction`。
2. **transform/model**：事务累积 `Step`，每步 `apply` 基于 model 产出新的不可变 `doc`。
3. **state**：`state.apply(tr)` 校验并产出**全新** `EditorState`。
4. **view**：`dispatchTransaction` 拿到新 state 后调用 `view.update`。
5. **view** 用 ViewDesc 虚拟描述树 diff，最小化更新真实 DOM，回到步骤 1。

数据永远单向流动，没有双向绑定；DOM 只是 state 的一个"投影"。

---

## 4. 各层核心导出一览

### model（数据结构）
- 类型系统：`Schema` / `NodeType` / `MarkType`（[schema.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts)）
- 数据：`Node` / `TextNode`（[node.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/node.ts)）、`Fragment`、`Mark`（[mark.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/mark.ts)）、`Slice`
- 定位：`ResolvedPos`（[resolvedpos.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/resolvedpos.ts)）
- 约束：`ContentMatch`（content.ts）
- DOM 桥接：`DOMParser` / `DOMSerializer`

### transform（变更）
- `Step`（抽象）及子类 `ReplaceStep`/`ReplaceAroundStep`/`AddMarkStep`/`AttrStep` 等
- `StepMap` / `Mapping` / `MapResult`（[map.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/map.ts)）
- `Transform`（步骤累积器，[transform.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/transform.ts)）

### state（状态编排）
- `EditorState`（[state.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts)）
- `Transaction`（[transaction.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts)）
- `Selection` 家族（[selection.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts)）
- `Plugin` / `PluginKey`（plugin.ts）

### view（展示交互）
- `EditorView`、`ViewDesc`（虚拟描述树，viewdesc.ts）
- `DOMObserver`（监听 DOM 变化，domobserver.ts）
- 输入/剪贴板/装饰：input.ts / clipboard.ts / decoration.ts

---

## 5. 架构设计的关键取舍

| 设计选择 | 好处 | 代价 |
|----------|------|------|
| 严格分层、单向依赖 | 每层可独立测试/替换；view 可换渲染后端 | 需要更多"胶水"代码组装 |
| 状态不可变 | 可预测、可撤销、可协同、易调试 | 频繁分配对象（靠结构共享缓解） |
| Schema 强约束 | 结构永远合法，杜绝脏数据 | 配置成本高，学习曲线陡 |
| Step 原子化 | 可逆、可映射、可序列化 → 协同编辑地基 | 复杂操作需拆解为多步 |
| 插件通过 state 字段扩展 | 扩展能力随事务一致演进 | 插件需理解事务生命周期 |

---

## 6. 一句话总结

ProseMirror 的核心架构是一条清晰的**单向管线**：

> **model 定义"文档是什么" → transform 定义"如何合法地改" → state 定义"当前状态与如何演进" → view 定义"如何展示与交互"。**

每一层都建立在下一层的不可变原语之上，数据单向流动、绝不循环。这正是它能同时做到"结构严谨、可协同、可扩展、框架无关"的根本原因。
