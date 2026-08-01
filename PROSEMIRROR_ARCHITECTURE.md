# ProseMirror 核心架构深度解析

> 本文档基于 ProseMirror v1.25.x 核心源码深度分析，涵盖文档模型、事务机制、选区系统、模块依赖及与同类编辑器的架构对比。

---

## 目录

- [一、整体架构图](#一整体架构图)
- [二、核心模块依赖关系](#二核心模块依赖关系)
- [三、文档模型（Document Model）](#三文档模型document-model)
  - [3.1 Schema：文档的"宪法"](#31-schema文档的宪法)
  - [3.2 Node：文档树的基本单元](#32-node文档树的基本单元)
  - [3.3 Mark：附加在内联内容上的标记](#33-mark附加在内联内容上的标记)
  - [3.4 Fragment 与 ResolvedPos](#34-fragment-与-resolvedpos)
- [四、Transaction 事务机制与不可变性](#四transaction-事务机制与不可变性)
  - [4.1 持久化数据结构](#41-持久化数据结构)
  - [4.2 Step：原子变更单元](#42-step原子变更单元)
  - [4.3 StepMap 与位置映射](#43-stepmap-与位置映射)
  - [4.4 Transform → Transaction 的继承链](#44-transform--transaction-的继承链)
  - [4.5 EditorState.apply：状态更新流程](#45-editorstateapply状态更新流程)
- [五、Selection 选区系统](#五selection-选区系统)
  - [5.1 选区基类与 Range 模型](#51-选区基类与-range-模型)
  - [5.2 三种内置选区类型](#52-三种内置选区类型)
  - [5.3 选区映射（Mapping）机制](#53-选区映射mapping机制)
  - [5.4 Bookmark：文档无关的选区快照](#54-bookmark文档无关的选区快照)
- [六、Plugin 插件系统](#六plugin-插件系统)
- [七、ProseMirror vs Slate vs Quill 架构对比](#七prosemirror-vs-slate-vs-quill-架构对比)
- [八、关键源码索引](#八关键源码索引)

---

## 一、整体架构图

ProseMirror 采用**分层、模块化**的架构设计，核心由四个低耦合的包组成，自底向上依次为 `model` → `transform` → `state` → `view`。

```
┌─────────────────────────────────────────────────────────────────────┐
│                        prosemirror-view                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │
│  │ EditorView  │  │  Decoration  │  │  NodeView / DOMSerializer │  │
│  │  (DOM 渲染)  │  │  (装饰系统)   │  │  (自定义节点视图)          │  │
│  └─────────────┘  └──────────────┘  └───────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ 依赖
┌───────────────────────────────▼─────────────────────────────────────┐
│                       prosemirror-state                             │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │
│  │ EditorState │  │ Transaction  │  │  Selection / Plugin       │  │
│  │  (不可变状态)│  │  (事务扩展)   │  │  (选区/插件)              │  │
│  └─────────────┘  └──────────────┘  └───────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ 依赖
┌───────────────────────────────▼─────────────────────────────────────┐
│                     prosemirror-transform                           │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │
│  │  Transform  │  │    Step      │  │  StepMap / Mapping        │  │
│  │ (变更构建器) │  │ (原子变更)    │  │  (位置映射/逆操作)         │  │
│  └─────────────┘  └──────────────┘  └───────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────┘
                                │ 依赖
┌───────────────────────────────▼─────────────────────────────────────┐
│                       prosemirror-model                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌─────────────────────┐ │
│  │  Schema  │  │   Node   │  │   Mark   │  │  Fragment / Slice   │ │
│  │ (模式定义)│  │ (文档节点)│  │ (内联标记)│  │  ResolvedPos / DOM  │ │
│  └──────────┘  └──────────┘  └──────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

### 数据流转全景图

```
                    ┌──────────────────┐
                    │   用户交互/命令    │
                    └────────┬─────────┘
                             │ 创建
                             ▼
                    ┌──────────────────┐
                    │   Transaction    │  继承自 Transform
                    │  (doc + selection│
                    │   + storedMarks  │
                    │   + meta)        │
                    └────────┬─────────┘
                             │ state.apply(tr)
                             ▼
              ┌──────────────────────────────┐
              │       EditorState            │
              │  ┌─────┐ ┌──────┐ ┌───────┐ │
              │  │ doc │ │ sel  │ │ marks │ │  不可变！
              │  └─────┘ └──────┘ └───────┘ │
              │  + plugin state fields       │
              └──────────────┬───────────────┘
                             │ view.updateState
                             ▼
                    ┌──────────────────┐
                    │   EditorView     │
                    │  (DOM 差异化更新)  │
                    └──────────────────┘
```

---

## 二、核心模块依赖关系

根据各包 `package.json` 的依赖声明：

| 模块 | 版本 | 依赖 | 职责 |
|------|------|------|------|
| **prosemirror-model** | 1.25.x | `orderedmap` | 文档模型：Schema、Node、Mark、Fragment、Slice、DOM 解析/序列化 |
| **prosemirror-transform** | 1.12.x | `prosemirror-model` | 文档转换：Step、Transform、StepMap、Mapping、结构操作 |
| **prosemirror-state** | 1.4.x | `prosemirror-model`, `prosemirror-transform`, `prosemirror-view` (类型) | 编辑器状态：EditorState、Transaction、Selection、Plugin |
| **prosemirror-view** | 1.42.x | `prosemirror-model`, `prosemirror-state`, `prosemirror-transform` | 视图层：DOM 渲染、事件处理、Decoration、NodeView |

> **注意**：`prosemirror-state` 对 `prosemirror-view` 的依赖仅用于类型导入（`import type`），不构成运行时循环依赖。真正的运行时依赖方向是 view → state → transform → model，形成清晰的单向依赖链。

---

## 三、文档模型（Document Model）

ProseMirror 的文档模型是其整个架构的基石。它定义了文档的**结构规则**和**数据表示**。

### 3.1 Schema：文档的"宪法"

`Schema` 是一个文档的"模式定义"，它规定了：
- 文档中允许出现哪些**节点类型**（NodeType）
- 允许出现哪些**标记类型**（MarkType）
- 每种节点的**内容表达式**（content expression）
- 节点的**属性**（attributes）及其默认值/验证规则
- 节点之间的**嵌套规则**

#### 核心源码分析

Schema 的构造函数位于 [schema.ts#L595-L630](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/schema.ts#L595-L630)：

```typescript
export class Schema<Nodes extends string = any, Marks extends string = any> {
  spec: {
    nodes: OrderedMap<NodeSpec>,
    marks: OrderedMap<MarkSpec>,
    topNode?: string
  }

  nodes: {readonly [name in Nodes]: NodeType} & {...}
  marks: {readonly [name in Marks]: MarkType} & {...}
  topNodeType: NodeType

  constructor(spec: SchemaSpec<Nodes, Marks>) {
    // 1. 将 nodes/marks 转换为 OrderedMap（保持顺序）
    instanceSpec.nodes = OrderedMap.from(spec.nodes)
    instanceSpec.marks = OrderedMap.from(spec.marks || {})

    // 2. 编译 NodeType 和 MarkType（每个类型只创建一次实例）
    this.nodes = NodeType.compile(this.spec.nodes, this)
    this.marks = MarkType.compile(this.spec.marks, this)

    // 3. 为每个 NodeType 解析内容表达式和 mark 集合
    for (let prop in this.nodes) {
      let type = this.nodes[prop]
      type.contentMatch = ContentMatch.parse(
        type.spec.content || "", this.nodes
      )
      type.inlineContent = type.contentMatch.inlineContent
      type.markSet = markExpr == "_" ? null
        : markExpr ? gatherMarks(this, markExpr.split(" "))
        : markExpr == "" || !type.inlineContent ? [] : null
    }
    // ...
  }
}
```

**设计要点**：

1. **类型对象单例化**：每个 `NodeType` 和 `MarkType` 在一个 Schema 中只创建一次，所有同类型的 Node 实例共享同一个 type 对象。这通过 `NodeType.compile()` 实现（[schema.ts#L235-L245](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/schema.ts#L235-L245)）。

2. **内容表达式（Content Expression）**：使用类似正则表达式的语法描述允许的子节点序列，如 `"paragraph+"`、`"(paragraph | blockquote)*"`、`"heading paragraph+"`。这些表达式被 `ContentMatch.parse()` 编译成有限状态自动机，用于验证内容合法性。

3. **NodeSpec 关键配置项**（[schema.ts#L371-L490](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/schema.ts#L371-L490)）：
   - `content`: 内容表达式
   - `marks`: 允许的 mark 集合
   - `group`: 所属分组（可在内容表达式中引用）
   - `inline`/`atom`/`selectable`/`draggable`: 行为标志
   - `defining`/`isolating`: 边界语义
   - `attrs`: 属性定义
   - `toDOM`/`parseDOM`: DOM 序列化/解析规则

#### Schema、NodeType、Node 的关系图

```
┌─────────────────────────────────────────────────────────┐
│                      Schema                              │
│  ┌─────────────────────────────────────────────────┐    │
│  │  nodes: {                                        │    │
│  │    doc: NodeType,    ──┐                         │    │
│  │    paragraph: NodeType ─┼──► 每个类型只有一个实例   │    │
│  │    text: NodeType,    ──┘                         │    │
│  │    ...                                           │    │
│  │  }                                               │    │
│  │  marks: {                                        │    │
│  │    em: MarkType,                                 │    │
│  │    strong: MarkType,                             │    │
│  │    link: MarkType                                │    │
│  │  }                                               │    │
│  └─────────────────────────────────────────────────┘    │
└──────────────────────┬──────────────────────────────────┘
                       │ type 指向
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │  Node A  │ │  Node B  │ │  Node C  │  多个 Node 实例
    │ type: ───┼─┤ type: ───┼─┤ type: ───┼─┼── 共享同一个
    │ paragraph│ │ paragraph│ │  text    │  │   NodeType
    │ attrs:{} │ │ attrs:{} │ │ text:"hi"│  │
    └──────────┘ └──────────┘ └──────────┘  │
                                            │
                                      Mark[] 数组
                                      附加在 Node 上
```

### 3.2 Node：文档树的基本单元

`Node` 是 ProseMirror 文档树的节点。文档本身就是一个顶级 `Node`（通常是 `doc` 类型），其内部嵌套子节点形成树形结构。

#### 核心源码分析

Node 类定义在 [node.ts#L22-L349](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/node.ts#L22-L349)：

```typescript
export class Node {
  constructor(
    readonly type: NodeType,      // 节点类型（单例）
    readonly attrs: Attrs,         // 属性对象
    content?: Fragment | null,     // 子节点集合
    readonly marks = Mark.none     // 附加的 marks
  ) {
    this.content = content || Fragment.empty
  }

  readonly content: Fragment
  readonly text: string | undefined  // 仅 TextNode 有值

  // 节点大小：叶子节点为 1，非叶子为 content.size + 2（开始/结束 token）
  get nodeSize(): number {
    return this.isLeaf ? 1 : 2 + this.content.size
  }
}
```

**关键设计决策——持久化数据结构（Persistent Data Structure）**：

Node 的注释明确说明（[node.ts#L10-L21](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/node.ts#L10-L21)）：

> Nodes are persistent data structures. Instead of changing them, you create new ones with the content you want. Old ones keep pointing at the old document shape. This is made cheaper by sharing structure between the old and new data as much as possible.

Node 提供了一系列"拷贝"方法，它们都返回**新的 Node 实例**，绝不修改自身：

```typescript
// [node.ts#L138-L141]
copy(content: Fragment | null = null): Node {
  if (content == this.content) return this  // 内容相同则返回自身
  return new Node(this.type, this.attrs, content, this.marks)
}

// [node.ts#L145-L147]
mark(marks: readonly Mark[]): Node {
  return marks == this.marks ? this : new Node(this.type, this.attrs, this.content, marks)
}

// [node.ts#L152-L155]
cut(from: number, to: number = this.content.size): Node {
  if (from == 0 && to == this.content.size) return this
  return this.copy(this.content.cut(from, to))
}
```

**位置索引系统**：ProseMirror 使用扁平的整数位置来定位文档中的点，而不是引用路径：
- 文本节点中：每个字符占一个位置
- 非叶子节点：开始 token 占 1 个位置，结束 token 占 1 个位置
- 叶子节点（非文本）：占 1 个位置

例如文档 `<doc><p>hello</p></doc>` 的位置：
```
位置:  0   1 2 3 4 5 6   7
       <doc> <p> h e l l o </p> </doc>
```

#### TextNode：文本节点的特殊化

`TextNode` 继承自 `Node`，表示文本内容（[node.ts#L353-L397](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/node.ts#L353-L397)）：

```typescript
export class TextNode extends Node {
  readonly text: string

  constructor(type: NodeType, attrs: Attrs, content: string, marks?: readonly Mark[]) {
    super(type, attrs, null, marks)  // 文本节点没有子 Fragment
    if (!content) throw new RangeError("Empty text nodes are not allowed")
    this.text = content
  }

  get nodeSize() { return this.text.length }  // 大小为字符数

  withText(text: string) {
    if (text == this.text) return this
    return new TextNode(this.type, this.attrs, text, this.marks)
  }
}
```

### 3.3 Mark：附加在内联内容上的标记

Mark 表示附加在节点上的"行内格式信息"，如加粗、斜体、链接等。Mark 不是独立的节点，而是**附着在节点上的标签**。

#### 核心源码分析

Mark 类定义在 [mark.ts#L10-L111](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/mark.ts#L10-L111)：

```typescript
export class Mark {
  constructor(
    readonly type: MarkType,   // Mark 类型（单例）
    readonly attrs: Attrs       // 属性（如链接的 href）
  ) {}

  // 将此 mark 添加到集合中，处理排除关系和排序
  addToSet(set: readonly Mark[]): readonly Mark[] {
    let copy, placed = false
    for (let i = 0; i < set.length; i++) {
      let other = set[i]
      if (this.eq(other)) return set                    // 已存在，返回原集合
      if (this.type.excludes(other.type)) {              // 互斥：移除对方
        if (!copy) copy = set.slice(0, i)
      } else if (other.type.excludes(this.type)) {       // 被对方排除：无法添加
        return set
      } else {
        if (!placed && other.type.rank > this.type.rank) {  // 按 rank 排序插入
          if (!copy) copy = set.slice(0, i)
          copy.push(this)
          placed = true
        }
        if (copy) copy.push(other)
      }
    }
    // ...
  }
}
```

**Mark 系统的核心特性**：

1. **有序集合**：每个 MarkType 有一个 `rank`（注册顺序决定），Mark 数组始终按 rank 排序（[mark.ts#L101-L107](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/mark.ts#L101-L107)）。这保证了相同的 mark 集合总是有相同的表示，可以用引用相等快速比较。

2. **互斥机制（excludes）**：MarkType 可以声明与其他 mark 互斥。例如，链接类型的 mark 可能会排除其他链接（一个位置不能同时有两个链接）。互斥关系在 Schema 构造时计算（[schema.ts#L621-L624](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/schema.ts#L621-L624)）。

3. **包含性（inclusive）**：MarkSpec 的 `inclusive` 选项（默认 true）控制光标在 mark 边界时是否"继承"该 mark。非 inclusive 的 mark（如链接）在光标移到其末尾时不会继续生效。这在 `ResolvedPos.marks()` 中实现（[resolvedpos.ts#L130-L152](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/resolvedpos.ts#L130-L152)）。

4. **Mark 与 Node 的关系**：Mark 存储在 Node 的 `marks` 数组中。每个内联节点（文本或内联叶子节点）都携带自己的 mark 集合。块级节点通常不携带 marks（除非配置允许）。

#### Node 与 Mark 的关系图

```
┌────────────────────────────────────────────────────────────┐
│  Node (paragraph)                                          │
│  type: paragraph, attrs: {}, marks: []                     │
│  content: Fragment [                                       │
│    ┌──────────────────────────────────────────────────┐    │
│    │ TextNode "Hello "  marks: [strong]                │    │
│    │ TextNode "world"  marks: [strong, em]             │    │
│    │ TextNode "!"      marks: []                       │    │
│    └──────────────────────────────────────────────────┘    │
│  ]                                                         │
└────────────────────────────────────────────────────────────┘

MarkType 注册顺序(rank): strong=0, em=1, link=2
Mark 集合按 rank 排序存储，保证集合的规范化表示
```

### 3.4 Fragment 与 ResolvedPos

#### Fragment：子节点的持久化集合

`Fragment` 是 Node 内部存储子节点的容器（[fragment.ts#L10-L261](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/fragment.ts#L10-L261)）：

```typescript
export class Fragment {
  readonly size: number
  readonly content: readonly Node[]

  // 相邻且 markup 相同的文本节点会被自动合并
  static fromArray(array: readonly Node[]) {
    // ... 合并逻辑
  }

  // 剪切操作返回新 Fragment，不修改原对象
  cut(from: number, to = this.size) {
    if (from == 0 && to == this.size) return this
    let result: Node[] = [], size = 0
    // ... 遍历 children，构建新数组
    return new Fragment(result, size)
  }

  static empty: Fragment = new Fragment([], 0)
}
```

**设计要点**：
- Fragment 同样是持久化的，所有修改操作返回新实例
- `Fragment.empty` 是全局共享的空实例
- `fromArray` 会自动合并相邻的同 markup 文本节点，保持文档规范化

#### ResolvedPos：位置的上下文解析

将一个扁平的数字位置"解析"为包含丰富上下文信息的对象（[resolvedpos.ts#L12-L250](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/resolvedpos.ts#L12-L250)）：

```typescript
export class ResolvedPos {
  depth: number        // 嵌套深度
  readonly pos: number // 原始位置
  readonly path: any[] // 路径：[node, index, startPos, ...] 三元组
  readonly parentOffset: number

  get parent() { return this.node(this.depth) }
  get doc() { return this.node(0) }

  node(depth?: number): Node { ... }
  index(depth?: number): number { ... }
  start(depth?: number): number { ... }
  end(depth?: number): number { ... }
  before(depth?: number): number { ... }
  after(depth?: number): number { ... }

  marks(): readonly Mark[] { ... }  // 获取该位置的有效 marks
}
```

`path` 数组的结构是每三个元素一组：`[node, childIndex, startPosition]`，从根节点到当前父节点。这种设计使得可以快速访问任意深度的祖先节点及其索引信息。

解析结果带有缓存（[resolvedpos.ts#L236-L249](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/resolvedpos.ts#L236-L249)），使用 WeakMap 按文档实例缓存最近解析的 12 个位置。

---

## 四、Transaction 事务机制与不可变性

ProseMirror 的状态管理是函数式和不可变思想的典范。所有状态变更都通过 Transaction 进行，生成全新的状态对象。

### 4.1 持久化数据结构

ProseMirror 的不可变性不是通过深拷贝实现的（那样性能太差），而是通过**结构共享（structural sharing）**的持久化数据结构：

```
文档树修改时的结构共享：

          旧文档树                          新文档树
             │                                │
          doc(Node)                        doc'(Node) ◄── 新对象
         /        \                       /        \
    p1(Node)    p2(Node)             p1(Node)    p2'(Node) ◄── 仅重建路径上的节点
    /    \        \                  /    \        \
  "a"   "b"     "c"               "a"   "b"     "c!" ◄── 新文本节点
   │      │        │                │      │        │
   └──────┴────────┴── 共享 ────────┴──────┘        （未修改的节点复用引用）
```

**核心机制**：
- Node/Fragment 的所有变更方法都返回新实例
- 如果新内容与旧内容相同（引用相等），直接返回 `this`
- 修改一个深层节点时，只需要重建从根到该节点路径上的节点，路径外的子树全部共享
- Mark 数组也是不可变的，添加/删除 mark 返回新数组

### 4.2 Step：原子变更单元

所有文档修改都被抽象为 `Step` 对象——一个可序列化、可反转、可重映射的原子操作。

Step 抽象类定义在 [step.ts#L16-L67](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/step.ts#L16-L67)：

```typescript
export abstract class Step {
  // 应用到文档，返回 StepResult（成功包含新 doc，失败包含错误信息）
  abstract apply(doc: Node): StepResult

  // 获取位置映射表
  getMap(): StepMap { return StepMap.empty }

  // 创建反向 Step（用于 undo）
  abstract invert(doc: Node): Step

  // 通过 mapping 重映射位置（用于 rebase/协同）
  abstract map(mapping: Mappable): Step | null

  // 尝试合并相邻 Step
  merge(other: Step): Step | null { return null }

  abstract toJSON(): any
  static fromJSON(schema: Schema, json: any): Step { ... }

  // 注册自定义 Step 类型
  static jsonID(id: string, stepClass: {...}) { ... }
}
```

**StepResult** 是应用结果的封装（[step.ts#L71-L97](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/step.ts#L71-L97)）：

```typescript
export class StepResult {
  constructor(
    readonly doc: Node | null,      // 成功时的新文档
    readonly failed: string | null  // 失败时的错误消息
  ) {}

  static ok(doc: Node) { return new StepResult(doc, null) }
  static fail(message: string) { return new StepResult(null, message) }
}
```

**内置 Step 类型**：

| Step 类 | 文件 | 作用 |
|---------|------|------|
| `ReplaceStep` | replace_step.ts | 替换文档范围 |
| `ReplaceAroundStep` | replace_step.ts | 围绕范围替换（用于结构化操作） |
| `AttrStep` | attr_step.ts | 修改节点属性 |
| `DocAttrStep` | attr_step.ts | 修改文档根节点属性 |
| `AddMarkStep` | mark_step.ts | 添加 mark 到范围 |
| `RemoveMarkStep` | mark_step.ts | 从范围移除 mark |
| `AddNodeMarkStep` | mark_step.ts | 给节点添加 mark |
| `RemoveNodeMarkStep` | mark_step.ts | 移除节点的 mark |

这种设计的优势：
1. **可审计**：每一步变更都是明确的数据对象，可以记录、回放、检查
2. **可撤销**：每个 Step 都能 `invert()` 生成逆操作，天然支持 undo
3. **可传输**：Step 可序列化为 JSON，是协同编辑的基础
4. **可重映射**：当文档因其他操作变化时，Step 的位置可以通过 Mapping 调整

### 4.3 StepMap 与位置映射

StepMap 记录了一个 Step 对文档位置造成的影响，用紧凑的三元组数组表示（[map.ts#L72-L164](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/map.ts#L72-L164)）：

```typescript
export class StepMap implements Mappable {
  // ranges: [start, oldSize, newSize, start, oldSize, newSize, ...]
  constructor(readonly ranges: readonly number[], readonly inverted = false) {}

  map(pos: number, assoc = 1): number {
    let diff = 0
    for (let i = 0; i < this.ranges.length; i += 3) {
      let start = this.ranges[i]
      let oldSize = this.ranges[i + 1]
      let newSize = this.ranges[i + 2]
      let end = start + oldSize
      if (pos <= end) {
        let side = !oldSize ? assoc : pos == start ? -1 : pos == end ? 1 : assoc
        return start + diff + (side < 0 ? 0 : newSize)
      }
      diff += newSize - oldSize
    }
    return pos + diff
  }
}
```

例如，在位置 5 删除 3 个字符、插入 2 个字符，对应 `[5, 3, 2]`：
- 位置 0-4：不变
- 位置 5-8（被删除范围）：映射到 5（assoc=1）或 5+2=7（assoc=-1，取决于关联方向）
- 位置 8+：偏移 -1（因为 3 字符被删，2 字符被插入）

**Mapping** 是多个 StepMap 的流水线（[map.ts#L172-L284](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/map.ts#L172-L284)），支持：
- 顺序映射位置通过多个 step
- **镜像（mirroring）**：记录互为逆操作的 step，在映射时跳过它们以避免位置信息丢失（这对协同编辑的 rebase 至关重要）
- `appendMap(map, mirrors?)` 添加映射并可指定镜像索引
- `invert()` 创建整体逆映射

### 4.4 Transform → Transaction 的继承链

#### Transform：纯文档转换构建器

`Transform` 是构建文档变更的基础类（[transform.ts#L28-L271](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/transform.ts#L28-L271)）：

```typescript
export class Transform {
  readonly steps: Step[] = []      // 所有已应用的步骤
  readonly docs: Node[] = []       // 每步之前的文档快照
  readonly mapping: Mapping = new Mapping()
  public doc: Node                 // 当前文档（每步后更新）

  get before() { return this.docs.length ? this.docs[0] : this.doc }

  step(step: Step) {
    let result = this.maybeStep(step)
    if (result.failed) throw new TransformError(result.failed)
    return this
  }

  maybeStep(step: Step) {
    let result = step.apply(this.doc)
    if (!result.failed) this.addStep(step, result.doc!)
    return result
  }

  // 内部：记录 step、保存旧 doc、更新 mapping、更新 doc
  addStep(step: Step, doc: Node) {
    this.docs.push(this.doc)
    this.steps.push(step)
    this.mapping.appendMap(step.getMap())
    this.doc = doc
  }
}
```

Transform 提供了丰富的链式 API：`replace`、`delete`、`insert`、`split`、`join`、`lift`、`wrap`、`setBlockType`、`addMark`、`removeMark` 等。每个方法内部构建相应的 Step 并调用 `this.step()`。

**关键：Transform 只关心文档（doc）的变更，不关心选区、marks 等编辑器状态。**

#### Transaction：编辑器状态事务

`Transaction` 继承自 `Transform`，在文档变更基础上增加了选区、存储 marks、元数据等编辑器状态（[transaction.ts#L42-L215](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/transaction.ts#L42-L215)）：

```typescript
export class Transaction extends Transform {
  time: number
  private curSelection: Selection
  private curSelectionFor = 0
  private updated = 0
  private meta: {[name: string]: any} = Object.create(null)
  storedMarks: readonly Mark[] | null

  constructor(state: EditorState) {
    super(state.doc)                           // 初始化 Transform
    this.time = Date.now()
    this.curSelection = state.selection       // 选区初始化为当前选区
    this.storedMarks = state.storedMarks
  }

  // 选区会自动通过 steps 的 mapping 进行映射
  get selection(): Selection {
    if (this.curSelectionFor < this.steps.length) {
      this.curSelection = this.curSelection.map(
        this.doc, this.mapping.slice(this.curSelectionFor)
      )
      this.curSelectionFor = this.steps.length
    }
    return this.curSelection
  }

  setSelection(selection: Selection): this {
    this.curSelection = selection
    this.curSelectionFor = this.steps.length
    this.updated = (this.updated | UPDATED_SEL) & ~UPDATED_MARKS
    this.storedMarks = null
    return this  // 链式调用
  }

  setStoredMarks(marks: readonly Mark[] | null): this { ... }
  setMeta(key: string | Plugin | PluginKey, value: any): this { ... }
  scrollIntoView(): this { ... }
}
```

**Transaction 的关键设计**：

1. **选区自动映射（lazy mapping）**：当向 Transaction 添加 step 后，选区不会立即重新映射，而是在下次读取 `.selection` 时，通过 `mapping.slice(curSelectionFor)` 增量映射。这避免了不必要的计算。

2. **元数据（metadata）系统**：`setMeta(key, value)` 允许插件或命令在事务中附加额外信息（如这是用户输入、粘贴、撤销等），其他插件可根据这些信息决定如何响应。

3. **更新标记（bitfield）**：使用位掩码 `UPDATED_SEL=1`、`UPDATED_MARKS=2`、`UPDATED_SCROLL=4` 高效跟踪哪些状态被显式修改。

4. **addStep 重写**：Transaction 重写了 `addStep`，在文档变更时自动清除 storedMarks（因为文档内容变化后旧的 marks 可能不再适用）。

### 4.5 EditorState.apply：状态更新流程

`EditorState.apply()` 是状态更新的入口（[state.ts#L118-L179](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/state.ts#L118-L179)）：

```typescript
export class EditorState {
  readonly config: Configuration
  declare doc: Node
  declare selection: Selection
  declare storedMarks: readonly Mark[] | null

  apply(tr: Transaction): EditorState {
    return this.applyTransaction(tr).state
  }

  applyTransaction(rootTr: Transaction): {state: EditorState, transactions: readonly Transaction[]} {
    // 1. 插件过滤事务
    if (!this.filterTransaction(rootTr)) return {state: this, transactions: []}

    let trs = [rootTr], newState = this.applyInner(rootTr), seen = null

    // 2. 插件追加事务循环（appendTransaction）
    for (;;) {
      let haveNew = false
      for (let i = 0; i < this.config.plugins.length; i++) {
        let plugin = this.config.plugins[i]
        if (plugin.spec.appendTransaction) {
          let tr = plugin.spec.appendTransaction.call(plugin, ...)
          if (tr && newState.filterTransaction(tr, i)) {
            trs.push(tr)
            newState = newState.applyInner(tr)
            haveNew = true
          }
        }
      }
      if (!haveNew) return {state: newState, transactions: trs}
    }
  }

  // 内部：为每个字段调用 apply，生成全新的 state 实例
  applyInner(tr: Transaction) {
    if (!tr.before.eq(this.doc))
      throw new RangeError("Applying a mismatched transaction")
    let newInstance = new EditorState(this.config)
    let fields = this.config.fields
    for (let i = 0; i < fields.length; i++) {
      let field = fields[i]
      newInstance[field.name] = field.apply(
        tr, this[field.name], this, newInstance
      )
    }
    return newInstance
  }
}
```

**状态字段系统**：EditorState 使用字段描述符（FieldDesc）管理状态，内置四个基础字段（[state.ts#L21-L41](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/state.ts#L21-L41)）：

```typescript
const baseFields = [
  new FieldDesc<Node>("doc", {
    init(config) { return config.doc || config.schema.topNodeType.createAndFill() },
    apply(tr) { return tr.doc }
  }),
  new FieldDesc<Selection>("selection", {
    init(config, instance) { return config.selection || Selection.atStart(instance.doc) },
    apply(tr) { return tr.selection }
  }),
  new FieldDesc<readonly Mark[] | null>("storedMarks", {
    init(config) { return config.storedMarks || null },
    apply(tr, _marks, _old, state) {
      return state.selection.$cursor ? tr.storedMarks : null
    }
  }),
  new FieldDesc<number>("scrollToSelection", {
    init() { return 0 },
    apply(tr, prev) { return tr.scrolledIntoView ? prev + 1 : prev }
  })
]
```

插件可以通过 `PluginSpec.state` 注册自己的状态字段，每个字段有 `init` 和 `apply` 方法，与内置字段完全平等地参与状态更新。

**不可变性保证**：
- `applyInner` 每次都创建 `new EditorState(this.config)`，绝不修改旧状态
- 每个字段的 `apply` 返回新值
- Configuration（包含 schema 和插件列表）在状态更新时共享，不重新创建

---

## 五、Selection 选区系统

ProseMirror 的选区系统是其最精巧的设计之一，支持多种选区类型、自动映射和序列化。

### 5.1 选区基类与 Range 模型

`Selection` 是所有选区的抽象基类（[selection.ts#L9-L188](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/selection.ts#L9-L188)）：

```typescript
export abstract class Selection {
  constructor(
    readonly $anchor: ResolvedPos,    // 锚点（选择时固定的一端）
    readonly $head: ResolvedPos,      // 头部（移动的一端）
    ranges?: readonly SelectionRange[]
  ) {
    this.ranges = ranges || [new SelectionRange($anchor.min($head), $anchor.max($head))]
  }

  ranges: readonly SelectionRange[]

  get anchor() { return this.$anchor.pos }
  get head() { return this.$head.pos }
  get from() { return this.$from.pos }
  get to() { return this.$to.pos }
  get $from() { return this.ranges[0].$from }
  get $to() { return this.ranges[0].$to }
  get empty(): boolean { ... }

  abstract eq(selection: Selection): boolean
  abstract map(doc: Node, mapping: Mappable): Selection
  abstract toJSON(): any
}
```

**anchor/head vs from/to**：
- `anchor` 和 `head` 模型用户的选择方向（anchor 是按下鼠标时的位置，head 是拖动到的位置）
- `from` 和 `to` 始终是范围的下界和上界（from ≤ to），不考虑方向
- 大多数情况下操作的是 `from`/`to`

`SelectionRange` 表示一个连续的选区范围（[selection.ts#L207-L215](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/selection.ts#L207-L215)），Selection 支持多个 ranges（为未来的多选区支持预留）。

### 5.2 三种内置选区类型

ProseMirror 内置三种选区类型，形成一个类层次：

```
Selection (abstract)
├── TextSelection      # 文本选区（光标是其空选特例）
├── NodeSelection      # 节点选区（选中整个原子/块节点）
└── AllSelection       # 全选（选中文档全部内容）
```

#### TextSelection

文本选区，两端都必须指向内联内容中（[selection.ts#L229-L305](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/selection.ts#L229-L305)）：

```typescript
export class TextSelection extends Selection {
  constructor($anchor: ResolvedPos, $head = $anchor) {
    checkTextSelection($anchor)
    checkTextSelection($head)
    super($anchor, $head)
  }

  // 空选区（光标）时返回 $head，否则 null
  get $cursor() {
    return this.$anchor.pos == this.$head.pos ? this.$head : null
  }
}
```

当 `anchor == head` 时，TextSelection 表示一个**光标（collapsed selection）**。`$cursor` 属性是判断是否为光标的便捷方式。

#### NodeSelection

节点选区，选中整个非文本节点（[selection.ts#L325-L377](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/selection.ts#L325-L377)）：

```typescript
export class NodeSelection extends Selection {
  node: Node

  constructor($pos: ResolvedPos) {
    let node = $pos.nodeAfter!
    let $end = $pos.node(0).resolve($pos.pos + node.nodeSize)
    super($pos, $end)
    this.node = node
  }

  content() {
    return new Slice(Fragment.from(this.node), 0, 0)
  }
}
```

NodeSelection 用于图片、表格、视频等原子节点的选择。其 `visible` 默认为 false（不显示原生浏览器选区，由 NodeView 自定义视觉表现）。

#### AllSelection

全选选区（[selection.ts#L399-L425](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/selection.ts#L399-L425)）：

```typescript
export class AllSelection extends Selection {
  constructor(doc: Node) {
    super(doc.resolve(0), doc.resolve(doc.content.size))
  }
}
```

AllSelection 用于处理文档开头/结尾有叶子块节点（如图片）时无法用 TextSelection 表示全选的情况。

#### 选区类型的注册与反序列化

每种选区类型通过 `Selection.jsonID()` 注册字符串 ID，支持 JSON 序列化/反序列化：

```typescript
Selection.jsonID("text", TextSelection)
Selection.jsonID("node", NodeSelection)
Selection.jsonID("all", AllSelection)
```

这使得选区可以随状态一起序列化保存，并在协同编辑中传输。

### 5.3 选区映射（Mapping）机制

当文档被 Transaction 修改后，选区需要相应调整。每个 Selection 子类实现 `map()` 方法：

```typescript
// TextSelection.map [selection.ts#L241-L246]
map(doc: Node, mapping: Mappable): Selection {
  let $head = doc.resolve(mapping.map(this.head))
  if (!$head.parent.inlineContent) return Selection.near($head)
  let $anchor = doc.resolve(mapping.map(this.anchor))
  return new TextSelection(
    $anchor.parent.inlineContent ? $anchor : $head, $head
  )
}

// NodeSelection.map [selection.ts#L338-L343]
map(doc: Node, mapping: Mappable): Selection {
  let {deleted, pos} = mapping.mapResult(this.anchor)
  let $pos = doc.resolve(pos)
  if (deleted) return Selection.near($pos)  // 节点被删除，退化为近邻选区
  return new NodeSelection($pos)
}
```

**映射的容错性**：
- 如果映射后的位置不再是有效的文本位置，使用 `Selection.near($pos)` 寻找最近的有效光标位置
- 如果 NodeSelection 选中的节点被删除，退化为最近的文本选区
- `Selection.findFrom($pos, dir, textOnly)` 实现了向上搜索父节点、在不同深度寻找有效选区位置的算法（[selection.ts#L118-L130](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/selection.ts#L118-L130)）

### 5.4 Bookmark：文档无关的选区快照

`SelectionBookmark` 是一个轻量级、文档无关的选区表示，主要用于历史记录（undo/redo）中保存和恢复选区（[selection.ts#L195-L204](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/selection.ts#L195-L204)）：

```typescript
export interface SelectionBookmark {
  map: (mapping: Mappable) => SelectionBookmark
  resolve: (doc: Node) => Selection
}
```

Bookmark 只存储位置数字，不持有 ResolvedPos 或文档引用。它可以通过 mapping 调整位置，并在需要时 resolve 回真实的 Selection。

每种选区类型都有对应的 Bookmark 实现（如 `TextBookmark`、`NodeBookmark`、`AllBookmark`），`getBookmark()` 方法返回当前选区的快照。

---

## 六、Plugin 插件系统

Plugin 是 ProseMirror 扩展机制的核心，它允许将功能打包为可复用的模块（[plugin.ts#L71-L89](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/plugin.ts#L71-L89)）：

```typescript
export class Plugin<PluginState = any> {
  constructor(readonly spec: PluginSpec<PluginState>) {
    if (spec.props) bindProps(spec.props, this, this.props)
    this.key = spec.key ? spec.key.key : createKey("plugin")
  }

  readonly props: EditorProps = {}
  key: string

  getState(state: EditorState): PluginState | undefined {
    return state[this.key]
  }
}
```

PluginSpec 支持的能力（[plugin.ts#L7-L45](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/plugin.ts#L7-L45)）：

| 配置项 | 作用 |
|--------|------|
| `props` | 注入编辑器 props（事件处理、DOM 序列化等） |
| `state` | 定义插件自有的状态字段（init/apply/toJSON/fromJSON） |
| `key` | PluginKey，用于唯一标识和检索插件 |
| `view` | 返回 PluginView，在视图创建时初始化（可访问 DOM） |
| `filterTransaction` | 在事务应用前过滤（可返回 false 取消） |
| `appendTransaction` | 在事务应用后追加新事务（实现自动修正/连锁响应） |

**StateField 接口**：

```typescript
export interface StateField<T> {
  init: (config: EditorStateConfig, instance: EditorState) => T
  apply: (tr: Transaction, value: T, oldState: EditorState, newState: EditorState) => T
  toJSON?: (value: T) => any
  fromJSON?: (config: EditorStateConfig, value: any, state: EditorState) => T
}
```

这种设计让每个插件都像编辑器状态的"一等公民"，拥有自己的不可变状态，随事务更新而更新。

---

## 七、ProseMirror vs Slate vs Quill 架构对比

| 维度 | ProseMirror | Slate | Quill |
|------|-------------|-------|-------|
| **架构风格** | 严格分层、模块化、函数式 | 单体核心 + 插件架构、面向对象 | 自包含编辑器、基于 Delta |
| **文档模型** | 强 Schema 的树形结构，Node + Mark 分离 | 树形 Model（Element/Text），Schema 可选 | 线性 Delta 格式（OT），扁平行模型 |
| **Schema** | **必须**定义，强类型约束，内容表达式编译为自动机 | 可选，运行时不强制校验，由插件保证 | 内置格式白名单，通过 register 扩展 |
| **状态管理** | 不可变 EditorState，通过 Transaction 更新，结构共享 | 不可变 Value，通过 Transforms/Operations 更新 | 可变内部状态，通过 Delta API 修改，事件驱动 |
| **变更表示** | Step 对象数组（ReplaceStep、AttrStep 等），可序列化、可反转、可 rebase | Operation 对象数组（insert_text、split_node 等） | Delta 操作（retain/insert/delete），基于 OT |
| **位置模型** | 扁平整数位置 + ResolvedPos 上下文解析，精确到 token | Path 数组（如 `[0, 1, 2]`）+ offset，层级路径 | Index + length，线性偏移 |
| **选区模型** | Selection 类层次（Text/Node/All），anchor/head，自动 mapping | Range 接口（anchor/focus），point path+offset | Range（index/length），原生 Selection 封装 |
| **视图层** | prosemirror-view 独立包，DOM 差异化更新，NodeView 自定义渲染 | React/Vue/Solid 等框架自定义渲染（slate-react 等） | 自身管理 DOM/contenteditable，主题系统 |
| **协同编辑** | 原生支持（Step + Mapping + rebase），prosemirror-collab 包 | 支持但需外部实现（slate-history 等） | 原生支持（基于 OT），核心设计目标 |
| **历史/撤销** | prosemirror-history（基于 Step invert） | slate-history（基于 operation 快照/反转） | 内置 history 模块 |
| **扩展机制** | Plugin 系统（state field + props + filter/appendTransaction） | Plugin 接口（renderElement、onChange 等钩子） | Module 系统（注册格式、模块、主题） |
| **学习曲线** | 陡峭（概念多、显式 Schema、函数式风格） | 中等（React 友好，文档相对清晰） | 平缓（API 简洁，开箱即用） |
| **灵活性** | 极高（自定义 Schema、Step、Selection、NodeView） | 高（但受限于树模型和框架渲染） | 中低（Delta 模型固定，定制复杂节点困难） |
| **适用场景** | 复杂结构化编辑（表格、嵌套块、协同）、需要强约束 | 中等到复杂编辑器、React 技术栈 | 富文本评论、简单博客、快速集成 |

### 核心设计哲学差异

**ProseMirror："正确性优先"**
- Schema 是强制性的，文档结构始终被验证
- 所有变更通过显式的 Step 对象，完全可追溯
- 不可变状态 + 结构化共享，兼顾正确性和性能
- 分层解耦，model/transform/state/view 可独立使用

**Slate："React 原生"**
- 文档模型更接近 React 的心智模型（组件树）
- 不强制 Schema，给予更多自由但也意味着需要自己保证合法性
- 视图层交给前端框架，与 React 生态无缝集成
- 变更操作更接近"命令式"的 Transform 调用

**Quill："简单够用"**
- Delta 模型简洁优雅，适合线性富文本
- 开箱即用，API 设计简洁
- 但 Delta 的扁平/行模型难以表达复杂嵌套结构（如表格内嵌套列表内嵌套代码块）
- 内部状态可变，通过事件通知外部

---

## 八、关键源码索引

### prosemirror-model

| 文件 | 核心内容 |
|------|---------|
| [schema.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/schema.ts) | Schema、NodeType、MarkType、NodeSpec、MarkSpec、Attribute |
| [node.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/node.ts) | Node、TextNode（持久化节点、copy/mark/cut/eq） |
| [mark.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/mark.ts) | Mark（addToSet/removeFromSet、排序、互斥） |
| [fragment.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/fragment.ts) | Fragment（子节点集合、文本合并、cut/append） |
| [resolvedpos.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/resolvedpos.ts) | ResolvedPos（位置解析、path 结构、marks()）、NodeRange |
| [content.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/content.ts) | ContentMatch（内容表达式自动机） |
| [replace.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-model/src/replace.ts) | Slice、replace 算法 |

### prosemirror-transform

| 文件 | 核心内容 |
|------|---------|
| [transform.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/transform.ts) | Transform（step/maybeStep/addStep、链式 API） |
| [step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/step.ts) | Step 抽象类、StepResult、jsonID 注册 |
| [map.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/map.ts) | StepMap、Mapping、MapResult（位置映射与镜像） |
| [replace_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/replace_step.ts) | ReplaceStep、ReplaceAroundStep |
| [attr_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/attr_step.ts) | AttrStep、DocAttrStep |
| [mark_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/mark_step.ts) | AddMarkStep、RemoveMarkStep、AddNodeMarkStep、RemoveNodeMarkStep |
| [structure.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-transform/src/structure.ts) | lift、wrap、split、join、setBlockType 等结构操作 |

### prosemirror-state

| 文件 | 核心内容 |
|------|---------|
| [state.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/state.ts) | EditorState、FieldDesc、Configuration、applyTransaction |
| [transaction.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/transaction.ts) | Transaction（selection 映射、storedMarks、meta、scrollIntoView） |
| [selection.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/selection.ts) | Selection、TextSelection、NodeSelection、AllSelection、SelectionBookmark |
| [plugin.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/node_modules/prosemirror-state/src/plugin.ts) | Plugin、PluginSpec、StateField、PluginKey |

---

## 总结

ProseMirror 的架构设计体现了几个核心原则：

1. **不可变性与持久化数据结构**：所有状态对象（Node、Fragment、Mark[]、EditorState）都是不可变的，修改产生新对象并通过结构共享保证性能。这使得状态变化可预测、可追踪、易于调试。

2. **显式的变更表示**：所有文档修改都封装为 Step 对象，而非直接突变。这带来了 undo/redo、协同编辑、变更审计等能力的"免费"支持。

3. **强 Schema 约束**：Schema 不仅定义了文档可以长什么样，还在运行时通过 ContentMatch 自动机验证文档合法性，从源头避免无效文档状态。

4. **分层与解耦**：model 不依赖任何其他包，transform 只依赖 model，state 在二者之上添加编辑器概念，view 负责 DOM 渲染。每一层都可以独立使用和测试。

5. **位置映射系统**：StepMap/Mapping 设计精巧，支持位置在多次变更间的映射和镜像优化，是协同编辑 rebase 的技术基础。

6. **插件即一等公民**：Plugin 可以拥有自己的状态字段，与内置字段完全平等地参与事务更新流程，filterTransaction 和 appendTransaction 钩子赋予插件强大的干预能力。

这些设计使得 ProseMirror 在面对复杂的富文本编辑需求（表格、嵌套结构、实时协同、自定义节点）时，依然能保持架构的清晰和可扩展性。
