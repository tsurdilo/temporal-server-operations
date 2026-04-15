# Worker Tuning Sessions – Notes & Approach

## Resources I Use to Prepare

### Metrics / Queries

**Namespace Action Graph**

This is the first thing I look at before anything else.

**Query:**
```sql
SELECT
    $__timeInterval(timestamp_interval) as time,
    action_type,
    sum(events_sum) / greatest($__interval_s, 30) as aps
FROM global_metric_saas_actions_agg_30s
WHERE
    $__timeFilter(timestamp_interval)
  AND cluster = '$cluster'
  AND failed_with_system_error = 'false'
  AND namespace_mode = 'active'
  AND namespace='$namespace'
GROUP BY time, action_type
ORDER BY time
```

**What I look for:**
- Understand the overall customer workload pattern
    - Whether there are spikes in workflow starts
    - Whether there are spikes in activities
    - Whether they use local activities
    - Whether they have a high volume of user timers, signals, or queries
- Understand what the workload was doing at a specific point in time
    - For production, I usually look at the last week
    - For a load test, I look at the time window when the test was run

**Why I use it:**
- It gives me a quick view of what kind of workload the customer is running
- It helps me connect worker behavior and tuning decisions to actual workload shape and timing

---

**Namespace Throttling**

Typically the next thing I check is whether the namespace is being throttled.

**Query:**
```sql
SELECT
    $__timeInterval(timestamp) AS time,
    operation,
    grpc_code,
    grpc_message,
    resource_exhausted_cause,
    response_flags,
    count() / $__interval_s as rps
FROM chronicle.global_event_envoy_ingress
WHERE
    timestamp >= toUnixTimestamp($__fromTime) AND timestamp <= toUnixTimestamp($__toTime)
    AND grpc_code ='ResourceExhausted'
    AND x_tmprl_namespace='$namespace'
    AND cluster IN ('$cluster')
    AND protocol = 'HTTP/2'
GROUP BY time, grpc_code, operation, path,
    grpc_message,
    resource_exhausted_cause,
    response_flags
ORDER BY time
```

**What I look for:**
- Whether the customer is being throttled at all
- What type of limits are being hit
    - RPS limits
    - APS limits
    - Concurrent limits
- Which operations are being throttled

**Why I use it:**
- It is difficult to properly tune SDK workers while throttling is occurring
- Throttling often impacts long-poll operations (e.g., PollWorkflowTaskQueue / PollActivityTaskQueue)
- This can effectively cripple workers depending on the severity and duration
- It can also create task backlogs that workers must later recover from

---

**Namespace RPS (Total & Per Operation)**

I also look at overall namespace RPS, as well as RPS broken down per operation.

**Query:**
```sql
SELECT
    $__timeInterval(timestamp) AS time,
    cluster,
    x_tmprl_namespace,
    count() / $__interval_s as rps
FROM global_event_envoy_ingress
WHERE
    timestamp >= toUnixTimestamp($__fromTime) AND timestamp <= toUnixTimestamp($__toTime)
    AND cluster='$cluster'
    AND x_tmprl_namespace='$namespace'
GROUP BY time, cluster, x_tmprl_namespace
ORDER BY time
```

**What I look for:**
- Overall RPS patterns
    - Identify spikes and general workload shape
- Per-operation RPS
    - Especially operations not captured in APS
    - Examples:
        - QueryWorkflowExecution
        - GetWorkflowExecutionHistory
        - PollWorkflowExecutionHistory
        - Visibility APIs (e.g., ListWorkflowExecutions)

**Why I use it:**
- Helps build a more complete picture of the customer’s workload beyond APS
- Surfaces “non-billable” operations that still impact system behavior
- Highlights additional pressure on workers from read/query-heavy patterns
- Shows how frequently these operations are being executed

---

## Approach to Scenarios

At this point, the above metrics give me the baseline understanding needed to decide where to go next.

There are multiple possible directions depending on the customer’s situation and priorities. Worker tuning sessions are not one-size-fits-all and require different approaches based on context.

Examples:

- Customer ran a load test for a non–high-throughput use case
    - Goal is typically to understand behavior and validate expectations

- Customer is testing for high load / high throughput
    - Focus shifts to optimizing for performance at a very granular level (every millisecond matters)

- Customer is in production experiencing issues (e.g., high latencies)
    - Focus is on identifying bottlenecks and stabilizing performance

- Customer is in production with business-impacting issues
    - Priority is rapid mitigation and recovery before deeper optimization

The metrics above are common starting points across all of these scenarios and help guide which path to take next.

---

### Scenario 1: Customer ran a load test for a non–high-throughput use case

**Goal:**
- Evaluate the overall “health” of the load test
- Validate that the system behaved as expected under the given workload

**Step 1: Check for Throttling (Highest Priority)**
- One of the main things I look for during load tests is whether any throttling occurred
- If there is throttling:
    - Almost always the first step is to increase namespace limits and have the customer re-run the load test
    - Tuning workers while throttling is present is not very effective
- Exception:
    - If throttling is very minimal, it may be acceptable to proceed
    - If throttling is substantial, it is best to stop and re-run after limits are increased

- Primary graph used:
    - ResourceExhausted (throttling) graph mentioned earlier

---

**Step 2: Analyze Schedule-To-Start Latencies**

**Query:**
```sql
SELECT
  $__timeInterval(timestamp_interval) as time,
  operation, cluster, namespace,
  quantileInterpolatedWeighted(0.95)(bucket, sum_bucket_events_count) as p99
FROM global_metric_schedule_to_start_latency__agg__heatmap_percentile_30s
WHERE
  $__timeFilter(timestamp_interval)
  AND cluster='$cluster'
  AND namespace='$namespace'
group by
  time, operation, cluster, namespace
order by time asc
```

- This graph shows:
    - Workflow Task and Activity Task ScheduleToStart latencies (ms)

**What I look for:**
- I am not overly concerned with small latency spikes
- I note spikes to understand:
    - What the workload was doing at that time
    - Whether spikes correlate with:
        - APS / RPS changes
        - Throttling events (ResourceExhausted)

---

**Step 3: Check for Errors (Non-OK gRPC Responses)**

**Query:**
```sql
SELECT
    toStartOfInterval(timestamp_interval, INTERVAL $__interval_s second) as time,
    grpc_code,
    namespace,
    sum(sum_bucket_events_count) / greatest($__interval_s, ${agg:value}) AS rps
FROM global_event_envoy_ingress__agg__heatmap_percentile_${agg:text}
WHERE
    timestamp_interval >= toUnixTimestamp($__fromTime) AND timestamp_interval <= toUnixTimestamp($__toTime)
    AND cluster in ($cluster)
    AND operation in ($operations)
    AND grpc_code !='OK'
    AND namespace='$namespace'
GROUP BY time, namespace, grpc_code
ORDER BY time
```

- This graph represents:
    - Non-OK gRPC responses from operations sent by customer workers and clients

- Equivalent to:
    - SDK metrics: temporal_request_failure and temporal_long_request_failure

**Important context:**
- Errors can originate from:
    - Temporal Cloud (server-side)
    - Envoy (before requests reach the server)

**What I look for:**
- Any noticeable spikes in error rates
- Correlation with:
    - Latency increases
    - Throttling
    - Changes in workload (APS / RPS)

**Notes:**
- Interpreting this graph correctly can be non-trivial
- It requires correlating with other metrics and understanding request flow
- Many of the same concepts from SDK failure metrics apply here as well
- Reference:
    - https://github.com/tsurdilo/temporal-metrics/blob/main/SDK_METRICS_REQUEST_FAILURES.md
---

**Outcome:**
- Determine whether the load test results are valid (i.e., not skewed by throttling or errors)
- Identify any obvious issues before moving into deeper tuning or optimization