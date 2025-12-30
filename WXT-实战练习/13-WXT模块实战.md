# WXT 实战练习 13: WXT 模块与 ES Modules

## 练习目标

通过本练习,你将:
- 理解 WXT 模块系统
- 使用官方和第三方模块
- 创建自定义 WXT 模块
- 掌握 ES Modules 在扩展中的应用

## 练习 13.1: 使用 WXT 官方模块

### 任务

安装和配置 WXT 官方模块。

### 常用官方模块

| 模块 | 功能 |
|------|------|
| @wxt-dev/module-react | React 集成 |
| @wxt-dev/module-vue | Vue 集成 |
| @wxt-dev/module-svelte | Svelte 集成 |
| @wxt-dev/module-solid | Solid 集成 |
| @wxt-dev/i18n | 国际化 |
| @wxt-dev/analytics | 分析统计 |

### 步骤

1. 安装模块:

```bash
# React 模块
pnpm add @wxt-dev/module-react react react-dom
pnpm add -D @types/react @types/react-dom

# i18n 模块
pnpm add @wxt-dev/i18n
```

2. 配置 wxt.config.ts:

```typescript
// wxt.config.ts
import { defineConfig } from 'wxt';

export default defineConfig({
  modules: [
    '@wxt-dev/module-react',
    '@wxt-dev/i18n',
  ],
});
```

3. 验证模块加载:

```bash
pnpm dev
# 查看控制台,应该看到模块加载日志
```

### 模块配置选项

某些模块支持配置:

```typescript
// wxt.config.ts
export default defineConfig({
  modules: [
    ['@wxt-dev/i18n', {
      // 模块选项
      defaultLocale: 'zh_CN',
    }],
  ],
});
```

---

## 练习 13.2: 创建自定义 WXT 模块

### 任务

创建一个自定义 WXT 模块。

### 步骤

1. 创建模块文件:

```typescript
// modules/my-logger/index.ts
import { defineWxtModule } from 'wxt/modules';

export default defineWxtModule({
  name: 'my-logger',
  
  setup(wxt) {
    // 模块初始化
    console.log('[my-logger] Module initialized');

    // 监听构建开始
    wxt.hook('build:before', async () => {
      console.log('[my-logger] Build started at', new Date().toISOString());
    });

    // 监听构建完成
    wxt.hook('build:done', async (output) => {
      console.log('[my-logger] Build completed');
      console.log('[my-logger] Output:', output.manifest.name);
    });

    // 监听清理
    wxt.hook('build:cleanup', async () => {
      console.log('[my-logger] Cleaning up...');
    });

    // 添加自定义 Vite 配置
    wxt.hook('vite:build:extendConfig', async (entries, config) => {
      console.log('[my-logger] Processing entries:', entries.length);
    });
  },
});
```

2. 使用模块:

```typescript
// wxt.config.ts
export default defineConfig({
  modules: [
    './modules/my-logger',
  ],
});
```

3. 运行测试:

```bash
pnpm build
# 应该在控制台看到日志输出
```

---

## 练习 13.3: 高级模块开发

### 任务

创建一个功能完整的 WXT 模块。

### 步骤

创建一个自动添加版本信息的模块:

```typescript
// modules/version-info/index.ts
import { defineWxtModule } from 'wxt/modules';
import { execSync } from 'child_process';
import fs from 'fs';
import path from 'path';

interface VersionInfoOptions {
  includeGitInfo?: boolean;
  includeBuildTime?: boolean;
  outputFile?: string;
}

export default defineWxtModule<VersionInfoOptions>({
  name: 'version-info',
  
  setup(wxt, options = {}) {
    const {
      includeGitInfo = true,
      includeBuildTime = true,
      outputFile = 'version.json',
    } = options;

    wxt.hook('build:before', async () => {
      const versionInfo: Record<string, any> = {
        version: wxt.config.manifest.version || '0.0.0',
      };

      if (includeBuildTime) {
        versionInfo.buildTime = new Date().toISOString();
      }

      if (includeGitInfo) {
        try {
          versionInfo.git = {
            commit: execSync('git rev-parse --short HEAD').toString().trim(),
            branch: execSync('git rev-parse --abbrev-ref HEAD').toString().trim(),
            dirty: execSync('git status --porcelain').toString().trim() !== '',
          };
        } catch {
          versionInfo.git = null;
        }
      }

      // 写入文件到 public 目录
      const publicDir = path.join(wxt.config.root, 'public');
      if (!fs.existsSync(publicDir)) {
        fs.mkdirSync(publicDir, { recursive: true });
      }

      fs.writeFileSync(
        path.join(publicDir, outputFile),
        JSON.stringify(versionInfo, null, 2)
      );

      console.log('[version-info] Generated version info:', versionInfo);
    });

    // 添加类型声明
    wxt.hook('prepare:types', async (entries) => {
      entries.push({
        path: 'version-info.d.ts',
        content: `
declare module 'virtual:version-info' {
  interface VersionInfo {
    version: string;
    buildTime?: string;
    git?: {
      commit: string;
      branch: string;
      dirty: boolean;
    } | null;
  }
  const versionInfo: VersionInfo;
  export default versionInfo;
}
        `.trim(),
      });
    });
  },
});
```

使用模块:

```typescript
// wxt.config.ts
export default defineConfig({
  modules: [
    ['./modules/version-info', {
      includeGitInfo: true,
      includeBuildTime: true,
    }],
  ],
});
```

在代码中使用:

```typescript
// entrypoints/popup/App.tsx
import { useEffect, useState } from 'react';

function App() {
  const [versionInfo, setVersionInfo] = useState<any>(null);

  useEffect(() => {
    fetch(browser.runtime.getURL('/version.json'))
      .then(res => res.json())
      .then(setVersionInfo);
  }, []);

  return (
    <div>
      {versionInfo && (
        <div>
          <p>Version: {versionInfo.version}</p>
          <p>Build: {versionInfo.buildTime}</p>
          {versionInfo.git && (
            <p>Commit: {versionInfo.git.commit} ({versionInfo.git.branch})</p>
          )}
        </div>
      )}
    </div>
  );
}
```

---

## 练习 13.4: ES Modules 在扩展中的应用

### 任务

理解并正确使用 ES Modules。

### ES Modules 基础

```typescript
// utils/math.ts
export function add(a: number, b: number): number {
  return a + b;
}

export function multiply(a: number, b: number): number {
  return a * b;
}

// 默认导出
export default class Calculator {
  add = add;
  multiply = multiply;
}
```

```typescript
// 使用命名导出
import { add, multiply } from '~/utils/math';

// 使用默认导出
import Calculator from '~/utils/math';

// 混合导入
import Calculator, { add } from '~/utils/math';

// 重命名导入
import { add as sum } from '~/utils/math';

// 导入所有
import * as math from '~/utils/math';
```

### 动态导入

```typescript
// entrypoints/content.ts
export default defineContentScript({
  matches: ['*://*/*'],
  
  async main() {
    // 条件加载
    const isSpecialSite = location.hostname.includes('special.com');
    
    if (isSpecialSite) {
      // 只在需要时加载
      const { specialHandler } = await import('~/handlers/special');
      specialHandler();
    }

    // 懒加载 UI
    const button = document.createElement('button');
    button.textContent = 'Show UI';
    button.onclick = async () => {
      const { createUI } = await import('./ui');
      createUI();
    };
    document.body.appendChild(button);
  },
});
```

### 路径别名

```typescript
// wxt.config.ts
export default defineConfig({
  alias: {
    // 默认别名
    '~': './src',  // 或 entrypoints 的父目录
    '@': './src',
    
    // 自定义别名
    '@components': './src/components',
    '@utils': './src/utils',
    '@hooks': './src/hooks',
  },
});
```

使用:

```typescript
// 使用别名
import { Button } from '@components/ui';
import { formatDate } from '@utils/format';
import { useStorage } from '@hooks/useStorage';
```

---

## 练习 13.5: 模块导出策略

### 任务

组织模块导出,提高代码可维护性。

### Barrel 文件模式

```typescript
// components/ui/Button.tsx
export function Button() { /* ... */ }

// components/ui/Input.tsx
export function Input() { /* ... */ }

// components/ui/Card.tsx
export function Card() { /* ... */ }

// components/ui/index.ts (Barrel 文件)
export { Button } from './Button';
export { Input } from './Input';
export { Card } from './Card';

// 也可以导出类型
export type { ButtonProps } from './Button';
```

使用:

```typescript
// 从 barrel 文件导入
import { Button, Input, Card } from '@components/ui';

// 而不是
import { Button } from '@components/ui/Button';
import { Input } from '@components/ui/Input';
```

### 按功能分组

```typescript
// features/auth/index.ts
export { AuthProvider, useAuth } from './AuthContext';
export { LoginForm } from './LoginForm';
export { authStorage } from './storage';
export type { User, AuthState } from './types';

// features/translation/index.ts
export { TranslationService } from './TranslationService';
export { useTranslation } from './hooks';
export { translationStorage } from './storage';
```

使用:

```typescript
import { AuthProvider, useAuth, LoginForm } from '@features/auth';
import { TranslationService, useTranslation } from '@features/translation';
```

---

## 练习 13.6: 共享代码与工具库

### 任务

创建扩展各部分共享的工具库。

### 步骤

```typescript
// lib/index.ts
// 统一导出所有共享工具

// API 客户端
export { ApiClient, createApiClient } from './api';
export type { ApiConfig, ApiResponse } from './api';

// 存储工具
export { StorageManager, createStorage } from './storage';
export type { StorageItem } from './storage';

// 消息工具
export { MessageBus, createMessageBus } from './messaging';
export type { Message, MessageHandler } from './messaging';

// 工具函数
export * from './utils';
```

```typescript
// lib/api.ts
export interface ApiConfig {
  baseUrl: string;
  timeout?: number;
  headers?: Record<string, string>;
}

export interface ApiResponse<T> {
  data: T;
  status: number;
}

export class ApiClient {
  constructor(private config: ApiConfig) {}

  async get<T>(path: string): Promise<ApiResponse<T>> {
    const response = await fetch(`${this.config.baseUrl}${path}`, {
      headers: this.config.headers,
    });
    return {
      data: await response.json(),
      status: response.status,
    };
  }

  async post<T>(path: string, body: any): Promise<ApiResponse<T>> {
    const response = await fetch(`${this.config.baseUrl}${path}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        ...this.config.headers,
      },
      body: JSON.stringify(body),
    });
    return {
      data: await response.json(),
      status: response.status,
    };
  }
}

export function createApiClient(config: ApiConfig): ApiClient {
  return new ApiClient(config);
}
```

在各入口点使用:

```typescript
// entrypoints/background.ts
import { createApiClient } from '~/lib';

const api = createApiClient({
  baseUrl: 'https://api.example.com',
});

export default defineBackground(() => {
  browser.runtime.onMessage.addListener(async (message) => {
    if (message.type === 'FETCH_DATA') {
      const result = await api.get('/data');
      return result.data;
    }
  });
});
```

```typescript
// entrypoints/popup/App.tsx
import { createApiClient } from '~/lib';

const api = createApiClient({
  baseUrl: 'https://api.example.com',
});

function App() {
  const [data, setData] = useState(null);

  useEffect(() => {
    api.get('/info').then(res => setData(res.data));
  }, []);

  return <div>{JSON.stringify(data)}</div>;
}
```

---

## 挑战练习

### 挑战: 创建完整的 WXT 模块包

创建一个可发布到 npm 的 WXT 模块:

```
my-wxt-module/
├── src/
│   ├── index.ts        # 模块入口
│   ├── runtime.ts      # 运行时代码
│   └── types.ts        # 类型定义
├── package.json
├── tsconfig.json
└── README.md
```

```typescript
// src/index.ts
import { defineWxtModule } from 'wxt/modules';
import type { ModuleOptions } from './types';

export default defineWxtModule<ModuleOptions>({
  name: '@my-org/wxt-module-analytics',
  
  setup(wxt, options) {
    // 1. 注入运行时代码
    wxt.hook('build:publicAssets', async (assets) => {
      assets.push({
        src: require.resolve('./runtime.js'),
        dest: 'analytics.js',
      });
    });

    // 2. 添加到 manifest
    wxt.hook('build:manifestGenerated', async (manifest) => {
      manifest.content_scripts = manifest.content_scripts || [];
      manifest.content_scripts.push({
        matches: ['<all_urls>'],
        js: ['analytics.js'],
        run_at: 'document_start',
      });
    });

    // 3. 添加类型
    wxt.hook('prepare:types', async (entries) => {
      entries.push({
        path: 'analytics.d.ts',
        content: `
declare module 'virtual:analytics' {
  export function trackEvent(name: string, data?: object): void;
  export function trackPageView(url?: string): void;
}
        `,
      });
    });
  },
});

export type { ModuleOptions } from './types';
```

---

## 自我检查

1. 如何在 wxt.config.ts 中配置模块?
2. defineWxtModule 的主要钩子有哪些?
3. ES Modules 动态导入的语法是什么?
4. 什么是 Barrel 文件?
5. 如何配置路径别名?

## 参考答案

1. 在 modules 数组中添加模块名或路径,可以是字符串或 [模块, 选项] 元组
2. build:before, build:done, vite:build:extendConfig, prepare:types 等
3. `const module = await import('./path')`
4. 将多个模块的导出集中到一个 index.ts 文件中统一导出
5. 在 wxt.config.ts 的 alias 配置中定义
