# Storage 存储系统

> **导航**: [上一篇: 消息通信机制](./06-消息通信机制.md) | [下一篇: React集成开发](./08-React集成开发.md) | [目录](../README.md)
>
> **实战练习**: [Storage存储实战](../WXT-实战练习/07-Storage存储实战.md)

---

## 前言

浏览器扩展需要持久化存储用户设置、缓存数据等信息。与网页的 localStorage 不同，扩展有自己独立的存储空间，且支持跨设备同步。WXT 提供了强大的存储 API，在浏览器原生接口基础上增加了类型安全和响应式特性。

**本章你将学到：**
- 浏览器扩展存储的4种区域及其适用场景
- WXT 存储 API 的基本读写操作
- 使用 defineItem 创建类型安全的存储项
- 监听存储变化实现响应式更新
- 数据迁移和版本管理
- 在 React 中优雅地使用存储

---

## 1. 存储概述

### 1.1 为什么不能用 localStorage?

你可能会问：网页开发中常用的 localStorage 能用吗？答案是：**能用但不推荐**。

| 特性 | localStorage | browser.storage |
|------|--------------|-----------------|
| 跨设备同步 | 不支持 | 支持 (sync区域) |
| 容量 | 5-10MB | local: 10MB+, sync: 100KB |
| 内容脚本访问 | 访问的是网页的存储 | 访问扩展独立存储 |
| 事件监听 | 仅同一窗口 | 跨所有扩展上下文 |
| 数据隔离 | 与网页共享 | 完全隔离 |

**结论**：扩展开发应该使用 `browser.storage` API，WXT 的 `wxt/storage` 模块是对它的封装。

### 1.2 四种存储区域

浏览器扩展提供了4种存储区域，每种有不同的用途：

| 区域 | 键前缀 | 同步到云端 | 容量限制 | 典型用途 |
|------|--------|------------|----------|----------|
| **local** | `local:` | 否 | 10MB+ | 大量本地数据、翻译缓存 |
| **sync** | `sync:` | 是 | 100KB总量, 单项8KB | 用户设置、偏好配置 |
| **session** | `session:` | 否 | 10MB | 临时状态，关闭浏览器清除 |
| **managed** | `managed:` | 否 | - | 企业策略，只读 |

**选择建议**：
- 用户设置用 `sync:`（跨设备同步）
- 缓存数据用 `local:`（容量大）
- 临时状态用 `session:`（自动清理）

### 1.3 权限配置

使用存储前，需要在 manifest 中声明权限：

```typescript
// wxt.config.ts
export default defineConfig({
  manifest: {
    permissions: ['storage'],
  },
});
```

这一行配置让扩展获得访问所有存储区域的权限。

---

## 2. 基础存储操作

### 2.1 导入存储模块

WXT 提供了统一的存储模块，无需直接操作浏览器 API：

```typescript
import { storage } from 'wxt/storage';
```

### 2.2 基本读写

所有存储键都必须带区域前缀，这是 WXT 的约定：

```typescript
// 存储数据 - 注意键名必须带区域前缀
await storage.setItem('local:username', 'John');
await storage.setItem('sync:theme', 'dark');
await storage.setItem('session:tempData', { count: 0 });

// 读取数据 - 使用泛型指定返回类型
const username = await storage.getItem<string>('local:username');
const theme = await storage.getItem<string>('sync:theme');

// 删除数据
await storage.removeItem('local:username');

// 清除指定区域的所有数据
await storage.clear('local');
```

**为什么要加前缀？** 这样可以明确数据存储在哪个区域，避免混淆。比如 `sync:theme` 一眼就知道是会同步到云端的主题设置。

### 2.3 批量操作

当需要同时操作多个数据时，批量操作比逐个操作更高效：

```typescript
// 批量存储 - 一次 I/O 操作完成多个写入
await storage.setItems([
  { key: 'local:key1', value: 'value1' },
  { key: 'local:key2', value: 'value2' },
]);

// 批量读取
const items = await storage.getItems(['local:key1', 'local:key2']);
// 返回: [{ key: 'local:key1', value: 'value1' }, ...]

// 批量删除
await storage.removeItems(['local:key1', 'local:key2']);
```

---

## 3. 类型安全的存储项

直接使用 `getItem/setItem` 存在两个问题：
1. 每次都要写完整的键名，容易拼错
2. 类型需要手动指定，容易不一致

WXT 提供了 `defineItem` 来解决这些问题。

### 3.1 定义存储项

```typescript
import { storage } from 'wxt/storage';

// 一次定义，到处使用
const usernameItem = storage.defineItem<string>('local:username');

// 使用时不需要再写键名
const username = await usernameItem.getValue();
await usernameItem.setValue('John');
await usernameItem.removeValue();
```

### 3.2 设置默认值

使用 `fallback` 选项设置默认值，当存储中没有数据时返回默认值：

```typescript
const counterItem = storage.defineItem<number>('local:counter', {
  fallback: 0,  // 默认值
});

// 首次读取时，存储中没有数据，返回默认值 0
const count = await counterItem.getValue(); // 0

// 更新值
await counterItem.setValue(count + 1); // 存储为 1
```

### 3.3 复杂类型存储

存储复杂对象时，TypeScript 会提供完整的类型检查：

```typescript
// 定义设置接口
interface UserSettings {
  theme: 'light' | 'dark';
  language: string;
  notifications: boolean;
  shortcuts: Record<string, string>;
}

// 创建类型安全的存储项
const settingsItem = storage.defineItem<UserSettings>('sync:settings', {
  fallback: {
    theme: 'light',
    language: 'en',
    notifications: true,
    shortcuts: {},
  },
});

// 完整的类型提示和检查
const settings = await settingsItem.getValue();
settings.theme;  // 'light' | 'dark' - TypeScript 知道具体类型

// 部分更新
await settingsItem.setValue({
  ...settings,
  theme: 'dark',  // 只改这一项
});
```

### 3.4 数据迁移 (版本管理)

当你的数据结构需要升级时，可以使用版本化存储进行平滑迁移：

```typescript
// 旧版本数据结构
interface SettingsV1 {
  darkMode: boolean;  // 旧的简单布尔值
}

// 新版本数据结构
interface SettingsV2 {
  theme: 'light' | 'dark';  // 改为字符串枚举
  fontSize: number;          // 新增字段
}

const settingsItem = storage.defineItem<SettingsV2>('local:settings', {
  fallback: {
    theme: 'light',
    fontSize: 14,
  },
  version: 2,  // 当前版本号
  migrations: {
    // 从任何旧版本迁移到 v2 的逻辑
    2: (oldValue: SettingsV1): SettingsV2 => ({
      theme: oldValue.darkMode ? 'dark' : 'light',  // 转换旧字段
      fontSize: 14,  // 新字段用默认值
    }),
  },
});
```

**迁移原理**：WXT 会自动检测存储中数据的版本，如果低于当前版本，自动执行迁移函数。

---

## 4. 监听存储变化

存储变化监听是实现响应式 UI 的关键。当一个地方修改了数据，其他地方可以立即收到通知。

### 4.1 监听单个存储项

```typescript
const themeItem = storage.defineItem<string>('sync:theme', {
  fallback: 'light',
});

// 监听变化 - 返回取消监听的函数
const unwatch = themeItem.watch((newValue, oldValue) => {
  console.log('Theme changed:', oldValue, '->', newValue);
  applyTheme(newValue);  // 立即应用新主题
});

// 当不再需要监听时，调用 unwatch 取消
unwatch();
```

### 4.2 在 React 中使用

将存储监听封装为 Hook，让 React 组件自动响应存储变化：

```typescript
import { useState, useEffect } from 'react';
import { storage } from 'wxt/storage';

const themeItem = storage.defineItem<string>('sync:theme', {
  fallback: 'light',
});

function useTheme() {
  const [theme, setTheme] = useState<string>('light');
  
  useEffect(() => {
    // 1. 初始加载
    themeItem.getValue().then(setTheme);
    
    // 2. 监听后续变化
    const unwatch = themeItem.watch((newValue) => {
      setTheme(newValue);
    });
    
    // 3. 组件卸载时取消监听
    return unwatch;
  }, []);
  
  // 4. 提供更新方法
  const updateTheme = async (newTheme: string) => {
    await themeItem.setValue(newTheme);
    // 不需要手动 setTheme，watch 会自动触发
  };
  
  return { theme, updateTheme };
}
```

### 4.3 通用 Storage Hook

创建一个可复用的 Hook：

```typescript
// hooks/useStorageItem.ts
import { useState, useEffect, useCallback } from 'react';
import { storage, StorageItemKey } from 'wxt/storage';

export function useStorageItem<T>(key: StorageItemKey, defaultValue: T) {
  const [value, setValue] = useState<T>(defaultValue);
  const [loading, setLoading] = useState(true);
  
  // 创建存储项引用
  const item = storage.defineItem<T>(key, { fallback: defaultValue });
  
  useEffect(() => {
    // 加载初始值
    item.getValue().then((v) => {
      setValue(v);
      setLoading(false);
    });
    
    // 监听变化
    return item.watch((newValue) => {
      setValue(newValue);
    });
  }, [key]);
  
  // 更新方法
  const update = useCallback(async (newValue: T) => {
    await item.setValue(newValue);
  }, [item]);
  
  // 删除方法
  const remove = useCallback(async () => {
    await item.removeValue();
  }, [item]);
  
  return { value, loading, update, remove };
}

// 使用示例
function SettingsPage() {
  const { value: theme, update: setTheme } = useStorageItem('sync:theme', 'light');
  
  return (
    <select value={theme} onChange={(e) => setTheme(e.target.value)}>
      <option value="light">Light</option>
      <option value="dark">Dark</option>
    </select>
  );
}
```

---

## 5. 元数据存储

除了存储数据本身，有时还需要存储关于数据的元信息（如最后更新时间）。

```typescript
const counterItem = storage.defineItem<number>('local:counter', {
  fallback: 0,
});

// 存储元数据
await counterItem.setMeta({
  lastUpdated: Date.now(),
  updatedBy: 'user',
});

// 读取元数据
const meta = await counterItem.getMeta<{
  lastUpdated: number;
  updatedBy: string;
}>();
console.log('Last updated:', new Date(meta.lastUpdated));

// 元数据存储在独立的键中：
// 数据键: local:counter
// 元数据键: local:counter$  (加了 $ 后缀)
```

---

## 6. 快照与恢复

快照功能可以用于导出/导入设置、调试、备份等场景。

```typescript
// 获取某个区域的所有数据快照
const snapshot = await storage.snapshot('local');
console.log('Snapshot:', snapshot);
// { 'local:key1': value1, 'local:key2': value2, ... }

// 获取所有区域的快照
const allSnapshot = await storage.snapshot('all');

// 恢复快照 - 用于导入设置
await storage.restoreSnapshot(snapshot);
```

---

## 7. 集中式存储管理

推荐将所有存储项定义在一个文件中，方便管理和复用：

```typescript
// stores/index.ts
import { storage } from 'wxt/storage';

// ========== 用户设置 (同步到云端) ==========
export const userSettings = storage.defineItem<{
  theme: 'light' | 'dark';
  language: string;
}>('sync:userSettings', {
  fallback: {
    theme: 'light',
    language: 'en',
  },
});

// ========== 翻译缓存 (本地存储) ==========
export const translationCache = storage.defineItem<Record<string, string>>(
  'local:translationCache',
  { fallback: {} }
);

// ========== 临时状态 (会话存储) ==========
export const sessionState = storage.defineItem<{
  activeTab: number | null;
  lastAction: string;
}>('session:state', {
  fallback: {
    activeTab: null,
    lastAction: '',
  },
});
```

然后在任何地方导入使用：

```typescript
import { userSettings, translationCache } from '~/stores';

const settings = await userSettings.getValue();
await userSettings.setValue({ ...settings, theme: 'dark' });
```

---

## 8. 性能优化

### 8.1 批量操作减少 I/O

```typescript
// 不好: 多次 I/O 操作
await storage.setItem('local:a', 1);
await storage.setItem('local:b', 2);
await storage.setItem('local:c', 3);

// 好: 一次 I/O 操作
await storage.setItems([
  { key: 'local:a', value: 1 },
  { key: 'local:b', value: 2 },
  { key: 'local:c', value: 3 },
]);
```

### 8.2 防抖写入

频繁写入（如用户打字时自动保存）应该使用防抖：

```typescript
import { debounce } from 'lodash-es';

const settingsItem = storage.defineItem('local:settings', { fallback: {} });

// 300ms 内多次调用只执行最后一次
const saveSettings = debounce(async (settings: object) => {
  await settingsItem.setValue(settings);
}, 300);
```

---

## 9. 常见问题

### 9.1 存储容量限制

```typescript
// Chrome 可以查询已用空间
const usage = await browser.storage.local.getBytesInUse();
console.log('Used bytes:', usage);

// sync 存储的限制：
// - 总容量: 102,400 bytes (100KB)
// - 单项最大: 8,192 bytes (8KB)
// - 最多项数: 512
```

### 9.2 什么数据不能存储?

存储的数据必须是可 JSON 序列化的，以下类型无法存储：
- Function (函数)
- undefined (会被忽略)
- Symbol
- DOM 元素
- 循环引用对象

```typescript
// 日期需要转换为字符串
const data = {
  createdAt: new Date().toISOString(),  // 存为字符串
};

// 读取时再转回来
const date = new Date(data.createdAt);
```

### 9.3 跨上下文共享

存储在所有扩展上下文中共享，包括：
- Background
- Content Script  
- Popup
- Options
- Side Panel

使用 watch 可以保持各个上下文的同步：

```typescript
settingsItem.watch((newValue) => {
  // 任何上下文修改都会触发这里
  updateUI(newValue);
});
```

---

## 10. 小结

| 方法 | 说明 |
|------|------|
| `storage.setItem(key, value)` | 存储单个值 |
| `storage.getItem(key)` | 读取单个值 |
| `storage.removeItem(key)` | 删除单个值 |
| `storage.defineItem(key, options)` | 定义类型安全的存储项 |
| `item.getValue()` | 获取存储项的值 |
| `item.setValue(value)` | 设置存储项的值 |
| `item.watch(callback)` | 监听存储项变化 |
| `storage.snapshot(area)` | 创建存储快照 |

---

> **下一步**: 学习 [React集成开发](./08-React集成开发.md)，了解如何在 React 中优雅地使用这些存储 API。
