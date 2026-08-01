# 文档模型（Document Model）

> 对应本地源码目录：[model/src/](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src)
>
> 核心文件：[schema.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/schema.ts)、[node.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/node.ts)、[mark.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/mark.ts)、[fragment.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/fragment.ts)、[resolvedpos.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/resolvedpos.ts)

![文档模型类关系图](../docs/architecture-document-model.svg)

ProseMirror 的文档模型是整个框架的基石。它定义了文档的结构规则与数据表示，具有强 Schema 约束、持久化不可变、Node/Mark 分离三大特征。

---

## 1. Schema：文档的"宪法"

`Schema` 规定了一个文档中允许出现哪些节点类型（NodeType）和标记类型（MarkType），以及它们之间的嵌套规则。每个编辑器实例持有一个 Schema 实例。

### 核心源码

Schema 构造函数位于 [schema.ts#L595-L630](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/schema.ts#L595-L630)：

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
    // 1. 将 nodes/marks 转为 OrderedMap（保持注册顺序）
    instanceSpec.nodes = OrderedMap.from(spec.nodes)
    instanceSpec.marks = OrderedMap.from(spec.marks || {})

    // 2. 编译 NodeType / MarkType（每个类型在 Schema 中只有一个实例）
    this.nodes = NodeType.compile(this.spec.nodes, this)
    this.marks = MarkType.compile(this.spec.marks, this)

    // 3. 为每个 NodeType 解析内容表达式与 mark 允许集
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
    // 4. 计算 MarkType 之间的排除关系
    for (let prop in this.marks) {
      let type = this.marks[prop], excl = type.spec.excludes
      type.excluded = excl == null ? [type] : excl == "" ? [] : gatherMarks(this, excl.split(" "))
    }
  }
}
```

### 设计要点

1. **类型对象单例化**：[NodeType.compile](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/schema.ts#L235-L245) 为每个节点名创建唯一的 NodeType 实例，所有同类型 Node 共享它。

2. **内容表达式（Content Expression）**：使用类似正则表达式的语法描述子节点序列，例如 `"paragraph+"`、`"(paragraph | blockquote)*"`、`"heading paragraph+"`。表达式被 [ContentMatch.parse](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/content.ts) 编译成有限状态自动机，用于在创建/替换节点时验证合法性。

3. **NodeSpec 关键配置**（[schema.ts#L371-L490](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/schema.ts#L371-L490)）：
   - `content`：内容表达式
   - `marks`：允许的 mark 集合
   - `group`：所属分组（可被内容表达式引用）
   - `inline` / `atom` / `selectable` / `draggable`：行为标记
   - `defining` / `isolating`：边界语义（决定替换操作时节点是否保留）
   - `attrs`：属性定义
   - `toDOM` / `parseDOM`：DOM 序列化与解析规则

---

## 2. Node：文档树的基本单元

文档本身是一个顶级 `Node`（通常是 `doc` 类型），内部嵌套子节点形成树形结构。

### 核心源码

Node 类定义于 [node.ts#L22-L349](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/node.ts#L22-L349)：

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
  readonly text: string | undefined

  // 非叶子节点：content.size + 2（开始/结束 token）；叶子节点：1；文本节点：字符数
  get nodeSize(): number {
    return this.isLeaf ? 1 : 2 + this.content.size
  }
}
```

### 持久化数据结构

Node 的文档注释明确说明（[node.ts#L10-L21](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/node.ts#L10-L21)）：

> Nodes are persistent data structures. Instead of changing them, you create new ones with the content you want. Old ones keep pointing at the old document shape. This is made cheaper by sharing structure between the old and new data as much as possible.

所有变更方法都返回新实例，不修改自身，并在内容相同时复用引用（结构共享）：

```typescript
// [node.ts#L138-L141]
copy(content: Fragment | null = null): Node {
  if (content == this.content) return this
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

修改一个深层节点时，只需重建从根到该节点路径上的节点，路径外的子树全部共享引用。

### 位置索引系统

ProseMirror 使用扁平整数位置定位文档点：
- 文本节点中每个字符占 1 个位置
- 非叶子节点的开始/结束 token 各占 1 个位置
- 非文本叶子节点占 1 个位置

文档 `<doc><p>hello</p></doc>` 的位置：

```
位置:  0   1 2 3 4 5 6   7
       <doc> <p> h e l l o </p> </doc>
```

### TextNode

[TextNode](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/node.ts#L353-L397) 继承自 Node，表示文本内容，没有子 Fragment：

```typescript
export class TextNode extends Node {
  readonly text: string
  constructor(type: NodeType, attrs: Attrs, content: string, marks?: readonly Mark[]) {
    super(type, attrs, null, marks)
    if (!content) throw new RangeError("Empty text nodes are not allowed")
    this.text = content
  }
  get nodeSize() { return this.text.length }
  withText(text: string) {
    if (text == this.text) return this
    return new TextNode(this.type, this.attrs, text, this.marks)
  }
}
```

---

## 3. Mark：附加在内联内容上的标记

Mark 表示加粗、斜体、链接等行内格式。它不是独立节点，而是附着在节点上的标签。

### 核心源码

[mark.ts#L10-L111](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/mark.ts#L10-L111)：

```typescript
export class Mark {
  constructor(
    readonly type: MarkType,
    readonly attrs: Attrs
  ) {}

  addToSet(set: readonly Mark[]): readonly Mark[] {
    let copy, placed = false
    for (let i = 0; i < set.length; i++) {
      let other = set[i]
      if (this.eq(other)) return set                    // 已存在，返回原集合
      if (this.type.excludes(other.type)) {              // 互斥：移除对方
        if (!copy) copy = set.slice(0, i)
      } else if (other.type.excludes(this.type)) {       // 被对方排除：无法加入
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

### 关键特性

1. **有序集合与规范化**：每个 MarkType 在 Schema 中有一个 `rank`（注册顺序决定）。Mark 数组始终按 rank 排序（[Mark.setFrom](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/mark.ts#L101-L107)），保证相同 mark 集合的表示一致，便于引用比较。

2. **互斥机制（excludes）**：MarkType 可声明与其他 mark 互斥（如同一位置不能有两个链接）。互斥关系在 Schema 构造时计算（[schema.ts#L621-L624](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/schema.ts#L621-L624)）。

3. **包含性（inclusive）**：MarkSpec 的 `inclusive`（默认 true）控制光标在 mark 边界处是否继承该 mark。非 inclusive 的 mark（如链接）在光标移到其末尾时不再延续。逻辑见 [ResolvedPos.marks](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/resolvedpos.ts#L130-L152)。

4. **与 Node 的关系**：Mark 存储在 Node 的 `marks` 数组中，每个内联节点携带自己的 mark 集合；块级节点默认不带 marks。

---

## 4. Fragment：子节点的持久化集合

[Fragment](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/fragment.ts#L10-L261) 是 Node 内部存储子节点的容器：

```typescript
export class Fragment {
  readonly size: number
  readonly content: readonly Node[]

  // 相邻且 markup 相同的文本节点自动合并
  static fromArray(array: readonly Node[]) { ... }

  cut(from: number, to = this.size) {
    if (from == 0 && to == this.size) return this
    let result: Node[] = [], size = 0
    // 遍历 children，构建新数组
    return new Fragment(result, size)
  }

  static empty: Fragment = new Fragment([], 0)
}
```

设计要点：
- Fragment 同样持久化，所有修改返回新实例
- `Fragment.empty` 全局共享，避免为每个空节点分配对象
- [fromArray](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/fragment.ts#L227-L242) 自动合并相邻同 markup 文本节点，保持文档规范化

---

## 5. ResolvedPos：位置的上下文解析

将扁平整数位置"解析"为包含祖先链、索引、偏移等丰富上下文的对象，定义于 [resolvedpos.ts#L12-L250](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/resolvedpos.ts#L12-L250)：

```typescript
export class ResolvedPos {
  depth: number
  readonly pos: number
  readonly path: any[]        // [node, index, startPos, node, index, startPos, ...]
  readonly parentOffset: number

  get parent() { return this.node(this.depth) }
  get doc() { return this.node(0) }
  node(depth?) { ... }
  index(depth?) { ... }
  start(depth?) { ... }
  end(depth?) { ... }
  marks(): readonly Mark[] { ... }
}
```

`path` 数组每三个元素为一组 `[node, childIndex, startPosition]`，从根节点到当前父节点，支持 O(depth) 访问任意祖先。

解析结果有缓存（[resolvedpos.ts#L236-L249](file:///Users/tog/Desktop/code/gsb/gsb-731/4_Tony/model/src/resolvedpos.ts#L236-L249)），使用 WeakMap 按文档实例缓存最近解析的 12 个位置，避免重复遍历树。

> 返回 [README](../README.md)
