# WXT 实战练习 14: Web Accessible Resources 与 WASM

## 练习目标

通过本练习,你将:
- 理解 Web Accessible Resources 机制
- 配置可访问资源
- 在扩展中使用 WASM
- 实现高性能计算功能

## 练习 14.1: Web Accessible Resources 基础

### 任务

理解并配置 Web Accessible Resources。

### 什么是 Web Accessible Resources?

默认情况下,扩展的资源 (脚本、图片等) 无法被网页直接访问。Web Accessible Resources 允许你指定哪些资源可以被网页访问。

### 步骤

1. 配置 manifest:

```typescript
// wxt.config.ts
export default defineConfig({
  manifest: {
    web_accessible_resources: [
      {
        // 可访问的资源
        resources: [
          'images/*',           // 所有图片
          'scripts/inject.js',  // 注入脚本
          'styles/inject.css',  // 注入样式
          'wasm/*.wasm',        // WASM 文件
        ],
        // 允许访问的页面
        matches: ['*://*/*'],
        // 可选: 扩展页面也能访问
        extension_ids: [],
      },
    ],
  },
});
```

2. 放置资源文件:

```
public/
├── images/
│   ├── logo.png
│   └── icon.svg
├── scripts/
│   └── inject.js
├── styles/
│   └── inject.css
└── wasm/
    └── processor.wasm
```

3. 在内容脚本中使用:

```typescript
// entrypoints/content.ts
export default defineContentScript({
  matches: ['*://*/*'],
  
  main() {
    // 获取资源 URL
    const logoUrl = browser.runtime.getURL('/images/logo.png');
    const injectScriptUrl = browser.runtime.getURL('/scripts/inject.js');
    const injectCssUrl = browser.runtime.getURL('/styles/inject.css');

    // 注入图片
    const img = document.createElement('img');
    img.src = logoUrl;
    img.style.cssText = 'position: fixed; top: 10px; right: 10px; width: 50px;';
    document.body.appendChild(img);

    // 注入样式
    const link = document.createElement('link');
    link.rel = 'stylesheet';
    link.href = injectCssUrl;
    document.head.appendChild(link);

    // 注入脚本到主世界
    const script = document.createElement('script');
    script.src = injectScriptUrl;
    document.head.appendChild(script);
  },
});
```

### 验证

- [ ] 图片正确显示
- [ ] 样式正确应用
- [ ] 脚本正确执行

---

## 练习 14.2: 动态资源 URL

### 任务

动态构建和使用资源 URL。

### 步骤

```typescript
// utils/resources.ts
export function getAssetURL(path: string): string {
  // 确保路径以 / 开头
  const normalizedPath = path.startsWith('/') ? path : `/${path}`;
  return browser.runtime.getURL(normalizedPath);
}

export function getImageURL(name: string): string {
  return getAssetURL(`/images/${name}`);
}

export function getScriptURL(name: string): string {
  return getAssetURL(`/scripts/${name}`);
}

export function getStyleURL(name: string): string {
  return getAssetURL(`/styles/${name}`);
}

// 预加载资源
export async function preloadAsset(url: string): Promise<void> {
  return new Promise((resolve, reject) => {
    const link = document.createElement('link');
    link.rel = 'preload';
    link.href = url;
    link.onload = () => resolve();
    link.onerror = reject;
    document.head.appendChild(link);
  });
}

// 检查资源是否可用
export async function checkAssetAvailable(path: string): Promise<boolean> {
  try {
    const url = getAssetURL(path);
    const response = await fetch(url, { method: 'HEAD' });
    return response.ok;
  } catch {
    return false;
  }
}
```

使用:

```typescript
// entrypoints/content.ts
import { getImageURL, getScriptURL, preloadAsset } from '~/utils/resources';

export default defineContentScript({
  matches: ['*://*/*'],
  
  async main() {
    // 预加载关键资源
    await preloadAsset(getImageURL('logo.png'));
    
    // 动态创建元素
    const img = document.createElement('img');
    img.src = getImageURL('logo.png');
    
    // 条件加载
    const isDarkMode = window.matchMedia('(prefers-color-scheme: dark)').matches;
    img.src = getImageURL(isDarkMode ? 'logo-dark.png' : 'logo-light.png');
  },
});
```

---

## 练习 14.3: WASM 集成基础

### 任务

在扩展中使用 WebAssembly。

### 步骤

1. 创建 WASM 模块 (使用 Rust):

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn fibonacci(n: u32) -> u32 {
    match n {
        0 => 0,
        1 => 1,
        _ => fibonacci(n - 1) + fibonacci(n - 2)
    }
}

#[wasm_bindgen]
pub fn process_text(text: &str) -> String {
    text.to_uppercase()
}

#[wasm_bindgen]
pub struct TextProcessor {
    buffer: String,
}

#[wasm_bindgen]
impl TextProcessor {
    #[wasm_bindgen(constructor)]
    pub fn new() -> TextProcessor {
        TextProcessor {
            buffer: String::new(),
        }
    }

    pub fn append(&mut self, text: &str) {
        self.buffer.push_str(text);
    }

    pub fn get_result(&self) -> String {
        self.buffer.clone()
    }

    pub fn word_count(&self) -> usize {
        self.buffer.split_whitespace().count()
    }
}
```

2. 编译 WASM:

```bash
# 安装 wasm-pack
cargo install wasm-pack

# 编译
wasm-pack build --target web
```

3. 配置 WXT:

```typescript
// wxt.config.ts
export default defineConfig({
  manifest: {
    web_accessible_resources: [
      {
        resources: ['wasm/*.wasm'],
        matches: ['*://*/*'],
      },
    ],
  },
  
  // 使用 WXT 模块处理 WASM
  hooks: {
    'build:publicAssets': async (wxt, assets) => {
      // 复制 WASM 文件到 public
      assets.push({
        src: './pkg/my_wasm_bg.wasm',
        dest: 'wasm/processor.wasm',
      });
    },
  },
});
```

4. 在代码中使用:

```typescript
// utils/wasm.ts
let wasmModule: any = null;

export async function loadWasm(): Promise<void> {
  if (wasmModule) return;

  const wasmUrl = browser.runtime.getURL('/wasm/processor.wasm');
  
  // 方法1: 使用 fetch + instantiate
  const response = await fetch(wasmUrl);
  const bytes = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(bytes);
  wasmModule = instance.exports;
}

export async function fibonacci(n: number): Promise<number> {
  await loadWasm();
  return wasmModule.fibonacci(n);
}
```

---

## 练习 14.4: 使用 wasm-bindgen

### 任务

使用 wasm-bindgen 生成的 JavaScript 绑定。

### 步骤

1. 复制生成的文件:

```
public/
└── wasm/
    ├── processor.wasm
    └── processor.js      # wasm-bindgen 生成
```

2. 创建加载器:

```typescript
// utils/wasmLoader.ts
interface WasmExports {
  fibonacci: (n: number) => number;
  process_text: (text: string) => string;
  TextProcessor: new () => {
    append(text: string): void;
    get_result(): string;
    word_count(): number;
  };
}

let wasmExports: WasmExports | null = null;
let loadingPromise: Promise<WasmExports> | null = null;

export async function getWasm(): Promise<WasmExports> {
  if (wasmExports) {
    return wasmExports;
  }

  if (loadingPromise) {
    return loadingPromise;
  }

  loadingPromise = (async () => {
    // 获取 WASM URL
    const wasmUrl = browser.runtime.getURL('/wasm/processor.wasm');
    
    // 动态导入 JS 绑定
    const jsUrl = browser.runtime.getURL('/wasm/processor.js');
    
    // 注意: 扩展中可能需要特殊处理
    const response = await fetch(jsUrl);
    const jsCode = await response.text();
    
    // 创建模块
    const blob = new Blob([jsCode], { type: 'text/javascript' });
    const blobUrl = URL.createObjectURL(blob);
    
    try {
      const module = await import(/* @vite-ignore */ blobUrl);
      await module.default(wasmUrl);
      wasmExports = module;
      return wasmExports;
    } finally {
      URL.revokeObjectURL(blobUrl);
    }
  })();

  return loadingPromise;
}

// 便捷函数
export async function fibonacci(n: number): Promise<number> {
  const wasm = await getWasm();
  return wasm.fibonacci(n);
}

export async function processText(text: string): Promise<string> {
  const wasm = await getWasm();
  return wasm.process_text(text);
}
```

3. 在组件中使用:

```tsx
// entrypoints/popup/App.tsx
import { useState } from 'react';
import { fibonacci, processText } from '~/utils/wasmLoader';

function App() {
  const [result, setResult] = useState<string>('');
  const [loading, setLoading] = useState(false);

  const handleCalculate = async () => {
    setLoading(true);
    try {
      const fib = await fibonacci(40);
      setResult(`Fibonacci(40) = ${fib}`);
    } catch (error) {
      setResult(`Error: ${error}`);
    } finally {
      setLoading(false);
    }
  };

  const handleProcess = async () => {
    setLoading(true);
    try {
      const processed = await processText('hello world');
      setResult(`Processed: ${processed}`);
    } catch (error) {
      setResult(`Error: ${error}`);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div style={{ padding: 20 }}>
      <h2>WASM Demo</h2>
      
      <div style={{ display: 'flex', gap: 8 }}>
        <button onClick={handleCalculate} disabled={loading}>
          Calculate Fibonacci
        </button>
        <button onClick={handleProcess} disabled={loading}>
          Process Text
        </button>
      </div>

      {loading && <p>Loading...</p>}
      {result && <p>{result}</p>}
    </div>
  );
}

export default App;
```

---

## 练习 14.5: WASM 性能优化

### 任务

优化 WASM 加载和执行性能。

### 步骤

1. 使用 streaming 编译:

```typescript
// utils/wasmStreaming.ts
export async function loadWasmStreaming(wasmPath: string) {
  const url = browser.runtime.getURL(wasmPath);
  
  // streaming 编译更快
  if (WebAssembly.instantiateStreaming) {
    const response = fetch(url);
    const { instance } = await WebAssembly.instantiateStreaming(response);
    return instance.exports;
  }
  
  // 降级方案
  const response = await fetch(url);
  const bytes = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(bytes);
  return instance.exports;
}
```

2. 预加载 WASM:

```typescript
// entrypoints/background.ts
export default defineBackground(() => {
  // 预编译 WASM
  let compiledModule: WebAssembly.Module | null = null;

  async function precompileWasm() {
    const url = browser.runtime.getURL('/wasm/processor.wasm');
    const response = await fetch(url);
    const bytes = await response.arrayBuffer();
    compiledModule = await WebAssembly.compile(bytes);
    console.log('WASM precompiled');
  }

  // 启动时预编译
  precompileWasm();

  // 响应请求
  browser.runtime.onMessage.addListener(async (message) => {
    if (message.type === 'GET_WASM_MODULE') {
      if (!compiledModule) {
        await precompileWasm();
      }
      // 实例化预编译的模块
      const instance = await WebAssembly.instantiate(compiledModule!);
      return { ready: true };
    }
  });
});
```

3. 使用 Web Worker:

```typescript
// workers/wasm-worker.ts
let wasmInstance: WebAssembly.Instance | null = null;

async function initWasm() {
  const response = await fetch('/wasm/processor.wasm');
  const bytes = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(bytes);
  wasmInstance = instance;
}

self.onmessage = async (e: MessageEvent) => {
  const { type, payload, id } = e.data;

  if (!wasmInstance) {
    await initWasm();
  }

  let result;
  
  switch (type) {
    case 'fibonacci':
      result = (wasmInstance!.exports.fibonacci as Function)(payload.n);
      break;
    case 'process_text':
      result = (wasmInstance!.exports.process_text as Function)(payload.text);
      break;
  }

  self.postMessage({ id, result });
};
```

使用 Worker:

```typescript
// utils/wasmWorker.ts
const worker = new Worker(
  browser.runtime.getURL('/workers/wasm-worker.js')
);

let messageId = 0;
const pending = new Map<number, (result: any) => void>();

worker.onmessage = (e: MessageEvent) => {
  const { id, result } = e.data;
  const resolve = pending.get(id);
  if (resolve) {
    resolve(result);
    pending.delete(id);
  }
};

export function wasmCall<T>(type: string, payload: any): Promise<T> {
  return new Promise((resolve) => {
    const id = messageId++;
    pending.set(id, resolve);
    worker.postMessage({ type, payload, id });
  });
}

export function fibonacci(n: number): Promise<number> {
  return wasmCall('fibonacci', { n });
}
```

---

## 练习 14.6: 实用案例 - 图片处理

### 任务

使用 WASM 实现高性能图片处理。

### 步骤

1. WASM 图片处理函数 (Rust):

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn grayscale(data: &mut [u8]) {
    for chunk in data.chunks_mut(4) {
        let r = chunk[0] as f32;
        let g = chunk[1] as f32;
        let b = chunk[2] as f32;
        let gray = (0.299 * r + 0.587 * g + 0.114 * b) as u8;
        chunk[0] = gray;
        chunk[1] = gray;
        chunk[2] = gray;
        // alpha 保持不变
    }
}

#[wasm_bindgen]
pub fn invert(data: &mut [u8]) {
    for chunk in data.chunks_mut(4) {
        chunk[0] = 255 - chunk[0];
        chunk[1] = 255 - chunk[1];
        chunk[2] = 255 - chunk[2];
    }
}

#[wasm_bindgen]
pub fn adjust_brightness(data: &mut [u8], factor: f32) {
    for chunk in data.chunks_mut(4) {
        chunk[0] = ((chunk[0] as f32 * factor).min(255.0)) as u8;
        chunk[1] = ((chunk[1] as f32 * factor).min(255.0)) as u8;
        chunk[2] = ((chunk[2] as f32 * factor).min(255.0)) as u8;
    }
}
```

2. TypeScript 封装:

```typescript
// utils/imageProcessor.ts
interface ImageProcessorWasm {
  grayscale(data: Uint8ClampedArray): void;
  invert(data: Uint8ClampedArray): void;
  adjust_brightness(data: Uint8ClampedArray, factor: number): void;
}

let processor: ImageProcessorWasm | null = null;

async function loadProcessor(): Promise<ImageProcessorWasm> {
  if (processor) return processor;
  
  const url = browser.runtime.getURL('/wasm/image_processor.wasm');
  const response = await fetch(url);
  const bytes = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(bytes);
  processor = instance.exports as unknown as ImageProcessorWasm;
  return processor;
}

export async function processImage(
  imageData: ImageData,
  operation: 'grayscale' | 'invert' | 'brightness',
  options?: { brightness?: number }
): Promise<ImageData> {
  const proc = await loadProcessor();
  
  // 复制数据
  const data = new Uint8ClampedArray(imageData.data);
  
  switch (operation) {
    case 'grayscale':
      proc.grayscale(data);
      break;
    case 'invert':
      proc.invert(data);
      break;
    case 'brightness':
      proc.adjust_brightness(data, options?.brightness ?? 1.2);
      break;
  }
  
  return new ImageData(data, imageData.width, imageData.height);
}
```

3. 在 Content Script 中使用:

```typescript
// entrypoints/image-editor.content.ts
import { processImage } from '~/utils/imageProcessor';

export default defineContentScript({
  matches: ['*://*/*'],
  
  main() {
    // 监听图片右键菜单
    document.addEventListener('contextmenu', async (e) => {
      const target = e.target as HTMLImageElement;
      if (target.tagName !== 'IMG') return;

      // 创建处理菜单
      showImageMenu(target, e);
    });
  },
});

async function applyFilter(
  img: HTMLImageElement,
  filter: 'grayscale' | 'invert' | 'brightness'
) {
  // 创建 canvas
  const canvas = document.createElement('canvas');
  canvas.width = img.naturalWidth;
  canvas.height = img.naturalHeight;
  
  const ctx = canvas.getContext('2d')!;
  ctx.drawImage(img, 0, 0);
  
  // 获取图片数据
  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
  
  // 使用 WASM 处理
  const processed = await processImage(imageData, filter);
  
  // 应用结果
  ctx.putImageData(processed, 0, 0);
  img.src = canvas.toDataURL();
}
```

---

## 挑战练习

### 挑战: 完整的图片处理扩展

创建一个包含以下功能的图片处理扩展:
1. 右键菜单处理页面图片
2. 多种滤镜效果 (WASM 实现)
3. 批量处理
4. 保存/下载

---

## 自我检查

1. Web Accessible Resources 有什么作用?
2. 如何获取扩展资源的 URL?
3. WASM 在扩展中有什么优势?
4. 如何优化 WASM 加载性能?
5. 为什么需要配置 web_accessible_resources 来使用 WASM?

## 参考答案

1. 允许网页访问扩展内的特定资源 (如注入脚本、图片等)
2. 使用 `browser.runtime.getURL('/path/to/resource')`
3. 高性能计算,接近原生速度,适合密集计算任务
4. 使用 streaming 编译、预编译、Web Worker 等技术
5. 因为内容脚本注入的资源需要被网页加载,必须声明为可访问资源
