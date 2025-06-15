# React Architecture v3 - Project Setup Guide

Complete guide for setting up new React applications following our official architecture convention with Feature-Sliced Design (FSD), modern tooling, and best practices.

## 📋 Setup Guide Overview

This guide is broken down into focused sections that you can follow sequentially or reference individually:

### 🚀 Core Setup (Required)
1. **[Project Initialization](./01-project-initialization.md)** - Vite setup, project configuration, and Git
2. **[Core Dependencies](./02-core-dependencies.md)** - Essential packages, tooling, and configurations
3. **[Architecture Setup](./03-architecture-setup.md)** - Feature-Sliced Design (FSD) structure and linting

### 🏗️ Foundation (Required)
4. **[Foundation Configuration](./04-foundation-configuration.md)** - TypeScript, Vite, and app layer setup
5. **[Development Tooling](./05-development-tooling.md)** - ESLint, Prettier, VS Code, and pre-commit hooks
6. **[UI Foundation](./06-ui-foundation.md)** - Tailwind CSS, shadcn/ui, and theme system

### 🎯 Application Layer (Required)
7. **[Routing Setup](./07-routing-setup.md)** - TanStack Router configuration and route management
8. **[State Management](./08-state-management.md)** - TanStack Query and Zustand setup
9. **[Authentication](./09-authentication.md)** - OAuth 2.0, JWT, and auth infrastructure

### 🔌 Integration Layer (Required)
10. **[API Integration](./10-api-integration.md)** - Axios configuration and API layer structure
11. **[Testing Setup](./11-testing-setup.md)** - Vitest, React Testing Library, and Playwright

### 🚢 Production Ready (Optional)
12. **[Build & Deployment](./12-build-deployment.md)** - Docker, nginx, and deployment preparation
13. **[First Feature](./13-first-feature.md)** - Example implementation and development workflow
14. **[Quality Assurance](./14-quality-assurance.md)** - Final verification and testing
15. **[Documentation](./15-documentation.md)** - Project documentation and handoff

## 🎯 Quick Start

For a minimal setup, complete sections 1-11. For production deployment, complete all sections.

### Prerequisites
- Node.js 22.12+
- pnpm package manager
- Git

### Time Estimate
- **Minimal Setup (1-11)**: ~2-3 hours
- **Complete Setup (1-15)**: ~4-6 hours
- **With Custom Features**: +1-2 hours per feature

## 🏗️ Architecture Overview

This setup creates a **unified SPA** using **Feature-Sliced Design (FSD)** methodology that handles both public landing pages and private authenticated application sections:

```
src/
├── app/          # Application-wide configuration
├── pages/        # Complete pages/screens
├── widgets/      # Large, self-contained UI blocks
├── features/     # Reusable user interactions
├── entities/     # Business domain objects
└── shared/       # Reusable utilities and components
```

### Key Benefits
- **Unified**: Single project handles both landing pages and authenticated app
- **Scalable**: Clear separation of concerns with FSD structure
- **Maintainable**: Predictable structure and import rules
- **Team-Friendly**: Consistent conventions reduce onboarding time
- **Type-Safe**: Full TypeScript integration with path mapping
- **Quality**: Automated linting with Steiger for architecture compliance

## 🛠️ Technology Stack

| Category | Technology | Purpose |
|----------|------------|---------|
| **Frontend** | Vite + React + TypeScript | Fast development and type safety |
| **Routing** | TanStack Router | Type-safe routing with file-based routes |
| **State** | TanStack Query + Zustand | Server state + client state management |
| **UI** | Tailwind CSS + shadcn/ui | Utility-first styling + accessible components |
| **Forms** | React Hook Form + Zod | Performant forms with schema validation |
| **Testing** | Vitest + Playwright + RTL | Unit, integration, and E2E testing |
| **Quality** | ESLint + Prettier + Steiger | Code quality and architecture compliance |
| **Build** | Vite + TypeScript | Optimized builds and type checking |

## 📖 Usage Instructions

1. **Start with Overview**: Read this file to understand the structure
2. **Follow Sequential**: Complete sections 1-3 for basic setup
3. **Choose Your Path**: Continue with foundation (4-6) or jump to specific needs
4. **Reference Individual**: Use individual files for specific configuration needs
5. **Verify Setup**: Use section 14 for final verification

## 🤝 Support

- **Issues**: Create GitHub issues for bugs or questions
- **Discussions**: Use GitHub discussions for architecture questions
- **Documentation**: Each section includes troubleshooting and verification steps

---

**Next Steps**: Start with [Project Initialization](./01-project-initialization.md) to begin your setup.