# 2. Core Dependencies Installation

This section covers the installation of all essential dependencies following our architecture convention. Install these packages in the exact order presented to avoid conflicts.

**⚠️ Important**: The shadcn/ui initialization process validates TypeScript path aliases (`@/*` imports) and will fail if they are not properly configured in both `tsconfig.json` and `vite.config.ts` before running the init command. Follow the exact order shown below to avoid validation errors.

## 2.1 Core Application Dependencies

### Install Routing and State Management

```bash
# TanStack Router for type-safe routing
pnpm add @tanstack/react-router @tanstack/react-router-devtools
pnpm add -D @tanstack/router-plugin

# TanStack Query v5 for server state management
pnpm add @tanstack/react-query

# Zustand for client state management
pnpm add zustand

# Axios for API communication
pnpm add axios
```

## 2.2 Forms and Validation

### Install Form Handling with Schema Validation

```bash
# React Hook Form for performant forms
pnpm add react-hook-form

# Zod for schema validation + resolver
pnpm add zod @hookform/resolvers
```

## 2.3 UI Foundation

### Install shadcn/ui with Tailwind CSS v4

```bash
# Install Tailwind CSS v4 and dependencies
pnpm add -D tailwindcss @tailwindcss/vite tailwindcss-animate
pnpm add -D @types/node
```

**Important**: Before initializing shadcn/ui, you must configure TypeScript path aliases and update configuration files. The shadcn/ui init process validates import aliases and will fail if they're not properly configured.

### Update Configuration Files (Required before shadcn/ui init)

#### 1. Update CSS File (`src/index.css`) for Tailwind v4

Replace the entire contents with:

```css
@import "tailwindcss";
```

#### 2. Update TypeScript Configuration (`tsconfig.json`)

Add path mapping to the main TypeScript config:

```json
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ],
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

#### 3. Update TypeScript App Configuration (`tsconfig.app.json`)

Add path mapping to the app config:

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "erasableSyntaxOnly": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src"]
}
```

#### 4. Update Vite Configuration (`vite.config.ts`)

```typescript
import path from "path"
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from "@tailwindcss/vite"

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
})
```

#### 5. Initialize shadcn/ui

After all configuration files are updated:

```bash
# Initialize shadcn/ui (will automatically configure for v4)
pnpm dlx shadcn@latest init --base-color=neutral
```

## 2.4 Development Dependencies

### Install Code Quality Tools

```bash
# ESLint with Airbnb TypeScript configuration
pnpm add -D eslint eslint-config-airbnb-typescript @typescript-eslint/eslint-plugin@^7.0.0 @typescript-eslint/parser@^7.0.0

# Prettier for code formatting
pnpm add -D prettier eslint-config-prettier

# Additional ESLint plugins for React and hooks
pnpm add -D eslint-plugin-react eslint-plugin-react-hooks eslint-plugin-import eslint-plugin-jsx-a11y
```

### Install Testing Framework

```bash
# Vitest with React Testing Library for unit/component tests
pnpm add -D vitest @vitest/browser playwright @testing-library/react

# Playwright browser installation (non-interactive, headless by default)
npx playwright install --with-deps
```

**Note on Playwright Installation**: 

- Playwright runs in **headless mode by default** - no special installation required
- Use `npx playwright install --with-deps` for automated/CI-friendly setup without interactive prompts
- This installs Chromium, Firefox, and WebKit browsers with system dependencies
- Alternative: `npm init playwright@latest` provides interactive setup but requires user input

## 2.5 Create ESLint Configuration (`.eslintrc.json`)

```json
{
  "extends": [
    "airbnb",
    "airbnb-typescript",
    "airbnb/hooks",
    "plugin:@typescript-eslint/recommended-type-checked",
    "plugin:@typescript-eslint/stylistic-type-checked",
    "prettier"
  ],
  "parser": "@typescript-eslint/parser",
  "parserOptions": {
    "project": "./tsconfig.json"
  },
  "rules": {
    "react/react-in-jsx-scope": "off",
    "import/prefer-default-export": "off",
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }]
  }
}
```

## 2.6 Create Prettier Configuration (`.prettierrc`)

```json
{
  "semi": false,
  "trailingComma": "es5",
  "singleQuote": true,
  "tabWidth": 2,
  "useTabs": false
}
```

## 2.7 CSS Configuration

**Note**: The CSS file (`src/index.css`) is automatically configured by shadcn/ui during initialization. After running `pnpm dlx shadcn@latest init`, your CSS file will contain:

- Tailwind v4 import statement
- Custom variant definitions
- Theme configuration with CSS variables
- Base layer styles

The shadcn/ui CLI automatically handles the CSS setup for Tailwind v4, including theme variables and layer configurations.

## 2.8 Updated Package.json Scripts

### Add Essential Scripts to Your `package.json`

```json
{
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

## Installation Verification

### Run These Commands to Verify Everything is Installed Correctly

```bash
# Verify TypeScript compilation
pnpm type-check

# Verify linting works
pnpm lint

# Verify tests can run
pnpm test --run

# Verify development server starts
pnpm dev
```

## Installation Checklist

- [ ] All core dependencies installed without errors
- [ ] shadcn/ui initialized successfully
- [ ] TypeScript compilation passes
- [ ] ESLint configuration loaded without errors
- [ ] Playwright browsers installed (Chromium, Firefox, WebKit)
- [ ] Development server starts on http://localhost:5173
- [ ] Hot reload works when editing files
- [ ] Test suite runs without errors

---

**Next Steps**: Continue to [Architecture Setup](./03-architecture-setup.md) to implement Feature-Sliced Design structure.