# ProseMirror vs Slate vs Quill 架构对比

本文从架构风格、文档模型、状态管理、变更表示等维度对比三大富文本编辑器。

## 1. 总览对比表

| 维度 | ProseMirror | Slate | Quill |
|------|-------------|-------|-------|
| **架构风格** | 严格分层、模块化、函数式 | 单体核心 + 插件架构、面向对象 | 自包含编辑器、基于 Delta |
| **文档模型** | 强 Schema 树形结构，Node + Mark 分离 | 树形 Model（Element/Text），Schema 可选 | 线性 Delta 格式（OT），扁平行模型 |
| **Schema** | **必须**定义，强类型约束，内容表达式编译为自动机 | 可选，运行时不强制校验，由插件保证 | 内置格式白名单，通过 register 扩展 |
| **状态管理** | 不可变 EditorState，Transaction 更新，结构共享 | 不可变 Value，Transforms/Operations 更新 | 可变内部状态，Delta API 修改，事件驱动 |
| **变更表示** | Step 对象数组（ReplaceStep、AttrStep 等），可序列化、反转、rebase | Operation 对象数组（insert_text、split_node 等） | Delta 操作（retain/insert/delete），基于 OT |
| **位置模型** | 扁平整数位置 + ResolvedPos 上下文，精确到 token | Path 数组（如 `[0, 1, 2]`）+ offset，层级路径 | Index + length，线性偏移 |
| **选区模型** | Selection 类层次（Text/Node/All），anchor/head，自动 mapping | Range（anchor/focus），point path+offset | Range（index/length），原生 Selection 封装 |
| **视图层** | prosemirror-view 独立包，DOM 差异化更新，NodeView 自定义渲染 | React/Vue/Solid 等框架自定义渲染（slate-react 等） | 自身管理 DOM/contenteditable，主题系统 |
| **协同编辑** | 原生支持（Step + Mapping + rebase），prosemirror-collab | 支持但需外部实现 | 原生支持（OT），核心设计目标 |
| **历史/撤销** | prosemirror-history（基于 Step invert） | slate-history（operation 快照/反转） | 内置 history 模块 |
| **扩展机制** | Plugin 系统（state field + props + filter/appendTransaction） | Plugin 接口（renderElement、onChange 等钩子） | Module 系统（注册格式、模块、主题） |
| **学习曲线** | 陡峭（概念多、显式 Schema、函数式风格） | 中等（React 友好，文档较清晰） | 平缓（API 简洁，开箱即用） |
| **灵活性** | 极高（自定义 Schema、Step、Selection、NodeView） | 高（但受限于树模型和框架渲染） | 中低（Delta 模型固定，复杂节点定制困难） |
| **适用场景** | 复杂结构化编辑（表格、嵌套块、协同）、需强约束 | 中等到复杂编辑器、React 技术栈 | 富文本评论、简单博客、快速集成 |

## 2. 核心设计哲学差异

### ProseMirror："正确性优先"

- Schema 是强制的，文档结构始终被验证
- 所有变更通过显式 Step 对象，完全可追溯
- 不可变状态 + 结构化共享，兼顾正确性和性能
- 分层解耦，model/transform/state/view 可独立使用

### Slate："React 原生"

- 文档模型更接近 React 组件树心智模型
- 不强制 Schema，给予更多自由但也需要自己保证合法性
- 视图层交给前端框架，与 React 生态无缝集成
- 变更操作更接近"命令式"Transform 调用

### Quill："简单够用"

- Delta 模型简洁优雅，适合线性富文本
- 开箱即用，API 设计简洁
- Delta 扁平行模型难以表达复杂嵌套结构（如表格内嵌套列表内嵌套代码块）
- 内部状态可变，通过事件通知外部

## 3. 文档模型对比

**ProseMirror 文档树示例**：
```
doc
└── paragraph
    ├── text "Hello " marks: [strong]
    └── text "world" marks: [strong, em]
```
- Node 表示结构（块/内联节点），Mark 表示行内格式
- Mark 不是节点，而是附着在节点上的标签
- 内容表达式（如 `"paragraph+"`）强约束子节点序列

**Slate 文档树示例**：
```javascript
{
  children: [
    { type: 'paragraph', children: [
      { text: 'Hello ', bold: true },
      { text: 'world', bold: true, italic: true }
    ]}
  ]
}
```
- Element 有 children，Text 节点有文本和格式属性
- 格式直接作为 Text 节点的属性，而非独立 Mark

**Quill Delta 示例**：
```javascript
[
  { insert: 'Hello ', attributes: { bold: true } },
  { insert: 'world', attributes: { bold: true, italic: true } },
  { insert: '\n' }
]
```
- 线性操作数组，用换行符 `\n` 分隔行
- 块级格式通过换行符的 attributes 表示

## 4. 协同编辑实现对比

- **ProseMirror**：Step 可通过 Mapping 重映射位置，配合镜像（mirroring）优化，prosemirror-collab 提供完整的 rebase 协议
- **Slate**：核心不内置 OT/CRDT，需依赖外部库（如 slate-collaborative）实现
- **Quill**：Delta 本身就是 OT 操作，rebase 算法内置于核心

> 返回 [README](../README.md)
