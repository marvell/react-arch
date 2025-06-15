# 6. UI Foundation

This guide establishes the complete UI foundation for your React application using Tailwind CSS and shadcn/ui, following our architectural conventions.

## Prerequisites

- Vite + React + TypeScript project initialized
- Node.js 22+ and npm/pnpm installed

## Step 1: Install and Configure Tailwind CSS

### 1.1 Install Tailwind CSS

```bash
npm install -D tailwindcss postcss autoprefixer @types/node
npx tailwindcss init -p
```

### 1.2 Configure TypeScript Path Mapping

Update your TypeScript configuration for path aliases:

**tsconfig.json:**
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

**tsconfig.app.json:**
```json
{
  "compilerOptions": {
    // ... existing config
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
    // ... rest of config
  }
}
```

### 1.3 Update Vite Configuration

**vite.config.ts:**
```typescript
import path from "path"
import react from "@vitejs/plugin-react"
import { defineConfig } from "vite"

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
})
```

### 1.4 Configure Tailwind

**tailwind.config.js:**
```javascript
/** @type {import('tailwindcss').Config} */
export default {
  darkMode: ["class"],
  content: [
    "./index.html",
    "./src/**/*.{js,ts,jsx,tsx}",
  ],
  prefix: "",
  theme: {
    container: {
      center: true,
      padding: "2rem",
      screens: {
        "2xl": "1400px",
      },
    },
    extend: {
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        secondary: {
          DEFAULT: "hsl(var(--secondary))",
          foreground: "hsl(var(--secondary-foreground))",
        },
        destructive: {
          DEFAULT: "hsl(var(--destructive))",
          foreground: "hsl(var(--destructive-foreground))",
        },
        muted: {
          DEFAULT: "hsl(var(--muted))",
          foreground: "hsl(var(--muted-foreground))",
        },
        accent: {
          DEFAULT: "hsl(var(--accent))",
          foreground: "hsl(var(--accent-foreground))",
        },
        popover: {
          DEFAULT: "hsl(var(--popover))",
          foreground: "hsl(var(--popover-foreground))",
        },
        card: {
          DEFAULT: "hsl(var(--card))",
          foreground: "hsl(var(--card-foreground))",
        },
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
      keyframes: {
        "accordion-down": {
          from: { height: "0" },
          to: { height: "var(--radix-accordion-content-height)" },
        },
        "accordion-up": {
          from: { height: "var(--radix-accordion-content-height)" },
          to: { height: "0" },
        },
      },
      animation: {
        "accordion-down": "accordion-down 0.2s ease-out",
        "accordion-up": "accordion-up 0.2s ease-out",
      },
    },
  },
  plugins: [require("tailwindcss-animate")],
}
```

## Step 2: Set up shadcn/ui Components

### 2.1 Initialize shadcn/ui

```bash
npx shadcn@latest init
```

Configure during the interactive setup:
- **Style**: New York
- **Base color**: Zinc
- **CSS variables**: Yes

### 2.2 Create CSS Variables

**src/index.css:**
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 221.2 83.2% 53.3%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96%;
    --secondary-foreground: 222.2 84% 4.9%;
    --muted: 210 40% 96%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96%;
    --accent-foreground: 222.2 84% 4.9%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 221.2 83.2% 53.3%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;
    --primary: 217.2 91.2% 59.8%;
    --primary-foreground: 222.2 84% 4.9%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 224.3 76.3% 94.1%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

### 2.3 Install Required Dependencies

```bash
npm install tailwindcss-animate class-variance-authority clsx tailwind-merge lucide-react
```

## Step 3: Configure Theme System and CSS Variables

### 3.1 Create Utility Functions

**src/shared/lib/utils.ts:**
```typescript
import { type ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

### 3.2 Set up Theme Provider

**src/shared/ui/theme-provider.tsx:**
```typescript
import { createContext, useContext, useEffect, useState } from "react"

type Theme = "dark" | "light" | "system"

type ThemeProviderProps = {
  children: React.ReactNode
  defaultTheme?: Theme
  storageKey?: string
}

type ThemeProviderState = {
  theme: Theme
  setTheme: (theme: Theme) => void
}

const initialState: ThemeProviderState = {
  theme: "system",
  setTheme: () => null,
}

const ThemeProviderContext = createContext<ThemeProviderState>(initialState)

export function ThemeProvider({
  children,
  defaultTheme = "system",
  storageKey = "vite-ui-theme",
  ...props
}: ThemeProviderProps) {
  const [theme, setTheme] = useState<Theme>(
    () => (localStorage.getItem(storageKey) as Theme) || defaultTheme
  )

  useEffect(() => {
    const root = window.document.documentElement

    root.classList.remove("light", "dark")

    if (theme === "system") {
      const systemTheme = window.matchMedia("(prefers-color-scheme: dark)")
        .matches
        ? "dark"
        : "light"

      root.classList.add(systemTheme)
      return
    }

    root.classList.add(theme)
  }, [theme])

  const value = {
    theme,
    setTheme: (theme: Theme) => {
      localStorage.setItem(storageKey, theme)
      setTheme(theme)
    },
  }

  return (
    <ThemeProviderContext.Provider {...props} value={value}>
      {children}
    </ThemeProviderContext.Provider>
  )
}

export const useTheme = () => {
  const context = useContext(ThemeProviderContext)

  if (context === undefined)
    throw new Error("useTheme must be used within a ThemeProvider")

  return context
}
```

## Step 4: Create Base UI Components in shared/ui

### 4.1 Install Core Components

Install essential shadcn/ui components:

```bash
npx shadcn@latest add button
npx shadcn@latest add input
npx shadcn@latest add label
npx shadcn@latest add card
npx shadcn@latest add avatar
npx shadcn@latest add badge
npx shadcn@latest add dialog
npx shadcn@latest add dropdown-menu
npx shadcn@latest add toast
npx shadcn@latest add form
```

### 4.2 Create Component Index

**src/shared/ui/index.ts:**
```typescript
// Export all shadcn/ui components for easy importing
export { Button, buttonVariants } from "./button"
export { Input } from "./input"
export { Label } from "./label"
export { Card, CardHeader, CardFooter, CardTitle, CardDescription, CardContent } from "./card"
export { Avatar, AvatarImage, AvatarFallback } from "./avatar"
export { Badge, badgeVariants } from "./badge"
export { Dialog, DialogTrigger, DialogContent, DialogHeader, DialogFooter, DialogTitle, DialogDescription } from "./dialog"
export { DropdownMenu, DropdownMenuTrigger, DropdownMenuContent, DropdownMenuItem, DropdownMenuCheckboxItem, DropdownMenuRadioItem, DropdownMenuLabel, DropdownMenuSeparator, DropdownMenuShortcut, DropdownMenuGroup, DropdownMenuPortal, DropdownMenuSub, DropdownMenuSubContent, DropdownMenuSubTrigger, DropdownMenuRadioGroup } from "./dropdown-menu"
export { toast, useToast } from "./use-toast"
export { Toaster } from "./toaster"
export { ThemeProvider, useTheme } from "./theme-provider"

// Re-export utility function
export { cn } from "../lib/utils"
```

### 4.3 Create Custom Components

**src/shared/ui/logo.tsx:**
```typescript
import { cn } from "../lib/utils"

interface LogoProps {
  className?: string
  size?: "sm" | "md" | "lg"
}

export function Logo({ className, size = "md" }: LogoProps) {
  const sizeClasses = {
    sm: "h-6 w-6",
    md: "h-8 w-8", 
    lg: "h-12 w-12"
  }

  return (
    <div className={cn(
      "rounded-md bg-primary flex items-center justify-center text-primary-foreground font-bold",
      sizeClasses[size],
      className
    )}>
      <span className="text-sm">L</span>
    </div>
  )
}
```

### 4.4 Set up App-level Theme Integration

**src/app/providers/theme-provider.tsx:**
```typescript
import { ThemeProvider as BaseThemeProvider } from "@/shared/ui"

interface ThemeProviderProps {
  children: React.ReactNode
}

export function ThemeProvider({ children }: ThemeProviderProps) {
  return (
    <BaseThemeProvider defaultTheme="system" storageKey="app-theme">
      {children}
    </BaseThemeProvider>
  )
}
```

### 4.5 Update App Component

**src/app/App.tsx:**
```typescript
import { ThemeProvider } from "./providers/theme-provider"
import { Toaster } from "@/shared/ui"

function App() {
  return (
    <ThemeProvider>
      <div className="min-h-screen bg-background font-sans antialiased">
        {/* Your app content goes here */}
        <main className="container mx-auto py-6">
          <h1 className="text-3xl font-bold">Welcome to Your App</h1>
        </main>
        <Toaster />
      </div>
    </ThemeProvider>
  )
}

export default App
```

## Step 5: Testing Your Setup

### 5.1 Create a Test Component

**src/shared/ui/theme-toggle.tsx:**
```typescript
import { Moon, Sun } from "lucide-react"
import { Button } from "./button"
import { useTheme } from "./theme-provider"

export function ThemeToggle() {
  const { setTheme, theme } = useTheme()

  return (
    <Button
      variant="outline"
      size="icon"
      onClick={() => setTheme(theme === "light" ? "dark" : "light")}
    >
      <Sun className="h-[1.2rem] w-[1.2rem] rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0" />
      <Moon className="absolute h-[1.2rem] w-[1.2rem] rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100" />
      <span className="sr-only">Toggle theme</span>
    </Button>
  )
}
```

### 5.2 Test Your Components

Add the theme toggle to your app to test the setup:

```typescript
import { ThemeToggle, Button, Card, CardContent, CardHeader, CardTitle } from "@/shared/ui"

// In your component
<div className="flex justify-between items-center">
  <h1>My App</h1>
  <ThemeToggle />
</div>

<Card className="w-full max-w-md">
  <CardHeader>
    <CardTitle>Welcome</CardTitle>
  </CardHeader>
  <CardContent>
    <Button className="w-full">Get Started</Button>
  </CardContent>
</Card>
```

## Verification Checklist

- [ ] ✅ Tailwind CSS installed and configured with custom theme
- [ ] ✅ shadcn/ui components installed and accessible via `@/shared/ui`
- [ ] ✅ CSS variables working for theme customization
- [ ] ✅ Dark/light mode switching functional
- [ ] ✅ `cn()` utility function available for class merging
- [ ] ✅ Components follow FSD structure in `shared/ui`
- [ ] ✅ TypeScript path aliases working (`@/*`)
- [ ] ✅ Theme provider integrated at app level

## Architecture Compliance

This setup follows our architectural conventions:
- **shadcn/ui + Tailwind CSS**: Official UI foundation stack
- **CSS Variables**: Enable dynamic theming as required
- **FSD Structure**: All UI components in `shared/ui` layer
- **cn() Utility**: Required for proper class name merging
- **TypeScript**: Full type safety throughout the UI system

---

**Next Steps**: Continue to [Routing Setup](./07-routing-setup.md) for navigation configuration.