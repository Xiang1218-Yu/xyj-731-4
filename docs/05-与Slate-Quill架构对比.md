# 05 · ProseMirror 与 Slate、Quill 的架构差异

## 5.1 对比总表

| 维度 | **ProseMirror** | **Slate** | **Quill** |
|---|---|---|---|
| 文档模型 | 受 Schema 强约束的不可变树；块级嵌套 + 行内 Mark 平行数组 | 嵌套 JSON 树（Element/Text），约定俗成、无强 Schema 校验（normalize 靠用户规则） | Parchment 文档对应 **Delta**：扁平的「retain/insert/delete」操作序列 |
| 格式表达 | Mark 挂在节点上，rank 排序、excludes 互斥，schema 级校验 | Text 节点上的普通属性（`{bold: true}`），无集合代数 | inline attributes 键值对，语义宽松 |
| 位置系统 | 全局整数 token 位置 + ResolvedPos 深度解析 | Path（`[0,2,1]` 数组）+ Point `{path, offset}` | 扁平 index/length（纯文本偏移） |
| 不可变性 | 手写持久化结构 + 结构共享（无外部依赖） | 全面依赖 **Immer**（immutable + draft API） | Delta 不可变，但文档（Parchment blot）是可变的 |
| 变更模型 | **Step**（可序列化、可逆、可 rebase）→ Transform → Transaction；Mapping 是一等公民 | Operation（insert_text/remove_node…）+ `withoutNormalizing` | Delta 天然就是操作；OT 由 Delta 的 compose/transform 提供 |
| 状态机 | EditorState 字段集合 + apply 纯函数 + 插件事务管线（filter/append） | 编辑器对象本身可变，靠 `onChange` + React 重渲染；无事务批处理概念 | 单一 Editor 实例持有可变文档，`updateContents(delta)` |
| 协同编辑 | Step/Mapping 原生支持 rebase（prosemirror-collab） | Operation 需自行 transform（以 shareDB 集成示例为主） | Delta 的 transform 使 OT 最直接，社区方案成熟 |
| DOM 策略 | DOM 为只读投影，MutationObserver 回收输入，ViewDesc 增量 patch | React 受控渲染 contenteditable，输入靠 beforeinput 拦截 | 自有 blot 体系渲染，输入拦截与 DOM 观察混合 |
| 规范化 | Schema 内容表达式自动机在创建/替换时强制校验，结构性不可能非法 | 靠用户编写的 `normalizeNode` 事后修复 | 模型扁平，天然少结构性非法状态 |
| 扩展机制 | Plugin（StateField + 事务钩子 + 视图钩子 + props） | 高阶函数包裹 editor 对象（`withXxx`） | Module（注册式：工具栏/快捷键/剪贴板） |

## 5.2 架构哲学的核心差异

### 1. 模型形状：严格 vs 自由 vs 极简

- **Quill** 选了最简的扁平线性模型——所有内容都是一条带属性的字符序列。适合 OT、学习成本低，但深层结构（表格套表格、脚注）表达力弱。
- **Slate** 选了最自由的嵌套 JSON——上手简单，但约束责任完全转移给用户（`normalizeNode` 写不好就会产出非法文档）。
- **ProseMirror** 居中偏严：**树表达结构、Mark 表达格式、Schema 保证任何时刻文档合法**。代价是概念多（Slice 的 openStart/openEnd、ResolvedPos），收益是所有编辑算法可以假设输入合法，大幅简化。

### 2. 不可变性的实现路径

- ProseMirror **手写结构共享**，零依赖且可控——`eq` 的引用短路贯穿 diff 全链路，未变子树比较 O(1)。
- Slate 用 **Immer** 换取可变风格 API（`editor.children` 直接"改"），本质相同但多一层 Proxy 开销。
- Quill 的文档是可变的，不可变性只存在于 Delta 层面。

### 3. 变更即数据：协同能力的分水岭

三家都认识到"操作对象"对撤销/协同的价值，但深度不同：

- ProseMirror 把 Step 的**可逆性**（`invert`）和**位置映射**（`Mapping`，含 mirror/rebase）做成库内一等公民，prosemirror-collab 只需几百行就实现中心化协同。
- Slate 的 Operation 可序列化，但 transform（解决并发冲突）留给集成方。
- Quill 靠 Delta 的数学性质天然具备 OT 能力，最省事，但受限于扁平模型。

### 4. 状态管线：可干预性的差异

- ProseMirror 的 Transaction 管线（filter → apply → append）相当于 Redux middleware：插件可以**否决**、**追加**、**注解**每一次变更，且所有中间事务可回放。
- Slate 没有等价物——只能包裹 `editor.apply` 拦截 Operation，无法表达"这批操作是一个语义整体"。
- Quill 通过 Delta 的 `source` 参数区分来源（user/api/silent），粒度较粗。

## 5.3 选型速记

| 需求 | 推荐 |
|---|---|
| 严格文档结构（表格、脚注、学术写作）、可靠协同 | **ProseMirror** |
| React 技术栈、快速定制、结构自由 | **Slate** |
| 经典工具栏编辑器、最少概念负担、成熟 OT | **Quill** |

---

**上一篇**：[04 · Selection 选区系统](04-Selection选区系统.md) ｜ **下一篇**：[06 · 关键代码索引](06-关键代码索引.md)
