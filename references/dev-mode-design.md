---
name: dev-mode-design
description: 开发者模式设计：调试面板、支付沙箱、API调试、性能分析
---

# 开发者模式设计指南

## 核心原则

开发者模式不是"后加的功能"，而是项目架构的一部分。在项目初期就要规划好。

---

## 开发者模式功能架构

```
devMode/
  index.ts                  # 开发者模式入口
  config.ts                 # 开发者模式配置
  hooks/
    useDevMode.ts           # 是否开启开发者模式
    useDebugPanel.ts        # 调试面板状态
  components/
    DevPanel/               # 开发者模式面板
      index.tsx
      components/
        UserSwitcher.tsx    # 用户切换
        PaymentSandbox.tsx  # 支付沙箱
        ApiInspector.tsx    # API调试
        StateInspector.tsx  # 状态查看
        PerformanceProfiler.tsx
        I18nTester.tsx      # 多语言测试
    DevButton/              # 开发者入口按钮
  utils/
    logger.ts               # 调试日志
    storage.ts              # 开发者配置存储
  services/
    mockService.ts          # 模拟数据服务
    sandboxPayment.ts       # 沙箱支付服务
```

---

## 核心功能实现

### 1. 开发者模式开关

```typescript
// config/index.ts

export function checkDevMode(): boolean {
  if (typeof window === 'undefined') return false;
  
  const urlParams = new URLSearchParams(window.location.search);
  
  // 优先级
  return (
    // 1. URL参数 ?dev=true (最高优先)
    urlParams.get('dev') === 'true' ||
    // 2. localStorage (浏览器存储，开发者可手动设置)
    localStorage.getItem('devMode') === 'true' ||
    // 3. 构建时配置 (生产环境一般关闭)
    process.env.DEV_MODE === 'true'
  );
}

// 开发者模式配置
export interface DevModeFeatures {
  // 支付沙箱
  mockPayment: boolean;
  
  // 用户切换
  impersonateUser: boolean;
  
  // API调试
  showApiInspector: boolean;
  mockApiResponse: boolean;
  delayAllRequests: number;  // 毫秒
  
  // 日志
  logLevel: 'debug' | 'info' | 'warn' | 'error';
  
  // 性能分析
  showRenderProfiler: boolean;
  showBundleAnalyzer: boolean;
  
  // 多语言
  localeOverride: string | null;
  rtlSimulator: boolean;
  
  // 可选：跳过认证
  skipAuth: boolean;
}

export const defaultDevFeatures: DevModeFeatures = {
  mockPayment: true,
  impersonateUser: true,
  showApiInspector: true,
  mockApiResponse: false,
  delayAllRequests: 0,
  logLevel: 'debug',
  showRenderProfiler: false,
  showBundleAnalyzer: false,
  localeOverride: null,
  rtlSimulator: false,
  skipAuth: false,
};
```

### 2. 开发者模式Hook

```typescript
// hooks/useDevMode.ts

import { createContext, useContext, useState, useEffect } from 'react';
import { checkDevMode, defaultDevFeatures, type DevModeFeatures } from '@/config';

interface DevModeContextType {
  enabled: boolean;
  features: DevModeFeatures;
  setFeature: (key: keyof DevModeFeatures, value: any) => void;
  
  // 快捷方法
  isFeatureEnabled: (feature: keyof DevModeFeatures) => boolean;
}

const DevModeContext = createContext<DevModeContextType | null>(null);

export function DevModeProvider({ children }: { children: React.ReactNode }) {
  const [enabled, setEnabled] = useState(() => checkDevMode());
  const [features, setFeatures] = useState<DevModeFeatures>(() => {
    // 从localStorage加载，或使用默认值
    const stored = localStorage.getItem('devModeFeatures');
    return stored ? JSON.parse(stored) : defaultDevFeatures;
  });
  
  // 持久化到localStorage
  useEffect(() => {
    if (enabled) {
      localStorage.setItem('devModeFeatures', JSON.stringify(features));
    }
  }, [enabled, features]);
  
  const setFeature = (key: keyof DevModeFeatures, value: any) => {
    setFeatures(prev => ({ ...prev, [key]: value }));
  };
  
  const isFeatureEnabled = (feature: keyof DevModeFeatures) => {
    return enabled && features[feature];
  };
  
  // URL参数切换
  useEffect(() => {
    const urlParams = new URLSearchParams(window.location.search);
    if (urlParams.get('dev') === 'true') {
      setEnabled(true);
      localStorage.setItem('devMode', 'true');
    }
  }, []);
  
  return (
    <DevModeContext.Provider value={{ enabled, features, setFeature, isFeatureEnabled }}>
      {children}
    </DevModeContext.Provider>
  );
}

export function useDevMode() {
  const context = useContext(DevModeContext);
  if (!context) throw new Error('useDevMode must be used within DevModeProvider');
  return context;
}
```

### 3. 调试面板UI

```typescript
// components/DevPanel/index.tsx

import { useState } from 'react';
import { useDevMode } from '@/hooks/useDevMode';
import { UserSwitcher } from './components/UserSwitcher';
import { PaymentSandbox } from './components/PaymentSandbox';
import { ApiInspector } from './components/ApiInspector';
import { StateInspector } from './components/StateInspector';
import { I18nTester } from './components/I18nTester';

export function DevPanel() {
  const { enabled, features, setFeature } = useDevMode();
  const [isOpen, setIsOpen] = useState(false);
  const [activeTab, setActiveTab] = useState<string>('overview');
  
  if (!enabled) return null;
  
  return (
    <>
      {/* 浮动入口按钮 */}
      <div
        className="fixed bottom-4 right-4 bg-blue-500 text-white px-3 py-2 rounded-full cursor-pointer z-50"
        onClick={() => setIsOpen(!isOpen)}
        title="开发者模式"
      >
        🛠️ Dev
      </div>
      
      {/* 调试面板 */}
      {isOpen && (
        <div className="fixed bottom-16 right-4 w-96 bg-gray-900 text-white rounded-lg shadow-xl z-50 overflow-hidden">
          {/* 标签页导航 */}
          <div className="flex border-b border-gray-700">
            {[
              { id: 'overview', label: '概览' },
              { id: 'users', label: '用户' },
              { id: 'payment', label: '支付' },
              { id: 'api', label: 'API' },
              { id: 'state', label: '状态' },
              { id: 'i18n', label: '语言' },
              { id: 'perf', label: '性能' },
            ].map(tab => (
              <button
                key={tab.id}
                className={`px-3 py-2 text-sm ${activeTab === tab.id ? 'bg-gray-700' : ''}`}
                onClick={() => setActiveTab(tab.id)}
              >
                {tab.label}
              </button>
            ))}
          </div>
          
          {/* 内容区域 */}
          <div className="p-4 max-h-96 overflow-auto">
            {activeTab === 'overview' && <OverviewTab />}
            {activeTab === 'users' && <UserSwitcher />}
            {activeTab === 'payment' && <PaymentSandbox />}
            {activeTab === 'api' && <ApiInspector />}
            {activeTab === 'state' && <StateInspector />}
            {activeTab === 'i18n' && <I18nTester />}
            {activeTab === 'perf' && <PerformanceProfiler />}
          </div>
        </div>
      )}
    </>
  );
}
```

### 4. 用户切换功能

```typescript
// components/DevPanel/components/UserSwitcher.tsx

import { useState } from 'react';
import { useDevMode } from '@/hooks/useDevMode';
import { api } from '@/lib/api';

interface TestUser {
  id: string;
  name: string;
  email: string;
  role: string;
}

export function UserSwitcher() {
  const [users, setUsers] = useState<TestUser[]>([]);
  const [currentUser, setCurrentUser] = useState<TestUser | null>(null);
  const [search, setSearch] = useState('');
  
  // 预设测试用户
  const presetUsers: TestUser[] = [
    { id: '1', name: '管理员', email: 'admin@test.com', role: 'admin' },
    { id: '2', name: '普通用户', email: 'user@test.com', role: 'user' },
    { id: '3', name: 'VIP用户', email: 'vip@test.com', role: 'vip' },
    { id: '4', name: '新用户', email: 'new@test.com', role: 'new' },
  ];
  
  const handleSwitchUser = async (user: TestUser) => {
    // 1. 调用模拟登录API
    try {
      await api.post('/dev/impersonate', { userId: user.id });
      setCurrentUser(user);
      localStorage.setItem('impersonatedUserId', user.id);
      // 刷新页面以应用新用户
      window.location.reload();
    } catch (err) {
      console.error('模拟登录失败', err);
    }
  };
  
  const handleStopImpersonate = async () => {
    await api.post('/dev/stop-impersonate');
    setCurrentUser(null);
    localStorage.removeItem('impersonatedUserId');
    window.location.reload();
  };
  
  return (
    <div className="space-y-4">
      <h3 className="font-bold">用户切换</h3>
      
      {currentUser && (
        <div className="bg-yellow-500/20 p-2 rounded flex justify-between items-center">
          <div>
            <div className="text-sm">当前模拟用户：</div>
            <div className="font-bold">{currentUser.name}</div>
          </div>
          <button
            className="text-xs bg-red-500 px-2 py-1 rounded"
            onClick={handleStopImpersonate}
          >
            停止
          </button>
        </div>
      )}
      
      {/* 搜索框 */}
      <input
        type="text"
        placeholder="搜索用户..."
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        className="w-full px-3 py-2 bg-gray-700 rounded"
      />
      
      {/* 预设用户 */}
      <div className="space-y-2">
        {presetUsers
          .filter(u => u.name.includes(search) || u.email.includes(search))
          .map(user => (
            <button
              key={user.id}
              className="w-full text-left p-2 bg-gray-700 hover:bg-gray-600 rounded"
              onClick={() => handleSwitchUser(user)}
            >
              <div className="font-bold">{user.name}</div>
              <div className="text-xs text-gray-400">{user.email}</div>
              <div className="text-xs text-blue-400">{user.role}</div>
            </button>
          ))}
      </div>
    </div>
  );
}
```

### 5. 支付沙箱

```typescript
// components/DevPanel/components/PaymentSandbox.tsx

import { useState } from 'react';
import { useDevMode } from '@/hooks/useDevMode';
import { api } from '@/lib/api';

export function PaymentSandbox() {
  const { features, setFeature } = useDevMode();
  const [scenario, setScenario] = useState<string>('success');
  const [logs, setLogs] = useState<string[]>([]);
  
  const scenarios = [
    { value: 'success', label: '支付成功', icon: '✅' },
    { value: 'insufficient_funds', label: '余额不足', icon: '💳' },
    { value: 'card_declined', label: '卡被拒绝', icon: '🚫' },
    { value: 'timeout', label: '支付超时', icon: '⏰' },
    { value: 'network_error', label: '网络错误', icon: '🌐' },
    { value: 'fraud', label: '欺诈检测', icon: '⚠️' },
  ];
  
  const handleTestPayment = async () => {
    try {
      setLogs(prev => [...prev, `[${new Date().toISOString()}] 开始测试支付...`]);
      
      const result = await api.post('/dev/payment/test', {
        scenario,
        amount: 100.00,
        currency: 'CNY',
        provider: 'sandbox',
      });
      
      setLogs(prev => [
        ...prev,
        `[${new Date().toISOString()}] 支付结果: ${JSON.stringify(result.data)}`,
      ]);
    } catch (err: any) {
      setLogs(prev => [
        ...prev,
        `[${new Date().toISOString()}] 错误: ${err.message}`,
      ]);
    }
  };
  
  return (
    <div className="space-y-4">
      <h3 className="font-bold">支付沙箱</h3>
      
      {/* 模拟场景选择 */}
      <div>
        <label className="text-sm text-gray-400 mb-2 block">模拟场景</label>
        <div className="grid grid-cols-2 gap-2">
          {scenarios.map(s => (
            <button
              key={s.value}
              className={`text-left p-2 rounded ${
                scenario === s.value ? 'bg-blue-500' : 'bg-gray-700'
              }`}
              onClick={() => setScenario(s.value)}
            >
              <span>{s.icon}</span>
              <span className="ml-1">{s.label}</span>
            </button>
          ))}
        </div>
      </div>
      
      {/* 测试按钮 */}
      <button
        className="w-full bg-green-500 text-white py-2 rounded font-bold"
        onClick={handleTestPayment}
      >
        测试支付
      </button>
      
      {/* 测试日志 */}
      <div className="bg-gray-800 p-2 rounded max-h-40 overflow-auto">
        <div className="text-xs font-bold text-gray-400 mb-1">日志</div>
        {logs.map((log, i) => (
          <div key={i} className="text-xs font-mono">{log}</div>
        ))}
      </div>
    </div>
  );
}
```

### 6. API调试工具

```typescript
// components/DevPanel/components/ApiInspector.tsx

import { useState, useEffect } from 'react';
import { useDevMode } from '@/hooks/useDevMode';

interface ApiRequest {
  id: string;
  timestamp: string;
  method: string;
  url: string;
  status: number;
  duration: number;
  request?: any;
  response?: any;
}

export function ApiInspector() {
  const { features, setFeature } = useDevMode();
  const [requests, setRequests] = useState<ApiRequest[]>([]);
  const [selectedRequest, setSelectedRequest] = useState<ApiRequest | null>(null);
  
  // Hook 拦截所有 fetch 请求
  useEffect(() => {
    const originalFetch = window.fetch;
    
    window.fetch = async (...args) => {
      const id = Math.random().toString(36).substr(2, 9);
      const startTime = Date.now();
      const url = typeof args[0] === 'string' ? args[0] : args[0].url;
      const method = args[1]?.method || 'GET';
      
      try {
        const response = await originalFetch(...args);
        const duration = Date.now() - startTime;
        
        // 复制响应体（不消费原始流）
        const clone = response.clone();
        const responseBody = await clone.text();
        
        const apiRequest: ApiRequest = {
          id,
          timestamp: new Date().toISOString(),
          method,
          url,
          status: response.status,
          duration,
          request: args[1]?.body ? JSON.parse(args[1].body) : null,
          response: responseBody ? JSON.parse(responseBody) : null,
        };
        
        setRequests(prev => [apiRequest, ...prev].slice(0, 100)); // 最多100条
        
        return response;
      } catch (err) {
        const duration = Date.now() - startTime;
        setRequests(prev => [
          {
            id,
            timestamp: new Date().toISOString(),
            method,
            url,
            status: 0,
            duration,
            response: { error: (err as Error).message },
          },
          ...prev,
        ]);
        throw err;
      }
    };
    
    return () => {
      window.fetch = originalFetch;
    };
  }, []);
  
  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center">
        <h3 className="font-bold">API调试</h3>
        <div className="flex gap-2">
          <label className="flex items-center text-xs">
            <input
              type="checkbox"
              checked={features.delayAllRequests > 0}
              onChange={(e) => setFeature('delayAllRequests', e.target.checked ? 1000 : 0)}
              className="mr-1"
            />
            慢网络
          </label>
          <button
            className="text-xs bg-red-500 px-2 py-1 rounded"
            onClick={() => setRequests([])}
          >
            清除
          </button>
        </div>
      </div>
      
      {/* 请求列表 */}
      <div className="space-y-1 max-h-60 overflow-auto">
        {requests.map(req => (
          <div
            key={req.id}
            className={`text-xs p-2 rounded cursor-pointer ${
              selectedRequest?.id === req.id ? 'bg-gray-700' : 'bg-gray-800'
            }`}
            onClick={() => setSelectedRequest(req)}
          >
            <span className={`font-bold ${
              req.status >= 200 && req.status < 300 ? 'text-green-400' :
              req.status >= 400 ? 'text-red-400' : 'text-yellow-400'
            }`}>
              {req.status || 'ERR'}
            </span>
            <span className="ml-2 text-gray-400">{req.method}</span>
            <span className="ml-2">{req.url.split('/').pop()}</span>
            <span className="ml-2 text-gray-500">{req.duration}ms</span>
          </div>
        ))}
      </div>
      
      {/* 选中请求详情 */}
      {selectedRequest && (
        <div className="bg-gray-800 p-2 rounded text-xs">
          <div className="font-bold mb-1">请求详情</div>
          <pre className="overflow-auto max-h-40">
            {JSON.stringify(selectedRequest.request, null, 2)}
          </pre>
          <div className="font-bold mb-1 mt-2">响应</div>
          <pre className="overflow-auto max-h-40">
            {JSON.stringify(selectedRequest.response, null, 2)}
          </pre>
        </div>
      )}
    </div>
  );
}
```

### 7. 多语言测试工具

```typescript
// components/DevPanel/components/I18nTester.tsx

import { useState } from 'react';
import { useDevMode } from '@/hooks/useDevMode';
import { useI18n } from '@/modules/i18n';
import { i18nConfig } from '@/modules/i18n/config';

export function I18nTester() {
  const { locale, setLocale, dir } = useI18n();
  const { features, setFeature } = useDevMode();
  const [previewText, setPreviewText] = useState('hello');
  
  return (
    <div className="space-y-4">
      <h3 className="font-bold">多语言测试</h3>
      
      {/* 当前语言信息 */}
      <div className="bg-gray-800 p-2 rounded text-xs">
        <div>当前语言: {locale}</div>
        <div>文本方向: {dir}</div>
        <div>RTL模式: {dir === 'rtl' ? '是' : '否'}</div>
      </div>
      
      {/* 快速切换语言 */}
      <div>
        <label className="text-sm text-gray-400 mb-2 block">切换语言</label>
        <div className="grid grid-cols-2 gap-2">
          {i18nConfig.supportedLocales.map(l => (
            <button
              key={l}
              className={`p-2 rounded text-left text-xs ${
                locale === l ? 'bg-blue-500' : 'bg-gray-700'
              }`}
              onClick={() => setLocale(l as any)}
            >
              {l}
              {i18nConfig.rtlLocales.includes(l) && (
                <span className="ml-1 text-yellow-400">RTL</span>
              )}
            </button>
          ))}
        </div>
      </div>
      
      {/* RTL模拟器 */}
      <div>
        <label className="flex items-center text-sm">
          <input
            type="checkbox"
            checked={features.rtlSimulator}
            onChange={(e) => setFeature('rtlSimulator', e.target.checked)}
            className="mr-2"
          />
          强制RTL模式（测试布局镜像）
        </label>
      </div>
      
      {/* 文本预览 */}
      <div>
        <label className="text-sm text-gray-400 mb-2 block">预览文本</label>
        <input
          type="text"
          value={previewText}
          onChange={(e) => setPreviewText(e.target.value)}
          className="w-full px-3 py-2 bg-gray-700 rounded text-sm"
          placeholder="输入要翻译的文本..."
        />
        <div className="mt-2 p-2 bg-gray-800 rounded text-sm">
          {previewText || '在此显示翻译结果'}
        </div>
      </div>
      
      {/* 布局预览 */}
      <div className="bg-white text-black p-4 rounded">
        <div className="text-sm font-bold mb-2">布局预览</div>
        <div className="flex justify-between items-center">
          <div>左边</div>
          <div>右边</div>
        </div>
        <div className="flex">
          <div className="mr-2">margin-right</div>
          <div>margin-left</div>
        </div>
        <div className="text-left">文本对齐</div>
      </div>
    </div>
  );
}
```

---

## 后端支持（沙箱路由）

```typescript
// app/api/dev/impersonate/route.ts

import { NextResponse } from 'next/server';
import { auth } from '@/lib/auth';

export async function POST(request: Request) {
  // 只在开发环境可用
  if (process.env.NODE_ENV !== 'development') {
    return NextResponse.json({ error: 'Not available in production' }, { status: 403 });
  }
  
  const { userId } = await request.json();
  
  // 生成临时session，伪装成目标用户
  const session = await auth.impersonate(userId);
  
  return NextResponse.json({ 
    success: true, 
    message: `Now impersonating user: ${userId}`,
    sessionId: session.id,
  });
}

// app/api/dev/payment/test/route.ts

export async function POST(request: Request) {
  if (process.env.NODE_ENV !== 'development') {
    return NextResponse.json({ error: 'Not available in production' }, { status: 403 });
  }
  
  const { scenario, amount, currency, provider } = await request.json();
  
  // 模拟支付流程，不调用真实支付接口
  const result = simulatePayment(scenario, amount, currency);
  
  // 记录测试日志
  console.log('[Payment Sandbox]', { scenario, amount, currency, result });
  
  return NextResponse.json(result);
}

function simulatePayment(scenario: string, amount: number, currency: string) {
  const scenarios: Record<string, any> = {
    success: { success: true, transactionId: `TEST_${Date.now()}`, status: 'completed' },
    insufficient_funds: { success: false, error: 'Insufficient funds', status: 'failed' },
    card_declined: { success: false, error: 'Card declined', status: 'failed' },
    timeout: { success: false, error: 'Payment timeout', status: 'timeout' },
    network_error: { success: false, error: 'Network error', status: 'error' },
    fraud: { success: false, error: 'Fraud detected', status: 'blocked' },
  };
  
  return scenarios[scenario] || scenarios.success;
}
```

---

## 使用指南

### 开启开发者模式

**方法1: URL参数**
```
http://localhost:3000?dev=true
```

**方法2: localStorage**
```javascript
localStorage.setItem('devMode', 'true');
window.location.reload();
```

**方法3: 按钮（仅在特定条件）**
```typescript
// 点击应用Logo 5次开启
let clickCount = 0;
const handleLogoClick = () => {
  clickCount++;
  if (clickCount >= 5) {
    localStorage.setItem('devMode', 'true');
    window.location.reload();
  }
};
```

### 快捷键

```typescript
// 按 Ctrl+Shift+D 打开开发者面板
document.addEventListener('keydown', (e) => {
  if (e.ctrlKey && e.shiftKey && e.key === 'D') {
    e.preventDefault();
    setDevPanelOpen(true);
  }
});
```
