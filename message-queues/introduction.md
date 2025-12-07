1. When to use Postgres as a Queue.

Magda suggests using postgres instead of a dedicated queue (like sqs or rabitmq) if:
- Transactional integrity is needed: you want the "job" to be created only if the business transaction (e.g., registering a user) succeeds. In postgres, this happens atomically.
- Volume is manageable: postgres is a single server (mostly). if you'ren't doing millions of events per second, it handles queues easily.
- Simplicity: You don't want to maintaine a separate piece of infrastructue.

2. The DIY Approach: Building a custom queue.
The text walks through building a queue from scratch using standard sql.

The schema.
---
You need a table to hold the jobs.

```
CREATE SCHEMA mq;

CREATE TYPE mq.status AS ENUM ('new', 'processing', 'completed');

CREATE TABLE mq.queue (
    id BIGSERIAL PRIMARY KEY,
    message JSON NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    status mq.status NOT NULL DEFAULT 'new'
);
```
