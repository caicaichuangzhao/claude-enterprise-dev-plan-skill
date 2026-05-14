---
name: enterprise-dev-plan
description: 企业级开发规划技能：扩展性、国际化、模块化、二次开发支持、开发者体验设计
---

# 企业级开发规划 (Enterprise Development Planning)

## 核心原则

**先规划，后开发。先聊透底层，再动工。**

越是底层基础的决策，后期改动代价越高。地基没打好，楼建起来后很难重建。

## 目录

- [references/checklist.md](references/checklist.md) - 项目启动前必确认清单
- [references/tech-stack-guide.md](references/tech-stack-guide.md) - 技术栈选型指南
- [references/architecture-patterns.md](references/architecture-patterns.md) - 架构设计模式
- [references/i18n-setup.md](references/i18n-setup.md) - 国际化配置指南
- [references/dev-mode-design.md](references/dev-mode-design.md) - 开发者模式设计

---

## 快速开始

### 开发前必须确认的问题

| 维度 | 关键问题 | 后期改动成本 |
|------|---------|------------|
| **语言/地区** | 多国家/多语言？RTL支持？ | 指数级上升 |
| **支付渠道** | 哪些支付方式？沙箱测试？ | 架构重构 |
| **部署环境** | 单服/分布式/云原生？ | 技术栈选择 |
| **终端形态** | Web/原生App/小程序/桌面？ | 是否共享API |
| **数据规模** | 预估用户数、增长曲线 | 数据库选型 |
| **合规要求** | GDPR/数据本地化/审计？ | 加密策略 |
| **第三方集成** | 需要对接外部服务？ | API抽象层 |
| **开发者体验** | 需要开发者模式/调试面板？ | 配置系统设计 |

### 执行口诀

1. **聊透再动工** - 底层决策错了，后面都是屎山
2. **模块化优先** - 堆在一起就是灾难的开始
3. **配置化一切** - 能配置的不硬编码，能开关的不写死
4. **开发者体验** - 调试工具和沙箱是效率倍增器
5. **国际化早做** - 中途改成本指数级上升
6. **文档即代码** - 注释写清楚，扩展点标明白
7. **为未来留白** - 预留20%的扩展空间

---

## 模块化强制规范

```
❌ 禁止做法
src/
  components/     (几百个文件堆在一起)
  utils/          (工具函数大杂烩)
  pages/          (页面组件+逻辑+样式混在一起)

✅ 推荐做法
src/
  modules/
    auth/              ← 独立模块
      components/      ← 组件只属于auth
      api/             ← API调用
      hooks/           ← 状态管理
      types/           ← 类型定义
      constants/       ← 常量
      index.ts         ← 统一导出，外部只通过这里访问
    payment/
      components/
      api/
      hooks/
      types/
      constants/
    i18n/
      locales/         ← 翻译文件
      config.ts
  shared/            ← 真正公共的（仅跨模块使用）
    ui/              ← 基础UI组件库
    lib/             ← 通用工具
    types/           ← 全局类型
```

---

## 配置文件分层

```typescript
// config/index.ts
import { devConfig } from './env.development';
import { prodConfig } from './env.production';

export const config = {
  // 基础配置（所有环境共享）
  appName: 'MyApp',
  apiVersion: 'v1',
  supportedLocales: ['zh-CN', 'en-US', 'ja-JP'],
  
  // 环境特定配置
  ...(process.env.NODE_ENV === 'production' ? prodConfig : devConfig),
  
  // 开发者模式开关
  devMode: {
    enabled: checkDevMode(), // ?dev=true 或 localStorage.devMode
    features: {
      mockPayment: true,     // 支付沙箱
      showDebugPanel: true,  // 调试面板
      logLevel: 'debug',     // 详细日志
      skipAuth: false,       // 可选跳过认证
    }
  }
};
```

---

## 文件注释规范

```typescript
/**
 * @module PaymentGateway
 * @description 支付网关核心模块，处理多渠道支付统一接入
 * 
 * @author Claude
 * @lastModified 2026-05-15
 * 
 * @dependencies
 *   - Stripe SDK (v12.x)
 *   - Alipay SDK
 *   - WeChat Pay SDK
 * 
 * @important
 *   1. 新增支付方式需要在 PaymentProvider 枚举中注册
 *   2. 回调URL必须在 payment.config.ts 中配置
 *   3. 沙箱测试通过 PaymentSandbox.enable() 开启
 * 
 * @todo
 *   - [ ] 支持加密货币支付
 *   - [ ] 接入 Apple Pay
 * 
 * @example
 * ```typescript
 * const gateway = new PaymentGateway();
 * await gateway.initiate({
 *   provider: PaymentProvider.ALIPAY,
 *   amount: 100,
 *   currency: 'CNY'
 * });
 * ```
 */
```

---

## 扩展点设计

```typescript
// 二次开发扩展点明确标记
/**
 * @extensionPoint
 * 自定义认证策略可实现此接口
 */
export interface CustomAuthStrategy {
  authenticate(credentials: unknown): Promise<AuthResult>;
}

// 插件系统
export interface Plugin {
  name: string;
  version: string;
  hooks: { [key: string]: HookHandler };
  components?: { [slot: string]: React.ComponentType };
}
```

---

## 2026 技术栈推荐

### Web 应用

| 场景 | 推荐 | 理由 |
|------|------|------|
| 企业后台 | Next.js + Prisma + PostgreSQL | 全栈统一、类型安全 |
| 内容网站 | Astro + Markdown/CMS | 极致性能、SEO友好 |
| 实时应用 | Next.js + Socket.io + Redis | 实时同步、状态共享 |
| 小程序 | Taro / UniApp | 跨端一套代码 |

### 移动端

| 场景 | 推荐 | 理由 |
|------|------|------|
| 高交互原生App | Flutter | 跨平台、性能接近原生 |
| 快速MVP | React Native + Expo | 开发快、热更新 |
| 轻量应用 | PWA (Capacitor) | 无需应用商店审核 |
