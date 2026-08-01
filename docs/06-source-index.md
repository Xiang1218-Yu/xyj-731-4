# 关键源码索引

本文档汇总 ProseMirror 核心模块中关键类、方法对应的本地源码位置，便于查阅。所有行号均与本地源码核对一致。

## prosemirror-model（[../model/src/](../model/src)）

| 文件 | 核心内容 |
|------|---------|
| [schema.ts](../model/src/schema.ts) | Schema、NodeType、MarkType、NodeSpec、MarkSpec、Attribute、SchemaSpec |
| [node.ts](../model/src/node.ts) | Node、TextNode（持久化节点、copy/mark/cut/eq/toJSON） |
| [mark.ts](../model/src/mark.ts) | Mark（addToSet/removeFromSet、rank 排序、excludes 互斥、sameSet） |
| [fragment.ts](../model/src/fragment.ts) | Fragment（子节点集合、文本合并、cut/append/findIndex） |
| [resolvedpos.ts](../model/src/resolvedpos.ts) | ResolvedPos（位置解析、path 结构、marks()）、NodeRange |
| [content.ts](../model/src/content.ts) | ContentMatch（内容表达式有限状态自动机） |
| [replace.ts](../model/src/replace.ts) | Slice、replace 算法、ReplaceError |
| [to_dom.ts](../model/src/to_dom.ts) | DOMOutputSpec、DOMSerializer |
| [from_dom.ts](../model/src/from_dom.ts) | DOMParser、ParseRule、TagParseRule |
| [diff.ts](../model/src/diff.ts) | findDiffStart、findDiffEnd |
| [comparedeep.ts](../model/src/comparedeep.ts) | compareDeep（深比较，用于 attrs/marks 比较） |

### 关键代码位置（model）

- Schema 构造函数：[schema.ts#L595-L630](../model/src/schema.ts#L595-L630)
- NodeType.compile：[schema.ts#L234-L245](../model/src/schema.ts#L234-L245)
- NodeType.create / createChecked / createAndFill：[schema.ts#L145-L183](../model/src/schema.ts#L145-L183)
- MarkType 类：[schema.ts#L276-L346](../model/src/schema.ts#L276-L346)
- Node 类声明与 constructor（含持久化注释）：[node.ts#L10-L38](../model/src/node.ts#L10-L38)
- Node.nodeSize：[node.ts#L49-L54](../model/src/node.ts#L49-L54)
- Node.copy / mark / cut：[node.ts#L136-L155](../model/src/node.ts#L136-L155)
- TextNode 类：[node.ts#L353-L397](../model/src/node.ts#L353-L397)
- Mark.addToSet：[mark.ts#L19-L45](../model/src/mark.ts#L19-L45)
- Mark.setFrom：[mark.ts#L99-L107](../model/src/mark.ts#L99-L107)
- Mark.none：[mark.ts#L109-L110](../model/src/mark.ts#L109-L110)
- Fragment.append：[fragment.ts#L71-L83](../model/src/fragment.ts#L71-L83)
- Fragment.cut：[fragment.ts#L85-L104](../model/src/fragment.ts#L85-L104)
- Fragment.fromArray：[fragment.ts#L225-L242](../model/src/fragment.ts#L225-L242)
- Fragment.empty：[fragment.ts#L257-L260](../model/src/fragment.ts#L257-L260)
- ResolvedPos.marks：[resolvedpos.ts#L126-L152](../model/src/resolvedpos.ts#L126-L152)
- ResolvedPos.resolve：[resolvedpos.ts#L217-L233](../model/src/resolvedpos.ts#L217-L233)
- ResolvedPos.resolveCached：[resolvedpos.ts#L235-L249](../model/src/resolvedpos.ts#L235-L249)

---

## prosemirror-transform（[../transform/src/](../transform/src)）

| 文件 | 核心内容 |
|------|---------|
| [transform.ts](../transform/src/transform.ts) | Transform（step/maybeStep/addStep、链式 API） |
| [step.ts](../transform/src/step.ts) | Step 抽象类、StepResult、jsonID 注册机制 |
| [map.ts](../transform/src/map.ts) | StepMap、Mapping、MapResult（位置映射与镜像） |
| [replace_step.ts](../transform/src/replace_step.ts) | ReplaceStep、ReplaceAroundStep |
| [attr_step.ts](../transform/src/attr_step.ts) | AttrStep、DocAttrStep |
| [mark_step.ts](../transform/src/mark_step.ts) | AddMarkStep、RemoveMarkStep、AddNodeMarkStep、RemoveNodeMarkStep |
| [structure.ts](../transform/src/structure.ts) | lift、wrap、split、join、setBlockType 等结构操作 |
| [replace.ts](../transform/src/replace.ts) | replaceStep、replaceRange 等替换算法 |
| [mark.ts](../transform/src/mark.ts) | addMark、removeMark、clearIncompatible |

### 关键代码位置（transform）

- Transform 类属性与 constructor：[transform.ts#L28-L44](../transform/src/transform.ts#L28-L44)
- Transform.step / maybeStep：[transform.ts#L46-L60](../transform/src/transform.ts#L46-L60)
- Transform.addStep：[transform.ts#L88-L94](../transform/src/transform.ts#L88-L94)
- Step 抽象类：[step.ts#L7-L67](../transform/src/step.ts#L7-L67)
- StepResult 类：[step.ts#L69-L97](../transform/src/step.ts#L69-L97)
- StepMap 类与 _map：[map.ts#L68-L117](../transform/src/map.ts#L68-L117)
- Mapping 类与 appendMap：[map.ts#L166-L284](../transform/src/map.ts#L166-L284)

---

## prosemirror-state（[../state/src/](../state/src)）

| 文件 | 核心内容 |
|------|---------|
| [state.ts](../state/src/state.ts) | EditorState、FieldDesc、Configuration、applyTransaction |
| [transaction.ts](../state/src/transaction.ts) | Transaction（选区懒映射、storedMarks、meta、scrollIntoView） |
| [selection.ts](../state/src/selection.ts) | Selection、TextSelection、NodeSelection、AllSelection、SelectionBookmark |
| [plugin.ts](../state/src/plugin.ts) | Plugin、PluginSpec、StateField、PluginKey、PluginView |

### 关键代码位置（state）

- UPDATED_SEL / UPDATED_MARKS / UPDATED_SCROLL 常量：[transaction.ts#L20](../state/src/transaction.ts#L20)
- Transaction 类与 constructor：[transaction.ts#L42-L65](../state/src/transaction.ts#L42-L65)
- Transaction.selection getter / setSelection：[transaction.ts#L67-L89](../state/src/transaction.ts#L67-L89)
- Transaction.addStep：[transaction.ts#L127-L132](../state/src/transaction.ts#L127-L132)
- Transaction.setMeta / getMeta：[transaction.ts#L185-L195](../state/src/transaction.ts#L185-L195)
- baseFields 数组：[state.ts#L21-L41](../state/src/state.ts#L21-L41)
- EditorState.applyTransaction：[state.ts#L132-L168](../state/src/state.ts#L132-L168)
- EditorState.applyInner：[state.ts#L170-L179](../state/src/state.ts#L170-L179)
- EditorState.create：[state.ts#L184-L191](../state/src/state.ts#L184-L191)
- Selection 基类 constructor 与属性：[selection.ts#L10-L48](../state/src/selection.ts#L10-L48)
- Selection.findFrom：[selection.ts#L113-L130](../state/src/selection.ts#L113-L130)
- SelectionBookmark 接口：[selection.ts#L192-L204](../state/src/selection.ts#L192-L204)
- TextSelection 类：[selection.ts#L225-L305](../state/src/selection.ts#L225-L305)
- NodeSelection 类：[selection.ts#L320-L376](../state/src/selection.ts#L320-L376)
- AllSelection 类：[selection.ts#L395-L425](../state/src/selection.ts#L395-L425)
- Selection.jsonID 注册调用：[selection.ts#L307](../state/src/selection.ts#L307)、[L380](../state/src/selection.ts#L380)、[L427](../state/src/selection.ts#L427)
- Plugin 类：[plugin.ts#L68-L89](../state/src/plugin.ts#L68-L89)
- PluginSpec 接口：[plugin.ts#L5-L45](../state/src/plugin.ts#L5-L45)
- StateField 接口：[plugin.ts#L91-L115](../state/src/plugin.ts#L91-L115)
- PluginKey 类：[plugin.ts#L125-L142](../state/src/plugin.ts#L125-L142)

---

## prosemirror-view（[../view/src/](../view/src)）

view 模块负责 DOM 渲染与用户交互，核心入口为 `EditorView`，还包含 Decoration（装饰系统）、NodeView（自定义节点视图）等。由于本文聚焦文档模型与状态机制，view 层仅在架构图中展示，不展开源码分析。

> 返回 [README](../README.md)
