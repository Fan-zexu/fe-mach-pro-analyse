# Mach Pro 高性能动态化框架 — 技术方案深度整理

> **用途**：技术学习 + 面试简历素材  
> **来源文档**：5篇学城文档综合整理  
> - [Mach Pro 的设计思路](https://km.sankuai.com/collabpage/1208642507)
> - [新动态化框架 Mach Pro 调研](https://km.sankuai.com/page/548872687)
> - [Mach Pro 高性能动态化方案](https://km.sankuai.com/page/1264045322)
> - [Mach Pro 框架概览](https://km.sankuai.com/page/1213701280)
> - [Mach Pro 实现原理分享](https://km.sankuai.com/page/1349546777)

---

## 目录

- [一、项目背景](#一项目背景)
- [二、现有方案的问题分析](#二现有方案的问题分析)
- [三、Mach Pro 技术方案总览](#三mach-pro-技术方案总览)
- [四、核心架构设计](#四核心架构设计)
- [五、核心实现原理](#五核心实现原理)
- [六、性能数据与业务落地](#六性能数据与业务落地)
- [七、行业方案对比](#七行业方案对比)
- [八、面试话术与核心考点](#八面试话术与核心考点)
- [九、学习笔记速查](#九学习笔记速查)

---

## 一、项目背景

### 1.1 业务背景

外卖业务动态化改造进入深水区。非核心页面已通过 MRN 或 H5 实现全页面动态化，但核心页面的改造陷入瓶颈：

- **部分核心页面**（首页、提单页、订单状态页）通过 Mach 只实现了局部模块动态化，动态化比例受限
- **复杂结构页面**（如点菜页）因结构复杂、交互复杂、动效复杂，Mach 和 MRN 都无法满足需求，几乎完全没有动态化

点菜页的典型难点：菜品列表与背景墙的复杂嵌套联动动画、长列表高性能滑动、接近 Native 的 UI 展示效果。动态化不能以牺牲业务功能和用户体验为代价。

### 1.2 核心诉求

| 维度 | 诉求 |
|------|------|
| 用户体验 | 动态化前提下保证滑动性能和嵌套联动动画流畅性，接近 Native UI 效果 |
| 性能 | 高 PV 场景下低 CPU 占用和内存消耗，首屏秒开 |
| 开发效率 | Android、iOS、小程序三端复用，React 技术栈 |

### 1.3 为什么需要第三种方案

外卖原有 "模块级用 Mach + 页面级用 MRN" 的改造思路存在根本矛盾：

- **Mach**：阉割了 JS 能力，无法实现复杂视图和交互逻辑，追赶 RN 但能力无法对齐
- **MRN**：三层架构导致内存膨胀，每个模块引入 ~35kb React 运行时，核心页面性能不达标
- **体验断层**：用户操作流中 Mach 与 RN 页面间白屏时间/滑动感受存在明显差异

很多业务模块的复杂度介于 Mach 和 MRN 之间，导致频繁使用取巧方案改造，临时方案无法沉淀，给后续迭代留下后遗症。

**核心目标**：从模块到页面的改造成本为线性复杂度，能力高度一致，一套方案统一模块级和页面级动态化。

---

## 二、现有方案的问题分析

### 2.1 Mach 的问题

Mach 的主体渲染流程与 RN 类似，但将绝大部分渲染步骤放在 Native 执行，JS 仅负责数据预处理和点击交互。

**优点**：模板渲染速度快、内存占用低，适合混排长列表（一张卡片一个模板）。

**缺点**：RN 的动态性完全来自 JS，阉割 JS 能力使得 Mach 能力边界极小，无法实现复杂视图和交互逻辑，只能局限于重展示轻交互的简单模块。如果要支持组件化/Hooks 等高阶能力，必须不断改造脚手架，陷入"追赶 RN 但能力无法对齐"的尴尬境地。

### 2.2 React Native (MRN) 的问题

#### 2.2.1 表面现象 — 三大问题

**问题一：首屏时间长**

RN 官方耗时占比中，JS Init + Require 占据一大半时间，主要操作是初始化 JS 引擎和加载 JS Bundle。MRN 实践数据显示引擎初始化 TP90：iOS 0.6s、Android 2.6s。

MRN 的优化方案是提前初始化引擎并预加载 JS Bundle，但预加载有条件限制和成本（内存压力、漏斗比例），在业务首页场景中效果不明显。

**问题二：交互动效卡顿**

以手势交互为例，每次手势交互产生两次 Native↔JS 通信（Native→JS 传递手势事件，JS→Native 驱动 UI 更新）。Native 和 JS 工作在不同线程，每次通信需要线程切换，耗时 >16ms 造成卡顿。快速滑动时回调频繁，卡顿明显。

阿里 BindingX 方案通过表达式绑定减少通信次数，但双端适配成本大且有 bind 不上的异常，没有完美解决问题。

**问题三：长列表白屏**

FlatList 在快速滑动、锚点跳转时出现空白。原因是列表滚动期间不断创建/销毁 Cell，由于异步渲染机制和桥通信耗时，容易出现渲染不及时。

#### 2.2.2 根本原因 — 三大瓶颈

**瓶颈一：JS 引擎执行慢**

实验数据（iOS/Android 低端机 Release 包，执行 10 万行业务 JS 代码 / 50 个业务模块）：

| 引擎 | 解析 | 执行 | 总时间 |
|------|------|------|--------|
| JSCore (iOS) | 78.13ms | 190.75ms | 268.88ms |
| V8 (Android) | 159.9ms | 234.6ms | 394.5ms |

JS 源码从加载到执行需经历 Lexer、Parser、Bytecode Generator 等多步，每步都很耗时。

**瓶颈二：React 框架执行慢**

根据 JS Framework Benchmark，React 比手动操作 DOM 慢 86%，在主流框架中几乎最慢。vdom + diff 机制引入了大量不必要的计算。

**瓶颈三：Native 和 JS 通信慢**

两个原因：一是引擎本身通信慢（JSCore 操作 2500 个 DOM 节点耗时 72.33ms，V8 耗时 113.4ms）；二是线程切换慢，异步通信带来很多复杂问题。

#### 2.2.3 RN 渲染机制详解（面试重点）

RN 渲染的六个关键步骤：

1. JS 代码生成虚拟视图树（vDom），记录组件属性和树状关系
2. 视图树转换为 Fiber 链表结构（FiberDom），用于分片操作（每 16ms 发送 1~N 个操作命令到 Native）
3. JS 侧分片后的操作命令通过 Bridge 发送到 Native（批量处理、异步通信、比较耗时）
4. Native 侧生成与 JS vDom 对应的 ShadowTree
5. 调用 Yoga 布局引擎计算每个组件的 size 和 position
6. 主线程绘制真实的 ViewTree

**两个关键特点：异步渲染、通过桥通信。**

RN 线程模型包含三个线程：JS 线程（加载执行 JS）、Shadow 线程（布局计算）、UI 线程（原生渲染/交互/原生能力）。一次简单的用户交互需要四次线程切换，消息队列拥堵导致 UI 不能及时刷新。

### 2.3 vdom + diff 的根本性问题

从 vdom 的更新机制来看，当 model 中某个字段变化时，信号在触发渲染前被合并（stateStream），丢失了"哪个字段变化"的信息。必须等到 render 函数执行完构造新 vdom 树，前后两棵 vdom 完成 diff 后才能分辨变化的节点。

**核心洞察**：如果在编译时就能将数据和视图元素绑定，就不需要在运行时构建 vdom 和执行 diff 操作，从而让方案能够工作在主线程上。

### 2.4 业界主线程方案的启示

陌陌 MLN 和滴滴 Hummer 都采用了主线程方案，特点是都没有 vdom+diff 机制，DSL 是命令式风格。但缺少运行时框架导致状态复杂页面代码质量失控。

两家都提供了第二套 DSL（陌陌 ArgoUI 类 SwiftUI，滴滴回归 Vue/React），说明**一个完备的动态化方案一定要有一个健壮的运行时框架**。

Svelte 框架的启示：通过静态编译减少框架运行时代码量，将模板编译为命令式 DOM 操作，不需要 vdom 的 diff/patch，性能接近 vanilla JS。

---

## 三、Mach Pro 技术方案总览

### 3.1 一句话描述

> Mach Pro 通过 QuickJS 引擎向 JS 提供一套精简的 DOM API，JS 可以通过 DOM API 在主线程同步操作 Native 的 UI 组件。

### 3.2 技术选型

Mach Pro 的核心构建方案：**轻量级 React + 轻量级 JS 引擎 + 主线程运行 + 原生组件渲染**

| 组件 | 选型 | 解决的问题 |
|------|------|-----------|
| JS 引擎 | QuickJS（纯 C 编写） | JS 执行慢、通信慢、内存占用高 |
| 前端框架 | Preact / 自研轻量级 React | React 框架执行慢、运行时体积大 |
| 工作线程 | 主线程 | 线程切换慢、异步通信延迟 |
| 渲染方式 | 原生组件（UIKit/Android View） | 接近 Native 的 UI 展示效果 |
| 通信方式 | C 函数直接调用 | 桥通信序列化/反序列化开销 |

### 3.3 三大核心解决方案

#### 解决方案一：QuickJS 引擎替代 JSCore/V8

QuickJS 由 Fabrice Bellard 开发，纯 C 编写，执行速度快、内存占用低、支持字节码（AOT 预编译）。

传统 JS 引擎加载 JS 源码后在运行时解析执行，需经历 Lexer→Parser→Bytecode Generator 等步骤。QuickJS 支持将 JS 源码预先编译为字节码，APP 直接执行字节码，省去解析步骤。

性能对比（50 个业务模块，2500 个 DOM 节点，10 万行代码）：

**iOS 低端机（iPhone 6s）**：

| 指标 | JSCore | QuickJS | 倍数 |
|------|--------|---------|------|
| 创建引擎 | 1.53ms | 1.37ms | 1.12x |
| 解析 JS/字节码 | 78.13ms | 26.53ms | 2.94x |
| 执行 JS/字节码 | 190.75ms | 53.79ms | 3.54x |
| 通信 | 72.33ms | 2.46ms | **29.4x** |
| 总时间 | 342.74ms | 84.15ms | **4.07x** |
| 内存占用 | 27.84MB | 6.75MB | **4.12x** |

**Android 低端机（vivo x9s）**：

| 指标 | V8 | QuickJS | 倍数 |
|------|-----|---------|------|
| 创建引擎 | 4.1ms | 2.8ms | 1.46x |
| 解析 JS/字节码 | 159.9ms | 50.5ms | 3.16x |
| 执行 JS/字节码 | 234.6ms | 97.2ms | 2.41x |
| 通信 (JS→C++) | 113.4ms | 3.2ms | **35.4x** |
| 总时间 | 512ms | 153.7ms | **3.33x** |
| 内存（峰值） | +33.8M | +9.7M | **3.48x** |

#### 解决方案二：轻量级 React 框架

联合 R2X 团队自研轻量级 React 框架（后切换为 Preact），具备 React 的开发效率，但编译产物不会引入 React 运行时代码。

- Preact 比 React 快 29.8%（JS Framework Benchmark）
- 体积小，与 React 标准对齐程度高
- 编译产物为直接操作 DOM 的指令，无 vdom + diff 开销

#### 解决方案三：C++ DOM 注入 + 主线程同步执行

通过 C++ 代码实现 JS 中 DOM 对象的定义并注入到 JS 环境，JS 对 DOM 的操作转化为同步调用 C++ 代码，再由 C++ 调用 Java/OC 实现同步调用 Native。JS 和 Native 运行在同一线程，互相直接调用，无线程切换、无通信延迟。

---

## 四、核心架构设计

### 4.1 分层架构

Mach Pro 采用三层架构设计（从上到下）：

```
┌─────────────────────────────────────────────────┐
│              JS Framework（绿色层）               │
│  ┌──────────────┐  ┌──────────────┐             │
│  │    React     │  │  Module API  │             │
│  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                      │
│  ┌──────▼───────┐         │                      │
│  │   DOM API    │         │                      │
│  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                      │
├─────────┼─────────────────┼──────────────────────┤
│         │   JS Engine（蓝色层）  │                │
│  ┌──────▼───────┐  ┌──────▼───────┐             │
│  │    Node      │  │ Mach Object  │             │
│  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                      │
│  ┌──────▼─────────────────▼───────┐             │
│  │          QuickJS               │             │
│  └────────────────────────────────┘             │
│                                                  │
├──────────────────────────────────────────────────┤
│              Native（黄色层）                     │
│  ┌──────────────┐  ┌──────────────┐             │
│  │  Component   │  │    Module    │             │
│  └──────────────┘  └──────────────┘             │
└──────────────────────────────────────────────────┘
```

**调用链路**：
- UI 渲染：React → DOM API → Node → QuickJS → Component → 原生视图
- 能力扩展：Module API → Mach Object → QuickJS → Module → Native 能力

### 4.2 五大功能模块

| 模块 | 职责 |
|------|------|
| **JS Framework** | 与业务代码直接交互，实现内置复杂组件（List/Swiper），定义 DOM API 和 Module API |
| **JS Engine** | 提供 JS 运行时环境，定义 Node 对象实现 DOM 与 Component 映射，包含异常收集和事件循环 |
| **Native** | 封装各平台原生视图组件，接收系统事件（点击/滚动）并传递给 JS 层，管理 Bundle 下载加载 |
| **研发工具** | CLI、Playground、Debugger，连接 Talos/DD 发布平台实现持续集成交付 |
| **性能监控** | 接入 Raptor/Metrics，异常上报报警，首屏时长和 FPS 性能指标上报 |

### 4.3 Mach Pro 的四大特点

**特点一：采用 QuickJS 引擎** — JS 执行速度提升 4 倍，通信速度提升 30 倍，内存占用降低 75%，解决首屏展示慢的问题。

**特点二：工作在主线程** — 所有模块（包括 JS 引擎）都工作在主线程。双向事件通信和 UI 操作都是同步的，JS 通过高性能 DOM API 直接同步操作 Native 视图，解决交互动效卡顿和列表白屏问题。

**特点三：支持模块级动态化** — 渲染快、内存占用低，模块级和页面级完全共用一套基建，业务方可先从模块级开始逐步改造，代码完全复用。适用于多业务合作开发页面（如外卖首页）。

**特点四：支持小程序** — 联合 R2X 团队完成小程序侧基建同构，一套前端代码同时运行于 Android、iOS、小程序三端。

---

## 五、核心实现原理

### 5.1 渲染原理 — 三步渲染流程

#### 第一步：编译期 — JSX 转化为 DOM 操作

Mach Pro 在编译期将 JSX 中声明的节点信息转换为对 Element 对象的操作：

- JSX 中声明 `<view>` 节点 → `new Element('view')`
- 子节点 → `element.insert(childElement)` 函数调用
- 节点属性 → `element.attr('property', value)` 函数调用

这一步类似 Svelte 的编译策略：将声明式模板编译为命令式 DOM 操作指令，无需运行时 vdom + diff。

#### 第二步：运行时 — DOM 注入和操作

JS 执行编译期生成的 Element 操作，在 JS 环境中生成 Element 树。Element 继承自 C++ 注入的 Node 对象：

```
JS 侧                          C++ 侧                        Native 侧
Element (继承 Node)     →    Node (C++ 注入)      →     Component
  - new(tag)                  - new() → 创建 Component    - 创建原生视图
  - insert(child)             - insert() → 调用 Native    - 添加子视图
  - attr(key, value)          - attr() → 设置属性         - 设置属性
```

JS 中调用 Element 的方法 → 通过 super 调用 C++ Node 的对应函数 → C++ 调用 Java/OC 创建/操作 Native Component → Native 层构建 Component 树。整个过程同步执行，无线程切换。

#### 第三步：组件测量和视图渲染

Native 层完成 Component 树构建后，调用 Yoga 布局引擎进行组件大小测量和位置确定，然后将布局信息传递给真正的视图控件，交给系统渲染。

### 5.2 Native Component 原理

Node 对象是 JS 与 Native 之间的桥梁：

- **创建时**：根据构造参数创建不同类型的 Native Component 并持有
- **操作时**：调用 Node 不同方法对持有的 Native Component 做相应操作
- **映射关系**：通过 C++ Node 对象完成 JS Element 和 Native Component 的双向映射

继承关系：

```
Node (C++ 注入到 JS)
  └── Element (JS 侧)
        ├── new(tag) → 调用 super.new() → C++ 创建 Native Component
        ├── insert(child) → 调用 super.insert() → C++ 调用 Native insert
        ├── attr(key, value) → 调用 super.attr() → C++ 设置 Native 属性
        └── addEventListener(event) → 调用 super.addEventListener() → Native 注册事件
```

### 5.3 Native Module 原理

Native Module 为 JS 环境提供平台相关能力（网络请求、数据存储、埋点等）。

**核心机制**：通过 JS Binding 实现 JS Module 对象和 Native Module 对象的绑定，将 Native Module 的导出函数注册为 JS Module 对象的同名函数。

**懒加载**：第一次 `requireModule` 时才创建 Module 对象（JS 和 Native 各一个），后续再次获取返回同一个缓存对象。

**调用流程**：

```
JS: const storage = Mach.requireModule('Storage')
     ↓
C++: 创建 JS Module 对象 + Native Module 对象，绑定方法
     ↓
Native: 创建 Storage Module 实例，导出方法签名
     ↓
JS: storage.getNumber("key") → C++ 同步调用 → Native: getNumber 返回结果
```

**方法类型**：
- 同步返回值：`const result = storage.getNumber("参数")`
- 回调函数：`storage.setItem(function(result) {...})`
- 参数 + 回调：`storage.setItemCallBack("参数", function(result) {...})`

### 5.4 事件系统原理

事件系统通过 Element 类中的 listeners 机制实现：

```javascript
class Element extends Node {
  listeners = {};
  
  addEventListener(event, handler, options) {
    if (event && handler) {
      super.addEventListener(event);  // 调用 C++ 在 Native 组件上注册事件
      this.listeners[event] = handler;  // JS 侧记录事件名与处理函数
    }
  }
  
  removeEventListener(event, handler, options) {
    super.removeEventListener(event);
    delete this.listeners[event];
  }
  
  dispatchEvent(event, data) {
    super.dispatchEvent(event, data);  // C++ 侧处理
    const handler = this.listeners[event];  // 查找处理函数
    if (handler) {
      return handler(data);  // 执行 JS 回调
    }
    return null;
  }
}
```

**事件流程**：Native 组件触发事件 → 调用 Element.dispatchEvent(event, data) → 通过 listeners 查找对应处理函数 → 执行 JS 回调。整个过程在主线程同步完成，无线程切换。

---

## 六、性能数据与业务落地

### 6.1 SDK 性能对比（Mach Pro vs MRN）

使用 Mach Pro 重构外卖 App「我的-无效代金券」页面，线上性能测试结果：

首次渲染耗时（从引擎创建到首次布局完成）：

| 方案 | iOS TP90 | Android TP90 |
|------|----------|-------------|
| Mach Pro | 23.95ms | 58.0ms |
| MRN | 124ms | 224ms |
| **倍数** | **5.1x** | **3.8x** |

### 6.2 点菜页业务落地

已完成「满减神器」和「商家信息」两个模块的线上改造：

**满减神器模块**：

| 指标 | Native | Mach Pro | 差异 |
|------|--------|----------|------|
| TP50 首屏 | 575ms | 684ms | +109ms |
| TP90 首屏 | 1025ms | 1185ms | +160ms |
| TP95 首屏 | 1266ms | 1438ms | +172ms |
| 滚动 FPS | 57.96 | 57.47 | 基本持平 |

**商家信息模块**：

| 指标 | Native | Mach Pro | 差异 |
|------|--------|----------|------|
| TP50 首屏 | 199ms | 273ms | +74ms |
| TP90 首屏 | 291ms | 397ms | +106ms |
| TP95 首屏 | 341ms | 457ms | +116ms |

**结论**：
- 首屏渲染时间相比 Native 略有升高（+74~172ms），对于动态化改造带来的开发效率和发板周期收益来说是可接受的
- FPS 和 Native 基本持平，实现了不影响用户滑动体验的前提下完成动态化改造
- 点菜页菜品列表改造预计 V7.76.0 版本上线

### 6.3 建设规划

| 时间 | 目标 | 关键结果 |
|------|------|---------|
| 2021 Q4 | 实现全页面动态化能力 | 点菜页 Mach Pro 局部验证；React DSL & 小程序基建三端复用 |
| 2022 Q1 | 适配 Mach 1.0 模板 | 多业务单独发版能力；点菜页 React DSL 全页面 Demo |
| 2022 Q2 | 点菜页全页面动态化 | 整个页面动态化；Mach 1.0 模板迁移方案 |
| 2022 Q3 | 外卖主路径动态化 | 外卖主路径全页面动态化能力 |

---

## 七、行业方案对比

| 方案 | 语言 | 引擎 | 前端框架 | 工作线程 | V-DOM | 通信方式 |
|------|------|------|---------|---------|-------|---------|
| Mach Pro | JS | **QuickJS** | **Svelte/Preact** | **主线程** | **无** | **C 函数** |
| React Native | JS | JSCore/V8/Hermes | React | 子线程 | JS 侧 | JSI（序列化） |
| Hippy | JS | JSCore/V8 | Vue/React | 子线程 | JS 侧 | JSI（NSDictionary） |
| Lynx | JS | JSCore/V8 | Vue | 子线程 | Native 侧 | JSI（NSDictionary） |
| MLN（陌陌） | Lua | Lua | 无 | 主线程 | 无 | C 函数 |
| Hummer（滴滴） | JS | JSCore/QuickJS | 无 | 主线程 | 无 | JSI（序列化） |
| Picasso | JS | JSCore/V8 | 无 | 子线程 | JS 侧 | JSI（NSDictionary） |

Mach Pro 的独特之处在于：**唯一同时具备"JS 语言 + 主线程运行 + 无 V-DOM + C 函数通信 + 声明式 DSL + 运行时框架"的方案**。其他主线程方案（MLN/Hummer）要么使用 Lua 语言，要么缺少运行时框架。

---

## 八、面试话术与核心考点

### 8.1 一分钟项目介绍（简历用）

"我参与了美团外卖 Mach Pro 高性能动态化框架的研发。该框架解决了外卖核心页面（如点菜页）在动态化改造中遇到的性能瓶颈——现有方案 Mach 能力不足、MRN 性能不达标。Mach Pro 通过三大技术创新（QuickJS 引擎替代 JSCore/V8、轻量级 React 框架去除 vdom+diff、C++ DOM 注入实现主线程同步通信），使 JS 执行速度提升 4 倍、通信速度提升 30 倍、内存占用降低 75%。在点菜页落地中，FPS 与 Native 基本持平，首屏时间仅增加约 100ms，实现了接近原生的动态化体验。"

### 8.2 面试核心考点

**Q1: 为什么选择 QuickJS 而不是 Hermes？**

QuickJS 纯 C 编写，体积小、执行快、内存低，最重要的是支持字节码预编译和 C 函数级别的 JS↔Native 通信（通信速度是 JSCore 的 29 倍、V8 的 35 倍）。Hermes 虽然也支持字节码，但通信仍需通过 JSI 序列化，且体积更大。

**Q2: Mach Pro 为什么能在主线程运行 JS？不会卡顿吗？**

关键在于 QuickJS 执行极快 + 轻量级 React 去除了 vdom+diff 的开销。传统 RN 必须在子线程运行是因为 vdom+diff 太耗时且桥通信序列化开销大，主线程扛不住。Mach Pro 的 JS 执行在 50 个模块 10 万行代码的场景下仅需 84ms（iOS），单次渲染操作在毫秒级，不会造成卡顿。

**Q3: 如何解决 RN 的长列表白屏问题？**

三个层面：1) QuickJS 通信速度极快（2.46ms vs JSCore 72.33ms），列表滚动时 Cell 创建/更新不再有通信延迟；2) 主线程同步执行，无异步渲染导致的渲染不及时；3) 列表组件可直接使用原生 UITableView/RecyclerView 封装，RN 无法这样做是因为 UITableView 接口不适合 RN 的异步特性。

**Q4: Mach Pro 和 Svelte 的关系？**

调研阶段受 Svelte 启发——通过静态编译减少运行时代码量，将模板编译为命令式 DOM 操作。Mach Pro 最初方案是 Svelte.JS + QuickJS VM，后演进为自研轻量级 React（R2X 团队）+ Preact，核心思想一致：编译期将 JSX 转为 DOM API 指令，运行时无 vdom+diff。

**Q5: C++ DOM 注入具体怎么实现的？**

在 JS 环境创建后，通过 QuickJS 的 C API 向 JS 全局环境中注入 Node Class。Node 类提供了操作视图结构、样式、内容的方法（new/insert/attr/addEventListener 等）。JS 侧的 Element 继承 Node，调用 Element 方法时通过 super 调用 C++ Node 对应函数，C++ 再调用 Java/OC 操作 Native Component。整个过程是 C 函数直接调用，无序列化/反序列化。

**Q6: 为什么不用 BindingX 解决交互卡顿？**

BindingX 通过表达式绑定减少通信次数，但双端适配成本大、存在 bind 不上的异常情况，且只在手势场景有效，不解决首屏和长列表问题。Mach Pro 从根本上解决了通信慢的问题（C 函数级别同步调用），不需要额外的表达式绑定方案。

**Q7: 模块级和页面级如何共用一套基建？**

Mach Pro 渲染快、内存占用低，单个模块的 JS 执行和内存开销极小（QuickJS 内存仅 6.75MB），因此可以作为模块嵌入 Native 页面，也可以作为独立页面运行。两种模式使用同一套 JS Framework、JS Engine、Native 组件，业务方可以从模块级开始逐步改造，最终实现整页动态化，代码完全复用。

### 8.3 简历技术关键词

`动态化框架` `QuickJS` `Preact` `主线程渲染` `C++ DOM 注入` `JS Binding` `字节码预编译` `vdom 消除` `Yoga 布局` `三端同构` `性能优化` `首屏优化` `长列表优化` `跨平台`

---

## 九、学习笔记速查

### 9.1 RN 三大问题的因果链

```
JS 引擎执行慢 ──→ 首屏时间长
                   ↗
React 框架慢 ──→ vdom+diff 耗时
                 ↗
通信慢 ←── 引擎通信慢 + 线程切换慢 ──→ 交互动效卡顿 + 长列表白屏
```

### 9.2 Mach Pro 解决方案映射

```
QuickJS 引擎（字节码）────→ 解决 JS 引擎执行慢
Preact/自研轻量 React ──→ 解决 React 框架执行慢（无 vdom+diff）
C++ DOM 注入 + 主线程 ──→ 解决通信慢（C 函数直接调用，无线程切换）
```

### 9.3 关键性能数据速记

| 指标 | 数值 |
|------|------|
| QuickJS vs JSCore 总时间（iOS） | 4.07 倍快 |
| QuickJS vs V8 总时间（Android） | 3.33 倍快 |
| QuickJS 通信速度提升 | 29~35 倍 |
| QuickJS 内存降低 | 75%（4.12 倍低） |
| Mach Pro vs MRN 首屏（iOS） | 5.1 倍快 |
| Mach Pro vs MRN 首屏（Android） | 3.8 倍快 |
| 点菜页 FPS vs Native | 基本持平（57.5 vs 58.0） |
| 点菜页首屏增加 | +74~172ms（可接受） |

### 9.4 核心架构记忆图

```
JS Framework:  React → DOM API / Module API
                    ↓
JS Engine:     Node / Mach Object → QuickJS
                    ↓ (C 函数同步调用)
Native:        Component / Module → 原生视图
```

### 9.5 关键设计决策总结

1. **选 QuickJS 不选 Hermes**：C 函数级通信 + 更小体积 + 字节码支持
2. **去 vdom 不去 React**：保留 React 开发体验，编译期消除 vdom+diff
3. **主线程不子线程**：QuickJS 足够快，同步通信无延迟，彻底解决线程切换
4. **C++ 中间层**：跨平台复用 + 高性能 DOM API 注入 + JS↔Native 桥梁
5. **模块级 + 页面级统一**：一套基建两种粒度，线性改造成本
6. **三端同构**：Android + iOS + 小程序，一套代码复用

---

> **备注**：本文档基于美团内部学城文档整理，仅供个人学习和面试准备使用。文中性能数据来自内部测试，具体数值可能随版本迭代有所变化。
