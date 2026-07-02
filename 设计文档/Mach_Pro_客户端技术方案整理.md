# Mach Pro 客户端总体技术方案 — 深度整理

> **来源文档**：[Mach Pro 总体技术方案](https://km.sankuai.com/collabpage/1576728972)（含 11 篇子文档）  
> **用途**：技术学习 + 面试素材  
> **视角**：客户端（Native/C++）实现视角

---

## 目录

- [一、SDK 整体结构](#一sdk-整体结构)
- [二、Element 与 Component 绑定机制](#二element-与-component-绑定机制)
- [三、Event Loop 实现](#三event-loop-实现)
- [四、Component 方法调用](#四component-方法调用)
- [五、Native Module 机制](#五native-module-机制)
- [六、JS Module 机制](#六js-module-机制)
- [七、事件通信与数据更新](#七事件通信与数据更新)
- [八、主流程梳理](#八主流程梳理)
- [九、分包加载方案](#九分包加载方案)
- [十、多线程技术方案（JS 线程）](#十多线程技术方案js-线程)
- [十一、面试要点](#十一面试要点)

---

## 一、SDK 整体结构

Mach Pro SDK 分为三层，从上到下依次为 JS Framework、JS Engine、Native，每层职责清晰。

### 1.1 三层架构

**JS Framework 层**（绿色）：与业务代码直接交互。包含 React/Preact 运行时、DOM API 的 JS 侧封装（Element 类）、Module API 的 JS 侧封装、内置复杂组件（List/Swiper）的 JS 逻辑。这层的代码运行在 QuickJS 引擎中。

**JS Engine 层**（蓝色）：提供 JS 运行时环境。核心是用 C 语言实现的 Node 类——它是 JS 与 Native 之间的桥梁。Node 类被注入到 QuickJS 的 JSContext 中，JS 侧的 Element 继承自 Node。这层还包含 JS 异常收集和 Event Loop 实现。

**Native 层**（黄色）：各平台原生视图组件的封装。JS 中对 DOM 的操作最终都通过这层转化为对原生 UI 组件的操作。负责组件测量布局（Yoga）、系统事件接收分发、Bundle 下载加载管理。

### 1.2 关键设计：C++ 作为中间层

C++ 层是整个方案的核心枢纽。它向上通过 QuickJS C API 向 JS 环境注入 Node Class 和 Mach 对象，向下通过 JNI/OC Runtime 调用 Native 层的 Component 和 Module。所有跨语言调用都是 C 函数级别的直接调用，无序列化/反序列化开销。

---

## 二、Element 与 Component 绑定机制

这是 Mach Pro 最核心的设计，理解了它就理解了整个框架的工作原理。

### 2.1 绑定三步走

**第一步：C 层实现 Node 类并注入 JS 环境**

Native 侧用 C 实现 Node 类（参考 Web 标准 [Node API](https://developer.mozilla.org/zh-CN/docs/Web/API/Node)），通过 QuickJS C API 将 Node Class 注入到 JSContext。Node 类提供了操作视图结构、样式、内容的方法。

**第二步：JS 侧 Element 继承 Node**

JS 侧的 Element 类继承自 C 层注入的 Node 类。JS 调用 Element 的方法时，Element 内部通过 super 调用父类 Node 的同名方法。这意味着 JS 侧的每一次 DOM 操作，都会同步穿透到 C 层。

**第三步：Node 持有 Native Component**

C 层的 Node 对象在创建时，会根据构造参数创建对应的 Native Component（OC/Java 层的 UI 组件）并持有。当 JS 调用 Node 的方法时，C 层直接操作持有的 Native Component，完成视图的创建、属性设置、样式设置、添加子视图等操作。

### 2.2 SetOpaque 绑定原理

JS 对象和 Native 对象的绑定通过 SetOpaque 实现。原理很简单：在 JS 对象内部保存一个指向 Native 对象的指针。这样 C 代码可以从 JS 对象取出 Native 指针，直接操作 Native 对象。

```
JS 侧                          C 层                          Native 侧
Element (继承 Node)     →    Node (C 实现)          →     Component
  new('view')                  创建 Node 对象               创建 UIView/View
  insert(child)                取出 child 的 Component      addSubview
  attr('color','red')          取出 Component              setBackgroundColor
```

整个过程同步执行，从 JS 调用到 Native 视图更新，没有线程切换，没有序列化。

---

## 三、Event Loop 实现

### 3.1 为什么需要 Event Loop

Svelte/Preact 中通过 Promise 微任务实现批量更新 UI，而 Promise 依赖 Event Loop。QuickJS 的 quickjs-libc.c 中有一个极简版 Event Loop（js_std_loop），但测试发现它会阻塞当前线程，无法满足需求。

### 3.2 自定义 Event Loop 方案

Event Loop 的核心作用是执行 Pending Job（微任务）。Mach Pro 的方案是定时检查并执行 Pending Job，不需要完整的 Event Loop。

**iOS 方案 — Runloop**

通过 CFRunLoopObserver 监听主线程 Runloop 的 BeforeTimers、BeforeSources、BeforeWaiting、AfterWaiting 事件，在每次回调时执行 Pending Job：

```objc
- (void)executePendingJob {
    while (JS_IsJobPending(_rt)) {
        JSContext *ctx;
        int ret = JS_ExecutePendingJob(_rt, &ctx);
        if (ret < 0) {
            [self dumpException:ctx];
        }
    }
}
```

**Android 方案 — FrameCallback**

通过 Choreographer.postFrameCallback 在每帧回调时执行 Pending Job：

```java
Choreographer.getInstance().postFrameCallback(new Choreographer.FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        if (isDestroyed) return;
        if (mJSContext != null) {
            mJSContext.executePendingJob();
        }
        Choreographer.getInstance().postFrameCallback(this);
    }
});
```

两种平台的本质都是：在系统每帧渲染前，检查并执行所有待处理的 JS 微任务。

---

## 四、Component 方法调用

### 4.1 背景

对于 `<list>`、`<video>` 等复杂组件，仅靠属性设置无法满足需求，需要提供方法调用能力（如 list.reloadData()）。

### 4.2 使用方式

JS 侧通过 `bind:this` 获取组件引用，然后直接调用组件方法：

```javascript
let logoImageView;
function onClickLogo() {
  const result = logoImageView.doSomething('abc');
}
<image bind:this={logoImageView} on:click={onClickLogo} />
```

支持任意参数、同步返回值和异步回调。

### 4.3 实现机制

**注册阶段**：Native 通过 `MP_EXPORT_COMPONENT` 宏将 Component 类导出。SDK 初始化时一次性读取所有 Component 类，将名称和方法列表保存到 MPComponentFactory 单例。方法列表包含方法名、参数个数、是否有回调等信息。

以 list 组件为例：

```objc
MP_EXPORT_COMPONENT(@"list");
+ (NSArray *)exportMethods {
    return @[ MP_EXPORT_METHOD(@"reloadData", 0) ];
}
- (void)reloadData:(JSContext *)ctx argc:(int)argc argv:(JSValueConst *)argv { }
```

**创建阶段**：创建 Element 对象时调用 Node 构造函数，在构造函数中：1) 创建 Native Component 对象；2) 通过 SetOpaque 绑定 JS Node 对象与 Native Component；3) 根据方法列表为 Node 对象添加方法。

**Magic 值机制**：为 Node 对象添加的所有方法都对应同一个 C 函数实现。每个方法在方法列表中的索引被设置为该方法的 magic 值。C 函数被调用时，通过 magic 值就能精确找到对应的 Native 方法：

```c
static JSValue js_node_callMethod(JSContext *ctx, JSValueConst this_val, 
                                   int argc, JSValueConst *argv, int magic) {
    // magic 值即方法在方法列表中的索引
}
```

**调用阶段**：JS 中调用 Element 方法 → 走到 Node 的 C 函数 → 取出绑定的 Component 对象 → 根据 magic 值取出 Native 方法信息 → 通过反射调用 Native 方法。

---

## 五、Native Module 机制

### 5.1 设计目标

为 JS 环境提供平台相关能力（网络请求、数据存储、埋点等）。Module 就是 JS 对象，使用方式直观：

```javascript
const storage = Mach.requireModule('storage');
storage.setItem('key', 'value');
const result = storage.getItem('key');
```

三个设计原则：Module 即对象、支持任意参数和同步/回调返回、懒加载 + 缓存。

### 5.2 完整工作流程

**注册阶段**：Native 通过 `MP_EXPORT_MODULE` 宏导出 Module 类。SDK 初始化时读取所有 Module 类，将名称和方法列表保存到 MPModuleFactory 单例。

以 WMNetwork 为例：

```objc
MP_EXPORT_MODULE(@"WMNetwork");
+ (NSArray *)exportMethods {
    return @[
        MP_EXPORT_METHOD(@"request", 1),       // 方法名 + 参数个数
        MP_EXPORT_METHOD(@"waimaiBaseURL", 0),
    ];
}
```

**requireModule 阶段**：

1. C 层在 JSContext 中创建了一个 JS 对象作为 Map，专门缓存 Module 对象（key=Module 名称，value=Module 对象）
2. 每次 requireModule 优先从缓存取
3. 缓存未命中时创建 JS Module 对象（空对象 `{}`），同时在 OC/Java 层创建 Native Module 对象
4. 通过 SetOpaque 绑定 JS Module 对象和 Native Module 对象
5. Native Module 由 MPModuleManager 持有，MPModuleManager 由 MPInstance 持有，生命周期与 MPInstance 一致
6. 根据方法列表为 JS Module 对象添加方法（同样使用 magic 值机制）

**方法调用阶段**：JS 调用 Module 方法 → C 函数取出绑定的 Native Module 对象 → 根据 magic 值找到对应方法 → 反射调用 Native 方法。

**销毁阶段**：MPInstance 销毁时，MPModuleManager 释放，所有 Native Module 对象随之销毁。

### 5.3 最终 JS 环境中的 Module 结构

```javascript
// WMNetwork Module 在 JS 中的结构
{
    "request": function(param) {},       // 1个参数
    "waimaiBaseURL": function() {},      // 0个参数
}
```

---

## 六、JS Module 机制

### 6.1 背景

Native Module 是 JS 调 Native，反过来 Native 也需要调用 JS（如 Native 从 JS 获取数据）。这就是 JS Module。

### 6.2 与事件的区别

| 通信方式 | Native→JS | JS→Native |
|---------|-----------|-----------|
| 事件 | 一对多，无返回值 | 一对一，无返回值 |
| Module | 不支持 | 一对一，有返回值 |

JS Module 的根本是方法调用，可以有返回值；事件没有返回值。JS Module 不支持回调函数（因为 JS 中极少有异步情况）。

### 6.3 实现

**JS 侧导出 Module**：

```javascript
// base.js
const __js_modules__ = {};
Mach.exportModule = function(moduleName, methods) {
    if (moduleName && methods) {
        __js_modules__[moduleName] = methods;
    }
}

// 业务代码
Mach.exportModule("foodList", {
    getFoodName: (spuId) => { return "鸡翅"; }
})
```

**JS 侧接收 Native 调用**：

```javascript
Mach.callModule = function(moduleName, methodName, params) {
    const module = __js_modules__[moduleName];
    if (module) {
        const method = module[methodName];
        if (method) {
            return params ? method(...params) : method();
        }
    }
    return undefined;
}
```

**Native 调用 JS Module**（iOS 为例）：

```objc
NSString *foodName = [self.instance callJSModule:@"foodList" 
                                          method:@"getFoodName" 
                                          params:@[@(10086)]];
```

Native 侧通过 C 函数获取 JS 全局对象的 Mach 对象，调用 `Mach.callModule` 方法，传入 Module 名称、方法名称和参数，同步获取返回值。

---

## 七、事件通信与数据更新

### 7.1 事件通信四象限

Mach Pro 支持四个方向的事件通信：

**JS 订阅 Native 事件**：`Mach.on("eventName", callback)`

base.js 中实现：1) 保存回调函数到 `__event_handlers__`（数组，支持一对多）；2) 调用 Native 的 `Mach.subscribeEvent` 真正订阅事件。

Native 侧的 subscribeEvent 仅记录事件名到 `_subscribedEvent` 数组。发送事件前检查 JS 是否已订阅，只有已订阅时才回调，避免无用通信。

**JS 向 Native 发送事件**：`Mach.sendEvent("hello", {aaa: "bbb"})`

Native 实现，直接将事件回调给业务层（`instance.onReceiveEvent`），SDK 不做额外处理。

**Native 接收事件**：通过 MPInstance 的 block 回调：

```objc
@interface MPInstance : NSObject
@property (nonatomic, copy, nullable) void (^onReceiveEvent)(
    NSString *eventName, NSDictionary *_Nullable params);
@end
```

**Native 向 JS 发送事件**：先检查 `_subscribedEvent` 是否包含该事件名，已订阅时通过 C 函数调用 JS 的 `Mach.receiveEvent`，传入事件名和参数。JS 侧的 receiveEvent 从 `__event_handlers__` 取出所有回调函数并执行。

### 7.2 数据更新

数据更新指首次渲染后 Native 重新向 JS 设置数据，适用于混排列表场景（每个 cell 有一个 MPInstance，cell 复用时新数据更新 UI）。

基于事件机制实现：

```javascript
// JS 侧
let api = Mach.data;  // 首次数据
Mach.on("dataChange", function() {
    api = Mach.data;  // 更新数据，UI 自动更新
});
```

```objc
// Native 侧
- (void)refreshWithData:(NSDictionary *)data {
    NSAssert([NSThread isMainThread], @"必须在主线程执行");
    if (!self.jsContext) return;  // 必须先完成首次渲染
    _data = data;
    [self.jsContext setData:_data];                    // 1. 更新 Mach.data
    [self.jsContext sendEvent:@"dataChange" params:nil]; // 2. 通知 JS 数据变化
}
```

Native 先更新 JS 全局对象 `Mach.data`，再发送 `dataChange` 事件通知 JS 重新读取数据并更新 UI。

---

## 八、主流程梳理

### 8.1 渲染流程

1. 启动 Activity，加载 Bundle，开始渲染
2. 创建 JSRuntime，注入 NodeClass、Mach 对象、全局函数、渲染参数、env 参数、scheme 参数
3. 执行字节码
4. JS Framework 层创建 DOM 对象，更新 DOM 属性，同步创建 Native 层的 MPComponent 和视图，同步更新视图属性
5. 字节码执行结束，Native 输出一棵 Component 树和视图树，根视图是 BodyView
6. 将 BodyView 挂载到 Container 上，完成渲染

### 8.2 事件响应流程

**添加事件**：
1. JS Framework 调用 `addEventListener`，参数是事件名（如 click）和回调函数
2. JS Framework 保存事件名和回调函数到 listeners，同时调用 Native 层 MPComponent 的 addEventListener，将 EventName 传递给 Native
3. Native 的 addEventListener 中根据 EventName 添加对应的事件监听（如 Click 事件添加点击手势/OnClickListener）

**事件响应**：
1. 用户点击视图，系统回调事件监听函数
2. 事件监听函数中执行 JS Framework 的 `dispatchEvent` 函数，传入 EventName 和业务参数
3. dispatchEvent 中根据 EventName 从 listeners 中找到之前保存的回调函数
4. 执行回调函数，传递业务参数

### 8.3 Native Module 调用流程

**获取 Module 对象**：
1. 业务调用 `Mach.requireModule(moduleName)`（C++ 层实现）
2. 检查缓存：已创建则直接返回，否则创建新 JS 对象
3. 通过 MPBridge 调用 Java/OC 获取该 Module 导出的方法列表（通过反射获取 `@JSMethod` 标记的函数）
4. C++ 遍历方法数组，为 JS 对象挂载同名函数（函数实现在 C++）
5. 返回 JS 对象给业务

**调用 Module 方法**：
1. 业务通过 Module 对象调用函数（C++ 层实现）
2. C++ 解析参数，根据 Method Magic 在方法数组中找到对应的 Method Name 和参数个数
3. 反射调用 Java/OC 层对应的 Module Method

---

## 九、分包加载方案

### 9.1 分包目的

Mach Pro 分包与 MRN/小程序分包目的不同：

| 方案 | 目的 |
|------|------|
| MRN | 解决多个 Bundle 间重复代码问题，抽取公共库 |
| 小程序 | 优化首启下载时间，多团队解耦 |
| Mach Pro | 解决不同业务方子模块修改导致整页重新发包的问题 |

Mach Pro 子包是页面中的一个模块，子模块数据由页面提供，因此子包依赖特定版本的主包。

### 9.2 配置文件

子包通过 `mach.config.js` 配置：

```javascript
module.exports = {
    "biz": "waimai",
    "page": "vip-card",
    "bundleType": 1,  // 0=主包, 1=子包
    "dependencies": [{
        "mainBundleName": "xxxx",  // 依赖的主包名
        "upperVersion": "",         // 上界（含）
        "lowerVersion": "0.0.3"     // 下界（含）
    }]
};
```

### 9.3 加载流程

主包 JS 调用 `Mach.requireBundle` → Native 查询对应文件夹 → 版本从高到低排序 → 逐个判断是否与主包版本匹配 → 找到匹配的 Bundle → 将其 JS 加载到执行环境 → 回调 JS 侧 SDK → JS 层 SDK 控制子包执行。

Mach Pro 执行主流程不变，变化仅在执行 JS 阶段内部通过 requireBundle 加载子包。

### 9.4 版本管理

**新增子包**：保留 5 个历史版本，以便主包降级时加载旧子包。主包始终用最新版本。

**删除旧包**：App 退后台时遍历 Bundle，删除标记废弃的主包和非前 5 版本的子包。删除前排查正在使用的 Bundle。

**下线**：checkupdate 接口未返回的 Bundle 标记废弃，App 退后台时删除。

---

## 十、多线程技术方案（JS 线程）

### 10.1 背景

单线程模型在传统列表页表现良好，但高达页面（多个独立列表卡片平铺在一个 scroller 中，一次性渲染）计算量暴增，主线程被长时间占用，FPS 下降严重（高端机 <50，低端机 ~30）。

JS 执行耗时是 Native 的 2 倍以上，占比 70%，复杂卡片单卡片总耗时接近 10ms（16ms 上限），优化空间有限。需要引入多线程模型。

### 10.2 核心思路

Mach Pro 提供给 JS 的能力分两块：DOM API 和 Module API。

**Module API**：绝大部分与 UI 无关，可以直接在 JS 线程执行。少部分与 UI 有关的 Module 需做兼容，切换到主线程执行。Module 需实现 `supportJSThread` 方法声明支持 JS 线程：

```objc
@implementation MPLoganModule
+ (BOOL)supportJSThread { return YES; }
@end
```

**DOM API**：全部与 UI 有关，iOS/Android 不允许子线程操作 UI，所有 DOM API 改为异步执行（与 Web 标准不同，Web 中 DOM API 是同步的）。异步执行时不等待主线程任务完成，直接返回。

### 10.3 API 改动详情

| 分类 | API | 多线程处理 |
|------|-----|----------|
| 标准 DOM API | createElement/appendChild/setAttribute 等 | 异步执行，JS 线程创建 Node，主线程创建/操作 Component |
| 扩展 DOM API | measure/measureInView/measureInWindow | 同步执行时返回 null（不在主线程），新增 measureAsync 系列异步方法 |
| Component 导出方法 | 各组件自定义方法 | 异步执行，已有同步返回改为 callback 回调 |
| Native Module | requireModule | 同步执行，Module 内部兼容主线程和 JS 线程 |
| 事件 | subscribeEvent/unsubscribeEvent | 同步执行 |
| 事件 | sendEvent | 异步执行，主线程回调 |
| Web API | console.log/setTimeout/setInterval | 同步执行 |
| Web API | alert/requestAnimationFrame | 异步执行，需主线程展示/回调 |

### 10.4 Yoga 布局时机调整

**单线程模型**：JS EventLoop = 主线程 RunLoop，修改组件属性后直接在下一个 RunLoop 做 Yoga 布局。

**多线程模型**：JS 线程 RunLoop 与主线程 RunLoop 不同步，直接在主线程 RunLoop 布局会导致多次无效布局（JS 还没执行完，不是最终结果）。

**解决方案**：将布局时机与 JS 线程 RunLoop 关联。主线程修改组件属性时，提交任务到 JS 线程，在 JS 线程下一个 RunLoop 执行任务，然后回调主线程进行 Yoga 布局。

### 10.5 开关方式

支持 scheme 参数（`mach_js_thread=1`）或 mach.config.js 配置（`runInJSThread: true`），两者为或关系。

### 10.6 性能数据

**iOS 高端机（iPhone 13 Pro）**：

| 指标 | 单线程 | 多线程 | 提升 |
|------|--------|--------|------|
| 第一页渲染耗时 | 155.96ms | 137.46ms | ↓11.86% |
| 第二页渲染耗时 | 163.11ms | 134.90ms | ↓17.30% |
| 加载更多 FPS | 46 | 57 | +9 |

**iOS 低端机（iPhone 7 Plus）**：

| 指标 | 单线程 | 多线程 | 提升 |
|------|--------|--------|------|
| 第一页渲染耗时 | 463.61ms | 453.16ms | ↓2.25% |
| 第二页渲染耗时 | 480.21ms | 508.50ms | ↑5.89% |
| 加载更多 FPS | 33.33 | 48.67 | **+15.34** |

低端机 FPS 提升尤为显著（+15.34），说明多线程模型有效释放了主线程压力。

### 10.7 包大小代价

iOS 增加 86KB。

---

## 十一、面试要点

### Q1: SetOpaque 绑定的原理是什么？为什么要用这种方式？

SetOpaque 是 QuickJS 提供的 API，原理是在 JS 对象内部保存一个指向 Native 对象的指针。这样 C 代码可以从 JS 对象直接取出 Native 指针，实现 JS 对象与 Native 对象的一一映射。选择这种方式是因为它是最轻量的绑定方式——无需维护额外的映射表，无需序列化，指针直接访问，性能最优。

### Q2: Magic 值机制是怎么工作的？为什么所有方法共用一个 C 函数？

QuickJS 的 JS_NewCFunctionMagic 允许为函数设置一个 magic 值，调用时会带回这个值。Mach Pro 为 Module/Component 的每个方法设置方法列表索引为 magic 值，所有方法共用同一个 C 函数实现。C 函数被调用时通过 magic 值在方法列表中查找对应的方法信息（名称、参数个数），然后反射调用 Native 方法。这样设计是因为：1) 减少注册的 C 函数数量；2) 方法信息集中在方法列表中管理，便于维护；3) 统一的调用入口便于做参数解析和异常处理。

### Q3: 单线程模型和多线程模型如何选择？

通过 scheme 参数或 mach.config.js 配置切换。单线程模型适用于传统列表页（JS 执行占比高但单次渲染量可控），优势是同步通信无延迟。多线程模型适用于复杂页面（如高达页面，多个列表卡片一次性渲染），将 JS 执行放到独立线程释放主线程压力，代价是 DOM API 变为异步、有通信延迟、包大小增加 86KB。低端机 FPS 提升 15+ 的收益证明多线程模型在复杂场景下是必要的。

### Q4: 事件通信中为什么要先 subscribeEvent 再 sendEvent？

Native 侧维护了 `_subscribedEvent` 数组，只有 JS 已订阅的事件才会被回调。这是一个优化设计——避免 Native 向 JS 发送无用的事件，减少 JS↔Native 通信次数。JS 侧的 Mach.on 先在本地保存回调函数，然后调用 Native 的 subscribeEvent 注册事件名。Native 发送事件时先检查 `_subscribedEvent`，已订阅才通过 C 函数调用 JS 的 receiveEvent。

### Q5: 分包加载中子包版本如何管理？

子包保留 5 个历史版本（主包始终用最新版本）。加载时从高版本到低版本逐个判断是否与当前主包版本匹配（通过 meta.json 中的 lowerVersion/upperVersion 闭区间判断）。保留多版本是因为子包可独立更新，主包降级或未更新成功时需要加载旧版子包。App 退后台时清理废弃版本，删除前会排查正在使用的 Bundle。

### Q6: 多线程模型中 Yoga 布局时机为什么要调整？

单线程模型中 JS EventLoop 就是主线程 RunLoop，属性修改后下一个 RunLoop 布局即可。多线程模型中 JS 线程 RunLoop 与主线程 RunLoop 不同步，JS 可能还在执行中，主线程就已经触发了多次无效布局。解决方案是将布局任务提交到 JS 线程，在 JS 线程下一个 RunLoop 执行后再回调主线程做 Yoga 布局，确保布局基于 JS 执行完毕的最终结果。

### Q7: JS Module 和 Native Module 的区别？

Native Module 是 Native 封装能力暴露给 JS（JS→Native），支持同步返回和异步回调，懒加载+缓存。JS Module 是 JS 导出方法供 Native 调用（Native→JS），支持同步返回值但不支持回调（JS 中极少异步情况）。两者都是一对一关系，而事件是一对多关系且无返回值。通信对比表：事件支持 Native→JS（一对多）和 JS→Native（一对一）但无返回值；Module 仅支持 JS→Native（一对一）有返回值，Native→JS 的 Module 不支持。

---

> **备注**：本文档基于美团内部学城文档整理，仅供个人学习和面试准备使用。
