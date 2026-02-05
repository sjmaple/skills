# PubNub Message Persistence

## Overview

Message Persistence (Storage & Playback) stores published messages for later retrieval via the History API.

### Default Behavior (Without Persistence)

- Messages cached briefly (~20 minutes)
- Last 100 messages per channel in buffer
- Lost if not received by subscribers

### With Message Persistence

- Messages stored for configured retention
- Retrievable via History API
- Supports catch-up for offline clients

## Enabling Message Persistence

1. Log in to PubNub Admin Portal
2. Select your Application and Keyset
3. Enable **Message Persistence** add-on
4. Configure **Retention Period**

### Retention Options

| Plan | Available Retention |
|------|---------------------|
| Free | Up to 7 days |
| Starter | Up to 180 days |
| Pro | Up to unlimited |

## Storing Messages

### Automatic Storage

Once enabled, all messages are stored automatically:

```javascript
await pubnub.publish({
  channel: 'chat-room',
  message: { text: 'Hello!' }
});
// Message automatically stored
```

### Selective Storage

Exclude specific messages from storage:

```javascript
await pubnub.publish({
  channel: 'chat-room',
  message: { text: 'Ephemeral message' },
  storeInHistory: false  // Don't store this message
});
```

## Retrieving Messages (History API)

### Basic History Fetch

```javascript
const result = await pubnub.fetchMessages({
  channels: ['chat-room'],
  count: 100  // Max 100 per request
});

const messages = result.channels['chat-room'];
messages.forEach(msg => {
  console.log('Message:', msg.message);
  console.log('Timetoken:', msg.timetoken);
  console.log('Publisher:', msg.uuid);
});
```

### Pagination

```javascript
async function fetchAllHistory(channel, since = null) {
  const allMessages = [];
  let startTimetoken = since;

  while (true) {
    const result = await pubnub.fetchMessages({
      channels: [channel],
      count: 100,
      start: startTimetoken  // Fetch older than this
    });

    const messages = result.channels[channel] || [];
    if (messages.length === 0) break;

    allMessages.push(...messages);
    startTimetoken = messages[messages.length - 1].timetoken;

    // Safety check
    if (messages.length < 100) break;
  }

  return allMessages;
}
```

### Time Range Fetch

```javascript
// Fetch messages between two times
const result = await pubnub.fetchMessages({
  channels: ['chat-room'],
  start: '17000000000000000',  // Before this timetoken
  end: '17100000000000000',    // After this timetoken
  count: 100
});
```

### Include Message Actions

```javascript
const result = await pubnub.fetchMessages({
  channels: ['chat-room'],
  count: 100,
  includeMessageActions: true  // Include reactions, etc.
});

messages.forEach(msg => {
  console.log('Message:', msg.message);
  console.log('Actions:', msg.actions);  // Reactions, etc.
});
```

### Multi-Channel Fetch

```javascript
const result = await pubnub.fetchMessages({
  channels: ['chat-room-1', 'chat-room-2', 'notifications'],
  count: 25  // 25 per channel
});

for (const [channel, messages] of Object.entries(result.channels)) {
  console.log(`${channel}: ${messages.length} messages`);
}
```

## Catch-Up Patterns

### Reconnection Catch-Up

```javascript
class MessageCatchUp {
  constructor(pubnub, channel) {
    this.pubnub = pubnub;
    this.channel = channel;
    this.lastTimetoken = this.loadLastTimetoken();
  }

  loadLastTimetoken() {
    return localStorage.getItem(`lastTimetoken_${this.channel}`);
  }

  saveLastTimetoken(timetoken) {
    localStorage.setItem(`lastTimetoken_${this.channel}`, timetoken);
    this.lastTimetoken = timetoken;
  }

  async catchUp() {
    if (!this.lastTimetoken) return [];

    const result = await this.pubnub.fetchMessages({
      channels: [this.channel],
      start: this.lastTimetoken,
      count: 100
    });

    const messages = result.channels[this.channel] || [];

    if (messages.length > 0) {
      this.saveLastTimetoken(messages[messages.length - 1].timetoken);
    }

    return messages;
  }

  onMessage(message) {
    this.saveLastTimetoken(message.timetoken);
  }
}

// Usage
const catchUp = new MessageCatchUp(pubnub, 'chat-room');

// On app start
const missedMessages = await catchUp.catchUp();
missedMessages.forEach(msg => handleMessage(msg));

// Listen for new messages
pubnub.addListener({
  message: (event) => {
    catchUp.onMessage(event);
    handleMessage(event);
  }
});
```

### Initial Load Pattern

```javascript
async function initializeChatRoom(channel) {
  // 1. Fetch recent history
  const history = await pubnub.fetchMessages({
    channels: [channel],
    count: 50,
    includeMessageActions: true
  });

  const messages = history.channels[channel] || [];

  // 2. Subscribe for new messages
  pubnub.subscribe({ channels: [channel] });

  // 3. Return initial messages
  return messages.map(m => ({
    id: m.timetoken,
    text: m.message.text,
    sender: m.uuid,
    timestamp: parseInt(m.timetoken) / 10000,
    reactions: m.actions?.reaction || {}
  }));
}
```

## Message Actions with Persistence

### Soft Delete Pattern

```javascript
// Mark message as deleted (don't actually delete)
await pubnub.addMessageAction({
  channel: 'chat-room',
  messageTimetoken: targetTimetoken,
  action: { type: 'deleted', value: 'true' }
});

// When loading history, filter deleted messages
const history = await pubnub.fetchMessages({
  channels: ['chat-room'],
  count: 100,
  includeMessageActions: true
});

const visibleMessages = history.channels['chat-room'].filter(msg => {
  const deleted = msg.actions?.deleted;
  return !deleted || !Object.keys(deleted).length;
});
```

### Read Receipts

```javascript
// Mark message as read
await pubnub.addMessageAction({
  channel: 'chat-room',
  messageTimetoken: messageTimetoken,
  action: { type: 'read', value: currentUserId }
});

// Check who has read a message
const actions = await pubnub.getMessageActions({
  channel: 'chat-room'
});

const readBy = {};
actions.data.forEach(action => {
  if (action.type === 'read') {
    const tt = action.messageTimetoken;
    readBy[tt] = readBy[tt] || [];
    readBy[tt].push(action.value);
  }
});
```

## Encryption with Persistence

Messages encrypted with `cipherKey` are stored encrypted:

```javascript
// Publish with encryption
const pubnub = new PubNub({
  publishKey: 'pub-c-...',
  subscribeKey: 'sub-c-...',
  userId: 'user-123',
  cipherKey: 'my-secret-key'
});

await pubnub.publish({
  channel: 'secure-chat',
  message: { text: 'Secret message' }
});

// Retrieve - automatically decrypted with same cipherKey
const history = await pubnub.fetchMessages({
  channels: ['secure-chat'],
  count: 10
});
// Messages decrypted automatically
```

## Performance Tips

1. **Limit count per request** - Max 100, use pagination
2. **Use time ranges** - Fetch only needed periods
3. **Cache locally** - Don't re-fetch same messages
4. **Batch multi-channel** - Single request for multiple channels
5. **Include only needed data** - Skip message actions if not needed

```javascript
// Efficient history loading
const result = await pubnub.fetchMessages({
  channels: ['channel'],
  count: 50,  // Only what's needed
  start: lastKnownTimetoken,  // Only new messages
  includeMessageActions: false  // Skip if not needed
});
```
