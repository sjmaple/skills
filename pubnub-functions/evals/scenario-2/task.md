# Chat Message Side Effects

## Context

A chat application uses a PubNub Before Publish function on `chat.*` channels. When a user sends a message, the function needs to handle two side effects:

1. **Analytics tracking**: Record a `message_sent` event to an internal `analytics.chat` channel that is consumed only by a separate analytics aggregation function (no client subscribers listen on this channel).

2. **Typing indicator clear**: When the message is published, clear the user's typing status by sending a lightweight `{ typing: false }` status to the `typing.{channelId}` channel. This data is ephemeral — it should not be stored in message history.

The message contains `request.message.userId` and the channel is available via `request.channels[0]`.

## Task

Write a PubNub Before Publish function that implements both side effects, choosing the appropriate PubNub messaging method for each based on the delivery requirements described above.

## Expected Outputs

A single JavaScript file containing the complete PubNub Function implementation.
