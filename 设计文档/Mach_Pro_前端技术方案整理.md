# Mach Pro 前端技术方案整理

> 本文档整理 Mach Pro 前端相关技术方案，涵盖 Native、H5、小程序三个平台的前端实现。文档来源为美团内部学城技术文档，仅供个人学习和面试准备使用。

---

## 一、整体定位：前端在 Mach Pro 中的角色

Mach Pro 是一个高性能动态化 UI 框架，客户端提供了底层的 DOM API 和基础标签（通过 C++ 注入 QuickJS 引擎），相当于在 Native 环境中创建了一个高性能的 Web 执行环境。前端的工作就是在这个环境之上，构建一套完整的开发框架和工具链，让业务开发者能用 React 语法开发页面，并实现三端（Native / H5 / 小程序）代码同构。

具体来说，前端负责的核心工作包括：

1. 编译时框架：Babel 插件将 JSX 声明式代码转换为 DOM API 命令式代码（类似 Svelte 的编译时方案）
2. 运行时框架：生命周期对齐 React、ElementContext 核心模块、数据更新机制
3. 小程序适配层：基于 Kbone 改造的虚拟节点树方案
4. H5 同构方案：浏览器 DOM API 对齐、构建流程改造、开发调试工具
5. 多 Bundle 加载方案：子包独立构建、异步加载、版本管理
6. 样式体系：三端统一的单位定义和 CSS 选择器规范
7. 基础组件库：三端对齐的基础组件封装

---

## 二、Native 方案：编译时框架

### 2.1 方案背景与目标

MachPro 容器为 JS Runtime 提供了类似 WebView 的简单版 DOM/BOM 执行环境。客户端实现了 Node、Mach 等基础 API，JS 侧负责封装完整的 DOM、BOM 能力。理论上传统的 Web 框架都能在 MachPro 容器中运行，但存在两个核心挑战：

第一，React、Vue 等重运行时框架在 MachPro 中执行会给主进程带来很大的 JS 执行压力，因为 MachPro 的 JS 执行是在主线程中进行的（这也是它高性能的主要原因）。第二，Svelte 这种编译时框架虽然足够轻量化，但在外卖以 React 为主导的技术栈下，业务复用和学习成本都存在弊端。

因此前端的最终目标是实现一款类 React 语法的编译时框架，具备两个核心特点：整体语法与 React 基本一致，对齐了 React 90% 的基本用法；重编译时框架，将所有 JSX 声明式代码通过编译时转换成 DOM API 命令式代码，运行时逻辑轻量化。

### 2.2 编译时：JSX → DOM API

编译时的核心目标是将 JSX 的声明式代码转换成 DOM API 的命令式代码。以一个包含属性、样式、事件、子节点的 button 组件为例：

声明式代码（开发者编写）：

```jsx
<button
  type="button"
  style={{ width: Math.random() * 100, height: Math.random() * 100 }}
  onClick={Math.random() > 0.5 ? this.increment : null}
>
  before
  <text>{this.state.value}</text>
</button>
```

命令式代码（编译后生成）：

```javascript
const btn = temp({
  "tagName": "button",
  "attr": [{ "key": "type", "value": "button" }],
  "children": [ "before", { "tagName": "text" }],
});
const c1 = btn.firstChild,
const text = c1.nextSibling;

// 设置样式属性
style(btn, "width", Math.random() * 100);
style(btn, "height", Math.random() * 100);

// 添加事件监听
listen(btn, "click", Math.random() > 0.5 ? this.increment : null, true);

// 插入动态节点
insert(text, this.state.value);
insert(btn, text);
```

从声明式到命令式的转换主要有三个步骤：解析 DOM 树结构（tagName、children 等属性生成基础 DOM 树形结构）；添加属性（解析 attributes，根据属性值和名称收集节点属性）；绑定事件（判断 on 和 oncapture 开头的属性，收集事件名称）。

前端实现了一个 Babel 插件来解决这个问题，主要通过 AST 对 JSXElement 进行解析。转换过程会生成一个 Result 对象，结构如下：

```javascript
{
  template: '<button type="button">before<text></text></button>',  // 模板语句
  templateInfo: {                                                   // 模板信息（JSON结构）
    "tagName": "button",
    "_a": [{ "key": "type", "value": "button" }],
    "_c": [ "before", { "tagName": "text", "_a": [], "_c": [] } ]
  },
  decl: [...],   // 变量定义宣言
  exprs: [...],  // DOM 命令式创建的表达式（insert, listen, style等）
  tagName: 'view',
  id: { type: 'Identifier', name: '_el$' },  // JSXElement 对应的 JS 变量名
}
```

最终根据 Result 对象生成以 DOM API 为基础的可执行命令式代码，通过 temp() 创建 DOM 树，insert() 插入动态节点，style() 设置样式，listen() 绑定事件，createComponent() 创建自定义组件。整个过程中静态节点（如固定的标签名、固定属性）通过 template 模板创建，动态节点（如表达式、状态值、事件回调）通过 insert/style/listen 等命令式 API 绑定。

### 2.3 运行时：生命周期对齐与 ElementContext

运行时参考了 React（v16.3 之前）的生命周期设计，主要分为三个时期：挂载时（组件首次渲染，根据初始化的 State & Props 完成首次渲染，创建并插入 DOM 节点）；更新时（属性、状态发生更新时执行更新逻辑并渲染）；卸载时（执行卸载前回调，移除 DOM 节点）。

核心思路是将 React 中最复杂的 render 函数替换为纯粹的 DOM API 操作。理论上只需要替换 render 函数的实现逻辑，并让其保持和 React 一致的执行过程与结果，就能实现一个类 React 的命令式 DOM 创建框架。通过 render(mount) 和 render(updating) 来区分是首次挂载还是更新操作。

ElementContext 是运行时的核心模块，通过 createElementContext 创建，用来表示各种类型的元素。它包含以下关键字段：key（根据代码位置生成的唯一 id）、type（元素类型，有 block、fragment、component 三种）、uniqId（唯一 id，根据代码位置、插入位置以及父级节点确定）、target（挂载的父级节点）、mount（挂载函数，当元素被创建时执行）、update（更新函数，当 context 更新时触发）、element（当前 ElementContext 的返回结果，最终被用于 insert 插入）、destroy（卸载函数，当前 ElementContext 被移除时触发）。

三种元素类型分别承担不同的职责：

component 类型通过 createComponent() 创建，所有的 <Comp/> 都会变成该类型。它的 element 返回结果一定是 block 类型，instance 是 new Comp(props) 的实例，只会创建一次。这对应 React 中自定义组件的概念。

block 类型通过 block() 创建，会处理所有 JSX 相关的逻辑判断、计算、数组等内容。它的 body 是模块内容的计算函数，执行后返回模块的内容结果；element 返回结果可能是 ElementContext、Element 或普通类型。这对应 JSX 中的条件渲染、列表渲染等逻辑。

fragment 类型通过 f() 创建，所有的 JSXElement 块都会被转换成该类型。它的 element 返回结果一定是 Element 或普通类型。这对应 JSX 中的标签元素。

### 2.4 数据更新：Re-Render 机制

数据更新是框架设计的另一个核心问题。最简单粗暴的方法是删除所有节点重新创建绑定，但这种方案对性能有较大损耗。

MachPro 参考了三种主流框架的更新逻辑来做设计选型：

React 的方式是根据新数据重新执行 render，生成新的 vnode 树，再对新旧树进行 diff，根据 diff 结果操作 DOM 更新。这种方式的问题在于 vdom+diff 过程耗时，且需要维护三份 DOM 树（VNode/ShadowNode/View）。

SolidJS 的方式是在编译时区分静态值和动态值，对动态值使用 effect 函数进行收集，当值变化时触发更新回调。缺点是响应式收集只包含节点属性事件等，对于大批量的节点结构变化更新主要依靠 innerHTML 等高性能 DOM API 重新创建。

SvelteJS 的方式是在编译时将所有赋值代码用 $$invalidate 包裹，并将所有使用到的值通过 instance 函数储存在 ctx 上，当 ctx 发生变化时执行 update 操作。优点是拥有 Fragment 和 DirtyCheck 机制，能做到局部片段复用以及节点复用；缺点是有较多的特异性语法要求。

MachPro 的期望设计是：通过 React 生命周期进行驱动，保持更新统一入口 render 函数；最小化数据更新对节点造成的影响，实现类 Svelte 的 Fragment & DirtyCheck 逻辑，最大程度地复用 DOM 节点。

### 2.5 后续演进：Preact 运行时方案

根据性能、业务适用、整体成本的调研考虑，MachPro 后续计划采用 Preact 作为 DSL 的运行时方案。当前使用的编译时方案已上线外卖点菜页、PickMe 页面等，性能均符合预期。运行时方案作为后续推进方向，会在跨端标准化事项中同步落地。这意味着 MachPro 的前端框架会从纯编译时方案演进到编译时 + 运行时（Preact）的混合方案，在保持性能的同时提供更完整的 React 兼容性。

---

## 三、小程序方案：Kbone 改造与虚拟节点树

### 3.1 方案概述

小程序方案的核心是基于 Kbone 进行改造，在 Kbone 基础上做减法。Kbone 是微信小程序官方提供的 Web 框架适配层，它在小程序环境中模拟了浏览器的 DOM/BOM API，使得 Web 框架可以在小程序中运行。MachPro 对 Kbone 进行了精简，移除了大部分非核心 API，以减小包体积和提升性能。

### 3.2 运行时设计

小程序运行时采用虚拟节点树方案，不使用 VDOM 运行时。核心设计包括以下几个方面：

节点管理方面，通过虚拟节点树管理页面结构，每个节点对应一个原生小程序组件。节点树的创建和更新通过 setData 驱动，但通过数据压缩和渲染节流来减少 setData 的调用频率和数据量。

组件映射方面，MachPro 的基础标签会映射到小程序原生组件，例如 view 标签映射到小程序的 view 组件。这种映射在编译时完成，运行时直接使用映射后的组件。

API 体系方面，提供了节点管理 API（创建、查询、修改节点）、组件 API（组件注册、生命周期）、事件 API（事件绑定、触发、冒泡）等完整的接口。

样式处理方面，小程序使用 CSS 内联方式，将样式直接内联到节点的 style 属性上，而不是使用外部样式表。这是因为小程序的样式系统与 Web 有较大差异，内联方式可以保证三端样式的一致性。

渲染节流方面，通过 setData 数据压缩和渲染节流机制，减少小程序渲染层的压力。对 setData、模板、事件等字段数据进行压缩，降低通信数据量。

### 3.3 代码构建

小程序的代码构建分为逻辑处理和样式处理两部分。逻辑处理负责将 MachPro 的 JSX 代码编译为小程序可执行的 JS 代码，包括 Babel 插件的 AST 转换、小程序适配层的注入等。样式处理负责将 CSS 样式转换为小程序兼容的格式，包括单位转换、选择器适配等。

动态加载方面，小程序支持子包加载，可以通过小程序的分包机制实现业务模块的独立更新。这与 Native 的多 Bundle 加载方案对应，但实现方式不同，小程序使用原生的分包能力。

### 3.4 性能与监控

小程序方案的性能风险点主要集中在 setData 的频率和数据量上。通过数据压缩、渲染节流、按需更新等策略来优化。同时建立了完整的监控体系，对渲染性能、包体积、加载时间等指标进行监控。

---

## 四、H5 方案：代码同构与开发提效

### 4.1 项目背景

H5 方案的推进有三个核心驱动因素。第一，微信小程序对动态化能力进行了封禁，小程序发版周期和包体积的问题重新出现，团队调整了动态化策略，小程序非核心业务使用 H5 支持。第二，MachPro Native/小程序的开发流程有较大提升空间，可以通过 Web 在浏览器中进行调试，然后在 Native、小程序中进行微调适配，提升研发效率。第三，C 端津贴联盟落地页为营销类活动页，活动多、迭代更新频率高，已有技术形态不符合当前业务场景，需要使用 MachPro 主子包对津贴落地页进行重构。

### 4.2 核心目标与收益

H5 方案的核心目标是代码同构，完成 MachPro-H5 的能力建设，保证与 MachPro Native/小程序的能力同构。通过完备的 Web 开发调试流程优化 MachPro 的开发体验，整体研发效率提效 30%。在业务层面，配合广告业务在营销业务中落地，支撑津贴裂变分享等业务形态。

### 4.3 核心技术模块

H5 方案包含以下核心技术模块：

构建流程改造方面，对 CLI 和 build 流程进行改造，支持 H5 端的构建产物输出。包括静态资源上传到 CDN、移动端适配（viewport、rem 等）。H5 构建产物需要适配浏览器的模块加载机制，与 Native 的字节码加载和小程序的分包加载都不同。

DOM & 环境对齐方面，实现 DOM 环境全量变量对齐。由于 Native 端的 DOM API 是客户端实现的简化版，H5 端使用浏览器原生 DOM API，需要确保两端的 API 行为一致。这包括 DOM 节点的创建、查询、修改 API，事件系统，以及 BOM 相关的 API（如 setTimeout、requestAnimationFrame 等）。

样式适配方面，实现多端样式对齐和移动端展示适配。由于三端的样式单位定义不同（详见样式体系部分），H5 端需要进行单位转换。同时需要处理浏览器与 Native 渲染引擎的差异，确保视觉效果一致。

基础组件对齐方面，与客户端和小程序的组件能力对齐。一期优先实现了 view、text、image、scroll-view、canvas、Modal、FlatList/SectionList 等核心组件。这些组件在 H5 端使用浏览器原生 HTML 元素实现，但需要保持与 Native/小程序一致的 API 接口和交互行为。

Mach 内置 API 方面，对齐全局 Mach 对象的能力，包括模块导出、模块调用、事件通信等。同时支持灵犀（埋点监控）和 Raptor（性能监控）等基础设施。

子包加载方面，实现 H5 端的子包加载能力。与 Native 的多 Bundle 加载方案对应，但 H5 端使用浏览器的动态 import 或 script 标签加载机制，而非客户端的 Mach.requireBundle。

开发调试工具方面，提供本地 devServer 和热更新能力。这是 H5 方案的核心价值之一，开发者可以在浏览器中使用完整的开发者工具进行调试，然后适配到 Native 和小程序端。相比直接在 Native 或小程序中调试，大幅提升了开发效率。

发布流水线方面，支持 H5 产物的自动构建、测试、发布流程，与 CI/CD 系统集成。

### 4.4 开发流程

H5 方案带来的开发流程改进是：先在浏览器中开发和调试（利用 Chrome DevTools 等工具），确保功能正确和样式一致，然后在 Native 和小程序环境中进行微调适配。这种流程相比直接在 Native 或小程序中开发调试，效率提升约 30%，因为浏览器的开发者工具远比 Native 和小程序的调试工具强大。

---

## 五、多 Bundle 加载方案

### 5.1 方案设计

多 Bundle 加载方案支持业务模块独立构建和按需加载，是三端共用的能力。核心接口设计如下：

Mach.requireBundle 是 Native 实现的接口，注入到 Mach 对象上，用于触发客户端加载子 bundle。接口签名为 (bundleId: string, callback: (error: Error) => void) => void，业务方不直接调用此接口。

requireAsync 是 JS 实现的接口，业务方调用，以 Promise 方式加载子 bundle。接口签名为 (bundleId: string) => Promise<BundleModule>，成功时 then 返回 bundle 模块，失败时 catch 返回错误信息。内部实现了缓存机制，已加载过的 bundle 不会重复加载。

AsyncComponent 是 JS 实现的辅助函数，方便使用异步组件。接口签名为 (loader: () => Promise<Component>) => Promise<Component>，接收一个加载函数，返回一个组件。加载失败时可以使用 ErrorComponent 兜底展示。

### 5.2 产物约定

子 Bundle 的产物定义为一个全局函数 global.__bundleCallback__，接收 SDK 内置依赖 $$ 作为参数，返回 exports 对象：

```javascript
global.__bundleCallback__ = function ($$ /* sdk 内置依赖 */) {
  class ChildComponent extends $$.Component {
    render() { /* ... */ }
  }
  const func = () => { };
  
  exports.default = ChildComponent;
  exports.func = func;
  return exports;
}
```

入口 Bundle 的产物中实现了 requireAsync 和 AsyncComponent。requireAsync 内部通过 Mach.requireBundle 触发客户端加载，加载完成后调用 global.__bundleCallback__ 传入 SDK，得到真实模块并缓存。AsyncComponent 内部创建一个包装组件，在 componentWillMount 时执行 loader 加载异步组件，加载成功后 setState 触发渲染，加载失败则渲染兜底组件。

### 5.3 业务使用示例

业务方使用异步组件和异步 API 的方式如下：

```javascript
// 使用异步组件，加载失败时使用 ErrorComponent 兜底
const ChildComponent = AsyncComponent(
  () => requireAsync('bundle_id').catch((err) => ErrorComponent)
);

// 使用异步 API
Mach.requireBundle('bundle_id').then(bundle => {
  bundle.func();
});

class App extends Component {
  render() {
    return (
      <view>
        <view>父组件</view>
        <ChildComponent
          name={this.state.childName}
          onClick={this.onChildClick}
        />
      </view>
    )
  }
}
```

---

## 六、样式体系：三端统一规范

### 6.1 单位定义

MachPro 在三端统一定义了样式单位，解决不同平台单位差异的问题：

dp 是标准 dp 单位，不缩放，保留对 dp 的支持是为了方便 Mach 1.0 的模板迁移。不带单位的值等于 dp，方便客户端同学的开发习惯。px 等于 dp，方便前端同学的开发习惯。PX（大写）在 MachPro 客户端中等于 1 物理像素，在 MachPro 小程序中等于 1px，用于需要精确控制物理像素的场景。

需要注意的是，小程序端不支持 dp 单位和不带单位的值，小程序中的 px 非物理像素而是等于客户端的 dp。这些差异在编译时由构建工具自动处理，开发者使用 px 或不带单位即可。

### 6.2 CSS 选择器限制

MachPro 只支持类名选择器（class selector），不支持元素选择器、级联选择器、ID 选择器等其他 CSS 选择器类型。这是由于 MachPro 的样式系统是通过内联方式实现的（特别是在小程序端），复杂的 CSS 选择器在 Native 和小程序环境中难以高效实现。开发者需要使用 className 来应用样式，遵循类似 CSS Modules 的写法。

---

## 七、三端对比与架构总结

### 7.1 三端技术方案对比

| 维度 | Native | 小程序 | H5 |
|------|--------|--------|-----|
| DOM API 来源 | 客户端 C++ 注入 QuickJS | Kbone 改造（精简版） | 浏览器原生 DOM API |
| JS 引擎 | QuickJS（字节码） | 小程序 JS 引擎 | 浏览器 V8/JSCore |
| 执行线程 | 主线程同步执行 | 小程序逻辑层线程 | 浏览器主线程 |
| 样式实现 | 内联到节点 style | 内联到节点 style | 浏览器 CSS |
| 组件映射 | 原生视图组件 | 小程序原生组件 | HTML 元素 |
| 子包加载 | Mach.requireBundle | 小程序分包 | 动态 import/script |
| 调试能力 | 有限 | 较弱 | Chrome DevTools |
| 性能水平 | 最高（接近 Native） | 中等 | 取决于浏览器 |

### 7.2 前端架构整体关系

```
                    业务代码（React 语法 JSX）
                           │
                    ┌──────┴──────┐
                    │  Babel 插件  │  编译时：JSX AST → DOM API 命令式代码
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │  运行时框架  │  ElementContext / 生命周期 / Re-Render
                    └──────┬──────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐
    │   Native    │ │   小程序     │ │     H5      │
    │ DOM API     │ │ Kbone 精简  │ │ 浏览器 DOM  │
    │ (C++ 注入)  │ │ (虚拟节点树) │ │ (原生 API)  │
    └─────────────┘ └─────────────┘ └─────────────┘
```

### 7.3 关键设计决策

编译时优先：选择编译时方案而非纯运行时方案，核心原因是 MachPro 在主线程执行 JS，重运行时框架（React/Vue）会给主线程带来过大压力。编译时方案将声明式代码预编译为命令式 DOM API 代码，运行时逻辑极轻量。

类 React 语法而非 Svelte：在外卖以 React 为主导的技术栈下，使用类 React 语法可以最大化业务代码复用，降低学习成本。通过 Babel 插件实现 JSX → DOM API 的转换，保留了 React 的开发体验。

Kbone 改造而非自研小程序适配层：基于成熟的 Kbone 方案做减法，避免重复造轮子，同时通过精简非核心 API 来控制包体积和性能。

H5 优先调试策略：利用浏览器的强大调试能力作为主要开发环境，再适配到 Native 和小程序，实现研发效率最大化。

三端统一单位体系：通过 dp/px/PX 的统一定义和构建时自动转换，让开发者无需关心平台差异，一套样式代码三端通用。

---

## 八、面试速记卡

### 8.1 前端在 Mach Pro 中做什么

Mach Pro 客户端提供了底层的 DOM API 和基础标签（C++ 注入 QuickJS），前端在此基础上构建了完整的开发框架：编译时通过 Babel 插件将 JSX 转换为 DOM API 命令式代码（类似 Svelte 的编译时方案，但保留 React 语法）；运行时通过 ElementContext 模块管理组件生命周期和数据更新；同时实现了小程序 Kbone 适配层、H5 浏览器同构方案、多 Bundle 加载、三端统一样式体系等。

### 8.2 编译时方案核心流程

Babel 插件解析 JSX AST → 生成 Result 对象（template + templateInfo + decl + exprs）→ 根据 Result 生成 DOM API 命令式代码（temp 创建树、insert 插入节点、style 设置样式、listen 绑定事件、createComponent 创建组件）。

### 8.3 运行时 ElementContext 三种类型

component 类型（createComponent 创建，对应自定义组件，返回 block 类型）；block 类型（block 创建，处理条件渲染、列表渲染等逻辑）；fragment 类型（f 创建，对应 JSXElement 标签元素）。

### 8.4 数据更新方案

参考 React（vdom+diff）、SolidJS（effect 响应式）、SvelteJS（$$invalidate + DirtyCheck）三种方案，最终选择通过 React 生命周期驱动，render(mount) 和 render(updating) 区分挂载和更新，实现类 Svelte 的 Fragment & DirtyCheck 逻辑最大化复用 DOM 节点。

### 8.5 三端同构的关键

Native 使用客户端 C++ 注入的 DOM API；小程序使用 Kbone 改造的虚拟节点树；H5 使用浏览器原生 DOM API。三端共享同一套编译时框架和运行时框架，通过不同的底层 DOM API 实现适配。样式通过统一的单位体系（dp/px/PX）和类名选择器规范保证一致性。

### 8.6 多 Bundle 加载方案

Mach.requireBundle（Native 注入）→ requireAsync（JS 封装，Promise + 缓存）→ AsyncComponent（辅助函数，加载失败兜底）。子 Bundle 产物为 global.__bundleCallback__ 函数，接收 SDK 依赖，返回 exports 模块。

---

> **备注**：本文档基于美团内部学城文档整理，涵盖以下文档：
> - 【Native】MachPro方案设计 (1204755803) 及其子文档：MachPro DSL方案介绍 (1350683278)、多Bundle加载方案 (1213585413)、样式相关 (1211835324)
> - 【H5】MachPro方案设计 (1344367458)
> - 小程序方案 (1215082557)
>
> 仅供个人学习和面试准备使用。文中技术细节来自内部文档，具体实现可能随版本迭代有所变化。
