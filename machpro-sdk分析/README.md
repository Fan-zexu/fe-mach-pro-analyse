# MachPro 全页面动态化框架 · 深度源码分析

> 基于 `waimai_fe_machpro` 代码库 v3.6.2-next.61 的完整源码级分析
> 兼具「系统化学习笔记」和「技术面试素材」双重用途

## 项目定位

MachPro 是美团外卖核心流量区的**全页面动态化解决方案**。它不是传统的 WebView 容器，而是一套自研的跨端渲染框架——开发者用 React/JSX 语法编写代码，通过编译时转换 + 多套运行时引擎，同时输出到 **Native（iOS/Android/HarmonyOS）、微信小程序、H5** 三端运行。

## 文档目录

```
catdesk/
├── README.md                          ← 你正在看的文件
├── 01-全局架构与设计哲学.md              ← 架构全景、双引擎决策、模块依赖
├── 02-编译时优化系统.md                  ← babel-plugin-mach-pro、Fragment/Block 编译策略
├── 03-mach-pro-core-自研React运行时.md   ← 组件系统、Hooks、DOM 操作、列表 Diff
├── 04-mach-pro-render-Native-DOM模拟层.md ← Window/Document/Element 模拟、JS-Native 桥梁
├── 05-Preact-Fork-H5小程序引擎.md        ← 10 处核心改动逐一分析
├── 06-构建系统-mach-pro-react-build.md   ← 三端构建管线、代码注入机制
├── 07-跨端适配体系.md                    ← Bridge Proxy 劫持、标签适配、小程序适配
├── 08-状态管理-mach-pro-store.md         ← 自研类 Rematch 方案
├── 09-性能优化与监控体系.md               ← 编译时/加载/运行时优化、Worker、性能埋点
├── 10-热更新与辅助模块.md                ← 热更新、H5 元素适配、Native Module 模拟
└── 11-技术亮点总结与面试素材.md           ← 8 大技术亮点、面试题库、对比分析表
```

## 阅读建议

**快速入门（30 分钟）**：先读第 1 章了解全局架构，再读第 11 章掌握技术亮点和面试话术。

**深入学习（3-5 小时）**：按章节顺序阅读第 1-6 章，重点关注第 3 章（自研 React 运行时）和第 4 章（Native DOM 模拟层），这两章包含最核心的源码解读。

**面试准备**：每章末尾都附有「面试素材」模块，包含高频面试题、参考回答和深度追问预判。第 11 章是汇总版。

## 核心技术栈

| 维度 | 技术选型 |
|------|---------|
| 语言 | TypeScript + JavaScript |
| Monorepo | Lerna 3 + Yarn |
| 构建工具 | Rollup (Native/小程序) + Vite (H5) |
| 渲染方案 | 自研 React 运行时 + fork Preact（双引擎） |
| DOM 模拟 | 自研 W3C DOM API 模拟层 (Native 端) |
| 客户端通信 | Mach 全局对象 (requireModule) + KNB JSBridge |
| 状态管理 | 自研类 Rematch 方案 |
| 样式方案 | CSS Module + PostCSS + px/dp/rpx/vw 自动转换 |
