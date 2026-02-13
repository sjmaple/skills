# Order Processing Pipeline

## Context

An e-commerce platform processes orders through a PubNub Before Publish function on `orders.*` channels. When an order message arrives, the function needs to:

1. Look up the customer's loyalty tier from KVStore
2. Record the order timestamp in KVStore
3. Fetch the current exchange rate from an external API
4. Send an analytics event via PubNub
5. Send a notification to a separate PubNub channel

The customer ID is in `request.message.customerId` and the order total is in `request.message.total`.

## Task

Write a PubNub Before Publish function that handles this order processing. Consider the runtime constraints of the platform when designing how many external calls the function makes.

## Expected Outputs

A single JavaScript file containing the complete PubNub Function implementation, with comments explaining any design decisions around platform constraints.
