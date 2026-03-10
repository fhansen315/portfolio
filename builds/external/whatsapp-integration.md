# WhatsApp Integration Platform

> Enterprise messaging intelligence with automated classification and analytics

## Overview

Built a TypeScript-based WhatsApp integration platform that manages 97 business groups with automated classification, real-time analytics, and intelligent message processing.

## The Problem

Enterprise WhatsApp usage was chaotic and unmanaged:
- 97 active groups with no organization
- Important messages buried in noise
- No analytics on engagement or response times
- Manual monitoring required 24/7 attention
- No integration with sales pipeline

## Solution Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                 WHATSAPP INTEGRATION PLATFORM                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────┐     ┌───────────────┐     ┌───────────────┐      │
│  │ WhatsApp  │     │   Webhook     │     │   Message     │      │
│  │   API     │────▶│   Server      │────▶│  Processor    │      │
│  └───────────┘     └───────────────┘     └───────────────┘      │
│                            │                     │               │
│                            ▼                     ▼               │
│                    ┌───────────────┐     ┌───────────────┐      │
│                    │   Storage     │     │  Analytics    │      │
│                    │   Layer       │     │   Engine      │      │
│                    └───────────────┘     └───────────────┘      │
│                            │                     │               │
│            ┌───────────────┼───────────────┐     │               │
│            ▼               ▼               ▼     ▼               │
│     ┌──────────┐    ┌──────────┐    ┌──────────────────┐       │
│     │PostgreSQL│    │  Redis   │    │    Reports &     │       │
│     │ Messages │    │  Cache   │    │   Dashboards     │       │
│     └──────────┘    └──────────┘    └──────────────────┘       │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

## Technical Implementation

### Webhook Server (Express + TypeScript)

```typescript
// Server setup with middleware and error handling
import express, { Application, Request, Response, NextFunction } from 'express';
import { webhookRouter } from './routes/webhook';
import { errorHandler } from './middleware/errorHandler';
import { logger } from './utils/logger';

export function createServer(): Application {
  const app = express();

  // Middleware
  app.use(express.json({ limit: '10mb' }));
  app.use(requestLogger);

  // Routes
  app.use('/webhook', webhookRouter);
  app.use('/health', healthCheck);

  // Error handling
  app.use(errorHandler);

  return app;
}

// Webhook route handler
router.post('/', async (req: Request, res: Response) => {
  const event = req.body as WebhookEvent;

  // Respond immediately to avoid timeout
  res.status(200).json({ received: true });

  // Process asynchronously
  setImmediate(async () => {
    try {
      await processWebhookEvent(event);
    } catch (error) {
      logger.error('Webhook processing failed', { error, event });
    }
  });
});
```

### Type-Safe Event Handling

```typescript
// Comprehensive type definitions for WhatsApp events
interface WebhookEvent {
  event: EventType;
  instance: string;
  data: MessageData | StatusData | ConnectionData;
  sender: string;
  destination?: string;
}

type EventType =
  | 'messages.upsert'
  | 'messages.update'
  | 'messages.delete'
  | 'presence.update'
  | 'connection.update';

interface MessageData {
  key: MessageKey;
  pushName: string;
  message: MessageContent;
  messageTimestamp: number;
  status: MessageStatus;
}

interface MessageContent {
  conversation?: string;
  imageMessage?: MediaMessage;
  videoMessage?: MediaMessage;
  audioMessage?: AudioMessage;
  documentMessage?: DocumentMessage;
  extendedTextMessage?: ExtendedText;
}

// Parsed message for simplified handling
interface ParsedMessage {
  id: string;
  from: string;
  fromName: string;
  groupId?: string;
  groupName?: string;
  content: string;
  type: 'text' | 'image' | 'video' | 'audio' | 'document';
  timestamp: Date;
  mentions: string[];
}
```

### Storage Factory Pattern

```typescript
// Multi-adapter storage with write-to-all, read-from-primary
interface StorageAdapter {
  save(message: ParsedMessage): Promise<void>;
  query(filters: QueryFilters): Promise<ParsedMessage[]>;
  getStats(groupId?: string): Promise<GroupStats>;
}

class StorageFactory {
  private adapters: Map<string, StorageAdapter> = new Map();
  private primaryAdapter: StorageAdapter;

  constructor(config: StorageConfig) {
    // Initialize adapters based on config
    if (config.postgres) {
      this.adapters.set('postgres', new PostgresAdapter(config.postgres));
    }
    if (config.redis) {
      this.adapters.set('redis', new RedisAdapter(config.redis));
    }
    if (config.file) {
      this.adapters.set('file', new FileAdapter(config.file));
    }

    this.primaryAdapter = this.adapters.get(config.primary)!;
  }

  async save(message: ParsedMessage): Promise<void> {
    // Write to all adapters concurrently
    const savePromises = Array.from(this.adapters.values()).map(
      adapter => adapter.save(message).catch(err => {
        logger.warn('Adapter save failed', { adapter: adapter.name, err });
      })
    );

    await Promise.all(savePromises);
  }

  async query(filters: QueryFilters): Promise<ParsedMessage[]> {
    // Read from primary only
    return this.primaryAdapter.query(filters);
  }
}
```

### Group Classification System

```typescript
// Automated group categorization
interface GroupClassification {
  id: string;
  name: string;
  category: GroupCategory;
  labels: string[];
  priority: 'high' | 'medium' | 'low';
  lastActivity: Date;
}

type GroupCategory =
  | 'partner-opportunity'   // 60 groups
  | 'partner-general'       // 20 groups
  | 'internal'              // 8 groups
  | 'event'                 // 2 groups
  | 'automation';           // 1 group

class GroupClassifier {
  private rules: ClassificationRule[] = [
    {
      pattern: /opportunity|deal|prospect/i,
      category: 'partner-opportunity',
      priority: 'high'
    },
    {
      pattern: /internal|team|staff/i,
      category: 'internal',
      priority: 'medium'
    },
    // ... more rules
  ];

  classify(group: RawGroup): GroupClassification {
    for (const rule of this.rules) {
      if (rule.pattern.test(group.name)) {
        return {
          id: group.id,
          name: group.name,
          category: rule.category,
          priority: rule.priority,
          labels: this.extractLabels(group),
          lastActivity: new Date(group.lastMessage)
        };
      }
    }

    return this.defaultClassification(group);
  }
}
```

### Analytics Engine

```typescript
// Real-time analytics with caching
interface GroupAnalytics {
  messageCount: number;
  activeMembers: number;
  avgResponseTime: number;
  peakHours: number[];
  sentiment: 'positive' | 'neutral' | 'negative';
  trends: TrendData[];
}

class AnalyticsEngine {
  private cache: RedisClient;
  private db: PostgresPool;

  async getGroupAnalytics(groupId: string): Promise<GroupAnalytics> {
    // Check cache first
    const cached = await this.cache.get(`analytics:${groupId}`);
    if (cached) return JSON.parse(cached);

    // Compute from database
    const analytics = await this.computeAnalytics(groupId);

    // Cache for 5 minutes
    await this.cache.setex(
      `analytics:${groupId}`,
      300,
      JSON.stringify(analytics)
    );

    return analytics;
  }

  private async computeAnalytics(groupId: string): Promise<GroupAnalytics> {
    const [messages, members, responseTimes] = await Promise.all([
      this.db.query('SELECT COUNT(*) FROM messages WHERE group_id = $1', [groupId]),
      this.db.query('SELECT DISTINCT sender FROM messages WHERE group_id = $1', [groupId]),
      this.db.query(`
        SELECT AVG(response_time) as avg_time
        FROM message_responses
        WHERE group_id = $1
      `, [groupId])
    ]);

    return {
      messageCount: messages.rows[0].count,
      activeMembers: members.rows.length,
      avgResponseTime: responseTimes.rows[0].avg_time || 0,
      peakHours: await this.computePeakHours(groupId),
      sentiment: await this.computeSentiment(groupId),
      trends: await this.computeTrends(groupId)
    };
  }
}
```

## Technologies Used

| Category | Technologies |
|----------|--------------|
| Runtime | Node.js 18+, TypeScript 5.0 |
| Framework | Express.js |
| API | Evolution API v2.3.7 (WhatsApp) |
| Database | PostgreSQL, Redis |
| Container | Docker, Docker Compose |
| Logging | Pino |

## Results & Impact

### Scale Achieved

| Metric | Value |
|--------|-------|
| Groups managed | 97 |
| Messages processed/day | 500+ |
| Labels synced with CRM | 13 |
| Uptime | 99.9% |

### Group Distribution

```
Category Distribution:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

partner-opportunity  ████████████████████  60 (62%)
partner-general      ███████              20 (21%)
internal             ███                   8 (8%)
event                █                     2 (2%)
automation           █                     1 (1%)

Total: 97 groups
```

### Business Value

- **24/7 monitoring** without manual attention
- **Instant alerts** for high-priority messages
- **CRM integration** via label synchronization
- **Audit trail** for compliance and reporting
- **Analytics** for engagement optimization

## Key Learnings

1. **Respond first, process later**: Webhook endpoints should return immediately; async processing prevents timeouts

2. **Type everything**: TypeScript types caught numerous bugs during development

3. **Multi-adapter storage**: Writing to multiple stores provides redundancy and flexibility

4. **Classification rules**: Simple pattern matching handles 90% of cases; AI for edge cases

5. **Cache aggressively**: Analytics queries are expensive; 5-minute cache dramatically improves performance

## Skills Demonstrated

- **TypeScript**: Strong typing, interfaces, generics
- **Event-Driven Architecture**: Webhooks, async processing
- **Database Design**: PostgreSQL schemas, Redis caching
- **API Integration**: WhatsApp Business API
- **DevOps**: Docker containerization
- **System Design**: Scalable message processing

---

*Part of my automation portfolio demonstrating TypeScript backend development and event-driven architecture.*
