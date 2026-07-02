# 第五章：Preact Fork —— H5/小程序端引擎

> 一句话概括：深度定制的 Preact fork，包含 10 处核心改动，覆盖事件系统重写、三平台样式适配、小程序生命周期时序修正和非法 VNode 容错。

## 5.1 为什么 Fork Preact

以下改动无法通过配置或插件实现，必须修改 Preact 源码：

1. **事件监听机制**需要改为闭包引用（避免小程序/Native DOM 模拟层中 `this` 指向问题）
2. **style diff**需要按平台做 gradient 属性映射
3. **createElement**需要传入第三个参数标识 MachPro 创建的元素
4. **commit 回调**需要在小程序环境中用 `wx.nextTick` 延迟
5. **className→class**需要保证属性顺序（影响小程序/Native 样式渲染）

## 5.2 十处核心改动详解

### 改动 1：H5 环境 Object 属性序列化

**文件**：`src/diff/props.js` L112-L114

```javascript
// 新增：H5 环境下 Object 类型属性值自动 JSON.stringify
if (process.env.MACH_ENV === 'h5' && 
    Object.prototype.toString.call(value) === '[object Object]') {
    value = JSON.stringify(value);
}
```

原因：H5 DOM 不支持直接 `setAttribute` Object 值，会显示 `[object Object]`。

### 改动 2：事件监听机制完全重写

**文件**：`src/diff/props.js` L50-L93

原版 Preact 使用 `eventProxy` 函数代理（依赖 `this._listeners`），MachPro 改为每个事件创建独立的 `{value, handler}` 对象，通过闭包捕获 `dom` 引用：

```javascript
listener = dom._listeners[name + useCapture] = {
    value,
    handler: e => {
        return dom._listeners[name + useCapture]?.value(e);
    },
};
dom.addEventListener(name, listener.handler, useCapture);
```

原因：小程序/Native DOM 模拟层中 `this` 指向可能不正确，闭包方案更安全。

### 改动 3：三平台 gradient 适配

**文件**：`src/diff/style.js` L27-L37

```javascript
if (process.env.MACH_ENV === 'react') {
    // Native：backgroundImage gradient → backgroundColor
    if (name == 'backgroundImage' && value.includes('gradient')) 
        name = 'backgroundColor';
}
```

原因：Native 渲染引擎不支持 `background-image` 的 gradient，但支持 `background-color` 接收渐变值。

### 改动 4：pxTransform 单位转换体系

**文件**：`compat/src/style.js`（整个文件新增，54 行）

劫持 `CSSStyleDeclaration.prototype.setProperty`，在所有 style 设置前自动做单位转换。支持 px→rpx/dp/vw 等多种转换规则。

### 改动 5：微信小程序 wx.nextTick 延迟回调

**文件**：`compat/src/index.js` L167-L177

```javascript
if (process.env.MACH_ENV === 'wechat') {
    options._commit = (root, commitQueue) => {
        commitQueue.forEach(c => {
            c._renderCallbacks = c._renderCallbacks.map(cb => function () {
                wx.nextTick(() => cb.call(this))
            })
        })
    }
}
```

原因：小程序的 setData 是异步的，生命周期回调需要延迟到 DOM 真正渲染后才执行。

### 改动 6：createElement 第三个参数标识

**文件**：`src/diff/index.js` L360-L365

```javascript
dom = document.createElement(nodeType, newProps.is && newProps, true);
// 第三个参数 true 标识是 MachPro H5 创建的元素
```

### 改动 7：全局 destroyReactApp

**文件**：`compat/src/index.js` L161-L165

```javascript
globalThis.destroyReactApp = function () {
    unmountComponentAtNode(document.body);
};
```

供 Native/小程序容器在页面退出时调用，确保 React 组件树正确卸载。

### 改动 8：className→class 顺序保证

**文件**：`compat/src/render.js` L109-L223

```javascript
normalizedProps = Object.assign({ class: props.className }, props);
// Object.assign 确保 class 在对象第一位
```

原因：小程序/Native DOM 模拟层中 setAttribute 的调用顺序影响样