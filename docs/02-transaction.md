# Transaction 事务机制与不可变性

> 对应本地源码目录：[transform/src/](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src)、[state/src/](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src)
>
> 核心文件：[transform.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/transform.ts)、[step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/step.ts)、[map.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/map.ts)、[transaction.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/transaction.ts)、[state.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/state.ts)

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

[step.ts#L16-L67](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/step.ts#L16-L67)：

```typescript
export abstract class Step {
  abstract apply(doc: Node): StepResult         // 应用，返回新 doc 或失败信息
  getMap(): StepMap { return StepMap.empty }    // 位置映射表
  abstract invert(doc: Node): Step              // 反向 Step（undo 基础）
  abstract map(mapping: Mappable): Step | null  // 重映射位置（rebase/协同）
  merge(other: Step): Step | null { return null }
  abstract toJSON(): any
  static fromJSON(schema: Schema, json: any): Step { ... }
  static jsonID(id: string, stepClass: {...}) { ... }
}
```

### StepResult

[step.ts#L71-L97](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/step.ts#L71-L97)：

```typescript
export class StepResult {
  constructor(
    readonly doc: Node | null,
    readonly failed: string | null
  ) {}
  static ok(doc: Node) { return new StepResult(doc, null) }
  static fail(message: string) { return new StepResult(null, message) }
}
```

### 内置 Step 类型

| Step 类 | 文件 | 作用 |
|---------|------|------|
| `ReplaceStep` | [replace_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/replace_step.ts) | 替换文档范围 |
| `ReplaceAroundStep` | [replace_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/replace_step.ts) | 围绕范围替换（结构化操作） |
| `AttrStep` | [attr_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/attr_step.ts) | 修改节点属性 |
| `DocAttrStep` | [attr_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/attr_step.ts) | 修改文档根节点属性 |
| `AddMarkStep` | [mark_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/mark_step.ts) | 给范围添加 mark |
| `RemoveMarkStep` | [mark_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/mark_step.ts) | 从范围移除 mark |
| `AddNodeMarkStep` | [mark_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/mark_step.ts) | 给节点添加 mark |
| `RemoveNodeMarkStep` | [mark_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/mark_step.ts) | 移除节点 mark |

这种设计带来的能力：**可审计**（每步变更是明确数据对象）、**可撤销**（每步可 invert）、**可传输**（JSON 序列化，协同编辑基础）、**可重映射**（文档变化后调整位置）。

---

## 3. StepMap 与位置映射

StepMap 用紧凑的三元组数组记录一个 Step 对位置的影响（[map.ts#L72-L164](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/map.ts#L72-L164)）：

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

例如在位置 5 删除 3 字符、插入 2 字符，对应 `[5, 3, 2]`：
- 位置 0-4：不变
- 位置 5-8（被删范围）：根据 `assoc` 方向映射到 5 或 7
- 位置 8+：偏移 -1

### Mapping：多步映射流水线

[Mapping](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/map.ts#L172-L284) 组合多个 StepMap，支持：
- 顺序映射位置通过多个 step
- **镜像（mirroring）**：记录互为逆操作的 step，映射时跳过它们避免位置信息丢失（协同 rebase 的关键）
- `appendMap(map, mirrors?)` 添加映射并指定镜像索引
- `invert()` 创建整体逆映射

---

## 4. Transform：纯文档转换构建器

[Transform](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/transform.ts#L28-L271) 是构建文档变更的基础类：

```typescript
export class Transform {
  readonly steps: Step[] = []
  readonly docs: Node[] = []
  readonly mapping: Mapping = new Mapping()
  public doc: Node

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

  addStep(step: Step, doc: Node) {
    this.docs.push(this.doc)        // 保存变更前文档快照
    this.steps.push(step)
    this.mapping.appendMap(step.getMap())
    this.doc = doc                  // 更新为新文档
  }
}
```

Transform 提供链式 API：`replace`、`delete`、`insert`、`split`、`join`、`lift`、`wrap`、`setBlockType`、`addMark`、`removeMark` 等。**Transform 只关心文档变更，不涉及选区、marks 等编辑器状态。**

---

## 5. Transaction：编辑器状态事务

[Transaction](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/transaction.ts#L42-L215) 继承 Transform，在文档变更之上增加选区、storedMarks、元数据：

```typescript
export class Transaction extends Transform {
  time: number
  private curSelection: Selection
  private curSelectionFor = 0
  private updated = 0
  private meta: {[name: string]: any} = Object.create(null)
  storedMarks: readonly Mark[] | null

  constructor(state: EditorState) {
    super(state.doc)
    this.time = Date.now()
    this.curSelection = state.selection
    this.storedMarks = state.storedMarks
  }

  // 选区懒映射：只有读取时才通过 mapping 增量映射
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
    return this
  }

  setMeta(key, value): this { ... }
  scrollIntoView(): this { ... }
}
```

关键设计：
1. **选区懒映射**：添加 step 后不立即重算选区，读取 `.selection` 时通过 `mapping.slice(curSelectionFor)` 增量映射，避免无用计算。
2. **元数据系统**：`setMeta(key, value)` 附加事务来源信息（用户输入、粘贴、撤销等），插件据此响应。
3. **位掩码更新标记**：`UPDATED_SEL=1`、`UPDATED_MARKS=2`、`UPDATED_SCROLL=4` 高效跟踪变更项。
4. **重写 addStep**：文档变更时自动清除 storedMarks（[transaction.ts#L128-L132](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/transaction.ts#L128-L132)）。

---

## 6. EditorState.apply：状态更新流程

[EditorState.apply](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/state.ts#L118-L179)：

```typescript
applyTransaction(rootTr: Transaction): {state: EditorState, transactions: readonly Transaction[]} {
  // 1. 插件 filterTransaction 可取消事务
  if (!this.filterTransaction(rootTr)) return {state: this, transactions: []}

  let trs = [rootTr], newState = this.applyInner(rootTr), seen = null

  // 2. 插件 appendTransaction 循环（可连锁追加事务）
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

applyInner(tr: Transaction) {
  if (!tr.before.eq(this.doc))
    throw new RangeError("Applying a mismatched transaction")
  let newInstance = new EditorState(this.config)  // 全新实例
  let fields = this.config.fields
  for (let i = 0; i < fields.length; i++) {
    newInstance[field.name] = field.apply(
      tr, this[field.name], this, newInstance
    )
  }
  return newInstance
}
```

### 状态字段系统

EditorState 用 FieldDesc 管理状态字段，内置四个基础字段（[state.ts#L21-L41](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4/xyj-731-4_Tony/state/src/state.ts#L21-L41)）：

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

插件通过 `PluginSpec.state` 注册自己的字段，与内置字段完全平等。

### 不可变性保证
- `applyInner` 每次 `new EditorState(this.config)`，绝不修改旧状态
- 每个字段的 `apply` 返回新值
- Configuration（schema、插件列表）在更新间共享，不重建

> 返回 [README](../README.md)
