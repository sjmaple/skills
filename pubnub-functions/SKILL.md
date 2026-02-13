---
name: pubnub-functions
description: Create, configure, and deploy PubNub Functions 2.0 event handlers, triggers, and serverless endpoints. Use when building real-time message transformations, PubNub modules, webhook integrations, or edge data processing.
license: PubNub
metadata:
  author: pubnub
  version: "0.2.2"
  domain: real-time
  triggers: pubnub, pubnub functions, functions, serverless, edge, kvstore, webhook, transform, event handler, real-time functions, message processing
  role: specialist
  scope: implementation
  output-format: code
---

# PubNub Functions 2.0

## Core Workflow

1. **Identify Function Type**: Before Publish, After Publish, On Request, or On Interval
2. **Design Logic**: Plan the transformation, integration, or business logic
3. **Implement Function**: Write async/await code with proper error handling
4. **Use Modules**: Leverage kvstore, xhr, vault, pubnub, crypto modules
5. **Validate Implementation**: Verify no hardcoded secrets (use vault), confirm async/await usage (no .then() chains), check operation count stays within the 3-operation limit, and ensure proper try/catch wrapping
6. **Handle Response**: Return ok()/abort() or send() appropriately
7. **Configure Channel Patterns**: Set wildcard patterns ending with `.*`, max two literal segments before wildcard
8. **Test in Staging**: Test the function in PubNub Admin Portal with sample messages before enabling on production channels
9. **Deploy to Production**: Enable the function on live channel patterns and monitor logs

## Reference Guide

| Reference | Purpose |
|-----------|---------|
| [functions-basics.md](references/functions-basics.md) | Function structure, event types, async/await patterns |
| [functions-modules.md](references/functions-modules.md) | KVStore, XHR, Vault, Crypto, JWT, UUID modules |
| [functions-patterns.md](references/functions-patterns.md) | Common patterns: counters, aggregation, webhooks |

## Key Implementation Requirements

### Function Structure

```javascript
// Always use default async export
export default async (request) => {
  const db = require('kvstore');
  const xhr = require('xhr');

  try {
    // Your logic here
    return request.ok();  // Allow message to proceed
  } catch (error) {
    console.error('Error:', error);
    return request.abort();  // Block message
  }
};
```

### HTTP Endpoint Function

```javascript
export default async (request, response) => {
  try {
    const body = await request.json();
    // Process request
    return response.send({ success: true }, 200);
  } catch (error) {
    return response.send({ error: 'Server error' }, 500);
  }
};
```

## Constraints

- Maximum 3 chained function executions
- Maximum 3 combined operations per execution (KV, XHR, publish)
- Always use async/await (not .then()/.catch())
- Always wrap logic in try/catch
- Use vault for secrets, never hardcode
- Wildcard patterns must end with `.*`

## Output Format

When providing implementations:
1. Include complete, working function code
2. Show proper async/await with try/catch
3. Explain module usage and imports
4. Note channel pattern configuration
5. Include deployment instructions
