### The Official Architecture Convention

This document is the single source of truth for the architecture, conventions, and best practices used in our React applications. Adherence to this guide is not optional; it is essential for maintaining code quality, scalability, and developer velocity.

### 0. Guiding Principles

Our architecture is built on three core principles:

1.  **Pragmatism over Dogma:** We use proven, stable technologies that solve real problems. The goal is to ship robust products, not to chase trends.
2.  **Long-Term Maintainability:** We write code for the developer who will maintain it in six months—which is likely ourselves. Clarity, predictability, and ease of refactoring are paramount.
3.  **Strict Separation of Concerns:** We enforce strong boundaries between UI, client state, and server state. This is the key to managing complexity.

### 1. High-Level Strategy: Unified vs. Separated Frontends

We choose the architecture that best fits the project requirements. For most projects, a unified approach reduces complexity and accelerates development.

#### 1.1. Unified SPA (Recommended for most projects) 

**When to use:** Landing pages (2-5 pages) + core application under authentication

*   **Single Vite + React project** with **TanStack Router** handling both public and private routes
*   **Public routes** (`/`, `/pricing`, `/about`) serve as marketing pages with proper SEO meta tags
*   **Private routes** (`/dashboard/*`) require authentication and serve the core application
*   **Shared components** between landing and app without additional tooling complexity
*   **Single deployment** pipeline and domain

#### 1.2. Separated Architecture (For complex cases)

**When to use:** Extensive marketing site (10+ pages) or different teams maintaining each part

*   **Core Applications (Dashboards, Tools):** Built as a **Single-Page Application (SPA)** using **Vite + React**. The primary focus is rich interactivity and complex state management.
*   **Marketing & Content Sites (Landings, Public Pages):** Built as a separate project using **Astro**. The primary focus is exceptional performance (Time-to-First-Byte), static site generation (SSG), and Search Engine Optimization (SEO).
    *   **Component Sharing:** Astro integrates seamlessly with React. You **must** reuse components from our core application's `shared/ui` library within the Astro project to maintain visual consistency.

### 2. The Core Technology Stack

This is the officially sanctioned technology stack. Do not introduce new libraries for these core functionalities without an architectural review.

| Category | Technology | Convention |
| :--- | :--- | :--- |
| **Bundler & Framework** | Vite & React 18+ | Standard configuration. Leverage modern React APIs (Hooks). |
| **Language** | TypeScript | `strict` mode is enabled and required. |
| **Architecture** | Feature-Sliced Design (FSD) | Provides a clear, scalable structure. Details below. |
| **Routing** | TanStack Router | For 100% type-safe routing in the SPA. |
| **Server State** | TanStack Query v5 | The **only** way to manage server data. |
| **Client State** | Zustand | For minimal, non-persistent UI state only. |
| **UI Components** | shadcn/ui + Tailwind CSS | The foundation of our design system. |
| **Forms** | React Hook Form + Zod | For performant forms and schema-based validation. |
| **API Client** | Axios | For all REST-based communication. |
| **Testing** | Vitest, RTL, Playwright | For a comprehensive testing pyramid. |
| **Code Style** | ESLint (Airbnb) + Prettier | Non-negotiable for code consistency. |

### 3. Directory Structure: Feature-Sliced Design (FSD)

FSD is the mandatory structure for our SPA projects. It enforces our core principles of separation and scalability.

```
src/
├── app/          # App setup: providers, router, global styles.
├── pages/        # Composition layer: Assembles widgets and features onto a route.
├── widgets/      # Complex UI blocks: Header, Sidebar, Cards with actions.
├── features/     # User actions: AuthByGoogle, AddToCart, UpdateProfile.
├── entities/     # Business nouns: User, Product, Order (models, API hooks, UI cards).
└── shared/       # Code with no business logic: UI kit, API config, helpers.
```

**The Golden Rules of FSD:**

1.  **The Dependency Rule:** Layers can only depend on layers below them (e.g., `features` can use `entities`, but not the other way around). This is enforced by ESLint rules.
2.  **The Public API Rule:** Every slice must have an `index.ts` that exports only what is needed externally. This creates encapsulated modules that are safe to refactor.

### 4. Key Architectural Patterns & Conventions

#### 4.1. State Management

*   **Server State:** Managed **exclusively** by **TanStack Query**. Never store server data in a Zustand store. Use `useQuery` for fetching data and `useMutation` for creating, updating, or deleting data. The `queryKey` is the source of truth for caching.
*   **Client State:** Managed **exclusively** by **Zustand**. Only use it for ephemeral UI state that does not need to be persisted on the server.
    *   **Example:** `create((set) => ({ isSidebarOpen: false, toggleSidebar: () => set((state) => ({ isSidebarOpen: !state.isSidebarOpen })) }))`
    *   **Anti-Pattern:** Storing user profile data from an API call in a Zustand store. This is the job of TanStack Query.

#### 4.2. API Communication

All API logic resides in `shared/api/` or within the `model/` directory of an `entities` slice.

*   **RESTful APIs:**
    *   Use the globally configured **Axios** instance from `shared/api/axios-instance.ts`.
    *   **Type Generation:** When consuming a REST API with an OpenAPI (Swagger) spec, use **Orval** to automatically generate TypeScript types and TanStack Query hooks. Commit the generated files to the repository.
*   **tRPC:** When using a tRPC backend, the typed client provides end-to-end type safety. TanStack Query's `useQuery` and `useMutation` are the preferred wrappers around tRPC procedures.
*   **WebSockets:** Encapsulate WebSocket logic within a custom hook (e.g., `useRealtimeUpdates`) that exposes a simple API to the components. This hook should be placed in the relevant `feature` or `entity` slice.

#### 4.3. Authentication

*   **Flow:** We use an OAuth 2.0 flow. The frontend redirects to the provider (e.g., Google), receives a `code` on callback, sends this `code` to our backend, and receives a **JWT** in return.
*   **Token Storage:** The JWT **must** be stored in `localStorage`.
*   **Request Authorization:** The Axios instance must have an **interceptor** that automatically reads the token from `localStorage` and attaches the `Authorization: Bearer <token>` header to every outgoing request.
*   **Auth State:** The user's authentication status (`isAuthenticated`, `user` object) is managed by a dedicated Zustand store (`useAuthStore`). This store is hydrated from `localStorage` on application load.

#### 4.4. Routing & Page Structure (Unified SPA)

For projects using the unified SPA approach, organize routes to clearly separate public and private sections:

*   **Public Routes:** Landing pages accessible without authentication
    ```
    /                    → Home/Landing page
    /pricing            → Pricing page  
    /about              → About page
    /auth/login         → Login page
    /auth/register      → Registration page
    ```

*   **Private Routes:** Application pages requiring authentication
    ```
    /dashboard          → Main dashboard (protected)
    /dashboard/settings → User settings (protected)
    /dashboard/profile  → User profile (protected)
    ```

*   **Route Structure:**
    ```typescript
    // routes/index.tsx - Public root
    export const Route = createRootRoute({
      component: RootComponent,
    })
    
    // routes/_auth.tsx - Private layout with auth guard
    export const Route = createFileRoute('/_auth')({
      beforeLoad: ({ context }) => {
        if (!context.auth.isAuthenticated) {
          throw redirect({ to: '/auth/login' })
        }
      },
      component: AuthLayout,
    })
    ```

*   **SEO for Public Routes:** Use TanStack Router's meta capabilities for proper SEO on landing pages
*   **Page Organization:** Follow FSD structure - public pages in `pages/landing/`, private pages in `pages/dashboard/`

#### 4.5. UI & Styling

*   **Component Source:** All `shadcn/ui` components reside in `src/shared/ui/`.
*   **Class Merging:** Always use the `cn` utility (a combination of `clsx` and `tailwind-merge`) from `shared/lib/utils.ts` for conditional or combined class names. This prevents style conflicts.
*   **Theming:** Themes are managed via CSS variables in `tailwind.config.js` and applied in the root `ThemeProvider` in the `app/` layer.

#### 4.6. Forms

*   **Implementation:** All forms **must** be built with **React Hook Form**.
*   **Validation:** All form validation **must** be done using **Zod**. Define a Zod schema and pass it to React Hook Form's resolver. This ensures data is valid from the form all the way to the API request.
    ```tsx
    const formSchema = z.object({ email: z.string().email() });
    const form = useForm({ resolver: zodResolver(formSchema) });
    ```

#### 4.7. Testing Strategy

We employ a "testing pyramid" strategy to ensure confidence and maintainable test suites.

1.  **Unit & Component Tests (Base of the pyramid):**
    *   **Tools:** `Vitest` + `React Testing Library (RTL)`.
    *   **Location:** Co-located with the source code (`component.test.tsx`).
    *   **Focus:** Test component behavior from a user's perspective. Find elements by accessible roles. Do not test implementation details.

2.  **E2E (End-to-End) Tests (Top of the pyramid):**
    *   **Tool:** `Playwright`.
    *   **Location:** In a separate `/e2e` directory at the project root.
    *   **Focus:** Test critical user flows across multiple pages (e.g., login -> create a resource -> edit it -> logout).

3.  **Visual Regression Tests:**
    *   **Tool:** `Playwright`'s built-in screenshot assertion (`await expect(page).toHaveScreenshot()`).
    *   **Focus:** Used for key `shared/ui` components and critical `widgets` to prevent unintended visual changes. Store snapshots in the repository.

### 5. Deployment

*   **Target:** Kubernetes.
*   **Method:** All applications must be containerized using a multi-stage `Dockerfile`.
    *   **Build Stage:** Use a `node:22-alpine` image to install dependencies and run `pnpm run build`.
    *   **Serve Stage:** Use a lightweight `nginx:alpine` image. Copy the build output from the `dist` folder into the Nginx server directory. Include a basic `nginx.conf` optimized for serving a Single-Page Application (i.e., redirecting all 404s to `index.html`).
