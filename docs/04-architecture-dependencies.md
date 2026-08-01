# 04 核心模块依赖关系图

> 本章通过 Mermaid 图表展示 ProseMirror 核心包的分层架构、包间依赖、各包内部模块依赖，以及 View 层的交互时序。

---

## 目录

- [4.1 整体分层架构](#41-整体分层架构)
- [4.2 包间依赖关系](#42-包间依赖关系)
- [4.3 Model 包内部依赖](#43-model-包内部依赖)
- [4.4 Transform 包内部依赖](#44-transform-包内部依赖)
- [4.5 State 包内部依赖](#45-state-包内部依赖)
- [4.6 View 包核心结构](#46-view-包核心结构)
- [4.7 View 层 DOM 交互时序](#47-view-层-dom-交互时序)
- [4.8 状态更新完整数据流](#48-状态更新完整数据流)

---

## 4.1 整体分层架构

```mermaid
graph TB
    subgraph "prosemirror-view"
        EditorView["EditorView<br/>DOM 渲染 / 事件处理<br/>view/src/index.ts"]
        ViewDesc["ViewDesc / NodeViewDesc<br/>视图描述树<br/>view/src/viewdesc.ts"]
        DOMObserver["DOMObserver<br/>DOM 变更观察<br/>view/src/domobserver.ts"]
        Decoration["Decoration<br/>装饰系统<br/>view/src/decoration.ts"]
        Input["InputState<br/>输入处理<br/>view/src/input.ts"]
    end

    subgraph "prosemirror-state"
        EditorState["EditorState<br/>不可变编辑器状态<br/>state/src/state.ts"]
        Transaction["Transaction<br/>状态事务<br/>state/src/transaction.ts"]
        Selection["Selection<br/>选区抽象<br/>state/src/selection.ts"]
        Plugin["Plugin<br/>插件系统<br/>state/src/plugin.ts"]
    end

    subgraph "prosemirror-transform"
        Transform["Transform<br/>文档变换构建器<br/>transform/src/transform.ts"]
        Step["Step (抽象)<br/>原子变更步骤<br/>transform/src/step.ts"]
        StepMap["StepMap / Mapping<br/>位置映射<br/>transform/src/map.ts"]
        ReplaceStep["ReplaceStep 等<br/>具体步骤实现<br/>transform/src/replace_step.ts"]
    end

    subgraph "prosemirror-model"
        Schema["Schema<br/>文档模式定义<br/>model/src/schema.ts"]
        Node["Node / TextNode<br/>文档树节点<br/>model/src/node.ts"]
        Mark["Mark<br/>行内标记<br/>model/src/mark.ts"]
        Fragment["Fragment<br/>子节点集合<br/>model/src/fragment.ts"]
        ContentMatch["ContentMatch<br/>内容表达式匹配<br/>model/src/content.ts"]
        ResolvedPos["ResolvedPos<br/>解析位置<br/>model/src/resolvedpos.ts"]
        Slice["Slice<br/>文档切片<br/>model/src/replace.ts"]
    end

    EditorView --> EditorState
    EditorView --> Input
    EditorView --> ViewDesc
    EditorView --> DOMObserver
    ViewDesc --> Decoration
    Input --> Transaction
    EditorState --> Transaction
    EditorState --> Selection
    EditorState --> Plugin
    Transaction --> Transform
    Transform --> Step
    Step --> StepMap
    Transform --> ReplaceStep
    Transform --> Node
    Step --> Node
    Selection --> Node
    Selection --> Slice
    Selection --> ResolvedPos
    Node --> Fragment
    Node --> Mark
    Node --> Schema
    Fragment --> Node
    Node --> ContentMatch
    Node --> ResolvedPos
    Schema --> Node
    Schema --> Mark
    Node --> Slice

    style EditorView fill:#4a90d9,color:#fff
    style EditorState fill:#50b87a,color:#fff
    style Transform fill:#e8a838,color:#fff
    style Schema fill:#d9604a,color:#fff
```

**分层原则：**
- 上层可以依赖下层，下层绝不依赖上层
- model 层零外部依赖（仅依赖 orderedmap 工具库）
- transform 层只依赖 model，不依赖 DOM
- state 层依赖 transform 和 model，不依赖浏览器 API
- view 层依赖 state 和 model，是唯一接触 DOM 的层

---

## 4.2 包间依赖关系

```mermaid
graph TD
    subgraph "核心包"
        VM["prosemirror-view<br/>view/"]
        ST["prosemirror-state<br/>state/"]
        TF["prosemirror-transform<br/>transform/"]
        MD["prosemirror-model<br/>model/"]
    end

    subgraph "扩展包（依赖核心）"
        KM["keymap"]
        IR["inputrules"]
        HIST["history"]
        COLLAB["collab"]
        CMD["commands"]
        GC["gapcursor"]
        DC["dropcursor"]
        SB["schema-basic"]
        SL["schema-list"]
        MENU["menu"]
        MD2["markdown"]
        CHS["changeset"]
        SRCH["search"]
        ES["example-setup"]
    end

    VM -->|"import"| ST
    VM -->|"import"| MD
    ST -->|"import"| TF
    ST -->|"import"| MD
    TF -->|"import"| MD

    KM --> ST
    IR --> ST
    HIST --> ST
    HIST --> TF
    COLLAB --> ST
    COLLAB --> TF
    CMD --> ST
    CMD --> TF
    CMD --> MD
    GC --> ST
    DC --> ST
    SB --> MD
    SL --> MD
    SL --> TF
    SL --> CMD
    MENU --> ST
    MD2 --> MD
    CHS --> TF
    CHS --> MD
    SRCH --> ST
    SRCH --> TF
    ES --> KM
    ES --> IR
    ES --> HIST
    ES --> CMD
    ES --> SB
    ES --> SL
    ES --> DC
    ES --> MENU

    style VM fill:#4a90d9,color:#fff
    style ST fill:#50b87a,color:#fff
    style TF fill:#e8a838,color:#fff
    style MD fill:#d9604a,color:#fff
```

核心包形成严格的线性依赖链：`view → state → transform → model`。这保证了底层包可以在无 DOM 环境（如 Node.js 服务端协作服务）中独立使用。

---

## 4.3 Model 包内部依赖

```mermaid
graph LR
    Index["index.ts<br/>统一导出"]
    Schema["schema.ts<br/>Schema / NodeType / MarkType"]
    Node["node.ts<br/>Node / TextNode"]
    Mark["mark.ts<br/>Mark"]
    Fragment["fragment.ts<br/>Fragment"]
    Content["content.ts<br/>ContentMatch DFA"]
    ResolvedPos["resolvedpos.ts<br/>ResolvedPos / NodeRange"]
    Replace["replace.ts<br/>Slice / ReplaceError"]
    Diff["diff.ts<br/>findDiffStart/End"]
    CompareDeep["comparedeep.ts<br/>compareDeep"]
    DOMParse["from_dom.ts<br/>DOMParser"]
    DOMSerialize["to_dom.ts<br/>DOMSerializer"]

    Index --> Schema
    Index --> Node
    Index --> Mark
    Index --> Fragment
    Index --> Replace
    Index --> ResolvedPos
    Index --> Content
    Index --> DOMParse
    Index --> DOMSerialize

    Schema --> Node
    Schema --> Fragment
    Schema --> Mark
    Schema --> Content
    Node --> Fragment
    Node --> Mark
    Node --> Replace
    Node --> ResolvedPos
    Node --> CompareDeep
    Fragment --> Node
    Fragment --> Diff
    Replace --> Fragment
    Replace --> Node
    Replace --> ResolvedPos
    ResolvedPos --> Mark
    ResolvedPos --> Node
    Mark --> CompareDeep
    DOMParse --> Schema
    DOMParse --> ResolvedPos
    DOMSerialize --> Schema

    style Schema fill:#d9604a,color:#fff
    style Node fill:#4a90d9,color:#fff
    style Mark fill:#e8a838,color:#fff
```

**关键依赖说明：**

- `schema.ts` 是核心枢纽，编译 Spec 为 Type/MarkType，并关联 ContentMatch
- `node.ts` 是最复杂的模型文件，依赖 Fragment、Mark、Replace、ResolvedPos
- `fragment.ts` 依赖 `diff.ts` 做差异查找
- `from_dom.ts` 和 `to_dom.ts` 是 DOM 序列化/解析，单向依赖 Schema
- `comparedeep.ts` 是无依赖的工具函数，用于深比较 attrs

---

## 4.4 Transform 包内部依赖

```mermaid
graph LR
    Index2["index.ts<br/>统一导出"]
    Transform["transform.ts<br/>Transform 构建器"]
    Step["step.ts<br/>Step 抽象 / StepResult"]
    Map["map.ts<br/>StepMap / Mapping / MapResult"]
    ReplaceStep["replace_step.ts<br/>ReplaceStep / ReplaceAroundStep"]
    Replace2["replace.ts<br/>replaceStep 工具函数"]
    Structure["structure.ts<br/>lift/wrap/split/join"]
    MarkT["mark.ts<br/>addMark/removeMark"]
    MarkStep["mark_step.ts<br/>AddMarkStep / RemoveMarkStep"]
    AttrStep["attr_step.ts<br/>AttrStep / DocAttrStep"]

    Index2 --> Transform
    Index2 --> Step
    Index2 --> Map
    Index2 --> ReplaceStep
    Index2 --> MarkStep
    Index2 --> AttrStep

    Transform --> Step
    Transform --> Map
    Transform --> ReplaceStep
    Transform --> MarkStep
    Transform --> AttrStep
    Transform --> Replace2
    Transform --> Structure
    Transform --> MarkT
    Step --> Map
    ReplaceStep --> Step
    ReplaceStep --> Map
    MarkStep --> Step
    AttrStep --> Step
    Structure --> Replace2
    MarkT --> Replace2
    Replace2 --> Step

    style Transform fill:#e8a838,color:#fff
    style Step fill:#50b87a,color:#fff
    style Map fill:#4a90d9,color:#fff
```

**关键依赖说明：**

- `step.ts` 定义抽象基类和注册表，依赖 `map.ts` 的 Mappable 接口
- `map.ts` 是自包含的位置映射引擎，不依赖其他 transform 文件
- `transform.ts` 是门面类，组合所有具体 Step 和工具函数
- `structure.ts`（lift/wrap/split/join）通过 `replace.ts` 的工具函数构造 ReplaceAroundStep
- 每个具体 Step 文件只依赖 `step.ts` 和 `map.ts`，彼此独立

---

## 4.5 State 包内部依赖

```mermaid
graph LR
    Index3["index.ts<br/>统一导出"]
    State["state.ts<br/>EditorState / Configuration / FieldDesc"]
    Transaction["transaction.ts<br/>Transaction"]
    Selection2["selection.ts<br/>Selection / TextSelection / ..."]
    Plugin["plugin.ts<br/>Plugin / PluginKey / StateField"]

    Index3 --> State
    Index3 --> Transaction
    Index3 --> Selection2
    Index3 --> Plugin

    State --> Transaction
    State --> Selection2
    State --> Plugin
    Transaction --> Selection2
    Transaction --> Plugin
    Selection2 --> Transaction

    style State fill:#50b87a,color:#fff
    style Transaction fill:#e8a838,color:#fff
    style Selection2 fill:#4a90d9,color:#fff
    style Plugin fill:#d9604a,color:#fff
```

**关键依赖说明：**

- `state.ts` 是核心，管理 Configuration（含 FieldDesc 列表）和 apply 逻辑
- `transaction.ts` 继承 transform 的 Transform，组合 Selection 和 Plugin（用于 setMeta 的 key）
- `selection.ts` 依赖 Transaction 类型（用于 replace/replaceWith 方法签名），但不依赖 State
- `plugin.ts` 是独立的插件描述，不依赖 state.ts，避免循环依赖

---

## 4.6 View 包核心结构

```mermaid
graph TB
    EditorView["EditorView<br/>view/src/index.ts"]
    DocView["docViewDesc<br/>NodeViewDesc 树根"]
    InputState["InputState<br/>view/src/input.ts"]
    DOMObs["DOMObserver<br/>view/src/domobserver.ts"]
    Decorations["Decoration / DecorationSet<br/>view/src/decoration.ts"]
    DOMChange["readDOMChange<br/>view/src/domchange.ts"]
    SelectionDOM["selectionToDOM<br/>view/src/selection.ts"]
    Clipboard["clipboard.ts<br/>复制/粘贴"]
    DOMCoords["domcoords.ts<br/>位置坐标转换"]

    EditorView --> DocView
    EditorView --> InputState
    EditorView --> DOMObs
    EditorView --> Decorations
    EditorView --> SelectionDOM
    EditorView --> Clipboard
    EditorView --> DOMCoords
    InputState --> DOMChange
    DOMObs --> DOMChange
    DOMChange --> DocView
    DocView --> Decorations

    style EditorView fill:#4a90d9,color:#fff
    style DocView fill:#50b87a,color:#fff
    style InputState fill:#e8a838,color:#fff
```

View 包是最大最复杂的包，负责 DOM 渲染、事件监听、选区同步、剪贴板、拖拽、IME 组合输入等。EditorView 持有：
- `docView`：文档的视图描述树（NodeViewDesc），负责增量 DOM 更新
- `input`：输入状态机，处理键盘、粘贴、拖拽等
- `domObserver`：MutationObserver 封装，检测外部 DOM 修改
- `dom`：可编辑的 DOM 根元素

---

## 4.7 View 层 DOM 交互时序

```mermaid
sequenceDiagram
    participant DOM as 浏览器 DOM
    participant Observer as DOMObserver
    participant Input as InputState
    participant View as EditorView
    participant TR as Transaction
    participant State as EditorState
    participant DocView as docView

    DOM->>Observer: mutation / beforeinput / selectionchange
    Observer->>Input: readDOMChange(from, to, typeOver, added)
    Input->>Input: 解析 DOM 变更
    Input->>Input: 构造 Step (ReplaceStep 等)
    Input->>TR: state.tr 创建事务
    Input->>TR: tr.step(step) / tr.setSelection()
    Input->>View: dispatch(tr)
    View->>View: someProp("dispatchTransaction")?
    alt 自定义 dispatchTransaction
        View->>View: 调用 props.dispatchTransaction(tr)
    else 默认行为
        View->>State: state.apply(tr)
        State->>State: filterTransaction(tr)
        State->>State: applyInner(tr)
        State-->>View: newState
        View->>View: updateStateInner(newState)
        View->>DocView: update(newDoc, outerDeco, innerDeco)
        DocView->>DocView: matchesNode? 差异比对
        alt 文档变更
            DocView->>DOM: 局部 DOM 修改
        end
        View->>DOM: selectionToDOM(view)
        View->>View: updatePluginViews(prevState)
    end
```

**关键设计：**

1. **DOMObserver** 使用 MutationObserver + 事件监听捕获 DOM 变更，不直接信任 DOM
2. **readDOMChange** 将 DOM 差异转换回 ProseMirror Step，保证单一数据源（EditorState）
3. **dispatch** 是唯一的状态更新出口，所有变更都经过 Transaction
4. **updateStateInner** 对比新旧状态，决定是否需要重绘文档、更新选区、滚动
5. **NodeViewDesc.update** 进行细粒度的 DOM 增量更新，避免全量重渲染

---

## 4.8 状态更新完整数据流

```mermaid
sequenceDiagram
    participant Plugin as Plugin(s)
    participant Cmd as Command
    participant TR as Transaction
    participant State as EditorState (old)
    participant NewState as EditorState (new)
    participant Fields as FieldDescs

    Cmd->>TR: state.tr 创建事务
    Note over TR: 继承 Transform<br/>doc = state.doc<br/>curSelection = state.selection
    Cmd->>TR: tr.replace()/tr.setSelection()/tr.setMeta()
    TR->>TR: addStep(step, newDoc)
    Note over TR: docs.push(oldDoc)<br/>steps.push(step)<br/>mapping.appendMap()<br/>doc = newDoc
    Cmd->>State: state.apply(tr)
    State->>Plugin: filterTransaction(tr, state)
    Plugin-->>State: true/false
    alt 通过过滤
        State->>NewState: new EditorState(config)
        State->>Fields: for each field:<br/>field.apply(tr, oldValue, oldState, newState)
        Fields-->>NewState: doc = tr.doc<br/>selection = tr.selection<br/>plugin states...
        State->>Plugin: appendTransaction(trs, oldState, newState)
        Plugin-->>State: appended tr or null
        Note over State: 循环直到无插件追加事务
        State-->>Cmd: {state: newState, transactions: [...]}
    else 被过滤
        State-->>Cmd: {state: this, transactions: []}
    end
```

这个流程体现了 ProseMirror 的单向数据流：**Command → Transaction → EditorState.apply → newState → View 更新**。插件可以通过 `filterTransaction` 拦截事务，通过 `appendTransaction` 追加响应事务。

---

← 返回 [03 选区系统](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Steve/docs/03-selection.md) | 返回 [主 README](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Steve/README.md) | 继续阅读 [05 架构对比 →](file:///Users/tog/Desktop/code/gsb/gsb-731/xyj-731-4/xyj-731-4_Steve/docs/05-comparison.md)
