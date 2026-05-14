---
name: i18n-setup
description: 国际化(i18n)配置指南：多语言支持、RTL、动态文本管理
---

# 国际化(i18n)配置指南

## 核心原则

**国际化是地基，不是后期加装的装饰。**

每个项目启动前必须确认语言需求。中途加入i18n成本指数级上升。

---

## 早期决策检查清单

### 基础问题

- [ ] 首发地区：__
- [ ] 支持语言清单：__
- [ ] 是否需要 RTL (阿拉伯语、希伯来语等)：__
- [ ] 是否支持日期格式变化：__
- [ ] 是否支持货币格式变化：__
- [ ] 静态文本还是动态内容：__
- [ ] 翻译资源存储方式：__

---

## 目录结构

### 推荐结构

```
src/
  modules/
    i18n/
      config.ts              # i18n配置
      utils.ts               # 辅助函数
      types.ts               # 类型定义
      hooks.ts               # React hooks
      
      locales/               # 翻译文件
        en-US/               # 英文
          common.json
          auth.json
          payment.json
          validation.json
        zh-CN/               # 简体中文
          common.json
          auth.json
          payment.json
          validation.json
        zh-TW/               # 繁体中文
        ja-JP/               # 日文
        ko-KR/               # 韩文
        ar-SA/               # 阿拉伯语 (RTL)
          common.json
          auth.json
          # RTL语言需特殊处理
      
      middleware.ts          # 语言检测/切换逻辑
      provider.tsx           # React Provider
      
  shared/
    ui/                      # 基础UI组件
      locale-switcher/       # 语言切换组件
        index.tsx
      rtl-wrapper/           # RTL包裹器
        index.tsx
```

---

## 配置实现

### 核心配置

```typescript
// modules/i18n/config.ts

export interface I18nConfig {
  defaultLocale: string;
  supportedLocales: string[];
  rtlLocales: string[];  // RTL语言列表
  fallbackLocale: string;
  
  // 货币和数字格式
  localeFormats: Record<string, LocaleFormat>;
  
  // 动态内容配置
  dynamicContent?: {
    enabled: boolean;
    source: 'database' | 'cms' | 'api';
    cacheDuration: number;
  };
}

export const i18nConfig: I18nConfig = {
  defaultLocale: 'zh-CN',
  supportedLocales: ['zh-CN', 'en-US', 'zh-TW', 'ja-JP', 'ko-KR', 'ar-SA'],
  rtlLocales: ['ar-SA', 'he-IL'],  // 从右到左语言
  fallbackLocale: 'en-US',
  
  localeFormats: {
    'zh-CN': {
      currency: 'CNY',
      currencyDisplay: '¥',
      dateFormat: 'YYYY-MM-DD',
      timeFormat: 'HH:mm:ss',
      numberFormat: '0,0.00',
      firstDayOfWeek: 1,  // 周一
    },
    'en-US': {
      currency: 'USD',
      currencyDisplay: '$',
      dateFormat: 'MM/DD/YYYY',
      timeFormat: 'h:mm A',
      numberFormat: '0,0.00',
      firstDayOfWeek: 0,  // 周日
    },
    'ar-SA': {
      currency: 'SAR',
      currencyDisplay: '﷼',
      dateFormat: 'DD/MM/YYYY',
      timeFormat: 'HH:mm:ss',
      numberFormat: '0,0.00',
      firstDayOfWeek: 6,  // 周六
      rtl: true,
    },
  },
  
  dynamicContent: {
    enabled: true,
    source: 'database',
    cacheDuration: 1000 * 60 * 60,  // 1小时
  },
};
```

### 数据库Schema（动态内容）

```typescript
// 如果需要动态管理翻译

// prisma/schema.prisma
model Translation {
  id          String   @id @default(uuid())
  key         String
  locale      String
  namespace   String   // 命名空间: common, auth, payment...
  value       String   @db.Text
  description String?  // 翻译注释
  updatedAt   DateTime @updatedAt
  updatedBy   String?  // 翻译人员
  
  @@unique([key, locale, namespace])
  @@index([locale, namespace])
}

// 等待翻译的键（自动识别）
model TranslationKey {
  id          String   @id @default(uuid())
  key         String   @unique
  namespace   String
  description String?
  createdAt   DateTime @default(now())
  status      TranslationStatus @default(PENDING)
}

enum TranslationStatus {
  PENDING     // 等待翻译
  TRANSLATED  // 已翻译
  REVIEWED    // 已审核
}
```

---

## 翻译文件格式

### 结构示例

```json
// modules/i18n/locales/zh-CN/common.json
{
  "app": {
    "name": "我的应用",
    "slogan": "简洁、强大、可扩展"
  },
  "actions": {
    "save": "保存",
    "cancel": "取消",
    "delete": "删除",
    "edit": "编辑",
    "confirm": "确认"
  },
  "errors": {
    "generic": "出现了错误，请稍后重试",
    "network": "网络连接失败",
    "unauthorized": "您没有权限执行此操作"
  }
}

// modules/i18n/locales/zh-CN/validation.json
{
  "required": "{{field}}不能为空",
  "minLength": "{{field}}至少需要{{min}}个字符",
  "maxLength": "{{field}}不能超过{{max}}个字符",
  "email": "请输入有效的邮箱地址",
  "phone": "请输入有效的手机号码"
}
```

### 多等级命名空间

```json
// 避免平铺，使用层级结构
{
  "payment": {
    "method": {
      "alipay": "支付宝",
      "wechat": "微信支付",
      "card": "信用卡"
    },
    "status": {
      "pending": "等待支付",
      "completed": "支付成功",
      "failed": "支付失败"
    }
  }
}

// 使用: t('payment.method.alipay')
```

---

## React 整合

### Hook 实现

```typescript
// modules/i18n/hooks.ts

import { createContext, useContext, useEffect, useState } from 'react';
import { i18nConfig } from './config';
import type { Locale, TranslationKey } from './types';

interface I18nContextType {
  locale: Locale;
  setLocale: (locale: Locale) => void;
  t: (key: TranslationKey, params?: Record<string, string | number>) => string;
  dir: 'ltr' | 'rtl';
  formatDate: (date: Date, options?: Intl.DateTimeFormatOptions) => string;
  formatCurrency: (amount: number) => string;
  formatNumber: (num: number) => string;
}

const I18nContext = createContext<I18nContextType | null>(null);

export function useI18n() {
  const context = useContext(I18nContext);
  if (!context) throw new Error('useI18n must be used within I18nProvider');
  return context;
}

// Provider 实现
export function I18nProvider({ children }: { children: React.ReactNode }) {
  const [locale, setLocale] = useState<Locale>(() => {
    // 检测浏览器语言
    const detected = detectLocale();
    return i18nConfig.supportedLocales.includes(detected) 
      ? detected 
      : i18nConfig.defaultLocale;
  });
  
  const [translations, setTranslations] = useState<Record<string, any>>({});
  
  // 加载翻译
  useEffect(() => {
    loadTranslations(locale).then(setTranslations);
    
    // 如果是动态内容，后台获取
    if (i18nConfig.dynamicContent?.enabled) {
      loadDynamicTranslations(locale).then(dynamic => {
        setTranslations(prev => ({ ...prev, ...dynamic }));
      });
    }
  }, [locale]);
  
  const t = (key: string, params?: Record<string, string | number>) => {
    const value = getNestedValue(translations, key) || key;
    if (!params) return value;
    
    // 替换占位符 {{field}}
    return value.replace(/\{\{(\w+)\}\}/g, (_, k) => String(params[k] ?? `{{${k}}}`));
  };
  
  const dir = i18nConfig.rtlLocales.includes(locale) ? 'rtl' : 'ltr';
  
  const formatDate = (date: Date, options?: Intl.DateTimeFormatOptions) => {
    return new Intl.DateTimeFormat(locale, options).format(date);
  };
  
  const formatCurrency = (amount: number) => {
    const format = i18nConfig.localeFormats[locale];
    return new Intl.NumberFormat(locale, {
      style: 'currency',
      currency: format.currency,
    }).format(amount);
  };
  
  const formatNumber = (num: number) => {
    return new Intl.NumberFormat(locale).format(num);
  };
  
  return (
    <I18nContext.Provider value={{ 
      locale, 
      setLocale, 
      t, 
      dir,
      formatDate,
      formatCurrency,
      formatNumber,
    }}>
      <div dir={dir} lang={locale}>
        {children}
      </div>
    </I18nContext.Provider>
  );
}
```

### RTL 支持

```typescript
// modules/i18n/rtl-utils.ts

import { i18nConfig } from './config';

/**
 * 检测是否需要RTL布局
 */
export function isRTL(locale: string): boolean {
  return i18nConfig.rtlLocales.includes(locale);
}

/**
 * RTL Wrapper 组件
 */
export function RTLWrapper({ children, locale }: { 
  children: React.ReactNode; 
  locale: string;
}) {
  const rtl = isRTL(locale);
  
  useEffect(() => {
    // 设置HTML dir属性
    document.documentElement.dir = rtl ? 'rtl' : 'ltr';
    document.documentElement.lang = locale;
    
    // 加载RTL样式补丁（如果需要）
    if (rtl) {
      loadRTLStyles();
    }
  }, [locale, rtl]);
  
  return (
    <div 
      dir={rtl ? 'rtl' : 'ltr'}
      className={rtl ? 'rtl-mode' : 'ltr-mode'}
    >
      {children}
    </div>
  );
}

/**
 * 镜像方法（仅对特定属性）
 */
export function mirrorCSSProperties(css: CSSProperties, isRTL: boolean): CSSProperties {
  if (!isRTL) return css;
  
  const mirrored: CSSProperties = { ...css };
  
  // 镜像边距
  if (css.marginLeft !== undefined) {
    mirrored.marginRight = css.marginLeft;
    delete mirrored.marginLeft;
  }
  if (css.marginRight !== undefined) {
    mirrored.marginLeft = css.marginRight;
    delete mirrored.marginRight;
  }
  
  if (css.paddingLeft !== undefined) {
    mirrored.paddingRight = css.paddingLeft;
    delete mirrored.paddingLeft;
  }
  if (css.paddingRight !== undefined) {
    mirrored.paddingLeft = css.paddingRight;
    delete mirrored.paddingRight;
  }
  
  // 镜像定位
  if (css.left !== undefined) {
    mirrored.right = css.left;
    delete mirrored.left;
  }
  if (css.right !== undefined) {
    mirrored.left = css.right;
    delete mirrored.right;
  }
  
  // 文本对齐
  if (css.textAlign === 'left') {
    mirrored.textAlign = 'right';
  } else if (css.textAlign === 'right') {
    mirrored.textAlign = 'left';
  }
  
  return mirrored;
}
```

---

## 最佳实践

### 1. 早期就要整合

```typescript
// app/layout.tsx
export default function RootLayout({ children }) {
  return (
    <I18nProvider>
      {/* 所有内容在i18n环境中 */}
      {children}
    </I18nProvider>
  );
}
```

### 2. 不要硬编码文本

```typescript
// ❌ 禁止
<button>保存</button>
<span>发送验证码</span>

// ✅ 正确
const { t } = useI18n();
<button>{t('actions.save')}</button>
<span>{t('auth.sendVerification')}</span>
```

### 3. 处理复数和性别

```typescript
// 提供复数支持
{
  "itemsCount": {
    "one": "{{count}} 件商品",
    "other": "{{count}} 件商品"
  },
  "welcomeMessage": {
    "male": "欢迎回来，{{name}}先生",
    "female": "欢迎回来，{{name}}女士"
  }
}

// 使用
const { t, locale } = useI18n();
const count = items.length;

// 复数处理
t('itemsCount', { count }, { count });

// 性别处理
t('welcomeMessage', { name, gender: user.gender });
```

### 4. 图片多语言化

```typescript
// 如果图片包含文字
interface LocalizedImage {
  src: string;
  alt: string;
  locale: string;
}

const bannerImages: LocalizedImage[] = [
  { src: '/banner-zh.jpg', alt: '官方应用', locale: 'zh-CN' },
  { src: '/banner-en.jpg', alt: 'Official App', locale: 'en-US' },
  { src: '/banner-ar.jpg', alt: 'التطبيق الرسمي', locale: 'ar-SA' },
];

// 使用
function LocalizedBanner({ locale }: { locale: string }) {
  const image = bannerImages.find(i => i.locale === locale) 
    || bannerImages[0];
  
  return <img src={image.src} alt={image.alt} />;
}
```

### 5. 路由国际化

```typescript
// next.config.js
module.exports = {
  i18n: {
    locales: ['zh-CN', 'en-US', 'ja-JP', 'ar-SA'],
    defaultLocale: 'zh-CN',
    domains: [
      {
        domain: 'example.com',
        defaultLocale: 'en-US',
      },
      {
        domain: 'example.cn',
        defaultLocale: 'zh-CN',
      },
    ],
  },
};
```

---

## 测试检查清单

- [ ] 每个语言的所有文本都已翻译
- [ ] RTL布局正确显示
- [ ] 日期格式按语言显示
- [ ] 货币格式按语言显示
- [ ] 数字格式按语言显示
- [ ] 图片文字也已本地化
- [ ] 邮件/通知模板已本地化
- [ ] SEO meta 信息已本地化
