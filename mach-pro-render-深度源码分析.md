# mach-pro-render 深度源码分析报告

> 包名: `@wmfe/mach-pro-render` v3.6.2-next.61
> 定位: MachPro 跨端渲染框架的 **Native DOM 模拟层**，在客户端 JS 引擎中模拟 W3C DOM API，使 React 等 Web 框架能够直接运行在 Native 渲染引擎之上。

---

## 整体架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                    React / 业务代码                              │
│          (createElement / JSX → DOM API 调用)                    │
├─────────────────────────────────────────────────────────────────┤
│                  mach-pro-render (本包)                          │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────────┐  │
│  │  Window   │ │ Document │ │ Element  │ │ elements-shim     │  │
│  │ (window.ts)│ │(document)│ │(element) │ │(jsx/ScrollView/..)│  │
│  └──────────┘ └──────────┘ └──────────┘ └───────────────────┘  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │  Style    │ │  Event   │ │  Mach    │                        │
│  │ (style.ts)│ │(event/*) │ │(mach.ts) │                        │
│  └──────────┘ └──────────┘ └──────────┘                        │
├─────────────────────────────── JS / Native 边界 ────────────────┤
│             Native Node (客户端 C++/OC/Java 层)                  │
│    - Node.appendChild / insertBefore / removeChild              │
│    - Node.setAttribute / removeAttribute                        │
│    - Node.addEventListener / removeEventListener                │
│    - Node.dispatchEvent                                         │
│    - Node.measureInWindow                                       │
│    - CSSStyleDeclaration.setProperty / removeProperty           │
│    - Mach.env / Mach.requireModule / Mach.subscribeEvent ...    │
└─────────────────────────────────────────────────────────────────┘
```

**核心设计思想**: 用 JS 类继承 Native 注入的 `Node` 基类，在 JS 层维护一套 DOM 树 (`childNodes`/`parentNode`)，同时通过 `super.xxx()` 调用将变更同步到 Native 渲染树。两棵树保持镜像同步，JS 层负责 API 兼容，Native 层负责实际渲染。

---

## 1. 入口文件 — `src/index.ts`

```typescript
// 第1行
import './window';
```

**整个入口只有一行副作用导入。** 执行 `import './window'` 时，`window.ts` 模块的顶层代码会立即执行：
1. 创建 `Window` 实例
2. 通过 `getClassKeys()` 遍历 Window 原型链上的所有属性
3. 将所有属性挂载到 `globalThis` 上

被注释掉的 `console.log` 重写（第3-22行）显示了一个历史设计：Native JS 引擎的 `console.log` 可能不支持对象类型输出，需要序列化为字符串。当前已禁用，说明 Native 侧已提供了更好的日志支持。

**关键理解**: 这是一个"纯副作用"包——`import '@wmfe/mach-pro-render'` 后，全局环境就被改造为"浏览器兼容"状态。

---

## 2. Window 模拟 — `src/window.ts`

### 2.1 Location 类 (第8-48行)

```typescript
class Location {
    host: string = "";
    hostname: string = "";
    hash: string = "";
    href: string = "";
    origin: string = "";
    port: string = "";
    protocol: string = "";
    pathname: string = "";
    search: string = "";
    params: Record<string, string>;  // 第20行: 非标准属性，保存 scheme 参数

    constructor(href: string, params: any) {
        this.href = href;
        this.params = params || {};
        // 第25-27行: 手动解析 URL，不依赖正则（Native JS引擎可能不支持完整正则）
        // 支持三种格式:
        //   web:    http://domain.com/pathname?params
        //   小程序:  /pathname?params
        //   native: imeituan://domain.com/pathname?params
```

**解析算法**（第29-46行）:
1. 先找 `?` 分离 search 部分
2. 再找 `://` 分离 protocol
3. 最后找第一个 `/` 分离 host 和 pathname
4. 拼接 origin = `protocol//host`

**JS/Native 边界**: `href` 和 `params` 来自 `Mach.env.scheme` 和 `Mach.env.schemeParams`，是 Native 容器启动时注入的页面入口信息。`imeituan://` 是美团 App 的自定义 scheme。

### 2.2 定时器安全包装 (第50-55行)

```typescript
const originalSetTimeout = globalThis.setTimeout;
const originalSetInterval = globalThis.setInterval;
// 包装: 确保 time 参数有默认值 0
const safeSetTimeout = (globalThis.setTimeout = (cb, time = 0) => originalSetTimeout(cb, time));
const safeSetInterval = (globalThis.setInterval = (cb, time = 0) => originalSetInterval(cb, time));
```

**设计目的**: Native JS 引擎的 `setTimeout`/`setInterval` 可能不像浏览器那样对 `undefined` 的 delay 参数默认取 0，这里做了安全兜底。这是跨端框架常见的"引擎差异抹平"手段。

### 2.3 Window 类 (第60-149行)

```typescript
export class Window extends EventTarget {
    // 第61-65行: 将 DOM 构造函数挂载为 window 的属性（模拟浏览器的 window.Event 等）
    CustomEvent = CustomEvent;
    Event = Event;
    Document = Document;
    Element = Element;
    EventTarget = EventTarget;

    self = this;  // 第70行: window.self === window

    location: Location;
    document = new Document();  // 第74行: 创建全局 document

    // 第79-93行: 屏幕尺寸参数，从 Mach.env 获取
    outerHeight = 0;
    outerWidth = 0;
    innerHeight = 0;
    innerWidth = 0;
    devicePixelRatio = 0;
```

**构造函数**（第105-121行）:

```typescript
    constructor() {
        super();
        if (!Mach) return;  // 防御: 非 Native 环境不初始化
        const { env, requireModule } = Mach;
        if (env) {
            if (env.scheme) this.location = new Location(env.scheme, env.schemeParams);
            this.outerWidth = env.screenWidth;   // Native 提供的屏幕宽度 (dp)
            this.innerWidth = env.screenWidth;
            this.outerHeight = env.screenHeight;  // Native 提供的屏幕高度 (dp)
            this.innerHeight = env.screenHeight;
            this.devicePixelRatio = env.scale;    // Native 提供的像素比
        }
        this.__router = requireModule('WMRouter');  // 获取 Native 路由模块
    }
```

**JS/Native 交互点**:
- `Mach.env`: Native 注入的环境信息对象（屏幕尺寸、scale、scheme 等）
- `Mach.requireModule('WMRouter')`: 获取 Native 路由模块实例
- `this.__router.navigateTo(url)`: 通过 Native 路由跳转（第127行）
- `this.__router.navigateBack()`: 通过 Native 路由返回（第135行）

**注意**: `innerWidth` === `outerWidth`，`innerHeight` === `outerHeight`，因为 Native 不存在浏览器工具栏/滚动条的概念。

### 2.4 全局挂载逻辑 (第152-174行)

```typescript
const window = new Window();  // 创建单例

// 遍历 Window 实例原型链上的所有 key（包括继承的方法）
function getClassKeys(obj) {
    const keys = new Set(Object.keys(obj));
    let c = obj.constructor;
    while (true) {
        Object.getOwnPropertyNames(c.prototype).forEach(key => keys.add(key));
        if (c.__proto__ === Object.__proto__) break;  // 到达原型链顶端
        c = c.__proto__;
    }
    keys.delete('constructor');
    return Array.from(keys);
}

// 第170行: 将 window 的所有属性"摊平"到 globalThis
getClassKeys(window).forEach(k => (globalThis[k] = window[k]));
globalThis.Window = Window;
globalThis.window = globalThis;  // 第174行: window === globalThis（与浏览器一致）
```

**关键设计**: `getClassKeys` 不仅收集实例自身属性，还沿原型链向上遍历收集所有方法（如 `addEventListener` 来自 `EventTarget.prototype`）。这确保了 `globalThis.addEventListener` 等浏览器全局 API 可用。

**第174行的精妙之处**: `globalThis.window = globalThis` 而非 `= window`。这意味着 `window.setTimeout` 访问的是 `globalThis.setTimeout`（已被安全包装的版本），而非 Window 实例上的原始版本。这与浏览器行为一致——`window` 就是全局对象本身。

---

## 3. Document 模拟 — `src/document.ts`

### 3.1 TextNode 内部类 (第6-33行)

```typescript
class TextNode {
    parentNode = null;
    tagName = 'textNode';   // 用 tagName 而非 nodeType 来标识文本节点
    textContent = '';

    constructor(value) {
        this.textContent = value;
    }

    toString() {
        return this.textContent;  // 第15行: 支持 String(textNode) 转换
    }

    get data() {
        return this.textContent;
    }

    set data(v) {
        this.textContent = v;
        this.parentNode?.computedTextContent();  // 第24行: 数据变更时通知父节点重算文本
    }

    unlinkParent() {
        if (this.parentNode) {
            this.parentNode.unlinkParent.call(this);  // 第30行: 借用 Element 的 unlinkParent
        }
    }
}
```

**关键设计**: `TextNode` 不继承 Native `Node`——因为 Native 渲染层不认识文本节点。文本内容最终通过父 `<text>` 元素的 `setAttribute('content', str)` 同步到 Native。

**第24行的联动**: 当文本内容通过 `textNode.data = 'xxx'` 修改时，自动触发父元素的 `computedTextContent()`，将所有子文本节点拼接后通过 `setAttribute` 同步到 Native。

### 3.2 Document 类 (第35-74行)

```typescript
export class Document extends EventTarget {
    body;
    constructor() {
        super();
        this.body = new Element('body');  // 第39行: 创建根节点
    }

    createElement(name) {
        return new Element(name);  // 第43行: 创建元素 → 实际创建的是继承 Native Node 的 Element
    }

    createElementNS(namespace, name) {
        return new Element(name);  // 第47行: 命名空间版本，忽略 namespace（Native 无此概念）
    }

    createTextNode(data) {
        return new TextNode(data);  // 第51行: 创建纯 JS 对象，不涉及 Native
    }

    createEvent(type) {
        if (type == 'CustomEvent') {
            return new CustomEvent();  // 第56行: 仅支持 CustomEvent
        } else {
            return null;
        }
    }

    getElementById(id) {
        return iterateElementWithId(id, this.body);  // 第64行: DFS 遍历 JS 侧 DOM 树
    }
}
```

**JS/Native 边界**:
- `createElement` → `new Element(tagName)` → `super(tagName)` 调用 Native `Node` 构造函数，此时 Native 侧已经创建了对应的渲染节点
- `createTextNode` → 纯 JS 对象，无 Native 对应物
- `getElementById` → 纯 JS 侧遍历，不调用 Native

---

## 4. Element 类——JS 与 Native 的桥梁（核心）— `src/node/element.ts`

这是整个包最核心的文件，`Element` 继承 Native 注入的 `Node` 类，在 JS 层维护 DOM 树的同时通过 `super.xxx()` 将操作同步到 Native 渲染树。

### 4.1 LIST 白名单与事件格式分发 (第8-22行, 第187-200行)

```typescript
// 第8-22行: 这些 Native 标签的事件回调会被包装成标准化 Event 对象
const LIST = [
    'view', 'text', 'image', 'image-background', 'rich-text',
    'modal', 'content', 'water-fall', 'input', 'keyboard-avoid',
    'textarea', 'keyboard-accessory', 'canvas',
];

// 第187-200行: dispatchEvent 中的分发逻辑
dispatchEvent(event, data) {
    const ev = new Event(event, {
        target: this,
        detail: data,
    });
    super.dispatchEvent(ev);  // 通知 Native 侧
    const handler = this.listeners[event];
    if (handler) {
        // 关键分支: LIST 中的标签 → handler(Event对象)
        //          其他标签 → handler(原始data)
        return LIST.includes(this.tagName) ? handler(ev) : handler(data);
    }
}
```

**设计意图**: Native 原生标签（view/text/image 等）的事件回调应接收标准化的 Event 对象（包含 `target`、`detail` 等字段），而自定义组件/非标准标签则直接透传原始数据。这是"渐进式标准化"的策略。

### 4.2 super 调用——JS/Native 桥接核心 (第45-55行)

```typescript
superInsertBefore(node, anchor) {
    // 第46行: TextNode 不发送到 Native（Native 不认识文本节点）
    if (node.tagName !== 'textNode') super.insertBefore(node, anchor);
}

superAppendChild(node) {
    if (node.tagName !== 'textNode') super.appendChild(node);
}

superRemoveChild(node) {
    if (node.tagName !== 'textNode') super.removeChild(node);
}
```

**核心理解**: 所有以 `super.` 开头的调用（`super.insertBefore`、`super.appendChild`、`super.removeChild`、`super.setAttribute`、`super.addEventListener` 等）都是跨越 JS/Native 边界的调用，由 Native Node 基类实现，负责操作 Native 渲染树。

**TextNode 过滤**: 文本节点只存在于 JS 侧 DOM 树中，不会被传递到 Native。文本内容通过 `<text>` 元素的 `content` 属性传递。

### 4.3 appendChild / insertBefore 完整实现 (第57-87行)

```typescript
validateTextNode(node) {
    // 第59行: 如果在非 <text> 标签下插入文本节点，视为非法
    if (node.tagName === 'textNode' && this.tagName !== 'text') {
        console.log(`禁止在非<text>标签中使用文本作为子节点，当前标签<${this.tagName}>`);
        // 第62行: 替换为空 <text> 元素，保持子节点顺序不被打乱
        node = new Element('text');
    }
    return node;
}

appendChild(node) {
    node = this.validateTextNode(node);  // 步骤1: 验证/替换文本节点
    node.unlinkParent();                 // 步骤2: 从旧父节点中移除
    this.superAppendChild(node);         // 步骤3: 同步到 Native → super.appendChild
    node.parentNode = this;              // 步骤4: 更新 JS 侧父引用
    this.childNodes.push(node);          // 步骤5: 更新 JS 侧子节点列表
    this.computedTextContent();          // 步骤6: 如果是 <text>，重算内容
}

insertBefore(node, anchor) {
    node = this.validateTextNode(node);
    if (!anchor) return this.appendChild(node);        // 无锚点 → 退化为 appendChild
    const index = this.childNodes.indexOf(anchor);
    if (index < 0) return this.appendChild(node);      // 锚点不存在 → 退化为 appendChild
    node.unlinkParent();
    this.superInsertBefore(node, anchor);               // 同步到 Native
    node.parentNode = this;
    this.childNodes.splice(index, 0, node);             // 在 JS 侧正确位置插入
    this.computedTextContent();
}
```

**两棵树同步模式**:
1. JS 侧: `this.childNodes`（数组）维护子节点顺序，`node.parentNode` 维护父引用
2. Native 侧: 通过 `super.appendChild` / `super.insertBefore` 同步
3. 两侧始终保持镜像

### 4.4 computedTextContent — 文本聚合 (第89-96行)

```typescript
computedTextContent() {
    if (this.tagName !== 'text') return;  // 只对 <text> 标签生效
    let str = '';
    this.childNodes.forEach(v => {
        str += String(v);  // 调用 TextNode.toString() 或 Element.toString()
    });
    this.setAttribute('content', str);  // 将拼接结果作为属性同步到 Native
}
```

**设计思路**: Native 的 `<text>` 组件通过 `content` 属性接收文本内容，而非通过子文本节点。所以 JS 层将所有子文本节点（可能有多个，如 `<text>Hello {'world'}</text>` 在 React 中会产生两个子节点）拼接后通过 `setAttribute('content', ...)` 传递给 Native。

### 4.5 节点销毁 — $destory (第101-114行)

```typescript
$destory() {
    this.dispatchEvent('$destory');     // 触发销毁事件
    this.parentNode = null;
    if (this.childNodes.length > 0) {
        this.childNodes.forEach(n => n.$destory && n.$destory());  // 递归销毁子节点
        this.childNodes = [];
    }
    this.tagName = null;
    this.parentNode = null;
    this.childNodes = [];
    this.attributes = {};
    this.listeners = {};               // 清空所有引用，帮助 GC
}
```

### 4.6 removeChild (第116-128行)

```typescript
removeChild(node) {
    const index = this.childNodes.indexOf(node);
    if (index >= 0) {
        this.superRemoveChild(node);   // 同步到 Native
        // 第122行: 默认会调用 $destory 彻底销毁节点
        // 可通过 window.__disabled_unlink_element__ 开关禁用
        if (!window.__disabled_unlink_element__) {
            node.$destory && node.$destory();
        }
        this.childNodes.splice(index, 1);
        this.computedTextContent();
    }
}
```

**`__disabled_unlink_element__` 开关**: 某些场景需要 `removeChild` 后再 `appendChild`（节点复用），此时不应销毁节点。这是一个运行时逃生舱。

### 4.7 setAttribute 的 data-* 映射 (第151-163行)

```typescript
setAttribute(attribute, value) {
    // 第152行: 拦截 data-* 属性，转换为 dataset 对象
    if (attribute.indexOf('data-') === 0) {
        const datasetName = dataSetFormat(attribute.substr(5));  // data-my-key → myKey
        if (this.dataset[datasetName] !== value) {
            this.dataset[datasetName] = value;
        }
    }

    if (this.attributes[attribute] === value) return;  // 第160行: 值未变则跳过（性能优化）
    super.setAttribute(attribute, value);               // 第161行: 同步到 Native
    this.attributes[attribute] = value;                 // 更新 JS 侧缓存
}
```

**性能优化**: 第160行的"值未变则跳过"避免了不必要的跨 JS/Native 边界调用。每次跨边界调用都有序列化开销，这个优化非常重要。

### 4.8 addEventListener / removeEventListener — 单 handler 设计 (第174-185行)

```typescript
addEventListener(event, handler, options) {
    if (event && handler) {
        if (this.listeners[event] === handler) return;  // 同一 handler 不重复注册
        super.addEventListener(event);                   // 通知 Native 订阅该事件类型
        this.listeners[event] = handler;                 // 直接赋值，覆盖旧 handler
    }
}

removeEventListener(event, handler, options) {
    super.removeEventListener(event);                    // 通知 Native 取消订阅
    delete this.listeners[event];
}
```

**重大设计决策 — "每事件类型只保留最后一个 handler"**:

`this.listeners[event] = handler` 是**赋值而非 push**。这意味着对同一个元素的同一事件类型，后注册的 handler 会覆盖先注册的。这与 W3C 标准（支持多 handler）不同。

**原因分析**:
1. 性能: Native 侧每个事件类型只需维护一个回调入口
2. `super.addEventListener(event)` 只传了事件名，没传 handler——Native 侧只关心"这个节点是否监听了某事件"，具体回调逻辑在 JS 侧处理
3. React 的合成事件系统本身就是在根节点统一监听，每个元素通常只有一个事件处理函数

### 4.9 getBoundingClientRect — 同步测量 (第221-237行)

```typescript
getBoundingClientRect() {
    const { screenWidth, screenHeight } = Mach.env;
    const { x, y, width, height } = this.measureInWindow();  // Native 同步调用
    const bottom = screenHeight - height - y;
    const right = screenWidth - width - x;

    return { x, y, width, height, bottom, right, top: y, left: x };
}
```

**JS/Native 边界**: `this.measureInWindow()` 是 Native Node 基类提供的方法，同步返回元素在屏幕上的绝对位置（x, y, width, height）。注意是**同步调用**，这在 React Native 中是异步的，MachPro 做了优化。

### 4.10 其他 DOM API 兼容

| API | 行号 | 实现方式 |
|-----|------|----------|
| `firstChild` | 239 | `this.childNodes[0]`（JS 侧） |
| `nextSibling` | 243 | 通过 `parentNode.childNodes` 查找（JS 侧） |
| `textContent` getter | 249 | 仅 `<text>` 标签有效，返回 `content` 属性 |
| `textContent` setter | 256 | 仅 `<text>` 标签有效，调用 `setAttribute` |
| `children` | 263 | 过滤掉 textNode 的 childNodes |
| `id` | 267 | 从 `attributes.id` 获取 |
| `lastChild` | 271 | `childNodes[length-1]` |
| `cloneNode` | 202 | 创建新 Element 并复制属性和子节点 |
| `replaceChild` | 146 | `insertBefore` + `removeChild` 组合 |
| `remove` | 130 | 委托给 `parentNode.removeChild` |
| `unlinkParent` | 136 | 仅更新 JS 侧引用，不调 Native |

---

## 5. Style 代理 — `src/node/style.ts`

```typescript
export class Style {
    constructor(public style: CSSStyleDeclaration) {}
    // style 是 Native Node 的 CSSStyleDeclaration 对象

    getPropertyValue(name) {
        if (typeof name !== 'string') return '';
        return this[name] || '';  // 从 JS 侧缓存读取
    }

    setProperty(name, value) {
        const oldValue = this[name];
        value = value !== undefined ? '' + value : undefined;  // 统一转字符串
        this[name] = value;                                     // 更新 JS 侧缓存
        if (oldValue !== value) this.style.setProperty(name, value);  // 仅变化时同步 Native
    }

    removeProperty(name) {
        delete this[name];
        this.style.removeProperty && this.style.removeProperty(name);
    }
}
```

**核心设计 — "JS 侧缓存 + 脏检查"**:

1. 每个样式属性同时存储在 `this[name]`（JS 侧缓存）和 `this.style`（Native CSSStyleDeclaration）
2. `setProperty` 时先比较 JS 缓存中的旧值，只有值发生变化才调用 `this.style.setProperty()` 同步到 Native
3. 这避免了 React 高频更新时对 Native 的无效调用

**`this.style`**: 来自 `super.style`（Element 第41行），是 Native Node 基类暴露的 CSSStyleDeclaration 对象。`style.setProperty(name, value)` 会直接修改 Native 渲染节点的样式。

**被注释掉的 `toCamel`**: 最初设计支持 `background-color` → `backgroundColor` 的转换，后来去掉了，说明样式名的格式约定已在上层（框架编译期）统一。

---

## 6. Mach 全局对象扩展 — `src/mach.ts`

`Mach` 是 Native 注入到 JS 全局作用域的核心对象，本文件为其补充了 JS 层的事件系统和模块系统。

### 6.1 destroyApp (第1-5行)

```typescript
globalThis.destroyApp = function() {
    Mach.__event_handlers__ = {};  // 清空所有事件监听
    if (globalThis.document) globalThis.document.dispatchEvent(new Event('unload'));
    if (globalThis.destroyReactApp) globalThis.destroyReactApp();  // React 清理入口
};
```

**调用场景**: Native 容器关闭页面时调用 `destroyApp()`，清理 JS 侧状态。`destroyReactApp` 由 React 运行时注册。

### 6.2 事件订阅系统 — on/off (第7-41行)

```typescript
Mach.__event_handlers__ = {};  // 事件名 → handler数组

Mach.on = function(event, handler) {
    const handlers = Mach.__event_handlers__[event];
    if (handlers) {
        handlers.push(handler);           // 已有订阅者 → 追加
    } else {
        Mach.__event_handlers__[event] = [handler];
        Mach.subscribeEvent(event);       // 首次订阅 → 通知 Native 开始推送该事件
    }
    return function() {                   // 返回取消订阅函数（类似 Rx 的 unsubscribe）
        Mach.off(event, handler);
    };
};

Mach.off = function(event, handler) {
    const handlers = Mach.__event_handlers__[event];
    if (handlers) {
        const currentHandlers = handlers.filter(eventFun => eventFun !== handler);
        if (currentHandlers.length > 0) {
            Mach.__event_handlers__[event] = currentHandlers;
        } else {
            delete Mach.__event_handlers__[event];
            if (Mach.unsubscribeEvent) {
                Mach.unsubscribeEvent(event);  // 最后一个订阅者移除 → 通知 Native 停止推送
            }
        }
    }
};
```

**JS/Native 联动**:
- `Mach.subscribeEvent(event)`: Native 提供，告知 Native "JS 层开始关心某事件"
- `Mach.unsubscribeEvent(event)`: Native 提供，告知 Native "JS 层不再关心某事件"
- 这是一个**引用计数式**的订阅管理：首个 handler 注册时订阅 Native，最后一个 handler 移除时取消订阅

### 6.3 模块系统 — exportModule / callModule (第43-69行)

```typescript
Mach.registerModule = function(moduleName, fun) {
    // 第44行: 禁止 JS 侧注册 Native 模块（安全考虑）
    console.log(`registerModule【${moduleName}】 Error,Native当中Module为客户端注入，不允许自行注入`)
}

Mach.__js_modules__ = {};  // JS 模块注册表

Mach.exportModule = function(moduleName, methods) {
    if (moduleName && methods) {
        Mach.__js_modules__[moduleName] = methods;  // 注册 JS 模块供 Native 调用
    }
};

Mach.callModule = function(moduleName, methodName, params) {
    // Native 调用 JS 模块的入口
    const module = Mach.__js_modules__[moduleName];
    if (module) {
        const method = module[methodName];
        if (method) {
            return params ? method(...params) : method();
        }
    }
    return undefined;
};
```

**双向通信模型**:
- **JS → Native**: `Mach.requireModule('ModuleName')` 获取 Native 模块，然后调用其方法
- **Native → JS**: Native 调用 `Mach.callModule('ModuleName', 'methodName', params)` 调用 JS 注册的模块

### 6.4 receiveEvent — Native 事件回调入口 (第71-78行)

```typescript
Mach.receiveEvent = function(event, data) {
    const handlers = Mach.__event_handlers__[event];
    if (handlers) {
        for (let i = 0, len = handlers.length; i < len; i++) {
            handlers[i](data);  // 逐个调用注册的 handler
        }
    }
};
```

**调用路径**: Native 事件发生 → Native 调用 `Mach.receiveEvent(eventName, data)` → JS 侧分发给所有 handler。

### 6.5 requireBundleAsync — 异步子包加载 (第80-106行)

```typescript
Mach.requireBundleAsync = (bundleId, options) => {
    // 第81行: Promise 缓存，相同 bundleId 只加载一次
    if (Mach.requireBundleAsync.cache[bundleId]) {
        return Mach.requireBundleAsync.cache[bundleId];
    }
    const promise = new Promise((resolve, reject) => {
        if (!Mach.__requireBundleAsync__) {
            reject('客户端不支持 __requireBundleAsync__ 方法');
            return;
        }
        // 第89行: 调用 Native 加载子包
        Mach.__requireBundleAsync__(bundleId, err => {
            if (err) { reject(err); return; }
            // 第94行: 子包加载完成后通过全局回调获取模块
            if (!globalThis.__bundleCallback__) {
                reject('__bundleCallback__未挂载,请检查子bundle内容');
                return;
            }
            const module = globalThis.__bundleCallback__();
            globalThis.__bundleCallback__ = null;  // 用完清空
            resolve(module);
        }, options);
    });
    Mach.requireBundleAsync.cache[bundleId] = promise;  // 缓存 Promise
    return promise;
};
Mach.requireBundleAsync.cache = {};
```

**子包加载流程**:
1. JS 调用 `Mach.requireBundleAsync(bundleId)`
2. Native `__requireBundleAsync__` 下载并执行子 bundle JS 文件
3. 子 bundle 执行时将自己注册到 `globalThis.__bundleCallback__`
4. 回调触发，JS 层通过 `__bundleCallback__()` 获取子模块导出
5. Promise 被缓存，避免重复加载

### 6.6 styleTransform — 像素单位缩放 (第108-123行)

```typescript
Mach.styleTransform = (num: number, useNumber: boolean = false) => {
    try {
        const { defaultUnit, transform } = MACH_PROJECT_CONFIG.pxTransform;
        let unit = defaultUnit;
        if (transform && transform[unit]) {
            num = (transform[unit].ratio || 1) * num;   // 按比例缩放数值
            unit = transform[unit].unit || unit;         // 可能转换单位
        }
        if (useNumber) return num;    // 返回纯数字
        return num + unit;            // 返回 "数字+单位" 字符串
    } catch (error) {
        return num;                   // 降级: 直接返回原数字
    }
};
```

**用途**: 编译时将 CSS 中的 `px` 值转换为设备适配值。`MACH_PROJECT_CONFIG` 是构建时注入的项目配置常量。

---

## 7. 元素垫片 (elements-shim)

### 7.1 jsx.ts — Monkey-patch React 创建元素 (第1-32行)

```typescript
export const ElementMap = new Map<string, any>();  // 标签名 → React 组件 的映射表

const oldCreateElement = React.createElement;
React.createElement = function createElement(type, ...args) {
    if (ElementMap.has(type)) {
        type = ElementMap.get(type);  // 将标签名替换为 React 组件
    }
    return oldCreateElement(type, ...args);
};

const oldJsx = JSX.jsx;
JSX.jsx = JSX.jsxs = function jsx(type, ...args) {
    if (ElementMap.has(type)) {
        type = ElementMap.get(type);
    }
    return oldJsx(type, ...args);
};

React.h2 = oldCreateElement;  // 保留原始 createElement 的引用
```

**设计目的**: 拦截 React 的元素创建过程。当 React 试图创建 `<scroll-view>`、`<image>` 等 Native 标签时，自动替换为对应的 React 垫片组件。这是"标签名 → 组件"的透明映射层。

**为什么同时 patch `createElement` 和 `jsx`**: React 17+ 的新 JSX Transform 使用 `jsx()` 而非 `createElement()`，需要两者都 patch。

### 7.2 index.ts — 注册垫片映射 (第1-12行)

```typescript
ElementMap.set('scroll-view', ScrollView);
ElementMap.set('image', Image);
ElementMap.set('image-background', ImageBackground);
```

### 7.3 ScrollView.tsx — H5/Native 双轨适配 (第1-24行)

```typescript
const ScrollView = forwardRef(({ children, ...rest }, ref) => {
    if (process.env.MACH_ENV === "h5") {
        // H5 环境: 直接使用 <scroll-view> 标签（由 H5 渲染层处理）
        return <scroll-view ref={ref} {...rest}>{children}</scroll-view>;
    }
    // Native 环境: 转换为 <scroller> + <content> 结构
    const contentStyle = rest.style || {};
    if (rest.scrollX) contentStyle.flexDirection = "row";   // 横向滚动
    if (rest.scrollY) contentStyle.flexDirection = "column"; // 纵向滚动
    return (
        <scroller ref={ref} {...rest} style={contentStyle}>
            <content>{children}</content>
        </scroller>
    );
});
```

**跨端差异**:
- H5: `<scroll-view>` 是 Web Components 或自定义标签
- Native: 需要 `<scroller>` 容器 + `<content>` 包裹子内容，并通过 `flexDirection` 控制滚动方向

### 7.4 Image.tsx / ImageBackground.tsx

```typescript
// Image.tsx
const Image = forwardRef(({ ...rest }, ref) => {
    if (!rest.resizeMode) rest.resizeMode = rest.mode;  // 兼容两种缩放模式属性名
    if (!rest.mode) rest.mode = rest.resizeMode;
    return <image ref={ref} {...rest} />;
});

// ImageBackground.tsx — 同理，标签改为 <image-background>
```

**适配层**: 统一 `resizeMode`（RN 风格）和 `mode`（小程序风格）两种属性命名。

---

## 8. 事件系统 — `src/event/`

### 8.1 EventTarget (event-target.ts) — 纯 JS 侧的通用事件基类

```typescript
export class EventTarget {
    listeners = {} as listeners;  // { [eventType]: [{ isOnce, cb }] }

    addEventListener(type, cb, options) {
        if (!cb || !type) return;
        const handlers = this.listeners[type];
        if (handlers) {
            handlers.push({ isOnce: options && options.once, cb });
        } else {
            this.listeners[type] = [{ isOnce: options && options.once, cb }];
        }
    }

    removeEventListener(type, cb) {
        const handlers = this.listeners[type];
        if (handlers) {
            const index = handlers.findIndex(v => cb === v.cb);
            if (index !== -1) handlers.splice(index, 1);
        }
    }

    dispatchEvent(event) {
        const type = event && event.type;
        if (!type) return;
        const handles = this.listeners[type];
        if (handles && handles.length > 0) {
            handles.forEach(v => {
                const { isOnce, cb } = v;
                cb && cb.call(this, event);
                if (isOnce) this.removeEventListener(type, cb);
            });
        }
    }
}
```

**注意**: 这个 `EventTarget` 是给 `Window` 和 `Document` 使用的纯 JS 事件系统，支持多 handler 和 `once` 选项。而 `Element` 虽然也继承了 `EventTarget`（通过 Window 暴露），但 `Element` 的 `addEventListener` 被完全覆盖为"单 handler"模式（见 4.8 节）。

**两套事件系统对比**:

| 特性 | EventTarget（Window/Document 用） | Element 覆盖版 |
|------|----------------------------------|----------------|
| 多 handler | 支持 | 不支持（覆盖） |
| once 选项 | 支持 | 不支持 |
| 回调传参 | Event 对象 | LIST 标签传 Event，其他传原始 data |
| Native 联动 | 无 | `super.addEventListener(event)` |

### 8.2 Event (event.ts)

```typescript
export class Event {
    readonly bubbles: boolean = false;           // 不支持冒泡
    readonly cancelable: boolean = true;
    readonly target: EventTarget | null = null;
    readonly currentTarget: EventTarget | null = null;
    readonly timeStamp: DOMHighResTimeStamp = Date.now();
    readonly type: string = '';
    detail = null;                               // 非标准属性，存储事件数据

    constructor(typeArg?, customEventInit = {}) {
        this.type = typeArg;
        this.bubbles = customEventInit.bubbles || false;
        this.cancelable = customEventInit.cancelable || false;
        this.detail = customEventInit.detail;
        this.target = customEventInit.target;
        this.currentTarget = customEventInit.currentTarget || customEventInit.target;
    }

    preventDefault() { return unsupport("preventDefault"); }
    stopImmediatePropagation() { return unsupport("stopImmediatePropagation"); }
    initEvent() { return unsupport("initEvent"); }

    toJSON() { /* 序列化 */ }
}
```

**关键设计决策**:
- `bubbles` 固定为 `false`: **不支持事件冒泡**。Native 布局引擎自有事件分发机制，JS 层不需要也无法实现冒泡
- `preventDefault` / `stopImmediatePropagation` 直接抛错: 这些依赖浏览器底层能力，Native 环境无法模拟
- `detail` 属性: 参考微信小程序的事件模型，用于携带自定义事件数据

### 8.3 CustomEvent (custom-event.ts)

```typescript
export class CustomEvent extends Event {
    constructor(typeArg?, customEventInit?) {
        super(typeArg, { ...customEventInit });
    }
}
```

纯透传，仅为 API 兼容。

---

## 9. 工具函数 — `src/utils.ts`

```typescript
// 深度优先搜索查找指定 id 的元素
export function iterateElementWithId(id, elm) {
    if (elm.id === id) return elm;
    const children = elm.children || [];
    for (let element of children) {
        const res = iterateElementWithId(id, element);
        if (res) return res;
    }
    return null;
}

// data-my-attr → myAttr 的驼峰转换（不使用正则，因为 Native JS 引擎可能不支持）
export function dataSetFormat(dataName) {
    const lowNames = Array.from(dataName.toLowerCase());
    const nameChars = [];
    lowNames.forEach((v, index) => {
        if ((v > 'a' && v < 'z') && lowNames[index - 1] === '-') {
            return nameChars[nameChars.length - 1] = v.toUpperCase();
        }
        return nameChars.push(v);
    });
    return nameChars.join('');
}

export function unsupport(name) {
    throw new Error(`暂不支持API ${name}`);
}
```

**`dataSetFormat` 的实现特点**: 手动逐字符处理而非正则替换。注释明确说明原因——"不支持正则"。这是 Native JS 引擎（如 JavaScriptCore 的精简版）的限制之一。

---

## 10. 整体架构总结

### 10.1 JS/Native 交互边界一览

| JS 侧调用 | 跨越边界到 Native | 说明 |
|-----------|-------------------|------|
| `new Element(tagName)` → `super(tagName)` | `Node` 构造函数 | 创建 Native 渲染节点 |
| `element.appendChild(node)` → `super.appendChild(node)` | `Node.appendChild` | Native 树添加子节点 |
| `element.insertBefore(node, anchor)` → `super.insertBefore(node, anchor)` | `Node.insertBefore` | Native 树插入子节点 |
| `element.removeChild(node)` → `super.removeChild(node)` | `Node.removeChild` | Native 树移除子节点 |
| `element.setAttribute(k, v)` → `super.setAttribute(k, v)` | `Node.setAttribute` | 设置 Native 节点属性 |
| `element.removeAttribute(k)` → `super.removeAttribute(k)` | `Node.removeAttribute` | 移除 Native 节点属性 |
| `element.addEventListener(event)` → `super.addEventListener(event)` | `Node.addEventListener` | 告知 Native 订阅事件 |
| `element.removeEventListener(event)` → `super.removeEventListener(event)` | `Node.removeEventListener` | 告知 Native 取消订阅 |
| `element.dispatchEvent(ev)` → `super.dispatchEvent(ev)` | `Node.dispatchEvent` | 向 Native 分发事件 |
| `element.getBoundingClientRect()` → `this.measureInWindow()` | `Node.measureInWindow` | 同步获取布局信息 |
| `element.style.setProperty(k, v)` → `nativeStyle.setProperty(k, v)` | `CSSStyleDeclaration.setProperty` | 设置 Native 样式 |
| `Mach.requireModule(name)` | 获取 Native 模块 | 如 WMRouter |
| `Mach.subscribeEvent(event)` | 订阅 Native 全局事件 | 如生命周期事件 |
| `Mach.unsubscribeEvent(event)` | 取消订阅 | — |
| `Mach.__requireBundleAsync__(bundleId, cb)` | 加载 Native 子包 | 异步 bundle 加载 |

### 10.2 设计模式总结

1. **双树镜像**: JS 侧 (`childNodes` 数组) 和 Native 侧 (Native Node 树) 保持同步。JS 侧用于 DOM 查询（`getElementById`、`firstChild`、`nextSibling`），Native 侧用于实际渲染。

2. **缓存 + 脏检查**: `setAttribute` 和 `Style.setProperty` 都在 JS 侧缓存值，仅在值变化时才跨边界调用 Native，减少序列化开销。

3. **TextNode 折叠**: 文本节点不进入 Native 树，而是通过 `computedTextContent()` 聚合为父 `<text>` 元素的 `content` 属性。

4. **标签垫片**: 通过 Monkey-patch `React.createElement` / `JSX.jsx`，在不修改业务代码的前提下将标准标签名映射为平台适配组件。

5. **单 handler 事件模型**: Element 层面每个事件类型只保留一个 handler，简化 Native 交互，契合 React 单一事件绑定的使用模式。

6. **引用计数订阅**: `Mach.on/off` 仅在首个/最后一个 handler 注册/移除时通知 Native，避免频繁的订阅/取消操作。

7. **无冒泡事件**: 事件不冒泡，`preventDefault`/`stopPropagation` 不可用——事件分发完全依赖 Native 渲染引擎自身的机制。

### 10.3 引擎适配痕迹

代码中多处体现了 Native JS 引擎（非标准 V8/JSC）的限制：
- 不使用正则表达式（`dataSetFormat`、`Location` 解析）
- `setTimeout`/`setInterval` 需要安全包装默认参数
- `console.error` 需要手动 fallback 到 `console.log`
- 注释中多次提到"目前没有支持正则"

### 10.4 文件依赖关系图

```
index.ts
  └── window.ts
        ├── event/event-target.ts  ← Window extends EventTarget
        │     └── types.ts
        ├── event/event.ts
        │     └── utils.ts (unsupport)
        ├── event/custom-event.ts
        │     └── event/event.ts
        ├── node/element.ts  ← Element extends Native Node
        │     ├── utils.ts (dataSetFormat)
        │     ├── event/event.ts
        │     └── node/style.ts
        ├── document.ts
        │     ├── event/event-target.ts
        │     ├── node/element.ts
        │     ├── event/custom-event.ts
        │     └── utils.ts (iterateElementWithId)
        └── utils.ts (unsupport)

mach.ts  ← 独立模块，扩展全局 Mach 对象

elements-shim/
  ├── jsx.ts  ← Monkey-patch React.createElement
  ├── index.ts  ← 注册标签映射
  ├── ScrollView.tsx
  ├── Image.tsx
  └── ImageBackground.tsx
```
