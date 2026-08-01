# 文档模型（Document Model）

> 对应本地源码目录：[../model/src/](../model/src)
>
> 核心文件：[schema.ts](../model/src/schema.ts)、[node.ts](../model/src/node.ts)、[mark.ts](../model/src/mark.ts)、[fragment.ts](../model/src/fragment.ts)、[resolvedpos.ts](../model/src/resolvedpos.ts)

![文档模型类关系图](architecture-document-model.svg)

ProseMirror 的文档模型是整个框架的基石。它定义了文档的结构规则与数据表示，具有强 Schema 约束、持久化不可变、Node/Mark 分离三大特征。

---

## 1. Schema：文档的"宪法"

`Schema` 规定了一个文档中允许出现哪些节点类型（NodeType）和标记类型（MarkType），以及它们之间的嵌套规则。每个编辑器实例持有一个 Schema 实例。

### 核心源码

Schema 构造函数位于 [schema.ts#L595-L630](../model/src/schema.ts#L595-L630)：

```typescript
  constructor(spec: SchemaSpec<Nodes, Marks>) {
    let instanceSpec = this.spec = {} as any
    for (let prop in spec) instanceSpec[prop] = (spec as any)[prop]
    instanceSpec.nodes = OrderedMap.from(spec.nodes),
    instanceSpec.marks = OrderedMap.from(spec.marks || {}),

    this.nodes = NodeType.compile(this.spec.nodes, this)
    this.marks = MarkType.compile(this.spec.marks, this)

    let contentExprCache = Object.create(null)
    for (let prop in this.nodes) {
      if (prop in this.marks)
        throw new RangeError(prop + " can not be both a node and a mark")
      let type = this.nodes[prop], contentExpr = type.spec.content || "", markExpr = type.spec.marks
      type.contentMatch = contentExprCache[contentExpr] ||
        (contentExprCache[contentExpr] = ContentMatch.parse(contentExpr, this.nodes))
      ;(type as any).inlineContent = type.contentMatch.inlineContent
      if (type.spec.linebreakReplacement) {
        if (this.linebreakReplacement) throw new RangeError("Multiple linebreak nodes defined")
        if (!type.isInline || !type.isLeaf) throw new RangeError("Linebreak replacement nodes must be inline leaf nodes")
        this.linebreakReplacement = type
      }
      type.markSet = markExpr == "_" ? null :
        markExpr ? gatherMarks(this, markExpr.split(" ")) :
        markExpr == "" || !type.inlineContent ? [] : null
    }
    for (let prop in this.marks) {
      let type = this.marks[prop], excl = type.spec.excludes
      type.excluded = excl == null ? [type] : excl == "" ? [] : gatherMarks(this, excl.split(" "))
    }

    this.nodeFromJSON = json => Node.fromJSON(this, json)
    this.markFromJSON = json => Mark.fromJSON(this, json)
    this.topNodeType = this.nodes[this.spec.topNode || "doc"]
    this.cached.wrappings = Object.create(null)
  }
```

### 设计要点

1. **类型对象单例化**：[NodeType.compile](../model/src/schema.ts#L234-L245) 为每个节点名创建唯一的 NodeType 实例，所有同类型 Node 共享它：

```typescript
  /// @internal
  static compile<Nodes extends string>(nodes: OrderedMap<NodeSpec>, schema: Schema<Nodes>): {readonly [name in Nodes]: NodeType} {
    let result = Object.create(null)
    nodes.forEach((name, spec) => result[name] = new NodeType(name, schema, spec))

    let topType = schema.spec.topNode || "doc"
    if (!result[topType]) throw new RangeError("Schema is missing its top node type ('" + topType + "')")
    if (!result.text) throw new RangeError("Every schema needs a 'text' type")
    for (let _ in result.text.attrs) throw new RangeError("The text node type should not have attributes")

    return result
  }
```

2. **内容表达式（Content Expression）**：使用类似正则表达式的语法描述子节点序列，例如 `"paragraph+"`、`"(paragraph | blockquote)*"`、`"heading paragraph+"`。表达式被 [ContentMatch.parse](../model/src/content.ts) 编译成有限状态自动机，用于在创建/替换节点时验证合法性。

3. **NodeSpec 关键配置**（[schema.ts#L370-L490](../model/src/schema.ts#L370-L490)）：
   - `content`（[L375](../model/src/schema.ts#L375)）：内容表达式
   - `marks`（[L382](../model/src/schema.ts#L382)）：允许的 mark 集合
   - `group`（[L387](../model/src/schema.ts#L387)）：所属分组（可被内容表达式引用）
   - `inline`（[L390](../model/src/schema.ts#L390)）/ `atom`（[L395](../model/src/schema.ts#L395)）：行为标记
   - `attrs`（[L398](../model/src/schema.ts#L398)）：属性定义
   - `defining`（[L438](../model/src/schema.ts#L438)）/ `isolating`（[L444](../model/src/schema.ts#L444)）：边界语义（决定替换操作时节点是否保留）
   - `toDOM`（[L458](../model/src/schema.ts#L458)）/ `parseDOM`（[L466](../model/src/schema.ts#L466)）：DOM 序列化与解析规则

---

## 2. Node：文档树的基本单元

文档本身是一个顶级 `Node`（通常是 `doc` 类型），内部嵌套子节点形成树形结构。

### 核心源码

Node 类定义于 [node.ts#L22-L38](../model/src/node.ts#L22-L38)，其文档注释明确阐述了持久化设计原则：

```typescript
/// This class represents a node in the tree that makes up a
/// ProseMirror document. So a document is an instance of `Node`, with
/// children that are also instances of `Node`.
///
/// Nodes are persistent data structures. Instead of changing them, you
/// create new ones with the content you want. Old ones keep pointing
/// at the old document shape. This is made cheaper by sharing
/// structure between the old and new data as much as possible, which a
/// tree shape like this (without back pointers) makes easy.
///
/// **Do not** directly mutate the properties of a `Node` object. See
/// [the guide](/docs/guide/#doc) for more information.
export class Node {
  /// @internal
  constructor(
    /// The type of node that this is.
    readonly type: NodeType,
    /// An object mapping attribute names to values. The kind of
    /// attributes allowed and required are
    /// [determined](#model.NodeSpec.attrs) by the node type.
    readonly attrs: Attrs,
    // A fragment holding the node's children.
    content?: Fragment | null,
    /// The marks (things like whether it is emphasized or part of a
    /// link) applied to this node.
    readonly marks = Mark.none
  ) {
    this.content = content || Fragment.empty
  }
```

### nodeSize 位置索引系统

[node.ts#L49-L54](../model/src/node.ts#L49-L54) 定义了节点大小计算规则：

```typescript
  /// The size of this node, as defined by the integer-based [indexing
  /// scheme](/docs/guide/#doc.indexing). For text nodes, this is the
  /// amount of characters. For other leaf nodes, it is one. For
  /// non-leaf nodes, it is the size of the content plus two (the
  /// start and end token).
  get nodeSize(): number { return this.isLeaf ? 1 : 2 + this.content.size }
```

ProseMirror 使用扁平整数位置定位文档点：
- 文本节点中每个字符占 1 个位置
- 非叶子节点的开始/结束 token 各占 1 个位置
- 非文本叶子节点占 1 个位置

文档 `<doc><p>hello</p></doc>` 的位置：

```
位置:  0   1 2 3 4 5 6   7
       <doc> <p> h e l l o </p> </doc>
```

### 持久化数据结构

所有变更方法都返回新实例，不修改自身，并在内容相同时复用引用（结构共享）：

[node.ts#L136-L155](../model/src/node.ts#L136-L155)：

```typescript
  /// Create a new node with the same markup as this node, containing
  /// the given content (or empty, if no content is given).
  copy(content: Fragment | null = null): Node {
    if (content == this.content) return this
    return new Node(this.type, this.attrs, content, this.marks)
  }

  /// Create a copy of this node, with the given set of marks instead
  /// of the node's own marks.
  mark(marks: readonly Mark[]): Node {
    return marks == this.marks ? this : new Node(this.type, this.attrs, this.content, marks)
  }

  /// Create a copy of this node with only the content between the
  /// given positions. If `to` is not given, it defaults to the end of
  /// the node.
  cut(from: number, to: number = this.content.size): Node {
    if (from == 0 && to == this.content.size) return this
    return this.copy(this.content.cut(from, to))
  }
```

修改一个深层节点时，只需重建从根到该节点路径上的节点，路径外的子树全部共享引用。

### TextNode

[TextNode](../model/src/node.ts#L353-L397) 继承自 Node，表示文本内容，没有子 Fragment：

```typescript
export class TextNode extends Node {
  readonly text: string

  /// @internal
  constructor(type: NodeType, attrs: Attrs, content: string, marks?: readonly Mark[]) {
    super(type, attrs, null, marks)
    if (!content) throw new RangeError("Empty text nodes are not allowed")
    this.text = content
  }

  toString() {
    if (this.type.spec.toDebugString) return this.type.spec.toDebugString(this)
    return wrapMarks(this.marks, JSON.stringify(this.text))
  }

  get textContent() { return this.text }

  textBetween(from: number, to: number) { return this.text.slice(from, to) }

  get nodeSize() { return this.text.length }

  mark(marks: readonly Mark[]) {
    return marks == this.marks ? this : new TextNode(this.type, this.attrs, this.text, marks)
  }

  withText(text: string) {
    if (text == this.text) return this
    return new TextNode(this.type, this.attrs, text, this.marks)
  }

  cut(from = 0, to = this.text.length) {
    if (from == 0 && to == this.text.length) return this
    return this.withText(this.text.slice(from, to))
  }

  eq(other: Node) {
    return this.sameMarkup(other) && this.text == other.text
  }

  toJSON() {
    let base = super.toJSON()
    base.text = this.text
    return base
  }
}
```

---

## 3. Mark：附加在内联内容上的标记

Mark 表示加粗、斜体、链接等行内格式。它不是独立节点，而是附着在节点上的标签。

### 核心源码

Mark.addToSet 位于 [mark.ts#L19-L45](../model/src/mark.ts#L19-L45)，实现了排序插入与互斥处理：

```typescript
  /// Given a set of marks, create a new set which contains this one as
  /// well, in the right position. If this mark is already in the set,
  /// the set itself is returned. If any marks that are set to be
  /// [exclusive](#model.MarkSpec.excludes) with this mark are present,
  /// those are replaced by this one.
  addToSet(set: readonly Mark[]): readonly Mark[] {
    let copy, placed = false
    for (let i = 0; i < set.length; i++) {
      let other = set[i]
      if (this.eq(other)) return set
      if (this.type.excludes(other.type)) {
        if (!copy) copy = set.slice(0, i)
      } else if (other.type.excludes(this.type)) {
        return set
      } else {
        if (!placed && other.type.rank > this.type.rank) {
          if (!copy) copy = set.slice(0, i)
          copy.push(this)
          placed = true
        }
        if (copy) copy.push(other)
      }
    }
    if (!copy) copy = set.slice()
    if (!placed) copy.push(this)
    return copy
  }
```

### 关键特性

1. **有序集合与规范化**：每个 MarkType 在 Schema 中有一个 `rank`（注册顺序决定）。Mark 数组始终按 rank 排序。[Mark.setFrom](../model/src/mark.ts#L99-L107) 在创建集合时排序：

```typescript
  /// Create a properly sorted mark set from null, a single mark, or an
  /// unsorted array of marks.
  static setFrom(marks?: Mark | readonly Mark[] | null): readonly Mark[] {
    if (!marks || Array.isArray(marks) && marks.length == 0) return Mark.none
    if (marks instanceof Mark) return [marks]
    let copy = marks.slice()
    copy.sort((a, b) => a.type.rank - b.type.rank)
    return copy
  }
```

这保证相同 mark 集合的表示一致，便于引用比较。空集合复用全局单例 [Mark.none](../model/src/mark.ts#L109-L110)：

```typescript
  /// The empty set of marks.
  static none: readonly Mark[] = []
```

2. **互斥机制（excludes）**：MarkType 可声明与其他 mark 互斥（如同一位置不能有两个链接）。互斥关系在 Schema 构造时计算（[schema.ts#L621-L624](../model/src/schema.ts#L621-L624)）。MarkSpec 的 `excludes` 字段定义见 [schema.ts#L502-L515](../model/src/schema.ts#L502-L515)。

3. **包含性（inclusive）**：MarkSpec 的 `inclusive`（默认 true）控制光标在 mark 边界处是否继承该 mark，字段定义见 [schema.ts#L497-L500](../model/src/schema.ts#L497-L500)。非 inclusive 的 mark（如链接）在光标移到其末尾时不再延续，逻辑见 [ResolvedPos.marks](../model/src/resolvedpos.ts#L126-L152)：

```typescript
  /// Get the marks at this position, factoring in the surrounding
  /// marks' [`inclusive`](#model.MarkSpec.inclusive) property. If the
  /// position is at the start of a non-empty node, the marks of the
  /// node after it (if any) are returned.
  marks(): readonly Mark[] {
    let parent = this.parent, index = this.index()

    // In an empty parent, return the empty array
    if (parent.content.size == 0) return Mark.none

    // When inside a text node, just return the text node's marks
    if (this.textOffset) return parent.child(index).marks

    let main = parent.maybeChild(index - 1), other = parent.maybeChild(index)
    // If the `after` flag is true of there is no node before, make
    // the node after this position the main reference.
    if (!main) { let tmp = main; main = other; other = tmp }

    // Use all marks in the main node, except those that have
    // `inclusive` set to false and are not present in the other node.
    let marks = main!.marks
    for (var i = 0; i < marks.length; i++)
      if (marks[i].type.spec.inclusive === false && (!other || !marks[i].isInSet(other.marks)))
        marks = marks[i--].removeFromSet(marks)

    return marks
  }
```

4. **与 Node 的关系**：Mark 存储在 Node 的 `marks` 数组中，每个内联节点携带自己的 mark 集合；块级节点默认不带 marks。

---

## 4. Fragment：子节点的持久化集合

[Fragment](../model/src/fragment.ts#L10-L261) 是 Node 内部存储子节点的容器。其核心方法：

[fragment.ts#L71-L83](../model/src/fragment.ts#L71-L83) 的 append 方法会自动合并相邻同 markup 文本节点：

```typescript
  /// Create a new fragment containing the combined content of this
  /// fragment and the other.
  append(other: Fragment) {
    if (!other.size) return this
    if (!this.size) return other
    let last = this.lastChild!, first = other.firstChild!, content = this.content.slice(), i = 0
    if (last.isText && last.sameMarkup(first)) {
      content[content.length - 1] = (last as TextNode).withText(last.text! + first.text!)
      i = 1
    }
    for (; i < other.content.length; i++) content.push(other.content[i])
    return new Fragment(content, this.size + other.size)
  }
```

[fragment.ts#L85-L104](../model/src/fragment.ts#L85-L104) 的 cut 方法返回新 Fragment：

```typescript
  /// Cut out the sub-fragment between the two given positions.
  cut(from: number, to = this.size) {
    if (from == 0 && to == this.size) return this
    let result: Node[] = [], size = 0
    if (to > from) for (let i = 0, pos = 0; pos < to; i++) {
      let child = this.content[i], end = pos + child.nodeSize
      if (end > from) {
        if (pos < from || end > to) {
          if (child.isText)
            child = child.cut(Math.max(0, from - pos), Math.min(child.text!.length, to - pos))
          else
            child = child.cut(Math.max(0, from - pos - 1), Math.min(child.content.size, to - pos - 1))
        }
        result.push(child)
        size += child.nodeSize
      }
      pos = end
    }
    return new Fragment(result, size)
  }
```

[Fragment.fromArray](../model/src/fragment.ts#L225-L242) 同样会合并相邻同 markup 文本节点：

```typescript
  /// Build a fragment from an array of nodes. Ensures that adjacent
  /// text nodes with the same marks are joined together.
  static fromArray(array: readonly Node[]) {
    if (!array.length) return Fragment.empty
    let joined: Node[] | undefined, size = 0
    for (let i = 0; i < array.length; i++) {
      let node = array[i]
      size += node.nodeSize
      if (i && node.isText && array[i - 1].sameMarkup(node)) {
        if (!joined) joined = array.slice(0, i)
        joined[joined.length - 1] = (node as TextNode)
                                      .withText((joined[joined.length - 1] as TextNode).text + (node as TextNode).text)
      } else if (joined) {
        joined.push(node)
      }
    }
    return new Fragment(joined || array, size)
  }
```

设计要点：
- Fragment 同样持久化，所有修改返回新实例
- `Fragment.empty`（[fragment.ts#L257-L260](../model/src/fragment.ts#L257-L260)）全局共享，避免为每个空节点分配对象：

```typescript
  /// An empty fragment. Intended to be reused whenever a node doesn't
  /// contain anything (rather than allocating a new empty fragment for
  /// each leaf node).
  static empty: Fragment = new Fragment([], 0)
```

---

## 5. ResolvedPos：位置的上下文解析

将扁平整数位置"解析"为包含祖先链、索引、偏移等丰富上下文的对象，定义于 [resolvedpos.ts#L12-L250](../model/src/resolvedpos.ts#L12-L250)。

`path` 数组每三个元素为一组 `[node, childIndex, startPosition]`，从根节点到当前父节点，支持 O(depth) 访问任意祖先。

位置解析的核心算法 [ResolvedPos.resolve](../model/src/resolvedpos.ts#L217-L233)：

```typescript
  /// @internal
  static resolve(doc: Node, pos: number): ResolvedPos {
    if (!(pos >= 0 && pos <= doc.content.size)) throw new RangeError("Position " + pos + " out of range")
    let path: Array<Node | number> = []
    let start = 0, parentOffset = pos
    for (let node = doc;;) {
      let {index, offset} = node.content.findIndex(parentOffset)
      let rem = parentOffset - offset
      path.push(node, index, start + offset)
      if (!rem) break
      node = node.child(index)
      if (node.isText) break
      parentOffset = rem - 1
      start += offset + 1
    }
    return new ResolvedPos(pos, path, parentOffset)
  }
```

解析结果有缓存（[resolvedpos.ts#L235-L249](../model/src/resolvedpos.ts#L235-L249)），使用 WeakMap 按文档实例缓存最近解析的 12 个位置，避免重复遍历树：

```typescript
  /// @internal
  static resolveCached(doc: Node, pos: number): ResolvedPos {
    let cache = resolveCache.get(doc)
    if (cache) {
      for (let i = 0; i < cache.elts.length; i++) {
        let elt = cache.elts[i]
        if (elt.pos == pos) return elt
      }
    } else {
      resolveCache.set(doc, cache = new ResolveCache)
    }
    let result = cache.elts[cache.i] = ResolvedPos.resolve(doc, pos)
    cache.i = (cache.i + 1) % resolveCacheSize
    return result
  }
```

> 返回 [README](../README.md)
