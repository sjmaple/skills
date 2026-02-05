# PubNub Functions 2.0 Basics

## Overview

PubNub Functions 2.0 execute serverless JavaScript code at the network edge in response to events. Key features:

- **Event-driven**: Trigger on messages, HTTP requests, or schedules
- **Serverless**: No infrastructure to manage
- **Edge execution**: Low latency, close to users
- **Built-in modules**: KVStore, XHR, Vault, Crypto, and more

## Function Types

### 1. Before/After Publish (Message Events)

Trigger when messages are published to matching channels.

```javascript
// Before Publish - Can modify or block messages
export default async (request) => {
  try {
    console.log('Channel:', request.channels[0]);
    console.log('Message:', request.message);

    // Modify message
    request.message.processedAt = new Date().toISOString();

    // Allow message to proceed (modified)
    return request.ok();

    // Or block the message
    // return request.abort();
  } catch (error) {
    console.error('Error:', error);
    return request.abort();
  }
};
```

**Key Methods:**
- `request.ok()` - Allow message to proceed
- `request.abort()` - Block message from delivery
- `request.message` - Access/modify message payload
- `request.channels` - Array of channel names

### 2. On Request (HTTP Endpoint)

Trigger when HTTP requests hit the function's URL.

```javascript
export default async (request, response) => {
  try {
    // Access request data
    const method = request.method;      // GET, POST, PUT, DELETE
    const params = request.params;      // URL path parameters
    const query = request.query;        // Query string parameters
    const body = await request.json();  // Parse JSON body

    // Set CORS headers
    response.headers['Access-Control-Allow-Origin'] = '*';
    response.headers['Access-Control-Allow-Methods'] = 'GET, POST';

    // Handle OPTIONS for CORS preflight
    if (method === 'OPTIONS') {
      return response.send('', 200);
    }

    // Process and respond
    return response.send({ status: 'success', data: body }, 200);
  } catch (error) {
    console.error('Error:', error);
    return response.send({ error: 'Internal server error' }, 500);
  }
};
```

**Key Methods:**
- `response.send(body, statusCode)` - Send HTTP response
- `request.json()` - Parse request body as JSON
- `request.method` - HTTP method
- `request.params` - URL path parameters
- `request.query` - Query string parameters

### 3. On Interval (Scheduled)

Trigger on a schedule (e.g., every 5 minutes).

```javascript
export default async (event) => {
  const db = require('kvstore');
  const pubnub = require('pubnub');

  try {
    console.log('Scheduled execution at:', new Date().toISOString());

    // Perform periodic tasks
    const stats = await db.get('daily_stats');

    // Publish aggregated data
    await pubnub.publish({
      channel: 'analytics',
      message: { type: 'daily_summary', data: stats }
    });

    return event.ok();
  } catch (error) {
    console.error('Scheduled task error:', error);
    return event.abort();
  }
};
```

**Key Methods:**
- `event.ok()` - Signal successful execution
- `event.abort()` - Signal execution failure

## Channel Patterns (Wildcards)

Configure which channels trigger the function.

### Rules

- Wildcard (`*`) must be at the **end**
- Maximum **two literal segments** before wildcard
- Period (`.`) is the delimiter

### Valid Patterns

```
alerts.*           → matches alerts.critical, alerts.info
user.notifications.* → matches user.notifications.email
chat.*             → matches chat.room1, chat.general
```

### Invalid Patterns

```
alerts.*.critical  → wildcard not at end
*.notifications    → wildcard at start
a.b.c.d.*          → too many segments
```

## Async/Await Best Practices

### Always Use async/await

```javascript
// CORRECT: async/await with try/catch
export default async (request) => {
  const db = require('kvstore');

  try {
    const data = await db.get('myKey');
    const count = await db.incrCounter('visits');

    request.message.data = data;
    request.message.visitCount = count;

    return request.ok();
  } catch (error) {
    console.error('Error:', error);
    return request.abort();
  }
};
```

```javascript
// INCORRECT: .then()/.catch() chains (avoid)
export default async (request) => {
  db.get('myKey')
    .then(data => { /* ... */ })
    .catch(err => { /* ... */ });
  // This won't work properly
};
```

### Parallel Operations with Promise.all

```javascript
export default async (request) => {
  const db = require('kvstore');
  const xhr = require('xhr');
  const vault = require('vault');

  try {
    // Execute independent operations in parallel
    const [userData, apiKey, config] = await Promise.all([
      db.get(`user:${request.message.userId}`),
      vault.get('external_api_key'),
      xhr.fetch('https://api.config.example.com/settings')
    ]);

    // Process results
    if (!userData) throw new Error('User not found');

    request.message.user = userData;
    request.message.config = JSON.parse(config.body);

    return request.ok();
  } catch (error) {
    console.error('Error:', error);
    return request.ok(); // Still allow message with error context
  }
};
```

## Execution Limits

| Limit | Value |
|-------|-------|
| Function chain depth | 3 executions |
| Operations per execution | 3 (KV + XHR + publish) |
| Execution timeout | Few seconds |
| Response payload | Reasonable size |

### Handling Limits

```javascript
export default async (request) => {
  const db = require('kvstore');
  const pubnub = require('pubnub');

  try {
    // 1 operation: KV read
    const data = await db.get('myKey');

    // 2 operation: KV write
    await db.set('lastAccess', Date.now(), 60);

    // 3 operation: publish
    await pubnub.fire({
      channel: 'analytics',
      message: { event: 'processed' }
    });

    // LIMIT REACHED - no more operations allowed
    // Any additional KV/XHR/publish will fail

    return request.ok();
  } catch (error) {
    console.error('Error:', error);
    return request.abort();
  }
};
```

## Logging and Debugging

```javascript
export default async (request) => {
  // Logs appear in PubNub Admin Portal
  console.log('Processing message:', JSON.stringify(request.message));
  console.log('Channel:', request.channels[0]);

  try {
    // Your logic
    console.log('Processing successful');
    return request.ok();
  } catch (error) {
    console.error('Processing failed:', error.message);
    console.error('Stack:', error.stack);
    return request.abort();
  }
};
```

View logs in the PubNub Admin Portal under Functions > Your Function > Logs.

## Function Template

```javascript
export default async (request) => {
  // 1. Import required modules
  const db = require('kvstore');
  const xhr = require('xhr');
  const vault = require('vault');
  const pubnub = require('pubnub');

  try {
    // 2. Validate input
    if (!request.message || !request.message.userId) {
      console.error('Invalid message format');
      return request.abort();
    }

    // 3. Perform business logic
    const userId = request.message.userId;
    const userData = await db.get(`user:${userId}`);

    // 4. Transform or enrich message
    request.message.enriched = true;
    request.message.userData = userData;
    request.message.processedAt = new Date().toISOString();

    // 5. Optional: side effects (analytics, notifications)
    await pubnub.fire({
      channel: 'analytics',
      message: { event: 'message_processed', userId }
    });

    // 6. Return success
    return request.ok();

  } catch (error) {
    // 7. Handle errors gracefully
    console.error('Function error:', error);

    // Option A: Block the message
    return request.abort();

    // Option B: Allow message with error flag
    // request.message.error = error.message;
    // return request.ok();
  }
};
```
