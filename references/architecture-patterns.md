---
name: architecture-patterns
description: 企业级架构设计模式：模块化、配置分层、扩展点、插件系统
---

# 架构设计模式

## 1. 模块化强制要求

### 禁止的目录结构

```
src/
  components/     # 几百个文件堆在一起，找不到属于哪个功能
  utils/          # 工具函数大杂烩，谁都放
  pages/          # 页面组件+逻辑+样式混在一起
  api/            # 所有API函数在一起
  hooks/          # 所有hooks在一起
```

### 推荐的模块化结构

```
src/
  modules/                    # 功能模块目录
    auth/                     # 认证模块（完全独立）
      components/
        LoginForm/
          index.tsx
          style.module.css
          types.ts
        UserProfile/
      api/
        login.ts
        register.ts
        refreshToken.ts
      hooks/
        useAuth.ts
        usePermissions.ts
      types/
        user.ts
        auth.ts
      constants/
        roles.ts
        permissions.ts
      utils/
        tokenHelper.ts
      store/                    # 如果需要状态管理
        authStore.ts
      index.ts                   # 统一导出点
      
    payment/                    # 支付模块（完全独立）
      components/
      api/
      hooks/
      types/
      constants/
      sandbox/                   # 支付沙箱（只有支付模块知道）
      index.ts
      
    i18n/                       # 国际化模块
      locales/
        zh-CN/
          common.json
          auth.json
          payment.json
        en-US/
          common.json
          auth.json
          payment.json
      config.ts
      utils.ts
      index.ts
      
    products/                   # 商品模块
    orders/                     # 订单模块
    cart/                       # 购物车模块
      
  shared/                       # 真正公共的（仅跨模块使用）
    ui/                         # 基础UI组件库
      Button/
      Input/
      Modal/
      index.ts
    lib/                        # 通用工具
      date.ts
      validate.ts
      format.ts
    types/                      # 全局类型
      index.ts
    constants/                  # 全局常量
      app.ts
```

### 模块导出规范

```typescript
// modules/auth/index.ts

// 公开API - 外部只能使用这些
export { AuthProvider } from './AuthProvider';
export { LoginForm } from './components/LoginForm';
export { UserProfile } from './components/UserProfile';
export { useAuth } from './hooks/useAuth';
export { usePermissions } from './hooks/usePermissions';
export type { User, LoginCredentials, AuthState } from './types';
export { UserRole, Permission } from './constants';

// 私有API - 不导出，模块内部使用
// 不导出: tokenHelper.ts 里的内部方法
// 不导出: 具体实现的API调用
```

---

## 2. 配置文件分层

### 三层配置架构

```typescript
// config/types.ts
export interface AppConfig {
  // 基础配置（所有环境共享）
  appName: string;
  appVersion: string;
  supportedLocales: string[];
  apiVersion: string;
  
  // 环境特定配置
  apiBaseUrl: string;
  cdnUrl: string;
  enableAnalytics: boolean;
  
  // 开发者模式
  devMode: DevModeConfig;
}

// config/base.ts - 基础配置
export const baseConfig = {
  appName: 'MyApp',
  appVersion: '1.0.0',
  supportedLocales: ['zh-CN', 'en-US', 'ja-JP'],
  apiVersion: 'v1',
};

// config/env.development.ts - 开发环境
export const devConfig = {
  apiBaseUrl: 'http://localhost:3000/api',
  cdnUrl: 'http://localhost:3000/static',
  enableAnalytics: false,
  devMode: {
    enabled: true,
    features: {
      mockPayment: true,
      showDebugPanel: true,
      logLevel: 'debug',
      skipAuth: true,
    }
  }
};

// config/env.production.ts - 生产环境
export const prodConfig = {
  apiBaseUrl: 'https://api.myapp.com',
  cdnUrl: 'https://cdn.myapp.com',
  enableAnalytics: true,
  devMode: {
    enabled: false,  // 生产环境关闭
    features: {
      mockPayment: false,
      showDebugPanel: false,
      logLevel: 'error',
      skipAuth: false,
    }
  }
};

// config/index.ts - 合并配置
import { baseConfig } from './base';
import { devConfig } from './env.development';
import { prodConfig } from './env.production';

function checkDevMode(): boolean {
  if (typeof window === 'undefined') return false;
  const urlParams = new URLSearchParams(window.location.search);
  return urlParams.get('dev') === 'true' || 
         localStorage.getItem('devMode') === 'true';
}

export const config: AppConfig = {
  ...baseConfig,
  ...(process.env.NODE_ENV === 'production' ? prodConfig : devConfig),
  // 可覆盖：URL参数 > localStorage > 构建配置
  devMode: {
    ...((process.env.NODE_ENV === 'production' ? prodConfig : devConfig).devMode),
    enabled: checkDevMode(),
  }
};
```

---

## 3. 扩展点设计

### 模块内扩展点

```typescript
// modules/auth/index.ts

/**
 * @extensionPoint 自定义认证策略
 * 二次开发时可实现此接口添加新的认证方式
 */
export interface CustomAuthStrategy {
  name: string;
  authenticate(credentials: unknown): Promise<AuthResult>;
  validateSession(session: unknown): Promise<boolean>;
}

// 用于注册自定义策略
const customStrategies: CustomAuthStrategy[] = [];

export function registerAuthStrategy(strategy: CustomAuthStrategy) {
  customStrategies.push(strategy);
}
```

### 插件系统架构

```typescript
// core/plugin-system.ts

export interface Plugin {
  name: string;
  version: string;
  dependencies?: string[];  // 依赖的其他插件
  
  // 生命周期钩子
  onInstall?: () => Promise<void>;
  onUninstall?: () => Promise<void>;
  onEnable?: () => void;
  onDisable?: () => void;
  
  // 扩展点注册
  hooks: {
    [key: string]: HookHandler;
  };
  
  // UI注入点
  components?: {
    [slot: string]: React.ComponentType<any>;
  };
}

// 使用示例：自定义支付插件
export const customPaymentPlugin: Plugin = {
  name: 'custom-payment',
  version: '1.0.0',
  dependencies: ['payment-core'],
  
  onInstall: async () => {
    console.log('Installing custom payment plugin');
  },
  
  hooks: {
    'payment:beforeProcess': async (data: PaymentData) => {
      // 自定义处理逻辑
      return { ...data, customField: 'value' };
    },
    'payment:afterProcess': async (result: PaymentResult) => {
      // 支付后处理
      return result;
    }
  },
  
  components: {
    'checkout:extraFields': CustomFieldComponent,
    'checkout:customButton': CustomPaymentButton,
  }
};
```

### Hook 系统实现

```typescript
// core/hook-system.ts

class HookSystem {
  private hooks: Map<string, HookHandler[]> = new Map();
  
  register(hookName: string, handler: HookHandler) {
    if (!this.hooks.has(hookName)) {
      this.hooks.set(hookName, []);
    }
    this.hooks.get(hookName)!.push(handler);
  }
  
  async execute(hookName: string, context: unknown) {
    const handlers = this.hooks.get(hookName) || [];
    let result = context;
    
    for (const handler of handlers) {
      result = await handler(result);
    }
    
    return result;
  }
}

// 使用
const hooks = new HookSystem();

// 主模块注册钩子
hooks.register('payment:beforeProcess', async (data) => {
  // 验证数据
  return data;
});

// 插件注册钩子
hooks.register('payment:beforeProcess', async (data) => {
  // 插件自定义逻辑
  return { ...data, pluginProcessed: true };
});
```

---

## 4. 配置化设计

### Feature Flags

```typescript
// config/feature-flags.ts

export const featureFlags = {
  // 简单开关
  enableNewCheckout: process.env.FEATURE_NEW_CHECKOUT === 'true',
  
  // 动态判断
  enableBetaFeatures: (user: User) => user.role === 'beta_tester',
  
  // 逐步放量（按用户ID或地区）
  enableNewDashboard: (user: User) => {
    const rolloutPercentage = 10; // 10%用户
    const hash = hashUserId(user.id);
    return hash % 100 < rolloutPercentage;
  },
  
  // 远程配置（可动态更新）
  enableRemoteFeature: await fetchRemoteConfig('new_feature'),
};
```

### 可配置的模块

```typescript
// modules/payment/config.ts

export interface PaymentProviderConfig {
  id: string;
  name: string;
  enabled: boolean;
  sandbox: boolean;
  apiConfig: {
    baseUrl: string;
    sandboxUrl?: string;
    timeout: number;
  };
}

// 可通过配置文件或数据库动态更新
export const paymentProviders: PaymentProviderConfig[] = [
  {
    id: 'alipay',
    name: '支付宝',
    enabled: true,
    sandbox: true,
    apiConfig: {
      baseUrl: 'https://openapi.alipay.com',
      sandboxUrl: 'https://openapi.alipaydev.com',
      timeout: 30000,
    }
  },
  {
    id: 'wechat',
    name: '微信支付',
    enabled: true,
    sandbox: true,
    apiConfig: {
      baseUrl: 'https://api.mch.weixin.qq.com',
      timeout: 30000,
    }
  },
  // 新增支付方式只需在此添加配置，无需修改核心代码
];
```

---

## 5. 代码分层架构

### 分层结构

```
src/
  presentation/          # 表示层
    components/          # React组件
    pages/               # 页面
    hooks/               # UI hooks
    
  application/           # 应用层
    services/            # 业务服务
    use-cases/           # 用例
    dto/                 # 数据传输对象
    
  domain/                # 领域层
    entities/            # 实体
    repositories/        # 存储层接口
    services/            # 领域服务
    value-objects/       # 值对象
    
  infrastructure/        # 基础设施层
    api/                 # HTTP客户端
    database/            # 数据库实现
    cache/               # 缓存实现
    storage/             # 文件存储
```

### 依赖方向

```
presentation -> application -> domain <- infrastructure
                      ^
                      |
               infrastructure
```

规则：
- Domain 不依赖任何其他层
- 外层依赖内层
- 可以通过接口逆向依赖（依赖注入）
