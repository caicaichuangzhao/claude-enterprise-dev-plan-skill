# Claude Code Skill: Enterprise Development Planning

A Claude Code skill for enterprise-grade development planning focusing on scalability, internationalization, modularity, and developer experience.

## Overview

This skill ensures projects are built with **future-proof foundations**. It emphasizes planning before coding, thorough requirement analysis, and architectural decisions that support long-term growth.

## Core Principle

**"Plan thoroughly before coding. The deeper the foundation, the higher you can build."**

## Key Focus Areas

### 1. Pre-Development Requirements

Before writing any code, confirm:

| Dimension | Key Questions | Impact |
|-----------|--------------|--------|
| **Internationalization** | Multi-language? RTL support? | Determines i18n architecture |
| **Payment Channels** | Which providers? Sandbox testing? | Affects payment abstraction layer |
| **Deployment Environment** | Single server / Distributed / Cloud-native? | Influences tech stack choice |
| **Client Platforms** | Web / Native App / Mini-programs? | Affects API design and sharing |
| **Data Scale** | Expected users? Growth curve? | Determines database and caching |
| **Compliance** | GDPR? Data localization? Audit logs? | Affects storage and encryption |
| **Third-party Integrations** | External services needed? | Affects API abstraction |
| **Developer Experience** | Debug panels needed? Sandbox payments? | Affects configuration system |

### 2. Modular Architecture

**Prohibited Structure:**
```
src/
  components/     # Hundreds of files mixed together
  utils/          # Tool functions dumped together
  pages/          # Logic + styles + data mixed
```

**Recommended Structure:**
```
src/
  modules/
    auth/         # Self-contained auth module
      components/
      api/
      hooks/
      types/
      constants/
      index.ts    # Unified export point
    payment/      # Self-contained payment module
      components/
      api/
      hooks/
      sandbox/    # Payment sandbox (only payment knows)
      index.ts
    i18n/         # Internationalization module
      locales/
  shared/         # Truly shared (cross-module only)
    ui/           # Base UI component library
    lib/          # General utilities
    types/        # Global types
```

### 3. Developer Mode Features

```
Developer Panel Tabs:
├── Overview      # System status, quick toggles
├── Users         # User switching / impersonation
├── Payment       # Payment sandbox with scenarios
├── API           # Request/response inspection
├── State         # Store/cache inspection
├── Performance   # Render profiling, bundle analysis
└── i18n          # Language switching, RTL testing
```

### 4. Internationalization (i18n)

**Early Decisions Checklist:**
- [ ] Which languages? (including RTL like Arabic, Hebrew)
- [ ] Date/currency formats per locale?
- [ ] Text storage: JSON files / Database / CMS?
- [ ] Dynamic content management?

**Directory Structure:**
```
src/modules/i18n/
├── locales/
│   ├── zh-CN/
│   │   ├── common.json
│   │   ├── auth.json
│   │   └── payment.json
│   ├── en-US/
│   ├── ja-JP/
│   └── ar-SA/      # RTL language
├── config.ts
├── hooks.ts
└── utils/
```

### 5. Payment Sandbox

```typescript
// Simulates payment flows without charging
const scenarios = [
  { value: 'success', label: 'Payment Success' },
  { value: 'insufficient_funds', label: 'Insufficient Funds' },
  { value: 'card_declined', label: 'Card Declined' },
  { value: 'timeout', label: 'Payment Timeout' },
  { value: 'network_error', label: 'Network Error' },
  { value: 'fraud', label: 'Fraud Detection' },
];
```

### 6. Technology Stack Recommendations (2026)

| Scenario | Recommendation | Reason |
|----------|---------------|--------|
| Enterprise Admin | Next.js + Prisma + PostgreSQL | Full-stack types, mature ecosystem |
| Content Site | Astro + Markdown/CMS | Performance, SEO-friendly |
| E-commerce | Next.js + Stripe + Prisma | Payment built-in, auth, SEO |
| Real-time App | Next.js + Socket.io + Redis | Live sync, state sharing |
| Mobile (High Interaction) | Flutter | Cross-platform, near-native performance |
| Mobile (Fast MVP) | React Native + Expo | Quick development, hot reload |

## Skill Structure

```
claude-enterprise-dev-plan-skill/
├── SKILL.md                        # Main entry point
└── references/
    ├── checklist.md                # Pre-development checklist
    ├── tech-stack-guide.md         # Technology recommendations
    ├── architecture-patterns.md    # Modular design patterns
    ├── i18n-setup.md              # Internationalization guide
    └── dev-mode-design.md         # Developer mode implementation
```

## Usage

This skill provides guidance during:
- New project initialization
- Architecture decisions
- Technology stack selection
- Setting up development environments
- Planning for internationalization
- Designing payment systems

## Core Principles

1. **Discuss thoroughly before coding** - Wrong foundation = expensive to fix later
2. **Modularize aggressively** - Piling together is the start of disaster
3. **Configure everything possible** - Never hardcode what can be configured
4. **Developer experience** - Debug tools and sandboxes are efficiency multipliers
5. **Internationalize early** - Mid-project i18n costs increase exponentially
6. **Document as code** - Comments should explain extension points
7. **Reserve 20% for future** - Always leave room for growth

## Installation

Place this skill in your Claude Code skills directory:
- Windows: `%USERPROFILE%/.claude/skills/enterprise-dev-plan/`
- macOS/Linux: `~/.claude/skills/enterprise-dev-plan/`

## Integration with Other Skills

- **no-hardcode**: Use real data sources during development
- **project-refactor**: Apply enterprise patterns during migrations

## License

MIT

---

*Part of the Claude Code skills system*
