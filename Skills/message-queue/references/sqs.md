# AWS SQS Reference

## Dependencies

```
@aws-sdk/client-sqs
@aws-sdk/lib-dynamodb   # optional, for deduplication table
```

## Key Concepts

| Concept            | Description                                                         |
|--------------------|---------------------------------------------------------------------|
| Standard Queue     | At-least-once delivery, best-effort ordering                        |
| FIFO Queue         | Exactly-once delivery, strict ordering (suffix `.fifo` required)    |
| Visibility Timeout | Time a message is hidden after being received — prevents duplicates |
| DLQ                | Native — configure `RedrivePolicy` on the source queue              |
| Long Polling       | `WaitTimeSeconds: 20` — reduces empty receives and cost             |
| Message Group ID   | FIFO only — ensures ordering within a group                         |

---

## SQS Client

```ts
// src/lib/sqs.ts
import { SQSClient } from '@aws-sdk/client-sqs';

export const sqsClient = new SQSClient({
  region: process.env.AWS_REGION!,
  // Uses IAM role in EC2/ECS — no hardcoded credentials
});
```

---

## Types

```ts
// src/queues/@types/index.d.ts

export enum SQSQueueUrl {
  EMAIL = 'https://sqs.{region}.amazonaws.com/{account}/email-queue',
  EMAIL_DLQ = 'https://sqs.{region}.amazonaws.com/{account}/email-queue-dlq',
}

export interface SQSMessageBody<T = unknown> {
  type: string;
  payload: T;
  traceId: string;
}

export interface EmailPayload {
  to: string;
  type: 'welcome' | 'reset-password' | 'verification';
  data: Record<string, unknown>;
}
```

---

## Producer

```ts
// src/queues/email/producer.ts
import { SendMessageCommand, SendMessageBatchCommand } from '@aws-sdk/client-sqs';
import { sqsClient } from '@/lib/sqs';
import { SQSQueueUrl, SQSMessageBody, EmailPayload } from '@/queues/@types';
import { randomUUID } from 'crypto';

export const enqueueEmail = async (payload: EmailPayload): Promise<void> => {
  const body: SQSMessageBody<EmailPayload> = {
    type: payload.type,
    payload,
    traceId: randomUUID(),
  };

  await sqsClient.send(
    new SendMessageCommand({
      QueueUrl: SQSQueueUrl.EMAIL,
      MessageBody: JSON.stringify(body),
      // For FIFO queues, add:
      // MessageGroupId: payload.to,
      // MessageDeduplicationId: randomUUID(),
    })
  );
};

export const enqueueEmailBatch = async (payloads: EmailPayload[]): Promise<void> => {
  await sqsClient.send(
    new SendMessageBatchCommand({
      QueueUrl: SQSQueueUrl.EMAIL,
      Entries: payloads.map((p, i) => ({
        Id: String(i),
        MessageBody: JSON.stringify({ type: p.type, payload: p, traceId: randomUUID() }),
      })),
    })
  );
};
```

---

## Consumer (Polling Loop)

```ts
// src/queues/email/worker.ts
import {
  ReceiveMessageCommand,
  DeleteMessageCommand,
  ChangeMessageVisibilityCommand,
} from '@aws-sdk/client-sqs';
import { sqsClient } from '@/lib/sqs';
import { SQSQueueUrl, SQSMessageBody, EmailPayload } from '@/queues/@types';
import { welcomeHandler } from './handlers/welcome.handler';
import { resetPasswordHandler } from './handlers/reset-password.handler';
import { logger } from '@/lib/logger';

const handlers = {
  welcome: welcomeHandler,
  'reset-password': resetPasswordHandler,
} as const;

let running = true;

export const startEmailConsumer = async (): Promise<void> => {
  logger.info('SQS email consumer started');

  while (running) {
    const response = await sqsClient.send(
      new ReceiveMessageCommand({
        QueueUrl: SQSQueueUrl.EMAIL,
        MaxNumberOfMessages: 10,
        WaitTimeSeconds: 20, // long polling
        VisibilityTimeout: 60,
      })
    );

    const messages = response.Messages ?? [];

    await Promise.allSettled(
      messages.map(async (msg) => {
        if (!msg.Body || !msg.ReceiptHandle) return;

        try {
          const parsed = JSON.parse(msg.Body) as SQSMessageBody<EmailPayload>;
          const handler = handlers[parsed.type as keyof typeof handlers];

          if (!handler) {
            logger.warn({ type: parsed.type }, 'Unknown job type — skipping');
            return;
          }

          await handler(parsed.payload);

          await sqsClient.send(
            new DeleteMessageCommand({
              QueueUrl: SQSQueueUrl.EMAIL,
              ReceiptHandle: msg.ReceiptHandle,
            })
          );
        } catch (err) {
          logger.error({ msgId: msg.MessageId, err }, 'Message processing failed');
          // Let visibility timeout expire — SQS will re-deliver
          // Or extend visibility for longer retry window:
          await sqsClient.send(
            new ChangeMessageVisibilityCommand({
              QueueUrl: SQSQueueUrl.EMAIL,
              ReceiptHandle: msg.ReceiptHandle!,
              VisibilityTimeout: 0, // make immediately available for retry
            })
          );
        }
      })
    );
  }
};

export const stopEmailConsumer = () => { running = false; };
```

---

## Handler

```ts
// src/queues/email/handlers/welcome.handler.ts
import { EmailPayload } from '@/queues/@types';

export const welcomeHandler = async (payload: EmailPayload): Promise<void> => {
  // send welcome email
};
```

---

## Native DLQ Setup (CDK / Terraform / Console)

Configure via `RedrivePolicy` on the source queue:

```json
{
  "deadLetterTargetArn": "arn:aws:sqs:{region}:{account}:email-queue-dlq",
  "maxReceiveCount": 3
}
```

No code change needed — SQS moves to DLQ automatically after `maxReceiveCount` failures.

---

## DLQ Consumer (Alerting)

```ts
// src/queues/dlq/worker.ts
// Polls the DLQ, logs/alerts, does NOT delete unless manually handled

export const startDLQConsumer = async (): Promise<void> => {
  // Same polling loop but:
  // - Log with high severity
  // - Send alert (PagerDuty, SNS, Slack webhook)
  // - Do NOT delete unless manually resolved
};
```

---

## Boot Initialization

```ts
// src/queues/index.ts
import { startEmailConsumer } from './email/worker';
import { logger } from '@/lib/logger';

export const initQueues = (): void => {
  startEmailConsumer().catch((err) => logger.error({ err }, 'Email consumer crashed'));
};
```

---

## Tradeoffs

| Feature           | Notes                                                              |
|-------------------|--------------------------------------------------------------------|
| Native DLQ        | Built-in — configure `RedrivePolicy`, no extra code               |
| Exactly-once      | FIFO queues only — standard queues are at-least-once              |
| Ordering          | FIFO only — standard queues have best-effort ordering             |
| Cron/scheduling   | Not native — use EventBridge Scheduler to trigger SQS             |
| Visibility timeout| Must tune carefully — too low = duplicate processing              |
| Cost              | Pay per request — long polling reduces empty receives             |
| Scaling           | Lambda integration scales automatically; EC2 needs manual workers |
| Max message size  | 256 KB — use S3 + pointer pattern for large payloads              |
