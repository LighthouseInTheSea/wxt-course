# WXT 浏览器插件开发教程目录

> 本教程系列基于 [WXT 框架](https://wxt.dev/) 开发，涵盖从入门到实战的完整内容。
> 
> 最后更新: 2025年12月

---

## 第一部分: WXT 基础教程

系统学习 WXT 框架的核心概念和 API，每章配套实战练习。

### 核心概念

| 章节 | 主题 | 实战练习 |
|------|------|----------|
| [01-WXT简介与环境搭建](./WXT-基础教程/01-WXT简介与环境搭建.md) | 框架介绍、环境配置、项目创建 | [环境搭建实战](./WXT-实战练习/01-环境搭建实战.md) |
| [02-项目结构详解](./WXT-基础教程/02-项目结构详解.md) | 目录结构、路径别名、输出模板 | [项目结构实战](./WXT-实战练习/02-项目结构实战.md) |
| [03-入口点系统](./WXT-基础教程/03-入口点系统.md) | 入口点类型、命名规范、配置选项 | [入口点实战](./WXT-实战练习/03-入口点实战.md) |

### 核心功能

| 章节 | 主题 | 实战练习 |
|------|------|----------|
| [04-后台脚本开发](./WXT-基础教程/04-后台脚本开发.md) | Service Worker、生命周期、事件处理 | [后台脚本实战](./WXT-实战练习/04-后台脚本实战.md) |
| [05-内容脚本开发](./WXT-基础教程/05-内容脚本开发.md) | DOM操作、CSS注入、Shadow DOM | [内容脚本实战](./WXT-实战练习/05-内容脚本实战.md) |
| [06-消息通信机制](./WXT-基础教程/06-消息通信机制.md) | 消息类型、双向通信、长连接 | [消息通信实战](./WXT-实战练习/06-消息通信实战.md) |
| [07-Storage存储系统](./WXT-基础教程/07-Storage存储系统.md) | 存储区域、类型安全、数据监听 | [Storage存储实战](./WXT-实战练习/07-Storage存储实战.md) |
| [08-React集成开发](./WXT-基础教程/08-React集成开发.md) | React配置、UI注入、状态管理 | [React集成实战](./WXT-实战练习/08-React集成实战.md) |
| [09-构建与发布](./WXT-基础教程/09-构建与发布.md) | 多浏览器构建、商店发布、自动化 | [构建发布实战](./WXT-实战练习/09-构建发布实战.md) |

### 进阶功能

| 章节 | 主题 | 实战练习 |
|------|------|----------|
| [10-自动导入系统](./WXT-基础教程/10-自动导入系统.md) | Auto-imports、ESLint集成 | [自动导入实战](./WXT-实战练习/10-自动导入实战.md) |
| [11-国际化i18n](./WXT-基础教程/11-国际化i18n.md) | 多语言支持、消息格式、动态切换 | [国际化实战](./WXT-实战练习/11-国际化实战.md) |
| [12-远程代码与脚本注入](./WXT-基础教程/12-远程代码与脚本注入.md) | Remote Code、MAIN World注入 | [远程代码实战](./WXT-实战练习/12-远程代码实战.md) |
| [13-WXT模块与ESM](./WXT-基础教程/13-WXT模块与ESM.md) | 模块系统、自定义模块、Hook | [WXT模块实战](./WXT-实战练习/13-WXT模块实战.md) |
| [14-WebAccessibleResources与WASM](./WXT-基础教程/14-WebAccessibleResources与WASM.md) | WAR配置、WASM集成 | [WAR与WASM实战](./WXT-实战练习/14-WAR与WASM实战.md) |

---

## 第二部分: 沉浸式翻译插件实战

完整的浏览器翻译插件项目，涵盖前端插件开发和后端服务搭建。

### 插件开发

| 章节 | 主题 | 核心技术 |
|------|------|----------|
| [01-项目初始化与架构设计](./WXT-沉浸式翻译实战/01-项目初始化与架构设计.md) | 项目结构、类型定义、架构设计 | WXT + React + TypeScript |
| [02-后台服务开发](./WXT-沉浸式翻译实战/02-后台服务开发.md) | 消息处理、翻译API、右键菜单 | 策略模式、处理器模式 |
| [03-内容脚本开发](./WXT-沉浸式翻译实战/03-内容脚本开发.md) | DOM遍历、文本提取、翻译注入 | Shadow DOM、MutationObserver |
| [04-Popup与Options页面](./WXT-沉浸式翻译实战/04-Popup与Options页面.md) | React组件、设置管理、UI设计 | Radix UI、Tailwind CSS |
| [05-高级功能与优化](./WXT-沉浸式翻译实战/05-高级功能与优化.md) | 缓存优化、性能调优、错误处理 | IndexedDB、Web Worker |
| [06-项目总结与发布](./WXT-沉浸式翻译实战/06-项目总结与发布.md) | 测试、发布、维护 | GitHub Actions、商店发布 |
| [07-国际化与自动导入集成](./WXT-沉浸式翻译实战/07-国际化与自动导入集成.md) | i18n集成、自动导入配置 | @wxt-dev/i18n |

### 后端服务

| 章节 | 主题 | 核心技术 |
|------|------|----------|
| [00-项目配置与运行指南](./WXT-沉浸式翻译实战/后端服务/00-项目配置与运行指南.md) | 环境配置、部署指南、配置清单 | 完整运行指南 |
| [01-后端服务架构](./WXT-沉浸式翻译实战/后端服务/01-后端服务架构.md) | API设计、数据库设计、项目结构 | Hono + Drizzle + PostgreSQL |
| [02-认证系统实现](./WXT-沉浸式翻译实战/后端服务/02-认证系统实现.md) | OAuth登录、Session管理 | better-auth + Google/GitHub/Email |
| [03-可扩展支付系统](./WXT-沉浸式翻译实战/后端服务/03-可扩展支付系统.md) | 支付集成、Webhook处理 | 策略模式 + Creem/Stripe |
| [04-落地页与定价页面](./WXT-沉浸式翻译实战/后端服务/04-落地页与定价页面.md) | 营销页面、定价组件 | React + Tailwind CSS |
| [05-会员系统与AI翻译服务](./WXT-沉浸式翻译实战/后端服务/05-会员系统与AI翻译服务.md) | 权限控制、用量统计、AI集成 | GPT-4 / DeepL |

---

## 学习路径推荐

### 初学者路径 (2-3周)

```
基础教程 01-03 -> 基础教程 04-06 -> 基础教程 07-09 -> 沉浸式翻译 01-04
```

1. **第1周**: 学习 WXT 基础概念 (01-03)
2. **第2周**: 掌握核心功能 (04-06)
3. **第3周**: 学习集成和构建 (07-09)，开始实战项目

### 进阶路径 (1-2周)

```
基础教程 10-14 -> 沉浸式翻译 后端服务
```

1. 学习进阶功能 (10-14)
2. 完成后端服务开发
3. 实现完整的 SaaS 产品

### 快速上手路径 (3-5天)

```
基础教程 01 -> 基础教程 04-05 -> 沉浸式翻译 01-02
```

适合有 Chrome 插件开发经验的开发者快速迁移到 WXT。

---

## 技术栈总览

### 前端插件

| 技术 | 用途 | 文档 |
|------|------|------|
| WXT | 插件框架 | [wxt.dev](https://wxt.dev/) |
| React 18 | UI框架 | [react.dev](https://react.dev/) |
| TypeScript | 类型系统 | [typescriptlang.org](https://www.typescriptlang.org/) |
| Tailwind CSS | 样式框架 | [tailwindcss.com](https://tailwindcss.com/) |
| Zustand | 状态管理 | [zustand](https://github.com/pmndrs/zustand) |

### 后端服务

| 技术 | 用途 | 文档 |
|------|------|------|
| Hono | Web框架 | [hono.dev](https://hono.dev/) |
| Drizzle ORM | 数据库ORM | [orm.drizzle.team](https://orm.drizzle.team/) |
| PostgreSQL | 主数据库 | [postgresql.org](https://www.postgresql.org/) |
| better-auth | 认证系统 | [better-auth.com](https://www.better-auth.com/) |
| Creem | 支付集成 | [creem.io](https://www.creem.io/) |

---

## 相关资源

### 官方文档
- [WXT 官方文档](https://wxt.dev/)
- [Chrome Extensions 开发文档](https://developer.chrome.com/docs/extensions/)
- [Firefox Add-ons 开发文档](https://extensionworkshop.com/)

### 示例项目
- [WXT 官方示例](https://github.com/wxt-dev/examples)
- [沉浸式翻译](https://github.com/nicepkg/immersive-translate) (灵感来源)

### 工具
- [Chrome 扩展商店](https://chrome.google.com/webstore/)
- [Firefox 附加组件](https://addons.mozilla.org/)

---

## 更新日志

- **2025-12-29**: 
  - 补充教程文字说明和原理解释
  - 添加章节间导航链接
  - 新增后端服务配置运行指南
  - 优化代码注释
