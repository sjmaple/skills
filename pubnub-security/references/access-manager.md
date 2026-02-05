# PubNub Access Manager (PAM)

## Overview

PubNub Access Manager provides fine-grained control over who can access channels and what actions they can perform.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Permissions** | Read (subscribe), Write (publish), Manage (channel groups) |
| **Auth-Key** | Client authentication token for permission grants |
| **Secret Key** | Server-side key for granting/revoking permissions |
| **TTL** | Time-to-live for permission grants (1-43,200 minutes) |

## Enabling Access Manager

1. Log in to PubNub Admin Portal
2. Select your Application and Keyset
3. Enable **Access Manager** add-on
4. **Securely store your Secret Key** - required for grants

> **Critical**: Once enabled, all clients need valid auth tokens to access channels.

## Server-Side Permission Grants

### Setup (Node.js)

```javascript
const PubNub = require('pubnub');

const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  secretKey: 'sec-c-...',  // Server-side only!
  userId: 'server'
});
```

### Grant Read/Write Access

```javascript
// Grant user access to specific channel
async function grantUserAccess(authKey, channel, permissions) {
  try {
    const result = await pubnub.grant({
      channels: [channel],
      authKeys: [authKey],
      read: permissions.read ?? true,
      write: permissions.write ?? false,
      manage: permissions.manage ?? false,
      ttl: 60  // 1 hour
    });
    console.log('Grant successful:', result);
    return result;
  } catch (error) {
    console.error('Grant failed:', error);
    throw error;
  }
}

// Usage
await grantUserAccess('user-123-auth-token', 'chat-room-private', {
  read: true,
  write: true
});
```

### Grant for Multiple Channels

```javascript
await pubnub.grant({
  channels: ['room-1', 'room-2', 'room-3'],
  authKeys: ['user-auth-token'],
  read: true,
  write: true,
  ttl: 1440  // 24 hours
});
```

### Grant Using Channel Patterns

```javascript
// Grant access to all channels matching pattern
await pubnub.grant({
  channels: ['chat-room-*'],  // Wildcard pattern
  authKeys: ['user-auth-token'],
  read: true,
  write: false,
  ttl: 60
});
```

### Grant for Channel Groups

```javascript
await pubnub.grant({
  channelGroups: ['user-feeds-group'],
  authKeys: ['user-auth-token'],
  read: true,
  manage: true,  // Allow adding/removing channels in group
  ttl: 60
});
```

### Grant for Presence Channels

```javascript
// Main channel access
await pubnub.grant({
  channels: ['chat-room'],
  authKeys: ['user-auth-token'],
  read: true,
  write: true,
  ttl: 60
});

// Presence channel access (required for withPresence)
await pubnub.grant({
  channels: ['chat-room-pnpres'],
  authKeys: ['user-auth-token'],
  read: true,
  ttl: 60
});
```

## Client-Side Configuration

### Correct Configuration (authKey)

```javascript
// After receiving auth token from your server
const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  publishKey: 'pub-c-...',  // Only if client needs to publish
  userId: 'user-123',
  authKey: 'auth-token-from-server'  // CORRECT
});
```

### WRONG Configuration (token)

```javascript
// DO NOT use 'token' parameter
const pubnub = new PubNub({
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  token: 'auth-token'  // WRONG - will not work
});
```

## Revoking Permissions

```javascript
// Revoke specific user's access
await pubnub.revoke({
  channels: ['private-room'],
  authKeys: ['user-auth-token']
});

// Revoke all access for a user
await pubnub.revoke({
  authKeys: ['user-auth-token']  // All channels
});
```

> **Note**: Revokes may take up to 60 seconds to propagate due to caching.

## Authentication Flow

### Complete Server Flow

```javascript
// Express.js example
const express = require('express');
const PubNub = require('pubnub');
const jwt = require('jsonwebtoken');

const app = express();

// Server PubNub instance with Secret Key
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  secretKey: 'sec-c-...',
  userId: 'server'
});

// Auth endpoint
app.post('/api/pubnub/auth', async (req, res) => {
  try {
    // 1. Verify user's session (your auth logic)
    const userToken = req.headers.authorization?.split(' ')[1];
    const user = jwt.verify(userToken, process.env.JWT_SECRET);

    // 2. Generate PubNub auth token
    const authKey = `pn_${user.id}_${Date.now()}`;

    // 3. Determine channels based on user permissions
    const channels = getUserChannels(user);

    // 4. Grant PubNub access
    await pubnub.grant({
      channels: channels,
      authKeys: [authKey],
      read: true,
      write: user.canWrite,
      ttl: 60  // 1 hour, then client must re-auth
    });

    // 5. Return credentials to client
    res.json({
      authKey: authKey,
      subscribeKey: 'sub-c-...',
      publishKey: user.canWrite ? 'pub-c-...' : null,
      channels: channels,
      expiresAt: Date.now() + (60 * 60 * 1000)  // 1 hour
    });
  } catch (error) {
    console.error('Auth error:', error);
    res.status(401).json({ error: 'Authentication failed' });
  }
});
```

### Client Flow

```javascript
async function initializePubNub() {
  // 1. Get auth token from your server
  const response = await fetch('/api/pubnub/auth', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${userSessionToken}`
    }
  });

  const credentials = await response.json();

  // 2. Initialize PubNub with auth token
  const pubnub = new PubNub({
    subscribeKey: credentials.subscribeKey,
    publishKey: credentials.publishKey,
    userId: currentUserId,
    authKey: credentials.authKey  // From server
  });

  // 3. Handle access denied errors
  pubnub.addListener({
    status: (statusEvent) => {
      if (statusEvent.category === 'PNAccessDeniedCategory') {
        console.error('Access denied - refreshing token');
        refreshAuthToken();
      }
    }
  });

  // 4. Schedule token refresh before expiry
  const refreshTime = credentials.expiresAt - Date.now() - 300000;  // 5 min before
  setTimeout(refreshAuthToken, refreshTime);

  return pubnub;
}
```

## Permission Levels

### Channel-Level Permissions

| Permission | Allows |
|------------|--------|
| `read` | Subscribe to channel, fetch history |
| `write` | Publish messages to channel |
| `manage` | Add/remove channels in channel groups |
| `delete` | Delete messages (if supported) |

### Grant Scope

```javascript
// User-channel specific
await pubnub.grant({
  channels: ['chat-room'],
  authKeys: ['user-auth-token'],
  read: true
});

// All channels for user (use carefully)
await pubnub.grant({
  authKeys: ['user-auth-token'],
  read: true,
  write: true
});
```

## TTL Best Practices

| Use Case | Recommended TTL |
|----------|-----------------|
| Short session (demo) | 15-60 minutes |
| Normal user session | 60-1440 minutes (1-24 hours) |
| Long-lived service | 1440-10080 minutes (1-7 days) |
| Maximum | 43200 minutes (30 days) |

```javascript
// Short TTL for sensitive operations
await pubnub.grant({
  channels: ['payment-updates'],
  authKeys: [authKey],
  read: true,
  ttl: 15  // 15 minutes
});
```
