# WXT 实战练习 08: React 集成开发

## 练习目标

通过本练习,你将:
- 掌握 WXT + React 的集成方式
- 创建 React 组件库用于扩展
- 实现 Shadow DOM 中的 React UI
- 构建响应式扩展界面

## 练习 8.1: React 项目配置

### 任务

配置 WXT 项目使用 React。

### 步骤

1. 安装依赖:

```bash
pnpm add react react-dom
pnpm add -D @types/react @types/react-dom @wxt-dev/module-react
```

2. 配置 wxt.config.ts:

```typescript
// wxt.config.ts
import { defineConfig } from 'wxt';

export default defineConfig({
  modules: ['@wxt-dev/module-react'],
  manifest: {
    name: 'React Extension',
    description: 'A WXT extension with React',
  },
});
```

3. 创建 React 入口:

```tsx
// entrypoints/popup/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './styles.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

```html
<!-- entrypoints/popup/index.html -->
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=400">
  <title>Popup</title>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="./main.tsx"></script>
</body>
</html>
```

### 验证

- [ ] 项目正常启动
- [ ] Popup 显示 React 组件

---

## 练习 8.2: 组件库构建

### 任务

创建可复用的 React 组件库。

### 步骤

```tsx
// components/ui/Button.tsx
import { forwardRef, ButtonHTMLAttributes } from 'react';

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
  size?: 'sm' | 'md' | 'lg';
  loading?: boolean;
}

export const Button = forwardRef<HTMLButtonElement, ButtonProps>(
  ({ variant = 'primary', size = 'md', loading, children, disabled, ...props }, ref) => {
    const baseStyles = `
      font-family: inherit;
      border: none;
      border-radius: 6px;
      cursor: pointer;
      display: inline-flex;
      align-items: center;
      justify-content: center;
      gap: 8px;
      transition: all 0.2s;
    `;

    const variantStyles = {
      primary: 'background: #1976d2; color: white;',
      secondary: 'background: #e0e0e0; color: #333;',
      danger: 'background: #d32f2f; color: white;',
      ghost: 'background: transparent; color: #1976d2;',
    };

    const sizeStyles = {
      sm: 'padding: 6px 12px; font-size: 12px;',
      md: 'padding: 8px 16px; font-size: 14px;',
      lg: 'padding: 12px 24px; font-size: 16px;',
    };

    const disabledStyle = disabled || loading ? 'opacity: 0.6; cursor: not-allowed;' : '';

    return (
      <button
        ref={ref}
        disabled={disabled || loading}
        style={{
          ...parseStyles(baseStyles + variantStyles[variant] + sizeStyles[size] + disabledStyle),
        }}
        {...props}
      >
        {loading && <Spinner />}
        {children}
      </button>
    );
  }
);

function Spinner() {
  return (
    <svg width="16" height="16" viewBox="0 0 24 24" style={{ animation: 'spin 1s linear infinite' }}>
      <circle cx="12" cy="12" r="10" stroke="currentColor" strokeWidth="3" fill="none" opacity="0.3" />
      <path d="M12 2a10 10 0 0 1 10 10" stroke="currentColor" strokeWidth="3" fill="none" strokeLinecap="round" />
    </svg>
  );
}

function parseStyles(styleString: string): React.CSSProperties {
  const styles: Record<string, string> = {};
  styleString.split(';').forEach(rule => {
    const [key, value] = rule.split(':').map(s => s.trim());
    if (key && value) {
      const camelKey = key.replace(/-([a-z])/g, (_, c) => c.toUpperCase());
      styles[camelKey] = value;
    }
  });
  return styles as React.CSSProperties;
}
```

```tsx
// components/ui/Card.tsx
import { ReactNode } from 'react';

interface CardProps {
  title?: string;
  children: ReactNode;
  footer?: ReactNode;
  padding?: number;
}

export function Card({ title, children, footer, padding = 16 }: CardProps) {
  return (
    <div style={{
      background: 'white',
      borderRadius: 8,
      boxShadow: '0 2px 8px rgba(0,0,0,0.1)',
      overflow: 'hidden',
    }}>
      {title && (
        <div style={{
          padding: '12px 16px',
          borderBottom: '1px solid #eee',
          fontWeight: 600,
        }}>
          {title}
        </div>
      )}
      <div style={{ padding }}>
        {children}
      </div>
      {footer && (
        <div style={{
          padding: '12px 16px',
          borderTop: '1px solid #eee',
          background: '#fafafa',
        }}>
          {footer}
        </div>
      )}
    </div>
  );
}
```

```tsx
// components/ui/Input.tsx
import { forwardRef, InputHTMLAttributes } from 'react';

interface InputProps extends InputHTMLAttributes<HTMLInputElement> {
  label?: string;
  error?: string;
  fullWidth?: boolean;
}

export const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ label, error, fullWidth, style, ...props }, ref) => {
    return (
      <div style={{ width: fullWidth ? '100%' : 'auto' }}>
        {label && (
          <label style={{
            display: 'block',
            marginBottom: 4,
            fontSize: 14,
            fontWeight: 500,
          }}>
            {label}
          </label>
        )}
        <input
          ref={ref}
          style={{
            width: fullWidth ? '100%' : 'auto',
            padding: '8px 12px',
            border: `1px solid ${error ? '#d32f2f' : '#ddd'}`,
            borderRadius: 6,
            fontSize: 14,
            outline: 'none',
            transition: 'border-color 0.2s',
            boxSizing: 'border-box',
            ...style,
          }}
          {...props}
        />
        {error && (
          <span style={{ color: '#d32f2f', fontSize: 12, marginTop: 4, display: 'block' }}>
            {error}
          </span>
        )}
      </div>
    );
  }
);
```

```tsx
// components/ui/index.ts
export { Button } from './Button';
export { Card } from './Card';
export { Input } from './Input';
```

### 使用组件

```tsx
// entrypoints/popup/App.tsx
import { useState } from 'react';
import { Button, Card, Input } from '~/components/ui';

function App() {
  const [loading, setLoading] = useState(false);
  const [name, setName] = useState('');
  const [error, setError] = useState('');

  const handleSubmit = async () => {
    if (!name.trim()) {
      setError('请输入名称');
      return;
    }
    
    setError('');
    setLoading(true);
    await new Promise(r => setTimeout(r, 1000));
    setLoading(false);
    alert('提交成功: ' + name);
  };

  return (
    <div style={{ padding: 16, width: 320 }}>
      <Card title="快速操作">
        <Input
          label="名称"
          value={name}
          onChange={e => setName(e.target.value)}
          error={error}
          fullWidth
          placeholder="请输入..."
        />
        
        <div style={{ display: 'flex', gap: 8, marginTop: 16 }}>
          <Button onClick={handleSubmit} loading={loading}>
            提交
          </Button>
          <Button variant="secondary" onClick={() => setName('')}>
            清空
          </Button>
        </div>
      </Card>
    </div>
  );
}

export default App;
```

### 验证

- [ ] 组件正确渲染
- [ ] Button loading 状态正常
- [ ] Input 验证功能正常

---

## 练习 8.3: 自定义 Hooks

### 任务

创建扩展开发常用的 React Hooks。

### 步骤

```tsx
// hooks/useCurrentTab.ts
import { useState, useEffect } from 'react';

interface TabInfo {
  id?: number;
  url?: string;
  title?: string;
  favIconUrl?: string;
}

export function useCurrentTab() {
  const [tab, setTab] = useState<TabInfo | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    browser.tabs.query({ active: true, currentWindow: true })
      .then(([currentTab]) => {
        setTab(currentTab || null);
        setLoading(false);
      });

    // 监听标签页变化
    const handleActivated = async () => {
      const [currentTab] = await browser.tabs.query({ 
        active: true, 
        currentWindow: true 
      });
      setTab(currentTab || null);
    };

    browser.tabs.onActivated.addListener(handleActivated);
    browser.tabs.onUpdated.addListener(handleActivated);

    return () => {
      browser.tabs.onActivated.removeListener(handleActivated);
      browser.tabs.onUpdated.removeListener(handleActivated);
    };
  }, []);

  return { tab, loading };
}
```

```tsx
// hooks/useStorage.ts
import { useState, useEffect, useCallback } from 'react';
import { storage } from 'wxt/storage';

export function useStorage<T>(key: string, defaultValue: T) {
  const [value, setValue] = useState<T>(defaultValue);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // 加载初始值
    storage.getItem<T>(key).then(stored => {
      if (stored !== null) {
        setValue(stored);
      }
      setLoading(false);
    });

    // 监听变化
    const unwatch = storage.watch<T>(key, (newValue) => {
      if (newValue !== null) {
        setValue(newValue);
      }
    });

    return unwatch;
  }, [key]);

  const setStoredValue = useCallback(async (newValue: T | ((prev: T) => T)) => {
    const resolved = typeof newValue === 'function' 
      ? (newValue as (prev: T) => T)(value) 
      : newValue;
    await storage.setItem(key, resolved);
    setValue(resolved);
  }, [key, value]);

  return [value, setStoredValue, loading] as const;
}
```

```tsx
// hooks/useMessage.ts
import { useEffect, useCallback } from 'react';

type MessageHandler = (message: any, sender: browser.Runtime.MessageSender) => any;

export function useMessageListener(handler: MessageHandler) {
  useEffect(() => {
    browser.runtime.onMessage.addListener(handler);
    return () => {
      browser.runtime.onMessage.removeListener(handler);
    };
  }, [handler]);
}

export function useSendMessage() {
  return useCallback(async <T = any>(message: any): Promise<T> => {
    return browser.runtime.sendMessage(message);
  }, []);
}

export function useSendToTab() {
  return useCallback(async <T = any>(tabId: number, message: any): Promise<T> => {
    return browser.tabs.sendMessage(tabId, message);
  }, []);
}
```

```tsx
// hooks/usePermission.ts
import { useState, useEffect, useCallback } from 'react';

export function usePermission(permission: string) {
  const [granted, setGranted] = useState<boolean | null>(null);

  useEffect(() => {
    browser.permissions.contains({ permissions: [permission] })
      .then(setGranted);
  }, [permission]);

  const request = useCallback(async () => {
    const result = await browser.permissions.request({ 
      permissions: [permission] 
    });
    setGranted(result);
    return result;
  }, [permission]);

  const remove = useCallback(async () => {
    const result = await browser.permissions.remove({ 
      permissions: [permission] 
    });
    if (result) {
      setGranted(false);
    }
    return result;
  }, [permission]);

  return { granted, request, remove };
}
```

### 使用示例

```tsx
// entrypoints/popup/App.tsx
import { useCurrentTab, useStorage, useSendMessage, usePermission } from '~/hooks';
import { Button, Card } from '~/components/ui';

function App() {
  const { tab, loading: tabLoading } = useCurrentTab();
  const [count, setCount, storageLoading] = useStorage('local:count', 0);
  const sendMessage = useSendMessage();
  const { granted, request } = usePermission('notifications');

  const handleAnalyze = async () => {
    if (tab?.id) {
      const result = await sendMessage({ type: 'ANALYZE_PAGE', tabId: tab.id });
      console.log('Analysis result:', result);
    }
  };

  if (tabLoading || storageLoading) {
    return <div>Loading...</div>;
  }

  return (
    <div style={{ padding: 16, width: 350 }}>
      <Card title="当前页面">
        <p><strong>标题:</strong> {tab?.title || 'N/A'}</p>
        <p style={{ fontSize: 12, wordBreak: 'break-all' }}>
          <strong>URL:</strong> {tab?.url || 'N/A'}
        </p>
        <Button onClick={handleAnalyze} size="sm" style={{ marginTop: 8 }}>
          分析页面
        </Button>
      </Card>

      <Card title="计数器" style={{ marginTop: 16 }}>
        <p>当前值: {count}</p>
        <div style={{ display: 'flex', gap: 8 }}>
          <Button onClick={() => setCount(c => c + 1)} size="sm">+1</Button>
          <Button onClick={() => setCount(0)} variant="secondary" size="sm">重置</Button>
        </div>
      </Card>

      <Card title="权限" style={{ marginTop: 16 }}>
        <p>通知权限: {granted ? '已授权' : '未授权'}</p>
        {!granted && (
          <Button onClick={request} size="sm">请求权限</Button>
        )}
      </Card>
    </div>
  );
}
```

### 验证

- [ ] Hooks 正常工作
- [ ] 数据响应式更新
- [ ] 权限请求功能正常

---

## 练习 8.4: Shadow DOM React UI

### 任务

在 Content Script 中使用 Shadow DOM 渲染 React 组件。

### 步骤

```tsx
// entrypoints/widget.content.tsx
import { createShadowRootUi } from 'wxt/client';
import ReactDOM from 'react-dom/client';
import { useState } from 'react';

export default defineContentScript({
  matches: ['*://*/*'],
  
  async main(ctx) {
    const ui = await createShadowRootUi(ctx, {
      name: 'floating-widget',
      position: 'inline',
      anchor: 'body',
      append: 'first',
      
      onMount(container) {
        // 注入样式
        const style = document.createElement('style');
        style.textContent = getWidgetStyles();
        container.appendChild(style);

        // 创建 React 根
        const appContainer = document.createElement('div');
        container.appendChild(appContainer);
        
        const root = ReactDOM.createRoot(appContainer);
        root.render(<FloatingWidget onClose={() => ui.remove()} />);
        
        return root;
      },
      
      onRemove(root) {
        root?.unmount();
      },
    });

    // 延迟显示
    setTimeout(() => ui.mount(), 1000);
  },
});

function FloatingWidget({ onClose }: { onClose: () => void }) {
  const [isExpanded, setIsExpanded] = useState(false);
  const [notes, setNotes] = useState('');

  return (
    <div className="widget-container">
      <button 
        className="widget-toggle"
        onClick={() => setIsExpanded(!isExpanded)}
      >
        {isExpanded ? 'x' : '+'}
      </button>

      {isExpanded && (
        <div className="widget-panel">
          <div className="widget-header">
            <span>页面笔记</span>
            <button onClick={onClose}>关闭</button>
          </div>
          <div className="widget-body">
            <textarea
              value={notes}
              onChange={e => setNotes(e.target.value)}
              placeholder="记录你的想法..."
            />
            <div className="widget-info">
              <small>当前页面: {document.title}</small>
            </div>
          </div>
          <div className="widget-footer">
            <button 
              className="save-btn"
              onClick={() => {
                // 保存笔记
                browser.storage.local.set({
                  [`note:${location.href}`]: {
                    content: notes,
                    savedAt: Date.now(),
                  },
                });
                alert('已保存!');
              }}
            >
              保存
            </button>
          </div>
        </div>
      )}
    </div>
  );
}

function getWidgetStyles() {
  return `
    .widget-container {
      position: fixed;
      bottom: 20px;
      right: 20px;
      z-index: 999999;
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    }

    .widget-toggle {
      width: 48px;
      height: 48px;
      border-radius: 50%;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      border: none;
      color: white;
      font-size: 24px;
      cursor: pointer;
      box-shadow: 0 4px 12px rgba(0,0,0,0.3);
      transition: transform 0.2s;
    }

    .widget-toggle:hover {
      transform: scale(1.1);
    }

    .widget-panel {
      position: absolute;
      bottom: 60px;
      right: 0;
      width: 300px;
      background: white;
      border-radius: 12px;
      box-shadow: 0 8px 32px rgba(0,0,0,0.2);
      overflow: hidden;
    }

    .widget-header {
      padding: 12px 16px;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      color: white;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }

    .widget-header button {
      background: none;
      border: none;
      color: white;
      cursor: pointer;
      font-size: 12px;
    }

    .widget-body {
      padding: 16px;
    }

    .widget-body textarea {
      width: 100%;
      height: 120px;
      border: 1px solid #ddd;
      border-radius: 8px;
      padding: 12px;
      resize: none;
      font-family: inherit;
      box-sizing: border-box;
    }

    .widget-info {
      margin-top: 8px;
      color: #666;
    }

    .widget-footer {
      padding: 12px 16px;
      border-top: 1px solid #eee;
      text-align: right;
    }

    .save-btn {
      padding: 8px 16px;
      background: #667eea;
      color: white;
      border: none;
      border-radius: 6px;
      cursor: pointer;
    }

    .save-btn:hover {
      background: #5a67d8;
    }
  `;
}
```

### 验证

- [ ] 页面右下角出现浮动按钮
- [ ] 点击展开笔记面板
- [ ] 样式不影响原页面
- [ ] 笔记可以保存

---

## 练习 8.5: 状态管理

### 任务

实现扩展级别的状态管理。

### 步骤

```tsx
// store/context.tsx
import { createContext, useContext, useReducer, ReactNode, useEffect } from 'react';
import { storage } from 'wxt/storage';

// 状态类型
interface AppState {
  theme: 'light' | 'dark';
  count: number;
  favorites: string[];
  isLoading: boolean;
}

// Action 类型
type Action =
  | { type: 'SET_THEME'; payload: 'light' | 'dark' }
  | { type: 'INCREMENT' }
  | { type: 'DECREMENT' }
  | { type: 'ADD_FAVORITE'; payload: string }
  | { type: 'REMOVE_FAVORITE'; payload: string }
  | { type: 'SET_LOADING'; payload: boolean }
  | { type: 'LOAD_STATE'; payload: Partial<AppState> };

const initialState: AppState = {
  theme: 'light',
  count: 0,
  favorites: [],
  isLoading: true,
};

// Reducer
function reducer(state: AppState, action: Action): AppState {
  switch (action.type) {
    case 'SET_THEME':
      return { ...state, theme: action.payload };
    case 'INCREMENT':
      return { ...state, count: state.count + 1 };
    case 'DECREMENT':
      return { ...state, count: state.count - 1 };
    case 'ADD_FAVORITE':
      return { ...state, favorites: [...state.favorites, action.payload] };
    case 'REMOVE_FAVORITE':
      return { ...state, favorites: state.favorites.filter(f => f !== action.payload) };
    case 'SET_LOADING':
      return { ...state, isLoading: action.payload };
    case 'LOAD_STATE':
      return { ...state, ...action.payload, isLoading: false };
    default:
      return state;
  }
}

// Context
const AppContext = createContext<{
  state: AppState;
  dispatch: React.Dispatch<Action>;
} | null>(null);

// Provider
export function AppProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(reducer, initialState);

  // 加载持久化状态
  useEffect(() => {
    storage.getItem<Partial<AppState>>('local:appState').then(saved => {
      if (saved) {
        dispatch({ type: 'LOAD_STATE', payload: saved });
      } else {
        dispatch({ type: 'SET_LOADING', payload: false });
      }
    });
  }, []);

  // 保存状态变化
  useEffect(() => {
    if (!state.isLoading) {
      const { isLoading, ...toSave } = state;
      storage.setItem('local:appState', toSave);
    }
  }, [state]);

  return (
    <AppContext.Provider value={{ state, dispatch }}>
      {children}
    </AppContext.Provider>
  );
}

// Hook
export function useAppState() {
  const context = useContext(AppContext);
  if (!context) {
    throw new Error('useAppState must be used within AppProvider');
  }
  return context;
}

// Action creators
export const actions = {
  setTheme: (theme: 'light' | 'dark'): Action => ({ type: 'SET_THEME', payload: theme }),
  increment: (): Action => ({ type: 'INCREMENT' }),
  decrement: (): Action => ({ type: 'DECREMENT' }),
  addFavorite: (url: string): Action => ({ type: 'ADD_FAVORITE', payload: url }),
  removeFavorite: (url: string): Action => ({ type: 'REMOVE_FAVORITE', payload: url }),
};
```

使用:

```tsx
// entrypoints/popup/main.tsx
import { AppProvider } from '~/store/context';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <AppProvider>
      <App />
    </AppProvider>
  </React.StrictMode>
);
```

```tsx
// entrypoints/popup/App.tsx
import { useAppState, actions } from '~/store/context';

function App() {
  const { state, dispatch } = useAppState();

  if (state.isLoading) {
    return <div>Loading...</div>;
  }

  return (
    <div style={{ 
      padding: 20, 
      background: state.theme === 'dark' ? '#333' : '#fff',
      color: state.theme === 'dark' ? '#fff' : '#333',
      minWidth: 300,
    }}>
      <h2>状态管理演示</h2>

      <div style={{ marginBottom: 16 }}>
        <p>计数: {state.count}</p>
        <button onClick={() => dispatch(actions.decrement())}>-</button>
        <button onClick={() => dispatch(actions.increment())}>+</button>
      </div>

      <div style={{ marginBottom: 16 }}>
        <p>主题: {state.theme}</p>
        <button onClick={() => dispatch(actions.setTheme(
          state.theme === 'light' ? 'dark' : 'light'
        ))}>
          切换主题
        </button>
      </div>

      <div>
        <p>收藏 ({state.favorites.length}):</p>
        <ul>
          {state.favorites.map(url => (
            <li key={url}>
              {url}
              <button onClick={() => dispatch(actions.removeFavorite(url))}>
                删除
              </button>
            </li>
          ))}
        </ul>
      </div>
    </div>
  );
}
```

### 验证

- [ ] 状态正确更新
- [ ] 刷新后状态保持
- [ ] 主题切换生效

---

## 挑战练习

### 挑战: 创建完整的 UI 框架

整合所有组件和 Hooks,创建一个迷你 UI 框架:

```tsx
// lib/extension-ui/index.ts
export * from './components';
export * from './hooks';
export * from './context';
export * from './utils';
```

包含:
- 10+ 基础组件 (Button, Input, Card, Modal, Toast, Tabs, Dropdown...)
- 5+ 自定义 Hooks (useStorage, useCurrentTab, useMessage...)
- 状态管理方案
- 主题系统

---

## 自我检查

1. 如何在 WXT 中配置 React?
2. Shadow DOM 中如何渲染 React 组件?
3. 如何创建响应式的存储 Hook?
4. 扩展组件库应该注意什么?
5. 如何处理 Content Script 中的 CSS 隔离?

## 参考答案

1. 安装 `@wxt-dev/module-react` 并在 wxt.config.ts 中配置 modules
2. 使用 `createShadowRootUi` + `ReactDOM.createRoot`
3. 结合 `useState`, `useEffect` 和 `storage.watch`
4. 使用内联样式或 CSS-in-JS 避免样式冲突,组件要考虑复用性
5. 使用 Shadow DOM 或 CSS Modules,避免全局样式
