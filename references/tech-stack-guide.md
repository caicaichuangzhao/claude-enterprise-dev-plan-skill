---
name: tech-stack-guide
description: 2026年技术栈选型决策树与推荐方案
---

# 技术栈选型指南 (2026)

## 选型决策树

### 第一层：终端形态

```
问：需要原生体验吗？
├── 是 → 问：预算充足吗？
│       ├── 是 → Swift (iOS) + Kotlin (Android) 原生开发
│       └── 否 → Flutter (8成效果，单底维护)
└── 否 → 问：需要离线能力吗？
        ├── 是 → PWA / 混合应用 (Capacitor/Ionic)
        └── 否 → 纯 Web 应用
```

### 第二层：Web 框架选择

```
问：项目类型？
├── 内容型网站 → Astro (静态生成、极致性能)
├── 企业后台/CRM → Next.js + TypeScript
├── 电商 → Next.js (支付安全、认证完善)
├── 实时应用(聊天/直播) → Next.js + Socket.io + Redis
└── 数据密集型看板 → React + 独立 API
```

### 第三层：后端/数据库

```
问：数据关系复杂度？
├── 高(多表关联、严格事务) → PostgreSQL + Prisma
├── 中 → 问：需要实时特性吗？
│       ├── 是 → Firebase / Supabase (实时数据、Auth内置)
│       └── 否 → PostgreSQL 或 MySQL
└── 低/灵活 Schema → MongoDB (MVP阶段快速迭代)
```

---

## 常见场景推荐

### 1. Web 应用

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 企业后台 | Next.js 15 + Prisma + PostgreSQL | 全栈统一、类型安全、生态成熟 |
| 内容网站 | Astro + Markdown/CMS | 极致性能、SEO友好、静态部署 |
| 电商平台 | Next.js + Stripe + Prisma | 支付内置、认证完善、SEO |
| 实时应用 | Next.js + Socket.io + Redis | 实时同步、状态共享、认证管理 |
| 数据看板 | React + TanStack Query + 独立 API | 实时更新、前后端解耦 |

### 2. 移动端

| 场景 | 推荐方案 | 理由 |
|------|----------|------|
| 高交互原生App | Flutter | 跨平台、性能接近原生、实时热更新 |
| 快速MVP | React Native + Expo | 开发快、热更新、JavaScript生态 |
| 轻量应用 | PWA (Capacitor) | 无需应用商店、Web+原生能力 |
| 小程序 | Taro / UniApp | 一套代码、多平台小程序 |

### 3. 实时/聊天应用

| 需求 | 推荐 |
|------|------|
| WebSocket 服务 | Socket.io / ws (Node.js) |
| 实时数据 | Supabase Realtime / Firebase |
| 消息队列 | Redis Pub/Sub / RabbitMQ |
| 离线同步 | 自定义 + Service Worker |

---

## 基础设施推荐

### 认证 (Auth)

| 场景 | 推荐 |
|------|------|
| 快速搭建 | Clerk (最简单的集成) |
| 自管数据 | NextAuth / Auth.js (Vercel) |
| 全栈方案 | Supabase Auth (带数据库) |
| 企业级 | Keycloak / Authing |

### 支付 (Payment)

| 场景 | 推荐 |
|------|------|
| 国际 | Stripe (标准) |
| 中国 | Stripe + Alipay SDK + WeChat Pay SDK |
| 沙箱测试 | 必须实现 PaymentSandbox |

### 文件存储

| 场景 | 推荐 |
|------|------|
| 国际 | AWS S3 / Cloudflare R2 |
| 国内 | 阿里OSS / 腾讯COS |
| 简单 | Supabase Storage |

### 缓存 (Cache)

| 场景 | 推荐 |
|------|------|
| 云缓存 | Upstash Redis / Redis Cloud |
| 自部署 | Redis / KeyDB |
| 无服务器 | 应用内缓存 (React Query) |

### 搜索 (Search)

| 场景 | 推荐 |
|------|------|
| 全文搜索 | Meilisearch (自部署) |
| 云方案 | Algolia |
| 简单 | Supabase Full-Text Search |

### 监控与日志

| 场景 | 推荐 |
|------|------|
| 错误监控 | Sentry |
| 性能分析 | Vercel Analytics / Datadog |
| 日志收集 | Logtail / Datadog |

### 部署

| 场景 | 推荐 |
|------|------|
| 前端 | Vercel / Netlify / Cloudflare Pages |
| 全栈 | Railway / Render / Fly.io |
| 自部署 | Docker + Docker Compose |
| 企业级 | Kubernetes |

---

## 批量更新策略 (2026最佳实践)

### 前端

```typescript
// TanStack Query (旧称 React Query)
import { useQuery, useMutation } from '@tanstack/react-query';

// 服务端获取 + 自动缓存 + 背景更新
const { data } = useQuery({
  queryKey: ['products'],
  queryFn: fetchProducts,
  staleTime: 1000 * 60 * 5, // 5分钟内直接用缓存
});

// 优化更新：离线优先 + 后台同步
const mutation = useMutation({
  mutationFn: updateProduct,
  onMutate: async (newData) => {
    // 离线优先更新UI
    await queryClient.cancelQueries(['products']);
    const previous = queryClient.getQueryData(['products']);
    queryClient.setQueryData(['products'], (old) => 
      old.map(p => p.id === newData.id ? newData : p)
    );
    return { previous };
  },
  onError: (err, newData, context) => {
    // 失败回滚
    queryClient.setQueryData(['products'], context.previous);
  }
});
```

### 数据库优化

```prisma
// Prisma 连接池配置
// 生产环境必须配置连接池，防止大并发时崩溃
```

---

## 选型检查清单

在确定技术栈前，确认：

- [ ] 团队技术肤胆是否匹配？
- [ ] 该技术有长期支持吗？
- [ ] 社区生态是否活跃？
- [ ] 能否找到成熟的运维方案？
- [ ] 扩展性能满足未来3-5年吗？
- [ ] 是否需要跨平台？
- [ ] 是否需要原生性能？
- [ ] 选型是否足够简单（避免过度工程）？
