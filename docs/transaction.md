# Transaction 事务机制与状态不可变性

> 对应源码：`state/src/{state,transaction}.ts`、`transform/src/{transform,step,replace_step,map}.ts`

本篇回答：**ProseMirror 如何在保证状态不可变的前提下描述、追踪、应用文档变更？**

---

## 1. 不可变状态：EditorState

`EditorState` 是编辑器"当前状态"的权威快照，聚合了 doc、selection、storedMarks、插件状态等字段。它明确是**持久化数据结构**——从不原地更新，而是从旧状态计算出新状态。

```ts
// state/src/state.ts:90-120（节选）
export class EditorState {
  constructor(readonly config: Configuration) {}
  declare doc: Node
  declare selection: Selection
  declare storedMarks: readonly Mark[] | null

  apply(tr: Transaction): EditorState {
    return this.applyTransaction(tr).state
  }
}
```

### 1.1 字段化的状态（Field 机制）

状态由一组 `FieldDesc` 描述，每个字段有 `init` 和 `apply`（[state.ts:11-41](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L11-L41)）：

```ts
// state/src/state.ts:21-41 —— 内置字段
const baseFields = [
  new FieldDesc<Node>("doc", {
    init(config) { return config.doc || config.schema!.topNodeType.createAndFill() },
    apply(tr) { return tr.doc }           // 新 doc 直接取自 transaction
  }),
  new FieldDesc<Selection>("selection", {
    init(config, instance) { return config.selection || Selection.atStart(instance.doc) },
    apply(tr) { return tr.selection }     // 新 selection 取自 transaction
  }),
  new FieldDesc<readonly Mark[] | null>("storedMarks", { ... }),
  new FieldDesc<number>("scrollToSelection", { ... })
]
```

插件也能通过 `plugin.spec.state` 注册自己的字段（[state.ts:57-58](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L57-L58)），从而挂载自定义的、随事务演进的状态。

### 1.2 apply 如何产出新 state（不可变的核心）

```ts
// state/src/state.ts:171-179
applyInner(tr: Transaction) {
  if (!tr.before.eq(this.doc)) throw new RangeError("Applying a mismatched transaction")
  let newInstance = new EditorState(this.config), fields = this.config.fields
  for (let i = 0; i < fields.length; i++) {
    let field = fields[i]
    ;(newInstance as any)[field.name] = field.apply(tr, (this as any)[field.name], this, newInstance)
  }
  return newInstance     // 全新实例；this（旧 state）完全没被改动
}
```

**不可变性的三重保证：**
1. 新建 `newInstance`，逐字段重新计算，**从不写回旧实例**。
2. `tr.before.eq(this.doc)` 校验，杜绝把事务应用到不匹配的文档。
3. 每个字段的新值都是 transaction 计算出来的不可变值（doc/selection 本身也不可变）。

### 1.3 事务级联：appendTransaction

`applyTransaction` 允许插件对事务作出响应、追加新事务，形成"事务风暴"直到稳定（[state.ts:137-168](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L137-L168)）。同时 `filterTransaction` 可以整体否决一个事务（[state.ts:123-130](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L123-L130)）。这就是插件参与状态演进的官方通道。

---

## 2. Transaction = Transform + 编辑器元信息

### 2.1 继承关系

```
Transform (transform 包)        Transaction (state 包)
 ├── steps: Step[]              extends Transform
 ├── docs: Node[]               ├── selection（getter，惰性映射）
 ├── mapping: Mapping           ├── storedMarks
 ├── doc（最新文档）             ├── time / meta / updated 位标记
 ├── step() / replace() ...     └── setSelection / insertText / setMeta ...
```

```ts
// state/src/transaction.ts:42-65
export class Transaction extends Transform {
  time: number
  private curSelection: Selection
  private curSelectionFor = 0
  private updated = 0                 // 位掩码：SEL/MARKS/SCROLL
  storedMarks: readonly Mark[] | null

  constructor(state: EditorState) {
    super(state.doc)                  // Transform 以当前 doc 起步
    this.time = Date.now()
    this.curSelection = state.selection
    this.storedMarks = state.storedMarks
  }
}
```

创建入口是 `state.tr`（[state.ts:181-182](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/state.ts#L181-L182)），每次访问都 new 一个全新事务。

### 2.2 Transform：只管文档变更

```ts
// transform/src/transform.ts:28-94（节选）
export class Transform {
  readonly steps: Step[] = []
  readonly docs: Node[] = []          // docs[i] 是 steps[i] 执行前的文档
  readonly mapping: Mapping = new Mapping
  constructor(public doc: Node) {}

  step(step: Step) {                  // 应用一步，失败即抛错
    let result = this.maybeStep(step)
    if (result.failed) throw new TransformError(result.failed)
    return this
  }
  maybeStep(step: Step) {             // 尝试应用，失败则忽略
    let result = step.apply(this.doc)
    if (!result.failed) this.addStep(step, result.doc!)
    return result
  }
  addStep(step: Step, doc: Node) {    // 纯追加：保留旧文档、累积映射
    this.docs.push(this.doc)
    this.steps.push(step)
    this.mapping.appendMap(step.getMap())
    this.doc = doc
  }
}
```

**关键**：`addStep` 是纯追加操作，历史文档全部保留在 `docs[]`。文档链条 `docs[0] → docs[1] → ... → doc` 完整可回溯，这是 Undo 和协同 rebase 的数据基础。

---

## 3. Step —— 不可变性的原子

### 3.1 抽象定义

```ts
// transform/src/step.ts:16-46（节选）
export abstract class Step {
  abstract apply(doc: Node): StepResult   // 应用 → 返回新文档或失败
  getMap(): StepMap { return StepMap.empty }
  abstract invert(doc: Node): Step         // 反转 → 撤销
  abstract map(mapping: Mappable): Step | null  // 映射 → 协同 rebase
  merge(other: Step): Step | null { return null }
  abstract toJSON(): any                   // 序列化 → 网络传输/持久化
}
```

### 3.2 ReplaceStep：最核心的 Step 实现

绝大多数文档变更（插入、删除、替换）都归结为 ReplaceStep：

```ts
// transform/src/replace_step.ts:28-47（节选）
apply(doc: Node) {
  if (this.structure && contentBetween(doc, this.from, this.to))
    return StepResult.fail("Structure replace would overwrite content")
  return StepResult.fromReplace(doc, this.from, this.to, this.slice)  // 产出新 Node
}
getMap() {
  return new StepMap([this.from, this.to - this.from, this.slice.size])  // [起点,旧长,新长]
}
invert(doc: Node) {
  // 反向 step：把新插入的范围替换回原来的内容
  return new ReplaceStep(this.from, this.from + this.slice.size, doc.slice(this.from, this.to))
}
map(mapping: Mappable) {
  let to = mapping.mapResult(this.to, -1)
  // 空替换（纯插入）且 MAP_BIAS<0 时，from 复用 to 的映射结果；否则单独映射 from
  let from = this.from == this.to && ReplaceStep.MAP_BIAS < 0
      ? to : mapping.mapResult(this.from, 1)
  if (from.deletedAcross && to.deletedAcross) return null  // 两端都被删除则整步作废
  return new ReplaceStep(from.pos, Math.max(from.pos, to.pos), this.slice, this.structure)
}
// 静态默认偏置，见 replace_step.ts:85 —— static MAP_BIAS: -1 | 1 = 1
```

注意 `apply` 通过 `StepResult.fromReplace` → `doc.replace(...)` 得到**全新文档**（[step.ts:89-95](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/step.ts#L89-L95)），原文档不变——不可变性从最底层贯穿到最顶层。

### 3.3 Step 四能力的意义

| 能力 | 方法 | 支撑的上层特性 |
|------|------|---------------|
| 可应用 | `apply` | 执行编辑 |
| 可反转 | `invert` | Undo / Redo（history 包） |
| 可映射 | `map` | 协同编辑冲突消解（rebase） |
| 可序列化 | `toJSON` | 通过 `Step.jsonID` 注册（[step.ts:61-66](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/step.ts#L61-L66)），网络同步 |

---

## 4. 位置映射：StepMap 与 Mapping

文档一变，旧位置（数字）就可能失效。StepMap 负责在新旧文档间换算位置。

```ts
// transform/src/map.ts:72-95（节选）
export class StepMap implements Mappable {
  // ranges 每 3 个数为一组：[start, oldSize, newSize]
  constructor(readonly ranges: readonly number[], readonly inverted = false) { ... }
  mapResult(pos, assoc = 1): MapResult { return this._map(pos, assoc, false) as MapResult }
  map(pos, assoc = 1): number { return this._map(pos, assoc, true) as number }
}
```

`_map`（[map.ts:98-116](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/map.ts#L98-L116)）遍历变更块，累积长度差 `diff`，把旧位置平移到新位置；`assoc`（-1/1）决定位置"贴向"哪一侧，处理插入点归属问题。`MapResult.deleted` 等标志（[map.ts:40-66](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/transform/src/map.ts#L40-L66)）告知位置是否被删除。

多个 StepMap 组成 `Mapping`，`Transform.mapping` 累积所有步骤的映射，`mapping.slice(from)` 可取"某步之后"的部分映射——这是选区跟随变更的关键（见下）。

---

## 5. 事务中选区/marks 的一致性维护

### 5.1 selection 惰性映射

```ts
// state/src/transaction.ts:71-89
get selection(): Selection {
  if (this.curSelectionFor < this.steps.length) {
    // 用"尚未映射的那部分 steps"把选区推进到最新文档
    this.curSelection = this.curSelection.map(this.doc, this.mapping.slice(this.curSelectionFor))
    this.curSelectionFor = this.steps.length
  }
  return this.curSelection
}
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

每次 `addStep` 会清掉 storedMarks（[transaction.ts:128-132](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts#L128-L132)），保证输入样式随内容变更失效。`updated` 位掩码记录哪些方面被显式修改过，供 `selectionSet`/`storedMarksSet` 判断。

### 5.2 元信息（metadata）

`setMeta/getMeta`（[transaction.ts:187-195](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/state/src/transaction.ts#L187-L195)）让插件给事务贴标签（如 `"paste"`、`"pointer"`），是插件间协作与识别事务意图的机制。

---

## 6. 完整数据流图

```
 用户操作 / 命令
      │
      ▼
 const tr = state.tr                    ← 从当前 state 派生一个事务
      │
      ▼
 tr.replace(...) / tr.insertText(...)   ← 累积 Step
      │        │
      │        ├─ step.apply(doc) ─▶ 新 doc（旧 doc 不变）
      │        ├─ docs.push(旧 doc)
      │        └─ mapping.appendMap(step.getMap())
      ▼
 tr.setSelection(...) / tr.setMeta(...) ← 附加状态意图
      │
      ▼
 newState = state.apply(tr)
      │
      ├─ filterTransaction: 插件可否决
      ├─ applyInner: 校验 tr.before == doc，新建 EditorState，逐字段 apply
      └─ appendTransaction: 插件追加响应事务，循环至稳定
      │
      ▼
 newState（全新不可变状态；旧 state 仍可用，可做 undo/时间旅行）
```

---

## 7. 不可变性带来的好处

1. **可预测**：任何持有旧 state 的代码都不会被"偷偷"改变，React 等框架可靠地 `===` 比较判断是否需重渲染。
2. **可撤销**：Step 的 `invert` + 保留的 `docs[]` 让 undo/redo 变得直接。
3. **可协同**：Step 可序列化、可 `map`（rebase），是 OT 协同编辑（`prosemirror-collab`）的地基。
4. **可调试**：状态是快照序列，可做时间旅行调试。
