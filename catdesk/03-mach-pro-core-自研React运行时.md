# 第三章：mach-pro-core —— 自研 React 运行时

> 一句话概括：完全自研的精简版 React 运行时，实现了 Component/Hooks/DOM 操作/列表 Diff 四大核心能力，用 62 行的 nextTick 替代了 React 的 Fiber Scheduler。

## 3.1 整体架构

mach-pro-core 共 29 个源文件（~1800 行代码），分为四层：

```
组件系统层 (Component/FunctionComponent/PureComponent + nextTick)
     ↓
Hooks 系统 (context/state/effect/memo/ref)
     ↓
元素处理管线 (processElementDeep + fragment/block/component)
     ↓
DOM 操作层 (insert/listen/attr/style/temp/child + array diff)
```

## 3.2 组件系统

### 3.2.1 nextTick —— 微任务批量更新调度器

**文件**：`src/react/nextTick.ts`（62 行）

```typescript
const callbacks = []          // 待执行的回调函数数组
let pending = false;          // 是否已排入微任务队列
const p = Promise.resolve()   // 预创建的已 resolve Promise
const timerFunc = () => p.then(flushCallbacks);

function flushCallbacks() {
    pending = false
    const copies = callbacks.slice(0)  // 复制一份避免执行中新回调干扰
    callbacks.length = 0               // 清空原队列
    for (let i = 0; i < copies.length; i++) {
        copies[i]()
    }
}

export function nextTick(cb?: Function, ctx?: Object) {
    callbacks.push(() => {
        if (cb) { ctx ? cb.call(ctx) : cb(); }
    })
    if (!pending) {
        pending = true
        timerFunc();
    }
}
```

**与 React/Vue 的对比**：

| 维度 | React 18 | Vue 3 | MachPro |
|------|---------|-------|---------|
| 调度机制 | Lane Model + Scheduler | nextTick (Promise.then) | nextTick (Promise.then) |
| 优先级 | 多优先级（连续/离散/同步） | 无优先级 | 无优先级 |
| 批量范围 | Automatic Batching（所有场景） | 依赖响应式系统 | 所有 setState 自动合并 |
| 可中断 | Fiber 可中断/可恢复 | 不可中断 | 不可中断 |

### 3.2.2 Component 基类

**文件**：`src/react/Class/Component.ts`（286 行）

#### setState 批量更新机制

```typescript
setState(state: S, callback?: () => void) {
    // 将新 state 合并到 nextState（不直接修改 this.state）
    this.nextState = Object.assign({}, this.nextState || this.state, state);
    this.$flush(callback);
}

$flush(callback?: () => void) {
    if (callback) this.updateCallbacks.push(callback);
    if (!this.$mounted) return;        // mounting 阶段不走 nextTick
    if (!this.flushing) {
        this.flushing = true;
        nextTick(() => {
            if (!this.flushing) return; // 防重入
            this.$run(true);
        });
    }
}
```

**批量更新时序图**：

```
t0: setState({a:1})  → nextState={a:1}, flushing=true, nextTick 入队
t1: setState({b:2})  → nextState={a:1,b:2}, flushing=true, 不再入队
t2: setState({c:3})  → nextState={a:1,b:2,c:3}, flushing=true, 不再入队
--- 微任务执行 ---
t3: $run(true)       → 一次性处理合并后的 {a:1,b:2,c:3}
```

#### $run —— 核心更新生命周期

```typescript
$run(async = true) {
    if (this.$unmount) return false;           // 已卸载不更新
    
    const prevState = { ...this.state };
    const prevProps = { ...this.props };
    let nextState = { ...(this.nextState || this.state) };
    let nextProps = { ...(this.nextProps || this.props) };
    
    // props 更新时触发 componentWillReceiveProps
    if (!async) {
        this.flushing = true;
        if (this.componentWillReceiveProps) 
            this.componentWillReceiveProps(nextProps, nextState, null);
        nextState = { ...(this.nextState || this.state) };  // cWRP 中可能调了 setState
    }
    
    // shouldComponentUpdate 检查
    if (!this.needForceUpdate) {
        if (this.shouldComponentUpdate && !this.shouldComponentUpdate(nextProps, nextState, null)) 
            return false;
    }
    
    // 应用新 state/props
    Object.assign(this.state, nextState);
    this.props = nextProps;
    
    // 注册 componentDidUpdate 回调（在 DOM 更新后触发）
    this.$DOMDidUpdateEvents.once(() => {
        if (this.componentDidUpdate) this.componentDidUpdate(prevProps, prevState);
    });
    
    this.$update(async);
    return true;
}
```

**关键设计**：`componentDidUpdate` 不在 `$run` 中同步调用，而是注册为 `$DOMDidUpdateEvents.once()` 回调，在 `insert` 函数完成 DOM 更新后通过 `processElementDeep` 收集的 components 列表统一触发。

#### $updatePropsSync —— 同步 Props 更新

```typescript
$updatePropsSync(props: any) {
    this.nextProps = Object.assign({}, props);
    return this.$run(false);  // async=false，同步更新
}
```

Props 更新是同步的（不走 nextTick），因为 props 变更来自父组件渲染，已经在一个更新批次中。

### 3.2.3 FunctionComponent 包装器

**文件**：`src/react/Class/FunctionComponent.ts`（53 行）

```typescript
export class FunctionComponent<P> extends Component<P, null> {
    public __hooks__ = {
        ref: { current: null },
        setters: {},        // useState setter 缓存
        dispatchers: {},    // useReducer dispatch 缓存
        effects: {},        // useEffect 集合
        layoutEffects: {},  // useLayoutEffect 集合
        refs: {},           // useRef 缓存
        memos: {}           // useMemo/useCallback 缓存
    }
    
    constructor(props, renderProps) {
        super(props);
        this.__hooks__.ref.current = this;
        this.render = bindComponent(this.__hooks__.ref, (props, _ref) => {
            const res = this.renderProps(props);       // 执行用户函数
            runEffects(this.__hooks__.effects);         // 执行 effects
            return res;
        });
    }
    
    componentWillUpdate()  { runEffects(this.__hooks__.layoutEffects) }
    componentDidUpdate()   { runEffects(this.__hooks__.effects) }
    componentWillUnmount() { cleanupEffects(this.__hooks__.effects) }
}
```

**核心设计**：函数组件被包装为类组件，`__hooks__` 对象以 counter 为 key 存储所有 hooks 状态。`bindComponent` 在 hooks context 栈中执行 render，使 hooks 能正确关联到当前组件实例。

## 3.3 Hooks 系统

### 3.3.1 Hooks 上下文管理

**文件**：`src/react/hooks/context.ts`（23 行）

```typescript
const hooksContextStack = []

export function useCounter() {
    const context = hooksContextStack[hooksContextStack.length - 1]
    const counter = context.counter++
    return { component: context.component, counter }
}

export function withContext(component, func) {
    return function (...args) {
        hooksContextStack.push({ component, counter: 0 })
        const result = func(...args)
        hooksContextStack.pop()
        return result
    }
}
```

**与 React 的差异**：React 用 Fiber 节点上的 `memoizedState` 链表存储 hooks。MachPro 用全局栈 + counter 索引，更简洁但同样依赖调用顺序一致性。

### 3.3.2 useState / useReducer

**文件**：`src/react/hooks/state.ts`（62 行）

```typescript
export function useState(defaultState) {
    const { component, counter } = useCounter()
    
    if (!component.state.hasOwnProperty(counter)) {
        // 首次：初始化 state 和 setter
        component.state[counter] = typeof defaultState === 'function' 
            ? defaultState() : defaultState;
        
        component.__hooks__.setters[counter] = (state, callback?) => {
            if (typeof state === 'function') {
                component.setState({
                    [counter]: state(component.state[counter])
                }, callback)
            } else {
                component.setState({[counter]: state}, callback)
            }
        }
    }
    
    return [component.state[counter], component.__hooks__.setters[counter]]
}
```

**关键**：useState 的状态存储在 `component.state[counter]` 中，setter 缓存在 `__hooks__.setters[counter]` 中（稳定引用，不会每次 render 重建）。

### 3.3.3 useEffect / useLayoutEffect

**文件**：`src/react/hooks/effect.ts`（58 行）

```typescript
export function runEffects(effects) {
    Object.getOwnPropertyNames(effects).forEach((_counter) => {
        const [effectFunc, cleanup] = effects[_counter]
        if (typeof effectFunc === 'function') {
            if (typeof cleanup === 'function') cleanup()   // 先 cleanup
            const nextCleanup = effectFunc()               // 再执行
            effects[_counter][0] = undefined               // 标记已执行
            effects[_counter][1] = nextCleanup || undefined
        }
    })
}
```

**⚠️ 与 React 差异**：MachPro 对每个 effect 交替执行 cleanup 和 effect（先 cleanup 再执行），而 React 是先 cleanup 所有再执行所有。此外，MachPro 的 useEffect 在 render 后立即执行，而 React 的 useEffect 在 paint 后异步执行。

## 3.4 DOM 操作层

### 3.4.1 insert —— 最核心的函数

**文件**：`src/dom/dom.ts`（L81-L218，~140 行）

insert 是整个框架最关键的函数，负责将值（DOM 节点/数组/组件/文本）插入到父节点的指定位置，实现增量 DOM 更新。

**marker + cache 机制**：

```
parent.$cache = {
    'a1': { originalValue: ..., array: [dom1, dom2], lookup: Map },
    'a2': { originalValue: ..., array: [dom3], lookup: Map },
}
```

每个 `insert(id, parent, value, marker)` 调用对应一个"插入槽位"。`parent.$cache[id]` 存储该槽位的上次状态。更新时：

1. **值比较短路**：`value === cache.originalValue` 直接返回
2. **有缓存数组**：通过 `updateArray` 做 diff，最小化 DOM 操作
3. **无缓存**：直接 `appendNodes`

**marker 定位**：当某个槽位从 null 变为有内容时，需要找到正确的插入位置。MachPro 通过查找 `id` 更大的下一个有内容的 cache 作为 marker，并通过 `beforeId` 属性向前修正。

### 3.4.2 listen —— 事件委托优化

```typescript
export function listen(id, node, name, handler) {
    node.events = node.events || new Map()
    const isRegister = node.events.has(id)
    node.events.set(id, handler)          // 直接替换 handler
    if (isRegister) return                // 已注册则不重复 addEventListener
    node.addEventListener(name, function (...args) {
        if (node.events && node.events.get(id)) 
            return node.events.get(id).apply(this, args)
    })
}
```

💡 **亮点**：每个节点+事件名只注册一次 `addEventListener`，后续 handler 更新只替换 Map 中的值，避免了重复注册的开销。

### 3.4.3 列表 Diff 算法

**文件**：`src/dom/array.ts`（85 行）

MachPro 使用**从后向前的双指针 + delta 启发式**算法：

1. **引用比较优先**：`newValue === oldValue` 直接跳过
2. **delta 启发式**：计算每个元素在新旧数组中的位置差距，delta 大的优先移动
3. **willMove/didMove 双标记**：避免重复操作

**与 React Reconciliation 对比**：

| 维度 | React | MachPro |
|------|-------|---------|
| 遍历方向 | 从前向后 | 从后向前 |
| 移动策略 | key 匹配 + 重排 | delta 启发式 |
| 复杂度 | O(n) | O(n) |
| 可中断 | 支持（Fiber） | 不支持 |

## 3.5 元素处理管线

### processElementDeep —— 渲染管线核心调度器

**文件**：`src/react/process/element.ts`（198 行）

```typescript
export function processElementDeep({ id, target, element, index, rerender, components }) {
    while (isElementContext(currentElement)) {
        const context = getContext({ target, id, index });
        let { ctx, uniqId } = context.info(element);
        
        if (ctx) {
            skip = ctx.update(element) === false;   // 已缓存：update
        } else {
            ctx = contextCreatorMap[type](element);  // 未缓存：创建
            ctx.mount({ uniqId, target, ... });      // 挂载
            context.set(element, ctx);               // 存入缓存
        }
        
        if (skip) return target.$elementCache[processKey];  // SCU=false：短路
        
        currentElement = ctx.element();     // 展开到下一层
        if (res.component) components.unshift(res.component);  // 收集组件实例
    }
}
```

**三层缓存实现组件复用**：

1. `target.$${type}Cache`（Map）：ElementContext 级缓存，按 uniqId 索引
2. `target.$elementCache`：DOM 结果级缓存，SCU=false 时直接返回
3. `target.$destoryCache`：销毁回调缓存

## 本章小结

mach-pro-core 用约 1800 行代码实现了精简版 React 运行时。核心亮点包括：62 行的 nextTick 批量更新调度器（借鉴 Vue）、编译产物驱动的 insert/attr/style 直接 DOM 操作（借鉴 SolidJS）、marker+cache 的增量 DOM 更新机制、delta 启发式的列表 diff 算法。整体设计在性能和代码量上取得了很好的平衡，特别适合资源受限的客户端 JS 引擎环境。

---

## 面试素材

### 高频面试题

**基础题**：MachPro 的 setState 批量更新是如何实现的？

**深度题**：请对比 MachPro 的 insert 函数和 React 的 reconciliation，分别是如何管理 DOM 增量更新的？

### 参考回答

> MachPro 的 setState 批量更新借鉴了 Vue 的 nextTick 思路。每个组件维护一个 `flushing` 标志，首次 setState 时标记 flushing=true 并通过 `Promise.resolve().then()` 入微任务队列。同一事件循环内后续的 setState 只合并 state（`Object.assign`），不重复入队。微任务执行时一次性 flush 所有合并后的 state。与 React 18 的 Automatic Batching 效果类似，但实现更轻量——没有优先级调度，没有可中断/恢复，适合跨端场景的简洁需求。

### 亮点话术

> "在 insert 函数的设计中，我们采用了 marker+cache 机制实现增量 DOM 更新。每个 insert 调用对应一个'插入槽位'，通过 `parent.$cache[id]` 维护该槽位的上次状态。更新时先做值比较短路，再通过自研的 delta 启发式 diff 算法最小化 DOM 操作。这种设计与 React 的 Fiber reconciliation 思路不同——React 是 tree-level 的 diff，我们是 slot-level 的增量更新，更适合编译时已知结构的场景。"
