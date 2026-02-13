# Public Status API

## Context

A service status page needs a simple API endpoint hosted on PubNub Functions. The endpoint will be called from browser JavaScript on a different domain to display current system status.

The endpoint should support:
- **GET**: Return the current status from KVStore (key: `system_status`)
- **POST**: Update the status (accepts JSON body with a `status` field, stores it in KVStore)

Browser-based clients from any origin will call this endpoint directly.

## Task

Write a PubNub On Request function that implements this status API. Ensure it works correctly when called from web browsers on different domains.

## Expected Outputs

A single JavaScript file containing the complete PubNub Function implementation.
