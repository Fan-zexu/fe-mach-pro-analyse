# Mach Pro 指标定义整理

> 来源：[学城文档 845282581](https://km.sankuai.com/collabpage/845282581)
> 仅提取 MachPro 相关部分（含 Mach 1.0 指标作为参考对比）

---

## 一、MachPro 指标体系

### 1.1 全景图

![MachPro 指标体系全景图](/Users/zhangnan53/machpro/machpro指标.png)

### 1.2 指标分类

#### 质量视角（10 个指标）

| 名称 | 描述 | 基线 |
|------|------|------|
| Crash率 | SDK 引发的 Crash 率 | 0.1‱ |
| checkupdate 接口请求成功率 | 模板更新接口请求成功率 | 99% |
| MPBundleDownloadSuccess | Bundle 下载成功率 | 99% |
| MPBundleDownloadTime | Bundle 下载耗时 | - |
| MPBundleLoadSuccess | Bundle 加载成功率 | 99.5% |
| MPPageLoadSuccess | 页面加载成功率，最终 Bundle 加载是否成功 | 99.5% |
| MPPageExitSuccess | 页面退出成功率。退出页面时满足：1. 首次布局完成；2. 页面展示期间没有发生 JS 异常 | - |
| MPFSTime | 业务首屏时间，即业务接口请求结束、页面渲染完成的时间 | - |
| MPFSRate | 秒开率（MPFSTime < 1秒的比例） | - |
| MPJSException | JS 异常数量 | - |

#### 性能视角（6 个指标）

| 名称 | 描述 | 基线 |
|------|------|------|
| MPPageLoadTime | 页面加载耗时（包含创建引擎、执行字节码和首次布局耗时） | - |
| MPBundleLoadTime | 加载 Bundle 时间 | - |
| MPCreateEngineTime | 创建 JS 引擎时间 | - |
| MPExecuteJSTime | 执行字节码时间 | - |
| MPPageFCPTime | 首次内容绘制（First Content Paint），即从创建 JS 引擎到首次布局结束 | - |
| MPScrollFps | 滚动 FPS | - |

---

## 二、Mach 1.0 指标体系（参考对比）

### 2.1 全景图

![Mach 指标体系全景图](https://km.sankuai.com/api/file/cdn/845282581/862725599?contentType=0&isNewContent=false)

### 2.2 指标分类

#### 质量视角

| 名称 | 释义 | 基线 | 备注 |
|------|------|------|------|
| Crash率 | Mach 基建引发的 Crash 率 | 0.1‱ | Android：Crash 平台 Mach 库 Crash 数目 / DAU；iOS：Crash 平台 WMMach 库 Crash 数目 / DAU |
| 模板更新接口请求成功率 | - | 99% | 美团实际：98.707%（Android: 98.936%, iOS: 98.541%） |
| 模板下载成功率 | - | 99% | - |
| 模板加载成功率 | - | 99.9% | - |
| 渲染成功率 | - | 99.9% | 同移动端模板渲染成功率（包含在业务监控中） |
| 业务成功率 | - | 99.9% | 美团：99.97%，外卖：99.98% |
| 表达式异常率 | - | 0.1‱ | - |
| JS 异常率 | - | 0.1‱ | - |

#### 性能视角

| 名称 | 释义 | 基线 | 备注 |
|------|------|------|------|
| 表达式解析耗时 | - | TP90: 40ms, TP50: 15ms | - |
| 首次渲染耗时 | - | TP90: 1000ms, TP50: 200ms | - |

#### 业务视角（广告业务全链路监控）

| 名称 | 释义 | 基线 |
|------|------|------|
| 移动端模板接收率 | 客户端接收到广告模板数量 / API 侧广告模板投放量 | 99% |
| 移动端模板匹配成功率 | 模板加载成功的数量 / 接收到广告模板数量 | 95% |
| 移动端模板渲染成功率 | 模板渲染成功的数量 / 模板加载成功的数量 | 99.99% |
| 移动端模板首次曝光率 | 模板首次曝光的数量 / 模板渲染成功的数量 | - |

### 2.3 指标查看方式

- **数据可视化 2.0（Raptor）**：优化了大盘可视化，简化查询过程；增加了业务 Bundle 维度的质量和性能，便于各业务方实现开发运维闭环管理
- **数据日报**：邮件报表和系统

---

## 三、指标命名规律总结

### 3.1 MachPro 指标前缀规律

MachPro 指标以 `MP` 为前缀，后面跟具体含义的驼峰命名：

- `MPBundle*` — Bundle 相关（下载、加载）
- `MPPage*` — 页面相关（加载、退出）
- `MPFS*` — 首屏相关（时间、秒开率）
- `MPJS*` — JS 相关（异常）
- `MPCreate*` — 创建相关（引擎）
- `MPExecute*` — 执行相关（字节码）

### 3.2 关键基线速记

| 指标类型 | 基线要求 |
|----------|----------|
| Crash率 | 0.1‱（万分之一） |
| 接口成功率 | 99% |
| Bundle 下载成功率 | 99% |
| Bundle/页面加载成功率 | 99.5%（MachPro）/ 99.9%（Mach） |
| 渲染成功率 | 99.9% |
| 业务成功率 | 99.9% |
| 表达式/JS 异常率 | 0.1‱ |
| 表达式解析耗时 | TP90: 40ms, TP50: 15ms |
| 首次渲染耗时 | TP90: 1000ms, TP50: 200ms |

### 3.3 MachPro vs Mach 指标差异

MachPro 相比 Mach 1.0 的指标变化：

1. **新增**：页面加载耗时拆解指标（MPCreateEngineTime、MPExecuteJSTime、MPPageFCPTime、MPPageLoadTime、MPBundleLoadTime）
2. **新增**：滚动 FPS（MPScrollFps）
3. **新增**：秒开率（MPFSRate）
4. **新增**：页面退出成功率（MPPageExitSuccess）
5. **简化**：去掉了表达式解析耗时、表达式异常率等 Mach 1.0 特有的指标（因为 MachPro 不再使用表达式引擎，而是编译时转换）
6. **简化**：去掉了业务视角的广告全链路指标（这些属于业务监控范畴）

---

## 四、面试速记

### Q：MachPro 有哪些核心质量指标？
Crash率（0.1‱）、checkupdate 接口成功率（99%）、Bundle 下载成功率（99%）、Bundle 加载成功率（99.5%）、页面加载成功率（99.5%）、页面退出成功率、业务首屏时间（MPFSTime）、秒开率（MPFSRate）、JS 异常数量。

### Q：MachPro 有哪些核心性能指标？
页面加载耗时（MPPageLoadTime，含创建引擎 + 执行字节码 + 首次布局）、Bundle 加载时间（MPBundleLoadTime）、创建 JS 引擎时间（MPCreateEngineTime）、执行字节码时间（MPExecuteJSTime）、首次内容绘制（MPPageFCPTime）、滚动 FPS（MPScrollFps）。

### Q：MachPro 相比 Mach 1.0 在指标上的主要变化？
MachPro 新增了页面加载全流程的拆解指标（引擎创建、字节码执行、FCP）和滚动 FPS，去掉了表达式相关指标（因为编译时方案不再需要运行时表达式解析），整体更关注端到端的页面加载性能和用户体验。

---

> **备注**：本文档基于学城文档 [845282581](https://km.sankuai.com/collabpage/845282581) 整理，仅供个人学习和面试准备使用。全景图需要内网认证才能访问，请点击链接查看。
