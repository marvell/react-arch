# 5. Development Tooling

This guide covers the essential development tools that ensure code quality, consistency, and team productivity. These configurations are mandatory and aligned with our architectural principles.

## 5.1. ESLint Configuration

We use ESLint with the modern **Flat Config** system (`eslint.config.js`) introduced in ESLint v9. This provides better performance and clearer configuration.

### Install Dependencies

```bash
# Core ESLint dependencies
npm install -D eslint

# TypeScript support
npm install -D @typescript-eslint/eslint-plugin @typescript-eslint/parser

# Airbnb configuration
npm install -D eslint-config-airbnb eslint-plugin-import eslint-plugin-react eslint-plugin-jsx-a11y eslint-plugin-react-hooks

# Feature-Sliced Design rules
npm install -D @feature-sliced/eslint-config eslint-plugin-boundaries eslint-import-resolver-typescript

# Prettier integration (removes conflicting rules)
npm install -D eslint-config-prettier
```

### ESLint Configuration File

Create `eslint.config.js` in your project root:

```javascript
// eslint.config.js
import js from '@eslint/js';
import typescriptParser from '@typescript-eslint/parser';
import typescriptPlugin from '@typescript-eslint/eslint-plugin';
import reactPlugin from 'eslint-plugin-react';
import reactHooksPlugin from 'eslint-plugin-react-hooks';
import jsxA11yPlugin from 'eslint-plugin-jsx-a11y';
import importPlugin from 'eslint-plugin-import';
import boundariesPlugin from 'eslint-plugin-boundaries';
import prettierConfig from 'eslint-config-prettier';

export default [
  // Global ignores
  {
    ignores: [
      'dist/',
      'build/',
      'node_modules/',
      '*.generated.*',
      'coverage/',
      '.next/',
      'out/',
    ],
  },

  // Base JavaScript configuration
  {
    files: ['**/*.{js,jsx}'],
    plugins: {
      js,
    },
    extends: [js.configs.recommended],
  },

  // TypeScript and React configuration
  {
    files: ['**/*.{ts,tsx}'],
    languageOptions: {
      parser: typescriptParser,
      parserOptions: {
        ecmaVersion: 'latest',
        sourceType: 'module',
        ecmaFeatures: {
          jsx: true,
        },
        project: './tsconfig.json',
      },
    },
    plugins: {
      '@typescript-eslint': typescriptPlugin,
      react: reactPlugin,
      'react-hooks': reactHooksPlugin,
      'jsx-a11y': jsxA11yPlugin,
      import: importPlugin,
      boundaries: boundariesPlugin,
    },
    extends: [
      'plugin:@typescript-eslint/recommended',
      'plugin:@typescript-eslint/recommended-type-checked',
      'plugin:react/recommended',
      'plugin:react/jsx-runtime',
      'plugin:react-hooks/recommended',
      'plugin:jsx-a11y/recommended',
      'plugin:import/recommended',
      'plugin:import/typescript',
      '@feature-sliced/eslint-config/rules/import-order',
      '@feature-sliced/eslint-config/rules/public-api',
      '@feature-sliced/eslint-config/rules/layers-slices',
      prettierConfig, // Must be last to override conflicting rules
    ],
    rules: {
      // React rules
      'react/react-in-jsx-scope': 'off', // Not needed with new JSX transform
      'react/prop-types': 'off', // TypeScript handles prop validation
      'react/jsx-filename-extension': [
        'error',
        { extensions: ['.jsx', '.tsx'] },
      ],

      // TypeScript rules
      '@typescript-eslint/no-unused-vars': 'warn',
      '@typescript-eslint/no-explicit-any': 'warn',
      '@typescript-eslint/prefer-const': 'error',

      // Import rules
      'import/order': 'off', // Handled by FSD rules
      'import/extensions': [
        'error',
        'ignorePackages',
        {
          js: 'never',
          jsx: 'never',
          ts: 'never',
          tsx: 'never',
        },
      ],

      // General rules
      'no-console': 'warn',
      'no-debugger': 'error',
    },
    settings: {
      react: {
        version: 'detect',
      },
      'import/resolver': {
        typescript: {
          alwaysTryTypes: true,
          project: './tsconfig.json',
        },
        node: {
          extensions: ['.js', '.jsx', '.ts', '.tsx'],
        },
      },
      // Feature-Sliced Design boundaries
      'boundaries/elements': [
        { type: 'app', pattern: 'src/app/*' },
        { type: 'pages', pattern: 'src/pages/*' },
        { type: 'widgets', pattern: 'src/widgets/*' },
        { type: 'features', pattern: 'src/features/*' },
        { type: 'entities', pattern: 'src/entities/*' },
        { type: 'shared', pattern: 'src/shared/*' },
      ],
    },
  },
];
```

### Package.json Scripts

Add these scripts to your `package.json`:

```json
{
  "scripts": {
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix",
    "lint:check": "eslint src/ --max-warnings 0"
  }
}
```

## 5.2. Prettier Configuration

Prettier enforces consistent code formatting across the team. It integrates seamlessly with ESLint.

### Install Prettier

```bash
npm install -D prettier
```

### Prettier Configuration

Create `.prettierrc.json` in your project root:

```json
{
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false,
  "semi": true,
  "singleQuote": true,
  "quoteProps": "as-needed",
  "trailingComma": "es5",
  "bracketSpacing": true,
  "bracketSameLine": false,
  "arrowParens": "avoid",
  "endOfLine": "lf",
  "overrides": [
    {
      "files": "*.{md,mdx}",
      "options": {
        "proseWrap": "always"
      }
    }
  ]
}
```

### Prettier Ignore File

Create `.prettierignore`:

```
# Build outputs
dist/
build/
coverage/
.next/
out/

# Dependencies
node_modules/
.pnpm-store/

# Generated files
*.generated.*
.eslintcache

# Logs
*.log

# Environment files
.env*

# IDE
.vscode/
.idea/
```

### Package.json Scripts

Add formatting scripts:

```json
{
  "scripts": {
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  }
}
```

## 5.3. VS Code Configuration

Optimize your development experience with the right VS Code settings and extensions.

### Workspace Settings

Create `.vscode/settings.json`:

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.preferences.importModuleSpecifier": "relative",
  "typescript.preferences.includePackageJsonAutoImports": "on",
  "typescript.suggest.autoImports": true,
  "emmet.includeLanguages": {
    "typescript": "html",
    "typescriptreact": "html"
  },
  "files.associations": {
    "*.tsx": "typescriptreact"
  },
  "search.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/build": true,
    "**/.next": true,
    "**/coverage": true
  }
}
```

### Recommended Extensions

Create `.vscode/extensions.json`:

```json
{
  "recommendations": [
    // Essential formatting and linting
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    
    // TypeScript support
    "ms-vscode.vscode-typescript-next",
    
    // React development
    "burkeholland.simple-react-snippets",
    "formulahendry.auto-rename-tag",
    "dsznajder.es7-react-js-snippets",
    
    // Styling
    "bradlc.vscode-tailwindcss",
    
    // Productivity
    "wix.vscode-import-cost",
    "aaron-bond.better-comments",
    "eamodio.gitlens",
    
    // File management
    "christian-kohler.path-intellisense",
    "ms-vscode.vscode-json"
  ]
}
```

## 5.4. Pre-commit Hooks

Automate code quality checks before commits to maintain consistency.

### Install Dependencies

```bash
# Install pre-commit (Python tool)
pip install pre-commit

# Or using npm alternative
npm install -D @pre-commit/cli

# Install husky and lint-staged for Node.js projects
npm install -D husky lint-staged
```

### Pre-commit Configuration

Create `.pre-commit-config.yaml`:

```yaml
repos:
  # General file cleanup
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-merge-conflict
      - id: check-added-large-files

  # JavaScript/TypeScript linting
  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v8.56.0
    hooks:
      - id: eslint
        files: \.(js|jsx|ts|tsx)$
        additional_dependencies:
          - eslint@^8.56.0
          - '@typescript-eslint/eslint-plugin@^6.0.0'
          - '@typescript-eslint/parser@^6.0.0'

  # Code formatting
  - repo: https://github.com/pre-commit/mirrors-prettier
    rev: v3.1.0
    hooks:
      - id: prettier
        files: \.(js|jsx|ts|tsx|json|css|md|yaml|yml)$

  # TypeScript type checking
  - repo: local
    hooks:
      - id: typescript-check
        name: TypeScript Type Check
        entry: npx tsc --noEmit
        language: system
        files: \.(ts|tsx)$
        pass_filenames: false
```

### Alternative: Husky + lint-staged

For Node.js-focused workflows, add to `package.json`:

```json
{
  "scripts": {
    "prepare": "husky install"
  },
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,css,md}": [
      "prettier --write"
    ]
  }
}
```

Then run:

```bash
# Initialize husky
npm run prepare

# Add pre-commit hook
npx husky add .husky/pre-commit "npx lint-staged"
```

### Install Hooks

```bash
# For pre-commit
pre-commit install

# For husky (already handled by prepare script)
# npm run prepare
```

## 5.5. Complete Setup Script

Create `scripts/setup-dev-tools.sh` for team onboarding:

```bash
#!/bin/bash

echo "🔧 Setting up development tooling..."

# Install all dependencies
echo "📦 Installing dependencies..."
npm install -D eslint prettier @typescript-eslint/eslint-plugin @typescript-eslint/parser \
  eslint-config-airbnb eslint-plugin-import eslint-plugin-react eslint-plugin-jsx-a11y \
  eslint-plugin-react-hooks @feature-sliced/eslint-config eslint-plugin-boundaries \
  eslint-import-resolver-typescript eslint-config-prettier husky lint-staged

# Initialize husky
echo "🪝 Setting up Git hooks..."
npm run prepare

# Install pre-commit (if available)
if command -v pre-commit &> /dev/null; then
    pre-commit install
    echo "✅ Pre-commit hooks installed"
fi

# Run initial format
echo "🎨 Running initial format..."
npm run format

# Run initial lint check
echo "🔍 Running lint check..."
npm run lint:check

echo "✅ Development tooling setup complete!"
echo ""
echo "Next steps:"
echo "1. Install recommended VS Code extensions: Cmd+Shift+P -> 'Extensions: Show Recommended Extensions'"
echo "2. Reload VS Code to apply settings"
echo "3. Run 'npm run lint' and 'npm run format' to verify setup"
```

Make it executable:

```bash
chmod +x scripts/setup-dev-tools.sh
```

## 5.6. CI/CD Integration

Add these checks to your CI pipeline:

```yaml
# .github/workflows/quality-check.yml
name: Code Quality

on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      
      - run: npm ci
      
      - name: Type Check
        run: npx tsc --noEmit
      
      - name: Lint Check
        run: npm run lint:check
      
      - name: Format Check
        run: npm run format:check
```

---

**Next Steps**: Continue to [UI Foundation](./06-ui-foundation.md) for styling setup.