# Alert Webhook Forwarder

## Context

A team wants to forward high-priority alert messages from their PubNub channel to an external notification service. When a message is published to any channel matching `alerts.*`, the function should send the alert text to a webhook endpoint.

The team's notification endpoint URL is: `https://notify.example.com/webhook/ingest/team-alerts`

## Task

Write a PubNub Before Publish function that forwards alert messages to the external webhook. The function should POST a JSON payload with a `text` field containing the alert message text from `request.message.text`.

## Expected Outputs

A single JavaScript file containing the complete PubNub Function implementation.
