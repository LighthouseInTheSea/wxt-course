# WXT 实战练习 07: Storage 存储系统

## 练习目标

通过本练习,你将:
- 掌握 WXT Storage API 的使用
- 理解不同存储区域的特点
- 实现响应式数据存储
- 构建设置管理系统

## 练习 7.1: 基础存储操作

### 任务

实现基本的数据存储和读取。

### 步骤

```typescript
// utils/storage.ts
import { storage } from 'wxt/storage';

// 基础操作示例
export async function storageBasics() {
  // 设置值 - 注意键名格式: 'area:key'
  await storage.setItem('local:username', 'John');
  await storage.setItem('local:settings', { theme: 'dark', lang: 'zh' });
  await storage.setItem('local:count', 42);

  // 获取值
  const username = await storage.getItem<string>('local:username');
  const settings = await storage.getItem<{ theme: string; lang: string }>('local:settings');
  const count = await storage.getItem<number>('local:count');

  console.log({ username, settings, count });

  // 删除值
  await storage.removeItem('local:count');

  // 检查是否存在
  const hasUsername = await storage.getItem('local:username') !== null;

  // 获取所有键
  const allKeys = await storage.snapshot('local');
  console.log('All local storage:', allKeys);
}
```

### 存储区域对比

| 区域 | 前缀 | 同步 | 容量 | 用途 |
|------|------|------|------|------|
| local | `local:` | 否 | 10MB | 本地大数据 |
| sync | `sync:` | 是 | 100KB | 跨设备同步 |
| session | `session:` | 否 | 10MB | 会话临时数据 |
| managed | `managed:` | - | - | 企业策略 |

### 验证

```typescript
// 测试代码
async function testStorage() {
  // 测试 local
  await storage.setItem('local:test', 'local value');
  console.log('Local:', await storage.getItem('local:test'));

  // 测试 sync
  await storage.setItem('sync:test', 'sync value');
  console.log('Sync:', await storage.getItem('sync:test'));

  // 测试 session (MV3 only)
  try {
    await storage.setItem('session:test', 'session value');
    console.log('Session:', await storage.getItem('session:test'));
  } catch (e) {
    console.log('Session storage not available');
  }
}
```

---

## 练习 7.2: 类型化存储项

### 任务

使用 `defineItem` 创建类型安全的存储项。

### 步骤

```typescript
// utils/storageItems.ts
import { storage } from 'wxt/storage';

// 定义用户设置
export const userSettings = storage.defineItem<{
  theme: 'light' | 'dark' | 'system';
  fontSize: number;
  language: string;
  notifications: boolean;
}>('local:userSettings', {
  fallback: {
    theme: 'system',
    fontSize: 16,
    language: 'zh-CN',
    notifications: true,
  },
});

// 定义计数器
export const visitCount = storage.defineItem<number>('local:visitCount', {
  fallback: 0,
});

// 定义收藏列表
export const favorites = storage.defineItem<Array<{
  id: string;
  title: string;
  url: string;
  addedAt: number;
}>>('sync:favorites', {
  fallback: [],
});

// 定义最近浏览
export const recentPages = storage.defineItem<Array<{
  url: string;
  title: string;
  visitedAt: number;
}>>('local:recentPages', {
  fallback: [],
});

// 定义功能开关
export const featureFlags = storage.defineItem<{
  enableBeta: boolean;
  debugMode: boolean;
  experimentalUI: boolean;
}>('sync:featureFlags', {
  fallback: {
    enableBeta: false,
    debugMode: false,
    experimentalUI: false,
  },
});
```

使用示例:

```tsx
// entrypoints/popup/App.tsx
import { useState, useEffect } from 'react';
import { userSettings, visitCount, favorites } from '~/utils/storageItems';

function App() {
  const [settings, setSettings] = useState(userSettings.fallback);
  const [count, setCount] = useState(0);

  useEffect(() => {
    // 加载设置
    userSettings.getValue().then(setSettings);
    
    // 增加访问计数
    visitCount.getValue().then(async (c) => {
      const newCount = c + 1;
      await visitCount.setValue(newCount);
      setCount(newCount);
    });
  }, []);

  const updateTheme = async (theme: 'light' | 'dark' | 'system') => {
    const newSettings = { ...settings, theme };
    await userSettings.setValue(newSettings);
    setSettings(newSettings);
  };

  const addFavorite = async (title: string, url: string) => {
    const current = await favorites.getValue();
    await favorites.setValue([
      ...current,
      { id: Date.now().toString(), title, url, addedAt: Date.now() },
    ]);
  };

  return (
    <div style={{ padding: 20 }}>
      <p>访问次数: {count}</p>
      
      <div>
        <label>主题:</label>
        <select 
          value={settings.theme}
          onChange={e => updateTheme(e.target.value as any)}
        >
          <option value="light">浅色</option>
          <option value="dark">深色</option>
          <option value="system">跟随系统</option>
        </select>
      </div>
    </div>
  );
}
```

### 验证

- [ ] 设置被正确保存
- [ ] 刷新后设置保持
- [ ] 类型检查正常工作

---

## 练习 7.3: 存储监听

### 任务

监听存储变化并响应。

### 步骤

```typescript
// utils/storageItems.ts
import { storage } from 'wxt/storage';

export const counter = storage.defineItem<number>('local:counter', {
  fallback: 0,
});

// 在后台脚本中监听
// entrypoints/background.ts
export default defineBackground(() => {
  // 监听特定项
  counter.watch((newValue, oldValue) => {
    console.log(`Counter changed: ${oldValue} -> ${newValue}`);
    
    // 可以在这里执行副作用
    if (newValue !== null && newValue >= 100) {
      browser.notifications.create({
        type: 'basic',
        iconUrl: browser.runtime.getURL('/icon/128.png'),
        title: '里程碑达成!',
        message: '计数器已达到 100!',
      });
    }
  });

  // 监听所有存储变化
  storage.watch<number>('local:counter', (newValue, oldValue) => {
    console.log('Storage changed via watch');
  });
});
```

React Hook 封装:

```tsx
// hooks/useStorageItem.ts
import { useState, useEffect } from 'react';
import type { WxtStorageItem } from 'wxt/storage';

export function useStorageItem<T>(item: WxtStorageItem<T, Record<string, unknown>>) {
  const [value, setValue] = useState<T>(item.fallback);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // 初始加载
    item.getValue().then((v) => {
      setValue(v);
      setLoading(false);
    });

    // 监听变化
    const unwatch = item.watch((newValue) => {
      if (newValue !== null) {
        setValue(newValue);
      }
    });

    return unwatch;
  }, [item]);

  const update = async (newValue: T | ((prev: T) => T)) => {
    const resolved = typeof newValue === 'function' 
      ? (newValue as (prev: T) => T)(value)
      : newValue;
    await item.setValue(resolved);
    setValue(resolved);
  };

  return [value, update, loading] as const;
}
```

使用 Hook:

```tsx
// entrypoints/popup/App.tsx
import { useStorageItem } from '~/hooks/useStorageItem';
import { counter, userSettings } from '~/utils/storageItems';

function App() {
  const [count, setCount, loading] = useStorageItem(counter);
  const [settings, setSettings] = useStorageItem(userSettings);

  if (loading) {
    return <div>Loading...</div>;
  }

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>+1</button>
      <button onClick={() => setCount(0)}>Reset</button>

      <hr />

      <p>Theme: {settings.theme}</p>
      <button onClick={() => setSettings({ ...settings, theme: 'dark' })}>
        Dark Mode
      </button>
    </div>
  );
}
```

### 验证

- [ ] 在一个页面修改值,其他页面自动更新
- [ ] Background 收到变化通知

---

## 练习 7.4: 存储元数据

### 任务

使用元数据记录版本和迁移信息。

### 步骤

```typescript
// utils/storageItems.ts
import { storage } from 'wxt/storage';

export const appData = storage.defineItem<{
  bookmarks: Array<{ id: string; url: string; title: string }>;
  tags: string[];
}>('local:appData', {
  fallback: { bookmarks: [], tags: [] },
});

// 获取元数据
async function checkDataVersion() {
  const meta = await appData.getMeta();
  console.log('Data version:', meta?.v);
  console.log('Last modified:', meta?.lastModified);
}

// 设置元数据
async function updateWithMeta() {
  await appData.setValue({ bookmarks: [], tags: ['work', 'personal'] });
  await appData.setMeta({
    v: 2,
    lastModified: Date.now(),
    migratedFrom: 1,
  });
}

// 数据迁移示例
async function migrateData() {
  const meta = await appData.getMeta();
  const currentVersion = meta?.v || 1;

  if (currentVersion < 2) {
    // 执行 v1 -> v2 迁移
    const data = await appData.getValue();
    
    // 迁移逻辑...
    const migratedData = {
      ...data,
      tags: data.tags || [], // 添加新字段
    };

    await appData.setValue(migratedData);
    await appData.setMeta({ ...meta, v: 2 });
    
    console.log('Migrated to v2');
  }
}
```

在扩展启动时运行迁移:

```typescript
// entrypoints/background.ts
export default defineBackground(() => {
  browser.runtime.onInstalled.addListener(async (details) => {
    if (details.reason === 'update') {
      await migrateData();
    }
  });
});
```

### 验证

- [ ] 元数据正确保存
- [ ] 版本更新时迁移执行

---

## 练习 7.5: 实现设置页面

### 任务

创建完整的设置管理页面。

### 步骤

```tsx
// entrypoints/options/App.tsx
import { useState, useEffect } from 'react';
import { storage } from 'wxt/storage';

// 定义设置结构
interface AppSettings {
  general: {
    language: string;
    autoStart: boolean;
  };
  appearance: {
    theme: 'light' | 'dark' | 'system';
    fontSize: 'small' | 'medium' | 'large';
    compactMode: boolean;
  };
  privacy: {
    collectAnalytics: boolean;
    saveHistory: boolean;
    historyDays: number;
  };
  advanced: {
    debugMode: boolean;
    experimentalFeatures: boolean;
    customCss: string;
  };
}

const defaultSettings: AppSettings = {
  general: { language: 'zh-CN', autoStart: true },
  appearance: { theme: 'system', fontSize: 'medium', compactMode: false },
  privacy: { collectAnalytics: false, saveHistory: true, historyDays: 30 },
  advanced: { debugMode: false, experimentalFeatures: false, customCss: '' },
};

const settingsItem = storage.defineItem<AppSettings>('sync:appSettings', {
  fallback: defaultSettings,
});

function App() {
  const [settings, setSettings] = useState<AppSettings>(defaultSettings);
  const [activeTab, setActiveTab] = useState<keyof AppSettings>('general');
  const [saved, setSaved] = useState(false);
  const [hasChanges, setHasChanges] = useState(false);

  useEffect(() => {
    settingsItem.getValue().then(setSettings);
  }, []);

  const updateSettings = <K extends keyof AppSettings>(
    category: K,
    updates: Partial<AppSettings[K]>
  ) => {
    setSettings(prev => ({
      ...prev,
      [category]: { ...prev[category], ...updates },
    }));
    setHasChanges(true);
  };

  const saveSettings = async () => {
    await settingsItem.setValue(settings);
    setSaved(true);
    setHasChanges(false);
    setTimeout(() => setSaved(false), 2000);
  };

  const resetSettings = async () => {
    if (confirm('确定要重置所有设置吗?')) {
      await settingsItem.setValue(defaultSettings);
      setSettings(defaultSettings);
      setHasChanges(false);
    }
  };

  const exportSettings = () => {
    const blob = new Blob([JSON.stringify(settings, null, 2)], {
      type: 'application/json',
    });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'extension-settings.json';
    a.click();
    URL.revokeObjectURL(url);
  };

  const importSettings = (event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0];
    if (!file) return;

    const reader = new FileReader();
    reader.onload = (e) => {
      try {
        const imported = JSON.parse(e.target?.result as string);
        setSettings({ ...defaultSettings, ...imported });
        setHasChanges(true);
      } catch {
        alert('无效的设置文件');
      }
    };
    reader.readAsText(file);
  };

  const tabs: { key: keyof AppSettings; label: string }[] = [
    { key: 'general', label: '常规' },
    { key: 'appearance', label: '外观' },
    { key: 'privacy', label: '隐私' },
    { key: 'advanced', label: '高级' },
  ];

  return (
    <div style={{ maxWidth: 800, margin: '0 auto', padding: 24 }}>
      <h1>扩展设置</h1>

      {/* 标签页导航 */}
      <div style={{ display: 'flex', gap: 8, marginBottom: 24 }}>
        {tabs.map(tab => (
          <button
            key={tab.key}
            onClick={() => setActiveTab(tab.key)}
            style={{
              padding: '8px 16px',
              background: activeTab === tab.key ? '#1976d2' : '#e0e0e0',
              color: activeTab === tab.key ? 'white' : 'black',
              border: 'none',
              borderRadius: 4,
              cursor: 'pointer',
            }}
          >
            {tab.label}
          </button>
        ))}
      </div>

      {/* 常规设置 */}
      {activeTab === 'general' && (
        <div>
          <h3>常规设置</h3>
          <label style={{ display: 'block', marginBottom: 16 }}>
            语言:
            <select
              value={settings.general.language}
              onChange={e => updateSettings('general', { language: e.target.value })}
              style={{ marginLeft: 8 }}
            >
              <option value="zh-CN">简体中文</option>
              <option value="en-US">English</option>
              <option value="ja-JP">日本語</option>
            </select>
          </label>
          <label style={{ display: 'block' }}>
            <input
              type="checkbox"
              checked={settings.general.autoStart}
              onChange={e => updateSettings('general', { autoStart: e.target.checked })}
            />
            随浏览器启动
          </label>
        </div>
      )}

      {/* 外观设置 */}
      {activeTab === 'appearance' && (
        <div>
          <h3>外观设置</h3>
          <label style={{ display: 'block', marginBottom: 16 }}>
            主题:
            <select
              value={settings.appearance.theme}
              onChange={e => updateSettings('appearance', { 
                theme: e.target.value as any 
              })}
              style={{ marginLeft: 8 }}
            >
              <option value="light">浅色</option>
              <option value="dark">深色</option>
              <option value="system">跟随系统</option>
            </select>
          </label>
          <label style={{ display: 'block', marginBottom: 16 }}>
            字体大小:
            <select
              value={settings.appearance.fontSize}
              onChange={e => updateSettings('appearance', { 
                fontSize: e.target.value as any 
              })}
              style={{ marginLeft: 8 }}
            >
              <option value="small">小</option>
              <option value="medium">中</option>
              <option value="large">大</option>
            </select>
          </label>
          <label style={{ display: 'block' }}>
            <input
              type="checkbox"
              checked={settings.appearance.compactMode}
              onChange={e => updateSettings('appearance', { 
                compactMode: e.target.checked 
              })}
            />
            紧凑模式
          </label>
        </div>
      )}

      {/* 隐私设置 */}
      {activeTab === 'privacy' && (
        <div>
          <h3>隐私设置</h3>
          <label style={{ display: 'block', marginBottom: 16 }}>
            <input
              type="checkbox"
              checked={settings.privacy.collectAnalytics}
              onChange={e => updateSettings('privacy', { 
                collectAnalytics: e.target.checked 
              })}
            />
            允许收集匿名使用数据
          </label>
          <label style={{ display: 'block', marginBottom: 16 }}>
            <input
              type="checkbox"
              checked={settings.privacy.saveHistory}
              onChange={e => updateSettings('privacy', { 
                saveHistory: e.target.checked 
              })}
            />
            保存浏览历史
          </label>
          {settings.privacy.saveHistory && (
            <label style={{ display: 'block' }}>
              历史保留天数:
              <input
                type="number"
                value={settings.privacy.historyDays}
                onChange={e => updateSettings('privacy', { 
                  historyDays: parseInt(e.target.value) || 30 
                })}
                min={1}
                max={365}
                style={{ marginLeft: 8, width: 80 }}
              />
            </label>
          )}
        </div>
      )}

      {/* 高级设置 */}
      {activeTab === 'advanced' && (
        <div>
          <h3>高级设置</h3>
          <label style={{ display: 'block', marginBottom: 16 }}>
            <input
              type="checkbox"
              checked={settings.advanced.debugMode}
              onChange={e => updateSettings('advanced', { 
                debugMode: e.target.checked 
              })}
            />
            调试模式
          </label>
          <label style={{ display: 'block', marginBottom: 16 }}>
            <input
              type="checkbox"
              checked={settings.advanced.experimentalFeatures}
              onChange={e => updateSettings('advanced', { 
                experimentalFeatures: e.target.checked 
              })}
            />
            启用实验性功能
          </label>
          <label style={{ display: 'block' }}>
            自定义 CSS:
            <textarea
              value={settings.advanced.customCss}
              onChange={e => updateSettings('advanced', { 
                customCss: e.target.value 
              })}
              style={{ 
                display: 'block', 
                width: '100%', 
                height: 120,
                marginTop: 8,
                fontFamily: 'monospace',
              }}
              placeholder="/* 自定义样式 */"
            />
          </label>
        </div>
      )}

      {/* 操作按钮 */}
      <div style={{ 
        marginTop: 32, 
        paddingTop: 16, 
        borderTop: '1px solid #e0e0e0',
        display: 'flex',
        gap: 8,
        alignItems: 'center',
      }}>
        <button
          onClick={saveSettings}
          disabled={!hasChanges}
          style={{
            padding: '10px 20px',
            background: hasChanges ? '#1976d2' : '#ccc',
            color: 'white',
            border: 'none',
            borderRadius: 4,
            cursor: hasChanges ? 'pointer' : 'not-allowed',
          }}
        >
          保存设置
        </button>
        <button onClick={resetSettings} style={{ padding: '10px 20px' }}>
          重置默认
        </button>
        <button onClick={exportSettings} style={{ padding: '10px 20px' }}>
          导出
        </button>
        <label style={{ padding: '10px 20px', cursor: 'pointer' }}>
          导入
          <input
            type="file"
            accept=".json"
            onChange={importSettings}
            style={{ display: 'none' }}
          />
        </label>
        {saved && <span style={{ color: 'green' }}>已保存!</span>}
      </div>
    </div>
  );
}

export default App;
```

### 验证

- [ ] 设置页面正常显示
- [ ] 切换标签页显示不同设置
- [ ] 保存、导出、导入功能正常

---

## 挑战练习

### 挑战: 实现历史记录管理

创建一个带有限制和清理功能的历史记录系统:

```typescript
// utils/history.ts
import { storage } from 'wxt/storage';

interface HistoryEntry {
  id: string;
  url: string;
  title: string;
  visitedAt: number;
  visitCount: number;
}

const MAX_ENTRIES = 1000;
const MAX_AGE_DAYS = 30;

export const browsingHistory = storage.defineItem<HistoryEntry[]>(
  'local:browsingHistory',
  { fallback: [] }
);

export async function addHistoryEntry(url: string, title: string) {
  const entries = await browsingHistory.getValue();
  const existingIndex = entries.findIndex(e => e.url === url);

  if (existingIndex >= 0) {
    // 更新已有条目
    entries[existingIndex].visitCount++;
    entries[existingIndex].visitedAt = Date.now();
    entries[existingIndex].title = title;
    // 移到开头
    const [entry] = entries.splice(existingIndex, 1);
    entries.unshift(entry);
  } else {
    // 添加新条目
    entries.unshift({
      id: Date.now().toString(),
      url,
      title,
      visitedAt: Date.now(),
      visitCount: 1,
    });
  }

  // 限制数量
  if (entries.length > MAX_ENTRIES) {
    entries.length = MAX_ENTRIES;
  }

  await browsingHistory.setValue(entries);
}

export async function cleanupHistory() {
  const entries = await browsingHistory.getValue();
  const cutoff = Date.now() - MAX_AGE_DAYS * 24 * 60 * 60 * 1000;
  
  const filtered = entries.filter(e => e.visitedAt > cutoff);
  
  if (filtered.length !== entries.length) {
    await browsingHistory.setValue(filtered);
    return entries.length - filtered.length;
  }
  
  return 0;
}

export async function searchHistory(query: string) {
  const entries = await browsingHistory.getValue();
  const lowerQuery = query.toLowerCase();
  
  return entries.filter(e => 
    e.title.toLowerCase().includes(lowerQuery) ||
    e.url.toLowerCase().includes(lowerQuery)
  );
}
```

---

## 自我检查

1. storage 键名的格式是什么?
2. sync 和 local 存储有什么区别?
3. 如何监听存储变化?
4. defineItem 的 fallback 有什么作用?
5. 如何获取存储项的元数据?

## 参考答案

1. `area:key` 格式,如 `local:settings`, `sync:config`
2. sync 会在登录的设备间同步 (100KB限制),local 只保存在本地 (10MB限制)
3. 使用 `item.watch()` 或 `storage.watch('key', callback)`
4. 当存储项不存在时返回的默认值
5. 使用 `item.getMeta()` 或 `storage.getMeta('key')`
