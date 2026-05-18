# Rate Limiting — System Design Notes

## What it is
Rate Limiting controls how many requests you can make to an API, service, server, etc.
The requirements for it to be properly set up are:
1. Setting up configurable rules
2. Respond with the adequate code and headers (HTTP 429 / Remaining rate and reset time)
3. Has minimal latency
4. It is highly available

## Algorithms comparison

| Algorithm | How it works | Best for | Trade-off |
|-----------|-------------|----------|-----------|
| Token Bucket | This system fills tokens at a fixed rate with a bucket capacity limit which cannot be exceeded. Every time a request is received the stablished amount of tokens is consumed. If not enough tokens are remaining the request is denied. | Public API rate-limiting | It is flexible when handling short-term burst requests however it does not guarantee a smooth request rate|
| Leaky Bucket | This system receives requests within a bucket capacity limit that can be injected at any rate. This requests are queued for processing until the threshold is reached where the requests get dropped, meanwhile the queued requests are processed at a fixed rate | Network Bandwith Management, Video Streaming | It has stable traffic control, however there is no flexibility |
| Fixed Window Counter | This system divides time into fixed windows and within each window counts the number of requests. The excess will be rejected until the next window begins | API Rate-Limiting, Access Control | It allows short-term peak requests and it is easy to implement, however there is a window boundary issue where the threshold can be overseen |
| Sliding Window Log | This system divides time into dynamic windows that shift continuously during the time. The system logs the timestamp of the accepted requests and checks the incomming ones against the number of accepted in a recent window.  | Authentication | It has fine-grained traffic control however the complexity increases in processing |
| Sliding Window Counter | This system is a balance between fixed window and sliding window log since it calculates the weight by taking into account the requests received on the last window as well as the time of the current one | API Rate-Limiting | It has an efficient memory since it only stores counters and good scalability, however is not as precise on the windows limits |

## When to use each

**Token Bucket**: It should be used when the users need flexibility for short bursts of traffic. For example: A developer calling your public API to sync data.

**Leaky Bucket**: It should be used when the traffic output is needed to flow at a perfectly constant rate regardless of how requests arrive. For example: A video streaming service where frames must be delivered at a fixed rate to avoid buffering.

**Fixed Window Counter**: It should be used when simplicity is the priority and small inaccuracies at window boundaries are acceptable. For example: An internal dashboard where users can refresh data 100 times per hour.

**Sliding Window Log**: It should be used when the limit is a hard requirement with no margin for error. For example: A login endpoint where exactly 3 failed attempts must trigger a block.

**Sliding Window Counter**: It should be used when you need to scale to millions of users without burning memory, and a small margin of imprecission is acceptable. For example Limiting tweets to 300 per 3 hours on a platform with 400 million users here storing two counters per user in Redis is viable, storing a timestamp log for every request is not.

## Key insight
There is no perfect rate limiting algorithm — every solution is a trade-off between precision, memory efficiency and implementation complexity. The right choice depends entirely on what your system cannot afford to get wrong.

In practice, most production APIs use Token Bucket or Sliding Window Counter because they balance flexibility and efficiency well enough for the majority of use cases. The middleware layer (not client-side, not server-side) is the recommended placement since it centralises the logic and works regardless of how many servers are behind it. Race conditions in distributed systems are solved with atomic operations in Redis, a detail that separates a theoretical understanding from a production-ready implementation.

## Applied
Already implemented token bucket in `inventory-api` with `express-rate-limit`.