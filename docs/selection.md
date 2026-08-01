# Selection 选区系统实现原理

> 对应源码：`state/src/selection.ts`、`model/src/resolvedpos.ts`

选区系统回答两个问题：**"用户当前选中了什么"**，以及 **"文档变化后选区如何保持有效"**。

---

## 1. 抽象基类 Selection

所有选区类型都继承自抽象类 `Selection`（[selection.ts:9-23](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L9-L23)）：

```ts
export abstract class Selection {
  constructor(
    readonly $anchor: ResolvedPos,   // 锚点：修改选区时不动的一端
    readonly $head: ResolvedPos,     // 头：随操作移动的一端
    ranges?: readonly SelectionRange[]
  ) {
    this.ranges = ranges || [new SelectionRange($anchor.min($head), $anchor.max($head))]
  }
  ranges: readonly SelectionRange[]  // 覆盖的范围（支持多范围，如单元格多选）
}
```

派生的便捷访问器（[selection.ts:29-56](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L29-L56)）：
- `anchor` / `head`：未解析的整数位置
- `from` / `to`：主范围的上下界（`from <= to`）
- `$from` / `$to`：其 ResolvedPos 版本
- `empty`：所有范围是否都为空（即光标）

抽象方法：`eq`（相等）、`map`（映射到新文档）、`toJSON`（序列化）。

---

## 2. ResolvedPos —— 选区定位的基石

选区不存裸整数，而存 `ResolvedPos`。一个整数位置被"解析"后携带完整上下文：

```ts
// model/src/resolvedpos.ts:12-52（节选）
export class ResolvedPos {
  depth: number
  constructor(
    readonly pos: number,          // 原始整数位置
    readonly path: any[],          // [node, index, offset, node, index, offset, ...] 逐层
    readonly parentOffset: number  // 在直接父节点中的偏移
  ) { this.depth = path.length / 3 - 1 }

  get parent() { return this.node(this.depth) }        // 直接父节点
  get doc() { return this.node(0) }                    // 根节点
  node(depth) { return this.path[this.resolveDepth(depth) * 3] }     // 第 depth 层祖先
  index(depth) { return this.path[this.resolveDepth(depth) * 3 + 1] } // 在该层的索引
}
```

一次解析即可 O(1) 回答："这个位置在树的第几层？父节点是谁？在父节点里排第几？"。选区的所有语义判断（能否放光标、往哪个方向找合法落点）都基于 ResolvedPos。

---

## 3. 三种内置选区类型

```
                    Selection (abstract)
                          │
        ┌─────────────────┼──────────────────┐
        ▼                 ▼                  ▼
  TextSelection     NodeSelection       AllSelection
```

### 3.1 TextSelection —— 经典文本选区/光标

```ts
// state/src/selection.ts:229-246（节选）
export class TextSelection extends Selection {
  constructor($anchor: ResolvedPos, $head = $anchor) {
    checkTextSelection($anchor)   // 端点必须落在含内联内容的节点里
    checkTextSelection($head)
    super($anchor, $head)
  }
  // 空文本选区即为光标；否则为 null
  get $cursor() { return this.$anchor.pos == this.$head.pos ? this.$head : null }

  map(doc: Node, mapping: Mappable): Selection {
    let $head = doc.resolve(mapping.map(this.head))
    if (!$head.parent.inlineContent) return Selection.near($head)  // 落点非法则就近
    let $anchor = doc.resolve(mapping.map(this.anchor))
    return new TextSelection($anchor.parent.inlineContent ? $anchor : $head, $head)
  }
}
```

- `$cursor` 非空 = 折叠光标（无选中区间）。`storedMarks` 只在有 `$cursor` 时才保留（[state.ts:32-35](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L32-L35)）。
- `between`（[selection.ts:287-304](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L287-L304)）在端点不合法时自动寻找就近的文本落点。

### 3.2 NodeSelection —— 整节点选区

```ts
// state/src/selection.ts:325-343（节选）
export class NodeSelection extends Selection {
  constructor($pos: ResolvedPos) {
    let node = $pos.nodeAfter!                          // 选中 $pos 之后的那个节点
    let $end = $pos.node(0).resolve($pos.pos + node.nodeSize)
    super($pos, $end)                                   // from/to 正好包住该节点
    this.node = node
  }
  map(doc, mapping): Selection {
    let {deleted, pos} = mapping.mapResult(this.anchor)
    let $pos = doc.resolve(pos)
    if (deleted) return Selection.near($pos)            // 节点被删则退化为就近选区
    return new NodeSelection($pos)
  }
  static isSelectable(node: Node) {                     // 文本节点不可整节点选中
    return !node.isText && node.type.spec.selectable !== false
  }
}
```

用于选中图片、水平线等原子节点。`visible = false`（[selection.ts:378](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L378)），因为它的高亮由视图另行绘制。

### 3.3 AllSelection —— 全文档选区

```ts
// state/src/selection.ts:399-420（节选）
export class AllSelection extends Selection {
  constructor(doc: Node) {
    super(doc.resolve(0), doc.resolve(doc.content.size))
  }
  replace(tr, content = Slice.empty) {
    if (content == Slice.empty) {                       // 全删有特殊快捷路径
      tr.delete(0, tr.doc.content.size)
      let sel = Selection.atStart(tr.doc)
      if (!sel.eq(tr.selection)) tr.setSelection(sel)
    } else { super.replace(tr, content) }
  }
  map(doc) { return new AllSelection(doc) }             // 映射恒为全选
}
```

处理"文档首尾是叶子块节点，纯文本选区无法覆盖整篇"的边界情况（Ctrl+A）。

---

## 4. 选区如何保持有效：near / findFrom

文档结构复杂（块、内联、叶子混杂），映射后的位置未必是合法选区落点。ProseMirror 用一组静态方法保证"选区永远有效"：

```ts
// state/src/selection.ts:118-151（节选）
static findFrom($pos, dir, textOnly = false): Selection | null {
  // 先看当前位置所在节点是否可放文本选区；否则在同级/祖先里按方向搜索
  let inner = $pos.parent.inlineContent ? new TextSelection($pos)
      : findSelectionIn($pos.node(0), $pos.parent, $pos.pos, $pos.index(), dir, textOnly)
  if (inner) return inner
  for (let depth = $pos.depth - 1; depth >= 0; depth--) { ...向上逐层找... }
  return null
}
static near($pos, bias = 1): Selection {
  // 先按 bias 方向找，找不到反向找，再兜底 AllSelection —— 保证一定返回合法选区
  return this.findFrom($pos, bias) || this.findFrom($pos, -bias) || new AllSelection($pos.node(0))
}
static atStart(doc) { return findSelectionIn(doc, doc, 0, 0, 1) || new AllSelection(doc) }
static atEnd(doc)   { return findSelectionIn(doc, doc, doc.content.size, doc.childCount, -1) || new AllSelection(doc) }
```

因此每种选区的 `map` 在落点非法时都会调用 `Selection.near` 退化到最近的合法位置，绝不会产生"悬空"选区。

---

## 5. 选区跟随文档变更（与事务的协作）

选区映射不由选区自己触发，而是在 Transaction 里惰性完成：

```ts
// state/src/transaction.ts:71-77
get selection(): Selection {
  if (this.curSelectionFor < this.steps.length) {
    this.curSelection = this.curSelection.map(this.doc, this.mapping.slice(this.curSelectionFor))
    this.curSelectionFor = this.steps.length
  }
  return this.curSelection
}
```

流程图：

```
 事务累积若干 Step（文档改变，mapping 增长）
        │
        ▼
 读取 tr.selection  ── curSelectionFor < steps.length ？
        │                        │ 是
        │                        ▼
        │        curSelection.map(newDoc, mapping.slice(curSelectionFor))
        │                        │
        │             各类型自行处理：落点非法 → Selection.near 退化
        │                        ▼
        └──────────────▶ 返回映射后、保证有效的新选区
```

`state` 通过 `selection` 字段的 `apply(tr) => tr.selection` 把它固化进新状态（[state.ts:27-30](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L27-L30)）。

---

## 6. 选区与内容操作

`Selection.replace / replaceWith`（[selection.ts:72-105](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L72-L105)）把"替换选区内容"落到事务的 replaceRange 上，并在插入后用 `selectionToInsertionEnd` 把光标移到插入内容末尾。多范围选区会逐个处理，只有主范围放入新内容，其余范围删除。

`Transaction.replaceSelection / replaceSelectionWith / deleteSelection / insertText`（[transaction.ts:141-183](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts#L141-L183)）是这些能力面向命令层的封装。

---

## 7. 可序列化与 Bookmark

- 每种选区通过 `Selection.jsonID` 注册（如 [selection.ts:307](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L307)），支持 `toJSON`/`fromJSON`。
- `SelectionBookmark`（[selection.ts:195-204](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L195-L204)）是不依赖具体文档的轻量选区表示，可先 `map` 再 `resolve`，主要供 history 存储/恢复旧选区使用（`TextBookmark`/`NodeBookmark`，[selection.ts:309-393](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/selection.ts#L309-L393)）。

---

## 8. 设计要点小结

1. **anchor/head 抽象**统一表达光标、区间、方向。
2. **ResolvedPos**为选区提供 O(1) 的结构上下文，是所有判断的基础。
3. **三种选区**分别覆盖文本、原子节点、整篇三类语义。
4. **near/findFrom 兜底 + 各类型 map 退化**保证选区在任意文档变更后都有效。
5. **惰性映射**让选区自动、无感地跟随事务中的文档变更。
