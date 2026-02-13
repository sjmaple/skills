# Page View Counter

## Context

A content platform wants to track view counts for articles. When a user views an article, a message is published to a `views.*` channel with `request.message.articleId` identifying the article.

The function should increment the view count for that article and add the updated total to the message so subscribers can display it. Multiple users may view the same article simultaneously from different regions.

## Task

Write a PubNub Before Publish function that tracks per-article view counts. The counting mechanism must produce correct results even when multiple requests for the same article arrive concurrently.

## Expected Outputs

A single JavaScript file containing the complete PubNub Function implementation.
