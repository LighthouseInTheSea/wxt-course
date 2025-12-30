# React 集成开发

> **导航**: [上一篇: Storage存储系统](./07-Storage存储系统.md) | [下一篇: 构建与发布](./09-构建与发布.md) | [目录](../README.md)
>
> **实战练习**: [React集成实战](../WXT-实战练习/08-React集成实战.md)

---

## 前言

React 是目前最流行的前端框架之一，WXT 对 React 提供了一流的支持。本章将介绍如何在浏览器扩展的各个场景中使用 React：Popup 页面、Options 页面、以及最具挑战性的内容脚本 UI。

**本章你将学到：**
- 如何配置 WXT + React 项目
- Popup 和 Options 页面的 React 开发
- 在内容脚本中使用 React（Shadow DOM 隔离）
- 扩展开发专用的自定义 Hooks
- 状态管理和组件库集成

---

## 1. 项目初始化

### 1.1 使用脚手架创建项目

最简单的方式是在创建项目时选择 React 模板：

```bash
pnpm dlx wxt@latest init my-react-extension
# 在交互式菜单中选择 React 模板
```

脚手架会自动配置好所有必要的依赖和配置。

### 1.2 手动配置 React

如果你有一个现有的 WXT 项目，可以手动添加 React 支持：

```bash
# 1. 安装 React 相关依赖
pnpm add react react-dom
pnpm add -D @types/react @types/react-dom @vitejs/plugin-react
```

```typescript
// 2. 配置 wxt.config.ts
import { defineConfig } from 'wxt';
import react from '@vitejs/plugin-react';

export default defineConfig({
  vite: () => ({
    plugins: [react()],  // 添加 React 插件
  }),
});
```

**为什么用 @vitejs/plugin-react?** WXT 基于 Vite 构建，这个插件提供了 Fast Refresh（热更新）和 JSX 转换支持。

---

## 2. Popup 页面开发

Popup 是用户点击扩展图标时弹出的小窗口，是扩展最常用的交互界面。

### 2.1 目录结构

```
entrypoints/
└── popup/
    ├── index.html     # HTML 入口
    ├── main.tsx       # React 入口
    ├── App.tsx        # 主组件
    ├── App.css        # 样式
    └── components/    # 子组件目录
        ├── Header.tsx
        └── Settings.tsx
```

### 2.2 HTML 入口文件

```html
<!-- entrypoints/popup/index.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>My Extension</title>
  <style>
    /* 重要：设置 popup 的尺寸 */
    body {
      width: 350px;       /* Popup 宽度 */
      min-height: 400px;  /* 最小高度 */
      margin: 0;
      padding: 0;
    }
  </style>
</head>
<body>
  <div id="root"></div>
  <!-- 注意：type="module" 是必须的 -->
  <script type="module" src="./main.tsx"></script>
</body>
</html>
```

**重要提示**：Popup 的尺寸由 CSS 控制，不设置的话会非常小。建议宽度在 300-400px 之间。

### 2.3 React 入口

```typescript
// entrypoints/popup/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './App.css';

// 标准的 React 18 挂载方式
ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### 2.4 主组件示例

下面是一个完整的 Popup 组件示例，展示了如何：
- 读取和更新存储
- 与当前标签页通信
- 打开 Options 页面

```typescript
// entrypoints/popup/App.tsx
import { useState, useEffect } from 'react';
import { storage } from 'wxt/storage';

// 定义存储项
const enabledItem = storage.defineItem<boolean>('local:enabled', {
  fallback: true,
});

function App() {
  const [enabled, setEnabled] = useState(true);
  const [loading, setLoading] = useState(true);
  
  // 组件挂载时读取存储
  useEffect(() => {
    enabledItem.getValue().then((value) => {
      setEnabled(value);
      setLoading(false);
    });
  }, []);
  
  // 切换功能开关
  const toggleEnabled = async () => {
    const newValue = !enabled;
    await enabledItem.setValue(newValue);
    setEnabled(newValue);
    
    // 通知当前标签页的内容脚本
    const [tab] = await browser.tabs.query({ 
      active: true, 
      currentWindow: true 
    });
    
    if (tab?.id) {
      browser.tabs.sendMessage(tab.id, {
        type: 'TOGGLE_FEATURE',
        enabled: newValue,
      });
    }
  };
  
  // 加载中状态
  if (loading) {
    return <div className="loading">Loading...</div>;
  }
  
  return (
    <div className="app">
      {/* 头部 */}
      <header className="header">
        <h1>My Extension</h1>
      </header>
      
      {/* 主要内容 */}
      <main className="main">
        <div className="toggle-row">
          <span>Enable Feature</span>
          <button 
            className={`toggle ${enabled ? 'active' : ''}`}
            onClick={toggleEnabled}
          >
            {enabled ? 'ON' : 'OFF'}
          </button>
        </div>
      </main>
      
      {/* 底部 - 打开设置页面 */}
      <footer className="footer">
        <button onClick={() => browser.runtime.openOptionsPage()}>
          Settings
        </button>
      </footer>
    </div>
  );
}

export default App;
```

---

## 3. Options 页面开发

Options 页面用于展示完整的扩展设置界面，通常比 Popup 更复杂。

### 3.1 目录结构

```
entrypoints/
└── options/
    ├── index.html
    ├── main.tsx
    ├── App.tsx
    └── pages/         # 可以按功能分页
        ├── General.tsx
        ├── Appearance.tsx
        └── About.tsx
```

### 3.2 完整 Options 示例

```typescript
// entrypoints/options/App.tsx
import { useState, useEffect } from 'react';
import { storage } from 'wxt/storage';

// 设置接口定义
interface Settings {
  theme: 'light' | 'dark' | 'system';
  language: string;
  autoTranslate: boolean;
  targetLanguage: string;
}

// 创建类型安全的存储项
const settingsItem = storage.defineItem<Settings>('sync:settings', {
  fallback: {
    theme: 'system',
    language: 'en',
    autoTranslate: false,
    targetLanguage: 'zh-CN',
  },
});

function App() {
  const [settings, setSettings] = useState<Settings | null>(null);
  const [saved, setSaved] = useState(false);
  
  // 加载设置
  useEffect(() => {
    settingsItem.getValue().then(setSettings);
  }, []);
  
  // 更新单个设置项的通用方法
  const updateSetting = async <K extends keyof Settings>(
    key: K,
    value: Settings[K]
  ) => {
    if (!settings) return;
    
    const newSettings = { ...settings, [key]: value };
    setSettings(newSettings);
    await settingsItem.setValue(newSettings);
    
    // 显示保存成功提示
    setSaved(true);
    setTimeout(() => setSaved(false), 2000);
  };
  
  if (!settings) {
    return <div>Loading...</div>;
  }
  
  return (
    <div className="options-container">
      <h1>Extension Settings</h1>
      
      {/* 保存成功提示 */}
      {saved && <div className="save-indicator">Settings saved!</div>}
      
      {/* 外观设置 */}
      <section className="section">
        <h2>Appearance</h2>
        
        <div className="form-group">
          <label>Theme</label>
          <select
            value={settings.theme}
            onChange={(e) => updateSetting('theme', e.target.value as Settings['theme'])}
          >
            <option value="light">Light</option>
            <option value="dark">Dark</option>
            <option value="system">System</option>
          </select>
        </div>
        
        <div className="form-group">
          <label>Language</label>
          <select
            value={settings.language}
            onChange={(e) => updateSetting('language', e.target.value)}
          >
            <option value="en">English</option>
            <option value="zh-CN">Simplified Chinese</option>
            <option value="ja">Japanese</option>
          </select>
        </div>
      </section>
      
      {/* 翻译设置 */}
      <section className="section">
        <h2>Translation</h2>
        
        <div className="form-group">
          <label>
            <input
              type="checkbox"
              checked={settings.autoTranslate}
              onChange={(e) => updateSetting('autoTranslate', e.target.checked)}
            />
            Auto-translate pages
          </label>
        </div>
        
        <div className="form-group">
          <label>Target Language</label>
          <select
            value={settings.targetLanguage}
            onChange={(e) => updateSetting('targetLanguage', e.target.value)}
          >
            <option value="zh-CN">Simplified Chinese</option>
            <option value="en">English</option>
            <option value="ja">Japanese</option>
          </select>
        </div>
      </section>
    </div>
  );
}

export default App;
```

---

## 4. 内容脚本中使用 React

在内容脚本中使用 React 是最具挑战性的场景，因为需要处理样式隔离问题。

### 4.1 为什么需要 Shadow DOM?

内容脚本注入的 UI 会直接出现在网页中，会遇到两个问题：
1. **样式冲突**：网页的 CSS 可能影响你的组件
2. **样式泄露**：你的 CSS 可能破坏网页布局

**解决方案**：使用 Shadow DOM 创建一个隔离的 DOM 环境。

### 4.2 创建 Shadow DOM UI

WXT 提供了 `createShadowRootUi` 函数来简化这个过程：

```typescript
// entrypoints/content.tsx
import ReactDOM from 'react-dom/client';
import { createShadowRootUi } from 'wxt/client';
import App from './content/App';

export default defineContentScript({
  matches: ['<all_urls>'],
  cssInjectionMode: 'ui',  // 关键：将 CSS 注入到 Shadow DOM
  
  async main(ctx) {
    // 创建 Shadow DOM UI
    const ui = await createShadowRootUi(ctx, {
      name: 'my-extension-ui',  // Shadow DOM 的标识名
      position: 'inline',       // 定位方式
      anchor: 'body',           // 插入位置
      append: 'first',          // 插入到 anchor 的第一个子元素位置
      
      // 挂载时创建 React 根
      onMount(container) {
        const root = ReactDOM.createRoot(container);
        root.render(<App />);
        return root;  // 返回值会传给 onRemove
      },
      
      // 卸载时清理
      onRemove(root) {
        root?.unmount();
      },
    });
    
    // 挂载 UI
    ui.mount();
  },
});
```

### 4.3 Shadow DOM 内的 React 组件

```typescript
// entrypoints/content/App.tsx
import { useState, useEffect } from 'react';
import './style.css';  // 这些样式只在 Shadow DOM 内生效

function App() {
  const [visible, setVisible] = useState(false);
  const [position, setPosition] = useState({ x: 0, y: 0 });
  
  // 监听来自 popup 或 background 的消息
  useEffect(() => {
    const handleMessage = (message: any) => {
      if (message.type === 'SHOW_UI') {
        setPosition(message.position);
        setVisible(true);
      }
    };
    
    browser.runtime.onMessage.addListener(handleMessage);
    return () => browser.runtime.onMessage.removeListener(handleMessage);
  }, []);
  
  if (!visible) return null;
  
  return (
    <div 
      className="floating-panel"
      style={{ left: position.x, top: position.y }}
    >
      <div className="panel-header">
        <span>Translation</span>
        <button onClick={() => setVisible(false)}>X</button>
      </div>
      <div className="panel-content">
        {/* 翻译内容 */}
      </div>
    </div>
  );
}

export default App;
```

### 4.4 Shadow DOM 内的样式

```css
/* entrypoints/content/style.css */
/* 这些样式完全隔离，不会影响网页，也不受网页影响 */

.floating-panel {
  position: fixed;
  z-index: 2147483647;  /* 最大 z-index，确保在最上层 */
  background: white;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}

.panel-header {
  display: flex;
  justify-content: space-between;
  padding: 12px 16px;
  border-bottom: 1px solid #eee;
}
```

---

## 5. 扩展开发专用 Hooks

扩展开发有一些独特的需求，可以封装成 Hooks 复用。

### 5.1 useStorage - 响应式存储

```typescript
// hooks/useStorage.ts
import { useState, useEffect, useCallback } from 'react';
import { storage } from 'wxt/storage';

export function useStorage<T>(key: string, defaultValue: T) {
  const [value, setValue] = useState<T>(defaultValue);
  const [loading, setLoading] = useState(true);
  
  const item = storage.defineItem<T>(key, { fallback: defaultValue });
  
  useEffect(() => {
    let mounted = true;
    
    // 加载初始值
    item.getValue().then((v) => {
      if (mounted) {
        setValue(v);
        setLoading(false);
      }
    });
    
    // 监听变化（其他地方修改时自动更新）
    const unwatch = item.watch((newValue) => {
      if (mounted) setValue(newValue);
    });
    
    return () => {
      mounted = false;
      unwatch();
    };
  }, [key]);
  
  // 更新方法，支持函数式更新
  const update = useCallback(async (newValue: T | ((prev: T) => T)) => {
    const resolvedValue = typeof newValue === 'function'
      ? (newValue as (prev: T) => T)(value)
      : newValue;
    
    await item.setValue(resolvedValue);
  }, [value, item]);
  
  return { value, loading, update };
}

// 使用示例
function MyComponent() {
  const { value: theme, update: setTheme } = useStorage('sync:theme', 'light');
  // theme 会自动同步所有修改
}
```

### 5.2 useMessage - 消息监听

```typescript
// hooks/useMessage.ts
import { useEffect, useCallback } from 'react';

type MessageHandler = (
  message: any,
  sender: Browser.runtime.MessageSender
) => void | Promise<any>;

export function useMessage(handler: MessageHandler) {
  useEffect(() => {
    const listener = (
      message: any,
      sender: Browser.runtime.MessageSender,
      sendResponse: (response?: any) => void
    ) => {
      const result = handler(message, sender);
      
      // 处理异步响应
      if (result instanceof Promise) {
        result.then(sendResponse);
        return true;  // 保持消息通道开放
      }
      
      if (result !== undefined) {
        sendResponse(result);
      }
    };
    
    browser.runtime.onMessage.addListener(listener);
    return () => browser.runtime.onMessage.removeListener(listener);
  }, [handler]);
}

// 发送消息的 Hook
export function useSendMessage() {
  return useCallback(async (message: any) => {
    return browser.runtime.sendMessage(message);
  }, []);
}

// 使用示例
function MyComponent() {
  useMessage((message, sender) => {
    if (message.type === 'PING') {
      return 'PONG';  // 自动作为响应返回
    }
  });
}
```

### 5.3 useActiveTab - 当前标签页

```typescript
// hooks/useActiveTab.ts
import { useState, useEffect } from 'react';

export function useActiveTab() {
  const [tab, setTab] = useState<Browser.tabs.Tab | null>(null);
  
  useEffect(() => {
    // 获取当前活动标签页
    browser.tabs.query({ active: true, currentWindow: true })
      .then(([activeTab]) => setTab(activeTab ?? null));
    
    // 监听标签页切换
    const handleActivated = (activeInfo: Browser.tabs.activeInfo) => {
      browser.tabs.get(activeInfo.tabId).then(setTab);
    };
    
    // 监听标签页更新（URL 变化等）
    const handleUpdated = (
      tabId: number,
      _changeInfo: Browser.tabs.TabChangeInfo,
      updatedTab: Browser.tabs.Tab
    ) => {
      if (tab?.id === tabId) {
        setTab(updatedTab);
      }
    };
    
    browser.tabs.onActivated.addListener(handleActivated);
    browser.tabs.onUpdated.addListener(handleUpdated);
    
    return () => {
      browser.tabs.onActivated.removeListener(handleActivated);
      browser.tabs.onUpdated.removeListener(handleUpdated);
    };
  }, [tab?.id]);
  
  return tab;
}

// 使用示例
function PopupApp() {
  const activeTab = useActiveTab();
  
  return (
    <div>
      <p>Current URL: {activeTab?.url}</p>
      <p>Title: {activeTab?.title}</p>
    </div>
  );
}
```

---

## 6. 状态管理

对于复杂的扩展，推荐使用状态管理库。

### 6.1 使用 Zustand

Zustand 是一个轻量级状态管理库，非常适合扩展开发：

```bash
pnpm add zustand
```

```typescript
// stores/settingsStore.ts
import { create } from 'zustand';
import { storage } from 'wxt/storage';

interface SettingsState {
  theme: 'light' | 'dark';
  language: string;
  setTheme: (theme: 'light' | 'dark') => void;
  setLanguage: (language: string) => void;
  load: () => Promise<void>;  // 初始化加载
}

// 定义持久化存储项
const settingsItem = storage.defineItem('sync:settings', {
  fallback: { theme: 'light', language: 'en' },
});

export const useSettingsStore = create<SettingsState>((set, get) => ({
  theme: 'light',
  language: 'en',
  
  setTheme: async (theme) => {
    set({ theme });
    // 同时更新存储
    const current = await settingsItem.getValue();
    await settingsItem.setValue({ ...current, theme });
  },
  
  setLanguage: async (language) => {
    set({ language });
    const current = await settingsItem.getValue();
    await settingsItem.setValue({ ...current, language });
  },
  
  load: async () => {
    // 从存储加载初始值
    const settings = await settingsItem.getValue();
    set(settings);
    
    // 监听外部变化（其他页面修改）
    settingsItem.watch((newValue) => {
      set(newValue);
    });
  },
}));

// 使用
function App() {
  const { theme, setTheme, load } = useSettingsStore();
  
  useEffect(() => {
    load();  // 初始化加载
  }, []);
  
  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      Toggle Theme
    </button>
  );
}
```

---

## 7. 组件库集成

### 7.1 Tailwind CSS

```bash
pnpm add -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

```javascript
// tailwind.config.js
export default {
  content: ['./entrypoints/**/*.{html,tsx,ts}'],
  theme: { extend: {} },
  plugins: [],
};
```

```css
/* assets/styles/global.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

### 7.2 Radix UI (无头组件)

Radix UI 提供无样式但功能完整的组件，非常适合自定义设计：

```bash
pnpm add @radix-ui/react-switch @radix-ui/react-select
```

```typescript
import * as Switch from '@radix-ui/react-switch';

function ToggleSwitch({ checked, onChange }: Props) {
  return (
    <Switch.Root
      checked={checked}
      onCheckedChange={onChange}
      className="w-11 h-6 bg-gray-200 rounded-full data-[state=checked]:bg-blue-500"
    >
      <Switch.Thumb className="block w-5 h-5 bg-white rounded-full transition-transform data-[state=checked]:translate-x-5" />
    </Switch.Root>
  );
}
```

---

## 8. 小结

| 场景 | 方案 |
|------|------|
| Popup/Options 页面 | 标准 React 应用，HTML 入口 + ReactDOM.createRoot |
| 内容脚本浮动 UI | createShadowRootUi + React，样式完全隔离 |
| 内容脚本内联 UI | createIntegratedUi + React，插入到页面元素旁 |
| 状态持久化 | wxt/storage + 自定义 useStorage Hook |
| 全局状态 | Zustand / Jotai + storage 同步 |
| 样式方案 | Tailwind CSS / CSS Modules |

---

> **下一步**: 学习 [构建与发布](./09-构建与发布.md)，了解如何将扩展打包并发布到各大浏览器商店。
