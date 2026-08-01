# Selection 选区系统

> 对应本地源码：[selection.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts)

ProseMirror 的选区系统支持多种选区类型、自动映射和序列化，是其最精巧的设计之一。

---

## 1. 选区基类与 Range 模型

[Selection](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L9-L188) 是所有选区的抽象基类：

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

### anchor/head vs from/to

- `anchor` 和 `head` 模型用户的选择方向（anchor 是按下鼠标时的位置，head 是拖动到的位置）
- `from` 和 `to` 始终是范围的下界和上界（from ≤ to），不考虑方向
- 大多数编辑操作使用 `from`/`to`

[SelectionRange](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L207-L215) 表示一个连续范围，Selection 支持多个 ranges，为未来多选区预留。

---

## 2. 三种内置选区类型

```
Selection (abstract)
├── TextSelection      # 文本选区（光标是其空选特例）
├── NodeSelection      # 节点选区（选中整个原子/块节点）
└── AllSelection       # 全选（选中文档全部内容）
```

### TextSelection

[TextSelection](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L229-L305) 两端都必须指向内联内容：

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

### NodeSelection

[NodeSelection](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L325-L377) 选中整个非文本节点：

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

用于图片、表格、视频等原子节点的选择。其 `visible` 默认为 false（不显示原生浏览器选区，由 NodeView 自定义视觉表现）。

### AllSelection

[AllSelection](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L399-L425) 用于全选：

```typescript
export class AllSelection extends Selection {
  constructor(doc: Node) {
    super(doc.resolve(0), doc.resolve(doc.content.size))
  }
}
```

用于文档开头/结尾有叶子块节点（如图片）时无法用 TextSelection 表示全选的情况。

### 类型注册与反序列化

每种选区类型通过 `Selection.jsonID()` 注册字符串 ID：

```typescript
Selection.jsonID("text", TextSelection)
Selection.jsonID("node", NodeSelection)
Selection.jsonID("all", AllSelection)
```

这使得选区可随状态序列化保存，并在协同编辑中传输。

---

## 3. 选区映射（Mapping）机制

文档被 Transaction 修改后，选区需相应调整。每个子类实现 `map()`：

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

映射的容错性：
- 映射后位置无效时，用 `Selection.near($pos)` 寻找最近有效光标
- NodeSelection 选中的节点被删除时，退化为最近文本选区
- [Selection.findFrom](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L118-L130) 实现向上搜索父节点、在不同深度寻找有效选区的算法

---

## 4. Bookmark：文档无关的选区快照

[SelectionBookmark](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L195-L204) 是轻量级、文档无关的选区表示，主要用于历史记录（undo/redo）：

```typescript
export interface SelectionBookmark {
  map: (mapping: Mappable) => SelectionBookmark
  resolve: (doc: Node) => Selection
}
```

Bookmark 只存储位置数字，不持有 ResolvedPos 或文档引用，可通过 mapping 调整位置，需要时 resolve 回真实 Selection。每种选区类型有对应 Bookmark 实现（TextBookmark、NodeBookmark、AllBookmark），`getBookmark()` 返回当前选区快照。

> 返回 [README](../README.md)
