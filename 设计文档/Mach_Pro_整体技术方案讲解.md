# Mach Pro 整体技术方案讲解

> **阅读指南**：这份文档把 Mach Pro 当作一个完整的系统来讲解，按照"先理解问题 → 再理解整体设计 → 再分别看前端和客户端各自做了什么 → 最后看它们怎么协作"的顺序展开。适合从零开始系统学习这个框架。

---

## 一、先搞清楚一个问题：为什么需要 Mach Pro？

外卖 App 有一个持续的痛点：**核心页面的动态化改造做不下去**。

所谓"动态化"，就是页面不随 App 发版更新，而是通过热更新推送。非核心页面早已通过 MRN（美团 React Native）或 H5 实现了动态化，但核心页面——首页、点菜页、提单页——一直没法彻底改造。原因是现有的两套方案各有硬伤。

**Mach（1.0）的问题**：它为了性能砍掉了大部分 JS 能力，只保留了数据预处理和简单交互。好处是渲染快、内存低，适合"一张卡片一个模板"的简单列表；坏处是能力边界太小，稍微复杂一点的视图和交互逻辑就做不了。想让它支持组件化、Hooks 等高阶能力，就得不断改造脚手架，陷入"永远在追赶 RN 但永远追不上"的循环。

**MRN（React Native）的问题**：能力足够强，但性能不达标。根本原因有三个：JS 引擎（JSCore/V8）执行慢（10 万行代码在低端机上需要 270~395ms）；React 的 vdom+diff 机制引入了大量不必要的计算（比手动操作 DOM 慢 86%）；JS 和 Native 运行在不同线程，每次通信需要线程切换和序列化，手势交互时明显卡顿。这三个问题叠加后，点菜页那种"菜品列表与背景墙的复杂嵌套联动动画 + 长列表高性能滑动"的场景完全扛不住。

于是团队需要一个**第三种方案**：性能接近 Native，能力接近 RN，一套代码三端复用（Android、iOS、小程序）。这就是 Mach Pro 的由来。

---

## 二、Mach Pro 的核心思路是什么？

用一句话概括：

> **通过 QuickJS 引擎向 JS 提供一套精简的 DOM API，JS 在主线程通过这套 API 同步操作 Native 的 UI 组件。**

展开来说，它做了三个关键决策来解决 MRN 的三大瓶颈：

| MRN 的瓶颈 | Mach Pro 的解法 | 效果 |
|-----------|---------------|------|
| JS 引擎执行慢 | 用 QuickJS 替代 JSCore/V8，支持字节码预编译 | 总执行时间快 3~4 倍，内存降低 75% |
| React vdom+diff 慢 | 编译时将 JSX 转为 DOM API 命令式代码，运行时无 vdom | 消除 diff 开销，性能接近原生 JS |
| Native↔JS 通信慢（线程切换 + 序列化） | C++ 实现 DOM 对象注入 JS 环境，主线程同步调用 | 通信速度快 29~35 倍，无线程切换 |

理解了这三个决策，就理解了 Mach Pro 的灵魂。下面从架构层面看看它是怎么落地的。

---

## 三、整体架构：六层结构，前端和客户端各管哪些？

从全景图可以看到，Mach Pro 从上到下分为六层。为了方便理解，我先标注每一层主要由谁负责，然后再分别展开。

```
┌─────────────────────────────────────────────────────────────────────┐
│  容器层          MPPageView / MPBlockView                    客户端  │
├─────────────────────────────────────────────────────────────────────┤
│  前端框架层       Mach Pro 框架（Rematch/SectionList/Swiper...）       │
│                 公共运行时对象（Mach Object/KNB/MSI/Worker）         │
│                 Preact 运行时                                前端  │
│                 DOM 模拟层（Document/Window/EventTarget/Element/     │
│                           Style/Location）                         │
├─────────────────────────────────────────────────────────────────────┤
│  引擎层          DOM API / Module API / SharedWorker / Event Loop   │
│                 Event / Debugger / 多线程 / ExceptionHandler  客户端  │
│                 ──── QuickJS ────                                  │
├─────────────────────────────────────────────────────────────────────┤
│  原生层          内置 UI 组件（view/text/image/list/scroller...）     │
│                 内置模块（KNB/MSI/Logan/Metrics/Raptor...）         │
│                 Bundle 管理（检查更新/下载/加载/缓存/主子包策略）  客户端  │
│                 渲染引擎（CSS 样式/属性/事件/Yoga 布局/动画/RTL）     │
├─────────────────────────────────────────────────────────────────────┤
│  平台层          Android / iOS / 鸿蒙                        客户端  │
├─────────────────────────────────────────────────────────────────────┤
│  周边设施         开发调试工具 / 打包发布 / Bundle 管理 & 监控运维        │
└─────────────────────────────────────────────────────────────────────┘
```

简单来说：**客户端负责"向下"——搭建引擎环境、实现原生组件和模块、管理 Bundle、处理平台差异；前端负责"向上"——在客户端提供的 DOM API 之上构建开发框架，让业务开发者能用 React 语法写页面，并实现三端代码同构。**

这两层的分界线就是 **DOM API**：客户端通过 C++ 向 JS 环境注入 DOM API（createElement、appendChild、setAttribute 等），前端在这套 API 之上构建一切。就像浏览器提供 DOM API、前端在上面用 React 开发一样，只不过这里的"浏览器"是客户端用 C++ 和 QuickJS 搭建的。

---

## 四、客户端做了什么？（从底向上看）

客户端的核心工作是**在 Native 环境中构建一个高性能的 JS 执行环境**，让 JS 代码能同步操作原生 UI。可以分为五个部分来理解。

### 4.1 引擎环境搭建：QuickJS + C++ 中间层

这是整个框架的基石。客户端选择了 QuickJS 作为 JS 引擎（纯 C 编写，由 FFmpeg 作者 Fabrice Bellard 开发），并用 C++ 实现了一个中间层，这个中间层是整个方案的核心枢纽。

C++ 层做两件事：向上，通过 QuickJS 的 C API 向 JS 环境注入 Node Class 和 Mach 对象，让 JS 能调用 DOM API 和 Module API；向下，通过 JNI（Android）或 OC Runtime（iOS）调用原生的 Component 和 Module。所有跨语言调用都是 C 函数级别的直接调用，不需要序列化/反序列化，这就是通信速度提升 29~35 倍的原因。

为什么不用 Hermes？虽然 Hermes 也支持字节码预编译，但它的通信仍需通过 JSI 序列化，且体积更大。QuickJS 的 C 函数直接调用方式在通信性能上有数量级的优势。

### 4.2 Element 与 Component 绑定：框架最核心的设计

理解这个机制就理解了 Mach Pro 的工作原理。整个绑定分三步走：

**第一步**，C 层用 C 实现了一个 Node 类（参考 Web 标准的 Node API），通过 QuickJS C API 将这个 Node Class 注入到 JS 环境。Node 类提供了操作视图结构、样式、内容的方法（new、insert、attr、addEventListener 等）。

**第二步**，JS 侧的 Element 类继承自这个 C 层注入的 Node 类。JS 调用 Element 的方法时，通过 super 调用父类 Node 的同名方法，这意味着 JS 侧的每一次 DOM 操作都会同步穿透到 C 层。

**第三步**，C 层的 Node 对象在创建时会根据构造参数创建对应的 Native Component（UIView 或 Android View）并持有。绑定通过 QuickJS 的 SetOpaque API 实现——在 JS 对象内部保存一个指向 Native 对象的指针，C 代码可以从 JS 对象直接取出这个指针操作 Native 对象。

这样一来，JS 中写 `new Element('view')` → C 层创建 Node 并创建原生 UIView → `element.insert(child)` → C 层调用 UIView 的 addSubview → `element.attr('color', 'red')` → C 层调用 UIView 的 setBackgroundColor。整个过程同步执行，从 JS 到原生视图更新没有线程切换，没有序列化。

### 4.3 Native Module 与 JS Module：能力扩展体系

UI 之外，JS 还需要调用平台能力（网络请求、数据存储、埋点等），这通过 Native Module 实现。

Native Module 的设计原则是"Module 即对象"。Native 侧通过宏（如 `MP_EXPORT_MODULE(@"WMNetwork")`）导出 Module 类和方法列表。JS 侧通过 `Mach.requireModule('WMNetwork')` 获取 Module 对象（懒加载 + 缓存），然后直接调用方法。底层机制和 Component 绑定一样：通过 SetOpaque 绑定 JS 对象和 Native 对象，通过 Magic 值机制（QuickJS 的 JS_NewCFunctionMagic API）让所有方法共用一个 C 函数入口，运行时根据 Magic 值找到对应的 Native 方法并反射调用。

反过来，Native 也需要调用 JS——这通过 JS Module 实现。JS 侧通过 `Mach.exportModule("foodList", { getFoodName: (id) => "鸡翅" })` 导出方法，Native 侧通过 `callJSModule:@"foodList" method:@"getFoodName" params:@[@(10086)]` 同步调用并获取返回值。

此外还有事件通信系统，支持四个方向：JS 订阅 Native 事件（`Mach.on`）、JS 向 Native 发事件（`Mach.sendEvent`）、Native 向 JS 发事件（先检查订阅再回调）、数据更新（基于 dataChange 事件实现 cell 复用时的数据刷新）。事件与 Module 的区别在于：事件是一对多、无返回值；Module 是一对一、有返回值。

### 4.4 Event Loop：让异步操作跑起来

Preact 的批量更新依赖 Promise 微任务，而 Promise 依赖 Event Loop。QuickJS 自带的 Event Loop 会阻塞线程，不能用。客户端自己实现了一个轻量级方案：iOS 通过 CFRunLoopObserver 监听主线程 Runloop 的关键事件（BeforeTimers、BeforeWaiting 等），在每次回调时执行所有 Pending Job；Android 通过 Choreographer.postFrameCallback 在每帧回调时执行。本质都是在系统每帧渲染前把 JS 微任务清空。

### 4.5 多线程优化：解决复杂页面的 FPS 问题

Mach Pro 默认在主线程运行 JS，这在传统列表页表现很好。但遇到"高达页面"（多个独立列表卡片平铺在一个 scroller 中一次性渲染）时，JS 执行量暴增，主线程被长时间占用，FPS 严重下降。

客户端为此引入了可选的多线程模型：将 JS 执行放到独立的 JS 线程。Module API 中与 UI 无关的部分直接在 JS 线程执行，与 UI 有关的切到主线程；DOM API 全部改为异步执行（JS 线程创建 Node，主线程创建 Component）。Yoga 布局时机也需要调整——将布局任务提交到 JS 线程，等 JS 执行完毕后再回调主线程做布局，避免多次无效布局。

效果：低端机 FPS 从 33 提升到 49（+15.34），代价是包大小增加 86KB。通过 scheme 参数或配置文件开关控制。

### 4.6 Bundle 管理与分包加载

客户端负责 Bundle 的完整生命周期：检查更新、下载、加载、缓存、版本管理。Mach Pro 的分包目的与 MRN/小程序不同——它要解决的是"不同业务方的子模块修改导致整页重新发包"的问题。子包保留 5 个历史版本，加载时从高到低逐个匹配主包版本。App 退后台时清理废弃版本。

此外还有高达定制的 Bundle 管理器，支持 Bundle 预热和加载优化，针对多卡片场景做了专门处理。

---

## 五、前端做了什么？（从上向下看）

客户端搭建好了"类浏览器"的执行环境并提供了 DOM API，前端的工作就是在这套 API 之上构建完整的开发框架，让业务开发者能用 React 语法写页面，并且一套代码跑在 Native、小程序、H5 三端。

### 5.1 编译时框架：Babel 插件将 JSX → DOM API 命令式代码

这是前端最核心的工作。为什么不直接用 React 或 Vue？因为 Mach Pro 的 JS 在主线程执行，React/Vue 等重运行时框架的 vdom+diff 计算量太大，会把主线程卡死。而 Svelte 虽然足够轻量，但在外卖 React 技术栈下学习和复用成本太高。

前端的解法是实现一个 Babel 插件，在编译时把 JSX 声明式代码转换为 DOM API 的命令式代码。开发者写的是标准的 React 风格代码：

```jsx
<button type="button" style={{ width: 100 }} onClick={this.increment}>
  <text>{this.state.value}</text>
</button>
```

编译后变成：

```javascript
const btn = temp({ tagName: "button", attr: [{ key: "type", value: "button" }], children: [{ tagName: "text" }] });
style(btn, "width", 100);
listen(btn, "click", this.increment, true);
insert(text, this.state.value);
```

转换过程分三步：解析 DOM 树结构（tagName、children 生成基础树形结构）；添加属性（解析 attributes 收集节点属性）；绑定事件（识别 on/oncapture 开头的属性收集事件）。静态节点通过 template 模板一次性创建，动态节点通过 insert/style/listen 等命令式 API 绑定。

这样做的效果是：开发者用 React 语法写代码，编译产物是直接操作 DOM 的指令，运行时没有 vdom，没有 diff，性能接近原生 JS。

### 5.2 运行时框架：ElementContext 与生命周期

编译时只是把"怎么创建和更新 DOM"的代码生成出来了，还需要一个运行时来管理组件的生命周期和数据更新。前端参考 React v16.3 之前的生命周期设计了运行时框架，核心是 ElementContext 模块。

ElementContext 有三种类型，分别承担不同职责：**component** 类型对应自定义组件（`<MyComp/>`），通过 createComponent() 创建，内部持有 `new Comp(props)` 实例，只创建一次；**block** 类型处理条件渲染、列表渲染等 JSX 逻辑，通过 block() 创建，body 函数执行后返回模块内容；**fragment** 类型对应具体的 JSXElement 标签元素，通过 f() 创建。

核心思路是把 React 中最复杂的 render 函数替换为纯粹的 DOM API 操作，通过 render(mount) 和 render(updating) 区分首次挂载和更新。

数据更新方面，前端参考了三种方案——React 的 vdom+diff、SolidJS 的 effect 响应式、Svelte 的 $$invalidate+DirtyCheck——最终选择了"React 生命周期驱动 + 类 Svelte 的 Fragment & DirtyCheck"策略，在保持 React 编程模型的同时实现 DOM 节点的最大化复用。

后续演进方向是引入 Preact 作为运行时方案（从全景图可以看到 Preact 已经成为前端框架层的一部分），在保持性能的同时提供更完整的 React 兼容性。

### 5.3 DOM 模拟层：Document / Window / Element / Style / ...

从全景图可以看到，前端框架层的底部有一排模拟对象：Document、Window、EventTarget、Element、Style、Location。这些是前端在客户端注入的 Node API 基础上，进一步封装出来的类浏览器环境。有了这层封装，Preact 这样依赖浏览器 API 的框架才能在 Mach Pro 容器中运行。

### 5.4 三端同构：同一套代码跑在 Native / 小程序 / H5

这是前端的另一大核心工作。三端共享同一套编译时框架和运行时框架，差异只在底层 DOM API 的实现上：

**Native 端**用客户端 C++ 注入的 DOM API，JS 在 QuickJS 中执行，性能最高。

**小程序端**基于 Kbone（微信官方的 Web 框架适配层）做减法，移除非核心 API 以控制包体积。运行时采用虚拟节点树方案，通过 setData 驱动渲染，并通过数据压缩和渲染节流减少 setData 的调用频率和数据量。只支持类名选择器、CSS 内联到节点 style 属性。

**H5 端**使用浏览器原生 DOM API，核心价值是**开发提效**——开发者先在浏览器中用 Chrome DevTools 调试，确认功能和样式正确后，再到 Native 和小程序中微调适配，研发效率提升约 30%。同时也承担了营销活动页等 H5 场景的业务需求。

三端差异对比一览：

| 维度 | Native | 小程序 | H5 |
|------|--------|--------|-----|
| DOM API 来源 | C++ 注入 QuickJS | Kbone 精简版 | 浏览器原生 |
| JS 引擎 | QuickJS（字节码） | 小程序引擎 | V8/JSCore |
| 执行线程 | 主线程同步 | 逻辑层线程 | 浏览器主线程 |
| 子包加载 | Mach.requireBundle | 小程序分包 | 动态 import |
| 性能水平 | 最高（接近 Native） | 中等 | 取决于浏览器 |

### 5.5 多 Bundle 加载方案

前端实现了三层接口设计来支持业务模块的独立构建和按需加载：

**Mach.requireBundle**（客户端注入）是底层接口，触发客户端加载子 Bundle 文件。**requireAsync**（JS 层封装）是业务方调用的接口，以 Promise 方式包装了 requireBundle，内部有缓存机制避免重复加载。**AsyncComponent**（JS 辅助函数）进一步封装，支持加载失败时用 ErrorComponent 兜底展示。

子 Bundle 的产物是一个全局函数 `global.__bundleCallback__`，接收 SDK 依赖作为参数，返回 exports 对象。这种设计让子包和主包解耦，可以独立发版。

### 5.6 样式体系与基础组件

**样式单位**：px 和不带单位的值等于 dp（分别照顾前端和客户端同学的习惯）；PX（大写）在 Native 中等于 1 物理像素，在小程序中等于 1px。构建工具自动处理平台差异。

**CSS 限制**：只支持类名选择器，不支持元素选择器、级联选择器、ID 选择器——这是因为样式通过内联方式实现，复杂选择器在 Native 和小程序环境中难以高效支持。

**基础组件**：从全景图可以看到，前端框架层提供了 Rematch、SectionList、WaterfallList、Swiper、ViewPager 等高级组件，底层对应原生层的 view/text/image/list/scroller 等通用组件和 async-list/async-waterfall 等异步组件。

---

## 六、前端与客户端如何协作？一次完整的渲染流程

把前端和客户端的工作串起来，看一次完整的页面渲染是怎么发生的：

```
1. 【客户端】启动 Activity，创建 JSRuntime（QuickJS）
2. 【客户端】向 JS 环境注入 Node Class、Mach 对象、全局函数、渲染参数
3. 【客户端】加载并执行字节码（编译后的 JS Bundle）

4. 【前端编译时】业务代码的 JSX 已被 Babel 插件编译为 DOM API 命令式代码
5. 【前端运行时】Preact/ElementContext 执行编译后的代码，调用 DOM API
    → new Element('view')
    → element.insert(childElement)
    → element.attr('color', 'red')

6. 【客户端】每次 DOM API 调用同步穿透到 C 层
    → C 层创建 Native Component（UIView / Android View）
    → C 层设置组件属性、添加子视图
    → 构建出一棵 Component 树，根视图是 BodyView

7. 【客户端】字节码执行结束，调用 Yoga 布局引擎计算组件尺寸和位置
8. 【客户端】将 BodyView 挂载到 Container 上，系统渲染到屏幕
```

用户交互时的事件响应流程：

```
1. 【客户端】用户点击视图，系统回调事件监听函数
2. 【客户端】事件监听函数调用 JS 的 dispatchEvent，传入事件名和参数
3. 【前端】dispatchEvent 从 listeners 中找到对应回调函数并执行
4. 【前端】回调函数中修改 state，触发 re-render
5. 【前端】re-render 执行 DOM API 更新变化的节点
6. 【客户端】DOM API 同步更新对应的 Native Component
7. 【客户端】Yoga 重新布局，系统重新渲染
```

整个过程在主线程同步完成，没有线程切换，没有序列化。这就是 Mach Pro 能做到接近 Native 性能的根本原因。

---

## 七、性能数据速查

### SDK 级对比（Mach Pro vs MRN）

| 指标 | iOS（iPhone 6s） | Android（vivo x9s） |
|------|-----------------|-------------------|
| 总执行时间 | 快 **4.07** 倍 | 快 **3.33** 倍 |
| 通信速度 | 快 **29.4** 倍 | 快 **35.4** 倍 |
| 内存占用 | 降低 **75%** | 降低 **71%** |
| 首次渲染（TP90） | **23.95ms** vs 124ms（5.1x） | **58ms** vs 224ms（3.8x） |

### 业务级对比（点菜页 Mach Pro vs Native）

| 指标 | 差异 |
|------|------|
| TP50 首屏 | +74~109ms（可接受） |
| 滚动 FPS | 57.47 vs 57.96（基本持平） |

结论：动态化改造后首屏略有增加，但 FPS 与 Native 基本持平，用户感知不到差异。

---

## 八、行业定位：Mach Pro 的独特之处

| 方案 | 引擎 | 框架 | 工作线程 | V-DOM | 通信方式 |
|------|------|------|---------|-------|---------|
| **Mach Pro** | **QuickJS** | **Preact（编译时）** | **主线程** | **无** | **C 函数** |
| React Native | JSCore/V8/Hermes | React | 子线程 | JS 侧 | JSI（序列化） |
| Hippy（腾讯） | JSCore/V8 | Vue/React | 子线程 | JS 侧 | JSI |
| MLN（陌陌） | Lua | 无 | 主线程 | 无 | C 函数 |
| Hummer（滴滴） | JSCore/QuickJS | 无 | 主线程 | 无 | JSI（序列化） |

Mach Pro 是**唯一同时具备"JS 语言 + 主线程运行 + 无 V-DOM + C 函数通信 + 声明式 DSL + 运行时框架"的方案**。其他主线程方案要么用 Lua 语言（MLN），要么缺少运行时框架（Hummer），要么通信仍走序列化（Hummer）。

---

## 九、全景图分区速查

最后，结合全景图做一个快速索引，方便你知道每个模块属于哪一层、由谁负责：

**容器层（客户端）**：Mach Pro 容器提供 MPPageView（页面级）和 MPBlockView（模块级）两种接入方式。Mach 2.0 容器是新一代，提供 MPListManager/MPListCard/MPListItem。

**前端框架层（前端）**：上层是业务级组件框架（Rematch/SectionList/WaterfallList/Swiper/ViewPager），中间是公共运行时对象（Mach Object/KNB/MSI/Worker），下方是 Preact 运行时和 DOM 模拟层（Document/Window/EventTarget/Element/Style/Location）。

**引擎层（客户端）**：基于 QuickJS，向上暴露 DOM API、Module API、SharedWorker/DedicatedWorker、Event Loop、Event、Debugger、多线程、ExceptionHandler 等能力。

**原生层（客户端）**：内置 UI 组件分为通用组件（view/text/image/modal/input 等）、同步组件（list/waterfall/swiper/view-pager）、异步组件（async-list/async-waterfall/async-swiper/async-view-pager）。内置模块包含 KNB/MSI/Logan/Metrics/Raptor/Perf/Keyboard/BackPress。Bundle 管理分通用管理器（检查更新/下载/加载/缓存/存储治理/主子包策略）和高达定制管理器（预热/加载/缓存/主子包策略）。渲染引擎处理 CSS 样式、属性、事件、Yoga 布局、关键帧动画、过渡动画、相交检测、RTL。

**平台层（客户端）**：Android / iOS / 鸿蒙。

**周边设施**：开发调试工具（前端命令行工具/客户端 DevTools/Playground/Local Server/VSCode Debugger/天枢可视化搭建/防崩检测）、打包发布（Dove/FSD/Talos 流水线/DD/Diva/Themis 灰度发布）、Bundle 管理 & 监控运维（天枢平台/Themis/Merlin/Raptor/Perf/Metrics/Crash/Logan）。

---

> **备注**：本文档整合自三份内部技术文档和全景图，仅供个人学习使用。建议阅读顺序：先通读本文建立全局认知 → 再看《客户端技术方案整理》深入 C++ 实现细节 → 再看《前端技术方案整理》深入编译时框架和三端同构 → 最后看《技术方案整理》补充面试话术和性能数据。
