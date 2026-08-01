# Transaction 事务机制与不可变性

> 对应本地源码目录：[../transform/src/](../transform/src)、[../state/src/](../state/src)
>
> 核心文件：[transform.ts](../transform/src/transform.ts)、[step.ts](../transform/src/step.ts)、[map.ts](../transform/src/map.ts)、[transaction.ts](../state/src/transaction.ts)、[state.ts](../state/src/state.ts)

ProseMirror 的状态管理是函数式不可变思想的典范。所有状态变更通过 Transaction 进行，生成全新状态对象，旧状态不被修改。

---

## 1. 持久化数据结构与结构共享

ProseMirror 的不可变性不是通过深拷贝实现的，而是通过**结构共享（structural sharing）**：

```
          旧文档树                          新文档树
             │                                │
          doc(Node)                        doc'(Node) ◄── 新对象
         /        \                       /        \
    p1(Node)    p2(Node)             p1(Node)    p2'(Node) ◄── 仅重建路径上的节点
    /    \        \                  /    \        \
  "a"   "b"     "c"               "a"   "b"     "c!" ◄── 新文本节点
   │      │        │                │      │        │
   └──────┴────────┴── 共享 ────────┴──────┘        （未修改节点复用引用）
```

核心保证：
- Node/Fragment/Mark[] 所有变更方法都返回新实例
- 新内容与旧内容引用相等时直接返回 `this`
- 只重建从根到变更点路径上的节点，其他子树共享
- Mark 数组不可变，增删返回新数组

---

## 2. Step：原子变更单元

所有文档修改抽象为 `Step` 对象——可序列化、可反转、可重映射的原子操作。

### Step 抽象类

[step.ts#L7-L67](../transform/src/step.ts#L7-L67)：

```typescript
/// A step object represents an atomic change. It generally applies
/// only to the document it was created for, since the positions
/// stored in it will only make sense for that document.
///
/// New steps are defined by creating classes that extend `Step`,
/// overriding the `apply`, `invert`, `map`, `getMap` and `fromJSON`
/// methods, and registering your class with a unique
/// JSON-serialization identifier using
/// [`Step.jsonID`](#transform.Step^jsonID).
export abstract class Step {
  /// Applies this step to the given document, returning a result
  /// object that either indicates failure, if the step can not be
  /// applied to this document, or indicates success by containing a
  /// transformed document.
  abstract apply(doc: Node): StepResult

  /// Get the step map that represents the changes made by this step,
  /// and which can be used to transform between positions in the old
  /// and the new document.
  getMap(): StepMap { return StepMap.empty }

  /// Create an inverted version of this step. Needs the document as it
  /// was before the step as argument.
  abstract invert(doc: Node): Step

  /// Map this step through a mappable thing, returning either a
  /// version of that step with its positions adjusted, or `null` if
  /// the step was entirely deleted by the mapping.
  abstract map(mapping: Mappable): Step | null

  /// Try to merge this step with another one, to be applied directly
  /// after it. Returns the merged step when possible, null if the
  /// steps can't be merged.
  merge(other: Step): Step | null { return null }

  /// Create a JSON-serializeable representation of this step. When
  /// defining this for a custom subclass, make sure the result object
  /// includes the step type's [JSON id](#transform.Step^jsonID) under
  /// the `stepType` property.
  abstract toJSON(): any

  /// Deserialize a step from its JSON representation. Will call
  /// through to the step class' own implementation of this method.
  static fromJSON(schema: Schema, json: any): Step {
    if (!json || !json.stepType) throw new RangeError("Invalid input for Step.fromJSON")
    let type = stepsByID[json.stepType]
    if (!type) throw new RangeError(`No step type ${json.stepType} defined`)
    return type.fromJSON(schema, json)
  }

  /// To be able to serialize steps to JSON, each step needs a string
  /// ID to attach to its JSON representation. Use this method to
  /// register an ID for your step classes. Try to pick something
  /// that's unlikely to clash with steps from other modules.
  static jsonID(id: string, stepClass: {fromJSON(schema: Schema, json: any): Step}) {
    if (id in stepsByID) throw new RangeError("Duplicate use of step JSON ID " + id)
    stepsByID[id] = stepClass
    ;(stepClass as any).prototype.jsonID = id
    return stepClass
  }
}
```

### StepResult

[step.ts#L69-L97](../transform/src/step.ts#L69-L97)：

```typescript
/// The result of [applying](#transform.Step.apply) a step. Contains either a
/// new document or a failure value.
export class StepResult {
  /// @internal
  constructor(
    /// The transformed document, if successful.
    readonly doc: Node | null,
    /// The failure message, if unsuccessful.
    readonly failed: string | null
  ) {}

  /// Create a successful step result.
  static ok(doc: Node) { return new StepResult(doc, null) }

  /// Create a failed step result.
  static fail(message: string) { return new StepResult(null, message) }

  /// Call [`Node.replace`](#model.Node.replace) with the given
  /// arguments. Create a successful result if it succeeds, and a
  /// failed one if it throws a `ReplaceError`.
  static fromReplace(doc: Node, from: number, to: number, slice: Slice) {
    try {
      return StepResult.ok(doc.replace(from, to, slice))
    } catch (e) {
      if (e instanceof ReplaceError) return StepResult.fail(e.message)
      throw e
    }
  }
}
```

### 内置 Step 类型

| Step 类 | 文件 | 作用 |
|---------|------|------|
| `ReplaceStep` | [replace_step.ts](../transform/src/replace_step.ts) | 替换文档范围 |
| `ReplaceAroundStep` | [replace_step.ts](../transform/src/replace_step.ts) | 围绕范围替换（结构化操作） |
| `AttrStep` | [attr_step.ts](../transform/src/attr_step.ts) | 修改节点属性 |
| `DocAttrStep` | [attr_step.ts](../transform/src/attr_step.ts) | 修改文档根节点属性 |
| `AddMarkStep` | [mark_step.ts](../transform/src/mark_step.ts) | 给范围添加 mark |
| `RemoveMarkStep` | [mark_step.ts](../transform/src/mark_step.ts) | 从范围移除 mark |
| `AddNodeMarkStep` | [mark_step.ts](../transform/src/mark_step.ts) | 给节点添加 mark |
| `RemoveNodeMarkStep` | [mark_step.ts](../transform/src/mark_step.ts) | 移除节点 mark |

这种设计带来的能力：**可审计**（每步变更是明确数据对象）、**可撤销**（每步可 invert）、**可传输**（JSON 序列化，协同编辑基础）、**可重映射**（文档变化后调整位置）。

---

## 3. StepMap 与位置映射

StepMap 用紧凑的三元组数组记录一个 Step 对位置的影响。

### StepMap

[map.ts#L68-L116](../transform/src/map.ts#L68-L116)：

```typescript
/// A map describing the deletions and insertions made by a step, which
/// can be used to find the correspondence between positions in the
/// pre-step version of a document and the same position in the
/// post-step version.
export class StepMap implements Mappable {
  /// Create a position map. The modifications to the document are
  /// represented as an array of numbers, in which each group of three
  /// represents a modified chunk as `[start, oldSize, newSize]`.
  constructor(
    /// @internal
    readonly ranges: readonly number[],
    /// @internal
    readonly inverted = false
  ) {
    if (!ranges.length && StepMap.empty) return StepMap.empty
  }

  /// @internal
  recover(value: number) {
    let diff = 0, index = recoverIndex(value)
    if (!this.inverted) for (let i = 0; i < index; i++)
      diff += this.ranges[i * 3 + 2] - this.ranges[i * 3 + 1]
    return this.ranges[index * 3] + diff + recoverOffset(value)
  }

  mapResult(pos: number, assoc = 1): MapResult { return this._map(pos, assoc, false) as MapResult }

  map(pos: number, assoc = 1): number { return this._map(pos, assoc, true) as number }

  /// @internal
  _map(pos: number, assoc: number, simple: boolean) {
    let diff = 0, oldIndex = this.inverted ? 2 : 1, newIndex = this.inverted ? 1 : 2
    for (let i = 0; i < this.ranges.length; i += 3) {
      let start = this.ranges[i] - (this.inverted ? diff : 0)
      if (start > pos) break
      let oldSize = this.ranges[i + oldIndex], newSize = this.ranges[i + newIndex], end = start + oldSize
      if (pos <= end) {
        let side = !oldSize ? assoc : pos == start ? -1 : pos == end ? 1 : assoc
        let result = start + diff + (side < 0 ? 0 : newSize)
        if (simple) return result
        let recover = pos == (assoc < 0 ? start : end) ? null : makeRecover(i / 3, pos - start)
        let del = pos == start ? DEL_AFTER : pos == end ? DEL_BEFORE : DEL_ACROSS
        if (assoc < 0 ? pos != start : pos != end) del |= DEL_SIDE
        return new MapResult(result, del, recover)
      }
      diff += newSize - oldSize
    }
    return simple ? pos + diff : new MapResult(pos + diff, 0, null)
  }
```

例如在位置 5 删除 3 字符、插入 2 字符，对应 `[5, 3, 2]`：
- 位置 0-4：不变
- 位置 5-8（被删范围）：根据 `assoc` 方向映射到 5 或 7
- 位置 8+：偏移 -1

### Mapping：多步映射流水线

[map.ts#L166-L211](../transform/src/map.ts#L166-L211) 组合多个 StepMap，展示类声明到 `appendMap` 方法（`Mapping` 类完整定义到 L284，还包含 appendMapping、getMirror、setMirror、invert、map、mapResult、_map 等方法）：

```typescript
/// A mapping represents a pipeline of zero or more [step
/// maps](#transform.StepMap). It has special provisions for losslessly
/// handling mapping positions through a series of steps in which some
/// steps are inverted versions of earlier steps. (This comes up when
/// ‘[rebasing](/docs/guide/#transform.rebasing)’ steps for
/// collaboration or history management.)
export class Mapping implements Mappable {
  /// Create a new mapping with the given position maps.
  constructor(
    maps?: readonly StepMap[],
    /// @internal
    public mirror?: number[],
    /// The starting position in the `maps` array, used when `map` or
    /// `mapResult` is called.
    public from = 0,
    /// The end position in the `maps` array.
    public to = maps ? maps.length : 0
  ) {
    this._maps = (maps as StepMap[]) || []
    this.ownData = !(maps || mirror)
  }

  /// The step maps in this mapping.
  get maps(): readonly StepMap[] { return this._maps }

  private _maps: StepMap[]
  // False if maps/mirror are shared arrays that we shouldn't mutate
  private ownData: boolean

  /// Create a mapping that maps only through a part of this one.
  slice(from = 0, to = this.maps.length) {
    return new Mapping(this._maps, this.mirror, from, to)
  }

  /// Add a step map to the end of this mapping. If `mirrors` is
  /// given, it should be the index of the step map that is the mirror
  /// image of this one.
  appendMap(map: StepMap, mirrors?: number) {
    if (!this.ownData) {
      this._maps = this._maps.slice()
      this.mirror = this.mirror && this.mirror.slice()
      this.ownData = true
    }
    this.to = this._maps.push(map)
    if (mirrors != null) this.setMirror(this._maps.length - 1, mirrors)
  }
```

Mapping 的关键能力：
- 顺序映射位置通过多个 step
- **镜像（mirroring）**：记录互为逆操作的 step，映射时跳过它们避免位置信息丢失（协同 rebase 的关键）
- `appendMap(map, mirrors?)` 添加映射并指定镜像索引
- `invert()` 创建整体逆映射

---

## 4. Transform：纯文档转换构建器

[Transform](../transform/src/transform.ts#L28-L271) 是构建文档变更的基础类。

### 类声明与核心字段

[transform.ts#L23-L44](../transform/src/transform.ts#L23-L44)：

```typescript
/// Abstraction to build up and track an array of
/// [steps](#transform.Step) representing a document transformation.
///
/// Most transforming methods return the `Transform` object itself, so
/// that they can be chained.
export class Transform {
  /// The steps in this transform.
  readonly steps: Step[] = []
  /// The documents before each of the steps.
  readonly docs: Node[] = []
  /// A mapping with the maps for each of the steps in this transform.
  readonly mapping: Mapping = new Mapping

  /// Create a transform that starts with the given document.
  constructor(
    /// The current document (the result of applying the steps in the
    /// transform).
    public doc: Node
  ) {}

  /// The starting document.
  get before() { return this.docs.length ? this.docs[0] : this.doc }
```

### step / maybeStep / addStep

[transform.ts#L46-L94](../transform/src/transform.ts#L46-L94)：

```typescript
  /// Apply a new step in this transform, saving the result. Throws an
  /// error when the step fails.
  step(step: Step) {
    let result = this.maybeStep(step)
    if (result.failed) throw new TransformError(result.failed)
    return this
  }

  /// Try to apply a step in this transformation, ignoring it if it
  /// fails. Returns the step result.
  maybeStep(step: Step) {
    let result = step.apply(this.doc)
    if (!result.failed) this.addStep(step, result.doc!)
    return result
  }
```

[transform.ts#L88-L94](../transform/src/transform.ts#L88-L94)：

```typescript
  /// @internal
  addStep(step: Step, doc: Node) {
    this.docs.push(this.doc)
    this.steps.push(step)
    this.mapping.appendMap(step.getMap())
    this.doc = doc
  }
```

Transform 提供链式 API：`replace`、`delete`、`insert`、`split`、`join`、`lift`、`wrap`、`setBlockType`、`addMark`、`removeMark` 等。**Transform 只关心文档变更，不涉及选区、marks 等编辑器状态。**

---

## 5. Transaction：编辑器状态事务

[Transaction](../state/src/transaction.ts#L42-L215) 继承 Transform，在文档变更之上增加选区、storedMarks、元数据。

### 类声明与 constructor

[state/src/transaction.ts#L20-L65](../state/src/transaction.ts#L20-L65)：

```typescript
const UPDATED_SEL = 1, UPDATED_MARKS = 2, UPDATED_SCROLL = 4
```

[transaction.ts#L42-L65](../state/src/transaction.ts#L42-L65)：

```typescript
export class Transaction extends Transform {
  /// The timestamp associated with this transaction, in the same
  /// format as `Date.now()`.
  time: number

  private curSelection: Selection
  // The step count for which the current selection is valid.
  private curSelectionFor = 0
  // Bitfield to track which aspects of the state were updated by
  // this transaction.
  private updated = 0
  // Object used to store metadata properties for the transaction.
  private meta: {[name: string]: any} = Object.create(null)

  /// The stored marks set by this transaction, if any.
  storedMarks: readonly Mark[] | null

  /// @internal
  constructor(state: EditorState) {
    super(state.doc)
    this.time = Date.now()
    this.curSelection = state.selection
    this.storedMarks = state.storedMarks
  }
```

### 选区懒映射

[transaction.ts#L67-L89](../state/src/transaction.ts#L67-L89)：

```typescript
  /// The transaction's current selection. This defaults to the editor
  /// selection [mapped](#state.Selection.map) through the steps in the
  /// transaction, but can be overwritten with
  /// [`setSelection`](#state.Transaction.setSelection).
  get selection(): Selection {
    if (this.curSelectionFor < this.steps.length) {
      this.curSelection = this.curSelection.map(this.doc, this.mapping.slice(this.curSelectionFor))
      this.curSelectionFor = this.steps.length
    }
    return this.curSelection
  }

  /// Update the transaction's current selection. Will determine the
  /// selection that the editor gets when the transaction is applied.
  setSelection(selection: Selection): this {
    if (selection.$from.doc != this.doc)
      throw new RangeError("Selection passed to setSelection must point at the current document")
    this.curSelection = selection
    this.curSelectionFor = this.steps.length
    this.updated = (this.updated | UPDATED_SEL) & ~UPDATED_MARKS
    this.storedMarks = null
    return this
  }
```

关键设计：添加 step 后不立即重算选区，读取 `.selection` 时通过 `mapping.slice(curSelectionFor)` 增量映射，避免无用计算。

### addStep 重写

[transaction.ts#L127-L132](../state/src/transaction.ts#L127-L132)：

```typescript
  /// @internal
  addStep(step: Step, doc: Node) {
    super.addStep(step, doc)
    this.updated = this.updated & ~UPDATED_MARKS
    this.storedMarks = null
  }
```

文档变更时自动清除 storedMarks（因为文档内容变化后旧的 marks 可能不再适用）。

### 元数据系统

[transaction.ts#L185-L195](../state/src/transaction.ts#L185-L195)：

```typescript
  /// Store a metadata property in this transaction, keyed either by
  /// name or by plugin.
  setMeta(key: string | Plugin | PluginKey, value: any): this {
    this.meta[typeof key == "string" ? key : key.key] = value
    return this
  }

  /// Retrieve a metadata property for a given name or plugin.
  getMeta(key: string | Plugin | PluginKey) {
    return this.meta[typeof key == "string" ? key : key.key]
  }
```

`setMeta(key, value)` 附加事务来源信息（用户输入、粘贴、撤销等），插件据此响应。位掩码 `UPDATED_SEL=1`、`UPDATED_MARKS=2`、`UPDATED_SCROLL=4`（[transaction.ts#L20](../state/src/transaction.ts#L20)）高效跟踪变更项。

---

## 6. EditorState.apply：状态更新流程

### applyTransaction

[state.ts#L132-L168](../state/src/state.ts#L132-L168)：

```typescript
  /// Verbose variant of [`apply`](#state.EditorState.apply) that
  /// returns the precise transactions that were applied (which might
  /// be influenced by the [transaction
  /// hooks](#state.PluginSpec.filterTransaction) of
  /// plugins) along with the new state.
  applyTransaction(rootTr: Transaction): {state: EditorState, transactions: readonly Transaction[]} {
    if (!this.filterTransaction(rootTr)) return {state: this, transactions: []}

    let trs = [rootTr], newState = this.applyInner(rootTr), seen = null
    // This loop repeatedly gives plugins a chance to respond to
    // transactions as new transactions are added, making sure to only
    // pass the transactions the plugin did not see before.
    for (;;) {
      let haveNew = false
      for (let i = 0; i < this.config.plugins.length; i++) {
        let plugin = this.config.plugins[i]
        if (plugin.spec.appendTransaction) {
          let n = seen ? seen[i].n : 0, oldState = seen ? seen[i].state : this
          let tr = n < trs.length &&
              plugin.spec.appendTransaction.call(plugin, n ? trs.slice(n) : trs, oldState, newState)
          if (tr && newState.filterTransaction(tr, i)) {
            tr.setMeta("appendedTransaction", rootTr)
            if (!seen) {
              seen = []
              for (let j = 0; j < this.config.plugins.length; j++)
                seen.push(j < i ? {state: newState, n: trs.length} : {state: this, n: 0})
            }
            trs.push(tr)
            newState = newState.applyInner(tr)
            haveNew = true
          }
          if (seen) seen[i] = {state: newState, n: trs.length}
        }
      }
      if (!haveNew) return {state: newState, transactions: trs}
    }
  }
```

### applyInner

[state.ts#L170-L179](../state/src/state.ts#L170-L179)：

```typescript
  /// @internal
  applyInner(tr: Transaction) {
    if (!tr.before.eq(this.doc)) throw new RangeError("Applying a mismatched transaction")
    let newInstance = new EditorState(this.config), fields = this.config.fields
    for (let i = 0; i < fields.length; i++) {
      let field = fields[i]
      ;(newInstance as any)[field.name] = field.apply(tr, (this as any)[field.name], this, newInstance)
    }
    return newInstance
  }
```

### 状态字段系统

EditorState 用 FieldDesc 管理状态字段，内置四个基础字段（[state.ts#L21-L41](../state/src/state.ts#L21-L41)）：

```typescript
const baseFields = [
  new FieldDesc<Node>("doc", {
    init(config) { return config.doc || config.schema!.topNodeType.createAndFill() },
    apply(tr) { return tr.doc }
  }),

  new FieldDesc<Selection>("selection", {
    init(config, instance) { return config.selection || Selection.atStart(instance.doc) },
    apply(tr) { return tr.selection }
  }),

  new FieldDesc<readonly Mark[] | null>("storedMarks", {
    init(config) { return config.storedMarks || null },
    apply(tr, _marks, _old, state) { return (state.selection as TextSelection).$cursor ? tr.storedMarks : null }
  }),

  new FieldDesc<number>("scrollToSelection", {
    init() { return 0 },
    apply(tr, prev) { return tr.scrolledIntoView ? prev + 1 : prev }
  })
]
```

插件通过 `PluginSpec.state` 注册自己的字段，与内置字段完全平等。

### 不可变性保证
- `applyInner` 每次 `new EditorState(this.config)`，绝不修改旧状态
- 每个字段的 `apply` 返回新值
- Configuration（schema、插件列表）在更新间共享，不重建

> 返回 [README](../README.md)
