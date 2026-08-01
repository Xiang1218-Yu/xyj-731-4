# Plugin 插件系统与模块依赖

> 对应本地源码：[plugin.ts](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/plugin.ts)

## 1. Plugin 核心设计

Plugin 是 ProseMirror 扩展机制的核心（[plugin.ts#L71-L89](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/plugin.ts#L71-L89)）：

```typescript
export class Plugin<PluginState = any> {
  constructor(readonly spec: PluginSpec<PluginState>) {
    if (spec.props) bindProps(spec.props, this, this.props)
    this.key = spec.key ? spec.key.key : createKey("plugin")
  }
  readonly props: EditorProps = {}
  key: string
  getState(state: EditorState): PluginState | undefined { return state[this.key] }
}
```

PluginSpec 支持的能力（[plugin.ts#L7-L45](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Tony/state/src/plugin.ts#L7-L45)）：

| 配置项 | 作用 |
|--------|------|
| `props` | 注入编辑器 props（事件处理、DOM 序列化等） |
| `state` | 定义插件自有状态字段（init/apply/toJSON/fromJSON） |
| `key` | PluginKey，唯一标识和检索插件 |
| `view` | 返回 PluginView，视图创建时初始化（可访问 DOM） |
| `filterTransaction` | 事务应用前过滤（返回 false 取消） |
| `appendTransaction` | 事务应用后追加新事务（自动修正/连锁响应） |

### StateField

```typescript
export interface StateField<T> {
  init: (config: EditorStateConfig, instance: EditorState) => T
  apply: (tr: Transaction, value: T, oldState: EditorState, newState: EditorState) => T
  toJSON?: (value: T) => any
  fromJSON?: (config: EditorStateConfig, value: any, state: EditorState) => T
}
```

插件状态字段与内置字段完全平等地参与状态更新，这使得每个插件都是编辑器状态的"一等公民"。

---

## 2. 核心模块依赖关系

根据各包 `package.json` 的依赖声明：

| 模块 | 版本 | 依赖 | 职责 |
|------|------|------|------|
| **model** | 1.25.x | `orderedmap` | 文档模型：Schema、Node、Mark、Fragment、Slice、DOM 解析/序列化 |
| **transform** | 1.12.x | `prosemirror-model` | 文档转换：Step、Transform、StepMap、Mapping、结构操作 |
| **state** | 1.4.x | `prosemirror-model`, `prosemirror-transform`, `prosemirror-view`（仅类型） | 编辑器状态：EditorState、Transaction、Selection、Plugin |
| **view** | 1.42.x | `prosemirror-model`, `prosemirror-state`, `prosemirror-transform` | 视图层：DOM 渲染、事件处理、Decoration、NodeView |

> `state` 对 `view` 的依赖仅为 `import type`，不构成运行时循环依赖。运行时依赖方向为 view → state → transform → model，形成清晰的单向链。

![核心架构分层图](architecture-layered.svg)

---

## 3. 数据流转全景

![数据流转图](architecture-dataflow.svg)

1. 用户交互或命令创建 Transaction
2. Transaction 内部累积 Step 数组，并维护选区的懒映射
3. `state.apply(tr)` 依次执行 filterTransaction、applyInner、appendTransaction 循环
4. applyInner 为每个状态字段调用 apply，生成全新 EditorState
5. EditorView 收到新状态后进行 DOM 差异化更新

> 返回 [README](../README.md)
