# Agenda Reference

## Dependencies

```
@hokify/agenda      # actively maintained fork of agenda
mongoose            # for MongoDB backend
# OR
pg                  # for PostgreSQL backend (use agenda-pg or raw pg polling)
```

> Note: Agenda's official package is largely unmaintained. Use `@hokify/agenda` for MongoDB.
> For PostgreSQL, consider `pg-boss` instead — it's production-grade and actively maintained.
> See PostgreSQL section below.

---

## Key Concepts

| Concept        | Description                                                     |
|----------------|-----------------------------------------------------------------|
| `define`       | Register a job type with its handler function                   |
| `schedule`     | Run once at a specific time                                     |
| `every`        | Repeatable — human-readable interval or cron expression        |
| `now`          | Enqueue immediately                                             |
| `cancel`       | Remove scheduled jobs matching a query                          |
| `concurrency`  | Global + per-job concurrency control                            |
| Locking        | MongoDB-backed distributed locking — safe for multi-instance   |

---

## MongoDB Setup

```ts
// src/lib/agenda.ts
import Agenda from '@hokify/agenda';

export const agenda = new Agenda({
  db: {
    address: process.env.MONGODB_URI!,
    collection: 'agendaJobs',
    options: { useUnifiedTopology: true },
  },
  processEvery: '10 seconds',
  maxConcurrency: 20,
  defaultConcurrency: 5,
  defaultLockLifetime: 10_000, // ms — extend for long-running jobs
});
```

---

## PostgreSQL — Use pg-boss Instead

Agenda does not have a mature PostgreSQL adapter. Use `pg-boss`:

```
pg-boss
```

```ts
// src/lib/pgBoss.ts
import PgBoss from 'pg-boss';

export const boss = new PgBoss(process.env.DATABASE_URL!);

await boss.start();
```

The rest of this file documents Agenda (MongoDB). For pg-boss patterns, the API differs — see pg-boss docs.

---

## Types

```ts
// src/queues/@types/index.d.ts

export type AgendaJobName =
  | 'send-email'
  | 'rebuild-sitemap'
  | 'sync-property-data';

export interface SendEmailJobData {
  to: string;
  type: 'welcome' | 'reset-password';
  payload: Record<string, unknown>;
}

export interface RebuildSitemapJobData {
  trigger: 'cron' | 'manual';
}
```

---

## Define Jobs (Registration)

Define all jobs before `agenda.start()`. One file per job type.

```ts
// src/queues/email/handlers/send-email.handler.ts
import { Agenda } from '@hokify/agenda';
import { SendEmailJobData } from '@/queues/@types';
import { logger } from '@/lib/logger';

export const defineSendEmailJob = (agenda: Agenda): void => {
  agenda.define<SendEmailJobData>(
    'send-email',
    { concurrency: 5, priority: 10 },
    async (job) => {
      const { to, type, payload } = job.attrs.data;
      // send email logic
    }
  );
};
```

---

## Producer (Enqueue)

```ts
// src/queues/email/producer.ts
import { agenda } from '@/lib/agenda';
import { SendEmailJobData } from '@/queues/@types';

export const enqueueEmail = (data: SendEmailJobData): Promise<unknown> =>
  agenda.now('send-email', data);

export const scheduleEmail = (when: Date | string, data: SendEmailJobData): Promise<unknown> =>
  agenda.schedule(when, 'send-email', data);
```

---

## Scheduler (Cron / Repeatable)

```ts
// src/queues/scheduler.ts
import { agenda } from '@/lib/agenda';
import { RebuildSitemapJobData } from '@/queues/@types';

export const registerScheduledJobs = async (): Promise<void> => {
  // Cancel existing to avoid duplicates on restart
  await agenda.cancel({ name: 'rebuild-sitemap' });

  await agenda.every(
    '0 2 * * *', // 2am daily
    'rebuild-sitemap',
    { trigger: 'cron' } satisfies RebuildSitemapJobData,
    { timezone: 'Africa/Cairo', skipImmediate: true }
  );
};
```

---

## Error Handling & Retry

Agenda retries via `failCount` — implement manually in the handler or via event listeners.

```ts
// src/lib/agenda.ts (extend setup)
import { agenda } from './agenda';
import { logger } from '@/lib/logger';

agenda.on('fail', (err, job) => {
  logger.error({ jobName: job.attrs.name, jobId: job.attrs._id, err: err.message }, 'Agenda job failed');

  // Manual retry with backoff
  if ((job.attrs.failCount ?? 0) < 3) {
    const delay = Math.pow(2, job.attrs.failCount ?? 0) * 2000;
    void job.schedule(new Date(Date.now() + delay)).save();
  } else {
    // Move to dead letter — log or insert into a deadJobs collection
    logger.error({ jobName: job.attrs.name }, 'Job exceeded max retries — dead letter');
  }
});

agenda.on('success', (job) => {
  logger.info({ jobName: job.attrs.name }, 'Agenda job completed');
});
```

---

## Dead Letter (MongoDB)

Agenda doesn't have a native DLQ. Write failed jobs to a separate collection:

```ts
import mongoose from 'mongoose';

const deadJobSchema = new mongoose.Schema({
  jobName: String,
  data: mongoose.Schema.Types.Mixed,
  failedReason: String,
  failCount: Number,
  failedAt: { type: Date, default: Date.now },
});

export const DeadJob = mongoose.model('DeadJob', deadJobSchema);
```

In the `fail` event handler after max retries:
```ts
await DeadJob.create({
  jobName: job.attrs.name,
  data: job.attrs.data,
  failedReason: err.message,
  failCount: job.attrs.failCount,
});
```

---

## Boot Initialization

```ts
// src/queues/index.ts
import { agenda } from '@/lib/agenda';
import { defineSendEmailJob } from './email/handlers/send-email.handler';
import { defineRebuildSitemapJob } from './sitemap/handlers/rebuild-sitemap.handler';
import { registerScheduledJobs } from './scheduler';
import { logger } from '@/lib/logger';

const jobDefiners = [defineSendEmailJob, defineRebuildSitemapJob];

export const initQueues = async (): Promise<void> => {
  jobDefiners.forEach((define) => define(agenda));

  await agenda.start();
  await registerScheduledJobs();

  logger.info('Agenda queues initialized');
};

export const gracefulShutdown = async (): Promise<void> => {
  await agenda.stop();
};
```

Register in `src/server.ts`:
```ts
await initQueues();

process.on('SIGTERM', gracefulShutdown);
process.on('SIGINT', gracefulShutdown);
```

---

## pg-boss Quick Reference (PostgreSQL)

```ts
// Define
await boss.createQueue('send-email');
boss.work<SendEmailJobData>('send-email', { teamSize: 5 }, async (jobs) => {
  for (const job of jobs) {
    // process job.data
  }
});

// Enqueue
await boss.send('send-email', { to: '...', type: 'welcome', payload: {} });

// Schedule
await boss.schedule('rebuild-sitemap', '0 2 * * *', {});
```

DLQ: pg-boss has native dead letter support via `deadLetter` queue option.

---

## Tradeoffs

| Feature        | Agenda (MongoDB)                              | pg-boss (PostgreSQL)                     |
|----------------|-----------------------------------------------|------------------------------------------|
| Maintenance    | Use `@hokify/agenda` fork                     | Actively maintained                      |
| Cron           | Native, human-readable + cron syntax          | Native                                   |
| Retry          | Manual via `fail` event                       | Built-in with configurable attempts      |
| DLQ            | Manual — write to dead collection             | Native `deadLetter` queue                |
| Distributed    | MongoDB locking — safe multi-instance         | PostgreSQL row locking — safe            |
| General queue  | Yes — `now()` for immediate jobs              | Yes                                      |
| Priority       | Via `priority` option on `define`             | Via `priority` on `send`                 |
| Timezone       | Supported on `every()`                        | Supported                                |
