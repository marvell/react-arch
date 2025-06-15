# 3. Architecture Setup (FSD Structure)

Feature-Sliced Design (FSD) is a methodology for organizing frontend applications that enforces strict separation of concerns through layers, slices, and segments. This section provides step-by-step implementation of FSD architecture for React TypeScript projects.

## 3.1 Understanding FSD Architecture

### Core Concepts
- **Layers**: Divide code by scope of influence and abstraction level
- **Slices**: Divide code by business domain within layers
- **Segments**: Divide code by technical purpose within slices

### Layer Hierarchy (high to low abstraction)
1. `app` - Application-wide configuration, providers, initialization
2. `pages` - Complete pages/screens (route-level components)
3. `widgets` - Large, self-contained UI blocks
4. `features` - Reusable user interactions with business value
5. `entities` - Business domain objects and their logic
6. `shared` - Reusable code with minimal dependencies

### Golden Rules
- **Import Rule**: Only import from layers below (features → entities → shared)
- **Public API Rule**: All external access goes through index.ts files
- **No Cross-Imports**: Slices on same layer cannot import each other directly

## 3.2 Create FSD Directory Structure

### Create the Complete FSD Structure

```bash
# Create the full FSD directory structure
mkdir -p src/{app,pages,widgets,features,entities,shared}

# Create app layer segments
mkdir -p src/app/{providers,router,store,styles}

# Create shared layer segments
mkdir -p src/shared/{ui,api,lib,config,types}
```

### Verify Structure Creation

```bash
# Check the created structure
tree src -I node_modules
```

## 3.3 Migrate Existing App Files to FSD Structure

After creating the FSD directory structure, we need to move existing application files to their appropriate locations and update imports to ensure the project builds correctly.

### Move Core Application Files

```bash
# Move App.tsx to app layer
mv src/App.tsx src/app/App.tsx

# Move App.css to app styles (if it exists)
if [ -f "src/App.css" ]; then
  mv src/App.css src/app/styles/App.css
fi

# Create app index file that exports the App component
cat > src/app/index.ts << 'EOF'
export { App } from './App'
EOF
```

### Update App.tsx Imports

```bash
# Update App.tsx to use correct import paths
cat > src/app/App.tsx << 'EOF'
import './styles/App.css'

function App() {
  return (
    <div className="App">
      <h1>React App with FSD</h1>
      <p>Feature-Sliced Design structure is ready!</p>
    </div>
  )
}

export { App }
EOF
```

### Create App Styles (if not exists)

```bash
# Create basic App.css in app styles directory
cat > src/app/styles/App.css << 'EOF'
.App {
  text-align: center;
  padding: 2rem;
}

.App h1 {
  color: #333;
  margin-bottom: 1rem;
}
EOF
```

### Update Main Entry Point

```bash
# Update main.tsx to import from FSD structure
cat > src/main.tsx << 'EOF'
import React from 'react'
import ReactDOM from 'react-dom/client'
import { App } from './app'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
EOF
```

### Verify Build After Migration

```bash
# Check TypeScript compilation
pnpm type-check

# Verify the build works with new structure
pnpm build

# Test development server
pnpm dev
```

### Expected Results

After migration, your project should:
- ✅ Build successfully with `pnpm build`
- ✅ Compile TypeScript without errors
- ✅ Start development server without issues
- ✅ Display the updated app content in browser

## 3.4 Configure shadcn/ui for FSD Structure

Update the shadcn/ui configuration to work with FSD directory structure:

```bash
# Update components.json for FSD structure
cat > components.json << 'EOF'
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "new-york",
  "rsc": false,
  "tsx": true,
  "tailwind": {
    "config": "",
    "css": "src/index.css",
    "baseColor": "neutral",
    "cssVariables": true,
    "prefix": ""
  },
  "aliases": {
    "components": "@/shared/ui",
    "utils": "@/shared/lib/utils",
    "ui": "@/shared/ui",
    "lib": "@/shared/lib",
    "hooks": "@/shared/lib"
  },
  "iconLibrary": "lucide"
}
EOF
```

## 3.5 Install and Configure Steiger (FSD Linter)

Steiger is the official architectural linter for Feature-Sliced Design that enforces FSD principles automatically.

### Install Steiger

```bash
# Install Steiger and FSD plugin
pnpm add -D steiger @feature-sliced/steiger-plugin
```

### Create Steiger Configuration

```bash
cat > steiger.config.ts << 'EOF'
import { defineConfig } from 'steiger'
import fsd from '@feature-sliced/steiger-plugin'

export default defineConfig([
  // Apply all recommended FSD rules
  ...fsd.configs.recommended,
  
  // Custom rule configurations
  {
    rules: {
      // Enforce public API usage (no internal imports)
      'fsd/no-public-api-sidestep': 'error',
      
      // Prevent imports from higher layers
      'fsd/no-higher-level-imports': 'error',
      
      // Prevent cross-imports between same-layer slices
      'fsd/no-cross-imports': 'error',
      
      // Require public API for all slices
      'fsd/public-api': 'error',
      
      // Warn about excessive slicing
      'fsd/excessive-slicing': 'warn',
      
      // Detect insignificant slices
      'fsd/insignificant-slice': 'warn',
    },
  },
  
  // Ignore specific files/folders
  {
    ignores: [
      '**/node_modules/**',
      '**/dist/**', 
      '**/.vite/**',
      '**/coverage/**',
      '**/__tests__/**',
      '**/*.test.{ts,tsx}',
      '**/*.spec.{ts,tsx}',
    ],
  },
])
EOF
```

### Add Steiger Scripts to package.json

```bash
# Add scripts using npm pkg command
npm pkg set scripts.arch-lint="steiger ./src"
npm pkg set scripts.arch-lint:watch="steiger ./src --watch"
npm pkg set scripts.arch-lint:fix="steiger ./src --fix"
```

### Update Existing Lint Script to Include Architecture Linting

```bash
# Update the main lint script
npm pkg set scripts.lint="eslint . --ext ts,tsx --report-unused-disable-directives --max-warnings 0 && steiger ./src"
```

## 3.6 Configure TypeScript Path Mapping for FSD

### Update tsconfig.json Paths for Clean Imports

```bash
# Update TypeScript configuration for FSD paths
cat > tsconfig.paths.json << 'EOF'
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@/app/*": ["./src/app/*"],
      "@/pages/*": ["./src/pages/*"],
      "@/widgets/*": ["./src/widgets/*"],
      "@/features/*": ["./src/features/*"], 
      "@/entities/*": ["./src/entities/*"],
      "@/shared/*": ["./src/shared/*"]
    }
  }
}
EOF
```

### Update Main tsconfig.json to Extend Paths

```bash
# Backup existing tsconfig.json
cp tsconfig.json tsconfig.json.backup

# Update main tsconfig.json to include FSD paths
cat > tsconfig.json << 'EOF'
{
  "files": [],
  "references": [
    { "path": "./tsconfig.app.json" },
    { "path": "./tsconfig.node.json" }
  ],
  "extends": "./tsconfig.paths.json"
}
EOF
```

### Update tsconfig.app.json

```bash
# Add FSD paths to app config
sed -i '' '/\"baseUrl\"/a\
    \"extends\": \"../tsconfig.paths.json\",
' tsconfig.app.json
```

## 3.7 Troubleshooting

### Common Issues and Solutions

#### Path Mapping Not Working
**Problem**: TypeScript can't resolve `@/shared/*` imports
**Solution**:
- Verify `tsconfig.paths.json` is properly extended in main configs
- Restart TypeScript language server in your IDE
- Check that `baseUrl` is set correctly

#### Steiger/Architecture Linting Issues
**Problem**: Steiger fails with parsing errors
**Solution**: 
- Check `steiger.config.ts` for proper ignores
- Ensure configuration files are excluded
- Use warnings instead of errors initially

## Architecture Setup Checklist

- [ ] ✅ Created complete FSD directory structure (7 layers + segments)
- [ ] ✅ Migrated existing app files to FSD structure
- [ ] ✅ Verified project builds successfully after migration
- [ ] ✅ Configured shadcn/ui to install components in @/shared/ui
- [ ] ✅ Configured Steiger linter with FSD rules
- [ ] ✅ Added TypeScript path mapping for clean FSD imports
- [ ] ✅ Updated package.json scripts for architecture maintenance

## What's Accomplished

- **Scalable Structure**: Complete FSD hierarchy with proper layer separation
- **Import Safety**: Steiger prevents architectural violations automatically  
- **Developer Experience**: TypeScript paths enable clean, readable imports
- **UI Foundation**: shadcn/ui configured for proper FSD component placement
- **Team Alignment**: Consistent structure reduces onboarding time and debates

---

**Next Steps**: Proceed to [Foundation Configuration](./04-foundation-configuration.md) to implement the core application setup within this FSD structure.