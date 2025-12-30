# WXT 沉浸式翻译插件实战 - 第四章: Popup与Options页面

## 4.1 页面概述

| 页面 | 用途 | 特点 |
|------|------|------|
| Popup | 点击图标弹出的快捷面板 | 小巧,快速操作 |
| Options | 完整的设置页面 | 功能完整,独立标签页 |

## 4.2 Popup 弹出页面

### src/entrypoints/popup/index.html

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>沉浸式翻译</title>
  <link rel="stylesheet" href="./styles.css">
</head>
<body>
  <div id="root"></div>
  <script type="module" src="./main.tsx"></script>
</body>
</html>
```

### src/entrypoints/popup/main.tsx

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { App } from './App';
import '@/assets/styles/popup.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### src/entrypoints/popup/App.tsx

```tsx
import React, { useState, useEffect, useCallback } from 'react';
import { browser } from 'wxt/browser';
import { settingsStorage, historyStorage } from '@/stores/storage';
import type { UserSettings } from '@/types/settings';
import type { TranslateResult } from '@/types/translate';
import { MessageType } from '@/types/message';
import { LanguageSelect } from '@/components/settings/LanguageSelect';
import { QuickTranslate } from '@/components/popup/QuickTranslate';
import { RecentHistory } from '@/components/popup/RecentHistory';

type TabType = 'translate' | 'history' | 'settings';

export const App: React.FC = () => {
  const [activeTab, setActiveTab] = useState<TabType>('translate');
  const [settings, setSettings] = useState<UserSettings | null>(null);
  const [history, setHistory] = useState<TranslateResult[]>([]);
  const [isEnabled, setIsEnabled] = useState(true);
  
  // 加载数据
  useEffect(() => {
    Promise.all([
      settingsStorage.getValue(),
      historyStorage.getValue(),
    ]).then(([loadedSettings, loadedHistory]) => {
      setSettings(loadedSettings);
      setHistory(loadedHistory.slice(0, 10));
      setIsEnabled(loadedSettings.enabled);
    });
  }, []);
  
  // 切换启用状态
  const handleToggleEnable = useCallback(async () => {
    if (!settings) return;
    
    const newEnabled = !isEnabled;
    setIsEnabled(newEnabled);
    
    await settingsStorage.setValue({
      ...settings,
      enabled: newEnabled,
    });
    
    // 通知当前标签页
    const [tab] = await browser.tabs.query({ active: true, currentWindow: true });
    if (tab?.id) {
      browser.tabs.sendMessage(tab.id, {
        type: MessageType.SETTINGS_CHANGED,
        payload: { enabled: newEnabled },
      }).catch(() => {});
    }
  }, [settings, isEnabled]);
  
  // 翻译当前页面
  const handleTranslatePage = useCallback(async () => {
    const [tab] = await browser.tabs.query({ active: true, currentWindow: true });
    if (tab?.id) {
      await browser.tabs.sendMessage(tab.id, {
        type: MessageType.TRANSLATE_PAGE,
        timestamp: Date.now(),
        payload: {},
      });
      window.close();
    }
  }, []);
  
  // 打开设置页面
  const handleOpenOptions = useCallback(() => {
    browser.runtime.openOptionsPage();
    window.close();
  }, []);
  
  // 更新语言设置
  const handleLanguageChange = useCallback(async (
    type: 'source' | 'target',
    value: string
  ) => {
    if (!settings) return;
    
    const newSettings = {
      ...settings,
      [type === 'source' ? 'sourceLanguage' : 'targetLanguage']: value,
    };
    
    setSettings(newSettings);
    await settingsStorage.setValue(newSettings);
  }, [settings]);
  
  if (!settings) {
    return (
      <div className="popup-container">
        <div className="loading">
          <div className="spinner"></div>
        </div>
      </div>
    );
  }
  
  return (
    <div className="popup-container">
      {/* 头部 */}
      <header className="popup-header">
        <div className="logo">
          <svg viewBox="0 0 24 24" width="24" height="24">
            <path fill="currentColor" d="M12.87 15.07l-2.54-2.51.03-.03c1.74-1.94 2.98-4.17 3.71-6.53H17V4h-7V2H8v2H1v1.99h11.17C11.5 7.92 10.44 9.75 9 11.35 8.07 10.32 7.3 9.19 6.69 8h-2c.73 1.63 1.73 3.17 2.98 4.56l-5.09 5.02L4 19l5-5 3.11 3.11.76-2.04zM18.5 10h-2L12 22h2l1.12-3h4.75L21 22h2l-4.5-12zm-2.62 7l1.62-4.33L19.12 17h-3.24z"/>
          </svg>
          <span>沉浸式翻译</span>
        </div>
        
        <label className="toggle-switch">
          <input
            type="checkbox"
            checked={isEnabled}
            onChange={handleToggleEnable}
          />
          <span className="slider"></span>
        </label>
      </header>
      
      {/* 标签页 */}
      <nav className="popup-tabs">
        <button
          className={`tab ${activeTab === 'translate' ? 'active' : ''}`}
          onClick={() => setActiveTab('translate')}
        >
          翻译
        </button>
        <button
          className={`tab ${activeTab === 'history' ? 'active' : ''}`}
          onClick={() => setActiveTab('history')}
        >
          历史
        </button>
        <button
          className={`tab ${activeTab === 'settings' ? 'active' : ''}`}
          onClick={() => setActiveTab('settings')}
        >
          设置
        </button>
      </nav>
      
      {/* 内容区 */}
      <main className="popup-content">
        {activeTab === 'translate' && (
          <div className="translate-tab">
            {/* 语言选择 */}
            <div className="language-bar">
              <LanguageSelect
                value={settings.sourceLanguage}
                onChange={(v) => handleLanguageChange('source', v)}
                includeAuto
              />
              <button className="swap-btn" onClick={() => {
                if (settings.sourceLanguage !== 'auto') {
                  handleLanguageChange('source', settings.targetLanguage);
                  handleLanguageChange('target', settings.sourceLanguage);
                }
              }}>
                <svg viewBox="0 0 24 24" width="20" height="20">
                  <path fill="currentColor" d="M6.99 11L3 15l3.99 4v-3H14v-2H6.99v-3zM21 9l-3.99-4v3H10v2h7.01v3L21 9z"/>
                </svg>
              </button>
              <LanguageSelect
                value={settings.targetLanguage}
                onChange={(v) => handleLanguageChange('target', v)}
              />
            </div>
            
            {/* 快速翻译 */}
            <QuickTranslate settings={settings} />
            
            {/* 快捷操作 */}
            <div className="quick-actions">
              <button className="action-btn primary" onClick={handleTranslatePage}>
                <svg viewBox="0 0 24 24" width="18" height="18">
                  <path fill="currentColor" d="M19 3H5c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h14c1.1 0 2-.9 2-2V5c0-1.1-.9-2-2-2zm-5 14H7v-2h7v2zm3-4H7v-2h10v2zm0-4H7V7h10v2z"/>
                </svg>
                翻译当前页面
              </button>
            </div>
          </div>
        )}
        
        {activeTab === 'history' && (
          <RecentHistory history={history} />
        )}
        
        {activeTab === 'settings' && (
          <div className="settings-tab">
            <div className="setting-group">
              <label>翻译模式</label>
              <select
                value={settings.translateMode}
                onChange={(e) => {
                  const newSettings = { ...settings, translateMode: e.target.value as any };
                  setSettings(newSettings);
                  settingsStorage.setValue(newSettings);
                }}
              >
                <option value="bilingual">双语对照</option>
                <option value="replace">替换原文</option>
                <option value="hover">悬停显示</option>
              </select>
            </div>
            
            <div className="setting-group">
              <label>翻译服务</label>
              <select
                value={settings.primaryService}
                onChange={(e) => {
                  const newSettings = { ...settings, primaryService: e.target.value as any };
                  setSettings(newSettings);
                  settingsStorage.setValue(newSettings);
                }}
              >
                <option value="google">Google 翻译</option>
                <option value="deepl">DeepL</option>
                <option value="openai">OpenAI</option>
                <option value="custom">自定义 API</option>
              </select>
            </div>
            
            <button className="open-options-btn" onClick={handleOpenOptions}>
              打开完整设置
              <svg viewBox="0 0 24 24" width="16" height="16">
                <path fill="currentColor" d="M19 19H5V5h7V3H5c-1.11 0-2 .9-2 2v14c0 1.1.89 2 2 2h14c1.1 0 2-.9 2-2v-7h-2v7zM14 3v2h3.59l-9.83 9.83 1.41 1.41L19 6.41V10h2V3h-7z"/>
              </svg>
            </button>
          </div>
        )}
      </main>
      
      {/* 底部 */}
      <footer className="popup-footer">
        <span>快捷键: {settings.shortcut}</span>
        <a href="#" onClick={(e) => { e.preventDefault(); handleOpenOptions(); }}>
          设置
        </a>
      </footer>
    </div>
  );
};
```

### src/components/popup/QuickTranslate.tsx

```tsx
import React, { useState, useCallback } from 'react';
import { sendToBackground } from '@/services/messaging';
import { MessageType } from '@/types/message';
import type { TranslateResult } from '@/types/translate';
import type { UserSettings } from '@/types/settings';

interface QuickTranslateProps {
  settings: UserSettings;
}

export const QuickTranslate: React.FC<QuickTranslateProps> = ({ settings }) => {
  const [inputText, setInputText] = useState('');
  const [result, setResult] = useState<TranslateResult | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  const handleTranslate = useCallback(async () => {
    if (!inputText.trim()) return;
    
    setIsLoading(true);
    setError(null);
    
    try {
      const response = await sendToBackground<TranslateResult>({
        type: MessageType.TRANSLATE_TEXT,
        timestamp: Date.now(),
        payload: {
          text: inputText.trim(),
          from: settings.sourceLanguage,
          to: settings.targetLanguage,
        },
      });
      
      if (response.success && response.data) {
        setResult(response.data);
      } else {
        setError(response.error || '翻译失败');
      }
    } catch (err) {
      setError('翻译请求失败');
    } finally {
      setIsLoading(false);
    }
  }, [inputText, settings]);
  
  const handleKeyDown = useCallback((e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      handleTranslate();
    }
  }, [handleTranslate]);
  
  const handleCopy = useCallback(() => {
    if (result?.translatedText) {
      navigator.clipboard.writeText(result.translatedText);
    }
  }, [result]);
  
  return (
    <div className="quick-translate">
      <div className="input-area">
        <textarea
          value={inputText}
          onChange={(e) => setInputText(e.target.value)}
          onKeyDown={handleKeyDown}
          placeholder="输入要翻译的文本..."
          rows={3}
        />
        <button
          className="translate-btn"
          onClick={handleTranslate}
          disabled={isLoading || !inputText.trim()}
        >
          {isLoading ? (
            <span className="spinner small"></span>
          ) : (
            <svg viewBox="0 0 24 24" width="20" height="20">
              <path fill="currentColor" d="M2.01 21L23 12 2.01 3 2 10l15 2-15 2z"/>
            </svg>
          )}
        </button>
      </div>
      
      {error && (
        <div className="error-message">{error}</div>
      )}
      
      {result && (
        <div className="result-area">
          <div className="result-text">{result.translatedText}</div>
          <button className="copy-btn" onClick={handleCopy} title="复制">
            <svg viewBox="0 0 24 24" width="16" height="16">
              <path fill="currentColor" d="M16 1H4c-1.1 0-2 .9-2 2v14h2V3h12V1zm3 4H8c-1.1 0-2 .9-2 2v14c0 1.1.9 2 2 2h11c1.1 0 2-.9 2-2V7c0-1.1-.9-2-2-2zm0 16H8V7h11v14z"/>
            </svg>
          </button>
        </div>
      )}
    </div>
  );
};
```

### src/components/popup/RecentHistory.tsx

```tsx
import React from 'react';
import type { TranslateResult } from '@/types/translate';

interface RecentHistoryProps {
  history: TranslateResult[];
}

export const RecentHistory: React.FC<RecentHistoryProps> = ({ history }) => {
  if (history.length === 0) {
    return (
      <div className="empty-history">
        <svg viewBox="0 0 24 24" width="48" height="48">
          <path fill="#ccc" d="M13 3c-4.97 0-9 4.03-9 9H1l3.89 3.89.07.14L9 12H6c0-3.87 3.13-7 7-7s7 3.13 7 7-3.13 7-7 7c-1.93 0-3.68-.79-4.94-2.06l-1.42 1.42C8.27 19.99 10.51 21 13 21c4.97 0 9-4.03 9-9s-4.03-9-9-9zm-1 5v5l4.28 2.54.72-1.21-3.5-2.08V8H12z"/>
        </svg>
        <p>暂无翻译历史</p>
      </div>
    );
  }
  
  return (
    <div className="history-list">
      {history.map((item, index) => (
        <div key={index} className="history-item">
          <div className="history-original">{item.originalText}</div>
          <div className="history-translated">{item.translatedText}</div>
          <div className="history-meta">
            <span className="service">{item.service}</span>
            <span className="time">
              {new Date(item.timestamp).toLocaleTimeString()}
            </span>
          </div>
        </div>
      ))}
    </div>
  );
};
```

### src/assets/styles/popup.css

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  width: 360px;
  min-height: 400px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  font-size: 14px;
  color: #333;
  background: #fff;
}

.popup-container {
  display: flex;
  flex-direction: column;
  min-height: 400px;
}

/* 头部 */
.popup-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 16px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.logo {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 16px;
  font-weight: 600;
}

/* 开关 */
.toggle-switch {
  position: relative;
  width: 44px;
  height: 24px;
}

.toggle-switch input {
  opacity: 0;
  width: 0;
  height: 0;
}

.slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(255, 255, 255, 0.3);
  transition: .3s;
  border-radius: 24px;
}

.slider:before {
  position: absolute;
  content: "";
  height: 18px;
  width: 18px;
  left: 3px;
  bottom: 3px;
  background-color: white;
  transition: .3s;
  border-radius: 50%;
}

input:checked + .slider {
  background-color: rgba(255, 255, 255, 0.6);
}

input:checked + .slider:before {
  transform: translateX(20px);
}

/* 标签页 */
.popup-tabs {
  display: flex;
  border-bottom: 1px solid #eee;
}

.tab {
  flex: 1;
  padding: 12px;
  background: none;
  border: none;
  border-bottom: 2px solid transparent;
  cursor: pointer;
  font-size: 14px;
  color: #666;
  transition: all .2s;
}

.tab:hover {
  color: #667eea;
}

.tab.active {
  color: #667eea;
  border-bottom-color: #667eea;
}

/* 内容区 */
.popup-content {
  flex: 1;
  padding: 16px;
  overflow-y: auto;
}

/* 语言选择栏 */
.language-bar {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 16px;
}

.language-bar select {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
  background: white;
  cursor: pointer;
}

.swap-btn {
  padding: 8px;
  background: none;
  border: 1px solid #ddd;
  border-radius: 6px;
  cursor: pointer;
  color: #666;
  transition: all .2s;
}

.swap-btn:hover {
  background: #f5f5f5;
  color: #667eea;
}

/* 快速翻译 */
.quick-translate {
  margin-bottom: 16px;
}

.input-area {
  position: relative;
}

.input-area textarea {
  width: 100%;
  padding: 12px;
  padding-right: 48px;
  border: 1px solid #ddd;
  border-radius: 8px;
  font-size: 14px;
  resize: none;
  font-family: inherit;
}

.input-area textarea:focus {
  outline: none;
  border-color: #667eea;
}

.translate-btn {
  position: absolute;
  right: 8px;
  bottom: 8px;
  width: 36px;
  height: 36px;
  border: none;
  border-radius: 50%;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: transform .2s;
}

.translate-btn:hover:not(:disabled) {
  transform: scale(1.1);
}

.translate-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.result-area {
  position: relative;
  margin-top: 12px;
  padding: 12px;
  background: #f8f9fa;
  border-radius: 8px;
}

.result-text {
  line-height: 1.6;
}

.copy-btn {
  position: absolute;
  top: 8px;
  right: 8px;
  padding: 4px;
  background: none;
  border: none;
  cursor: pointer;
  color: #999;
  transition: color .2s;
}

.copy-btn:hover {
  color: #667eea;
}

.error-message {
  margin-top: 8px;
  padding: 8px 12px;
  background: #fee;
  color: #c00;
  border-radius: 6px;
  font-size: 13px;
}

/* 快捷操作 */
.quick-actions {
  margin-top: 16px;
}

.action-btn {
  width: 100%;
  padding: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  transition: all .2s;
}

.action-btn.primary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.action-btn.primary:hover {
  opacity: 0.9;
  transform: translateY(-1px);
}

/* 历史记录 */
.empty-history {
  text-align: center;
  padding: 40px 20px;
  color: #999;
}

.empty-history p {
  margin-top: 12px;
}

.history-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.history-item {
  padding: 12px;
  background: #f8f9fa;
  border-radius: 8px;
}

.history-original {
  font-size: 13px;
  color: #666;
  margin-bottom: 8px;
  line-height: 1.4;
}

.history-translated {
  font-size: 14px;
  line-height: 1.5;
  margin-bottom: 8px;
}

.history-meta {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  color: #999;
}

.history-meta .service {
  text-transform: capitalize;
}

/* 设置项 */
.setting-group {
  margin-bottom: 16px;
}

.setting-group label {
  display: block;
  margin-bottom: 6px;
  font-size: 13px;
  color: #666;
}

.setting-group select {
  width: 100%;
  padding: 10px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
  background: white;
}

.open-options-btn {
  width: 100%;
  padding: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  background: #f8f9fa;
  border: 1px solid #ddd;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
  color: #666;
  transition: all .2s;
}

.open-options-btn:hover {
  background: #f0f0f0;
  color: #667eea;
}

/* 底部 */
.popup-footer {
  display: flex;
  justify-content: space-between;
  padding: 12px 16px;
  border-top: 1px solid #eee;
  font-size: 12px;
  color: #999;
}

.popup-footer a {
  color: #667eea;
  text-decoration: none;
}

/* 加载动画 */
.loading {
  display: flex;
  align-items: center;
  justify-content: center;
  height: 200px;
}

.spinner {
  width: 32px;
  height: 32px;
  border: 3px solid #f3f3f3;
  border-top: 3px solid #667eea;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

.spinner.small {
  width: 16px;
  height: 16px;
  border-width: 2px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
```

## 4.3 Options 设置页面

### src/entrypoints/options/index.html

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>沉浸式翻译 - 设置</title>
</head>
<body>
  <div id="root"></div>
  <script type="module" src="./main.tsx"></script>
</body>
</html>
```

### src/entrypoints/options/main.tsx

```tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { App } from './App';
import '@/assets/styles/options.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

### src/entrypoints/options/App.tsx

```tsx
import React, { useState, useEffect, useCallback } from 'react';
import { settingsStorage } from '@/stores/storage';
import type { UserSettings } from '@/types/settings';
import { defaultSettings } from '@/types/settings';
import { GeneralSettings } from '@/components/options/GeneralSettings';
import { TranslateSettings } from '@/components/options/TranslateSettings';
import { ServiceSettings } from '@/components/options/ServiceSettings';
import { AdvancedSettings } from '@/components/options/AdvancedSettings';
import { About } from '@/components/options/About';

type TabType = 'general' | 'translate' | 'service' | 'advanced' | 'about';

export const App: React.FC = () => {
  const [activeTab, setActiveTab] = useState<TabType>('general');
  const [settings, setSettings] = useState<UserSettings>(defaultSettings);
  const [isSaving, setIsSaving] = useState(false);
  const [saveMessage, setSaveMessage] = useState<string | null>(null);
  
  // 加载设置
  useEffect(() => {
    settingsStorage.getValue().then(setSettings);
    
    // 检查 URL hash 是否有欢迎页参数
    if (window.location.hash === '#welcome') {
      setActiveTab('about');
    }
  }, []);
  
  // 更新设置
  const updateSettings = useCallback(async (newSettings: Partial<UserSettings>) => {
    const updated = { ...settings, ...newSettings };
    setSettings(updated);
  }, [settings]);
  
  // 保存设置
  const handleSave = useCallback(async () => {
    setIsSaving(true);
    setSaveMessage(null);
    
    try {
      await settingsStorage.setValue(settings);
      setSaveMessage('设置已保存');
      setTimeout(() => setSaveMessage(null), 2000);
    } catch (error) {
      setSaveMessage('保存失败');
    } finally {
      setIsSaving(false);
    }
  }, [settings]);
  
  // 重置设置
  const handleReset = useCallback(async () => {
    if (window.confirm('确定要重置所有设置吗?')) {
      setSettings(defaultSettings);
      await settingsStorage.setValue(defaultSettings);
      setSaveMessage('设置已重置');
      setTimeout(() => setSaveMessage(null), 2000);
    }
  }, []);
  
  // 导出设置
  const handleExport = useCallback(() => {
    const data = JSON.stringify(settings, null, 2);
    const blob = new Blob([data], { type: 'application/json' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'immersive-translate-settings.json';
    a.click();
    URL.revokeObjectURL(url);
  }, [settings]);
  
  // 导入设置
  const handleImport = useCallback(() => {
    const input = document.createElement('input');
    input.type = 'file';
    input.accept = '.json';
    input.onchange = async (e) => {
      const file = (e.target as HTMLInputElement).files?.[0];
      if (!file) return;
      
      try {
        const text = await file.text();
        const imported = JSON.parse(text);
        const merged = { ...defaultSettings, ...imported };
        setSettings(merged);
        await settingsStorage.setValue(merged);
        setSaveMessage('设置已导入');
        setTimeout(() => setSaveMessage(null), 2000);
      } catch {
        setSaveMessage('导入失败: 文件格式错误');
      }
    };
    input.click();
  }, []);
  
  const tabs: { key: TabType; label: string; icon: string }[] = [
    { key: 'general', label: '基本设置', icon: 'M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-6h2v6zm0-8h-2V7h2v2z' },
    { key: 'translate', label: '翻译设置', icon: 'M12.87 15.07l-2.54-2.51.03-.03c1.74-1.94 2.98-4.17 3.71-6.53H17V4h-7V2H8v2H1v1.99h11.17C11.5 7.92 10.44 9.75 9 11.35 8.07 10.32 7.3 9.19 6.69 8h-2c.73 1.63 1.73 3.17 2.98 4.56l-5.09 5.02L4 19l5-5 3.11 3.11.76-2.04z' },
    { key: 'service', label: '翻译服务', icon: 'M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm-1 17.93c-3.95-.49-7-3.85-7-7.93 0-.62.08-1.21.21-1.79L9 15v1c0 1.1.9 2 2 2v1.93zm6.9-2.54c-.26-.81-1-1.39-1.9-1.39h-1v-3c0-.55-.45-1-1-1H8v-2h2c.55 0 1-.45 1-1V7h2c1.1 0 2-.9 2-2v-.41c2.93 1.19 5 4.06 5 7.41 0 2.08-.8 3.97-2.1 5.39z' },
    { key: 'advanced', label: '高级设置', icon: 'M19.14 12.94c.04-.31.06-.63.06-.94 0-.31-.02-.63-.06-.94l2.03-1.58c.18-.14.23-.41.12-.61l-1.92-3.32c-.12-.22-.37-.29-.59-.22l-2.39.96c-.5-.38-1.03-.7-1.62-.94l-.36-2.54c-.04-.24-.24-.41-.48-.41h-3.84c-.24 0-.43.17-.47.41l-.36 2.54c-.59.24-1.13.57-1.62.94l-2.39-.96c-.22-.08-.47 0-.59.22L2.74 8.87c-.12.21-.08.47.12.61l2.03 1.58c-.04.31-.06.63-.06.94s.02.63.06.94l-2.03 1.58c-.18.14-.23.41-.12.61l1.92 3.32c.12.22.37.29.59.22l2.39-.96c.5.38 1.03.7 1.62.94l.36 2.54c.05.24.24.41.48.41h3.84c.24 0 .44-.17.47-.41l.36-2.54c.59-.24 1.13-.56 1.62-.94l2.39.96c.22.08.47 0 .59-.22l1.92-3.32c.12-.22.07-.47-.12-.61l-2.01-1.58z' },
    { key: 'about', label: '关于', icon: 'M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 17h-2v-2h2v2zm2.07-7.75l-.9.92C13.45 12.9 13 13.5 13 15h-2v-.5c0-1.1.45-2.1 1.17-2.83l1.24-1.26c.37-.36.59-.86.59-1.41 0-1.1-.9-2-2-2s-2 .9-2 2H8c0-2.21 1.79-4 4-4s4 1.79 4 4c0 .88-.36 1.68-.93 2.25z' },
  ];
  
  return (
    <div className="options-container">
      {/* 侧边栏 */}
      <aside className="sidebar">
        <div className="sidebar-header">
          <svg viewBox="0 0 24 24" width="32" height="32">
            <path fill="currentColor" d="M12.87 15.07l-2.54-2.51.03-.03c1.74-1.94 2.98-4.17 3.71-6.53H17V4h-7V2H8v2H1v1.99h11.17C11.5 7.92 10.44 9.75 9 11.35 8.07 10.32 7.3 9.19 6.69 8h-2c.73 1.63 1.73 3.17 2.98 4.56l-5.09 5.02L4 19l5-5 3.11 3.11.76-2.04zM18.5 10h-2L12 22h2l1.12-3h4.75L21 22h2l-4.5-12zm-2.62 7l1.62-4.33L19.12 17h-3.24z"/>
          </svg>
          <h1>沉浸式翻译</h1>
        </div>
        
        <nav className="sidebar-nav">
          {tabs.map((tab) => (
            <button
              key={tab.key}
              className={`nav-item ${activeTab === tab.key ? 'active' : ''}`}
              onClick={() => setActiveTab(tab.key)}
            >
              <svg viewBox="0 0 24 24" width="20" height="20">
                <path fill="currentColor" d={tab.icon} />
              </svg>
              <span>{tab.label}</span>
            </button>
          ))}
        </nav>
        
        <div className="sidebar-footer">
          <button onClick={handleExport} className="footer-btn">
            导出设置
          </button>
          <button onClick={handleImport} className="footer-btn">
            导入设置
          </button>
        </div>
      </aside>
      
      {/* 主内容区 */}
      <main className="main-content">
        <header className="content-header">
          <h2>{tabs.find((t) => t.key === activeTab)?.label}</h2>
          <div className="header-actions">
            {saveMessage && (
              <span className={`save-message ${saveMessage.includes('失败') ? 'error' : 'success'}`}>
                {saveMessage}
              </span>
            )}
            <button onClick={handleReset} className="btn secondary">
              重置
            </button>
            <button onClick={handleSave} className="btn primary" disabled={isSaving}>
              {isSaving ? '保存中...' : '保存'}
            </button>
          </div>
        </header>
        
        <div className="content-body">
          {activeTab === 'general' && (
            <GeneralSettings settings={settings} onChange={updateSettings} />
          )}
          {activeTab === 'translate' && (
            <TranslateSettings settings={settings} onChange={updateSettings} />
          )}
          {activeTab === 'service' && (
            <ServiceSettings settings={settings} onChange={updateSettings} />
          )}
          {activeTab === 'advanced' && (
            <AdvancedSettings settings={settings} onChange={updateSettings} />
          )}
          {activeTab === 'about' && <About />}
        </div>
      </main>
    </div>
  );
};
```

### src/components/options/GeneralSettings.tsx

```tsx
import React from 'react';
import type { UserSettings } from '@/types/settings';
import { LanguageSelect } from '@/components/settings/LanguageSelect';

interface GeneralSettingsProps {
  settings: UserSettings;
  onChange: (settings: Partial<UserSettings>) => void;
}

export const GeneralSettings: React.FC<GeneralSettingsProps> = ({
  settings,
  onChange,
}) => {
  return (
    <div className="settings-section">
      <div className="setting-card">
        <h3>插件状态</h3>
        <div className="setting-row">
          <div className="setting-info">
            <label>启用插件</label>
            <p>关闭后插件将停止工作</p>
          </div>
          <label className="switch">
            <input
              type="checkbox"
              checked={settings.enabled}
              onChange={(e) => onChange({ enabled: e.target.checked })}
            />
            <span className="slider"></span>
          </label>
        </div>
      </div>
      
      <div className="setting-card">
        <h3>语言设置</h3>
        <div className="setting-row">
          <div className="setting-info">
            <label>源语言</label>
            <p>要翻译的语言,选择"自动检测"可自动识别</p>
          </div>
          <LanguageSelect
            value={settings.sourceLanguage}
            onChange={(v) => onChange({ sourceLanguage: v as any })}
            includeAuto
          />
        </div>
        <div className="setting-row">
          <div className="setting-info">
            <label>目标语言</label>
            <p>翻译结果的语言</p>
          </div>
          <LanguageSelect
            value={settings.targetLanguage}
            onChange={(v) => onChange({ targetLanguage: v as any })}
          />
        </div>
      </div>
      
      <div className="setting-card">
        <h3>快捷键</h3>
        <div className="setting-row">
          <div className="setting-info">
            <label>翻译快捷键</label>
            <p>快速切换翻译状态</p>
          </div>
          <input
            type="text"
            className="shortcut-input"
            value={settings.shortcut}
            onChange={(e) => onChange({ shortcut: e.target.value })}
            placeholder="如: Alt+T"
          />
        </div>
      </div>
    </div>
  );
};
```

### src/components/options/ServiceSettings.tsx

```tsx
import React, { useState } from 'react';
import type { UserSettings } from '@/types/settings';

interface ServiceSettingsProps {
  settings: UserSettings;
  onChange: (settings: Partial<UserSettings>) => void;
}

export const ServiceSettings: React.FC<ServiceSettingsProps> = ({
  settings,
  onChange,
}) => {
  const [showApiKey, setShowApiKey] = useState<Record<string, boolean>>({});
  
  const toggleShowApiKey = (key: string) => {
    setShowApiKey((prev) => ({ ...prev, [key]: !prev[key] }));
  };
  
  const updateApiKey = (service: string, value: string) => {
    onChange({
      apiKeys: {
        ...settings.apiKeys,
        [service]: value,
      },
    });
  };
  
  return (
    <div className="settings-section">
      <div className="setting-card">
        <h3>翻译服务选择</h3>
        <div className="setting-row">
          <div className="setting-info">
            <label>主要翻译服务</label>
            <p>首选的翻译服务</p>
          </div>
          <select
            value={settings.primaryService}
            onChange={(e) => onChange({ primaryService: e.target.value as any })}
          >
            <option value="google">Google 翻译 (免费)</option>
            <option value="deepl">DeepL (需要 API Key)</option>
            <option value="openai">OpenAI (需要 API Key)</option>
            <option value="custom">自定义 API</option>
          </select>
        </div>
        <div className="setting-row">
          <div className="setting-info">
            <label>备用翻译服务</label>
            <p>主服务失败时使用</p>
          </div>
          <select
            value={settings.fallbackService || ''}
            onChange={(e) => onChange({ fallbackService: e.target.value as any || undefined })}
          >
            <option value="">无</option>
            <option value="google">Google 翻译</option>
            <option value="deepl">DeepL</option>
            <option value="openai">OpenAI</option>
          </select>
        </div>
      </div>
      
      <div className="setting-card">
        <h3>DeepL 设置</h3>
        <div className="setting-row">
          <div className="setting-info">
            <label>API Key</label>
            <p>
              <a href="https://www.deepl.com/pro-api" target="_blank" rel="noopener">
                获取 DeepL API Key
              </a>
            </p>
          </div>
          <div className="api-key-input">
            <input
              type={showApiKey.deepl ? 'text' : 'password'}
              value={settings.apiKeys.deepl || ''}
              onChange={(e) => updateApiKey('deepl', e.target.value)}
              placeholder="输入 DeepL API Key"
            />
            <button
              type="button"
              className="toggle-visibility"
              onClick={() => toggleShowApiKey('deepl')}
            >
              {showApiKey.deepl ? '隐藏' : '显示'}
            </button>
          </div>
        </div>
      </div>
      
      <div className="setting-card">
        <h3>OpenAI 设置</h3>
        <div className="setting-row">
          <div className="setting-info">
            <label>API Key</label>
            <p>
              <a href="https://platform.openai.com/api-keys" target="_blank" rel="noopener">
                获取 OpenAI API Key
              </a>
            </p>
          </div>
          <div className="api-key-input">
            <input
              type={showApiKey.openai ? 'text' : 'password'}
              value={settings.apiKeys.openai || ''}
              onChange={(e) => updateApiKey('openai', e.target.value)}
              placeholder="输入 OpenAI API Key"
            />
            <button
              type="button"
              className="toggle-visibility"
              onClick={() => toggleShowApiKey('openai')}
            >
              {showApiKey.openai ? '隐藏' : '显示'}
            </button>
          </div>
        </div>
      </div>
      
      <div className="setting-card">
        <h3>自定义 API</h3>
        <div className="setting-row">
          <div className="setting-info">
            <label>API 地址</label>
            <p>自定义翻译 API 的 URL</p>
          </div>
          <input
            type="url"
            value={settings.customApi?.url || ''}
            onChange={(e) =>
              onChange({
                customApi: {
                  ...settings.customApi,
                  url: e.target.value,
                  method: settings.customApi?.method || 'POST',
                },
              })
            }
            placeholder="https://api.example.com/translate"
          />
        </div>
        <div className="setting-row">
          <div className="setting-info">
            <label>请求方法</label>
          </div>
          <select
            value={settings.customApi?.method || 'POST'}
            onChange={(e) =>
              onChange({
                customApi: {
                  ...settings.customApi,
                  url: settings.customApi?.url || '',
                  method: e.target.value as 'GET' | 'POST',
                },
              })
            }
          >
            <option value="POST">POST</option>
            <option value="GET">GET</option>
          </select>
        </div>
      </div>
    </div>
  );
};
```

### src/components/settings/LanguageSelect.tsx

```tsx
import React from 'react';

interface LanguageSelectProps {
  value: string;
  onChange: (value: string) => void;
  includeAuto?: boolean;
}

const languages = [
  { code: 'zh-CN', name: '简体中文' },
  { code: 'zh-TW', name: '繁体中文' },
  { code: 'en', name: '英语' },
  { code: 'ja', name: '日语' },
  { code: 'ko', name: '韩语' },
  { code: 'fr', name: '法语' },
  { code: 'de', name: '德语' },
  { code: 'es', name: '西班牙语' },
  { code: 'ru', name: '俄语' },
];

export const LanguageSelect: React.FC<LanguageSelectProps> = ({
  value,
  onChange,
  includeAuto = false,
}) => {
  return (
    <select value={value} onChange={(e) => onChange(e.target.value)}>
      {includeAuto && <option value="auto">自动检测</option>}
      {languages.map((lang) => (
        <option key={lang.code} value={lang.code}>
          {lang.name}
        </option>
      ))}
    </select>
  );
};
```

## 4.4 本章小结

本章实现了用户界面的两个核心页面:

1. **Popup 弹出页面**: 提供快速翻译、历史记录和基本设置
2. **Options 设置页面**: 完整的插件配置界面

下一章我们将实现高级功能和优化。
