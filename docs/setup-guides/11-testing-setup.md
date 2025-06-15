# 11. Testing Setup

Configure a comprehensive testing pyramid with Vitest, React Testing Library, and Playwright according to our architecture standards.

## Prerequisites

- Vite project with React and TypeScript
- Node.js 22+ and pnpm installed

## 1. Configure Vitest for Unit and Component Tests

### Install Dependencies

```bash
pnpm add -D vitest @vitejs/plugin-react jsdom @testing-library/jest-dom
```

### Configure Vite/Vitest

Update `vite.config.ts` to include Vitest configuration:

```typescript
/// <reference types="vitest/config" />
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    include: ['src/**/*.{test,spec}.{js,mjs,cjs,ts,mts,cts,jsx,tsx}'],
    exclude: ['**/node_modules/**', '**/dist/**', '**/e2e/**'],
    coverage: {
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/test/',
        '**/*.d.ts',
        '**/*.config.{js,ts}',
        '**/index.{js,ts}',
      ],
    },
  },
})
```

### Create Test Setup File

Create `src/test/setup.ts`:

```typescript
import '@testing-library/jest-dom'
import { vi } from 'vitest'

// Mock global objects
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: vi.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: vi.fn(), // deprecated
    removeListener: vi.fn(), // deprecated
    addEventListener: vi.fn(),
    removeEventListener: vi.fn(),
    dispatchEvent: vi.fn(),
  })),
})

// Mock IntersectionObserver
global.IntersectionObserver = vi.fn().mockImplementation(() => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
}))

// Mock ResizeObserver
global.ResizeObserver = vi.fn().mockImplementation(() => ({
  observe: vi.fn(),
  unobserve: vi.fn(),
  disconnect: vi.fn(),
}))
```

### Update TypeScript Configuration

Add to `tsconfig.json`:

```json
{
  "compilerOptions": {
    "types": ["vitest/globals", "@testing-library/jest-dom"]
  }
}
```

### Add Test Scripts

Update `package.json`:

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui"
  }
}
```

## 2. Set up React Testing Library

### Install Dependencies

```bash
pnpm add -D @testing-library/react @testing-library/user-event
```

### Create Test Utilities

Create `src/test/utils.tsx`:

```typescript
import { render, RenderOptions } from '@testing-library/react'
import { ReactElement, ReactNode } from 'react'
import { BrowserRouter } from 'react-router-dom'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

// Create providers wrapper for tests
const AllTheProviders = ({ children }: { children: ReactNode }) => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  })

  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        {children}
      </BrowserRouter>
    </QueryClientProvider>
  )
}

// Custom render function with providers
const customRender = (
  ui: ReactElement,
  options?: Omit<RenderOptions, 'wrapper'>
) => render(ui, { wrapper: AllTheProviders, ...options })

export * from '@testing-library/react'
export { customRender as render }
export { default as userEvent } from '@testing-library/user-event'
```

### Example Component Test

Create `src/shared/ui/button/button.test.tsx`:

```typescript
import { render, screen } from '@/test/utils'
import userEvent from '@testing-library/user-event'
import { Button } from './button'

describe('Button', () => {
  test('renders with correct text', () => {
    render(<Button>Click me</Button>)
    expect(screen.getByRole('button', { name: /click me/i })).toBeInTheDocument()
  })

  test('calls onClick handler when clicked', async () => {
    const user = userEvent.setup()
    const handleClick = vi.fn()
    
    render(<Button onClick={handleClick}>Click me</Button>)
    
    await user.click(screen.getByRole('button'))
    expect(handleClick).toHaveBeenCalledTimes(1)
  })

  test('applies variant classes correctly', () => {
    render(<Button variant="destructive">Delete</Button>)
    expect(screen.getByRole('button')).toHaveClass('bg-destructive')
  })
})
```

## 3. Configure Playwright for E2E Testing

### Install Playwright

```bash
pnpm create playwright@latest
```

This creates:
- `playwright.config.ts` - Configuration file
- `tests/` - Test directory
- `tests-examples/` - Example tests

### Configure Playwright

Update `playwright.config.ts`:

```typescript
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ['html'],
    ['json', { outputFile: 'playwright-report/results.json' }],
  ],
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    // Mobile testing
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
  ],
  webServer: {
    command: 'pnpm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
    timeout: 120 * 1000,
  },
})
```

### Example E2E Test

Create `e2e/auth.spec.ts`:

```typescript
import { test, expect } from '@playwright/test'

test.describe('Authentication Flow', () => {
  test('should login successfully', async ({ page }) => {
    await page.goto('/')
    
    // Navigate to login
    await page.getByRole('button', { name: /sign in/i }).click()
    
    // Fill login form
    await page.getByLabel(/email/i).fill('user@example.com')
    await page.getByLabel(/password/i).fill('password123')
    await page.getByRole('button', { name: /log in/i }).click()
    
    // Verify successful login
    await expect(page.getByText(/welcome back/i)).toBeVisible()
    await expect(page.getByRole('button', { name: /sign out/i })).toBeVisible()
  })

  test('should show error for invalid credentials', async ({ page }) => {
    await page.goto('/login')
    
    await page.getByLabel(/email/i).fill('invalid@example.com')
    await page.getByLabel(/password/i).fill('wrongpassword')
    await page.getByRole('button', { name: /log in/i }).click()
    
    await expect(page.getByText(/invalid credentials/i)).toBeVisible()
  })
})
```

## 4. Set up Visual Regression Testing

### Configure Visual Testing

Update `playwright.config.ts` to include visual testing configuration:

```typescript
export default defineConfig({
  // ... existing config
  expect: {
    timeout: 10000,
    toHaveScreenshot: {
      mode: 'local',
      maxDiffPixels: 100,
    },
    toMatchSnapshot: {
      maxDiffPixelRatio: 0.05,
    },
  },
  use: {
    // ... existing use config
    // Disable animations for consistent screenshots
    reducedMotion: 'reduce',
  },
})
```

### Example Visual Test

Create `e2e/visual/components.spec.ts`:

```typescript
import { test, expect } from '@playwright/test'

test.describe('Component Visual Tests', () => {
  test('button variants should match snapshots', async ({ page }) => {
    await page.goto('/storybook') // or component showcase page
    
    // Test primary button
    const primaryButton = page.getByTestId('button-primary')
    await expect(primaryButton).toHaveScreenshot('button-primary.png')
    
    // Test button states
    await primaryButton.hover()
    await expect(primaryButton).toHaveScreenshot('button-primary-hover.png')
  })

  test('dashboard layout should match snapshot', async ({ page }) => {
    await page.goto('/dashboard')
    
    // Wait for data to load
    await page.waitForLoadState('networkidle')
    
    // Hide dynamic content
    await page.addStyleTag({
      content: `
        [data-testid="timestamp"],
        [data-testid="dynamic-content"] {
          visibility: hidden !important;
        }
      `
    })
    
    await expect(page).toHaveScreenshot('dashboard-layout.png', {
      fullPage: true,
      mask: [page.getByTestId('user-avatar')],
    })
  })
})
```

## 5. Create Test Utilities and Mocks

### API Mocks

Create `src/test/mocks/handlers.ts`:

```typescript
import { http, HttpResponse } from 'msw'

export const handlers = [
  // Mock user authentication
  http.post('/api/auth/login', () => {
    return HttpResponse.json({
      user: { id: '1', email: 'user@example.com', name: 'Test User' },
      token: 'mock-jwt-token',
    })
  }),

  // Mock user profile
  http.get('/api/user/profile', () => {
    return HttpResponse.json({
      id: '1',
      email: 'user@example.com',
      name: 'Test User',
      avatar: 'https://example.com/avatar.jpg',
    })
  }),

  // Mock error response
  http.get('/api/protected', () => {
    return new HttpResponse(null, { status: 401 })
  }),
]
```

Create `src/test/mocks/server.ts`:

```typescript
import { setupServer } from 'msw/node'
import { handlers } from './handlers'

export const server = setupServer(...handlers)
```

### Custom Test Hooks

Create `src/test/hooks.ts`:

```typescript
import { renderHook, RenderHookOptions } from '@testing-library/react'
import { ReactNode } from 'react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  })

  return ({ children }: { children: ReactNode }) => (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}

export const renderHookWithProviders = <TProps, TResult>(
  hook: (props: TProps) => TResult,
  options?: RenderHookOptions<TProps>
) => {
  return renderHook(hook, {
    wrapper: createWrapper(),
    ...options,
  })
}
```

### Page Object Models (E2E)

Create `e2e/pages/login.page.ts`:

```typescript
import { Page, Locator } from '@playwright/test'

export class LoginPage {
  readonly page: Page
  readonly emailInput: Locator
  readonly passwordInput: Locator
  readonly loginButton: Locator
  readonly errorMessage: Locator

  constructor(page: Page) {
    this.page = page
    this.emailInput = page.getByLabel(/email/i)
    this.passwordInput = page.getByLabel(/password/i)
    this.loginButton = page.getByRole('button', { name: /log in/i })
    this.errorMessage = page.getByTestId('error-message')
  }

  async goto() {
    await this.page.goto('/login')
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email)
    await this.passwordInput.fill(password)
    await this.loginButton.click()
  }
}
```

## Script Commands

Add comprehensive test scripts to `package.json`:

```json
{
  "scripts": {
    "test": "vitest",
    "test:run": "vitest run",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui",
    "e2e": "playwright test",
    "e2e:ui": "playwright test --ui",
    "e2e:headed": "playwright test --headed",
    "e2e:debug": "playwright test --debug",
    "test:all": "pnpm test:run && pnpm e2e"
  }
}
```

## CI/CD Integration

### GitHub Actions Example

Create `.github/workflows/test.yml`:

```yaml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
        with:
          version: 8
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'pnpm'
      
      - name: Install dependencies
        run: pnpm install
      
      - name: Run unit tests
        run: pnpm test:coverage
      
      - name: Install Playwright browsers
        run: pnpm exec playwright install --with-deps
      
      - name: Run E2E tests
        run: pnpm e2e
      
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: test-results
          path: |
            coverage/
            playwright-report/
```

## Best Practices

### Component Testing
- Focus on user behavior, not implementation details
- Use accessible queries (ByRole, ByLabelText, ByText)
- Test component contracts and user interactions
- Mock external dependencies (APIs, third-party libraries)

### E2E Testing
- Test critical user journeys end-to-end
- Use page object models for complex pages
- Wait for network idle state before assertions
- Mask or hide dynamic content in visual tests

### General Testing
- Follow the testing pyramid: More unit tests, fewer E2E tests
- Write descriptive test names that explain the expected behavior
- Group related tests using `describe` blocks
- Use `test.only` and `test.skip` for development, but never commit them

---

**Next Steps**: Continue to [Build & Deployment](./12-build-deployment.md) for production preparation.