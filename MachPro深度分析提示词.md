# MachPro 代码库深度分析提示词

## 角色设定

你是一位精通跨端渲染框架底层原理的资深前端架构师，对 React 内部机制、VDOM diff 算法、编译时优化策略、Native 渲染引擎交互有深刻理解。你需要对 MachPro 这个全页面动态化框架的代码库进行全面、深入的源码级分析，输出一份兼具「系统化学习笔记」和「面试素材」双重用途的技术文档。

## 项目背景

MachPro 是美团外卖核心流量区的**全页面动态化解决方案**，不是传统的 WebView 容器，而是一套自研的跨端渲染框架。它的核心思路是：**开发者用 React/JSX 语法编写代码，通过编译时转换 + 多套运行时引擎，同时输出到 Native（iOS/Android/HarmonyOS）、微信小程序、H5 三端运行**。

关键技术特征（已从源码确认）：
- **双引擎架构**：Native 端使用自研 React 运行时（mach-pro-core），H5/小程序端使用深度定制的 Preact fork
- **编译时优化**：通过 babel-plugin-mach-pro 在编译阶段将 JSX 转换为直接 DOM 操作（类似 SolidJS），而非运行时 VDOM diff
- **Native DOM 模拟**：mach-pro-render 在客户端 JS 引擎中模拟了完整的 W3C DOM API 子集
- **客户端通信**：通过全局 Mach 对象的 requireModule 机制调用 Native 能力
- **Monorepo 架构**：Lerna + Yarn，核心包 20+，构建工具基于 Rollup + Vite

## 代码库结构概览

```
packages/
├── mach-pro-core/              # 自研 React 运行时（Native 端核心引擎）
├── mach-pro-render/            # Native 端 W3C DOM API 模拟层
├── preact/                     # 深度定制的 Preact fork（H5/小程序引擎）
├── mach-pro-react-build/       # 三端构建系统（Rollup + Vite）
├── mach-pro-bridge-adaptor/    # Native Module 调用适配层
├── mach-pro-native-element/    # Native 标签 → Web 标签映射
├── mach-pro-store/             # 自研状态管理（类 Rematch）
├── mach-pro-components/        # 跨端组件库
├── mach-pro-h5-element/        # H5 端自定义元素适配
├── mach-pro-h5-native-module/  # H5 端 Native Module 模拟
├── mach-pro-knb/               # KNB JSBridge 封装
├── mach-pro-knb-adaptor/       # KNB 接口适配
├── mach-pro-worker/            # Native 端 Worker 线程支持
├── mach-pro-metric-reporter/   # 性能埋点上报
├── mach-pro-update/            # 热更新模块
├── mach-pro-s3/                # 静态资源上传（MOS S3）
├── mach-pro-native-devtools/   # Native 端调试工具
├── mach_pro_tools_qjsc/        # QuickJS 字节码预编译
├── wechat-standard-components/ # 微信小程序标准组件
└── types/                      # 类型定义
```

## 分析框架

请按照以下结构逐一展开，每个部分都要求有源码引用和原理说明：

---

### 一、全局架构与设计哲学

1. **架构全景图**：绘制 MachPro 的整体分层架构图（Mermaid 语法），标注以下层次及职责边界：
   - 业务代码层（JSX/React 语法）
   - 编译转换层（babel-plugin-mach-pro）
   - 运行时引擎层（mach-pro-core vs preact fork）
   - DOM 抽象层（mach-pro-render vs 浏览器 DOM vs 小程序渲染器）
   - 平台适配层（bridge-adaptor, native-element, h5-element 等）
   - 客户端 Native 层（Mach 容器、JS 引擎、原生渲染引擎）

2. **双引擎决策**：为什么要同时维护两套渲染引擎？`MACH_ENV` 环境变量（`react`/`wechat`/`h5`）如何在编译时切换引擎？两套引擎的能力边界和性能差异分别是什么？

3. **与同类方案的定位对比**：MachPro vs React Native vs Weex vs Flutter vs 小程序原生的架构差异，MachPro 选择"模拟 DOM API + 编译时优化"这条路线的核心考量是什么？

4. **核心模块依赖拓扑**：各 package 之间的依赖关系图（Mermaid 语法），区分运行时依赖和构建时依赖

---

### 二、编译时优化系统（babel-plugin-mach-pro）

这是 MachPro 区别于传统 React 应用的**最核心技术创新**，重点分析：

1. **JSX 编译策略**：传统 React 将 JSX 编译为 `React.createElement` 调用，然后运行时做 VDOM diff。MachPro 的 babel 插件将 JSX 编译为什么形态？与 SolidJS 的编译策略有何异同？

2. **Fragment 静态模板提取**：编译器如何识别"静态 JSX 结构"并提取为 `fragment`？fragment 在运行时如何避免重复 DOM 创建？给出编译前 → 编译后的代码对照示例

3. **Block 条件渲染**：动态内容（条件渲染、列表渲染）如何在编译时标记为 `block`？block 和 fragment 的关系是什么？

4. **编译产物分析**：拿一个具体的 JSX 组件，展示编译前后的代码对比，逐行解释编译产物中每个函数调用（`i`/`l`/`a`/`s`/`t`/`c`）的含义

5. **编译时 vs 运行时的职责划分**：哪些工作在编译期完成？哪些留到运行时？这种划分的性能收益分析

---

### 三、mach-pro-core —— 自研 React 运行时（核心重点）

这个包完全自研了精简版 React 运行时，需要逐模块深入：

#### 3.1 组件系统
1. **Component 基类**（`src/react/Class/Component.ts`）：生命周期实现（componentWillMount → componentDidMount → shouldComponentUpdate → componentWillUpdate → componentDidUpdate → componentWillUnmount）与标准 React 的差异
2. **setState 批量更新**：源码中采用了什么批量更新策略？与 React 的 Transaction 模式、Vue 的 nextTick 模式分别做对比
3. **FunctionComponent 包装**（`src/react/Class/FunctionComponent.ts`）：函数组件如何被包装为类组件？`withContext` 提供的 hooks 上下文栈工作原理

#### 3.2 Hooks 实现
1. **Hooks 上下文管理**：`hooksContextStack` 全局栈 + `useCounter()` 的设计，与 React Fiber 中 hooks 链表方案的对比
2. **useState / useReducer**：状态更新如何触发重渲染？dispatcher 机制
3. **useEffect / useLayoutEffect**：副作用的收集、执行时机、清理机制
4. **useMemo / useRef**：依赖比较策略

#### 3.3 DOM 操作层
1. **核心 DOM 函数**（`src/dom/dom.ts`）：`insert`、`listen`、`attr`、`style`、`temp`、`spread` 每个函数的实现原理
2. **单字母别名优化**：`i`/`l`/`a`/`s`/`t`/`c` 的映射关系，这种优化能减少多少产物体积？
3. **render 入口**（`src/dom/render.ts`）：`render()` 和 `destroyApp()` 的完整流程

#### 3.4 列表 Diff 算法
1. **array.ts 算法分析**：这个自研的列表 diff 算法具体是什么策略？双指针 + LIS（最长递增子序列）的实现细节
2. **与 React reconciliation 的对比**：为什么不用 fiber + key 的 diff 方案？性能对比和适用场景分析

#### 3.5 元素处理管线
1. **三种核心元素类型**：`component`、`fragment`、`block` 各自的处理流程
2. **处理管线的数据流**：从 createElement 到实际 DOM 操作的完整链路

---

### 四、mach-pro-render —— Native DOM 模拟层

1. **Window 模拟**（`src/window.ts`）：
   - 如何从 `Mach.env` 获取屏幕尺寸并模拟 `window.outerWidth/Height`？
   - `location` 对象如何解析 `imeituan://` scheme？
   - `setTimeout/setInterval/requestAnimationFrame` 如何映射到 Native 实现？

2. **Document 模拟**（`src/document.ts`）：
   - `createElement`、`createTextNode`、`createEvent` 的实现
   - 与真实浏览器 DOM API 的差异点

3. **Element 类 —— JS 与 Native 的桥梁**（`src/node/element.ts`）：
   - **这是整个框架最关键的桥接层**。Element 如何继承客户端注入的 Node 基类？
   - `super.insertBefore`、`super.appendChild` 等调用如何传递到 Native 渲染引擎？
   - JS 侧维护的 `childNodes`/`attributes`/`listeners` 与 Native 侧的同步机制
   - 事件系统：Native 事件如何冒泡到 JS 层？JS 事件监听如何注册到 Native？

4. **Style 代理**（`src/node/style.ts`）：CSS 属性如何同步到 Native 渲染引擎？样式单位（px/dp/rpx/vw）转换逻辑

5. **Mach 全局对象扩展**（`src/mach.ts`）：
   - 事件系统：`Mach.on/off/receiveEvent`
   - 模块系统：`Mach.exportModule/callModule`
   - 异步 Bundle 加载：`Mach.requireBundleAsync`
   - 样式转换：`Mach.styleTransform`

6. **元素垫片**（`src/elements-shim/`）：标签映射规则（如 `scroll-view` → `scroller` + `content` 嵌套）

---

### 五、Preact Fork —— H5/小程序端引擎

1. **为什么 Fork Preact**：哪些核心改动无法通过配置或插件实现，必须修改源码？逐一列举主要改动点

2. **跨平台适配注入**：
   - `diff/props.js` 中的 `setProperty`：H5 模式下 Object 属性的 JSON.stringify 处理
   - `diff/style.js`：Native 端 `backgroundImage` gradient → `backgroundColor` 的映射逻辑
   - `compat/src/index.js`：小程序模式下用 `wx.nextTick` 包裹 `options._commit` 回调

3. **React 17 兼容层**（`compat/`）：如何实现版本号伪装 `"17.0.2"`？兼容了哪些 React API？有哪些未覆盖的 API？

4. **体积优化策略**：IIFE 格式打包 + 全局变量 `Preact` + globals 映射的设计

---

### 六、构建系统（mach-pro-react-build）

1. **三端构建管线**：
   - `MACH_ENV=react`（Native）：Rollup 打包 → IIFE 格式 → prepend 注入基建代码
   - `MACH_ENV=wechat`（小程序）：Rollup 打包 → 微信模板处理 → 包裹 createApp
   - `MACH_ENV=h5`（Web）：Vite 打包

2. **代码注入机制**（`rollup-plugin-injectjs.ts`）—— 构建的核心插件：
   - Native 产物组成 = MachBaseCode + NativeBaseCode + PreactCode + NativeElementCode + 业务代码
   - 小程序产物组成 = MPBaseCode + PreactCode + 业务代码（包裹在 createApp 中）
   - 每层注入代码的职责是什么？注入顺序为什么重要？

3. **关键 Babel/Rollup 插件链**：CSS Module、自动埋点、死代码消除、px 单位转换、资源路径处理、多态协议等插件的协作流程

4. **最终产物结构**：bundle.js + bundle.css + assets → zip → CDN 的完整发布链路

---

### 七、跨端适配体系

1. **Native Module 适配**（`mach-pro-bridge-adaptor`）：
   - Proxy 劫持 `Mach.requireModule` 的实现原理
   - 适配的核心 Native Module 清单（WMNetwork/WMRouter/WMABTest/WMStorage 等）及每个模块的作用
   - HarmonyOS 适配策略（`Mach.env.isHarmony` 分支）

2. **标签适配体系**：
   - `mach-pro-native-element`：如何 monkeypatch `React.createElement` 实现标签映射？
   - `mach-pro-h5-element`：H5 端如何用 Web Custom Elements 模拟 `view`/`text`/`image`？
   - `mach-pro-render/elements-shim`：Native 端标签垫片
   - 三套适配方案的统一抽象思路

3. **小程序适配**：`miniprogram-element` + `miniprogram-render` + `mach-pro-wechat-native-module` 如何在小程序环境中运行 React 代码？

---

### 八、状态管理（mach-pro-store）

1. **自研 vs 使用开源**：为什么声明依赖了 Redux 但实际自己实现？自研方案相比直接使用 Rematch/Redux Toolkit 的优劣
2. **核心实现**（`rematch.ts`）：`createStore` 的 rootState/rootReducers/rootEffects/rootSelectors 管理机制
3. **connect HOC**：store 变更 → 浅比较 → 决定是否 rerender 的完整链路
4. **全局单例**：`globalThis.__MACH_PRO_STORE__` 的跨包单例保证机制

---

### 九、性能优化体系

1. **编译时优化**：Fragment 静态模板、直接 DOM 操作（跳过 VDOM diff）、单字母别名压缩
2. **加载性能**：QuickJS 字节码预编译（`mach_pro_tools_qjsc`）、离线包/预加载策略
3. **运行时性能**：setState 批量更新、LIS 列表 diff、样式单位预转换
4. **Worker 线程**（`mach-pro-worker`）：`NativeWorker`/`NativeWorkerService` 如何实现计算密集型任务的线程分离？
5. **性能监控**（`mach-pro-metric-reporter`）：采集了哪些性能指标？上报机制

---

### 十、热更新与版本管理（mach-pro-update）

1. **热更新流程**：如何实现不发版的动态更新？
2. **版本管理**：多版本并存策略、灰度发布机制
3. **回滚机制**：异常时如何快速回滚到上一版本？

---

## 输出要求

### 学习笔记部分

- 每个模块先给出「一句话概括」，让我 30 秒内理解这个模块是干什么的
- 关键流程必须配有**流程图**（使用 Mermaid 语法），至少包括：
  - 一次完整的页面渲染链路（从 JSX 编译到 Native 屏幕显示）
  - 一次完整的 Bridge 调用链路（JS → Native → 回调）
  - 构建系统的三端构建流程
- 核心源码片段标注**文件路径和行号**，添加逐行注释
- 复杂概念用类比辅助理解（比如"Fragment 类似于 HTML 模板引擎的预编译模板"）
- 各模块之间的关联要交叉引用，形成知识网络

### 面试素材部分

在每个主要模块分析完成后，附加以下内容：

1. **高频面试题**：列出 3-5 个该模块相关的典型面试问题，分为「基础题」和「深度题」
2. **参考回答**：给出结构化的回答思路（问题背景 → 技术挑战 → 方案设计 → 实现细节 → 效果收益）
3. **深度追问预判**：面试官可能的追问方向及应对策略
4. **亮点话术**：可以在面试中主动展示的技术深度，用第一人称表述。示例格式：
   > "在 MachPro 的渲染引擎设计中，我们采用了编译时优化策略，将 JSX 在编译阶段直接转换为 DOM 操作指令，跳过了传统 React 运行时的 VDOM diff 过程。这种设计的灵感来源于 SolidJS，但我们针对 Native 渲染场景做了进一步适配，比如..."

### 对比分析部分

将 MachPro 的核心技术方案与业界方案进行对比，重点对比以下维度：

| 对比维度 | MachPro | React Native | Weex | Flutter | 小程序原生 |
|---|---|---|---|---|---|
| 渲染架构 | | | | | |
| JS-Native 通信 | | | | | |
| 编译策略 | | | | | |
| 性能特征 | | | | | |
| 开发体验 | | | | | |
| 包体积 | | | | | |

### 技术亮点总结

最后提炼 5-8 个 MachPro 最值得深入讨论的技术亮点，每个亮点需包含：
- 解决了什么问题
- 技术方案概述
- 与业界主流方案的差异化
- 对业务的实际价值

---

## 格式规范

- 使用清晰的 Markdown 层级标题组织内容
- 关键术语首次出现时用加粗标注并给出简要解释
- 代码片段标注文件路径和行号
- 流程图使用 Mermaid 语法
- 每个大章节末尾附一个「本章小结」，用 3-5 句话提炼核心要点

## 附录：核心源码解读清单

以下是每个核心包需要重点解读的文件、函数和关键逻辑点。分析时请逐一阅读这些文件，对标注的关键函数进行**逐行级别**的源码走读。

---

### A. mach-pro-core（自研 React 运行时，29 个源文件）

入口：`packages/mach-pro-core/src/index.ts` → 重导出 `./core` 和 `./dom/render`

#### A1. 组件系统

**文件：`src/react/Class/Component.ts`（~286 行）**
- `EventMap` 类（L8-L33）：事件回调管理器，支持 add/delete/once/call/clean，是组件生命周期回调的基础设施
- `Component<P,S>` 抽象类（L36-L269）：**重点逐行分析以下方法**：
  - `setState(state, callback)`：如何将更新推入队列？何时合并？如何触发 `nextTick` 异步刷新？
  - `forceUpdate(callback)`：与 setState 的区别，跳过 shouldComponentUpdate
  - `$init()`：首次渲染流程——调用 componentWillMount → 执行 render → 调用 componentDidMount
  - `$run()`：完整更新生命周期——shouldComponentUpdate → componentWillUpdate → render → componentDidUpdate
  - `$destroy()`：卸载清理——componentWillUnmount → 事件清理 → 子组件递归销毁
  - `$updatePropsSync(newProps)`：父组件同步更新 props 的机制（对比 React 的 reconciliation）
- `forwardRef` 函数（L273-L285）：ref 转发的工厂实现

**文件：`src/react/Class/FunctionComponent.ts`（~53 行）**
- `FunctionComponent` 类：继承 Component，核心在于 `__hooks__` 对象结构（setters/dispatchers/effects/layoutEffects/refs/memos）
- `bindComponent` 函数：在 hooks context 环境中执行 render，是函数组件与 hooks 系统的桥梁
- 生命周期映射：componentWillUpdate → 执行 layoutEffects，componentDidUpdate → 执行 effects，componentWillUnmount → cleanupEffects

**文件：`src/react/Class/PureComponent.ts`（~11 行）**
- 重写 shouldComponentUpdate 使用 shallowEqual

#### A2. Hooks 系统

**文件：`src/react/hooks/context.ts`（~23 行）——Hooks 基础设施，最先读**
- `hooksContextStack` 全局栈：维护当前正在渲染的组件
- `withContext(component, fn)`：push 组件到栈 → 执行 fn → pop 出来
- `useCounter()`：从栈顶获取当前组件并返回递增计数器，每个 hook 调用按顺序编号

**文件：`src/react/hooks/state.ts`（~61 行）**
- `useState(initialState)`：用计数器作为 key 存入 `component.state`，setter 内部调用 `component.setState`
- `useReducer(reducer, initialState, init)`：通过 reducer 函数处理 action

**文件：`src/react/hooks/effect.ts`（~58 行）**
- `useEffect` vs `useLayoutEffect`：前者注册到 `__hooks__.effects`（异步），后者注册到 `__hooks__.layoutEffects`（同步）
- `runEffects`：遍历执行 + 收集 cleanup
- `cleanupEffects`：卸载时清理
- `inputsChange`（来自 `inputs.ts`，~19 行）：依赖数组逐项严格相等比较

**文件：`src/react/hooks/memo.ts`（~27 行）**
- `useMemo`：缓存计算结果，依赖变化时重算
- `useCallback`：是 useMemo 的语法糖

**文件：`src/react/hooks/ref.ts`（~57 行）**
- `useRef`：首次创建并缓存，后续返回同一引用
- `useImperativeMethods`：类似 React 的 useImperativeHandle

#### A3. DOM 操作层——编译产物的运行时消费端

**文件：`src/dom/dom.ts`（~410 行）——最大最核心的文件**
- **`insert` / `i`**（L81-L218）：**这是整个框架最关键的函数**，负责将值（DOM 节点/数组/组件上下文/文本）插入到父节点指定位置。内部调用 `processElementDeep` 处理组件/fragment，通过 `insertExpression` 做实际 DOM 操作，使用 marker 机制管理插入位置和增量更新。必须理解 `marker` 和 `cache` 的协作机制。
- **`listen` / `l`**（L220-L238）：事件绑定，通过 events Map 实现事件替换而非重复注册
- **`attr` / `a`**（L242-L301）：属性设置，含 shallowEqual 优化和小程序/Native 的 image mode 特殊处理
- **`style` / `s`**（L332-L362）：样式设置，驼峰转连字符、自动加 px、gradient 处理
- **`temp` / `t`**（L366-L381）：从编译器生成的模板描述对象创建 DOM 树——这是理解编译产物的关键
- **`child` / `c`**（L387-L395）：通过 id 缓存子节点引用
- **`empty`**：创建占位文本节点（marker）
- **`spread`**：props 展开绑定

**文件：`src/dom/render.ts`（~24 行）**
- `render(element, container)`：将根元素 insert 到 document.body
- `destroyApp()`：销毁应用，全局注册 `globalThis.destroyReactApp`

**文件：`src/dom/array.ts`（~85 行）**
- `updateArray`：基于 key 映射的列表 diff，delta 位移比较策略决定移动方向，willMove/didMove 集合避免重复操作。注意与 React fiber reconciliation 和 Vue3 的最长递增子序列算法的区别。

**文件：`src/dom/styleUtils.ts`（~87 行）**
- `STYLE_CASE_MAP`：预定义常用 CSS 属性的驼峰→连字符映射 + 是否需要 px 单位的标记
- `styleName`/`styleUnit`：带缓存的样式转换

#### A4. 元素处理管线——连接组件系统和 DOM 操作层

**文件：`src/react/process/utils.ts`（~76 行）**
- `Type` 枚举：fragment / component / block 三种节点类型
- `createElementContext` 工厂函数：创建带 ctxMount/ctxDestroy 生命周期的上下文对象
- `destroyContext`：递归销毁上下文树

**文件：`src/react/process/element.ts`（~198 行）——渲染管线核心调度器**
- `processElementDeep`：递归处理虚拟 DOM 树，对 ElementContext 查找缓存→命中则 update，未命中则创建新实例并 mount。收集所有 component 实例用于触发生命周期回调。
- `getContext`：基于 type + key 的缓存查找/存储机制

**文件：`src/react/process/fragment.ts`（~103 行）**
- `fragment` 函数：Fragment 的 define → render → diff 逻辑。首次调用 define 初始化 DOM 模板，后续只调用 render 做差量更新。
- `f` 函数：Fragment 对应的 ElementContext 工厂，处理 mount/update/merge/destroy

**文件：`src/react/process/block.ts`（~77 行）**
- `block` 函数：创建 block 类型上下文，mount 时递归执行 body 函数，update 时重新执行并 diff

**文件：`src/react/process/component.ts`（~51 行）**
- `initComponent`：区分类组件和函数组件的实例化逻辑
- `createComponent`：创建 component 类型上下文

#### A5. 其他关键文件

**文件：`src/react/nextTick.ts`（~61 行）**
- 微任务批量更新调度器：`Promise.resolve().then()` + pending 标志 + `flushCallbacks` 批量执行

**文件：`src/utils.ts`（~64 行）**
- `shallowEqual`：PureComponent 和 hooks 依赖比较的基础
- `uid`：自增唯一 ID 生成器
- `humpToLine`：驼峰转连字符

---

### B. mach-pro-render（Native DOM 模拟层，18 个源文件）

入口：`packages/mach-pro-render/src/index.ts` → 副作用导入 `./window`

**文件：`src/window.ts`（~175 行）**
- `Location` 内部类（L8-L48）：解析三种 URL 格式——http/https（Web）、/path（小程序）、imeituan://（Native scheme）
- `Window` 类（L60-L149）：
  - 构造函数（L105-L121）：从 `Mach.env` 读 screenWidth/screenHeight/scale → 计算 outerWidth/innerWidth/devicePixelRatio；通过 `Mach.requireModule('WMRouter')` 获取路由
  - `open(url)` → `WMRouter.navigateTo`（L126-L128）
  - `close()` → `WMRouter.navigateBack`（L133-L136）
  - 定时器：setTimeout/setInterval 用 safeSetTimeout 包装（默认 delay=0）
- 全局挂载逻辑（L152-L175）：`getClassKeys` 收集 Window 所有属性 → 逐一挂到 `globalThis` → 实现 `window === globalThis`

**文件：`src/document.ts`（~75 行）**
- `TextNode` 内部类（L6-L33）：tagName='textNode'，data setter 触发父节点 computedTextContent
- `Document` 类（L35-L74）：构造函数创建 `this.body = new Element('body')`；createElement/createTextNode/createEvent/getElementById

**文件：`src/node/element.ts`（~275 行）——JS 与 Native 的桥梁，重中之重**
- `LIST` 白名单（L8-L22）：决定 dispatchEvent 时 handler 收到 Event 对象还是原始 data
- 核心方法逐一分析：
  - `superInsertBefore/superAppendChild/superRemoveChild`（L45-L55）：包装 super（Native Node）方法，过滤 textNode
  - `appendChild/insertBefore`（L67-L87）：validateTextNode → unlinkParent → super 调用 → 维护 childNodes
  - `computedTextContent`（L89-L96）：拼接子节点文本 → 设为 `content` 属性传给 Native
  - `$destory`（L101-L114）：派发 `$destory` 事件 → 递归销毁子节点 → 清空引用
  - `setAttribute`（L151-L163）：处理 data-* 属性→dataset 映射，值变化时调用 super.setAttribute 通知 Native
  - `addEventListener/removeEventListener`（L174-L185）：包装 super + 维护 listeners（每事件类型只保留最后一个 handler）
  - `dispatchEvent`（L187-L200）：创建 Event → super.dispatchEvent → 调用 listeners handler。**注意白名单机制**
  - `getBoundingClientRect`（L221-L237）：从 Mach.env 获取屏幕尺寸 + `measureInWindow()`（Native 方法）

**文件：`src/node/style.ts`（~41 行）**
- `Style` 类：代理 Native CSSStyleDeclaration，JS 侧维护缓存，仅变化时 setProperty 同步到 Native

**文件：`src/mach.ts`（~124 行）——直接修改全局 Mach 对象**
- `Mach.on(event, handler)`（L8-L23）：首次注册时调用 `Mach.subscribeEvent` 通知 Native 订阅
- `Mach.off(event, handler)`（L25-L41）：最后一个 handler 移除时 `Mach.unsubscribeEvent`
- `Mach.exportModule(name, methods)`（L48-L52）：导出 JS 模块供 Native 调用
- `Mach.callModule(name, method, params)`（L54-L69）：调用已导出的 JS 模块
- `Mach.receiveEvent(event, data)`（L71-L78）：Native 事件回调入口
- `Mach.requireBundleAsync(bundleId)`（L80-L106）：异步加载子 bundle，带 Promise 缓存
- `Mach.styleTransform(num)`（L108-L123）：px 单位缩放

**文件：`src/elements-shim/jsx.ts`（~33 行）**
- Monkey-patch `React.createElement` 和 `JSX.jsx/jsxs`：检查 ElementMap → 替换标签 → 调原始方法
- `React.h2 = oldCreateElement`：保留原始引用供绕过垫片

**文件：`src/elements-shim/ScrollView.tsx`（~25 行）**
- H5 分支：直接渲染 `<scroll-view>`
- Native 分支：`<scroller><content>{children}</content></scroller>`，根据 scrollX/scrollY 设置 flexDirection

**事件系统：`src/event/` 目录**
- `EventTarget`（~43 行）：listeners 字典 + addEventListener(支持 once) + dispatchEvent
- `Event`（~48 行）：只读 bubbles/target/type + 可写 detail
- `CustomEvent`（~9 行）：继承 Event

---

### C. Preact Fork（H5/小程序引擎，10 处核心改动）

包名：`@wmfe/mach-pro-preact`，版本伪装 `"17.0.2"`

**以下是相对于原版 Preact 的全部关键改动，请逐一对比分析：**

**改动 1：`src/diff/props.js` L112-L114** — H5 环境 Object 属性 JSON.stringify
```javascript
if (process.env.MACH_ENV === 'h5' && Object.prototype.toString.call(value) === '[object Object]') {
    value = JSON.stringify(value);
}
```

**改动 2：`src/diff/props.js` L50-L93** — 事件监听机制完全重写，用 `{value, handler}` 对象替代原版 eventProxy

**改动 3：`src/diff/style.js` L27-L37** — 三平台 gradient 适配
- react (Native)：backgroundImage 含 gradient → 转为 backgroundColor
- wechat/h5：预留逻辑（当前注释）

**改动 4：`compat/src/style.js`（整个文件新增，~54 行）** — pxTransform 单位转换体系 + 劫持 `style.setProperty` 原型方法

**改动 5：`compat/src/index.js` L167-L177** — 微信小程序 `wx.nextTick` 包裹所有 renderCallbacks

**改动 6：`src/diff/index.js` L360-L365** — createElement 第三个参数 `true` 标识 MachPro H5 创建的元素

**改动 7：`compat/src/index.js` L161-L165** — 全局注册 `destroyReactApp`

**改动 8：`compat/src/render.js` L109-L223** — className→class 处理重写，Object.assign 确保 class 在第一位

**改动 9：`src/diff/children.js` L91-L114** — 非法 VNode 对象容错处理（转为文本节点而非报错）

**改动 10：`src/diff/props.js` L109-L118** — 移除 dangerouslySetInnerHTML 判断

---

### D. mach-pro-react-build（构建系统）

**文件：`src/index.ts`（~177 行）**
- `build(params)` 总入口：根据 MACH_ENV 分发到 runBuild/runDev/vite.runBuild 等

**文件：`src/constant.ts`（~27 行）**
- `enum MACH_ENV { WECHAT, REACT, H5 }` 定义

**文件：`src/plugins/rollup-plugin-injectjs.ts`（~230 行）——构建核心插件**
- `injectNativeChunk`：MagicString prepend 注入 mach.js → mach-pro-render → preact → native-element → (devtools)
- `injectMPChunk`：注入 base-wechat.js → preact → createApp 包裹
- `injectMachConfig`：在 chunk 头部注入 `var MACH_PROJECT_CONFIG = {...}`
- 插件组合策略（L202-L229）：根据 isWorker/isSub 组合不同注入子插件

**文件：`src/plugins/rollup-plugin-sub.ts`（~92 行）**
- 子包产物处理：用 Babel AST 将 chunk 包裹为 `globalThis.__bundleCallback__ = function() { return module.exports }`

**文件：`src/rollupConfig.ts`（~185 行）**
- 完整 Rollup 插件链组装：watch → tsPaths → multilImport → postcss → replace → commonjs → resolve → typescript → assets → babel → inject → wechatTemplate → compileOutput → stat → autoMetric

**文件：`src/getConfig.ts`（~386 行）**
- `getConfig()`：核心配置聚合
- `replaceGloablEnv()`：全局环境变量替换 map（process.env.MACH_ENV, NODE_ENV 等）
- `getAlias()`：路径别名（@, @pages, @common 等）

**文件：`src/temp/base-wechat.ts`（~64 行）**
- 小程序运行时包裹模板：设置 window/document/globalThis 别名

**文件：`src/temp/base-h5.ts`（~315 行）**
- H5 Mach SDK 实现：Mach 全局对象 + NativeModule 注册 + KNB Bridge + visibilitychange

---

### E. mach-pro-bridge-adaptor（Native Module 适配）

**文件：`src/index.js`（~49 行）——Proxy 劫持核心**
- 保存原始 `Mach.requireModule` → 创建 Proxy → apply 拦截器中：
  - 检查 `globalContent` 是否包含该 module
  - 检查 `isToggleByModuleCall` 是否启用
  - 合并原始方法 + 适配器覆写方法：`{ ...Reflect.apply(target, thisArg, args), ...methods }`
- 两级开关：Bundle 级（bundleList 白名单 或 isHarmony 全量）+ Module 级

**适配模块清单（`src/adapter/` 目录）**：
| 文件 | 适配的 Native Module | 核心方法 | 底层实现 |
|------|---------------------|---------|---------|
| WMNetwork.js (~118行) | WMNetwork | request, waimaiBaseURL, getNetworkType | msi.request |
| WMRouter.js (~103行) | WMRouter | navigateTo, navigateBack, navigateToForResult | msi.openLink |
| WMABTest.js (~30行) | WMABTest | getStrategy | msi-wm-api.getWMABSync |
| WMEventCenter.js (~64行) | WMEventCenter | addListener, emit, removeListener | msi.subscribe/publish |
| WMLogin.js (~37行) | WMLogin | getUserInfo, isLogin, login | msi.mtGetUserInfoSync |
| WMStorage.js (~151行) | WMStorage | setString/getString/setBool/... (11个方法) | msi.setSharedStorageSync |
| WMToast.js (~21行) | WMToast | show | msi.showToast |
| WMStatistics.js (~117行) | WMStatistics | pv, pd, view, click, setTag, order | msi-lx-api.lxlogmsi |
| WMPay.js (~36行) | WMPay | pay | msi.mtRequestPayment |
| WMActivityIndicator.js (~38行) | WMActivityIndicator | showLoading, hideLoading | msi.showLoading |
| WMVideo.js (~35行) | WMVideo | getVolume, observeVolume | msi.getVoicePlayVolume |
| MPPreRequestModule.js (~22行) | MPPreRequestModule | readPreLoadInfo | msi.registerExtendedAPI |

---

## 开始分析

请先通读上述核心源码清单中列出的所有文件，然后按照「分析框架」章节的结构逐一展开分析。分析过程中：

1. **源码解读要求**：对标注了行号的关键函数，必须给出源码片段 + 逐行注释。不能只描述"它做了什么"，还要解释"它为什么这样做"以及"它与标准 React/Preact 的差异在哪里"
2. **编译时优化部分**：代码库中如果有 `babel-plugin-mach-pro` 包则重点分析；如果没有，则基于 `dom.ts` 中 `insert`/`attr`/`style`/`temp` 等函数的参数签名和 `process/fragment.ts` 的 define/render 结构，**反推编译器的输入输出**，给出编译前→编译后的代码对照示例
3. **交叉引用**：分析某个模块时，如果涉及到其他模块的调用，明确标注"参见第 X 章 Y 节"，形成知识网络
4. **独特设计亮点**：如发现代码库有独特的设计或可优化的空间，用 `💡 亮点` 或 `⚠️ 可优化` 标注
5. **所有源码引用标注具体文件路径**（相对于 `packages/` 目录），便于后续查证
