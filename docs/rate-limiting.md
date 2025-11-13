# Rate Limiting and Retry Strategies

This document covers Notion API rate limits, implementation strategies, and best practices for handling 429 errors.

## Official Limits

**Notion API Rate Limit:**
- **Average:** 3 requests per second per integration
- **Burst Window:** 15 minutes (2,700 requests total)
- **HTTP Error:** 429 with `rate_limited` error code
- **Response Header:** `retry-after` (seconds to wait)

**Key Points:**
- Limit is per integration token, not per user
- Short bursts allowed (can exceed 3/sec briefly)
- Sustained load must stay under 3/sec average
- Exceeding limit returns 429, not request drop

## Rate Limit Strategy

### 1. Proactive Throttling

**Don't wait for 429s** - throttle requests proactively to stay under limit.

**Implementation:**
```javascript
class RateLimiter {
  constructor(requestsPerSecond = 3) {
    this.interval = 1000 / requestsPerSecond;  // ~333ms between requests
    this.lastRequest = 0;
  }

  async throttle() {
    const now = Date.now();
    const timeSinceLastRequest = now - this.lastRequest;
    const waitTime = Math.max(0, this.interval - timeSinceLastRequest);

    if (waitTime > 0) {
      await new Promise(resolve => setTimeout(resolve, waitTime));
    }

    this.lastRequest = Date.now();
  }
}

// Usage
const limiter = new RateLimiter(3);

async function makeRequest(url, options) {
  await limiter.throttle();
  return fetch(url, options);
}
```

**Conservative Approach (Recommended for Edit Operations):**
```javascript
// Use 2 req/sec for safety margin on writes
const limiter = new RateLimiter(2);  // 500ms between requests
```

### 2. Exponential Backoff with Jitter

When 429 errors occur, retry with exponential backoff and jitter to avoid thundering herd.

**Implementation:**
```javascript
async function retryWithBackoff(fn, maxRetries = 4) {
  let attempt = 0;

  while (attempt < maxRetries) {
    try {
      return await fn();
    } catch (error) {
      // Check if it's a rate limit error
      if (error.status !== 429 || attempt >= maxRetries - 1) {
        throw error;
      }

      // Exponential backoff: 2s, 4s, 8s, 16s
      const baseDelay = Math.pow(2, attempt + 1) * 1000;

      // Add jitter: ±25% randomness
      const jitter = baseDelay * 0.25 * (Math.random() * 2 - 1);
      const delay = baseDelay + jitter;

      console.warn(`Rate limited (429). Retrying in ${delay}ms... (attempt ${attempt + 1}/${maxRetries})`);

      await new Promise(resolve => setTimeout(resolve, delay));
      attempt++;
    }
  }

  throw new Error("Max retries exceeded");
}

// Usage
const result = await retryWithBackoff(async () => {
  return await notionClient.pages.retrieve({ page_id: "abc123" });
});
```

**Use `retry-after` Header (Preferred):**
```javascript
async function retryWithBackoff(fn, maxRetries = 4) {
  let attempt = 0;

  while (attempt < maxRetries) {
    try {
      return await fn();
    } catch (error) {
      if (error.status !== 429 || attempt >= maxRetries - 1) {
        throw error;
      }

      // Use retry-after header if available, otherwise exponential backoff
      const retryAfter = error.headers?.get('retry-after');
      const delay = retryAfter
        ? parseInt(retryAfter) * 1000
        : Math.pow(2, attempt + 1) * 1000;

      const jitter = delay * 0.25 * (Math.random() * 2 - 1);
      const finalDelay = delay + jitter;

      console.warn(`Rate limited. Retrying in ${finalDelay}ms...`);

      await new Promise(resolve => setTimeout(resolve, finalDelay));
      attempt++;
    }
  }

  throw new Error("Max retries exceeded");
}
```

### 3. Request Queue with Concurrency Limit

For bulk operations, use a queue to control concurrency and prevent rate limit bursts.

**Implementation with p-queue:**
```javascript
import PQueue from 'p-queue';

// Max 3 concurrent requests, with 333ms minimum time between starts
const queue = new PQueue({
  concurrency: 3,
  interval: 1000,
  intervalCap: 3
});

// Add requests to queue
const results = await Promise.all([
  queue.add(() => notionClient.pages.retrieve({ page_id: "id1" })),
  queue.add(() => notionClient.pages.retrieve({ page_id: "id2" })),
  queue.add(() => notionClient.pages.retrieve({ page_id: "id3" })),
  // ... many more
]);
```

**Manual Queue Implementation:**
```javascript
class RequestQueue {
  constructor(requestsPerSecond = 3) {
    this.queue = [];
    this.processing = false;
    this.interval = 1000 / requestsPerSecond;
    this.lastRequest = 0;
  }

  async add(fn) {
    return new Promise((resolve, reject) => {
      this.queue.push({ fn, resolve, reject });
      this.process();
    });
  }

  async process() {
    if (this.processing || this.queue.length === 0) return;

    this.processing = true;

    while (this.queue.length > 0) {
      const { fn, resolve, reject } = this.queue.shift();

      // Throttle
      const now = Date.now();
      const timeSinceLastRequest = now - this.lastRequest;
      const waitTime = Math.max(0, this.interval - timeSinceLastRequest);

      if (waitTime > 0) {
        await new Promise(r => setTimeout(r, waitTime));
      }

      // Execute with retry
      try {
        const result = await retryWithBackoff(fn);
        resolve(result);
      } catch (error) {
        reject(error);
      }

      this.lastRequest = Date.now();
    }

    this.processing = false;
  }
}

// Usage
const queue = new RequestQueue(2);  // 2 req/sec for safety

const results = await Promise.all([
  queue.add(() => notionClient.pages.retrieve({ page_id: "id1" })),
  queue.add(() => notionClient.pages.retrieve({ page_id: "id2" })),
  // ... many more
]);
```

### 4. Batch Operations

Group operations to minimize API calls where possible.

**Append Multiple Blocks:**
```javascript
// Bad: 10 API calls
for (const block of blocks) {
  await notionClient.blocks.append({
    block_id: parentId,
    children: [block]
  });
}

// Good: 1 API call (up to 100 blocks)
await notionClient.blocks.append({
  block_id: parentId,
  children: blocks  // Max 100
});
```

**Parallel Fetches (with throttling):**
```javascript
// Bad: Sequential (slow)
const pages = [];
for (const id of pageIds) {
  const page = await notionClient.pages.retrieve({ page_id: id });
  pages.push(page);
}

// Good: Parallel with queue
const queue = new RequestQueue(2);
const pages = await Promise.all(
  pageIds.map(id =>
    queue.add(() => notionClient.pages.retrieve({ page_id: id }))
  )
);
```

## Complete Integration

Combine all strategies into a production-ready client wrapper:

```javascript
import fetch from 'node-fetch';

class NotionClientWithRateLimiting {
  constructor(token, requestsPerSecond = 2) {
    this.token = token;
    this.baseUrl = 'https://api.notion.com/v1';
    this.headers = {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
      'Notion-Version': '2022-06-28'
    };

    // Rate limiting
    this.interval = 1000 / requestsPerSecond;
    this.lastRequest = 0;
    this.queue = [];
    this.processing = false;
  }

  async request(endpoint, options = {}, maxRetries = 4) {
    return this.enqueue(async () => {
      return this.executeWithRetry(endpoint, options, maxRetries);
    });
  }

  async enqueue(fn) {
    return new Promise((resolve, reject) => {
      this.queue.push({ fn, resolve, reject });
      this.processQueue();
    });
  }

  async processQueue() {
    if (this.processing || this.queue.length === 0) return;
    this.processing = true;

    while (this.queue.length > 0) {
      const { fn, resolve, reject } = this.queue.shift();

      try {
        const result = await fn();
        resolve(result);
      } catch (error) {
        reject(error);
      }
    }

    this.processing = false;
  }

  async executeWithRetry(endpoint, options, maxRetries) {
    let attempt = 0;

    while (attempt < maxRetries) {
      // Throttle
      await this.throttle();

      try {
        const response = await fetch(`${this.baseUrl}${endpoint}`, {
          ...options,
          headers: { ...this.headers, ...options.headers }
        });

        // Check for rate limit
        if (response.status === 429) {
          const retryAfter = response.headers.get('retry-after');
          const delay = retryAfter
            ? parseInt(retryAfter) * 1000
            : Math.pow(2, attempt + 1) * 1000;

          const jitter = delay * 0.25 * (Math.random() * 2 - 1);
          const finalDelay = delay + jitter;

          if (attempt < maxRetries - 1) {
            console.warn(`Rate limited (429). Retrying in ${finalDelay}ms... (attempt ${attempt + 1}/${maxRetries})`);
            await new Promise(resolve => setTimeout(resolve, finalDelay));
            attempt++;
            continue;
          }
        }

        // Success or non-retryable error
        const data = await response.json();

        if (!response.ok) {
          throw new Error(`Notion API error: ${data.message || response.statusText}`);
        }

        return data;

      } catch (error) {
        // Network errors - retry with backoff
        if (attempt < maxRetries - 1 && !error.message.includes('Notion API error')) {
          const delay = Math.pow(2, attempt + 1) * 1000;
          console.warn(`Network error. Retrying in ${delay}ms...`);
          await new Promise(resolve => setTimeout(resolve, delay));
          attempt++;
          continue;
        }

        throw error;
      }
    }

    throw new Error('Max retries exceeded');
  }

  async throttle() {
    const now = Date.now();
    const timeSinceLastRequest = now - this.lastRequest;
    const waitTime = Math.max(0, this.interval - timeSinceLastRequest);

    if (waitTime > 0) {
      await new Promise(resolve => setTimeout(resolve, waitTime));
    }

    this.lastRequest = Date.now();
  }

  // API methods
  async retrievePage(pageId) {
    return this.request(`/pages/${pageId}`, { method: 'GET' });
  }

  async queryDatabase(databaseId, filter = {}, sorts = [], startCursor, pageSize = 100) {
    return this.request(`/databases/${databaseId}/query`, {
      method: 'POST',
      body: JSON.stringify({ filter, sorts, start_cursor: startCursor, page_size: pageSize })
    });
  }

  // ... other methods
}

// Usage
const client = new NotionClientWithRateLimiting(process.env.NOTION_API_TOKEN, 2);

// Automatically throttled and retried
const page = await client.retrievePage('abc123');
```

## Best Practices Summary

1. **Throttle Proactively**
   - Use 2-2.5 req/sec for safety margin
   - Especially important for write operations
   - Prevents hitting rate limits in first place

2. **Implement Exponential Backoff**
   - Handle 429s gracefully with retry logic
   - Use `retry-after` header when available
   - Add jitter to prevent thundering herd
   - Max 4 retries: 2s, 4s, 8s, 16s

3. **Use Request Queue**
   - Control concurrency for bulk operations
   - Prevents bursts that trigger rate limits
   - Better resource utilization

4. **Batch Where Possible**
   - Append multiple blocks in one call (max 100)
   - Parallel fetches with throttling
   - Reduce total API call count

5. **Monitor and Log**
   - Track 429 frequency
   - Log retry attempts
   - Alert on excessive rate limiting

6. **Cache Aggressively**
   - Block IDs are stable - cache by ID
   - Invalidate on updates
   - Reduces API load significantly

7. **Optimize Nested Fetches**
   - Fetch only necessary depth
   - Use `has_children` to skip unnecessary calls
   - Consider lazy loading for deep structures

## Monitoring Example

```javascript
class RateLimitMonitor {
  constructor() {
    this.stats = {
      total: 0,
      rateLimited: 0,
      retries: 0,
      errors: 0
    };
  }

  recordRequest() {
    this.stats.total++;
  }

  recordRateLimit() {
    this.stats.rateLimited++;
  }

  recordRetry() {
    this.stats.retries++;
  }

  recordError() {
    this.stats.errors++;
  }

  report() {
    console.log('Rate Limit Stats:', {
      ...this.stats,
      rateLimitRate: (this.stats.rateLimited / this.stats.total * 100).toFixed(2) + '%',
      avgRetriesPerRateLimit: (this.stats.retries / Math.max(this.stats.rateLimited, 1)).toFixed(2)
    });
  }
}

const monitor = new RateLimitMonitor();

// In your client
async executeWithRetry(endpoint, options, maxRetries) {
  monitor.recordRequest();

  // ... retry logic with monitor.recordRateLimit() and monitor.recordRetry()
}

// Periodic reporting
setInterval(() => monitor.report(), 60000);  // Every minute
```

## Resources

- [Notion API Rate Limits](https://developers.notion.com/reference/rate-limits)
- [HTTP 429 Status Code](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429)
- [Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
