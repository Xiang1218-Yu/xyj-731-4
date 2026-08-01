# 关键源码索引

本文档汇总 ProseMirror 核心模块中关键类、方法对应的本地源码位置，便于查阅。

## prosemirror-model（[model/src/](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src)）

| 文件 | 核心内容 |
|------|---------|
| [schema.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/schema.ts) | Schema、NodeType、MarkType、NodeSpec、MarkSpec、Attribute、SchemaSpec |
| [node.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/node.ts) | Node、TextNode（持久化节点、copy/mark/cut/eq/toJSON） |
| [mark.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/mark.ts) | Mark（addToSet/removeFromSet、rank 排序、excludes 互斥、sameSet） |
| [fragment.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/fragment.ts) | Fragment（子节点集合、文本合并、cut/append/findIndex） |
| [resolvedpos.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/resolvedpos.ts) | ResolvedPos（位置解析、path 结构、marks()）、NodeRange |
| [content.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/content.ts) | ContentMatch（内容表达式有限状态自动机） |
| [replace.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/replace.ts) | Slice、replace 算法、ReplaceError |
| [to_dom.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/to_dom.ts) | DOMOutputSpec、DOMSerializer |
| [from_dom.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/from_dom.ts) | DOMParser、ParseRule、TagParseRule |
| [diff.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/diff.ts) | findDiffStart、findDiffEnd |
| [comparedeep.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/comparedeep.ts) | compareDeep（深比较，用于 attrs/marks 比较） |

### 关键代码位置（model）

- Schema 构造函数：[schema.ts#L595-L630](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/schema.ts#L595-L630)
- NodeType.compile：[schema.ts#L235-L245](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/schema.ts#L235-L245)
- NodeType.create/createAndFill：[schema.ts#L151-L183](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/schema.ts#L151-L183)
- Node 类定义与持久化注释：[node.ts#L10-L38](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/node.ts#L10-L38)
- Node.copy/mark/cut：[node.ts#L138-L155](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/node.ts#L138-L155)
- Node.nodeSize：[node.ts#L49-L54](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/node.ts#L49-L54)
- TextNode 类：[node.ts#L353-L397](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/node.ts#L353-L397)
- Mark.addToSet：[mark.ts#L24-L45](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/mark.ts#L24-L45)
- Mark.setFrom（rank 排序）：[mark.ts#L101-L107](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/mark.ts#L101-L107)
- Fragment.cut/append：[fragment.ts#L73-L104](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/fragment.ts#L73-L104)
- Fragment.fromArray（文本合并）：[fragment.ts#L227-L242](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/fragment.ts#L227-L242)
- ResolvedPos.resolve：[resolvedpos.ts#L218-L233](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/resolvedpos.ts#L218-L233)
- ResolvedPos.marks（inclusive 处理）：[resolvedpos.ts#L130-L152](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/model/src/resolvedpos.ts#L130-L152)

---

## prosemirror-transform（[transform/src/](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src)）

| 文件 | 核心内容 |
|------|---------|
| [transform.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/transform.ts) | Transform（step/maybeStep/addStep、链式 API） |
| [step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/step.ts) | Step 抽象类、StepResult、jsonID 注册机制 |
| [map.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/map.ts) | StepMap、Mapping、MapResult（位置映射与镜像） |
| [replace_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/replace_step.ts) | ReplaceStep、ReplaceAroundStep |
| [attr_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/attr_step.ts) | AttrStep、DocAttrStep |
| [mark_step.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/mark_step.ts) | AddMarkStep、RemoveMarkStep、AddNodeMarkStep、RemoveNodeMarkStep |
| [structure.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/structure.ts) | lift、wrap、split、join、setBlockType 等结构操作 |
| [replace.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/replace.ts) | replaceStep、replaceRange 等替换算法 |
| [mark.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/mark.ts) | addMark、removeMark、clearIncompatible |

### 关键代码位置（transform）

- Transform 类与 addStep：[transform.ts#L28-L94](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/transform.ts#L28-L94)
- Step 抽象类：[step.ts#L16-L67](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/step.ts#L16-L67)
- StepResult：[step.ts#L71-L97](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/step.ts#L71-L97)
- StepMap 构造与 map：[map.ts#L72-L116](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/map.ts#L72-L116)
- Mapping 类与镜像：[map.ts#L172-L284](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/transform/src/map.ts#L172-L284)

---

## prosemirror-state（[state/src/](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src)）

| 文件 | 核心内容 |
|------|---------|
| [state.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/state.ts) | EditorState、FieldDesc、Configuration、applyTransaction |
| [transaction.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/transaction.ts) | Transaction（选区懒映射、storedMarks、meta、scrollIntoView） |
| [selection.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts) | Selection、TextSelection、NodeSelection、AllSelection、SelectionBookmark |
| [plugin.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/plugin.ts) | Plugin、PluginSpec、StateField、PluginKey、PluginView |

### 关键代码位置（state）

- EditorState.applyTransaction：[state.ts#L137-L168](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/state.ts#L137-L168)
- EditorState.applyInner（生成新实例）：[state.ts#L171-L179](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/state.ts#L171-L179)
- baseFields 定义：[state.ts#L21-L41](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/state.ts#L21-L41)
- EditorState.create：[state.ts#L185-L191](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/state.ts#L185-L191)
- Transaction 类与选区懒映射：[transaction.ts#L42-L89](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/transaction.ts#L42-L89)
- Transaction.setMeta/scrollIntoView：[transaction.ts#L187-L214](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/transaction.ts#L187-L214)
- Selection 基类：[selection.ts#L9-L188](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L9-L188)
- TextSelection：[selection.ts#L229-L305](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L229-L305)
- NodeSelection：[selection.ts#L325-L377](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L325-L377)
- AllSelection：[selection.ts#L399-L425](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L399-L425)
- Selection.findFrom（容错搜索）：[selection.ts#L118-L130](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/selection.ts#L118-L130)
- Plugin 类：[plugin.ts#L71-L89](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/plugin.ts#L71-L89)
- PluginSpec/StateField：[plugin.ts#L7-L45](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/plugin.ts#L7-L45)、[plugin.ts#L95-L115](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/plugin.ts#L95-L115)

---

## prosemirror-view（[view/src/](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/view/src)）

view 模块负责 DOM 渲染与用户交互，核心入口为 `EditorView`，还包含 Decoration（装饰系统）、NodeView（自定义节点视图）等。由于本文聚焦文档模型与状态机制，view 层仅在架构图中展示，不展开源码分析。

> 返回 [README](../README.md)
