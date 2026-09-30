# ส่วนที่ 75: Monitoring และ Observability ใน TypeScript

## บทนำ

Observability เป็นความสามารถในการเข้าใจสถานะภายในของระบบจากข้อมูลที่แสดงออกมาภายนอก ประกอบด้วย 3 เสาหลัก: Logs, Metrics, และ Traces

---

## 1. Logging ด้วย Winston

### 1.1 การติดตั้งและ Setup พื้นฐาน

```typescript
import winston from 'winston';
import { Format } from 'logform';

// Custom log levels
const customLevels = {
  levels: {
    fatal: 0,
    error: 1,
    warn: 2,
    info: 3,
    http: 4,
    debug: 5,
    trace: 6,
  },
  colors: {
    fatal: 'red bold',
    error: 'red',
    warn: 'yellow',
    info: 'green',
    http: 'magenta',
    debug: 'blue',
    trace: 'cyan',
  }
};

// เพิ่ม colors
winston.addColors(customLevels.colors);

// Structured log format
const structuredFormat: Format = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DDTHH:mm:ss.SSSZ' }),
  winston.format.errors({ stack: true }),
  winston.format.metadata({ fillExcept: ['message', 'level', 'timestamp', 'label'] }),
  winston.format.json()
);

// Console format สำหรับ development
const consoleFormat: Format = winston.format.combine(
  winston.format.colorize({ all: true }),
  winston.format.timestamp({ format: 'HH:mm:ss' }),
  winston.format.printf(({ timestamp, level, message, ...meta }) => {
    const metaStr = Object.keys(meta).length ? JSON.stringify(meta, null, 2) : '';
    return `${timestamp} [${level}] ${message} ${metaStr}`;
  })
);

// สร้าง logger
const logger = winston.createLogger({
  levels: customLevels.levels,
  level: process.env.LOG_LEVEL ?? 'info',
  format: structuredFormat,
  defaultMeta: {
    service: process.env.SERVICE_NAME ?? 'my-service',
    version: process.env.APP_VERSION ?? '1.0.0',
    environment: process.env.NODE_ENV ?? 'development'
  },
  transports: [
    // Console สำหรับ development
    ...(process.env.NODE_ENV !== 'production' ? [
      new winston.transports.Console({ format: consoleFormat })
    ] : []),

    // File transport
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      maxsize: 5 * 1024 * 1024, // 5MB
      maxFiles: 5,
      tailable: true
    }),

    new winston.transports.File({
      filename: 'logs/combined.log',
      maxsize: 10 * 1024 * 1024, // 10MB
      maxFiles: 10
    })
  ],
  exceptionHandlers: [
    new winston.transports.File({ filename: 'logs/exceptions.log' })
  ],
  rejectionHandlers: [
    new winston.transports.File({ filename: 'logs/rejections.log' })
  ]
});

export default logger;
```

### 1.2 Structured Logging

```typescript
// Context-aware logger
interface LogContext {
  requestId?: string;
  userId?: string;
  tenantId?: string;
  traceId?: string;
  spanId?: string;
  [key: string]: unknown;
}

class ContextLogger {
  private context: LogContext;

  constructor(
    private baseLogger: winston.Logger,
    context: LogContext = {}
  ) {
    this.context = context;
  }

  withContext(additionalContext: LogContext): ContextLogger {
    return new ContextLogger(this.baseLogger, {
      ...this.context,
      ...additionalContext
    });
  }

  debug(message: string, meta?: Record<string, unknown>): void {
    this.baseLogger.debug(message, { ...this.context, ...meta });
  }

  info(message: string, meta?: Record<string, unknown>): void {
    this.baseLogger.info(message, { ...this.context, ...meta });
  }

  warn(message: string, meta?: Record<string, unknown>): void {
    this.baseLogger.warn(message, { ...this.context, ...meta });
  }

  error(message: string, error?: Error, meta?: Record<string, unknown>): void {
    this.baseLogger.error(message, {
      ...this.context,
      ...meta,
      ...(error ? {
        errorMessage: error.message,
        errorStack: error.stack,
        errorName: error.name
      } : {})
    });
  }

  fatal(message: string, error?: Error, meta?: Record<string, unknown>): void {
    this.baseLogger.log('fatal', message, {
      ...this.context,
      ...meta,
      ...(error ? {
        errorMessage: error.message,
        errorStack: error.stack
      } : {})
    });
  }

  http(message: string, meta?: Record<string, unknown>): void {
    this.baseLogger.http(message, { ...this.context, ...meta });
  }
}

// Async Local Storage สำหรับ request context
import { AsyncLocalStorage } from 'async_hooks';

const logContext = new AsyncLocalStorage<LogContext>();

function getLogger(): ContextLogger {
  const context = logContext.getStore() ?? {};
  return new ContextLogger(logger, context);
}

// Express middleware สำหรับ inject request context
function requestLoggingMiddleware(req: Request, res: Response, next: NextFunction): void {
  const context: LogContext = {
    requestId: req.headers['x-request-id'] as string ?? generateId(),
    userId: (req as any).user?.id,
    tenantId: req.headers['x-tenant-id'] as string
  };

  res.setHeader('x-request-id', context.requestId!);

  logContext.run(context, () => {
    const startTime = Date.now();
    const log = getLogger();

    log.http('Incoming request', {
      method: req.method,
      url: req.url,
      userAgent: req.headers['user-agent'],
      ip: req.ip
    });

    res.on('finish', () => {
      const duration = Date.now() - startTime;
      log.http('Request completed', {
        method: req.method,
        url: req.url,
        statusCode: res.statusCode,
        durationMs: duration
      });
    });

    next();
  });
}
```

---

## 2. Logging ด้วย Pino

```typescript
import pino from 'pino';
import pinoHttp from 'pino-http';

// Pino มีประสิทธิภาพสูงกว่า Winston มาก
const pinoLogger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  base: {
    pid: process.pid,
    service: process.env.SERVICE_NAME ?? 'my-service',
    version: process.env.APP_VERSION ?? '1.0.0'
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  redact: {
    paths: ['req.headers.authorization', 'req.headers.cookie', '*.password', '*.token'],
    censor: '[REDACTED]'
  },
  serializers: {
    err: pino.stdSerializers.err,
    req: pino.stdSerializers.req,
    res: pino.stdSerializers.res
  },
  transport: process.env.NODE_ENV !== 'production' ? {
    target: 'pino-pretty',
    options: {
      colorize: true,
      translateTime: 'SYS:standard',
      ignore: 'pid,hostname'
    }
  } : undefined
});

// HTTP middleware สำหรับ Pino
const httpLogger = pinoHttp({
  logger: pinoLogger,
  customLogLevel: (req, res, err) => {
    if (res.statusCode >= 500 || err) return 'error';
    if (res.statusCode >= 400) return 'warn';
    return 'info';
  },
  customSuccessMessage: (req, res) =>
    `${req.method} ${req.url} completed with status ${res.statusCode}`,
  customErrorMessage: (req, res, err) =>
    `${req.method} ${req.url} failed: ${err.message}`
});

app.use(httpLogger);
```

---

## 3. Metrics ด้วย Prometheus

```typescript
import client from 'prom-client';

// Enable default metrics (CPU, memory, etc.)
const register = new client.Registry();
client.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  registers: [register]
});

const httpRequestTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register]
});

const activeConnections = new client.Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  registers: [register]
});

const databaseQueryDuration = new client.Histogram({
  name: 'database_query_duration_seconds',
  help: 'Database query duration in seconds',
  labelNames: ['operation', 'table'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
  registers: [register]
});

const cacheHitRatio = new client.Gauge({
  name: 'cache_hit_ratio',
  help: 'Cache hit ratio (0-1)',
  labelNames: ['cache_type'],
  registers: [register]
});

const errorTotal = new client.Counter({
  name: 'errors_total',
  help: 'Total number of errors',
  labelNames: ['type', 'service'],
  registers: [register]
});

// Middleware สำหรับ HTTP metrics
function metricsMiddleware(req: Request, res: Response, next: NextFunction): void {
  const start = process.hrtime.bigint();

  res.on('finish', () => {
    const durationNs = process.hrtime.bigint() - start;
    const durationSec = Number(durationNs) / 1e9;
    const route = req.route?.path ?? req.path;

    httpRequestDuration
      .labels(req.method, route, String(res.statusCode))
      .observe(durationSec);

    httpRequestTotal
      .labels(req.method, route, String(res.statusCode))
      .inc();
  });

  next();
}

// Metrics endpoint
app.get('/metrics', async (req: Request, res: Response) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Database metrics wrapper
function trackDatabaseQuery<T>(
  operation: string,
  table: string,
  fn: () => Promise<T>
): Promise<T> {
  const end = databaseQueryDuration.labels(operation, table).startTimer();
  return fn().finally(() => end());
}
```

---

## 4. Health Checks

```typescript
interface HealthCheck {
  name: string;
  check(): Promise<HealthStatus>;
}

interface HealthStatus {
  status: 'healthy' | 'degraded' | 'unhealthy';
  message?: string;
  details?: Record<string, unknown>;
  duration?: number;
}

interface SystemHealth {
  status: 'healthy' | 'degraded' | 'unhealthy';
  timestamp: string;
  version: string;
  uptime: number;
  checks: Record<string, HealthStatus & { duration: number }>;
}

class HealthCheckService {
  private checks: HealthCheck[] = [];

  register(check: HealthCheck): this {
    this.checks.push(check);
    return this;
  }

  async getHealth(): Promise<SystemHealth> {
    const results: Record<string, HealthStatus & { duration: number }> = {};

    await Promise.all(
      this.checks.map(async (check) => {
        const start = Date.now();
        try {
          const status = await Promise.race([
            check.check(),
            new Promise<HealthStatus>((_, reject) =>
              setTimeout(() => reject(new Error('Health check timeout')), 5000)
            )
          ]);
          results[check.name] = { ...status, duration: Date.now() - start };
        } catch (error) {
          results[check.name] = {
            status: 'unhealthy',
            message: (error as Error).message,
            duration: Date.now() - start
          };
        }
      })
    );

    const statuses = Object.values(results).map(r => r.status);
    const overallStatus = statuses.includes('unhealthy')
      ? 'unhealthy'
      : statuses.includes('degraded')
      ? 'degraded'
      : 'healthy';

    return {
      status: overallStatus,
      timestamp: new Date().toISOString(),
      version: process.env.APP_VERSION ?? '1.0.0',
      uptime: process.uptime(),
      checks: results
    };
  }
}

// ตัวอย่าง health checks
class DatabaseHealthCheck implements HealthCheck {
  name = 'database';

  constructor(private db: Database) {}

  async check(): Promise<HealthStatus> {
    const start = Date.now();
    try {
      await this.db.query('SELECT 1');
      return {
        status: 'healthy',
        details: { responseTime: Date.now() - start }
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        message: (error as Error).message
      };
    }
  }
}

class RedisHealthCheck implements HealthCheck {
  name = 'redis';

  constructor(private redis: RedisClient) {}

  async check(): Promise<HealthStatus> {
    try {
      await this.redis.ping();
      const info = await this.redis.info();
      return {
        status: 'healthy',
        details: {
          version: info.match(/redis_version:([^\r\n]+)/)?.[1]
        }
      };
    } catch (error) {
      return {
        status: 'unhealthy',
        message: (error as Error).message
      };
    }
  }
}

class MemoryHealthCheck implements HealthCheck {
  name = 'memory';

  constructor(private thresholdMB: number = 1000) {}

  async check(): Promise<HealthStatus> {
    const memUsage = process.memoryUsage();
    const heapUsedMB = memUsage.heapUsed / 1024 / 1024;
    const heapTotalMB = memUsage.heapTotal / 1024 / 1024;
    const usagePercent = (heapUsedMB / heapTotalMB) * 100;

    const status: HealthStatus['status'] =
      heapUsedMB > this.thresholdMB ? 'unhealthy'
      : usagePercent > 90 ? 'degraded'
      : 'healthy';

    return {
      status,
      details: {
        heapUsedMB: Math.round(heapUsedMB),
        heapTotalMB: Math.round(heapTotalMB),
        usagePercent: Math.round(usagePercent),
        rss: Math.round(memUsage.rss / 1024 / 1024)
      }
    };
  }
}

// Setup health check endpoints
const healthService = new HealthCheckService()
  .register(new DatabaseHealthCheck(db))
  .register(new MemoryHealthCheck(512));

app.get('/health', async (req: Request, res: Response) => {
  const health = await healthService.getHealth();
  const statusCode = health.status === 'unhealthy' ? 503
    : health.status === 'degraded' ? 207
    : 200;
  res.status(statusCode).json(health);
});

// Liveness probe (ระบบยังทำงานอยู่ไหม)
app.get('/health/liveness', (req: Request, res: Response) => {
  res.json({ status: 'alive', timestamp: new Date().toISOString() });
});

// Readiness probe (ระบบพร้อมรับ traffic ไหม)
app.get('/health/readiness', async (req: Request, res: Response) => {
  const health = await healthService.getHealth();
  if (health.status === 'unhealthy') {
    return res.status(503).json(health);
  }
  res.json(health);
});
```

---

## 5. Distributed Tracing ด้วย OpenTelemetry

```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { Resource } from '@opentelemetry/resources';
import { SemanticResourceAttributes } from '@opentelemetry/semantic-conventions';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { SimpleSpanProcessor } from '@opentelemetry/sdk-trace-base';
import { trace, context, propagation, SpanStatusCode, SpanKind } from '@opentelemetry/api';
import { W3CTraceContextPropagator } from '@opentelemetry/core';

// Setup OpenTelemetry SDK
const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: process.env.SERVICE_NAME ?? 'my-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION ?? '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: process.env.NODE_ENV ?? 'development'
  }),
  spanProcessor: new SimpleSpanProcessor(
    new OTLPTraceExporter({
      url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT ?? 'http://localhost:4318/v1/traces'
    })
  )
});

sdk.start();

// Graceful shutdown
process.on('SIGTERM', () => {
  sdk.shutdown().finally(() => process.exit(0));
});

// Tracer
const tracer = trace.getTracer('my-service', '1.0.0');

// Tracing helper functions
async function withSpan<T>(
  name: string,
  fn: (span: Span) => Promise<T>,
  options?: SpanOptions
): Promise<T> {
  const span = tracer.startSpan(name, options);
  const ctx = trace.setSpan(context.active(), span);

  return context.with(ctx, async () => {
    try {
      const result = await fn(span);
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: (error as Error).message
      });
      span.recordException(error as Error);
      throw error;
    } finally {
      span.end();
    }
  });
}

// Database tracing wrapper
class TracedDatabasePool extends DatabasePool {
  async query<T>(sql: string, params?: unknown[]): Promise<T[]> {
    return withSpan('db.query', async (span) => {
      // Extract table name from SQL
      const table = sql.match(/(?:FROM|INTO|UPDATE)\s+(\w+)/i)?.[1] ?? 'unknown';
      const operation = sql.trim().split(' ')[0].toLowerCase();

      span.setAttributes({
        'db.system': 'postgresql',
        'db.operation': operation,
        'db.sql.table': table,
        'db.statement': sql.substring(0, 500) // Truncate long queries
      });

      return super.query<T>(sql, params);
    }, {
      kind: SpanKind.CLIENT,
      attributes: { 'db.system': 'postgresql' }
    });
  }
}

// HTTP tracing middleware
function tracingMiddleware(req: Request, res: Response, next: NextFunction): void {
  const tracer = trace.getTracer('http');
  const parentContext = propagation.extract(context.active(), req.headers);

  const span = tracer.startSpan(
    `${req.method} ${req.route?.path ?? req.path}`,
    {
      kind: SpanKind.SERVER,
      attributes: {
        'http.method': req.method,
        'http.url': req.url,
        'http.host': req.hostname,
        'http.scheme': req.protocol,
        'http.user_agent': req.headers['user-agent']
      }
    },
    parentContext
  );

  const ctx = trace.setSpan(context.active(), span);

  context.with(ctx, () => {
    res.on('finish', () => {
      span.setAttributes({
        'http.status_code': res.statusCode
      });

      if (res.statusCode >= 500) {
        span.setStatus({ code: SpanStatusCode.ERROR });
      }

      span.end();
    });

    // Inject trace context into response headers
    propagation.inject(ctx, res, {
      set: (carrier: any, key: string, value: string) => {
        carrier.setHeader(key, value);
      }
    });

    next();
  });
}
```

---

## 6. Error Tracking ด้วย Sentry

```typescript
import * as Sentry from '@sentry/node';
import { ProfilingIntegration } from '@sentry/profiling-node';

// Initialize Sentry
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV ?? 'development',
  release: process.env.APP_VERSION ?? '1.0.0',
  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
    new Sentry.Integrations.Express({ app }),
    new ProfilingIntegration()
  ],
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  profilesSampleRate: 1.0,
  beforeSend(event, hint) {
    // Filter out expected errors
    if (event.exception) {
      const error = hint.originalException;
      if (error instanceof NotFoundError) return null;
      if (error instanceof ValidationError) return null;
    }
    return event;
  }
});

// Error tracking wrapper
class ErrorTracker {
  static captureError(error: Error, context?: Record<string, unknown>): string {
    const eventId = Sentry.captureException(error, {
      extra: context
    });
    return eventId;
  }

  static setUser(user: { id: string; email?: string; username?: string }): void {
    Sentry.setUser(user);
  }

  static clearUser(): void {
    Sentry.setUser(null);
  }

  static addBreadcrumb(message: string, data?: Record<string, unknown>): void {
    Sentry.addBreadcrumb({
      message,
      data,
      timestamp: Date.now() / 1000
    });
  }

  static withTransaction<T>(name: string, fn: () => Promise<T>): Promise<T> {
    const transaction = Sentry.startTransaction({ name, op: 'function' });
    Sentry.getCurrentHub().configureScope(scope => {
      scope.setSpan(transaction);
    });

    return fn()
      .then(result => {
        transaction.setStatus('ok');
        return result;
      })
      .catch(error => {
        transaction.setStatus('internal_error');
        throw error;
      })
      .finally(() => {
        transaction.finish();
      });
  }
}

// Express error handler ด้วย Sentry
app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.tracingHandler());

// ... routes ...

app.use(Sentry.Handlers.errorHandler({
  shouldHandleError: (error) => {
    if (error.status === 404 || error.status === 422) return false;
    return true;
  }
}));
```

---

## 7. Performance Monitoring

```typescript
class PerformanceMonitor {
  private metrics: Map<string, number[]> = new Map();

  async measure<T>(name: string, fn: () => Promise<T>): Promise<T> {
    const start = process.hrtime.bigint();
    try {
      return await fn();
    } finally {
      const duration = Number(process.hrtime.bigint() - start) / 1e6; // ms
      this.record(name, duration);
    }
  }

  record(name: string, value: number): void {
    if (!this.metrics.has(name)) {
      this.metrics.set(name, []);
    }
    const values = this.metrics.get(name)!;
    values.push(value);

    // Keep last 1000 samples
    if (values.length > 1000) {
      values.shift();
    }
  }

  getStats(name: string): PerformanceStats | null {
    const values = this.metrics.get(name);
    if (!values?.length) return null;

    const sorted = [...values].sort((a, b) => a - b);
    const sum = values.reduce((a, b) => a + b, 0);

    return {
      count: values.length,
      min: sorted[0],
      max: sorted[sorted.length - 1],
      mean: sum / values.length,
      median: sorted[Math.floor(sorted.length / 2)],
      p95: sorted[Math.floor(sorted.length * 0.95)],
      p99: sorted[Math.floor(sorted.length * 0.99)]
    };
  }

  getAllStats(): Record<string, PerformanceStats> {
    const result: Record<string, PerformanceStats> = {};
    for (const name of this.metrics.keys()) {
      const stats = this.getStats(name);
      if (stats) result[name] = stats;
    }
    return result;
  }

  reset(name?: string): void {
    if (name) {
      this.metrics.delete(name);
    } else {
      this.metrics.clear();
    }
  }
}

interface PerformanceStats {
  count: number;
  min: number;
  max: number;
  mean: number;
  median: number;
  p95: number;
  p99: number;
}

// ตัวอย่างการใช้งาน
const perfMonitor = new PerformanceMonitor();

async function getUserFromCache(id: string): Promise<User | null> {
  return perfMonitor.measure('cache.get', async () => {
    return cache.get(id);
  });
}

// Endpoint สำหรับดู performance stats
app.get('/metrics/performance', (req: Request, res: Response) => {
  res.json(perfMonitor.getAllStats());
});
```

---

## 8. Alerting

```typescript
interface AlertRule {
  name: string;
  condition: () => Promise<boolean>;
  message: string;
  severity: 'info' | 'warning' | 'critical';
  cooldownMs: number;
}

interface AlertChannel {
  send(alert: Alert): Promise<void>;
}

interface Alert {
  rule: string;
  message: string;
  severity: AlertRule['severity'];
  timestamp: Date;
  context?: Record<string, unknown>;
}

class AlertManager {
  private rules: AlertRule[] = [];
  private channels: AlertChannel[] = [];
  private lastFired: Map<string, Date> = new Map();

  addRule(rule: AlertRule): this {
    this.rules.push(rule);
    return this;
  }

  addChannel(channel: AlertChannel): this {
    this.channels.push(channel);
    return this;
  }

  async checkAll(): Promise<void> {
    for (const rule of this.rules) {
      try {
        const triggered = await rule.condition();
        if (!triggered) continue;

        const lastFire = this.lastFired.get(rule.name);
        const now = new Date();

        if (lastFire && (now.getTime() - lastFire.getTime()) < rule.cooldownMs) {
          continue; // In cooldown period
        }

        this.lastFired.set(rule.name, now);
        const alert: Alert = {
          rule: rule.name,
          message: rule.message,
          severity: rule.severity,
          timestamp: now
        };

        await this.fireAlert(alert);
      } catch (error) {
        console.error(`Error checking rule ${rule.name}:`, error);
      }
    }
  }

  private async fireAlert(alert: Alert): Promise<void> {
    logger.warn('Alert fired', { alert });
    await Promise.all(this.channels.map(c => c.send(alert)));
  }

  startMonitoring(intervalMs: number = 60000): NodeJS.Timeout {
    return setInterval(() => this.checkAll(), intervalMs);
  }
}

// Slack alert channel
class SlackAlertChannel implements AlertChannel {
  constructor(private webhookUrl: string) {}

  async send(alert: Alert): Promise<void> {
    const emoji = {
      info: 'ℹ️',
      warning: '⚠️',
      critical: '🚨'
    }[alert.severity];

    const payload = {
      text: `${emoji} *Alert: ${alert.rule}*`,
      attachments: [{
        color: alert.severity === 'critical' ? 'danger'
          : alert.severity === 'warning' ? 'warning'
          : 'good',
        fields: [
          { title: 'Message', value: alert.message, short: false },
          { title: 'Severity', value: alert.severity, short: true },
          { title: 'Time', value: alert.timestamp.toISOString(), short: true }
        ]
      }]
    };

    await fetch(this.webhookUrl, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(payload)
    });
  }
}

// ตัวอย่าง Alert Rules
const alertManager = new AlertManager()
  .addChannel(new SlackAlertChannel(process.env.SLACK_WEBHOOK_URL!))
  .addRule({
    name: 'high-memory-usage',
    condition: async () => {
      const { heapUsed, heapTotal } = process.memoryUsage();
      return (heapUsed / heapTotal) > 0.9;
    },
    message: 'Memory usage is above 90%',
    severity: 'warning',
    cooldownMs: 5 * 60 * 1000 // 5 นาที
  })
  .addRule({
    name: 'error-rate-high',
    condition: async () => {
      const errors = await getErrorRateLastMinute();
      return errors > 10;
    },
    message: 'Error rate exceeds 10 errors/minute',
    severity: 'critical',
    cooldownMs: 10 * 60 * 1000 // 10 นาที
  });

alertManager.startMonitoring(60000); // ตรวจสอบทุก 1 นาที
```

---

## 9. Complete Monitoring Setup

```typescript
// monitoring/index.ts - รวมทุกอย่างไว้ด้วยกัน
class MonitoringSetup {
  private logger: ContextLogger;
  private metrics: typeof client;
  private healthService: HealthCheckService;
  private perfMonitor: PerformanceMonitor;
  private alertManager: AlertManager;

  constructor(private app: Application, private config: MonitoringConfig) {
    this.logger = new ContextLogger(
      winston.createLogger({ /* ... */ })
    );

    this.metrics = client;
    this.healthService = new HealthCheckService();
    this.perfMonitor = new PerformanceMonitor();
    this.alertManager = new AlertManager();
  }

  setup(): void {
    this.setupLogging();
    this.setupMetrics();
    this.setupHealthChecks();
    this.setupTracing();
    this.setupAlerting();
    this.registerEndpoints();
  }

  private setupLogging(): void {
    this.app.use(requestLoggingMiddleware);

    // Uncaught exception handler
    process.on('uncaughtException', (error) => {
      this.logger.fatal('Uncaught exception', error);
      process.exit(1);
    });

    process.on('unhandledRejection', (reason) => {
      this.logger.error('Unhandled promise rejection', reason as Error);
    });
  }

  private setupMetrics(): void {
    this.app.use(metricsMiddleware);
    client.collectDefaultMetrics({ register });
  }

  private setupHealthChecks(): void {
    this.healthService
      .register(new DatabaseHealthCheck(this.config.db))
      .register(new MemoryHealthCheck(512));
  }

  private setupTracing(): void {
    if (this.config.otlpEndpoint) {
      this.app.use(tracingMiddleware);
    }
  }

  private setupAlerting(): void {
    if (this.config.slackWebhook) {
      this.alertManager
        .addChannel(new SlackAlertChannel(this.config.slackWebhook));
    }
    this.alertManager.startMonitoring();
  }

  private registerEndpoints(): void {
    // Metrics
    this.app.get('/metrics', async (req, res) => {
      res.set('Content-Type', register.contentType);
      res.end(await register.metrics());
    });

    // Health
    this.app.get('/health', async (req, res) => {
      const health = await this.healthService.getHealth();
      const code = health.status === 'unhealthy' ? 503 : 200;
      res.status(code).json(health);
    });

    this.app.get('/health/liveness', (req, res) => {
      res.json({ status: 'alive' });
    });

    this.app.get('/health/readiness', async (req, res) => {
      const health = await this.healthService.getHealth();
      res.status(health.status === 'unhealthy' ? 503 : 200).json(health);
    });

    // Performance stats
    this.app.get('/metrics/performance', (req, res) => {
      res.json(this.perfMonitor.getAllStats());
    });
  }
}

interface MonitoringConfig {
  db: Database;
  otlpEndpoint?: string;
  slackWebhook?: string;
  sentryDsn?: string;
}

// ใช้งาน
const monitoring = new MonitoringSetup(app, {
  db: dbPool,
  otlpEndpoint: process.env.OTEL_ENDPOINT,
  slackWebhook: process.env.SLACK_WEBHOOK,
  sentryDsn: process.env.SENTRY_DSN
});

monitoring.setup();
```

---

## 10. Log Levels และ Best Practices

```typescript
// Log levels guideline
const logLevelGuide = {
  // FATAL: ระบบจะหยุดทำงาน
  fatal: [
    'Database connection completely lost',
    'Critical configuration missing',
    'Out of memory - cannot continue'
  ],

  // ERROR: มีข้อผิดพลาดที่ต้องแก้ไข
  error: [
    'Unhandled exceptions',
    'Payment processing failed',
    'External service unavailable after retries',
    'Data corruption detected'
  ],

  // WARN: มีบางอย่างผิดปกติแต่ระบบยังทำงานได้
  warn: [
    'High memory usage',
    'Slow database query (>1s)',
    'Cache miss rate high',
    'Deprecated API being used',
    'Authentication failures'
  ],

  // INFO: ข้อมูลทั่วไปเกี่ยวกับการทำงาน
  info: [
    'Service started/stopped',
    'User logged in/out',
    'Order placed',
    'Scheduled job started/completed',
    'Configuration loaded'
  ],

  // DEBUG: ข้อมูลสำหรับ debugging
  debug: [
    'Request/response details',
    'Database query parameters',
    'Cache operations',
    'Function entry/exit'
  ],

  // TRACE: ข้อมูลละเอียดมากสำหรับ deep debugging
  trace: [
    'Variable values',
    'Loop iterations',
    'Detailed execution flow'
  ]
};

// Log sensitive data อย่างปลอดภัย
function sanitizeLogData(data: Record<string, unknown>): Record<string, unknown> {
  const sensitiveFields = new Set([
    'password', 'token', 'secret', 'apiKey',
    'authorization', 'creditCard', 'ssn', 'cvv'
  ]);

  const result: Record<string, unknown> = {};

  for (const [key, value] of Object.entries(data)) {
    const lowerKey = key.toLowerCase();
    if (sensitiveFields.has(lowerKey)) {
      result[key] = '[REDACTED]';
    } else if (typeof value === 'object' && value !== null) {
      result[key] = sanitizeLogData(value as Record<string, unknown>);
    } else {
      result[key] = value;
    }
  }

  return result;
}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Monitoring และ Observability ที่สำคัญ:

1. **Winston Logging** - Structured logging, multiple transports, custom levels
2. **Pino Logging** - High-performance logging, pretty printing
3. **Prometheus Metrics** - Counters, Gauges, Histograms, custom metrics
4. **Health Checks** - Liveness, Readiness probes สำหรับ Kubernetes
5. **OpenTelemetry Tracing** - Distributed tracing, context propagation
6. **Sentry Error Tracking** - Error capture, user context, breadcrumbs
7. **Performance Monitoring** - Response times, percentiles, tracking
8. **Alerting** - Rule-based alerts, Slack notifications, cooldowns
9. **Complete Setup** - รวมทุกอย่างเข้าด้วยกันในระบบเดียว
10. **Log Level Guidelines** - Best practices สำหรับการใช้ log levels

การมี Observability ที่ดีช่วยให้ทีม:
- เข้าใจสถานะของระบบได้ตลอดเวลา
- ค้นหาสาเหตุของปัญหาได้รวดเร็ว
- แก้ไขปัญหาได้ก่อนที่ผู้ใช้จะสังเกตเห็น
- วางแผนการขยายระบบได้อย่างมีข้อมูล
