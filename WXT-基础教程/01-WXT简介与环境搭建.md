# WXT 简介与环境搭建

> **导航**: [目录](../README.md) | [下一章: 项目结构详解 >>](./02-项目结构详解.md)

---

## 1. WXT 是什么

### 1.1 浏览器扩展开发的痛点

在传统的浏览器扩展开发中,开发者需要面对诸多挑战:

**配置繁琐**: 需要手动编写 manifest.json,不同浏览器(Chrome/Firefox/Edge)的 manifest 格式存在差异,Chrome 使用 Manifest V3 而 Firefox 仍需支持 V2。

**构建复杂**: 如果想使用 TypeScript、React/Vue 等现代技术栈,需要自行配置 webpack 或 rollup,处理各种 loader 和 plugin。

**开发体验差**: 每次修改代码都需要手动刷新扩展,没有热重载支持,调试效率低下。

**跨浏览器适配**: Chrome 使用 `chrome.` API 命名空间,Firefox 使用 `browser.`,需要手动处理兼容性。

### 1.2 WXT 如何解决这些问题

WXT (Web Extension Tools) 是一个现代化的浏览器扩展开发框架,它借鉴了 Next.js 的设计理念,将复杂的配置封装起来,让开发者专注于业务逻辑。

**核心特性一览**:

| 特性 | 说明 | 带来的好处 |
|------|------|-----------|
| **Vite 驱动** | 基于 Vite 构建系统 | 极速的冷启动和热更新 |
| **自动 Manifest** | 根据文件结构自动生成 | 无需手写 manifest.json |
| **跨浏览器** | 一套代码多端运行 | 大幅减少适配工作 |
| **MV2/MV3 兼容** | 自动处理版本差异 | Firefox/Chrome 无缝切换 |
| **TypeScript 优先** | 开箱即用的类型支持 | 更好的开发体验和代码质量 |
| **热模块替换** | 修改代码自动重载 | 开发效率提升 10 倍 |

### 1.3 WXT 与传统开发方式对比

让我们通过一个具体例子来理解 WXT 的优势。假设我们要创建一个简单的弹出页面:

**传统方式** - 需要以下步骤:
1. 创建 manifest.json 并配置 action/browser_action
2. 配置 webpack 打包 React 代码
3. 设置 HTML 模板和入口文件
4. 手动处理资源路径

**WXT 方式** - 只需:
1. 在 `entrypoints/popup/` 目录下创建文件
2. WXT 自动识别并生成对应的 manifest 配置

这种"约定优于配置"的理念大大简化了开发流程。

---

## 2. 环境准备

### 2.1 系统要求

在开始之前,请确保你的开发环境满足以下条件:

| 要求 | 最低版本 | 推荐版本 | 说明 |
|------|---------|---------|------|
| Node.js | 18.0.0 | 20.x LTS | WXT 使用了较新的 Node.js 特性 |
| 包管理器 | - | pnpm | 推荐使用 pnpm,速度更快 |
| 操作系统 | - | Windows/macOS/Linux | 全平台支持 |

**检查 Node.js 版本**:

```bash
node --version
# 应该输出 v18.0.0 或更高版本
```

如果版本过低,请访问 [nodejs.org](https://nodejs.org) 下载最新的 LTS 版本。

### 2.2 为什么推荐 pnpm

pnpm 相比 npm 和 yarn 有以下优势:

1. **更快的安装速度**: pnpm 使用硬链接和符号链接,避免重复下载相同的包
2. **更少的磁盘占用**: 全局存储 + 链接机制,一个包只需下载一次
3. **更严格的依赖管理**: 不会提升(hoist)依赖,避免幽灵依赖问题

**安装 pnpm**:

```bash
# 使用 npm 安装
npm install -g pnpm

# 或使用 corepack (Node.js 16.13+ 内置)
corepack enable
corepack prepare pnpm@latest --activate
```

### 2.3 推荐的开发工具

为了获得最佳开发体验,建议安装以下工具:

**VS Code 插件**:
- **ESLint**: 代码检查
- **Prettier**: 代码格式化
- **TypeScript Vue Plugin (Volar)**: 如果使用 Vue
- **ES7+ React/Redux/React-Native snippets**: 如果使用 React

**浏览器开发工具**:
- Chrome DevTools: F12 打开,用于调试扩展
- React Developer Tools: 调试 React 组件树
- Vue.js devtools: 调试 Vue 组件

---

## 3. 创建第一个 WXT 项目

### 3.1 使用脚手架创建项目

WXT 提供了交互式的脚手架工具,只需一行命令即可创建项目:

```bash
# 推荐: 使用 pnpm
pnpm dlx wxt@latest init my-extension

# 或使用 npm
npx wxt@latest init my-extension

# 或使用 bun
bunx wxt@latest init my-extension
```

**命令解析**:
- `pnpm dlx`: 类似 npx,用于执行远程包的命令
- `wxt@latest`: 使用最新版本的 wxt 包
- `init`: 初始化命令
- `my-extension`: 项目名称(可自定义)

运行命令后,脚手架会提出几个问题:

```
? Choose a template: (选择模板)
  vanilla    # 纯 JavaScript,无框架
  vue        # Vue 3
> react      # React 18
  svelte     # Svelte
  solid      # Solid.js

? Choose a package manager: (选择包管理器)
> pnpm
  npm
  yarn
  bun
```

**选择建议**:
- 如果你熟悉 React,选择 `react` 模板
- 如果你熟悉 Vue,选择 `vue` 模板
- 如果只是简单扩展,`vanilla` 就够用了

### 3.2 项目初始化过程详解

选择完成后,脚手架会执行以下操作:

```
Creating project in ./my-extension...
  - Creating package.json           # 创建包配置文件
  - Creating wxt.config.ts          # 创建 WXT 配置
  - Creating tsconfig.json          # 创建 TypeScript 配置
  - Creating entrypoints/           # 创建入口点目录
  - Installing dependencies...      # 安装依赖
```

### 3.3 手动创建项目(进阶)

如果你想在现有项目中集成 WXT,或者想更细粒度地控制项目配置,可以手动创建:

```bash
# 1. 创建并进入项目目录
mkdir my-extension && cd my-extension

# 2. 初始化 package.json
pnpm init

# 3. 安装 WXT 作为开发依赖
pnpm add -D wxt

# 4. 如果使用 React,还需要安装相关依赖
pnpm add react react-dom
pnpm add -D @types/react @types/react-dom @wxt-dev/module-react
```

**为什么 WXT 是开发依赖(devDependencies)?**

WXT 是构建工具,只在开发和打包时使用,最终的扩展包中不包含 WXT 代码。类似于 webpack、vite 等构建工具,它们都应该放在 devDependencies 中。

---

## 4. 配置 package.json

### 4.1 完整的 scripts 配置

在 package.json 中添加以下脚本:

```json
{
  "name": "my-extension",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "wxt",
    "dev:firefox": "wxt -b firefox",
    "build": "wxt build",
    "build:firefox": "wxt build -b firefox",
    "zip": "wxt zip",
    "zip:firefox": "wxt zip -b firefox",
    "compile": "tsc --noEmit",
    "postinstall": "wxt prepare"
  }
}
```

**各脚本详解**:

| 脚本 | 命令 | 用途 |
|------|------|------|
| `dev` | `wxt` | 启动开发服务器,默认目标是 Chrome |
| `dev:firefox` | `wxt -b firefox` | 针对 Firefox 启动开发服务器 |
| `build` | `wxt build` | 构建生产版本(Chrome) |
| `build:firefox` | `wxt build -b firefox` | 构建 Firefox 版本 |
| `zip` | `wxt zip` | 打包为 ZIP 文件,用于上传到商店 |
| `zip:firefox` | `wxt zip -b firefox` | 打包 Firefox 版本 |
| `compile` | `tsc --noEmit` | 类型检查,不输出文件 |
| `postinstall` | `wxt prepare` | 安装依赖后自动执行 |

### 4.2 postinstall 脚本的重要性

`postinstall` 是 npm/pnpm 的生命周期钩子,在每次安装依赖后自动执行。

`wxt prepare` 命令会:
1. 创建 `.wxt/` 目录
2. 生成 TypeScript 类型定义文件
3. 配置自动导入的类型提示
4. 设置 tsconfig.json 的 paths 别名

**如果遇到类型错误,首先尝试**:
```bash
pnpm run postinstall
# 或直接
pnpm wxt prepare
```

---

## 5. 项目配置文件详解

### 5.1 wxt.config.ts - WXT 核心配置

这是 WXT 最重要的配置文件,控制着整个构建过程:

```typescript
// wxt.config.ts
import { defineConfig } from 'wxt';

export default defineConfig({
  // ========== 源码配置 ==========
  // 源码目录,默认是项目根目录
  // 设置为 'src' 后,入口点目录变为 src/entrypoints/
  srcDir: 'src',
  
  // 入口点目录名,相对于 srcDir
  entrypointsDir: 'entrypoints',
  
  // 输出目录
  outDir: '.output',

  // ========== Manifest 配置 ==========
  manifest: {
    // 扩展名称 - 显示在浏览器扩展管理页面
    name: 'My Extension',
    
    // 版本号 - 建议遵循语义化版本 (semver)
    version: '1.0.0',
    
    // 描述 - 显示在扩展商店
    description: 'A browser extension built with WXT',
    
    // 权限声明 - 根据实际需求添加
    permissions: [
      'storage',      // 存储 API
      'activeTab',    // 当前标签页操作
    ],
    
    // 主机权限 - 允许访问的网站
    host_permissions: [
      'https://*.example.com/*',  // 特定网站
    ],
  },
  
  // ========== 开发服务器配置 ==========
  dev: {
    server: {
      port: 3000,  // 开发服务器端口
    },
  },
  
  // ========== 模块配置 ==========
  modules: [
    '@wxt-dev/module-react',  // React 支持模块
  ],
});
```

**配置说明**:

**srcDir 的作用**: 
将源码放在 `src/` 目录下可以让项目结构更清晰,配置文件(如 wxt.config.ts、tsconfig.json)保留在根目录,业务代码全部在 src 中。

**权限的最小化原则**: 
只声明扩展实际需要的权限。过多的权限会让用户产生不信任感,也可能导致商店审核被拒。

### 5.2 tsconfig.json - TypeScript 配置

WXT 会在 `.wxt/` 目录生成基础的 TypeScript 配置,你的 tsconfig.json 应该继承它:

```json
{
  "extends": "./.wxt/tsconfig.json",
  "compilerOptions": {
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

**为什么要继承 .wxt/tsconfig.json?**

WXT 生成的配置包含:
- 正确的模块解析设置
- 路径别名 (`@/` 指向 srcDir, `~/` 也指向 srcDir)
- WXT 相关类型的引用
- 适合浏览器扩展的编译目标

---

## 6. 启动开发服务器

### 6.1 运行开发模式

```bash
pnpm dev
```

执行后,WXT 会:
1. **编译代码**: 将 TypeScript/React 代码编译为浏览器可运行的 JavaScript
2. **启动开发服务器**: 用于热重载通信
3. **打开浏览器**: 自动打开一个新的浏览器窗口
4. **安装扩展**: 自动将编译后的扩展加载到浏览器中

**首次运行可能需要等待几秒钟**,因为需要下载和编译依赖。

### 6.2 开发模式的工作原理

WXT 的热重载机制是这样工作的:

```
[源码修改] -> [Vite 检测变化] -> [重新编译] -> [通知扩展] -> [自动重载]
     |              |               |              |            |
   你的编辑器    文件系统监听     快速 ESBuild    WebSocket    runtime.reload()
```

**不同类型文件的重载行为**:

| 文件类型 | 重载方式 | 说明 |
|---------|---------|------|
| Background | 完全重载 | 后台脚本会重新执行 |
| Content Script | 页面刷新 | 需要刷新目标页面 |
| Popup/Options | 热更新 | 只更新变化的部分 |
| CSS | 热更新 | 样式即时生效 |

### 6.3 指定目标浏览器

WXT 支持多种浏览器:

```bash
# Chrome (默认)
pnpm dev

# Firefox
pnpm dev -b firefox

# Edge
pnpm dev -b edge

# Safari (需要额外配置)
pnpm dev -b safari

# 同时为多个浏览器构建
pnpm build -b chrome,firefox
```

**浏览器差异处理**:

WXT 会自动处理不同浏览器的差异:
- Chrome 使用 Manifest V3,Firefox 使用 V2
- API 命名空间自动适配 (`chrome.` vs `browser.`)
- 不支持的 API 自动跳过或 polyfill

---

## 7. 目录结构预览

使用 React 模板创建的项目结构:

```
my-extension/
├── .wxt/                    # [自动生成] WXT 配置和类型定义
│   ├── tsconfig.json        # 基础 TypeScript 配置
│   └── types/               # 自动导入的类型定义
│
├── entrypoints/             # [重要] 扩展入口点
│   ├── background.ts        # 后台脚本 (Service Worker)
│   ├── content.ts           # 内容脚本 (注入到网页)
│   └── popup/               # 弹出页面
│       ├── index.html       # HTML 入口
│       ├── main.tsx         # React 入口
│       └── App.tsx          # 根组件
│
├── public/                  # [可选] 静态资源
│   └── icon/                # 扩展图标
│       ├── 16.png           # 16x16 图标
│       ├── 32.png           # 32x32 图标
│       ├── 48.png           # 48x48 图标
│       └── 128.png          # 128x128 图标
│
├── .output/                 # [自动生成] 构建输出
│   ├── chrome-mv3/          # Chrome 构建结果
│   └── firefox-mv2/         # Firefox 构建结果
│
├── wxt.config.ts            # WXT 配置文件
├── package.json             # 项目配置
└── tsconfig.json            # TypeScript 配置
```

**目录说明**:

- **entrypoints/**: 这是最重要的目录,WXT 会根据这里的文件自动生成 manifest 配置
- **.wxt/**: 由 WXT 自动管理,不要手动修改,应加入 .gitignore
- **.output/**: 构建输出目录,应加入 .gitignore
- **public/**: 静态资源会原样复制到输出目录

---

## 8. 验证安装

### 8.1 创建一个简单的后台脚本

在 `entrypoints/background.ts` 中添加:

```typescript
// entrypoints/background.ts

// defineBackground 是 WXT 提供的函数,用于定义后台脚本
// 它会自动处理 Manifest V2/V3 的差异
export default defineBackground(() => {
  // 这段代码在扩展安装后立即执行
  console.log('Hello from WXT!', {
    // browser.runtime.id 是扩展的唯一标识符
    id: browser.runtime.id,
  });
  
  // 监听扩展安装/更新事件
  browser.runtime.onInstalled.addListener((details) => {
    console.log('Extension installed:', details.reason);
  });
});
```

**代码解析**:

1. `defineBackground` 是 WXT 的核心函数,不需要手动导入(自动导入)
2. `browser` 是统一的 API 命名空间,WXT 会自动处理 `chrome` 和 `browser` 的差异
3. 后台脚本在 Manifest V3 中作为 Service Worker 运行

### 8.2 运行并验证

```bash
pnpm dev
```

**查看后台脚本日志**:

1. 打开 Chrome,进入 `chrome://extensions/`
2. 找到你的扩展,点击 "Service Worker" 或 "背景页" 链接
3. 在打开的 DevTools 控制台中应该能看到:
   ```
   Hello from WXT! {id: "xxxxx"}
   Extension installed: install
   ```

---

## 9. 常见问题排查

### 9.1 TypeScript 类型错误

**问题**: `Cannot find name 'defineBackground'` 或 `Cannot find name 'browser'`

**原因**: WXT 的类型定义未正确生成

**解决方案**:
```bash
# 重新生成类型定义
pnpm wxt prepare

# 如果还不行,尝试删除 .wxt 目录后重新生成
rm -rf .wxt && pnpm wxt prepare
```

### 9.2 开发服务器启动失败

**问题**: 端口被占用或无法连接

**解决方案**:
```typescript
// wxt.config.ts
export default defineConfig({
  dev: {
    server: {
      port: 3001,  // 换一个端口
    },
  },
});
```

### 9.3 浏览器无法自动打开

**问题**: 开发服务器启动了,但浏览器没有打开

**解决方案**:

手动加载扩展:
1. 打开 `chrome://extensions/`
2. 开启右上角的 "开发者模式"
3. 点击 "加载已解压的扩展程序"
4. 选择项目中的 `.output/chrome-mv3` 目录

### 9.4 热重载不工作

**问题**: 修改代码后扩展没有自动更新

**可能原因**:
1. 后台脚本需要完全重载(这是正常的)
2. 内容脚本需要刷新目标页面
3. 开发服务器连接断开

**解决方案**:
- 检查终端是否有错误信息
- 尝试手动刷新扩展(在扩展管理页面点击刷新按钮)
- 重启开发服务器

---

## 10. 本章小结

本章我们完成了:

| 内容 | 状态 |
|------|------|
| 理解 WXT 框架的优势 | ✅ |
| 配置开发环境 | ✅ |
| 创建第一个 WXT 项目 | ✅ |
| 理解 package.json 配置 | ✅ |
| 理解 wxt.config.ts 配置 | ✅ |
| 启动开发服务器 | ✅ |
| 验证安装是否成功 | ✅ |

**关键概念回顾**:
- WXT 是基于 Vite 的浏览器扩展开发框架
- `defineBackground` 用于定义后台脚本
- `browser` API 是跨浏览器兼容的统一接口
- `.wxt/` 目录由 WXT 自动管理

---

> **导航**: [目录](../README.md) | [下一章: 项目结构详解 >>](./02-项目结构详解.md)
