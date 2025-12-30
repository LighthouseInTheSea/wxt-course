# WXT Web Accessible Resources 与 WASM

## 1. Web Accessible Resources 概述

Web Accessible Resources (WAR) 是扩展中可以被网页访问的资源。默认情况下,扩展的文件对网页不可见,但通过 WAR 配置,可以让特定资源可被网页访问。

### 1.1 使用场景

- 内容脚本需要加载的图片、字体
- 注入到页面的 CSS 样式
- WASM 文件
- 需要在页面主世界中加载的脚本

## 2. 配置 Web Accessible Resources

### 2.1 MV3 配置

```typescript
// wxt.config.ts
export default defineConfig({
  manifest: {
    web_accessible_resources: [
      {
        resources: [
          'images/*',
          'fonts/*',
          'styles/*.css',
        ],
        matches: ['<all_urls>'],
      },
      {
        resources: ['injected.js'],
        matches: ['*://*.example.com/*'],
      },
    ],
  },
});
```

### 2.2 MV2 配置 (Firefox)

```typescript
// wxt.config.ts
export default defineConfig({
  manifest: {
    // MV2 使用简单数组格式
    web_accessible_resources: [
      'images/*',
      'fonts/*',
      'injected.js',
    ],
  },
});
```

### 2.3 动态配置

```typescript
// wxt.config.ts
export default defineConfig({
  manifest: ({ browser, manifestVersion }) => {
    const resources = ['images/*', 'injected.js'];
    
    if (manifestVersion === 3) {
      return {
        web_accessible_resources: [
          {
            resources,
            matches: ['<all_urls>'],
          },
        ],
      };
    } else {
      return {
        web_accessible_resources: resources,
      };
    }
  },
});
```

## 3. 在内容脚本中访问资源

### 3.1 获取资源 URL

```typescript
// entrypoints/content.ts
export default defineContentScript({
  matches: ['*://*.google.com/*'],
  
  main() {
    // 获取扩展资源 URL
    const imageUrl = browser.runtime.getURL('/images/logo.png');
    console.log(imageUrl); 
    // "chrome-extension://<id>/images/logo.png"
    
    // 创建图片元素
    const img = document.createElement('img');
    img.src = imageUrl;
    document.body.appendChild(img);
  },
});
```

### 3.2 加载 CSS 文件

```typescript
// entrypoints/content.ts
export default defineContentScript({
  matches: ['<all_urls>'],
  
  main() {
    // 注入样式表
    const link = document.createElement('link');
    link.rel = 'stylesheet';
    link.href = browser.runtime.getURL('/styles/inject.css');
    document.head.appendChild(link);
  },
});
```

### 3.3 使用 import 导入资源

```typescript
// entrypoints/content.ts
import iconUrl from '/icon/128.png';

export default defineContentScript({
  matches: ['<all_urls>'],
  
  main() {
    // iconUrl 是相对路径 "/icon/128.png"
    console.log(iconUrl);
    
    // 转换为完整 URL
    const fullUrl = browser.runtime.getURL(iconUrl);
    console.log(fullUrl);
    // "chrome-extension://<id>/icon/128.png"
  },
});
```

## 4. WASM 支持

### 4.1 复制 WASM 文件

使用 WXT 模块复制 WASM 文件到输出目录:

```typescript
// modules/wasm-loader.ts
import { defineWxtModule } from 'wxt/modules';
import { resolve } from 'node:path';

export default defineWxtModule((wxt) => {
  wxt.hook('build:publicAssets', (_, assets) => {
    assets.push({
      absoluteSrc: resolve(
        'node_modules/@oxc-parser/wasm/web/oxc_parser_wasm_bg.wasm'
      ),
      relativeDest: 'parser.wasm',
    });
  });
});
```

### 4.2 配置 WAR

```typescript
// wxt.config.ts
export default defineConfig({
  modules: ['./modules/wasm-loader.ts'],
  manifest: {
    web_accessible_resources: [
      {
        resources: ['parser.wasm'],
        matches: ['*://*.github.com/*'],
      },
    ],
  },
});
```

### 4.3 在内容脚本中加载 WASM

```typescript
// entrypoints/content.ts
import initWasm, { parseSync } from '@oxc-parser/wasm';

export default defineContentScript({
  matches: ['*://*.github.com/*'],
  
  async main(ctx) {
    // 检查是否是 TypeScript 文件
    if (!location.pathname.endsWith('.ts')) return;
    
    // 加载 WASM
    await initWasm({
      module_or_path: browser.runtime.getURL('/parser.wasm'),
    });
    
    // 获取页面代码
    const codeElement = document.querySelector('pre code');
    if (!codeElement) return;
    
    const code = codeElement.textContent || '';
    
    // 使用 WASM 解析代码
    const ast = parseSync(code, { 
      sourceFilename: location.pathname 
    });
    
    console.log('Parsed AST:', ast);
  },
});
```

### 4.4 完整 WASM 示例

```typescript
// modules/sql-wasm.ts
import { defineWxtModule } from 'wxt/modules';
import { resolve } from 'node:path';

export default defineWxtModule((wxt) => {
  wxt.hook('build:publicAssets', (_, assets) => {
    assets.push({
      absoluteSrc: resolve('node_modules/sql.js/dist/sql-wasm.wasm'),
      relativeDest: 'sql-wasm.wasm',
    });
  });
});
```

```typescript
// entrypoints/background.ts
import initSqlJs, { Database } from 'sql.js';

let db: Database | null = null;

export default defineBackground({
  type: 'module',
  
  async main() {
    // 初始化 SQL.js
    const SQL = await initSqlJs({
      locateFile: (file) => browser.runtime.getURL(`/${file}`),
    });
    
    // 创建数据库
    db = new SQL.Database();
    
    // 创建表
    db.run(`
      CREATE TABLE IF NOT EXISTS bookmarks (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        url TEXT NOT NULL,
        title TEXT,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
      )
    `);
    
    console.log('SQLite database initialized');
  },
});
```

## 5. 动态资源加载

### 5.1 Fetch 远程资源

```typescript
// entrypoints/content.ts
export default defineContentScript({
  matches: ['<all_urls>'],
  
  async main() {
    // 加载扩展中的 JSON 配置
    const configUrl = browser.runtime.getURL('/config/settings.json');
    const response = await fetch(configUrl);
    const config = await response.json();
    
    console.log('Loaded config:', config);
  },
});
```

### 5.2 加载文本文件

```typescript
// entrypoints/background.ts
export default defineBackground(async () => {
  // 加载模板文件
  const templateUrl = browser.runtime.getURL('/templates/email.html');
  const response = await fetch(templateUrl);
  const template = await response.text();
  
  console.log('Template loaded:', template.length, 'chars');
});
```

## 6. 资源安全性

### 6.1 限制访问范围

```typescript
// wxt.config.ts
export default defineConfig({
  manifest: {
    web_accessible_resources: [
      {
        // 只允许特定网站访问
        resources: ['injected.js'],
        matches: [
          'https://example.com/*',
          'https://*.example.org/*',
        ],
        // 不允许从扩展以外的地方访问
        use_dynamic_url: true,
      },
      {
        // 允许所有网站访问图片
        resources: ['images/*'],
        matches: ['<all_urls>'],
      },
    ],
  },
});
```

### 6.2 使用动态 URL

MV3 支持动态 URL,增加安全性:

```typescript
// wxt.config.ts
export default defineConfig({
  manifest: {
    web_accessible_resources: [
      {
        resources: ['sensitive-script.js'],
        matches: ['*://*.trusted-site.com/*'],
        use_dynamic_url: true,  // 使用动态 URL
      },
    ],
  },
});
```

### 6.3 扩展 ID 变化处理

```typescript
// 不要硬编码扩展 ID
// 错误
const url = 'chrome-extension://abcdefg/image.png';

// 正确: 使用 runtime API
const url = browser.runtime.getURL('/image.png');
```

## 7. assets vs public 目录

### 7.1 assets 目录

- 位置: `src/assets/` 或 `assets/`
- 会被 Vite 处理 (压缩、哈希等)
- 使用 `~/assets/` 导入

```typescript
// 导入 assets 中的图片
import logo from '~/assets/images/logo.png';

// logo 包含处理后的路径和哈希
console.log(logo); // "/assets/logo-abc123.png"
```

### 7.2 public 目录

- 位置: `public/`
- 原样复制,不经过处理
- 使用绝对路径 `/` 导入

```typescript
// 导入 public 中的图片
import icon from '/icon/128.png';

// 保持原始路径
console.log(icon); // "/icon/128.png"
```

### 7.3 选择建议

| 场景 | 推荐目录 |
|------|----------|
| 需要在多处导入的图片 | assets |
| 需要哈希防缓存 | assets |
| 扩展图标 | public |
| _locales 语言文件 | public |
| 需要固定路径访问 | public |
| WASM 文件 | public (通过模块复制) |

## 8. 实战示例: 自定义字体注入

### 8.1 准备字体文件

```
public/
└── fonts/
    ├── custom-font.woff2
    └── custom-font.woff
```

### 8.2 配置

```typescript
// wxt.config.ts
export default defineConfig({
  manifest: {
    web_accessible_resources: [
      {
        resources: ['fonts/*'],
        matches: ['<all_urls>'],
      },
    ],
  },
});
```

### 8.3 注入字体

```typescript
// entrypoints/content.ts
export default defineContentScript({
  matches: ['<all_urls>'],
  
  main() {
    // 创建 @font-face 规则
    const fontUrl = browser.runtime.getURL('/fonts/custom-font.woff2');
    
    const style = document.createElement('style');
    style.textContent = `
      @font-face {
        font-family: 'CustomFont';
        src: url('${fontUrl}') format('woff2');
        font-weight: normal;
        font-style: normal;
      }
      
      .use-custom-font {
        font-family: 'CustomFont', sans-serif;
      }
    `;
    
    document.head.appendChild(style);
  },
});
```

## 9. 故障排除

### 9.1 资源加载失败

```typescript
// 检查资源是否可访问
const url = browser.runtime.getURL('/image.png');
fetch(url)
  .then(res => {
    if (!res.ok) {
      console.error('Resource not accessible:', res.status);
    }
  })
  .catch(err => {
    console.error('Failed to load resource:', err);
  });
```

### 9.2 常见错误

**错误: "Denying load of chrome-extension://..."**
- 原因: 资源未在 web_accessible_resources 中声明
- 解决: 添加资源到 WAR 配置

**错误: WASM 加载失败**
- 原因: WASM 文件路径错误或未配置 WAR
- 解决: 检查文件是否存在于输出目录,确认 WAR 配置正确

## 10. 小结

| 概念 | 说明 |
|------|------|
| WAR | 让扩展资源可被网页访问 |
| matches | 限制哪些网站可以访问资源 |
| runtime.getURL() | 获取扩展资源的完整 URL |
| use_dynamic_url | MV3 安全特性,使用动态 URL |
| assets/ | Vite 处理的资源 |
| public/ | 原样复制的资源 |
| WASM | 通过模块复制到 public,配置 WAR |
