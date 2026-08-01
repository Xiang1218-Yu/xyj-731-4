# 06 关键代码索引

> 本章按包分类列出 ProseMirror 核心源码中的关键类、方法及其行号，所有链接可直接点击跳转到对应源码。

---

## Model 包（prosemirror-model）

源码目录：[model/src/](../model/src/)

### Schema 与类型系统

| 类/接口 | 文件 | 关键内容 |
|---------|------|----------|
| [Schema](../model/src/schema.ts#L571-L686) | schema.ts | Schema 类，构造函数编译 nodes/marks，解析 content 表达式和 marks 白名单 |
| [Schema 构造函数](../model/src/schema.ts#L595-L630) | schema.ts | 完整构造逻辑：编译 NodeType/MarkType、解析 ContentMatch DFA、处理 markSet/excluded |
| [Schema.node()](../model/src/schema.ts#L645-L657) | schema.ts | 创建节点的工厂方法，委托给 NodeType.createChecked |
| [Schema.text()](../model/src/schema.ts#L661-L664) | schema.ts | 创建文本节点 |
| [Schema.mark()](../model/src/schema.ts#L667-L670) | schema.ts | 创建 mark |
| [NodeType](../model/src/schema.ts#L59-L246) | schema.ts | 节点类型描述，享元对象，持有 contentMatch、markSet |
| [NodeType.create()](../model/src/schema.ts#L151-L154) | schema.ts | 创建 Node 实例（不校验内容） |
| [NodeType.createChecked()](../model/src/schema.ts#L159-L163) | schema.ts | 创建 Node 并校验内容合法性 |
| [NodeType.createAndFill()](../model/src/schema.ts#L171-L183) | schema.ts | 自动填充必要包裹节点后创建 |
| [NodeType.validContent()](../model/src/schema.ts#L187-L193) | schema.ts | 校验 Fragment 是否符合 contentMatch |
| [NodeType.compile()](../model/src/schema.ts#L235-L245) | schema.ts | 静态方法，批量编译 NodeSpec 为 NodeType |
| [MarkType](../model/src/schema.ts#L280-L346) | schema.ts | Mark 类型描述，持有 excluded 列表和默认实例缓存 |
| [MarkType.create()](../model/src/schema.ts#L308-L311) | schema.ts | 创建 Mark 实例（有默认值时返回缓存） |
| [gatherMarks()](../model/src/schema.ts#L688-L704) | schema.ts | 解析 mark 名称/group/"_"为 MarkType 数组 |
| [SchemaSpec](../model/src/schema.ts#L350-L368) | schema.ts | Schema 构造参数接口 |
| [NodeSpec](../model/src/schema.ts#L371-L490) | schema.ts | 节点规格接口（content/marks/group/attrs/atom 等） |
| [MarkSpec](../model/src/schema.ts#L493-L543) | schema.ts | Mark 规格接口（attrs/excludes/inclusive/spanning 等） |

### Node 与 Fragment

| 类/方法 | 文件 | 关键内容 |
|---------|------|----------|
| [Node](../model/src/node.ts#L22-L349) | node.ts | 文档树节点基类，所有属性 readonly |
| [Node.copy()](../model/src/node.ts#L138-L141) | node.ts | 结构共享的核心：content 不变时返回 this |
| [Node.mark()](../model/src/node.ts#L145-L147) | node.ts | 返回带新 marks 集合的节点副本 |
| [Node.cut()](../model/src/node.ts#L152-L155) | node.ts | 切片节点，返回指定范围的副本 |
| [Node.slice()](../model/src/node.ts#L159-L167) | node.ts | 切出 Slice（含 openStart/openEnd） |
| [Node.nodeSize](../model/src/node.ts#L54) | node.ts | 节点大小：叶子为1，非叶子为 2+content.size |
| [Node.resolve()](../model/src/node.ts#L211) | node.ts | 解析位置返回 ResolvedPos（带缓存） |
| [Node.check()](../model/src/node.ts#L304-L316) | node.ts | 递归校验文档是否符合 Schema |
| [Node.toJSON()/fromJSON()](../model/src/node.ts#L319-L348) | node.ts | JSON 序列化/反序列化 |
| [TextNode](../model/src/node.ts#L353-L397) | node.ts | 文本节点子类，持有 text 字符串 |
| [TextNode.withText()](../model/src/node.ts#L378-L381) | node.ts | 返回新文本内容的 TextNode |
| [Fragment](../model/src/fragment.ts#L10-L261) | fragment.ts | 不可变子节点集合 |
| [Fragment.append()](../model/src/fragment.ts#L73-L83) | fragment.ts | 追加 Fragment，自动合并相邻同 markup 文本节点 |
| [Fragment.cut()](../model/src/fragment.ts#L86-L104) | fragment.ts | 切片，处理跨界文本/非文本节点 |
| [Fragment.replaceChild()](../model/src/fragment.ts#L115-L122) | fragment.ts | 替换子节点返回新 Fragment |
| [Fragment.findIndex()](../model/src/fragment.ts#L193-L205) | fragment.ts | 根据位置查找子节点索引和偏移 |
| [Fragment.fromArray()](../model/src/fragment.ts#L227-L242) | fragment.ts | 从数组构建，合并相邻同 markup 文本 |
| [Fragment.from()](../model/src/fragment.ts#L248-L255) | fragment.ts | 从多种类型（Fragment/Node/Array/null）构建 |
| [Fragment.empty](../model/src/fragment.ts#L260) | fragment.ts | 空 Fragment 单例 |

### Mark

| 类/方法 | 文件 | 关键内容 |
|---------|------|----------|
| [Mark](../model/src/mark.ts#L10-L111) | mark.ts | 行内标记，值对象 |
| [Mark.addToSet()](../model/src/mark.ts#L24-L45) | mark.ts | 加入集合：处理互斥、按 rank 排序 |
| [Mark.removeFromSet()](../model/src/mark.ts#L49-L54) | mark.ts | 从集合移除 |
| [Mark.eq()](../model/src/mark.ts#L65-L68) | mark.ts | 类型相同 + attrs 深比较 |
| [Mark.sameSet()](../model/src/mark.ts#L91-L97) | mark.ts | 比较两个 mark 集合是否相同 |
| [Mark.setFrom()](../model/src/mark.ts#L101-L107) | mark.ts | 从 null/Mark/数组构建排序后的 mark 集合 |
| [Mark.none](../model/src/mark.ts#L110) | mark.ts | 空集合单例 |

### 内容匹配与位置

| 类/方法 | 文件 | 关键内容 |
|---------|------|----------|
| [ContentMatch](../model/src/content.ts#L10-L133) | content.ts | 内容表达式 DFA 自动机 |
| [ContentMatch.parse()](../model/src/content.ts#L23-L31) | content.ts | 解析表达式：TokenStream → NFA → DFA |
| [ContentMatch.matchType()](../model/src/content.ts#L35-L39) | content.ts | 匹配单个节点类型 |
| [ContentMatch.fillBefore()](../model/src/content.ts#L79-L98) | content.ts | DFS 搜索需要填充的节点 |
| [ContentMatch.findWrapping()](../model/src/content.ts#L104-L110) | content.ts | BFS 搜索包裹节点链（带缓存） |
| [ContentMatch.computeWrapping()](../model/src/content.ts#L113-L133) | content.ts | BFS 实现包裹查找 |
| [ResolvedPos](../model/src/resolvedpos.ts#L12-L250) | resolvedpos.ts | 解析后的位置，提供树路径上下文 |
| [ResolvedPos.node()](../model/src/resolvedpos.ts#L47) | resolvedpos.ts | 获取指定深度的祖先节点 |
| [ResolvedPos.start()/end()](../model/src/resolvedpos.ts#L63-L73) | resolvedpos.ts | 指定层节点的起止绝对位置 |
| [ResolvedPos.before()/after()](../model/src/resolvedpos.ts#L78-L90) | resolvedpos.ts | 指定层节点前后的位置 |
| [ResolvedPos.marks()](../model/src/resolvedpos.ts#L130-L152) | resolvedpos.ts | 获取位置处的 marks（处理 inclusive） |
| [ResolvedPos.resolve()](../model/src/resolvedpos.ts#L218-L233) | resolvedpos.ts | 静态方法：从整数位置构建 path 数组 |
| [ResolvedPos.resolveCached()](../model/src/resolvedpos.ts#L236-L249) | resolvedpos.ts | 带 WeakMap + 环形缓存的解析 |
| [NodeRange](../model/src/resolvedpos.ts#L261-L289) | resolvedpos.ts | 同一父节点内的扁平范围 |
| [Slice](../model/src/replace.ts#L13-L75) | replace.ts | 文档切片（Fragment + openStart/openEnd） |
| [Slice.empty](../model/src/replace.ts#L80) | replace.ts | 空切片单例 |

---

## Transform 包（prosemirror-transform）

源码目录：[transform/src/](../transform/src/)

### Step 体系

| 类/方法 | 文件 | 关键内容 |
|---------|------|----------|
| [Step (抽象类)](../transform/src/step.ts#L16-L67) | step.ts | 原子变更步骤抽象基类 |
| [Step.apply()](../transform/src/step.ts#L21) | step.ts | 应用步骤到文档，返回 StepResult |
| [Step.invert()](../transform/src/step.ts#L30) | step.ts | 创建反转步骤 |
| [Step.map()](../transform/src/step.ts#L35) | step.ts | 通过映射调整步骤位置 |
| [Step.merge()](../transform/src/step.ts#L40) | step.ts | 合并相邻步骤（默认返回 null） |
| [Step.jsonID()](../transform/src/step.ts#L61-L66) | step.ts | 注册步骤类型 ID 用于反序列化 |
| [StepResult](../transform/src/step.ts#L71-L97) | step.ts | 步骤应用结果（成功含 doc，失败含 message） |
| [ReplaceStep](../transform/src/replace_step.ts#L7-L88) | replace_step.ts | 替换范围的步骤 |
| [ReplaceStep.apply()](../transform/src/replace_step.ts#L28-L32) | replace_step.ts | 调用 doc.replace 执行替换 |
| [ReplaceStep.getMap()](../transform/src/replace_step.ts#L34-L36) | replace_step.ts | 返回 [from, oldSize, newSize] |
| [ReplaceStep.invert()](../transform/src/replace_step.ts#L38-L40) | replace_step.ts | 用原始内容反转 |
| [ReplaceStep.map()](../transform/src/replace_step.ts#L42-L47) | replace_step.ts | rebase 时调整 from/to |
| [ReplaceStep.merge()](../transform/src/replace_step.ts#L49-L63) | replace_step.ts | 合并相邻 ReplaceStep |
| [ReplaceAroundStep](../transform/src/replace_step.ts#L93-L168) | replace_step.ts | 保留间隙的替换（用于 wrap/lift） |
| [ReplaceAroundStep.getMap()](../transform/src/replace_step.ts#L131-L134) | replace_step.ts | 两段式 StepMap |
| [AttrStep / DocAttrStep](../transform/src/attr_step.ts) | attr_step.ts | 修改节点属性的步骤 |
| [AddMarkStep / RemoveMarkStep](../transform/src/mark_step.ts) | mark_step.ts | 添加/移除行内 mark 的步骤 |
| [AddNodeMarkStep / RemoveNodeMarkStep](../transform/src/mark_step.ts) | mark_step.ts | 添加/移除节点级 mark 的步骤 |

### 位置映射

| 类/方法 | 文件 | 关键内容 |
|---------|------|----------|
| [StepMap](../transform/src/map.ts#L72-L164) | map.ts | 单个步骤的位置映射表 |
| [StepMap._map()](../transform/src/map.ts#L98-L116) | map.ts | 核心映射算法，处理 assoc/delInfo/recover |
| [StepMap.forEach()](../transform/src/map.ts#L134-L142) | map.ts | 遍历所有变更范围 |
| [StepMap.invert()](../transform/src/map.ts#L146-L148) | map.ts | 创建逆映射 |
| [Mapping](../transform/src/map.ts#L172-L284) | map.ts | 多步骤映射管道，支持镜像 |
| [Mapping.appendMap()](../transform/src/map.ts#L203-L211) | map.ts | 追加 StepMap，可选记录镜像关系 |
| [Mapping.appendMappingInverted()](../transform/src/map.ts#L237-L242) | map.ts | 逆序追加逆映射（用于 undo） |
| [Mapping._map()](../transform/src/map.ts#L264-L283) | map.ts | 管道映射，处理 recover + mirror 跳过 |
| [Mapping.getMirror()/setMirror()](../transform/src/map.ts#L225-L234) | map.ts | 镜像关系查询/设置 |
| [MapResult](../transform/src/map.ts#L40-L66) | map.ts | 映射结果，含 deleted 信息 |
| [Mappable 接口](../transform/src/map.ts#L3-L17) | map.ts | 可映射对象接口（map + mapResult） |

### Transform

| 类/方法 | 文件 | 关键内容 |
|---------|------|----------|
| [Transform](../transform/src/transform.ts#L28-L271) | transform.ts | 文档变换构建器 |
| [Transform.step()](../transform/src/transform.ts#L48-L52) | transform.ts | 应用步骤（失败抛异常） |
| [Transform.maybeStep()](../transform/src/transform.ts#L56-L60) | transform.ts | 应用步骤（失败返回结果不抛异常） |
| [Transform.addStep()](../transform/src/transform.ts#L89-L94) | transform.ts | 记录步骤/文档/映射的核心方法 |
| [Transform.replace()](../transform/src/transform.ts#L98-L102) | transform.ts | 用 Slice 替换范围 |
| [Transform.delete()](../transform/src/transform.ts#L111-L113) | transform.ts | 删除范围 |
| [Transform.insert()](../transform/src/transform.ts#L116-L118) | transform.ts | 插入内容 |
| [Transform.split()](../transform/src/transform.ts#L243-L246) | transform.ts | 分割节点 |
| [Transform.join()](../transform/src/transform.ts#L173-L176) | transform.ts | 合并节点 |
| [Transform.wrap()](../transform/src/transform.ts#L181-L184) | transform.ts | 包裹范围 |
| [Transform.lift()](../transform/src/transform.ts#L166-L169) | transform.ts | 提升范围 |
| [Transform.setBlockType()](../transform/src/transform.ts#L188-L191) | transform.ts | 设置块类型 |
| [Transform.addMark()/removeMark()](../transform/src/transform.ts#L249-L261) | transform.ts | 添加/移除 mark |

---

## State 包（prosemirror-state）

源码目录：[state/src/](../state/src/)

### EditorState

| 类/方法 | 文件 | 关键内容 |
|---------|------|----------|
| [EditorState](../state/src/state.ts#L90-L266) | state.ts | 不可变编辑器状态容器 |
| [EditorState.apply()](../state/src/state.ts#L118-L120) | state.ts | 应用事务产生新状态 |
| [EditorState.applyInner()](../state/src/state.ts#L171-L179) | state.ts | 核心：创建新实例并逐字段 apply |
| [EditorState.applyTransaction()](../state/src/state.ts#L137-L168) | state.ts | 含插件 appendTransaction 循环 |
| [EditorState.filterTransaction()](../state/src/state.ts#L123-L130) | state.ts | 调用插件 filterTransaction 钩子 |
| [EditorState.tr (getter)](../state/src/state.ts#L182) | state.ts | 创建基于当前状态的 Transaction |
| [EditorState.create()](../state/src/state.ts#L185-L191) | state.ts | 静态工厂方法 |
| [EditorState.reconfigure()](../state/src/state.ts#L199-L210) | state.ts | 重新配置插件集 |
| [Configuration](../state/src/state.ts#L45-L61) | state.ts | 状态配置（schema + plugins + fields） |
| [FieldDesc](../state/src/state.ts#L11-L19) | state.ts | 状态字段描述符 |
| [baseFields](../state/src/state.ts#L21-L41) | state.ts | 内置字段：doc/selection/storedMarks/scrollToSelection |

### Transaction

| 类/方法 | 文件 | 关键内容 |
|---------|------|----------|
| [Transaction](../state/src/transaction.ts#L42-L215) | transaction.ts | 状态事务，继承 Transform |
| [Transaction 构造函数](../state/src/transaction.ts#L60-L65) | transaction.ts | 从 state 初始化 doc/selection/storedMarks/time |
| [Transaction.selection (getter)](../state/src/transaction.ts#L71-L77) | transaction.ts | 惰性自动映射选区 |
| [Transaction.setSelection()](../state/src/transaction.ts#L81-L89) | transaction.ts | 显式设置选区 |
| [Transaction.setStoredMarks()](../state/src/transaction.ts#L97-L101) | transaction.ts | 设置存储 marks |
| [Transaction.addStep()](../state/src/transaction.ts#L128-L132) | transaction.ts | 重写：添加步骤后清空 storedMarks |
| [Transaction.insertText()](../state/src/transaction.ts#L165-L183) | transaction.ts | 插入文本的便捷方法 |
| [Transaction.setMeta()/getMeta()](../state/src/transaction.ts#L187-L195) | transaction.ts | 元数据存取（插件通信） |
| [Transaction.scrollIntoView()](../state/src/transaction.ts#L206-L209) | transaction.ts | 标记滚动到选区 |
| [Command 类型](../state/src/transaction.ts#L18) | transaction.ts | 命令函数类型签名 |

### Selection

| 类/方法 | 文件 | 关键内容 |
|---------|------|----------|
| [Selection (抽象类)](../state/src/selection.ts#L9-L188) | selection.ts | 选区抽象基类 |
| [Selection.replace()](../state/src/selection.ts#L72-L89) | selection.ts | 用 Slice 替换选区内容 |
| [Selection.replaceWith()](../state/src/selection.ts#L93-L105) | selection.ts | 用 Node 替换选区 |
| [Selection.findFrom()](../state/src/selection.ts#L118-L130) | selection.ts | 从位置沿方向搜索有效选区 |
| [Selection.near()](../state/src/selection.ts#L135-L137) | selection.ts | 位置附近的有效选区 |
| [Selection.atStart()/atEnd()](../state/src/selection.ts#L143-L151) | selection.ts | 文档开头/结尾的选区 |
| [Selection.getBookmark()](../state/src/selection.ts#L180-L182) | selection.ts | 获取无文档依赖的书签 |
| [TextSelection](../state/src/selection.ts#L229-L305) | selection.ts | 文本选区/光标 |
| [TextSelection.$cursor](../state/src/selection.ts#L239) | selection.ts | 空选区时返回光标位置 |
| [TextSelection.map()](../state/src/selection.ts#L241-L246) | selection.ts | 文档变更后映射选区 |
| [TextSelection.between()](../state/src/selection.ts#L287-L304) | selection.ts | 容错创建文本选区 |
| [NodeSelection](../state/src/selection.ts#L325-L377) | selection.ts | 节点选区 |
| [NodeSelection.isSelectable()](../state/src/selection.ts#L373-L375) | selection.ts | 判断节点是否可选 |
| [AllSelection](../state/src/selection.ts#L399-L427) | selection.ts | 全选 |
| [SelectionRange](../state/src/selection.ts#L207-L215) | selection.ts | 选区范围（$from + $to） |
| [SelectionBookmark 接口](../state/src/selection.ts#L195-L204) | selection.ts | 书签接口 |
| [findSelectionIn()](../state/src/selection.ts#L439-L452) | selection.ts | 递归查找有效选区位置 |
| [selectionToInsertionEnd()](../state/src/selection.ts#L454-L462) | selection.ts | 替换后将选区放到插入内容末尾 |

### Plugin

| 类/方法 | 文件 | 关键内容 |
|---------|------|----------|
| [Plugin](../state/src/plugin.ts#L71-L89) | plugin.ts | 插件类 |
| [Plugin.getState()](../state/src/plugin.ts#L88) | plugin.ts | 从 EditorState 获取插件状态 |
| [PluginSpec](../state/src/plugin.ts#L7-L45) | plugin.ts | 插件规格接口 |
| [StateField](../state/src/plugin.ts#L95-L115) | plugin.ts | 插件状态字段接口（init/apply/toJSON/fromJSON） |
| [PluginKey](../state/src/plugin.ts#L129-L142) | plugin.ts | 插件键，用于按 key 查找插件和状态 |

---

## View 包（prosemirror-view）

源码目录：[view/src/](../view/src/)

| 类/方法 | 文件 | 关键内容 |
|---------|------|----------|
| [EditorView](../view/src/index.ts#L30-L509) | index.ts | 编辑器视图主类 |
| [EditorView 构造函数](../view/src/index.ts#L69-L93) | index.ts | 创建 DOM、docView、domObserver、input |
| [EditorView.update()](../view/src/index.ts#L125-L134) | index.ts | 更新 props |
| [EditorView.updateStateInner()](../view/src/index.ts#L153-L234) | index.ts | 核心状态更新逻辑（DOM/选区/插件视图） |
| [EditorView.dispatch()](../view/src/index.ts#L511-L515) | index.ts | 默认 dispatch：state.apply + updateState |
| [EditorView.someProp()](../view/src/index.ts#L295-L315) | index.ts | 遍历 props/directPlugins/state plugins 查找 prop |
| [EditorView.focus()](../view/src/index.ts#L338-L343) | index.ts | 聚焦编辑器 |
| [EditorView.posAtCoords()](../view/src/index.ts#L374-L376) | index.ts | 坐标转文档位置 |
| [EditorView.coordsAtPos()](../view/src/index.ts#L384-L386) | index.ts | 文档位置转坐标 |
| [EditorView.domAtPos()](../view/src/index.ts#L396-L398) | index.ts | 文档位置转 DOM 位置 |
| [EditorView.posAtDOM()](../view/src/index.ts#L421-L425) | index.ts | DOM 位置转文档位置 |
| [EditorView.destroy()](../view/src/index.ts#L461-L474) | index.ts | 销毁视图 |
| [EditorProps](../view/src/index.ts#L621-L801) | index.ts | 编辑器属性配置接口（事件/粘贴/节点视图等） |
| [DirectEditorProps](../view/src/index.ts#L805-L826) | index.ts | 直接传给 View 的 props（含 state/dispatchTransaction） |

---

## 扩展包速览

| 包 | 目录 | 核心职责 |
|----|------|----------|
| [keymap](../keymap/src/) | keymap/ | 键位绑定 |
| [inputrules](../inputrules/src/) | inputrules/ | 输入规则（如 Markdown 快捷输入） |
| [history](../history/src/) | history/ | 撤销/重做（基于 Transaction 反转） |
| [collab](../collab/src/) | collab/ | 协作编辑（Step rebase） |
| [commands](../commands/src/) | commands/ | 常用编辑命令 |
| [gapcursor](../gapcursor/src/) | gapcursor/ | 间隙光标（在不可编辑节点间定位） |
| [dropcursor](../dropcursor/src/) | dropcursor/ | 拖拽时的落点指示 |
| [schema-basic](../schema-basic/src/) | schema-basic/ | 基础 Schema（paragraph/heading/列表等） |
| [schema-list](../schema-list/src/) | schema-list/ | 列表 Schema 和命令 |
| [menu](../menu/src/) | menu/ | 菜单栏 UI |
| [markdown](../markdown/src/) | markdown/ | Markdown 序列化/解析 |
| [changeset](../changeset/src/) | changeset/ | 变更追踪/审阅 |
| [search](../search/src/) | search/ | 搜索替换 |

---

← 返回 [05 架构对比](05-comparison.md) | 返回 [主 README](../README.md)
