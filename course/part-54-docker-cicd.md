# Part 54: Docker & CI/CD สำหรับ TypeScript

## บทนำ

Docker และ CI/CD pipeline เป็นส่วนสำคัญของการพัฒนาซอฟต์แวร์สมัยใหม่ บทนี้จะครอบคลุมการตั้งค่า Docker สำหรับ TypeScript application การสร้าง CI/CD pipeline ด้วย GitHub Actions และ GitLab CI รวมถึงการ deploy สู่ production

---

## 1. Dockerfile สำหรับ TypeScript

### Basic Dockerfile

```dockerfile
# Dockerfile พื้นฐานสำหรับ TypeScript Node.js app
FROM node:20-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY tsconfig.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY src/ ./src/

# Build TypeScript
RUN npm run build

# Expose port
EXPOSE 3000

# Start app
CMD ["node", "dist/index.js"]
```

```typescript
// src/index.ts - TypeScript app ตัวอย่าง
import express from 'express';

const app = express();
const PORT = process.env.PORT || 3000;

app.use(express.json());

app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION || '1.0.0'
  });
});

app.get('/', (req, res) => {
  res.json({ message: 'TypeScript App is running!' });
});

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

export default app;
```

```json
// package.json
{
  "name": "typescript-docker-app",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node-dev --respawn src/index.ts",
    "test": "jest",
    "lint": "eslint src --ext .ts",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "@types/express": "^4.17.17",
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0",
    "ts-node-dev": "^2.0.0",
    "jest": "^29.0.0",
    "@types/jest": "^29.0.0",
    "ts-jest": "^29.0.0",
    "eslint": "^8.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0"
  }
}
```

---

## 2. Multi-Stage Builds

```dockerfile
# Multi-stage Dockerfile (Production-ready)
# Stage 1: Builder
FROM node:20-alpine AS builder

WORKDIR /app

# Install dependencies
COPY package*.json ./
COPY tsconfig.json ./
RUN npm ci --include=dev

# Build
COPY src/ ./src/
RUN npm run build
RUN npm prune --production

# Stage 2: Production Image (เล็กกว่ามาก)
FROM node:20-alpine AS production

# Security: ใช้ non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

WORKDIR /app

# Copy only built files and production dependencies
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=nextjs:nodejs /app/package.json ./

# Switch to non-root user
USER nextjs

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) })"

CMD ["node", "dist/index.js"]
```

```dockerfile
# Multi-stage สำหรับ Next.js
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000

CMD ["node", "server.js"]
```

```dockerfile
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
.env
.env.local
.env.*.local
dist
coverage
.nyc_output
*.test.ts
*.spec.ts
README.md
docker-compose*.yml
.github
.vscode
```

---

## 3. Docker Compose

```yaml
# docker-compose.yml - Development
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: builder
    ports:
      - "3000:3000"
    volumes:
      - ./src:/app/src
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - PORT=3000
      - DATABASE_URL=postgresql://user:password@postgres:5432/mydb
      - REDIS_URL=redis://redis:6379
    command: npm run dev
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydb
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

```yaml
# docker-compose.prod.yml - Production
version: '3.8'

services:
  app:
    image: myregistry/my-app:${APP_VERSION:-latest}
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - JWT_SECRET=${JWT_SECRET}
    secrets:
      - db_password
      - jwt_secret
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

secrets:
  db_password:
    external: true
  jwt_secret:
    external: true
```

```yaml
# docker-compose.test.yml - สำหรับ Testing
version: '3.8'

services:
  test:
    build:
      context: .
      target: builder
    command: npm test -- --coverage --forceExit
    environment:
      - NODE_ENV=test
      - DATABASE_URL=postgresql://user:password@test-postgres:5432/testdb
    depends_on:
      test-postgres:
        condition: service_healthy

  test-postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: testdb
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d testdb"]
      interval: 5s
      timeout: 3s
      retries: 5
    tmpfs:
      - /var/lib/postgresql/data
```

---

## 4. Environment Management

```typescript
// src/config/env.ts - Type-safe environment variables

import { z } from 'zod';

// Schema สำหรับ validate environment variables
const envSchema = z.object({
  // Server
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.string().transform(Number).default('3000'),
  HOST: z.string().default('0.0.0.0'),
  
  // Database
  DATABASE_URL: z.string().url(),
  DATABASE_POOL_MIN: z.string().transform(Number).default('2'),
  DATABASE_POOL_MAX: z.string().transform(Number).default('10'),
  
  // Redis
  REDIS_URL: z.string().url().optional(),
  REDIS_TTL: z.string().transform(Number).default('3600'),
  
  // Auth
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRES_IN: z.string().default('1d'),
  
  // App
  APP_VERSION: z.string().default('1.0.0'),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  
  // Optional: External Services
  SMTP_HOST: z.string().optional(),
  SMTP_PORT: z.string().transform(Number).optional(),
  SMTP_USER: z.string().optional(),
  SMTP_PASS: z.string().optional(),
  
  S3_BUCKET: z.string().optional(),
  S3_REGION: z.string().optional(),
  AWS_ACCESS_KEY_ID: z.string().optional(),
  AWS_SECRET_ACCESS_KEY: z.string().optional(),
});

// Parse และ validate
const parseEnv = () => {
  const result = envSchema.safeParse(process.env);
  
  if (!result.success) {
    console.error('❌ Invalid environment variables:');
    result.error.issues.forEach(issue => {
      console.error(`  ${issue.path.join('.')}: ${issue.message}`);
    });
    process.exit(1);
  }
  
  return result.data;
};

export const env = parseEnv();
export type Env = typeof env;

// การใช้งาน
// import { env } from './config/env';
// console.log(env.DATABASE_URL);
// console.log(env.PORT); // number, ไม่ใช่ string
```

```bash
# .env.example - Template สำหรับ developers
NODE_ENV=development
PORT=3000
HOST=0.0.0.0

DATABASE_URL=postgresql://user:password@localhost:5432/mydb
DATABASE_POOL_MIN=2
DATABASE_POOL_MAX=10

REDIS_URL=redis://localhost:6379

JWT_SECRET=your-super-secret-jwt-key-min-32-chars
JWT_EXPIRES_IN=1d

APP_VERSION=1.0.0
LOG_LEVEL=debug

# Optional
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=noreply@example.com
SMTP_PASS=smtp-password
```

---

## 5. GitHub Actions - Build และ Test

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '20'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Job 1: Lint และ Type Check
  lint-and-typecheck:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: TypeScript Type Check
        run: npm run typecheck
  
  # Job 2: Unit Tests
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      
      - name: Run unit tests
        run: npm test -- --coverage --testPathPattern=unit
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
  
  # Job 3: Integration Tests
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: unit-tests
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    env:
      DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
      REDIS_URL: redis://localhost:6379
      JWT_SECRET: test-secret-key-min-32-characters-long
      NODE_ENV: test
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      
      - name: Run database migrations
        run: npm run db:migrate
      
      - name: Run integration tests
        run: npm test -- --testPathPattern=integration
  
  # Job 4: Build
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [lint-and-typecheck, unit-tests]
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - run: npm ci
      - run: npm run build
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: dist
          path: dist/
          retention-days: 7
  
  # Job 5: Docker Build
  docker-build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'push'
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            APP_VERSION=${{ github.sha }}
```

---

## 6. GitHub Actions - Deployment

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches:
      - main
    tags:
      - 'v*'

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.myapp.com
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build and push to ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: my-app
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
      
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster my-staging-cluster \
            --service my-app-service \
            --force-new-deployment \
            --region ap-southeast-1
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster my-staging-cluster \
            --services my-app-service \
            --region ap-southeast-1
      
      - name: Run smoke tests
        run: |
          curl -f https://staging.myapp.com/health || exit 1

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://myapp.com
    if: startsWith(github.ref, 'refs/tags/v')
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
          aws-region: ap-southeast-1
      
      - name: Deploy to Production ECS
        run: |
          aws ecs update-service \
            --cluster my-prod-cluster \
            --service my-app-service \
            --force-new-deployment \
            --region ap-southeast-1
      
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true

  notify:
    name: Notify Slack
    runs-on: ubuntu-latest
    needs: [deploy-staging, deploy-production]
    if: always()
    
    steps:
      - name: Send Slack notification
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployment ${{ job.status }}: ${{ github.repository }} - ${{ github.ref }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 7. GitLab CI

```yaml
# .gitlab-ci.yml
image: node:20-alpine

stages:
  - install
  - test
  - build
  - docker
  - deploy-staging
  - deploy-production

variables:
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKER_DRIVER: overlay2
  IMAGE_NAME: $CI_REGISTRY_IMAGE
  NODE_ENV: test

# Cache
.node_cache: &node_cache
  cache:
    key:
      files:
        - package-lock.json
    paths:
      - node_modules/
    policy: pull-push

# Install
install:
  stage: install
  <<: *node_cache
  script:
    - npm ci
  only:
    - merge_requests
    - main
    - tags

# Lint
lint:
  stage: test
  <<: *node_cache
  cache:
    policy: pull
  script:
    - npm run lint
    - npm run typecheck
  dependencies:
    - install

# Unit Tests
unit-tests:
  stage: test
  <<: *node_cache
  cache:
    policy: pull
  script:
    - npm test -- --coverage --testPathPattern=unit
  coverage: '/Lines\s*:\s*(\d+\.?\d*)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
      junit: test-results/junit.xml
    when: always
    expire_in: 1 week
  dependencies:
    - install

# Integration Tests
integration-tests:
  stage: test
  services:
    - postgres:15-alpine
    - redis:7-alpine
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
    DATABASE_URL: "postgresql://testuser:testpass@postgres:5432/testdb"
    REDIS_URL: "redis://redis:6379"
    JWT_SECRET: "test-secret-key-min-32-characters-long"
  script:
    - npm run db:migrate
    - npm test -- --testPathPattern=integration
  dependencies:
    - install
  only:
    - merge_requests
    - main

# Build
build:
  stage: build
  <<: *node_cache
  cache:
    policy: pull
  script:
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 hour
  dependencies:
    - install

# Docker Build
docker-build:
  stage: docker
  image: docker:24
  services:
    - docker:24-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      docker build \
        --cache-from $IMAGE_NAME:latest \
        --tag $IMAGE_NAME:$CI_COMMIT_SHA \
        --tag $IMAGE_NAME:latest \
        --build-arg APP_VERSION=$CI_COMMIT_SHA \
        .
    - docker push $IMAGE_NAME:$CI_COMMIT_SHA
    - docker push $IMAGE_NAME:latest
  only:
    - main
    - tags
  dependencies:
    - build

# Deploy Staging
deploy-staging:
  stage: deploy-staging
  image: alpine:3.18
  environment:
    name: staging
    url: https://staging.myapp.com
  before_script:
    - apk add --no-cache curl openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - ssh-keyscan -H $STAGING_HOST >> ~/.ssh/known_hosts
  script:
    - |
      ssh deploy@$STAGING_HOST "
        docker pull $IMAGE_NAME:$CI_COMMIT_SHA
        docker stop my-app || true
        docker rm my-app || true
        docker run -d \
          --name my-app \
          --restart unless-stopped \
          -p 3000:3000 \
          -e NODE_ENV=staging \
          -e DATABASE_URL=$DATABASE_URL \
          -e JWT_SECRET=$JWT_SECRET \
          $IMAGE_NAME:$CI_COMMIT_SHA
      "
    - sleep 10
    - curl -f https://staging.myapp.com/health
  only:
    - main

# Deploy Production
deploy-production:
  stage: deploy-production
  image: alpine:3.18
  environment:
    name: production
    url: https://myapp.com
  when: manual
  script:
    - echo "Deploying to production..."
  only:
    - tags
```

---

## 8. Health Check Endpoints

```typescript
// src/health/health.controller.ts
import { Request, Response, Router } from 'express';
import { HealthCheckService } from './health.service';

const router = Router();
const healthService = new HealthCheckService();

// Basic health check
router.get('/health', (req: Request, res: Response) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString()
  });
});

// Detailed health check
router.get('/health/detailed', async (req: Request, res: Response) => {
  const health = await healthService.checkAll();
  
  const statusCode = health.status === 'healthy' ? 200 : 503;
  res.status(statusCode).json(health);
});

// Liveness probe (Kubernetes)
router.get('/health/live', (req: Request, res: Response) => {
  res.status(200).json({ status: 'alive' });
});

// Readiness probe (Kubernetes)
router.get('/health/ready', async (req: Request, res: Response) => {
  const isReady = await healthService.isReady();
  
  if (isReady) {
    res.status(200).json({ status: 'ready' });
  } else {
    res.status(503).json({ status: 'not ready' });
  }
});

export { router as healthRouter };

// src/health/health.service.ts
interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  version: string;
  uptime: number;
  checks: {
    database: CheckResult;
    redis: CheckResult;
    memory: CheckResult;
    disk?: CheckResult;
  };
}

interface CheckResult {
  status: 'up' | 'down' | 'degraded';
  latency?: number;
  message?: string;
  details?: any;
}

class HealthCheckService {
  constructor(
    private db?: Database,
    private redis?: RedisClient
  ) {}
  
  async checkAll(): Promise<HealthStatus> {
    const [dbCheck, redisCheck, memoryCheck] = await Promise.allSettled([
      this.checkDatabase(),
      this.checkRedis(),
      this.checkMemory()
    ]);
    
    const checks = {
      database: this.getResult(dbCheck),
      redis: this.getResult(redisCheck),
      memory: this.getResult(memoryCheck)
    };
    
    const allUp = Object.values(checks).every(c => c.status === 'up');
    const anyDown = Object.values(checks).some(c => c.status === 'down');
    
    return {
      status: allUp ? 'healthy' : anyDown ? 'unhealthy' : 'degraded',
      timestamp: new Date().toISOString(),
      version: process.env.APP_VERSION || '1.0.0',
      uptime: process.uptime(),
      checks
    };
  }
  
  async isReady(): Promise<boolean> {
    try {
      if (this.db) {
        await this.checkDatabase();
      }
      return true;
    } catch {
      return false;
    }
  }
  
  private async checkDatabase(): Promise<CheckResult> {
    if (!this.db) {
      return { status: 'up', message: 'No database configured' };
    }
    
    const start = Date.now();
    try {
      await this.db.query('SELECT 1');
      return {
        status: 'up',
        latency: Date.now() - start
      };
    } catch (error) {
      return {
        status: 'down',
        message: error instanceof Error ? error.message : 'Database check failed'
      };
    }
  }
  
  private async checkRedis(): Promise<CheckResult> {
    if (!this.redis) {
      return { status: 'up', message: 'No Redis configured' };
    }
    
    const start = Date.now();
    try {
      await this.redis.ping();
      return {
        status: 'up',
        latency: Date.now() - start
      };
    } catch (error) {
      return {
        status: 'down',
        message: error instanceof Error ? error.message : 'Redis check failed'
      };
    }
  }
  
  private checkMemory(): CheckResult {
    const used = process.memoryUsage();
    const heapUsedMB = Math.round(used.heapUsed / 1024 / 1024);
    const heapTotalMB = Math.round(used.heapTotal / 1024 / 1024);
    const percentage = (heapUsedMB / heapTotalMB) * 100;
    
    return {
      status: percentage > 90 ? 'degraded' : 'up',
      details: {
        heapUsed: `${heapUsedMB} MB`,
        heapTotal: `${heapTotalMB} MB`,
        usage: `${percentage.toFixed(1)}%`,
        rss: `${Math.round(used.rss / 1024 / 1024)} MB`
      }
    };
  }
  
  private getResult(result: PromiseSettledResult<CheckResult>): CheckResult {
    if (result.status === 'fulfilled') {
      return result.value;
    }
    return {
      status: 'down',
      message: result.reason?.message || 'Check failed'
    };
  }
}
```

---

## 9. Kubernetes Basics

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-typescript-app
  namespace: production
  labels:
    app: my-typescript-app
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-typescript-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-typescript-app
    spec:
      containers:
        - name: app
          image: myregistry/my-app:1.0.0
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: "production"
            - name: PORT
              value: "3000"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
            - name: JWT_SECRET
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: jwt-secret
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health/live
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 30
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: app-config
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - my-typescript-app
                topologyKey: kubernetes.io/hostname

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-typescript-app
  namespace: production
spec:
  selector:
    app: my-typescript-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-typescript-app
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.com
      secretName: myapp-tls
  rules:
    - host: myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-typescript-app
                port:
                  number: 80

---
# k8s/hpa.yaml - Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-typescript-app-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-typescript-app
  minReplicas: 2
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

---

## 10. Monitoring Setup

```typescript
// src/monitoring/metrics.ts - Prometheus metrics
import { Counter, Histogram, Gauge, register } from 'prom-client';

// HTTP Request metrics
export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.001, 0.01, 0.1, 0.5, 1, 2, 5, 10]
});

export const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections'
});

// Business metrics
export const ordersCreated = new Counter({
  name: 'orders_created_total',
  help: 'Total number of orders created',
  labelNames: ['status']
});

export const orderRevenue = new Counter({
  name: 'order_revenue_total',
  help: 'Total revenue from orders',
  labelNames: ['currency']
});

// Middleware สำหรับ collect metrics
export const metricsMiddleware = (req: any, res: any, next: any) => {
  const start = Date.now();
  
  activeConnections.inc();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const labels = {
      method: req.method,
      route: req.route?.path || req.path,
      status_code: res.statusCode.toString()
    };
    
    httpRequestsTotal.inc(labels);
    httpRequestDuration.observe(labels, duration);
    activeConnections.dec();
  });
  
  next();
};

// Metrics endpoint
export const metricsRouter = Router();
metricsRouter.get('/metrics', async (req: Request, res: Response) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});
```

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'my-typescript-app'
    static_configs:
      - targets: ['app:3000']
    metrics_path: '/metrics'
    scrape_interval: 10s

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

```yaml
# monitoring/docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
    ports:
      - "3001:3000"
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:
```

---

## 11. Complete Production Deployment Example

```yaml
# .github/workflows/production-deploy.yml
name: Production Deployment

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  validate:
    name: Validate Release
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Get version
        id: version
        run: echo "version=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
      
      - name: Validate version format
        run: |
          echo "Deploying version: ${{ steps.version.outputs.version }}"
          [[ "${{ steps.version.outputs.version }}" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]] || exit 1

  test:
    name: Full Test Suite
    runs-on: ubuntu-latest
    needs: validate
    
    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    env:
      DATABASE_URL: postgresql://testuser:testpass@localhost:5432/testdb
      JWT_SECRET: test-secret-key-min-32-characters-long
      NODE_ENV: test
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run db:migrate
      - run: npm test -- --coverage --forceExit
      
      - name: Check coverage threshold
        run: |
          coverage=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          echo "Coverage: $coverage%"
          [ $(echo "$coverage >= 80" | bc) -eq 1 ] || (echo "Coverage below 80%!" && exit 1)

  build-and-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: test
    
    permissions:
      contents: read
      packages: write
    
    outputs:
      image: ${{ steps.push.outputs.image }}
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: docker/setup-buildx-action@v3
      
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        id: push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ needs.validate.outputs.version }}
            ghcr.io/${{ github.repository }}:latest
          build-args: |
            APP_VERSION=${{ needs.validate.outputs.version }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - run: echo "image=ghcr.io/${{ github.repository }}:${{ needs.validate.outputs.version }}" >> $GITHUB_OUTPUT

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [validate, build-and-push]
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
          aws-region: ap-southeast-1
      
      - name: Update Kubernetes deployment
        run: |
          aws eks update-kubeconfig \
            --region ap-southeast-1 \
            --name my-prod-cluster
          
          kubectl set image deployment/my-typescript-app \
            app=${{ needs.build-and-push.outputs.image }} \
            -n production
          
          kubectl rollout status deployment/my-typescript-app \
            -n production \
            --timeout=300s
      
      - name: Verify deployment
        run: |
          sleep 30
          curl -f https://myapp.com/health || exit 1
          echo "Deployment successful!"
      
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          generate_release_notes: true
          body: |
            ## Deployment
            
            Version: ${{ needs.validate.outputs.version }}
            Image: ${{ needs.build-and-push.outputs.image }}
            
            Deployed to production at $(date -u)

  rollback:
    name: Rollback on Failure
    runs-on: ubuntu-latest
    needs: deploy-production
    if: failure()
    
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID_PROD }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY_PROD }}
          aws-region: ap-southeast-1
      
      - name: Rollback Kubernetes deployment
        run: |
          aws eks update-kubeconfig \
            --region ap-southeast-1 \
            --name my-prod-cluster
          
          kubectl rollout undo deployment/my-typescript-app \
            -n production
          
          kubectl rollout status deployment/my-typescript-app \
            -n production \
            --timeout=300s
      
      - name: Notify rollback
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": ":warning: Production deployment FAILED and rolled back for version ${{ needs.validate.outputs.version }}"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## สรุป

Docker และ CI/CD เป็นส่วนสำคัญของ modern software development:

1. **Multi-stage Dockerfile** ช่วยลดขนาด image และเพิ่มความปลอดภัย
2. **Docker Compose** ช่วยจัดการหลาย services ในการพัฒนาและ test
3. **GitHub Actions / GitLab CI** ช่วย automate build, test, และ deploy
4. **Health Checks** ทำให้ Kubernetes และ load balancer ตรวจสอบสถานะได้
5. **Monitoring** ด้วย Prometheus และ Grafana ช่วย track performance
6. **Rollback Strategy** เตรียมพร้อมสำหรับกรณีที่ deployment ล้มเหลว
