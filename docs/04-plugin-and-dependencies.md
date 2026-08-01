# Plugin 插件系统与模块依赖

> 对应本地源码：[plugin.ts](../state/src/plugin.ts)、[state.ts](../state/src/state.ts)

## 1. Plugin 核心设计

Plugin 是 ProseMirror 扩展机制的核心。[Plugin](../state/src/plugin.ts#L68-L89) 类定义：

```typescript
/// Plugins bundle functionality that can be added to an editor.
/// They are part of the [editor state](#state.EditorState) and
/// may influence that state and the view that contains it.
export class Plugin<PluginState = any> {
  /// Create a plugin.
  constructor(
    /// The plugin's [spec object](#state.PluginSpec).
    readonly spec: PluginSpec<PluginState>
  ) {
    if (spec.props) bindProps(spec.props, this, this.props)
    this.key = spec.key ? spec.key.key : createKey("plugin")
  }

  /// The [props](#view.EditorProps) exported by this plugin.
  readonly props: EditorProps<Plugin<PluginState>> = {}

  /// @internal
  key: string

  /// Extract the plugin's state field from an editor state.
  getState(state: EditorState): PluginState | undefined { return (state as any)[this.key] }
}
```

[PluginSpec](../state/src/plugin.ts#L5-L45) 支持的能力：

```typescript
/// This is the type passed to the [`Plugin`](#state.Plugin)
/// constructor. It provides a definition for a plugin.
export interface PluginSpec<PluginState> {
  /// The [view props](#view.EditorProps) added by this plugin. Props
  /// that are functions will be bound to have the plugin instance as
  /// their `this` binding.
  props?: EditorProps<Plugin<PluginState>>

  /// Allows a plugin to define a [state field](#state.StateField), an
  /// extra slot in the state object in which it can keep its own data.
  state?: StateField<PluginState>

  /// Can be used to make this a keyed plugin. You can have only one
  /// plugin with a given key in a given state, but it is possible to
  /// access the plugin's configuration and state through the key,
  /// without having access to the plugin instance object.
  key?: PluginKey

  /// When the plugin needs to interact with the editor view, or
  /// set something up in the DOM, use this field. The function
  /// will be called when the plugin's state is associated with an
  /// editor view.
  view?: (view: EditorView) => PluginView

  /// When present, this will be called before a transaction is
  /// applied by the state, allowing the plugin to cancel it (by
  /// returning false).
  filterTransaction?: (tr: Transaction, state: EditorState) => boolean

  /// Allows the plugin to append another transaction to be applied
  /// after the given array of transactions. When another plugin
  /// appends a transaction after this was called, it is called again
  /// with the new state and new transactions—but only the new
  /// transactions, i.e. it won't be passed transactions that it
  /// already saw.
  appendTransaction?: (transactions: readonly Transaction[], oldState: EditorState, newState: EditorState) => Transaction | null | undefined

  /// Additional properties are allowed on plugin specs, which can be
  /// read via [`Plugin.spec`](#state.Plugin.spec).
  [key: string]: any
}
```

| 配置项 | 作用 |
|--------|------|
| `props` | 注入编辑器 props（事件处理、DOM 序列化等） |
| `state` | 定义插件自有状态字段（init/apply/toJSON/fromJSON） |
| `key` | PluginKey，唯一标识和检索插件 |
| `view` | 返回 PluginView，视图创建时初始化（可访问 DOM） |
| `filterTransaction` | 事务应用前过滤（返回 false 取消） |
| `appendTransaction` | 事务应用后追加新事务（自动修正/连锁响应） |

### StateField

[StateField](../state/src/plugin.ts#L91-L115) 接口：

```typescript
/// A plugin spec may provide a state field (under its
/// [`state`](#state.PluginSpec.state) property) of this type, which
/// describes the state it wants to keep. Functions provided here are
/// always called with the plugin instance as their `this` binding.
export interface StateField<T> {
  /// Initialize the value of the field. `config` will be the object
  /// passed to [`EditorState.create`](#state.EditorState^create). Note
  /// that `instance` is a half-initialized state instance, and will
  /// not have values for plugin fields initialized after this one.
  init: (config: EditorStateConfig, instance: EditorState) => T

  /// Apply the given transaction to this state field, producing a new
  /// field value. Note that the `newState` argument is again a partially
  /// constructed state does not yet contain the state from plugins
  /// coming after this one.
  apply: (tr: Transaction, value: T, oldState: EditorState, newState: EditorState) => T

  /// Convert this field to JSON. Optional, can be left off to disable
  /// JSON serialization for the field.
  toJSON?: (value: T) => any

  /// Deserialize the JSON representation of this field. Note that the
  /// `state` argument is again a half-initialized state.
  fromJSON?: (config: EditorStateConfig, value: any, state: EditorState) => T
}
```

插件状态字段与内置字段完全平等地参与状态更新，这使得每个插件都是编辑器状态的"一等公民"。

[PluginKey](../state/src/plugin.ts#L125-L142) 提供类型安全的插件检索：

```typescript
/// A key is used to [tag](#state.PluginSpec.key) plugins in a way
/// that makes it possible to find them, given an editor state.
/// Assigning a key does mean only one plugin of that type can be
/// active in a state.
export class PluginKey<PluginState = any> {
  /// @internal
  key: string

  /// Create a plugin key.
  constructor(name = "key") { this.key = createKey(name) }

  /// Get the active plugin with this key, if any, from an editor
  /// state.
  get(state: EditorState): Plugin<PluginState> | undefined { return state.config.pluginsByKey[this.key] }

  /// Get the plugin's state from an editor state.
  getState(state: EditorState): PluginState | undefined { return (state as any)[this.key] }
}
```

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
