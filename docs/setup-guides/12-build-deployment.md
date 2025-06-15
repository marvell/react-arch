# 12. Build & Deployment Preparation

This comprehensive guide covers production-ready build optimization and deployment strategies for React SPAs using our architectural standards.

## Table of Contents
- [Multi-Stage Docker Build](#multi-stage-docker-build)
- [Nginx Configuration for SPA](#nginx-configuration-for-spa)
- [Vite Build Optimization](#vite-build-optimization)
- [Environment Variables Management](#environment-variables-management)
- [Kubernetes Deployment](#kubernetes-deployment)

---

## Multi-Stage Docker Build

Following our architecture's containerization requirements, use this optimized multi-stage Dockerfile that separates build and runtime concerns:

### Complete Dockerfile

```dockerfile
# Build stage
FROM node:22-alpine AS builder

# Install pnpm globally
RUN npm install -g pnpm

# Set working directory
WORKDIR /app

# Copy package files for dependency installation
COPY package.json pnpm-lock.yaml ./

# Install dependencies with cache mount for faster builds
RUN --mount=type=cache,target=/root/.local/share/pnpm/store \
    pnpm install --frozen-lockfile

# Copy source code
COPY . .

# Build the application
RUN pnpm run build

# Production stage
FROM nginx:alpine AS production

# Copy custom nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy built assets from builder stage
COPY --from=builder /app/dist /usr/share/nginx/html

# Add labels for better container management
LABEL maintainer="your-team@company.com"
LABEL version="1.0"
LABEL description="React SPA with Nginx"

# Expose port 80
EXPOSE 80

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/health || exit 1

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

### Key Optimizations

1. **Layer Caching**: Dependencies are installed before copying source code
2. **Cache Mounts**: pnpm store is cached between builds using `--mount=type=cache`
3. **Frozen Lockfile**: Ensures reproducible builds with `--frozen-lockfile`
4. **Multi-Stage**: Build artifacts are copied to a minimal runtime image
5. **Health Check**: Built-in container health monitoring

---

## Nginx Configuration for SPA

SPAs require special nginx configuration to handle client-side routing. Here's the production-ready configuration:

### nginx.conf

```nginx
server {
    listen 80;
    server_name localhost;
    
    root /usr/share/nginx/html;
    index index.html;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # Enable gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/javascript
        application/json
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    # Handle client-side routing - CRITICAL for SPAs
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache static assets aggressively
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }

    # Cache JSON with shorter duration
    location ~* \.json$ {
        expires 1d;
        add_header Cache-Control "public";
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # Block access to sensitive files
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

### Why This Configuration Works

1. **`try_files $uri $uri/ /index.html`**: Routes all unmatched URLs to index.html for client-side routing
2. **Aggressive Caching**: Static assets cached for 1 year with immutable flag
3. **Security Headers**: Essential security headers for production
4. **Compression**: Reduces bundle sizes significantly
5. **Health Check**: Kubernetes-compatible health endpoint

---

## Vite Build Optimization

Optimize your Vite build configuration for production performance:

### vite.config.ts

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { resolve } from 'path'

export default defineConfig({
  plugins: [react()],
  
  // Build optimizations
  build: {
    // Disable source maps in production for smaller bundles
    sourcemap: false,
    
    // Disable compressed size reporting for faster builds
    reportCompressedSize: false,
    
    // Optimize chunk splitting
    rollupOptions: {
      output: {
        manualChunks: {
          // Separate vendor chunks for better caching
          vendor: ['react', 'react-dom'],
          router: ['@tanstack/react-router'],
          query: ['@tanstack/react-query'],
          ui: ['@radix-ui/react-dialog', '@radix-ui/react-dropdown-menu'],
        },
      },
    },
    
    // Target modern browsers as per architecture
    target: 'esnext',
    
    // Minification settings
    minify: 'esbuild', // Faster than terser
    
    // CSS minification
    cssMinify: 'esbuild',
  },

  // Performance optimizations
  optimizeDeps: {
    include: [
      'react',
      'react-dom',
      '@tanstack/react-router',
      '@tanstack/react-query',
    ],
  },

  // Path resolution
  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
      '@/shared': resolve(__dirname, './src/shared'),
      '@/entities': resolve(__dirname, './src/entities'),
      '@/features': resolve(__dirname, './src/features'),
      '@/widgets': resolve(__dirname, './src/widgets'),
      '@/pages': resolve(__dirname, './src/pages'),
      '@/app': resolve(__dirname, './src/app'),
    },
  },
})
```

### Advanced Build Script

Add this to your `package.json`:

```json
{
  "scripts": {
    "build": "tsc && vite build",
    "build:analyze": "npm run build && npx vite-bundle-analyzer dist",
    "build:preview": "npm run build && vite preview"
  }
}
```

### Bundle Analysis

Monitor your bundle size with regular analysis:

```bash
# Analyze bundle composition
pnpm build:analyze

# Preview production build locally
pnpm build:preview
```

---

## Environment Variables Management

### Production Environment Variables

Create environment-specific `.env` files following Vite's conventions:

#### .env.production

```bash
# Base configuration
VITE_APP_TITLE=Production App
VITE_API_BASE_URL=https://api.yourapp.com
VITE_APP_ENV=production

# Feature flags
VITE_ENABLE_ANALYTICS=true
VITE_ENABLE_ERROR_REPORTING=true

# External services
VITE_SENTRY_DSN=your-sentry-dsn
VITE_GOOGLE_ANALYTICS_ID=GA-XXXXXXX
```

#### .env.staging

```bash
VITE_APP_TITLE=Staging App
VITE_API_BASE_URL=https://staging-api.yourapp.com
VITE_APP_ENV=staging
VITE_ENABLE_ANALYTICS=false
VITE_ENABLE_ERROR_REPORTING=true
```

### Environment Variable Usage

```typescript
// src/shared/config/env.ts
export const env = {
  APP_TITLE: import.meta.env.VITE_APP_TITLE,
  API_BASE_URL: import.meta.env.VITE_API_BASE_URL,
  APP_ENV: import.meta.env.VITE_APP_ENV,
  ENABLE_ANALYTICS: import.meta.env.VITE_ENABLE_ANALYTICS === 'true',
  ENABLE_ERROR_REPORTING: import.meta.env.VITE_ENABLE_ERROR_REPORTING === 'true',
} as const

// Type safety for environment variables
interface ImportMetaEnv {
  readonly VITE_APP_TITLE: string
  readonly VITE_API_BASE_URL: string
  readonly VITE_APP_ENV: 'development' | 'staging' | 'production'
  readonly VITE_ENABLE_ANALYTICS: string
  readonly VITE_ENABLE_ERROR_REPORTING: string
}
```

### Build-Time Validation

Add environment validation to your build process:

```typescript
// scripts/validate-env.ts
import { z } from 'zod'

const envSchema = z.object({
  VITE_API_BASE_URL: z.string().url(),
  VITE_APP_ENV: z.enum(['development', 'staging', 'production']),
  VITE_ENABLE_ANALYTICS: z.enum(['true', 'false']),
})

try {
  envSchema.parse(import.meta.env)
  console.log('✅ Environment variables validated')
} catch (error) {
  console.error('❌ Invalid environment variables:', error)
  process.exit(1)
}
```

---

## Kubernetes Deployment

Deploy your containerized React SPA to Kubernetes following production best practices:

### Deployment Manifest

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: react-spa
  labels:
    app: react-spa
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: react-spa
  template:
    metadata:
      labels:
        app: react-spa
        version: v1
    spec:
      containers:
      - name: react-spa
        image: your-registry/react-spa:latest
        ports:
        - containerPort: 80
          name: http
        
        # Resource management
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        
        # Health checks
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        
        # Environment variables
        env:
        - name: NODE_ENV
          value: "production"
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        fsGroup: 101
```

### Service Configuration

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: react-spa-service
  labels:
    app: react-spa
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: react-spa
```

### Ingress Configuration

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: react-spa-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - yourapp.com
    secretName: react-spa-tls
  rules:
  - host: yourapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: react-spa-service
            port:
              number: 80
```

### Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: react-spa-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: react-spa
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### Deployment Commands

```bash
# Apply all configurations
kubectl apply -f k8s/

# Monitor deployment
kubectl rollout status deployment/react-spa

# Check pods
kubectl get pods -l app=react-spa

# View logs
kubectl logs -l app=react-spa --tail=100 -f

# Scale manually if needed
kubectl scale deployment react-spa --replicas=5
```

---

## Complete Build and Deploy Pipeline

### Build Script

```bash
#!/bin/bash
# scripts/build-deploy.sh

set -e

# Environment validation
echo "🔍 Validating environment..."
npm run validate-env

# Type checking
echo "🔧 Type checking..."
npm run typecheck

# Linting
echo "🧹 Linting..."
npm run lint

# Testing
echo "🧪 Running tests..."
npm run test

# Build
echo "🏗️ Building application..."
npm run build

# Build Docker image
echo "🐳 Building Docker image..."
docker build -t your-registry/react-spa:latest .

# Push to registry (if in CI/CD)
if [ "$CI" = "true" ]; then
  echo "📤 Pushing to registry..."
  docker push your-registry/react-spa:latest
fi

echo "✅ Build complete!"
```

### Make it executable

```bash
chmod +x scripts/build-deploy.sh
```

---

## Checklist

- [ ] ✅ Multi-stage Dockerfile implemented
- [ ] ✅ Nginx configuration for SPA routing
- [ ] ✅ Vite build optimization configured
- [ ] ✅ Environment variables properly managed
- [ ] ✅ Kubernetes manifests created
- [ ] ✅ Health checks implemented
- [ ] ✅ Resource limits configured
- [ ] ✅ Autoscaling setup
- [ ] ✅ Build pipeline script created

---

**Next Steps**: Continue to [First Feature](./13-first-feature.md) for implementation workflow.