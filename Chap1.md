# Chapter 1 - Reliable, Scalable, and Maintainable Applications

## Reliability
- Perform function as expected
- Tolerate user mistakes / unexpected inputs
- 'Good enough' performance for expected load
- Security - prevent unauthorized access/abuse

Types of Faults
- Hardware Faults - Hard disks crashing, RAM failure, power failure. If hard drives are supposed to last 10-50 years - On a cluster with 10,000 disks, we can expect one disk to fail daily on average.
- Software Errors - Bugs, resource-hogging programs (eg if one spark job takes up a lot of the cache?), cascading failures
- Human Errors - Dev/Test/Prod environments, minimize error opportunities, testing, roll-backs, monitoring/logging

## Scalability
Load Parameters - e.g. requests per second to webserver, database read/write ratio, cache hit rate, etc

Performance Parameters - throughput (number of records processed per second), response time (seconds between client sending a request and receiving a response)

Due to random variations, distribution is more informative than individual request times. You can also use 'median', 'mean' or 'percentiles' for more clarity. Tail latency amplification - Tail latencies also matter: if a browser makes multiple calls to a server, the request has to wait till the slowest call is completed despite faster calls being completed already.

Approaches
- Scaling up / Vertical scaling - Bigger machines
- Scaling out / Horizontal scaling - Distributing load across multiple machines

## Maintainability
- Operability
- Simplicity
- Evolvability
