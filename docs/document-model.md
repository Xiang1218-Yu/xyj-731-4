# 文档模型深度解析：Schema / Node / Mark

> 对应源码：`model/src/{schema,node,mark,fragment,content,resolvedpos}.ts`

ProseMirror 的文档模型回答一个根本问题：**"一个文档是什么，以及什么样的文档是合法的？"** 答案是一棵持久化不可变的树，其形态由 Schema 严格约束。

---

## 1. Schema —— 文档的类型系统

### 1.1 定位

`Schema`（[schema.ts:571-630](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L571-L630)）是整个模型的入口和工厂。它持有：

- `nodes`：名字 → `NodeType` 的映射
- `marks`：名字 → `MarkType` 的映射
- `topNodeType`：顶层节点类型（默认 `doc`）

### 1.2 构造时做了什么（关键代码）

```ts
// model/src/schema.ts:595-630（节选）
constructor(spec: SchemaSpec<Nodes, Marks>) {
  // 1. 把 spec 转成 OrderedMap（顺序很重要，决定解析优先级）
  instanceSpec.nodes = OrderedMap.from(spec.nodes)
  instanceSpec.marks = OrderedMap.from(spec.marks || {})

  // 2. 编译出唯一的 NodeType / MarkType 实例
  this.nodes = NodeType.compile(this.spec.nodes, this)
  this.marks = MarkType.compile(this.spec.marks, this)

  // 3. 把每个节点的 content 表达式编译成内容状态机 ContentMatch
  let contentExprCache = Object.create(null)
  for (let prop in this.nodes) {
    let type = this.nodes[prop], contentExpr = type.spec.content || ""
    type.contentMatch = contentExprCache[contentExpr] ||
      (contentExprCache[contentExpr] = ContentMatch.parse(contentExpr, this.nodes))
    type.inlineContent = type.contentMatch.inlineContent
    // 4. 解析该节点允许的 mark 集合
    type.markSet = markExpr == "_" ? null : markExpr ? gatherMarks(...) : ...
  }
  // 5. 解析 mark 之间的互斥关系
  for (let prop in this.marks) {
    let type = this.marks[prop], excl = type.spec.excludes
    type.excluded = excl == null ? [type] : excl == "" ? [] : gatherMarks(this, excl.split(" "))
  }
}
```

要点：
- **类型只实例化一次**（compile 阶段），因此全程可用 `===` 比较类型，无需按名字字符串比对。
- **content 表达式 → ContentMatch 状态机**：像正则一样，`"paragraph+"`、`"(paragraph | heading) block*"` 描述子节点序列的合法性。相同表达式共享同一状态机（缓存）。`ContentMatch.parse` 调用见 [schema.ts:609-610](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L609-L610)。
- **markSet 语义**：`"_"` → null（允许所有）；空串或非内联节点 → `[]`（禁止）；否则解析成具体 MarkType 数组。见 [schema.ts:617-619](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L617-L619)。

### 1.3 创建节点必经校验

`schema.node(...)` 最终走 `createChecked`，强制校验内容合法性：

```ts
// model/src/schema.ts:159-163
createChecked(attrs, content, marks) {
  content = Fragment.from(content)
  this.checkContent(content)   // 不合法直接 throw RangeError
  return new Node(this, this.computeAttrs(attrs), content, Mark.setFrom(marks))
}
```

这就是"非法结构无法被构造"的保证来源（[checkContent](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L198-L201)）。

---

## 2. NodeType 与 Node

### 2.1 NodeType —— 节点的"类"

`NodeType`（[schema.ts:59-246](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L59-L246)）描述一类节点的元信息与行为。关键属性/方法：

| 成员 | 含义 |
|------|------|
| `contentMatch` | 内容状态机，决定合法子节点序列 |
| `markSet` | 允许的 mark 集合（null = 全部） |
| `isBlock / isText / isInline` | 分类标志（[schema.ts:84-96](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L84-L96)） |
| `isTextblock` | 块级且含内联内容（如 paragraph） |
| `isLeaf` | 无内容（如 image） |
| `isAtom` | 视图中视为不可分整体 |
| `createAndFill` | 自动补齐必需子节点后创建（[schema.ts:171-183](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L171-L183)） |

`createAndFill` 尤其体现 schema 智能：它用 `contentMatch.fillBefore` 自动在内容前后补齐必需节点，是很多编辑命令能"自愈"出合法结构的原因。

### 2.2 Node —— 持久化不可变节点

```ts
// model/src/node.ts:22-38（构造）
export class Node {
  constructor(
    readonly type: NodeType,      // 类型
    readonly attrs: Attrs,        // 属性（如 heading 的 level）
    content?: Fragment | null,    // 子节点容器
    readonly marks = Mark.none    // 附着的 marks
  ) { this.content = content || Fragment.empty }
}
```

**不可变的实现方式**——所有"修改"方法都返回新对象，复用旧内容（结构共享）：

```ts
// model/src/node.ts:138-155
copy(content = null): Node {
  if (content == this.content) return this        // 内容没变直接复用自身
  return new Node(this.type, this.attrs, content, this.marks)
}
mark(marks): Node {
  return marks == this.marks ? this : new Node(this.type, this.attrs, this.content, marks)
}
cut(from, to = this.content.size): Node {
  if (from == 0 && to == this.content.size) return this
  return this.copy(this.content.cut(from, to))
}
```

**位置索引方案**（[node.ts:49-54](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/node.ts#L49-L54)）：
- 文本节点 `nodeSize` = 字符数
- 其他叶子节点 = 1
- 非叶子节点 = 内容大小 + 2（首尾各一个 token）

这套整数偏移方案让位置可以用单个数字表达，是 Step/Selection/Mapping 全部建立的坐标系。

### 2.3 Fragment —— 子节点序列

`Node.content` 是一个 `Fragment`（子节点数组的不可变包装），Node 的遍历、切割方法（`forEach`、`nodesBetween`、`textBetween` 等）大多委托给它。

---

## 3. Mark —— 内联装饰

### 3.1 为什么 Mark 独立于树结构

在 DOM 里，加粗的链接是 `<a><strong>text</strong></a>` 或反过来——嵌套顺序有歧义、难以规范化。ProseMirror 的做法：**文本节点只是携带一组 mark**，顺序由 `MarkType.rank` 唯一确定，从根本上消除歧义。

```ts
// model/src/mark.ts:10-17
export class Mark {
  constructor(
    readonly type: MarkType,   // 类型（strong/em/link...）
    readonly attrs: Attrs      // 属性（如 link 的 href）
  ) {}
}
```

### 3.2 Mark 集合运算（关键代码）

Mark 集合是有序、去重、支持互斥替换的不可变数组：

```ts
// model/src/mark.ts:24-45 —— 加入一个 mark
addToSet(set: readonly Mark[]): readonly Mark[] {
  let copy, placed = false
  for (let i = 0; i < set.length; i++) {
    let other = set[i]
    if (this.eq(other)) return set                  // 已存在，原样返回
    if (this.type.excludes(other.type)) {           // 互斥：踢掉旧的
      if (!copy) copy = set.slice(0, i)
    } else if (other.type.excludes(this.type)) {
      return set                                    // 被对方互斥，不能加
    } else {
      if (!placed && other.type.rank > this.type.rank) {  // 按 rank 有序插入
        if (!copy) copy = set.slice(0, i)
        copy.push(this); placed = true
      }
      if (copy) copy.push(other)
    }
  }
  if (!copy) copy = set.slice()
  if (!placed) copy.push(this)
  return copy
}
```

配套的 `removeFromSet` / `isInSet` / `sameSet` / `setFrom`（排序去重）见 [mark.ts:49-107](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/mark.ts#L49-L107)。

### 3.3 互斥关系（excludes）

`MarkType.excludes`（[schema.ts:343-345](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/schema.ts#L343-L345)）查询互斥表，该表在 Schema 构造时由 `excludes` spec 解析而来。默认一个 mark 只与自身同类型互斥（例如不能同时有两个不同 href 的 link），这样 `addToSet` 新 link 会替换旧 link。

---

## 4. 三者关系全景

```
             ┌──────────────────────────────────────────┐
             │                 Schema                     │
             │  (每 Schema 仅一份 NodeType/MarkType 实例) │
             └───────────────┬──────────────┬────────────┘
                   compile   │              │  compile
                   ┌─────────▼──┐        ┌──▼─────────┐
                   │  NodeType  │        │  MarkType  │
                   │ contentMatch│       │  excluded  │
                   │  markSet   │        │   rank     │
                   └─────┬──────┘        └─────┬──────┘
              create/check│  实例化              │ 实例化
                   ┌──────▼──────┐        ┌──────▼──────┐
                   │    Node     │  持有  │    Mark     │
                   │ type/attrs  │◀───────│ type/attrs  │
                   │ content     │ marks  └─────────────┘
                   │ marks ──────┼──────────┘
                   └──────┬──────┘
                          │ content
                   ┌──────▼──────┐
                   │  Fragment   │  ← 子 Node 序列
                   └─────────────┘
```

- **Schema 造 NodeType/MarkType；NodeType/MarkType 造 Node/Mark。**
- **Node 通过 `content: Fragment` 组成树，通过 `marks` 附着装饰。**
- **合法性双重校验**：结构靠 `NodeType.contentMatch`，装饰靠 `NodeType.markSet` + `MarkType.excluded`。

---

## 5. 设计要点小结

1. **不可变 + 结构共享**：改动只新建受影响路径上的节点，其余子树复用，比全量拷贝高效得多（[node.ts:14-18](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Thor/model/src/node.ts#L14-L18) 注释）。
2. **Schema 是硬约束而非建议**：非法文档在构造阶段就抛错，从源头杜绝脏数据。
3. **Mark 与结构解耦**：内联样式不污染树形结构，规避了 HTML 嵌套歧义。
4. **整数坐标系**：统一的 `nodeSize` 索引方案让位置、变更、选区都能用数字精确表达。
