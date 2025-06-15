# 14. Quality Assurance

Comprehensive quality assurance checklist and best practices for maintaining code quality, performance, and reliability in React applications following our architectural standards.

## Quick Start Checklist

```bash
# Run all quality checks
npm run lint                    # ESLint + Prettier
npm run type-check             # TypeScript strict mode
npm run test                   # Unit/Component tests (Vitest + RTL)
npm run test:e2e              # End-to-end tests (Playwright)
npm run build                 # Production build verification
```

## 1. Code Quality & Static Analysis

### 1.1 ESLint Configuration

**Required Rules (Airbnb + TypeScript + React):**
```javascript
// eslint.config.js - Modern Flat Config (2024 standard)
import js from "@eslint/js";
import react from "eslint-plugin-react";
import reactHooks from "eslint-plugin-react-hooks";
import typescript from "@typescript-eslint/eslint-plugin";
import typescriptParser from "@typescript-eslint/parser";

export default [
  {
    files: ["**/*.{js,jsx,ts,tsx}"],
    plugins: {
      react,
      "react-hooks": reactHooks,
      "@typescript-eslint": typescript,
    },
    languageOptions: {
      parser: typescriptParser,
      parserOptions: {
        ecmaFeatures: { jsx: true },
        project: "./tsconfig.json",
      },
    },
    rules: {
      // Airbnb base rules
      "@typescript-eslint/no-unused-vars": "error",
      "@typescript-eslint/no-floating-promises": "error",
      "react-hooks/rules-of-hooks": "error",
      "react-hooks/exhaustive-deps": "warn",
      // FSD architecture enforcement
      "import/no-restricted-paths": ["error", {
        zones: [
          { target: "src/shared", from: ["src/entities", "src/features", "src/widgets", "src/pages"] },
          { target: "src/entities", from: ["src/features", "src/widgets", "src/pages"] },
          { target: "src/features", from: ["src/widgets", "src/pages"] },
          { target: "src/widgets", from: ["src/pages"] },
        ]
      }],
    },
  },
];
```

**Pre-commit Hook Setup:**
```bash
# Install husky for git hooks
npm install --save-dev husky lint-staged

# .husky/pre-commit
npx lint-staged

# package.json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,css}": ["prettier --write"]
  }
}
```

### 1.2 TypeScript Strict Mode

**Required tsconfig.json settings:**
```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

**Type Checking Command:**
```bash
# Run without building
npx tsc --noEmit

# With custom config
npx tsc --noEmit --project tsconfig.json
```

## 2. Testing Strategy (Testing Pyramid)

### 2.1 Unit & Component Tests (Vitest + React Testing Library)

**Best Practices:**
- **Test user behavior**, not implementation details
- **Query priority**: `getByRole` > `getByLabelText` > `getByText` > `getByTestId`
- **Avoid** testing internal state or component instances
- **Use** async utilities for dynamic content

**Example Test Structure:**
```typescript
// src/entities/user/ui/user-card.test.tsx
import { render, screen } from '@testing-library/react';
import { UserCard } from './user-card';

describe('UserCard', () => {
  it('displays user information', async () => {
    render(<UserCard user={{ name: 'John Doe', email: 'john@example.com' }} />);
    
    // ✅ Query by role (highest priority)
    expect(screen.getByRole('heading', { name: /john doe/i })).toBeInTheDocument();
    
    // ✅ Query by text for user-visible content
    expect(screen.getByText('john@example.com')).toBeInTheDocument();
    
    // ❌ Avoid: screen.getByTestId() unless absolutely necessary
  });

  it('handles loading state', async () => {
    render(<UserCard isLoading />);
    
    // ✅ Use findBy for async content
    expect(await screen.findByRole('progressbar')).toBeInTheDocument();
  });
});
```

**Test Coverage Requirements:**
```bash
# Minimum 80% coverage for:
# - src/entities (business logic)
# - src/features (user actions)
# - src/shared/ui (reusable components)

npm run test -- --coverage --threshold=80
```

### 2.2 E2E Tests (Playwright)

**Test Organization:**
```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication Flow', () => {
  test.beforeEach(async ({ page }) => {
    // Clean state for each test
    await page.goto('/login');
  });

  test('should authenticate with Google OAuth', async ({ page }) => {
    // Test critical user flows
    await page.getByRole('button', { name: /sign in with google/i }).click();
    
    // Auto-retrying assertion (best practice)
    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByRole('navigation')).toContainText('Dashboard');
  });

  test('should handle authentication failure', async ({ page }) => {
    // Mock failed OAuth response
    await page.route('**/auth/callback*', route => 
      route.fulfill({ status: 401 })
    );
    
    await page.getByRole('button', { name: /sign in/i }).click();
    await expect(page.getByRole('alert')).toContainText('Authentication failed');
  });
});
```

**Playwright Configuration:**
```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  // Test isolation
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
  },
  
  // Parallel execution (CI: workers: 1 for stability)
  workers: process.env.CI ? 1 : undefined,
  
  // Retry failed tests
  retries: process.env.CI ? 2 : 0,
  
  // Projects for different browsers
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],
});
```

### 2.3 Visual Regression Tests

```typescript
// e2e/visual.spec.ts
test('dashboard layout is visually consistent', async ({ page }) => {
  await page.goto('/dashboard');
  await page.waitForLoadState('networkidle');
  
  // Screenshot comparison
  await expect(page).toHaveScreenshot('dashboard.png');
});
```

## 3. Build Process Verification

### 3.1 Production Build Checks

```bash
# Build verification script
npm run build

# Check build output size
npm run build -- --mode production
ls -la dist/

# Test production server locally
npm run preview
```

**Build Optimization Checklist:**
- [ ] Bundle size analysis (`npm run build -- --analyze`)
- [ ] Tree-shaking verification (no unused exports)
- [ ] Code splitting by routes
- [ ] Asset optimization (images, fonts)
- [ ] Environment variables properly configured

### 3.2 Bundle Analysis

```bash
# Install bundle analyzer
npm install --save-dev rollup-plugin-visualizer

# Add to vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    visualizer({ filename: 'dist/stats.html', open: true })
  ]
});
```

## 4. Authentication Flow Testing

### 4.1 OAuth 2.0 Flow Verification

**Manual Testing Checklist:**
- [ ] Login redirect to OAuth provider
- [ ] Callback handling with authorization code
- [ ] JWT token storage in localStorage
- [ ] Automatic token refresh
- [ ] Logout and token cleanup

**Automated E2E Tests:**
```typescript
test('complete auth flow', async ({ page, context }) => {
  // Test with actual OAuth provider (staging/test env)
  await page.goto('/login');
  
  // Mock OAuth provider response for consistent testing
  await page.route('**/oauth/google', route => 
    route.fulfill({
      status: 302,
      headers: { 'Location': '/auth/callback?code=test_code' }
    })
  );
  
  await page.getByRole('button', { name: /sign in with google/i }).click();
  
  // Verify token storage
  const token = await page.evaluate(() => localStorage.getItem('authToken'));
  expect(token).toBeTruthy();
  
  // Verify authenticated state
  await expect(page).toHaveURL('/dashboard');
});
```

### 4.2 Token Management

**Test Authorization Headers:**
```typescript
// Verify Axios interceptor adds auth headers
test('API requests include auth token', async ({ page }) => {
  let requestHeaders: Record<string, string> = {};
  
  await page.route('**/api/**', route => {
    requestHeaders = route.request().headers();
    route.continue();
  });
  
  await page.goto('/dashboard'); // Triggers API calls
  
  expect(requestHeaders['authorization']).toMatch(/^Bearer /);
});
```

## 5. Routing & Navigation Validation

### 5.1 TanStack Router Testing

**Route Type Safety:**
```typescript
// Verify all routes are properly typed
test('routing type safety', () => {
  // This should fail TypeScript compilation if routes are invalid
  const navigate = useNavigate();
  navigate({ to: '/dashboard/users/$userId', params: { userId: '123' } });
});
```

**Navigation Tests:**
```typescript
test('navigation between routes', async ({ page }) => {
  await page.goto('/dashboard');
  
  // Test programmatic navigation
  await page.getByRole('link', { name: /users/i }).click();
  await expect(page).toHaveURL('/dashboard/users');
  
  // Test browser back/forward
  await page.goBack();
  await expect(page).toHaveURL('/dashboard');
});
```

### 5.2 Route Guards & Protection

```typescript
test('protected routes redirect unauthenticated users', async ({ page }) => {
  // Clear auth token
  await page.addInitScript(() => {
    localStorage.removeItem('authToken');
  });
  
  await page.goto('/dashboard');
  await expect(page).toHaveURL('/login');
});
```

## 6. Performance Testing

### 6.1 Core Web Vitals

```typescript
// e2e/performance.spec.ts
test('meets Core Web Vitals thresholds', async ({ page }) => {
  await page.goto('/dashboard');
  
  const vitals = await page.evaluate(() => {
    return new Promise(resolve => {
      new PerformanceObserver(list => {
        const lcpEntry = list.getEntries().find(e => e.entryType === 'largest-contentful-paint');
        resolve({ lcp: lcpEntry?.startTime });
      }).observe({ entryTypes: ['largest-contentful-paint'] });
    });
  });
  
  expect(vitals.lcp).toBeLessThan(2500); // 2.5s threshold
});
```

### 6.2 Bundle Size Monitoring

```bash
# Add to CI pipeline
npm run build
bundlesize check
```

**package.json:**
```json
{
  "bundlesize": [
    {
      "path": "./dist/assets/index-*.js",
      "maxSize": "500kb"
    }
  ]
}
```

## 7. CI/CD Quality Gates

### 7.1 GitHub Actions Workflow

```yaml
# .github/workflows/quality.yml
name: Quality Assurance
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
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test
      - run: npm run build
      - run: npx playwright install --with-deps
      - run: npm run test:e2e
      
      # Quality gates
      - name: Check bundle size
        run: npm run bundlesize
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

### 7.2 Quality Metrics Dashboard

**Key Metrics to Track:**
- Test coverage (>80% for critical paths)
- Build time (<5 minutes)
- Bundle size (<500KB main chunk)
- E2E test success rate (>95%)
- TypeScript strict mode compliance (100%)

## 8. Troubleshooting Common Issues

### 8.1 Test Debugging

```bash
# Debug failing tests
npm run test -- --reporter=verbose

# Debug E2E tests with UI
npx playwright test --ui

# Debug with trace viewer
npx playwright show-trace trace.zip
```

### 8.2 Build Issues

```bash
# Check for circular dependencies
npm run build -- --mode development

# Analyze bundle composition
npm run build -- --analyze

# Verbose build output
npm run build -- --verbose
```

---

**Quality Gate Requirements:**
- ✅ All linting passes (0 errors)
- ✅ TypeScript strict mode (0 errors)
- ✅ Test coverage >80% (critical paths)
- ✅ All E2E tests pass
- ✅ Build succeeds without warnings
- ✅ Bundle size within limits
- ✅ Authentication flow verified
- ✅ Core routes accessible

**Next Steps**: Continue to [Documentation](./15-documentation.md) for project handoff.