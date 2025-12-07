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

The Secret Sauce: `FOR UPDATE SKIP LOCKED`
---
This is the most important concept in the chapter. To prevent two workers (consumers) from grabbing the same job, you use `SKIP LOCKED`. 
- Standard Select: Locking a row waits for the lock to release.
- Skip Locked: If a row is locked by another worker, Postgres skips it and move to the next one. This allows parallel processing.

The Dequeu Function
---
Here is the cleaned-up logic for pulling a job off the queue safely:

```
CREATE OR REPLACE FUNCTION mq.dequeue(messages_cnt INT)
RETURNS TABLE (msg_id BIGINT, message JSON, enqueued_at TIMESTAMPTZ) AS $$
BEGIN
  RETURN QUERY
  WITH new_messages AS (
    SELECT id FROM mq.queue
    WHERE status = 'new' ORDER BY created_at
    -- This is the critical line that allows concurrency:
    FOR UPDATE SKIP LOCKED
    LIMIT messages_cnt
  )
  UPDATE mq.queue q
  SET status = 'processing'
  FROM new_messages WHERE q.id = new_messages.id
  RETURNING q.id, q.message,
    date_trunc('seconds',q.created_at) AS created_at;
END;
$$ LANGUAGE plpgsql;
```

3. Real-Time Notifications (LISTEN/NOTIFY)

To avoid "polling" (asking the database every 1 second "do you have work?"), the text suggests using Postgres's native event system.

    The Worker runs: LISTEN queue_new_message;

    The Enqueue Function is updated to fire a notification:


