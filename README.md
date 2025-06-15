# React Architecture v3 🚀

> **Production-ready React architecture with Feature-Sliced Design, TypeScript, and modern tooling for building scalable applications fast.**

## 🎯 Quick Start

```bash
# Clone and setup
git clone <your-fork-url>
cd react-arch

# Follow the setup guides
cd docs/setup-guides
# Start with 00-overview.md
```

**⏱️ Time to Production:** 2-3 hours for minimal setup | 4-6 hours for complete setup

## 🏗️ What This Architecture Provides

### 🔥 **Unified SPA Architecture**
- **Single project** handles both landing pages AND authenticated app sections
- **Type-safe routing** with TanStack Router
- **Smart state separation** - server state (TanStack Query) + client state (Zustand)
- **OAuth 2.0 authentication** with JWT token management

### 📐 **Feature-Sliced Design (FSD)**
```
src/
├── app/          # Application-wide configuration
├── pages/        # Complete pages/screens  
├── widgets/      # Large, self-contained UI blocks
├── features/     # Reusable user interactions
├── entities/     # Business domain objects
└── shared/       # Reusable utilities and components
```

### 🛠️ **Complete Tech Stack**
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
| **Deploy** | Docker + Kubernetes | Container-ready with nginx |

## 📚 Documentation Structure

### 🚀 **Core Setup** (Required - 2-3 hours)
1. **[Project Initialization](./docs/setup-guides/01-project-initialization.md)** - Vite, TypeScript, Git setup
2. **[Core Dependencies](./docs/setup-guides/02-core-dependencies.md)** - Essential packages and tooling  
3. **[Architecture Setup](./docs/setup-guides/03-architecture-setup.md)** - FSD structure + Steiger linting
4. **[Foundation Configuration](./docs/setup-guides/04-foundation-configuration.md)** - TypeScript, Vite, app layer
5. **[Development Tooling](./docs/setup-guides/05-development-tooling.md)** - ESLint, Prettier, VS Code
6. **[UI Foundation](./docs/setup-guides/06-ui-foundation.md)** - Tailwind + shadcn/ui + themes
7. **[Routing Setup](./docs/setup-guides/07-routing-setup.md)** - TanStack Router configuration
8. **[State Management](./docs/setup-guides/08-state-management.md)** - TanStack Query + Zustand
9. **[Authentication](./docs/setup-guides/09-authentication.md)** - OAuth 2.0 + JWT infrastructure
10. **[API Integration](./docs/setup-guides/10-api-integration.md)** - Axios configuration + API layer
11. **[Testing Setup](./docs/setup-guides/11-testing-setup.md)** - Vitest + RTL + Playwright

### 🚢 **Production Ready** (Optional - +2-3 hours)
12. **[Build & Deployment](./docs/setup-guides/12-build-deployment.md)** - Docker + nginx + K8s
13. **[First Feature](./docs/setup-guides/13-first-feature.md)** - Complete task management example
14. **[Quality Assurance](./docs/setup-guides/14-quality-assurance.md)** - Final verification + testing
15. **[Documentation](./docs/setup-guides/15-documentation.md)** - Project docs + handoff

### 📖 **Architecture Reference**
- **[Architecture Convention](./docs/arch.md)** - Complete architectural principles and patterns
- **[Setup Overview](./docs/setup-guides/00-overview.md)** - Detailed prerequisites and roadmap

## 🌟 Key Benefits

### ⚡ **Developer Velocity**
- **Consistent structure** reduces decision fatigue and onboarding time
- **Type-safe everything** catches errors at compile time, not runtime
- **Automated quality checks** with Steiger for architectural compliance
- **Hot reloading** with Vite for instant feedback

### 🎯 **Production Ready**
- **Authentication built-in** with OAuth 2.0 and JWT best practices
- **Error boundaries** and proper error handling patterns
- **Performance optimized** with code splitting and caching strategies
- **Container ready** with multi-stage Docker builds

### 👥 **Team Friendly**
- **Clear import rules** prevent circular dependencies and architectural violations
- **Predictable file locations** using Feature-Sliced Design methodology
- **Comprehensive testing** with unit, integration, and E2E test examples
- **Pre-commit hooks** ensure code quality before commits

## 🏃‍♂️ Getting Started

### Prerequisites
- **Node.js 22.12+**
- **pnpm** package manager  
- **Git**

### Choose Your Path

**🎯 Minimal Setup (2-3 hours)**
Complete guides 1-11 for a fully functional development environment.

**🚀 Production Setup (4-6 hours)**  
Complete all guides 1-15 for production-ready deployment.

**🔧 Custom Integration**
Use individual guides as reference for specific configuration needs.

### Start Here
1. **Read** [Setup Overview](./docs/setup-guides/00-overview.md) for detailed roadmap
2. **Follow** [Project Initialization](./docs/setup-guides/01-project-initialization.md) to begin
3. **Reference** [Architecture Convention](./docs/arch.md) for principles and patterns

## 🎯 Architecture Principles

### **Pragmatism over Dogma**
We use proven, stable technologies that solve real problems. The goal is to ship robust products, not chase trends.

### **Long-Term Maintainability** 
We write code for the developer who will maintain it in six months—which is likely ourselves. Clarity and predictability are paramount.

### **Strict Separation of Concerns**
We enforce strong boundaries between UI, client state, and server state. This is the key to managing complexity.

## 💡 Example: Task Management Feature

The setup guides include a complete **Task Management** feature implementation showing:

- ✅ **Entity Layer**: TypeScript types + TanStack Query hooks
- ✅ **Feature Layer**: CRUD operations with React Hook Form + Zod validation
- ✅ **Page Layer**: Complete UI with TanStack Router integration  
- ✅ **Authentication**: OAuth 2.0 flow with JWT token management
- ✅ **Testing**: Unit, integration, and E2E test examples

## 🤝 Support & Contributing

- **📖 Documentation**: Each guide includes troubleshooting and verification steps
- **🐛 Issues**: Create GitHub issues for bugs or questions  
- **💬 Discussions**: Use GitHub discussions for architecture questions
- **🔄 Contributing**: PRs welcome for improvements and additional examples

## 📄 License

MIT License - feel free to use this architecture for your projects.

---

**Ready to build fast?** Start with [Setup Overview](./docs/setup-guides/00-overview.md) 🚀