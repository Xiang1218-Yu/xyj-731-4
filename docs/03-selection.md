# Selection 选区系统

> 对应本地源码：[selection.ts](../state/src/selection.ts)

ProseMirror 的选区系统支持多种选区类型、自动映射和序列化，是其最精巧的设计之一。

---

## 1. 选区基类与 Range 模型

[Selection](../state/src/selection.ts#L9-L188) 是所有选区的抽象基类。其 constructor 和核心属性位于 [selection.ts#L10-L48](../state/src/selection.ts#L10-L48)：

```typescript
  /// Initialize a selection with the head and anchor and ranges. If no
  /// ranges are given, constructs a single range across `$anchor` and
  /// `$head`.
  constructor(
    /// The resolved anchor of the selection (the side that stays in
    /// place when the selection is modified).
    readonly $anchor: ResolvedPos,
    /// The resolved head of the selection (the side that moves when
    /// the selection is modified).
    readonly $head: ResolvedPos,
    ranges?: readonly SelectionRange[]
  ) {
    this.ranges = ranges || [new SelectionRange($anchor.min($head), $anchor.max($head))]
  }

  /// The ranges covered by the selection.
  ranges: readonly SelectionRange[]

  /// The selection's anchor, as an unresolved position.
  get anchor() { return this.$anchor.pos }

  /// The selection's head.
  get head() { return this.$head.pos }

  /// The lower bound of the selection's main range.
  get from() { return this.$from.pos }

  /// The upper bound of the selection's main range.
  get to() { return this.$to.pos }

  /// The resolved lower  bound of the selection's main range.
  get $from() {
    return this.ranges[0].$from
  }

  /// The resolved upper bound of the selection's main range.
  get $to() {
    return this.ranges[0].$to
  }
```

### anchor/head vs from/to

- `anchor` 和 `head` 模型用户的选择方向（anchor 是按下鼠标时的位置，head 是拖动到的位置）
- `from` 和 `to` 始终是范围的下界和上界（from ≤ to），不考虑方向
- 大多数编辑操作使用 `from`/`to`

[SelectionRange](../state/src/selection.ts#L207-L215) 表示一个连续范围，Selection 支持多个 ranges，为未来多选区预留：

```typescript
/// Represents a selected range in a document.
export class SelectionRange {
  /// Create a range.
  constructor(
    /// The lower bound of the range.
    readonly $from: ResolvedPos,
    /// The upper bound of the range.
    readonly $to: ResolvedPos
  ) {}
}
```

---

## 2. 三种内置选区类型

```
Selection (abstract)
├── TextSelection      # 文本选区（光标是其空选特例）
├── NodeSelection      # 节点选区（选中整个原子/块节点）
└── AllSelection       # 全选（选中文档全部内容）
```

### TextSelection

[TextSelection](../state/src/selection.ts#L225-L305) 两端都必须指向内联内容：

```typescript
/// A text selection represents a classical editor selection, with a
/// head (the moving side) and anchor (immobile side), both of which
/// point into textblock nodes. It can be empty (a regular cursor
/// position).
export class TextSelection extends Selection {
  /// Construct a text selection between the given points.
  constructor($anchor: ResolvedPos, $head = $anchor) {
    checkTextSelection($anchor)
    checkTextSelection($head)
    super($anchor, $head)
  }

  /// Returns a resolved position if this is a cursor selection (an
  /// empty text selection), and null otherwise.
  get $cursor() { return this.$anchor.pos == this.$head.pos ? this.$head : null }

  map(doc: Node, mapping: Mappable): Selection {
    let $head = doc.resolve(mapping.map(this.head))
    if (!$head.parent.inlineContent) return Selection.near($head)
    let $anchor = doc.resolve(mapping.map(this.anchor))
    return new TextSelection($anchor.parent.inlineContent ? $anchor : $head, $head)
  }

  replace(tr: Transaction, content = Slice.empty) {
    super.replace(tr, content)
    if (content == Slice.empty) {
      let marks = this.$from.marksAcross(this.$to)
      if (marks) tr.ensureMarks(marks)
    }
  }

  eq(other: Selection): boolean {
    return other instanceof TextSelection && other.anchor == this.anchor && other.head == this.head
  }

  getBookmark() {
    return new TextBookmark(this.anchor, this.head)
  }

  toJSON(): any {
    return {type: "text", anchor: this.anchor, head: this.head}
  }

  /// @internal
  static fromJSON(doc: Node, json: any) {
    if (typeof json.anchor != "number" || typeof json.head != "number")
      throw new RangeError("Invalid input for TextSelection.fromJSON")
    return new TextSelection(doc.resolve(json.anchor), doc.resolve(json.head))
  }

  /// Create a text selection from non-resolved positions.
  static create(doc: Node, anchor: number, head = anchor) {
    let $anchor = doc.resolve(anchor)
    return new this($anchor, head == anchor ? $anchor : doc.resolve(head))
  }

  /// Return a text selection that spans the given positions or, if
  /// they aren't text positions, find a text selection near them.
  /// `bias` determines whether the method searches forward (default)
  /// or backwards (negative number) first. Will fall back to calling
  /// [`Selection.near`](#state.Selection^near) when the document
  /// doesn't contain a valid text position.
  static between($anchor: ResolvedPos, $head: ResolvedPos, bias?: number): Selection {
    let dPos = $anchor.pos - $head.pos
    if (!bias || dPos) bias = dPos >= 0 ? 1 : -1
    if (!$head.parent.inlineContent) {
      let found = Selection.findFrom($head, bias, true) || Selection.findFrom($head, -bias, true)
      if (found) $head = found.$head
      else return Selection.near($head, bias)
    }
    if (!$anchor.parent.inlineContent) {
      if (dPos == 0) {
        $anchor = $head
      } else {
        $anchor = (Selection.findFrom($anchor, -bias, true) || Selection.findFrom($anchor, bias, true))!.$anchor
        if (($anchor.pos < $head.pos) != (dPos < 0)) $anchor = $head
      }
    }
    return new TextSelection($anchor, $head)
  }
}
```

当 `anchor == head` 时，TextSelection 表示一个**光标（collapsed selection）**，`$cursor` 属性返回该位置，否则返回 null。

### NodeSelection

[NodeSelection](../state/src/selection.ts#L320-L376) 选中整个非文本节点：

```typescript
/// A node selection is a selection that points at a single node. All
/// nodes marked [selectable](#model.NodeSpec.selectable) can be the
/// target of a node selection. In such a selection, `from` and `to`
/// point directly before and after the selected node, `anchor` equals
/// `from`, and `head` equals `to`..
export class NodeSelection extends Selection {
  /// Create a node selection. Does not verify the validity of its
  /// argument.
  constructor($pos: ResolvedPos) {
    let node = $pos.nodeAfter!
    let $end = $pos.node(0).resolve($pos.pos + node.nodeSize)
    super($pos, $end)
    this.node = node
  }

  /// The selected node.
  node: Node

  map(doc: Node, mapping: Mappable): Selection {
    let {deleted, pos} = mapping.mapResult(this.anchor)
    let $pos = doc.resolve(pos)
    if (deleted) return Selection.near($pos)
    return new NodeSelection($pos)
  }

  content() {
    return new Slice(Fragment.from(this.node), 0, 0)
  }

  eq(other: Selection): boolean {
    return other instanceof NodeSelection && other.anchor == this.anchor
  }

  toJSON(): any {
    return {type: "node", anchor: this.anchor}
  }

  getBookmark() { return new NodeBookmark(this.anchor) }

  /// @internal
  static fromJSON(doc: Node, json: any) {
    if (typeof json.anchor != "number")
      throw new RangeError("Invalid input for NodeSelection.fromJSON")
    return new NodeSelection(doc.resolve(json.anchor))
  }

  /// Create a node selection from non-resolved positions.
  static create(doc: Node, from: number) {
    return new NodeSelection(doc.resolve(from))
  }

  /// Determines whether the given node may be selected as a node
  /// selection.
  static isSelectable(node: Node) {
    return !node.isText && node.type.spec.selectable !== false
  }
}
```

用于图片、表格、视频等原子节点的选择。其 `visible` 默认为 false（不显示原生浏览器选区，由 NodeView 自定义视觉表现），设置见 [selection.ts#L378](../state/src/selection.ts#L378)。

### AllSelection

[AllSelection](../state/src/selection.ts#L395-L425) 用于全选：

```typescript
/// A selection type that represents selecting the whole document
/// (which can not necessarily be expressed with a text selection, when
/// there are for example leaf block nodes at the start or end of the
/// document).
export class AllSelection extends Selection {
  /// Create an all-selection over the given document.
  constructor(doc: Node) {
    super(doc.resolve(0), doc.resolve(doc.content.size))
  }

  replace(tr: Transaction, content = Slice.empty) {
    if (content == Slice.empty) {
      tr.delete(0, tr.doc.content.size)
      let sel = Selection.atStart(tr.doc)
      if (!sel.eq(tr.selection)) tr.setSelection(sel)
    } else {
      super.replace(tr, content)
    }
  }

  toJSON(): any { return {type: "all"} }

  /// @internal
  static fromJSON(doc: Node) { return new AllSelection(doc) }

  map(doc: Node) { return new AllSelection(doc) }

  eq(other: Selection) { return other instanceof AllSelection }

  getBookmark() { return AllBookmark }
}
```

用于文档开头/结尾有叶子块节点（如图片）时无法用 TextSelection 表示全选的情况。

### 类型注册与反序列化

每种选区类型通过 `Selection.jsonID()` 注册字符串 ID。以下三行分别位于 [selection.ts#L307](../state/src/selection.ts#L307)、[L380](../state/src/selection.ts#L380)、[L427](../state/src/selection.ts#L427)（互不相邻，此处合并展示）：

```typescript
Selection.jsonID("text", TextSelection)
Selection.jsonID("node", NodeSelection)
Selection.jsonID("all", AllSelection)
```

这使得选区可随状态序列化保存，并在协同编辑中传输。

---

## 3. 选区映射（Mapping）机制

文档被 Transaction 修改后，选区需相应调整。TextSelection.map 已在上文展示，NodeSelection.map 的关键逻辑是：选中节点被删除时（`deleted` 为 true），退化为最近的文本选区。

映射的容错性由 [Selection.findFrom](../state/src/selection.ts#L113-L130) 提供：

```typescript
  /// Find a valid cursor or leaf node selection starting at the given
  /// position and searching back if `dir` is negative, and forward if
  /// positive. When `textOnly` is true, only consider cursor
  /// selections. Will return null when no valid selection position is
  /// found.
  static findFrom($pos: ResolvedPos, dir: number, textOnly: boolean = false): Selection | null {
    let inner = $pos.parent.inlineContent ? new TextSelection($pos)
        : findSelectionIn($pos.node(0), $pos.parent, $pos.pos, $pos.index(), dir, textOnly)
    if (inner) return inner

    for (let depth = $pos.depth - 1; depth >= 0; depth--) {
      let found = dir < 0
          ? findSelectionIn($pos.node(0), $pos.node(depth), $pos.before(depth + 1), $pos.index(depth), dir, textOnly)
          : findSelectionIn($pos.node(0), $pos.node(depth), $pos.after(depth + 1), $pos.index(depth) + 1, dir, textOnly)
      if (found) return found
    }
    return null
  }
```

该方法向上搜索父节点、在不同深度寻找有效选区位置。映射后位置无效时，使用 `Selection.near($pos)` 寻找最近有效光标。

---

## 4. Bookmark：文档无关的选区快照

[SelectionBookmark](../state/src/selection.ts#L192-L204) 是轻量级、文档无关的选区表示，主要用于历史记录（undo/redo）：

```typescript
/// A lightweight, document-independent representation of a selection.
/// You can define a custom bookmark type for a custom selection class
/// to make the history handle it well.
export interface SelectionBookmark {
  /// Map the bookmark through a set of changes.
  map: (mapping: Mappable) => SelectionBookmark

  /// Resolve the bookmark to a real selection again. This may need to
  /// do some error checking and may fall back to a default (usually
  /// [`TextSelection.between`](#state.TextSelection^between)) if
  /// mapping made the bookmark invalid.
  resolve: (doc: Node) => Selection
}
```

Bookmark 只存储位置数字，不持有 ResolvedPos 或文档引用，可通过 mapping 调整位置，需要时 resolve 回真实 Selection。每种选区类型有对应 Bookmark 实现（TextBookmark、NodeBookmark、AllBookmark），`getBookmark()` 返回当前选区快照。

> 返回 [README](../README.md)
