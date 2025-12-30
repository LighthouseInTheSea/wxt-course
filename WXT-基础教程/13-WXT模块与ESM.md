# WXT 模块系统与 ES Modules

## 1. WXT 模块概述

WXT 模块是一种扩展 WXT 功能的机制,类似于 Nuxt 模块。模块可以:

- 添加自定义入口点
- 修改构建配置
- 添加 Vite 插件
- 生成运行时代码
- 添加自动导入

## 2. 使用官方模块

### 2.1 React 模块

```bash
pnpm add @wxt-dev/module-react
```

```typescript
// wxt.config.ts
export default defineConfig({
  modules: ['@wxt-dev/module-react'],
  
  // 可选: 模块配置
  react: {
    vite: {
      // Vite React 插件选项
    },
  },
});
```

### 2.2 i18n 模块

```bash
pnpm add @wxt-dev/i18n
```

```typescript
// wxt.config.ts
export default defineConfig({
  modules: ['@wxt-dev/i18n'],
});
```

### 2.3 模块加载顺序

模块按数组顺序加载,后面的模块可以覆盖前面的配置:

```typescript
export default defineConfig({
  modules: [
    '@wxt-dev/module-react',  // 先加载
    '@wxt-dev/i18n',          // 后加载
    './modules/custom.ts',     // 本地模块
  ],
});
```

## 3. 创建自定义模块

### 3.1 基础模块结构

```typescript
// modules/my-module.ts
import { defineWxtModule } from 'wxt/modules';

export default defineWxtModule({
  name: 'my-module',
  
  setup(wxt) {
    console.log('Module initialized!');
    
    // 使用 hooks 扩展功能
    wxt.hook('build:before', () => {
      console.log('Build starting...');
    });
    
    wxt.hook('build:done', () => {
      console.log('Build completed!');
    });
  },
});
```

### 3.2 简化写法

```typescript
// modules/simple-module.ts
import { defineWxtModule } from 'wxt/modules';

export default defineWxtModule((wxt) => {
  // 直接使用 wxt 实例
  wxt.hook('ready', () => {
    console.log('WXT is ready!');
  });
});
```

### 3.3 在配置中使用

```typescript
// wxt.config.ts
export default defineConfig({
  modules: ['./modules/my-module.ts'],
});
```

## 4. 模块钩子 (Hooks)

### 4.1 可用钩子列表

| 钩子 | 触发时机 |
|------|----------|
| `ready` | WXT 初始化完成 |
| `build:before` | 构建开始前 |
| `build:done` | 构建完成后 |
| `build:manifestGenerated` | manifest.json 生成后 |
| `build:publicAssets` | 处理公共资源时 |
| `prepare:types` | 生成类型定义时 |
| `vite:build:extendConfig` | 扩展 Vite 配置 |
| `vite:devServer:extendConfig` | 扩展开发服务器配置 |

### 4.2 修改 Manifest

```typescript
// modules/manifest-modifier.ts
import { defineWxtModule } from 'wxt/modules';

export default defineWxtModule((wxt) => {
  wxt.hook('build:manifestGenerated', (wxt, manifest) => {
    // 添加权限
    manifest.permissions = manifest.permissions || [];
    manifest.permissions.push('notifications');
    
    // 添加命令
    manifest.commands = {
      'toggle-feature': {
        suggested_key: {
          default: 'Ctrl+Shift+Y',
          mac: 'Command+Shift+Y',
        },
        description: 'Toggle feature',
      },
    };
  });
});
```

### 4.3 添加公共资源

```typescript
// modules/copy-assets.ts
import { defineWxtModule } from 'wxt/modules';
import { resolve } from 'node:path';

export default defineWxtModule((wxt) => {
  wxt.hook('build:publicAssets', (wxt, assets) => {
    // 复制 WASM 文件
    assets.push({
      absoluteSrc: resolve('node_modules/some-package/file.wasm'),
      relativeDest: 'file.wasm',
    });
    
    // 复制预构建资源
    assets.push({
      absoluteSrc: resolve('prebuild/data.json'),
      relativeDest: 'data/config.json',
    });
  });
});
```

## 5. 模块辅助函数

### 5.1 addEntrypoint - 添加入口点

```typescript
import { defineWxtModule, addEntrypoint } from 'wxt/modules';

export default defineWxtModule((wxt) => {
  addEntrypoint(wxt, {
    type: 'content-script',
    name: 'dynamic-script',
    inputPath: resolve(__dirname, 'scripts/dynamic.ts'),
    outputDir: wxt.config.outDir,
    options: {
      matches: ['*://*.example.com/*'],
    },
  });
});
```

### 5.2 addViteConfig - 添加 Vite 配置

```typescript
import { defineWxtModule, addViteConfig } from 'wxt/modules';

export default defineWxtModule((wxt) => {
  addViteConfig(wxt, () => ({
    build: {
      sourcemap: true,
    },
    define: {
      __VERSION__: JSON.stringify('1.0.0'),
      __DEBUG__: wxt.config.mode === 'development',
    },
  }));
});
```

### 5.3 addImportPreset - 添加自动导入

```typescript
import { defineWxtModule, addImportPreset } from 'wxt/modules';

export default defineWxtModule((wxt) => {
  addImportPreset(wxt, {
    from: 'my-utils',
    imports: ['formatDate', 'parseUrl', 'debounce'],
  });
});
```

### 5.4 addAlias - 添加路径别名

```typescript
import { defineWxtModule, addAlias } from 'wxt/modules';
import { resolve } from 'node:path';

export default defineWxtModule((wxt) => {
  addAlias(wxt, '#utils', resolve(wxt.config.srcDir, 'utils'));
  addAlias(wxt, '#components', resolve(wxt.config.srcDir, 'components'));
});
```

### 5.5 addPublicAssets - 添加公共资源目录

```typescript
import { defineWxtModule, addPublicAssets } from 'wxt/modules';

export default defineWxtModule((wxt) => {
  addPublicAssets(wxt, './additional-assets');
});
```

## 6. 模块配置

### 6.1 构建时配置

```typescript
// modules/analytics-module.ts
import { defineWxtModule } from 'wxt/modules';
import 'wxt';

export interface AnalyticsOptions {
  trackingId: string;
  debug?: boolean;
}

// 扩展 WXT 配置类型
declare module 'wxt' {
  interface InlineConfig {
    analytics?: AnalyticsOptions;
  }
}

export default defineWxtModule<AnalyticsOptions>({
  name: 'analytics',
  configKey: 'analytics',
  
  setup(wxt, options) {
    if (!options?.trackingId) {
      console.warn('Analytics: No tracking ID provided');
      return;
    }
    
    console.log('Analytics configured with ID:', options.trackingId);
    
    // 使用配置
    if (options.debug) {
      wxt.hook('build:done', () => {
        console.log('Analytics: Debug mode enabled');
      });
    }
  },
});
```

使用:

```typescript
// wxt.config.ts
export default defineConfig({
  modules: ['./modules/analytics-module.ts'],
  analytics: {
    trackingId: 'G-XXXXXX',
    debug: true,
  },
});
```

### 6.2 运行时配置

```typescript
// modules/feature-flags.ts
import { defineWxtModule, addWxtPlugin } from 'wxt/modules';
import 'wxt/utils/define-app-config';

export interface FeatureFlagsOptions {
  enableNewUI?: boolean;
  experimentalFeatures?: string[];
}

// 扩展运行时配置类型
declare module 'wxt/utils/define-app-config' {
  interface WxtAppConfig {
    featureFlags?: FeatureFlagsOptions;
  }
}

export default defineWxtModule((wxt) => {
  // 运行时可以通过 useAppConfig() 访问配置
});
```

## 7. ES Modules 支持

### 7.1 后台脚本作为 ES Module

```typescript
// entrypoints/background.ts
export default defineBackground({
  type: 'module',  // 启用 ESM
  
  main() {
    console.log('Background running as ES Module');
    
    // 支持动态导入
    import('./heavy-module').then((module) => {
      module.doSomething();
    });
  },
});
```

### 7.2 ESM 的优势

- **代码分割**: 自动分割代码,减少初始加载体积
- **动态导入**: 按需加载模块
- **更好的 Tree Shaking**: 更有效的死代码消除

### 7.3 注意事项

```typescript
// 仅 MV3 支持 ESM 后台脚本
export default defineConfig({
  manifest: {
    manifest_version: 3,  // 必须是 MV3
  },
});
```

## 8. 生成运行时模块

### 8.1 生成代码文件

```typescript
// modules/config-generator.ts
import { defineWxtModule, addAlias } from 'wxt/modules';
import { resolve } from 'node:path';

export default defineWxtModule((wxt) => {
  const configPath = resolve(wxt.config.wxtDir, 'generated/config.ts');
  
  // 添加别名
  addAlias(wxt, '#config', configPath);
  
  // 生成代码
  wxt.hook('prepare:types', async (wxt, entries) => {
    const configCode = `
      // 自动生成的配置文件
      export const config = {
        version: '${wxt.config.manifest?.version || '1.0.0'}',
        mode: '${wxt.config.mode}',
        browser: '${wxt.config.browser}',
      } as const;
      
      export type Config = typeof config;
    `;
    
    entries.push({
      path: configPath,
      text: configCode,
    });
  });
});
```

### 8.2 使用生成的模块

```typescript
// 在代码中使用
import { config } from '#config';

console.log('Version:', config.version);
console.log('Mode:', config.mode);
```

## 9. 完整模块示例

### 9.1 功能增强模块

```typescript
// modules/enhanced-logging.ts
import { defineWxtModule, addViteConfig, addImportPreset } from 'wxt/modules';
import { resolve } from 'node:path';

export interface LoggingOptions {
  level: 'debug' | 'info' | 'warn' | 'error';
  prefix?: string;
}

declare module 'wxt' {
  interface InlineConfig {
    logging?: LoggingOptions;
  }
}

export default defineWxtModule<LoggingOptions>({
  name: 'enhanced-logging',
  configKey: 'logging',
  
  imports: [
    { from: '#logger', name: 'logger' },
    { from: '#logger', name: 'log' },
  ],
  
  setup(wxt, options = { level: 'info' }) {
    const loggerPath = resolve(wxt.config.wxtDir, 'logger/index.ts');
    
    // 添加路径别名
    wxt.hook('ready', () => {
      // addAlias 需要在这里调用
    });
    
    // 生成 logger 代码
    wxt.hook('prepare:types', async (wxt, entries) => {
      const code = `
        const LEVELS = { debug: 0, info: 1, warn: 2, error: 3 };
        const currentLevel = LEVELS['${options.level}'];
        const prefix = '${options.prefix || '[EXT]'}';
        
        export const logger = {
          debug: (...args: any[]) => {
            if (currentLevel <= LEVELS.debug) {
              console.debug(prefix, ...args);
            }
          },
          info: (...args: any[]) => {
            if (currentLevel <= LEVELS.info) {
              console.info(prefix, ...args);
            }
          },
          warn: (...args: any[]) => {
            if (currentLevel <= LEVELS.warn) {
              console.warn(prefix, ...args);
            }
          },
          error: (...args: any[]) => {
            if (currentLevel <= LEVELS.error) {
              console.error(prefix, ...args);
            }
          },
        };
        
        export const log = logger.info;
      `;
      
      entries.push({ path: loggerPath, text: code });
    });
  },
});
```

### 9.2 使用模块

```typescript
// wxt.config.ts
export default defineConfig({
  modules: ['./modules/enhanced-logging.ts'],
  logging: {
    level: 'debug',
    prefix: '[MyExt]',
  },
});
```

```typescript
// entrypoints/background.ts
// logger 和 log 自动导入
export default defineBackground(() => {
  logger.debug('Debug message');
  logger.info('Info message');
  log('Also info');  // log 是 logger.info 的别名
});
```

## 10. 小结

| 功能 | API | 说明 |
|------|-----|------|
| 定义模块 | `defineWxtModule` | 创建 WXT 模块 |
| 添加入口 | `addEntrypoint` | 动态添加入口点 |
| Vite 配置 | `addViteConfig` | 扩展 Vite 配置 |
| 自动导入 | `addImportPreset` | 添加自动导入 |
| 路径别名 | `addAlias` | 添加 import 别名 |
| 公共资源 | `addPublicAssets` | 添加静态资源目录 |
| ESM 后台 | `type: 'module'` | 后台脚本使用 ESM |

模块系统是 WXT 的强大特性,可以用来:
- 封装可复用的功能
- 修改构建流程
- 生成运行时代码
- 扩展配置选项
