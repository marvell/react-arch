# 1. Project Initialization

This section covers the foundational setup of your React project with Vite, TypeScript, workspace configuration, and Git initialization.

## 1.1 Create New Vite + React + TypeScript Project

### Prerequisites
- Node.js 22.12+ installed
- pnpm package manager installed globally

### Create the Project

```bash
# Create new React TypeScript project with Vite
pnpm create vite@latest my-react-app -t react-ts

# Navigate to project directory
cd my-react-app

# Install dependencies
pnpm install
```

### Alternative Templates Available
- `react-ts`: Standard React with TypeScript
- `react-swc-ts`: React with TypeScript using SWC for faster builds
- `react`: Standard React with JavaScript
- `react-swc`: React with JavaScript using SWC

### Verify Installation

```bash
# Start development server to test
pnpm dev
# Should open http://localhost:5173
```

## 1.2 Configure package.json with Proper Naming and Scripts

Update your `package.json` with proper project metadata and essential development scripts:

```json
{
  "name": "my-react-app",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "packageManager": "pnpm@10.11.0",
  "engines": {
    "node": ">=22.12.0"
  },
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0",
    "lint:fix": "eslint . --ext ts,tsx --fix",
    "type-check": "tsc --noEmit",
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:e2e": "playwright test",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

## 1.3 Initialize Git Repository with Proper .gitignore

### Initialize Git Repository

```bash
# Initialize git repository
git init

# Create initial commit structure
git add .
git commit -m "Initial project setup with Vite + React + TypeScript"
```

### Create Comprehensive `.gitignore`

```gitignore
# Node modules and package manager
node_modules/
.pnpm-store/
.pnpm-debug.log*

# Build outputs
dist/
build/
.vite/

# TypeScript compilation
*.tsbuildinfo

# Environment variables
.env
.env.local
.env.*.local

# Log files
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*
lerna-debug.log*

# OS generated files
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# Editor directories and files
.vscode/
!.vscode/extensions.json
!.vscode/settings.json
.idea/
*.suo
*.ntvs*
*.njsproj
*.sln
*.sw?

# Testing
coverage/
.nyc_output/
test-results/
playwright-report/

# Temporary files
*.tmp
*.temp
.cache/

# Local development
.local
```

### Verify Git Setup

```bash
# Check git status
git status

# Verify .gitignore is working
git add .
git status
# Should not show node_modules or other ignored files
```

## 1.4 Optional: pnpm Workspace Setup (Advanced)

> **⚠️ Skip this section unless you need multiple related projects (e.g., SPA + separate marketing site + shared packages)**

### When to Use Workspace:
- Building separate SPA and Astro marketing site 
- Sharing UI components between multiple projects
- Team with 3+ developers managing multiple related apps

### Create `pnpm-workspace.yaml` in Project Root

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
  - 'tools/*'
  - '!**/test/**'

# pnpm configuration (as of pnpm 10.2.1+)
verifyDepsBeforeRun: warn
optimisticRepeatInstall: true
strictPeerDependencies: false
sharedWorkspaceLockfile: true
publicHoistPattern:
  - '*eslint*'
  - '*prettier*'
  - '*typescript*'
```

### Update Root package.json for Workspace

```json
{
  "name": "my-react-monorepo",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "packageManager": "pnpm@latest",
  "engines": {
    "node": ">=20.19.0"
  },
  "scripts": {
    "dev": "pnpm --filter './apps/*' dev",
    "build": "pnpm --recursive build",
    "lint": "pnpm --recursive lint",
    "test": "pnpm --recursive test"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "typescript": "^5.6.0"
  }
}
```

### Move Your App to Workspace Structure

```bash
# Create workspace directories
mkdir -p apps packages tools

# Move your Vite app to apps directory
mv src public index.html vite.config.ts tsconfig.json package.json apps/web/

# Or start fresh with workspace-aware creation:
pnpm create vite@latest apps/web -- --template react-ts
```

## Final Verification Checklist

### Standard Project:
- [ ] Vite dev server runs without errors (`pnpm dev`)
- [ ] TypeScript compilation works (`pnpm type-check`)
- [ ] Git ignores all unnecessary files
- [ ] Package.json contains all required scripts and metadata

### Workspace Project (if applicable):
- [ ] Workspace structure created properly
- [ ] Apps can be built individually
- [ ] Shared dependencies are hoisted correctly

---

**Next Steps**: Continue to [Core Dependencies](./02-core-dependencies.md) to install essential packages.