# Activity Heartbeat Timeouts — Finding the Cause and Tuning Playbook

**Temporal activities can run for a long time.** A single attempt of an activity is bounded by its
**StartToClose timeout**, and an activity's StartToClose timeout is set by its business logic. Some
activities need a longer StartToClose timeout, for example an activity that transfers a large file
to a downstream service, or a single activity that processes many items.

> **Note:** "Long-running" is relative in this playbook. There is no duration above which an activity
> counts as long-running, and, depending on your use case, an activity that runs for a minute can
> need the same care as one that runs for hours. The sections that follow give you what you need to
> judge it for your own activities.

**When an activity needs a long StartToClose timeout, three things are worth thinking about:**

- **The server can be slow to notice an SDK worker that crashes while running an activity attempt.**
  If the SDK worker crashes or loses its network after starting to execute the activity, the server
  waits for the attempt's result until the StartToClose timeout runs out, although in this case the
  result never comes. Only then does the server time out the attempt and check the activity's retry
  policy, to either retry the activity or fail it with the timeout.
- **Each activity retry is a new execution (attempt) of the activity.** Each attempt runs the
  activity code from the beginning, with the same input.
- **The SDK worker cannot learn that the activity was cancelled, paused or reset while its attempt
  is running.** The attempt keeps running until its code finishes. If the activity was cancelled or
  paused, the server still accepts the result; if the attempt timed out or the workflow was reset,
  the server rejects it.

**Activity heartbeats are an optional Temporal feature for long-running activities.** An **activity
heartbeat** is a short "still working" message, like a ping, that the SDK worker sends to the server
while it runs an activity attempt. An SDK worker running many activities at the same time sends
heartbeats for each of them separately. Each heartbeat is a `RecordActivityTaskHeartbeat` API call
to the server.

**Application developers choose which activities heartbeat, by:**

- **Setting a heartbeat timeout in the activity options** (`HeartbeatTimeout`), in the workflow code
  that schedules the activity:

  ```go
  ao := workflow.ActivityOptions{
      StartToCloseTimeout: 2 * time.Hour,
      HeartbeatTimeout:    2 * time.Minute,
  }
  ctx = workflow.WithActivityOptions(ctx, ao)
  err := workflow.ExecuteActivity(ctx, ProcessFiles, files).Get(ctx, nil)
  ```

- **Calling the SDK's heartbeat API in the activity code** (`RecordHeartbeat` in the Go SDK,
  `heartbeat()` in the Java SDK), for example once per processed item. The SDK sends a heartbeat only
  when the activity code calls it, never on its own:

  ```go
  func ProcessFiles(ctx context.Context, files []string) error {
      start := 0
      if activity.HasHeartbeatDetails(ctx) {
          var lastDone int
          if err := activity.GetHeartbeatDetails(ctx, &lastDone); err == nil {
              start = lastDone + 1 // Resume after the last file done
          }
      }
      for i := start; i < len(files); i++ {
          if err := processFile(ctx, files[i]); err != nil {
              return err
          }
          activity.RecordHeartbeat(ctx, i) // Heartbeat details: the last file done
          if ctx.Err() != nil {
              // Do any cleanup if needed
              return ctx.Err() // Return the error to end this attempt
          }
      }
      return nil
  }
  ```

  When the server's reply to a heartbeat says the activity was cancelled, paused or reset, the Go SDK
  cancels the activity's context, so the activity checks the context after each heartbeat and returns.
  For examples in another language, see the documentation of your Temporal SDK.

> **Recommendation:** set the heartbeat timeout much shorter than the StartToClose timeout, for
> example two minutes for an activity with a two-hour StartToClose timeout. The server then uses both
> timeouts. It times out an activity attempt when the StartToClose timeout runs out, or earlier, when
> no heartbeat arrives within the heartbeat timeout. In both cases the server decides whether to retry
> the activity based on the retry policy set in its activity options.

**With a heartbeat timeout set and the heartbeat API called, heartbeats address all three points
above:**

- **A crashed SDK worker is noticed sooner.** When no heartbeat arrives within the heartbeat timeout,
  the server times out the attempt, which is an **activity heartbeat timeout**, and applies the retry
  policy without waiting for the StartToClose timeout.
- **An activity retry can continue where the previous attempt stopped.** A heartbeat can carry
  **heartbeat details**, a small payload recording the activity's progress, and the next activity
  attempt receives the last ones recorded.
- **A running attempt learns about cancellation, pause and reset.** The server answers every
  heartbeat it receives, and the answer says whether the activity was cancelled, paused or reset, or
  whether the attempt has already timed out. The SDK reads the answer and tells the activity code.
  The Go SDK does this by cancelling the activity's context, which is why the sample above checks the
  context after each heartbeat.

If the heartbeat timeout is not set in the activity options, the activity code can still call the
heartbeat API. The server then times out the activity attempt only at the StartToClose timeout. The
server's answer to a heartbeat still tells the activity about cancellation, pause or reset.

**Heartbeating is available only to regular activities.** Local activities cannot heartbeat. If
local activity code calls the heartbeat API, the SDK ignores the call and sends nothing to the
server.

So far, this playbook has covered what to think about when an activity needs a long StartToClose
timeout, what activity heartbeats are, and how the server uses the heartbeat timeout to time out an
activity attempt.

**The rest of this playbook is about unexpected heartbeat timeouts.** An activity attempt can time
out on its heartbeat timeout even though its SDK worker did not crash and its code kept calling the
heartbeat API, because the heartbeats did not reach the server in time. The playbook explains:

- how a heartbeat travels from the activity code to the server;
- how to tell that heartbeat timeouts are happening;
- how to find where the heartbeats were delayed;
- how to fix each cause and prevent it;
- the load that heartbeats put on a cluster;
- the alerts on heartbeat timeouts.

**Audience**

Operators of self-hosted Temporal clusters.

**Related playbooks**

- [History Persistence QPS Limits](./history-persistence-qps-limits.md) covers the database limits
  that a heartbeat's write counts against.
- [History Task Processing](./history-task-processing-tuning.md) covers how the history service
  processes its tasks, including the timer tasks that check the heartbeat deadline.

**What this playbook does not cover**

- **Worker heartbeats** (`frontend.WorkerHeartbeatsEnabled`, on by default from server v1.30.0).
  Each SDK worker sends a worker heartbeat to the server on a regular interval, with the SDK worker's
  own status. Worker heartbeats are sent once per SDK worker, not per activity, and have no effect on
  activity heartbeat timeouts.

**Applies to:**

- Any persistence store.
- The Go and Java SDKs. The SDKs built on the Core SDK (Python, TypeScript, .NET and Ruby) are not
  covered yet.
- Server v1.29.0 or later, for the panels that count every timed-out attempt.

---

## Contents

1. [How Temporal SDKs send activity heartbeats](#1-how-temporal-sdks-send-activity-heartbeats)
    - [1.1 The batching window](#11-the-batching-window)
    - [1.2 What the SDKs do when a heartbeat call fails](#12-what-the-sdks-do-when-a-heartbeat-call-fails)
    - [1.3 Why an SDK worker can send heartbeats late](#13-why-an-sdk-worker-can-send-heartbeats-late)
2. [How the server receives and records a heartbeat](#2-how-the-server-receives-and-records-a-heartbeat)
    - [2.1 The gRPC message size limit](#21-the-grpc-message-size-limit)
    - [2.2 The namespace rate limit in the frontend service](#22-the-namespace-rate-limit-in-the-frontend-service)
    - [2.3 The payload size limit](#23-the-payload-size-limit)
    - [2.4 The history service's requests-per-second limits](#24-the-history-services-requests-per-second-limits)
    - [2.5 How waiting for the workflow lock can make heartbeats late](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late)
    - [2.6 What a heartbeat writes to the database, and the persistence QPS limits](#26-what-a-heartbeat-writes-to-the-database-and-the-persistence-qps-limits)
    - [2.7 When the database write of a heartbeat fails](#27-when-the-database-write-of-a-heartbeat-fails)
3. [How the server works out and checks the heartbeat deadline](#3-how-the-server-works-out-and-checks-the-heartbeat-deadline)
    - [3.1 The heartbeat deadline](#31-the-heartbeat-deadline)
    - [3.2 How a timer task checks the heartbeat deadline](#32-how-a-timer-task-checks-the-heartbeat-deadline)
    - [3.3 A timer task processed late never causes an early heartbeat timeout](#33-a-timer-task-processed-late-never-causes-an-early-heartbeat-timeout)
    - [3.4 Heartbeat timeouts under about one second](#34-heartbeat-timeouts-under-about-one-second)
    - [3.5 Raising the heartbeat timeout of an activity that is already running](#35-raising-the-heartbeat-timeout-of-an-activity-that-is-already-running)
4. [What happens after the server times out an activity on its heartbeat timeout](#4-what-happens-after-the-server-times-out-an-activity-on-its-heartbeat-timeout)
    - [4.1 Whether the server retries the activity after a heartbeat timeout](#41-whether-the-server-retries-the-activity-after-a-heartbeat-timeout)
    - [4.2 What the workflow and its event history show after a heartbeat timeout](#42-what-the-workflow-and-its-event-history-show-after-a-heartbeat-timeout)
    - [4.3 The next activity attempt and the timed-out activity attempt](#43-the-next-activity-attempt-and-the-timed-out-activity-attempt)
    - [4.4 How long after the heartbeats stop the server times out the activity](#44-how-long-after-the-heartbeats-stop-the-server-times-out-the-activity)
    - [4.5 Where heartbeat timeouts show on the dashboard and in `temporal workflow describe`](#45-where-heartbeat-timeouts-show-on-the-dashboard-and-in-temporal-workflow-describe)
5. [How to find where the heartbeats were delayed](#5-how-to-find-where-the-heartbeats-were-delayed)
    - [5.1 Which activity types are timing out on their heartbeat timeout](#51-which-activity-types-are-timing-out-on-their-heartbeat-timeout)
    - [5.2 Heartbeat calls that the server refused](#52-heartbeat-calls-that-the-server-refused)
    - [5.3 Heartbeat calls that were slow on the server](#53-heartbeat-calls-that-were-slow-on-the-server)
    - [5.4 Heartbeat calls that the SDK worker sent late, or that failed before the server](#54-heartbeat-calls-that-the-sdk-worker-sent-late-or-that-failed-before-the-server)
    - [5.5 One activity in detail](#55-one-activity-in-detail)
6. [Which recommendations apply to each cause of late heartbeats](#6-which-recommendations-apply-to-each-cause-of-late-heartbeats)
    - [6.1 Recommendations for heartbeat calls that the server refused](#61-recommendations-for-heartbeat-calls-that-the-server-refused)
    - [6.2 Recommendations for heartbeat calls that were slow on the server](#62-recommendations-for-heartbeat-calls-that-were-slow-on-the-server)
    - [6.3 Recommendations for heartbeat calls that the SDK workers sent late, or that failed before the server](#63-recommendations-for-heartbeat-calls-that-the-sdk-workers-sent-late-or-that-failed-before-the-server)
    - [6.4 Recommendation for an activity whose attempt number keeps going up](#64-recommendation-for-an-activity-whose-attempt-number-keeps-going-up)
7. [Heartbeat load on a cluster with many activities that heartbeat](#7-heartbeat-load-on-a-cluster-with-many-activities-that-heartbeat)
    - [7.1 How much load heartbeats put on the cluster](#71-how-much-load-heartbeats-put-on-the-cluster)
    - [7.2 Where heartbeat load shows on the Temporal Server dashboard](#72-where-heartbeat-load-shows-on-the-temporal-server-dashboard)
    - [7.3 Lowering heartbeat load](#73-lowering-heartbeat-load)
8. [Alerting on heartbeat timeouts](#8-alerting-on-heartbeat-timeouts)
    - [8.1 The alert on heartbeat timeouts](#81-the-alert-on-heartbeat-timeouts)
    - [8.2 SDK alerts for the SDK workers' side of heartbeat timeouts](#82-sdk-alerts-for-the-sdk-workers-side-of-heartbeat-timeouts)
    - [8.3 Why refused heartbeat calls, heartbeat load and late timer tasks have no alert](#83-why-refused-heartbeat-calls-heartbeat-load-and-late-timer-tasks-have-no-alert)

---

## 1. How Temporal SDKs send activity heartbeats

**To describe how the SDK sends activity heartbeats, it first helps to know when it sends them at
all.** The SDK sends heartbeats only when the activity code calls the heartbeat API, and whether
those heartbeats keep an activity attempt from timing out depends on the heartbeat timeout in the
activity options:

- **Heartbeat timeout set, and the activity code calls the heartbeat API.** The SDK sends the
  heartbeats to the server, and each has to arrive within the heartbeat timeout.
- **Heartbeat timeout set, but the activity code never calls the heartbeat API.** The SDK sends
  nothing. The server counts the heartbeat timeout from the start of the attempt, so the attempt
  times out one heartbeat timeout after it started.
- **No heartbeat timeout set, but the activity code calls the heartbeat API.** The SDK still sends
  the heartbeats, and the server times out the attempt only at the StartToClose timeout.

> **Recommendation:** when an activity should heartbeat, do both. Set the heartbeat timeout in its
> activity options, and call the heartbeat API from the activity code.

**This playbook assumes the recommendation above is followed**, so every activity it describes has
a heartbeat timeout set and calls the heartbeat API. Its findings come from tests with the Temporal
Go and Java SDKs. The subsections below describe how the SDK sends those heartbeats, covering when it
sends each heartbeat ([1.1](#11-the-batching-window)), what it does when a heartbeat call fails
([1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails)), and why an SDK worker can send them
late ([1.3](#13-why-an-sdk-worker-can-send-heartbeats-late)).

### 1.1 The batching window

**The SDK does not send every heartbeat call to the server the moment the activity code makes it.**
After it sends a heartbeat, it waits a while before it sends the next one. This wait is the
**batching window**. Heartbeat calls made during the window only replace the details waiting to be
sent, so an activity that calls the heartbeat API many times a second does not make as many calls
to the server.

**When the batching window ends, the SDK sends the latest details if the activity code called the
heartbeat API during the window.** If there was no call, nothing is sent until the next call, which
is then sent at once. So the first heartbeat call, and any call after a quiet period, goes to the
server without waiting.

**The batching window is 80% of the heartbeat timeout, but never longer than 60 seconds**, in both
SDKs. For an activity with a two-minute heartbeat timeout, 80% is 96 seconds, so the window is 60
seconds. The 60-second limit is a worker option, `worker.Options.MaxHeartbeatThrottleInterval` in
the Go SDK and `WorkerOptions.Builder.setMaxHeartbeatThrottleInterval` in the Java SDK. The window
starts when the server has answered the previous heartbeat, not when that heartbeat was sent, so a
heartbeat call that takes a long time pushes the next heartbeat later.

> **Note:** for any heartbeat timeout above 75 seconds, the batching window is 60 seconds, not 80%
> of the heartbeat timeout. So as long as the activity code keeps calling the heartbeat API, the SDK
> sends a heartbeat about once a minute, whether the heartbeat timeout is 2 minutes or 10 minutes.
>
> This does not make a longer heartbeat timeout better by itself. A longer heartbeat timeout gives a
> heartbeat slowed down by a busy SDK worker or a busy server more time to reach the server, but the
> server also notices a crashed SDK worker later, only once the full heartbeat timeout has passed. Up
> to a heartbeat timeout of 75 seconds, a longer heartbeat timeout also lengthens the batching
> window, so the activity learns about cancellation, pause and reset later too. The right heartbeat
> timeout depends on how quickly the activity needs a crash, a cancellation, a pause or a reset to be
> noticed.

> **Recommendation:** leave `MaxHeartbeatThrottleInterval` at its 60-second default, also for
> activities with a long heartbeat timeout. Raising it makes the SDK send fewer heartbeats, but it
> also makes two things worse:
>
> - **The activity learns about cancellation, pause and reset later**, because it learns about them
>   only from the server's answer to a heartbeat.
> - **More heartbeat details can be lost when an SDK worker crashes.** The SDK keeps the details of
>   heartbeat calls made during the batching window on the SDK worker until the window ends. If the
>   SDK worker crashes before then, those details never reach the server. The next activity attempt
>   receives only the last heartbeat details the server got, so activity code that uses them to
>   continue where the previous attempt stopped repeats more work.
>
> Raise `MaxHeartbeatThrottleInterval` only when the number of heartbeats is itself a load on the
> server, as described in [section 7](#7-heartbeat-load-on-a-cluster-with-many-activities-that-heartbeat).

The SDK times heartbeats this way only when every heartbeat call succeeds. A heartbeat call can
also fail, and the Go and Java SDKs handle that differently.

### 1.2 What the SDKs do when a heartbeat call fails

**When the SDK's heartbeat calls keep failing, the server times out the activity attempt while
the activity is still running on the SDK worker.** The heartbeat calls fail before the server
records them, for example because of a network error, or because the server refuses them with
`RESOURCE_EXHAUSTED` when a rate limit is reached. When the heartbeat timeout passes, the server
times out the activity attempt and, if the retry policy allows, starts the next activity attempt,
possibly on another SDK worker. The first SDK worker learns that its activity attempt timed out
only when a later heartbeat call gets through, or when the activity finishes and the SDK reports
the result to the server. Until then, two attempts of the same activity run at the same time, and
the old activity attempt keeps its activity slot on the SDK worker.

**The Go and Java SDKs can stop the old activity attempt on their own**, in different ways:

- **The Go SDK retries the failed heartbeat call until the batching window ends or 10 seconds have
  passed, whichever comes first, then cancels the activity's context.** The 10 seconds is the
  default of the client option `RPCTimeout`, from Go SDK v1.51.0. Go SDK releases up to v1.50.0
  retry until half the batching window has passed or 10 seconds, whichever comes first, and the 10
  seconds cannot be changed. When it cancels the context, the Go SDK logs the message
  `RecordActivityHeartbeat with error` at log level WARN. Every Go SDK release does this.
- **The Java SDK retries the failed heartbeat call once a second, and from Java SDK v1.34.0 it stops
  the activity at the activity's first heartbeat call after the heartbeat timeout plus 5 seconds
  have passed without a successful heartbeat call:**
  - **For each failed heartbeat call, the Java SDK logs the message `Heartbeat failed` at log level
    WARN**, and the activity code is not told about the failures.
  - **At that first heartbeat call, the Java SDK logs the message `Activity heartbeat timed out
    locally` at log level WARN, and the heartbeat call throws an `ActivityCanceledException`.**
    Activity code that has stopped calling the heartbeat API gets neither, and keeps running.
  - **Older Java SDK releases keep retrying the heartbeat call and never stop the activity.**

When a heartbeat call does get through after the server has timed out the activity attempt, the
server answers `NOT_FOUND`. The Go SDK then cancels the activity's context, and the Java SDK throws
an exception from the activity's heartbeat call, the same as when the heartbeat calls keep failing.

> **Recommendation:** write the activity code so that it ends the activity when the SDK signals it
> to stop. Without that, the activity keeps running, and keeps its activity slot on the SDK worker,
> until it finishes:
>
> - **Go SDK.** The Go SDK cancels the activity's context, the standard Go way to tell code to
>   stop. Check the context after each heartbeat call and, when it is cancelled, do any cleanup
>   and return, as the sample at the top of this playbook does.
> - **Java SDK.** The heartbeat call throws an exception, always a subclass of
>   `ActivityCompletionException`. Catch it only to log or clean up, then rethrow it, so the
>   activity stops. Use Java SDK v1.34.0 or later, because older releases never throw it while the
>   heartbeat calls keep failing.
> - **Java SDK, the `temporal.activity.localTimeoutBufferMs` system property.** Do not set it.
>   Without it, the Java SDK waits 5 seconds past the heartbeat timeout before it stops the
>   activity, so that the server times out the activity attempt first.

A heartbeat call can also reach the server late without failing at all, when the SDK worker is
short of CPU, or, on the Java SDK, when the SDK worker cannot send heartbeats for all its
activities fast enough.

### 1.3 Why an SDK worker can send heartbeats late

**An SDK worker can send heartbeats to the server late, even when the activity code calls the
heartbeat API often.** When no heartbeat reaches the server within the heartbeat timeout, the
server times out the activity attempt, even though the activity is still working. An SDK worker
can send heartbeats late for either of these reasons, or for both at the same time:

- **If the SDK worker is short of CPU for the activities it runs at the same time.** This can
  happen on the Go and Java SDKs.
- **On the Java SDK, if heartbeat calls take a long time.** The SDK worker then cannot send
  heartbeats for all its activities fast enough.

To understand these two reasons, first look at how the Go and Java SDKs send heartbeats.

**The Go and Java SDKs send heartbeats in different ways**, and that decides how late an
SDK worker sends them:

- <a id="go-sdk-heartbeat-goroutines"></a>**The Go SDK sends each activity's heartbeats separately from every other activity's.** A
  heartbeat call made after a quiet period is sent at once, inside the activity's heartbeat call.
  A heartbeat call made during the batching window is sent when the window ends, by a new goroutine
  that the Go SDK starts for that one activity. No two activities share a goroutine for sending
  heartbeats, so one activity's heartbeat calls do not depend on any other activity's. When one
  activity's heartbeat call takes a long time, the heartbeats of the other activities on the SDK
  worker do not wait for it.
- <a id="java-sdk-heartbeat-threads"></a>**The Java SDK shares 4 threads among all activities on the SDK worker for sending
  heartbeats.** A heartbeat call made after a quiet period is sent at once, on the activity's own
  thread. A heartbeat call made during the batching window is sent when the window ends, by one of
  4 threads that every activity on the SDK worker shares. The 4 threads are named
  `TemporalActivityHeartbeat-<namespace>-<taskQueue>`, and no worker option changes the number of
  heartbeat threads. Each heartbeat call sent at the end of a batching window keeps one of the 4
  threads busy until the server answers it, or for up to 10 seconds, the Java SDK's default timeout
  for every call to the server (`WorkflowServiceStubsOptions.Builder.setRpcTimeout`). Because the
  4 threads are shared, one activity's heartbeat calls depend on the other activities' heartbeat
  calls. When many activities have heartbeats due at the end of a batching window and heartbeat
  calls take a long time, all 4 threads can be busy at once, and the heartbeats of the other
  activities wait until one of the 4 threads is free.

**Tests on a test cluster show how a CPU shortage and slow heartbeat calls make an SDK worker
send heartbeats late.** The tests ran with this setup:

- Java SDK v1.40.0 and Go SDK v1.48.0.
- A 10-second heartbeat timeout on every activity, so an 8-second batching window.
- 200 activities started at the same time, on one SDK worker that ran only those activities.
- Room for 200 activities at the same time on the SDK worker, and 5 activity pollers. The Go SDK
  uses 2 activity pollers by default and the Java SDK uses 5, so the tests set both to 5.
- The Java SDK ran each test twice. Once on platform threads, the ordinary Java threads that the
  Java SDK uses by default. Once on Java virtual threads, which the worker option
  `WorkerOptions.Builder.setUsingVirtualThreadsOnActivityWorker` turns on. The 4 heartbeat threads
  are platform threads either way.
- Between heartbeat calls, each activity did 1 millisecond of CPU work. In the two 14-CPU rows of
  the table below, each activity also waited 100 milliseconds, the way an activity that waits on a
  network call does.

**The server did not cause any of the timeouts.** During every test run:

- The server rejected no heartbeat calls and throttled nothing.
- The p99 latency of each kind of database call was less than 100 milliseconds. The highest was
  for the database call that starts a workflow, not for a heartbeat.
- The p99 latency of heartbeat calls on the server was less than 80 milliseconds.

**Each test started 200 activities, and the table below counts the activity attempts that timed
out** for each SDK worker setup:

| SDK worker setup | Activity attempts that timed out, Java SDK, platform threads | Activity attempts that timed out, Java SDK, virtual threads | Activity attempts that timed out, Go SDK |
|---|---|---|---|
| 14 CPUs, heartbeat calls completed at normal speed | 0 of 200 | 0 of 200 | 0 of 200 |
| Limited to 1 CPU | **200 of 200** | **200 of 200** | **152 of 200** |
| 14 CPUs, every heartbeat call took 300 milliseconds to complete | **173 of 200** | **174 of 200** | 0 of 200 |

What each row of the table above shows:

- **14 CPUs, heartbeat calls completed at normal speed.** No activity attempt timed out on either
  SDK. Each SDK worker used about 3 of its 14 CPUs.
- **Limited to 1 CPU.** On both SDKs, most activity attempts timed out.
  - The Java SDK worker on platform threads picked up all 200 activities within 1 second. All 200
    activity attempts timed out, on platform threads and on virtual threads.
  - The Go SDK worker took up to 60 seconds to pick up an activity, and never ran more than 82
    activities at the same time. 152 activity attempts timed out.
- **14 CPUs, every heartbeat call took 300 milliseconds to complete.** The 300 milliseconds were
  added inside the SDK worker, to stand in for a slow network or server.
  - On the Java SDK, 173 activity attempts timed out on platform threads and 174 on virtual
    threads. All activities share the Java SDK's
    [4 heartbeat threads](#java-sdk-heartbeat-threads), and each heartbeat call keeps a thread
    busy for 300 milliseconds:
    - The 4 heartbeat threads can send about 13 heartbeats a second (4 ÷ 0.3).
    - The 200 activities need about 25 heartbeats a second, one per activity at the end of each
      8-second batching window (200 ÷ 8).
    - Heartbeats come due about twice as fast as the 4 heartbeat threads can send them. The
      heartbeats waiting for a free thread pile up, each heartbeat waits longer than the one
      before, and the waits grow until they pass the 10-second heartbeat timeout.
  - On the Go SDK, no activity attempt timed out. The Go SDK sends each activity's heartbeats on
    [a goroutine of the activity's own](#go-sdk-heartbeat-goroutines), so the 300 milliseconds
    slowed only each activity's own heartbeat calls, and no activity waited for another
    activity's heartbeat call.

> <a id="rec-sdk-worker-cpu"></a>**Recommendation:**
>
> - **Give each SDK worker enough CPU for the activities it runs at the same time**, or lower that
>   number with `worker.Options.MaxConcurrentActivityExecutionSize` in the Go SDK (1,000 by
>   default) or `WorkerOptions.Builder.setMaxConcurrentActivityExecutionSize` in the Java SDK (200
>   by default).
> - **On the Java SDK, keep the number of activities per SDK worker low enough for the 4 heartbeat
>   threads.** When heartbeat calls can take a long time, lower the number of activities each SDK worker
>   runs at the same time, or spread the activities over more SDK workers.

**A second test, on the Java SDK only, kept the SDK worker setup the same and changed what the
activities do.** The two tests differ in what changes from one run to the next:

- **The first test changed the SDK worker setup**, the number of CPUs and how long each heartbeat
  call takes. Every activity did the same work.
- **The second test kept one SDK worker setup**, limited to 1 CPU, **and changed what the
  activities do.** Some activities were CPU-bound, spending most of their time using the CPU.
  Others were I/O-bound, spending most of their time waiting on the network or disk.

Whether virtual threads help depends on whether the activities are CPU-bound or I/O-bound, so the
second test ran each kind of activity once on platform threads and once on virtual threads. Java virtual threads on the SDK
worker:

- Were added in Java SDK v1.27.0, and need Java 21 or later. The worker option
  `WorkerOptions.Builder.setUsingVirtualThreadsOnActivityWorker` turns them on.
- Run on a few platform threads that all the virtual threads share, one per CPU by default. A
  virtual thread uses one of these platform threads until the activity code waits, for example on a
  network call. Only then can another activity's virtual thread use that platform thread.
- Can keep a platform thread for one activity for a long time, when the activity code does not
  wait. On an SDK worker with 1 CPU there is only one such platform thread, so the other activities
  could not run, or call the heartbeat API, until that activity waits or finishes.
- Run only the activity code. Turning on virtual threads does not change the
  [4 heartbeat threads](#java-sdk-heartbeat-threads). They stay platform threads, and still send
  every heartbeat that is due at the end of a batching window.

The second test ran with this setup:

- The same setup as the test above, including the 10-second heartbeat timeout, on an SDK worker
  limited to 1 CPU, where the activities compete for the CPU.
- Three kinds of activity, each run once on platform threads and once on virtual threads. The
  table below lists the work each kind did between heartbeat calls.
- The server again did not cause any of the timeouts. It rejected no heartbeat calls and throttled
  nothing, the p99 latency of each kind of database call was less than 50 milliseconds, and the
  p99 latency of heartbeat calls on the server was less than 20 milliseconds.

**Each test started 200 activities, and the table below counts the activity attempts that timed
out** for each kind of activity:

| Kind of activity | CPU work between heartbeat calls | Wait on the network or disk between heartbeat calls | CPU the SDK worker used | Activity attempts that timed out, platform threads | Activity attempts that timed out, virtual threads |
|---|---|---|---|---|---|
| CPU-bound | 1 millisecond | None | All of its 1 CPU | 200 of 200 | 200 of 200 |
| I/O-bound, little CPU | 1 millisecond | 500 milliseconds | About half of its 1 CPU | 0 of 200 | 0 of 200 |
| I/O-bound, more CPU | 5 milliseconds | 500 milliseconds | All of its 1 CPU | **192 of 200** | **0 of 200** |

What each row of the table above shows:

- **CPU-bound.** All activity attempts timed out on both kinds of thread. On virtual threads the
  test took 9 minutes to finish, against 23 seconds on platform threads, because each activity kept
  the one platform thread for a long time while the other activities waited for it. One activity
  went 56 seconds between its own heartbeat calls.
- **I/O-bound, little CPU.** No activity attempt timed out on either kind of thread. The 200
  activities needed less CPU than the SDK worker had.
- **I/O-bound, more CPU.** The 200 activities needed more CPU than the SDK worker had.
  - On platform threads, 192 activity attempts timed out. The 4 heartbeat threads competed for the
    CPU with 200 activity threads, and sent 202 of the 1,400 heartbeats that came due.
  - On virtual threads, no activity attempt timed out. The activity code ran on 1 platform thread,
    so the 4 heartbeat threads competed for the CPU with 1 thread instead of 200, and sent 1,396
    heartbeats.

> **Recommendation:**
>
> - **On the Java SDK, consider virtual threads for I/O-bound activities on an SDK worker that is
>   short of CPU.** Virtual threads do not replace giving the SDK worker enough CPU.
> - **Do not run CPU-bound activities on virtual threads.** Virtual threads did not stop the
>   timeouts, and the activities took much longer to finish.
> - **Do not expect virtual threads to help when heartbeat calls take a long time.** The 4
>   heartbeat threads stay platform threads.
> - **Run CPU-bound activities on a different task queue from I/O-bound activities that run on
>   virtual threads.** The worker option applies to every activity on the SDK worker.

**How the SDK sends heartbeats is now covered.** [Section
2](#2-how-the-server-receives-and-records-a-heartbeat) follows a heartbeat call through the server, to
the moment the heartbeat counts as recorded.

---

## 2. How the server receives and records a heartbeat

**Each heartbeat is one `RecordActivityTaskHeartbeat` API call.** The call carries:

- **The activity's task token**, which tells the server which activity the heartbeat is for. The
  server sends the task token to the SDK worker with the activity task, and the SDK adds it to each
  heartbeat call. The activity code does not set it.
- **The heartbeat details**, only if the activity code passes them to the heartbeat call. Heartbeat
  details are optional. They are a payload, the same as a workflow's input or an activity's input.

**On the server, a heartbeat call goes through six steps, one after another:**

1. **The frontend service checks the gRPC message size limit**, a limit on the size of the whole
   call ([2.1](#21-the-grpc-message-size-limit)).
2. **The frontend service checks the namespace's rate limit**
   ([2.2](#22-the-namespace-rate-limit-in-the-frontend-service)).
3. **The frontend service checks the payload size limit**, a limit on the size of the heartbeat
   details ([2.3](#23-the-payload-size-limit)).
4. **The history service instance that owns the workflow's shard checks its requests-per-second
   limits** ([2.4](#24-the-history-services-requests-per-second-limits)). Each workflow belongs to
   one shard, and each shard is owned by one history service instance at a time. The frontend
   service sends the heartbeat call to that instance.
5. **That history service instance waits for the workflow's lock and records the heartbeat's time**
   ([2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late)).
6. **That history service instance checks its persistence QPS limits and writes the heartbeat to the
   database** ([2.6](#26-what-a-heartbeat-writes-to-the-database-and-the-persistence-qps-limits)).
   The persistence QPS limits cap how many database calls a second the history service makes. If a
   limit refuses the write, or the write fails, the heartbeat is not recorded
   ([2.7](#27-when-the-database-write-of-a-heartbeat-fails)).

**A heartbeat counts only once the history service has saved it in the workflow's mutable state in
the database.** Mutable state is the server's current record of a workflow run, kept in the
database, with a record of each running activity. A heartbeat sets two fields in the activity's
record:

- **The last heartbeat time**, taken from the history service's clock in step 5, once the heartbeat
  call has the workflow lock.
- **The heartbeat details**, if the call carries any.

The history service then saves the mutable state to the database in step 6. If that save fails, the
new heartbeat time and details are lost ([2.7](#27-when-the-database-write-of-a-heartbeat-fails)).
No event is added to the workflow's event history.

**The server times out the activity when the last heartbeat time is older than the heartbeat
timeout.** The server checks only the last heartbeat time saved in mutable state. A heartbeat call
still going through the server's checks, or refused at one of them, does not change the last
heartbeat time.

**A heartbeat timeout shows that the heartbeats were late, but not why.** The server shows a
heartbeat timeout in one of two places:

- **When the heartbeat timeout leads to a retry of the activity**, the timeout shows as the
  activity's last failure under the workflow's pending activities, in the Temporal UI and in the
  Temporal CLI command `temporal workflow describe`. The failure has timeout type `Heartbeat`. No
  event is added to the event history.
- **When the heartbeat timeout ends the activity**, because it happened on the last attempt that the
  activity's retry policy allows, the event history gets an `ActivityTaskTimedOut` event with
  timeout type `Heartbeat`.

**This playbook shows how to watch for heartbeat timeouts and find their cause on the dashboards.**
The Temporal Server dashboard counts heartbeat timeouts in **[Activity Heartbeat Timeout (Every
Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)**,
retries included, and in **[Activity Heartbeat Timeout (Final
Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)**, only
the timeouts that ended the activity.

**Several different causes end in the same heartbeat timeout, and each cause shows on a different panel:**

- **The server refused the heartbeat call**, at one of the server's checks, or the call's
  database write failed. The Temporal Server dashboard shows most refusals in **[Heartbeat Calls
  Refused by
  Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**.
  Each subsection below names the panel for its step.
- **The heartbeat call was slow on the server.** **[Heartbeat Call Latency (Frontend and
  History)](../observability/dashboards/server/temporal-server-readme.md#4-service-latencies)**
  shows whether the time was spent in the frontend service or in the history service.
- **The heartbeat call failed before it reached the server, or the server refused it.** The SDK
  dashboards show every failed heartbeat call in **Request Failures**, by status code
  ([Go](../observability/dashboards/sdk/temporal-sdk-go-readme.md#1-grpc-requests-and-request-latencies),
  [Java
  Micrometer](../observability/dashboards/sdk/temporal-sdk-java-micrometer-readme.md#1-grpc-requests-and-request-latencies),
  [Java
  OpenTelemetry](../observability/dashboards/sdk/temporal-sdk-java-otel-readme.md#1-grpc-requests-and-request-latencies)).
- **The SDK worker sent the heartbeat call too late, or stopped sending it**, for example because
  the SDK worker was busy or crashed ([1.3](#13-why-an-sdk-worker-can-send-heartbeats-late)). A
  crashed SDK worker's own metrics stop with it. The server still shows fewer heartbeat calls in
  **[Activity Heartbeat
  Calls](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**.

**[Section 5](#5-how-to-find-where-the-heartbeats-were-delayed) goes through the panel for each cause,
one at a time, to find which cause made the heartbeats late.** The subsections below follow a heartbeat call through the server, in the
order the server checks it, starting with the gRPC message size limit in
[2.1](#21-the-grpc-message-size-limit).

### 2.1 The gRPC message size limit

**The gRPC message size limit refuses a heartbeat call over 4 MB.** The refusal happens before the
frontend service handles the call. 4 MB is the gRPC default for any call the frontend service
receives. The limit covers the whole call, including the heartbeat details. The heartbeat call
fails with `RESOURCE_EXHAUSTED`, and the heartbeat is not recorded.

**On Go SDK v1.43.0 and later and Java SDK v1.41.0 and later, a heartbeat call over 4 MB is not
sent at all.** Those SDKs check the heartbeat details against the payload size limit, 2 MB by
default, before sending the heartbeat call. When the details are over the limit, the SDK fails the
activity attempt instead ([2.3](#23-the-payload-size-limit)). The gRPC message size limit refuses a
heartbeat call only when the SDK does not check the heartbeat details:

- on Go SDK versions before v1.43.0 and Java SDK versions before v1.41.0;
- with servers before v1.30.0;
- with the SDK's payload size check turned off. The SDK's payload size check is on by default. The
  application code turns it off in the options it creates the SDK worker with, with
  `DisablePayloadErrorLimit` in the Go SDK's `worker.Options`, or with
  `setDisablePayloadErrorLimit(true)` on the Java SDK's `WorkerOptions` builder.

**No Temporal Server metric counts the refused call.** The SDK counts it, and it shows on the SDK
dashboards in **Request Failures**, with operation `RecordActivityTaskHeartbeat` and status code
`RESOURCE_EXHAUSTED`
([Go](../observability/dashboards/sdk/temporal-sdk-go-readme.md#1-grpc-requests-and-request-latencies),
[Java Micrometer](../observability/dashboards/sdk/temporal-sdk-java-micrometer-readme.md#1-grpc-requests-and-request-latencies),
[Java OpenTelemetry](../observability/dashboards/sdk/temporal-sdk-java-otel-readme.md#1-grpc-requests-and-request-latencies)).

The gRPC message size limit is not a Temporal setting, no setting changes it, and it applies to
every namespace the same way.

> <a id="rec-small-heartbeat-details"></a>**Recommendation:** keep heartbeat details small, far below 4 MB. Put in them only what the next
> activity attempt needs to continue the work after a heartbeat timeout, such as the index of the
> last item done. Keep large data in your own store, and put only a reference to it in the
> heartbeat details.

A heartbeat call within the gRPC message size limit then counts against the namespace's rate limit,
which [2.2](#22-the-namespace-rate-limit-in-the-frontend-service) covers.

### 2.2 The namespace rate limit in the frontend service

**The frontend service checks the namespace's request rate limit.** Heartbeat calls count against
the same per-namespace limit as every other API call. Two server dynamic config settings set it:

- `frontend.namespaceRPS`, 2,400 calls a second per frontend service instance by default.
- `frontend.globalNamespaceRPS`, a limit for the whole cluster. It replaces the per-instance limit
  when it is set, and it is 0, not set, by default.

**Heartbeat calls have a high priority within the namespace's rate limit**, the same as workflow
starts, signals, updates and task completions. When the namespace is near its limit, the frontend
service refuses calls with a lower priority first, such as the SDK workers' polls for tasks, and
refuses heartbeat calls only after them.

A refused heartbeat call fails with `RESOURCE_EXHAUSTED`, cause `RpsLimit`, and the SDK retries it
as [1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails) describes. Refused heartbeat calls show
on the Temporal Server dashboard in **[Heartbeat Calls Refused by
Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**.

> **Recommendation:** watch **[Activity Heartbeat
> Calls](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**
> for a rise you do not expect, for example after a change that shortened the heartbeat timeouts.
> Each running activity that heartbeats sends about one heartbeat call per batching window
> ([1.1](#11-the-batching-window)), so shorter heartbeat timeouts mean more heartbeat calls.
> Because heartbeat calls have a high priority, the namespace's lower-priority calls, such as the
> SDK workers' polls for tasks, could be refused. After shortening heartbeat timeouts, also watch
> **[Resource Exhausted with
> Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**
> for `RpsLimit` refusals by the frontend service, by operation.

> <a id="rec-namespace-rate-limit"></a>**Recommendation:** when the frontend service refuses heartbeat calls, find out which limit
> refused them before changing anything. **[Heartbeat Calls Refused by Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**
> shows one line for each service and cause, labelled with the service that refused the heartbeat
> calls and the `RESOURCE_EXHAUSTED` cause. For example, `frontend - RpsLimit` means that the
> frontend service refused heartbeat calls with cause `RpsLimit`. The frontend service also counts
> the refusals that the history service made and that the frontend service passed on to the SDK.
> The `frontend - RpsLimit` line can therefore show refusals by either limit:
>
> - **Only the `frontend - RpsLimit` line shows refusals** means that the namespace's rate limit
>   refused the heartbeat calls.
> - **The `frontend - RpsLimit` and `history - RpsLimit` lines show refusals at the same time** means
>   that the history service refused the heartbeat calls, and the frontend service passed the
>   refusals on. The history service's limits are `history.rps` and, on server v1.32.0 and later,
>   `history.namespaceRPS`. The recommendation in
>   [2.4](#24-the-history-services-requests-per-second-limits) covers both settings. Afterwards,
>   check the panel again. Refusals left only on the `frontend - RpsLimit` line come from the
>   namespace's rate limit.
>
> When the namespace's rate limit refused the heartbeat calls:
>
> - **Raise `frontend.namespaceRPS`, or `frontend.globalNamespaceRPS` when the cluster sets it**,
>   for the namespace.
> - **Or lower the namespace's calls.** Every call of the namespace counts against the limit, the
>   heartbeat calls among them. Each running activity sends about one heartbeat call per batching
>   window ([1.1](#11-the-batching-window)), so fewer activities running at the same time, or a
>   longer heartbeat timeout, means fewer heartbeat calls.
>   [2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late) covers what a longer
>   heartbeat timeout costs.

A heartbeat call within the namespace's rate limit then has its heartbeat details checked against
the payload size limit, which [2.3](#23-the-payload-size-limit) covers.

### 2.3 The payload size limit

**The payload size limit is the server's dynamic config setting `limit.blobSize.error`, 2 MB by
default.** It applies to the heartbeat details, and is the same limit that applies to workflow
input, activity input and signals. `limit.blobSize.error` is set per namespace.

**On Go SDK v1.43.0 and later and Java SDK v1.41.0 and later, the SDK checks the size of the
heartbeat details before sending the heartbeat call.** The SDK learns the namespace's
`limit.blobSize.error` when the SDK worker starts, from the server's answer to the SDK worker's
`DescribeNamespace` API call. Servers before v1.30.0 do not include the limit in that answer, so
with those servers the SDK does not check. When the heartbeat details are over the limit:

- **The SDK does not send the heartbeat call.** Instead, the SDK reports the activity attempt to the
  server as failed, with a `RespondActivityTaskFailed` API call. The failure message is `[TMPRL1103]
  Attempted to upload payloads with size that exceeded the error limit.`, and the Temporal UI shows
  it as the activity's last failure. The failure message is the same when the activity's result is
  over the limit, so only the SDK worker's log line shows that the heartbeat details were the cause:
  - **The Go SDK** logs the message `RecordActivityHeartbeat with error` at log level WARN, with the
    failure message as the error. An activity result over the limit logs the message
    `Activity task postprocess error: [TMPRL1103] …` at log level ERROR instead.
  - **The Java SDK** logs the failure message at log level ERROR, with the size, the limit and
    `path=details`. An activity result over the limit logs the same message with `path=result`.
- **The server handles that failure like any other activity failure.** The failure is not marked as
  non-retryable, so the server retries the activity as its retry policy allows. Each attempt fails
  the same way while the heartbeat details stay over the limit.
- **The SDK cancels the running activity, so that the activity code stops.** In the Go SDK, the
  activity's context is cancelled. In the Java SDK, the heartbeat call throws
  `ActivityCanceledException`, right away when the SDK sent the heartbeat at once, or on the next
  heartbeat call when the SDK held the details until the end of the batching window
  ([1.1](#11-the-batching-window)). Activity code that ignores the cancellation keeps running until
  it returns, and the SDK does not report what it returns, because the attempt is already reported
  as failed.
- **The server never receives the heartbeat call**, so **Blob Size Errors** does not count it. The
  SDKs count it differently in **Activity Execution Failed**:
  - **The Java SDK** counts it on the Java SDK dashboards ([Java
    Micrometer](../observability/dashboards/sdk/temporal-sdk-java-micrometer-readme.md#7-activity-task-info),
    [Java
    OpenTelemetry](../observability/dashboards/sdk/temporal-sdk-java-otel-readme.md#7-activity-task-info)),
    with failure reason `PayloadsTooLarge`, together with activity results over the limit.
  - **The Go SDK** counts only activity results over the limit there, on the [Go SDK
    dashboard](../observability/dashboards/sdk/temporal-sdk-go-readme.md#7-activity-task-info), with failure
    reason `PayloadsTooLarge`. The Go SDK does not count heartbeat details over the limit under any
    failure reason. Only the log message `RecordActivityHeartbeat with error` at log level WARN
    shows them.

> **Recommendation:** to catch heartbeat details over the limit, alert on the SDK workers' logs.
> With the SDK's check on, the server never receives the heartbeat call, so the Temporal Server
> dashboard does not show it, and the Go SDK does not count it in any metric. Look for:
>
> - **On the Go SDK**, the log message `RecordActivityHeartbeat with error` at log level WARN, with
>   an error that contains `[TMPRL1103]`. The Go SDK also logs that message for other heartbeat call
>   errors, so match on the log message and on `[TMPRL1103]` together. The log line includes the
>   workflow ID, the activity type and the attempt.
> - **On the Java SDK**, a log line at log level ERROR that contains `[TMPRL1103]` and
>   `path=details`. **Activity Execution Failed** with failure reason `PayloadsTooLarge` also counts
>   it, but counts activity results over the limit too.

> **Recommendation:** to stop activities whose attempts keep failing because their heartbeat details
> are over the limit, deploy a fix to the activity code that makes the heartbeat details smaller. The
> SDK reports the failure as retryable so that, once the fix is deployed, the next activity attempt
> continues the work. Until then:
>
> - **Each activity attempt starts the activity code again and fails at its first heartbeat call**,
>   at the pace of the activity's retry policy. The server's default retry policy waits 1 second
>   before the first retry, doubles the wait each time up to 100 seconds, and has no limit on
>   attempts.
> - **To stop the attempts until the fix is deployed, pause the activities** with the Temporal CLI
>   command `temporal activity pause`. The server then starts no new attempt of a paused activity.
>   There is no batch pause, so each activity is paused on its own. After the fix is deployed, the
>   Temporal CLI command `temporal activity unpause --query` unpauses the activities of all matching
>   workflows in one batch. A pause does not stop the activity's schedule-to-close timeout, so a
>   paused activity can still time out.

**With the SDK's check turned off, the frontend service checks the heartbeat details instead.** The
worker option `DisablePayloadErrorLimit` in the Go SDK, and `setDisablePayloadErrorLimit` in the
Java SDK, turns the SDK's check off. The frontend service also checks the heartbeat details on Go
SDK versions before v1.43.0, on Java SDK versions before v1.41.0, and on servers before v1.30.0.
When the heartbeat details are over the limit, the frontend service fails the activity at once:

- **The frontend service reports the activity attempt to the history service as failed, and the
  heartbeat is not recorded.** The failure message is "Heartbeat details exceed size limit.", and
  the Temporal UI shows it on the activity's `ActivityTaskFailed` event. The failure is marked as
  non-retryable, so the server does not retry the activity, whatever its retry policy allows.
- **The heartbeat call does not fail. Instead, the frontend service answers the heartbeat call with
  a request to cancel the activity, and the SDK cancels the running activity.** In the Go SDK, the
  activity's context is cancelled. In the Java SDK, a heartbeat call throws
  `ActivityCanceledException`.
- **The frontend service logs the message `Blob data size exceeds the warning limit.` at log level
  WARN**, with the workflow ID, the size of the heartbeat details and the operation
  `RecordActivityTaskHeartbeat`. The frontend service limits how often it logs that message, so not
  every heartbeat call over the limit is logged.
- **Each heartbeat call over the limit adds to [Blob Size
  Errors](../observability/dashboards/server/temporal-server-readme.md#11-workflow-stats) on the
  Temporal Server dashboard.** That panel also counts workflow input, activity input and signals
  over the limit, so the panel alone does not show that the heartbeat details were too large.

> **Recommendation:** watch the Temporal Server dashboard for activities that fail with no retry
> left. A heartbeat call over the limit ends the activity, so the activity shows in **Activity
> Failed (Final Attempt)** *(panel to come)*, by activity type. That panel counts every activity
> that fails with no retry left, not only those whose heartbeat details were over the limit. The
> failure message on the `ActivityTaskFailed` event shows the cause.

> **Recommendation:** keep heartbeat details to the minimum the next activity attempt needs, as
> [2.1](#21-the-grpc-message-size-limit) recommends, and leave `limit.blobSize.error` at its
> default. The setting applies to every payload in the namespace, including workflow input,
> activity input and signals, so changing it for heartbeat details changes it for all of them. A
> heartbeat call over 4 MB is also refused by the gRPC message size limit
> ([2.1](#21-the-grpc-message-size-limit)) whatever the setting.

A heartbeat call within the payload size limit goes on to the history service.

### 2.4 The history service's requests-per-second limits

**Before a heartbeat call waits for the workflow lock, the history service checks its
requests-per-second (RPS) limits, `history.rps` and `history.namespaceRPS`.** Each history service
instance limits how many calls a second it accepts. The two server dynamic config settings set those
limits:

- `history.rps`, 3,000 calls a second per history service instance by default. All namespaces
  together count against `history.rps`.
- `history.namespaceRPS`, a limit for each namespace on each history service instance, on server
  v1.32.0 and later. `history.namespaceRPS` is 0, not set, by default, and each namespace can then
  use up to the `history.rps` limit.

**Heartbeat calls have a high priority within the history service's RPS limits**, the same as
workflow starts, signals, updates and task completions. The calls the server makes for its own
background work have a lower priority. When a history service instance is near `history.rps`, the
history service refuses those background calls first, and refuses heartbeat calls only after them.

**A heartbeat call that the history service refuses fails with `RESOURCE_EXHAUSTED`, cause
`RpsLimit`:**

- **Refused by `history.rps`:** the frontend service waits 1 second and tries the call once more, if
  the call's deadline leaves time for that wait. The call's deadline is the time limit that the SDK
  sets on every heartbeat call
  ([2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late)). If the call is refused
  again, or there is no time to wait, the SDK gets `RESOURCE_EXHAUSTED` and retries the heartbeat
  call as [1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails) describes.
  - **From server v1.32.0, the server dynamic config setting
    `system.retryUnboundedOnSystemResourceExhausted` changes that single retry.** The setting is off
    by default. When it is on, the frontend service keeps retrying the call, waiting longer each
    time, until the call's deadline. A refused heartbeat call then uses up more of its deadline
    before the SDK learns of the refusal. The setting applies to every call between Temporal
    services that a system-wide limit refuses, not only to heartbeat calls.
- **Refused by `history.namespaceRPS`:** the frontend service passes the refusal straight back to
  the SDK.

**Refusals by the history service show in
[Heartbeat Calls Refused by Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)
under the history service.** Each of those refusals also shows a second time on the same panel,
under the frontend service, because the frontend service counts every refusal it sends back to the
SDK, including refusals that came from the history service. The panel shows one line for each
service and cause, labelled for example `history - RpsLimit` and `frontend - RpsLimit`. So on that
panel:

- **When the `history - RpsLimit` line and the `frontend - RpsLimit` line show refusals at the same
  time**, the history service refused the heartbeat calls, and the frontend service only passed the
  refusals on.
- **When only the `frontend - RpsLimit` line shows refusals**, the frontend service refused the
  heartbeat calls itself, because the namespace was over its rate limit
  ([2.2](#22-the-namespace-rate-limit-in-the-frontend-service)).

> <a id="rec-history-rps"></a>**Recommendation:** when the `history - RpsLimit` line on **[Heartbeat Calls Refused by
> Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)** shows refusals:
>
> - **Check [Resource Exhausted with
>   Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits) for other
>   operations refused by the history service.** `history.rps` counts the calls of every namespace
>   together, so heartbeat calls from one namespace can use up the limit for every namespace whose
>   workflows are on the same history service instance.
> - **Check how many heartbeat calls the namespace sends, on [Activity Heartbeat
>   Calls](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits).** Each running
>   activity sends about one heartbeat call per batching window ([1.1](#11-the-batching-window)),
>   so many running activities with short heartbeat timeouts mean many heartbeat calls.
> - **On server v1.32.0 and later, when several namespaces share the cluster, set
>   `history.namespaceRPS` below `history.rps` for a namespace with many running activities that
>   heartbeat.** The namespace then cannot use up `history.rps` for the others. The history service
>   refuses that namespace's calls sooner, and the frontend service passes those refusals straight
>   back to the SDK without trying again.
> - **After setting `history.namespaceRPS`, watch that namespace's `history - RpsLimit` line on
>   [Heartbeat Calls Refused by
>   Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits).** Refusals there
>   mean the value is too low for the namespace's own heartbeat calls, and its activities can time
>   out.

A heartbeat call within the history service's rate limit then waits for the workflow's lock.

### 2.5 How waiting for the workflow lock can make heartbeats late

**The history service lets only one API call or history task at a time read or update a workflow
run's mutable state.** Each workflow run has one **workflow lock**, and every API call and every
history task that reads or updates the run's mutable state holds that lock while it does:

- **History tasks are the work the history service saves with a workflow run and processes
  later.** Examples are the timer tasks that check an activity's deadlines, and the transfer tasks
  that pass an activity to the matching service.
  [History Task Processing](./history-task-processing-tuning.md#2-how-task-processing-works)
  explains how the history service processes them.
- **Heartbeat calls get the workflow lock before the workflow run's history tasks.** A heartbeat
  call holds the lock to save the heartbeat's time and details in the mutable state.
  - **When the lock is freed and both a heartbeat call and a history task of the same workflow run
    are waiting for it**, the history service gives the lock to the heartbeat call first. The same
    holds for every other API call, such as signals and task completions.
  - **A history task that already holds the lock keeps it** until the task finishes.
  - **Many heartbeat calls on one workflow run can therefore keep its history tasks waiting.** Those
    tasks give up after half a second and are tried again later.

**Many running activities on one workflow run also make their heartbeat calls wait for each
other.** Every heartbeat call for the run's activities needs the same workflow lock, and holds it
until its database write is done. So the run's mutable state is written one call at a time, and a
slow database caps one workflow run at about one write per database write time, however many
activities the run has.

> **Test cluster result:** heartbeat calls on one workflow run waited for each other at its
> workflow lock, and the workflow ran slower.
>
> - **Goal:** reproduce many activities heartbeating at the same time on one workflow run, with a
>   database made slower on purpose, to see how long heartbeat calls wait for the workflow lock.
>   The numbers show this scenario only, not normal server speed.
> - **Setup:** server v1.31.0, 300 activities heartbeating on one workflow, and 50 milliseconds
>   added on purpose to every database write.
> - **The workflow's writes held at 17 to 19 a second**, one at a time, about 55 milliseconds
>   each.
> - **Heartbeat calls took up to 1.55 seconds in the history service at p99**, against 84
>   milliseconds at the median. **[Heartbeat Call Latency (Frontend and
>   History)](../observability/dashboards/server/temporal-server-readme.md#4-service-latencies)** and
>   **[Workflow Lock
>   Latency](../observability/dashboards/server/temporal-server-readme.md#2-shard-and-workflow-lock-latencies)**
>   show this wait.
> - **No heartbeat timed out, because the heartbeat calls stayed within the time the batching window
>   leaves.** With a 10-second heartbeat timeout, the batching window is 8 seconds, and the SDK
>   starts it only when the previous heartbeat call returns ([1.1](#11-the-batching-window)). So the
>   time between two recorded heartbeats was about 8 seconds plus how long the next heartbeat call
>   took, at most about 1.55 seconds here, still under 10 seconds.
> - **The workflow took 4 minutes 25 seconds instead of 1 minute, with no errors.** Its history
>   tasks kept losing the workflow lock to the heartbeat calls, so the work those tasks do, such as
>   checking the activities' deadlines, waited.

> <a id="rec-fewer-heartbeating-activities-per-run"></a>**Recommendation:** keep down the number of activities that heartbeat at the same time on one
> workflow run. Every heartbeat call for a workflow run's activities waits for that run's one lock.
> There are two ways to do this:
>
> - **Run fewer of the activities at the same time.** For example, instead of starting 100
>   activities at once, start 50, and start the next 50 when those finish.
> - **When the activities must run at the same time, spread them over several workflow runs**, for
>   example child workflows, so each run's lock serves fewer heartbeat calls.
>
> As the test cluster result above shows, the heartbeat calls waited for each other, and the
> workflow took 4 minutes 25 seconds instead of 1 minute.

**The history service records the heartbeat's time in its in-memory copy of the workflow's mutable
state**, from the history service's own clock, once the heartbeat call has the workflow lock and the
mutable state is in memory. Only then does it save the mutable state to the database. If the
mutable state is not in memory, the history service first reads it from the database. The heartbeat
call has two parts, before and after the history service records the heartbeat's time:

- **Anything that slows the heartbeat call before the history service records the heartbeat's time
  makes this heartbeat later:**
  - the SDK worker sending the call, and the call's trip over the network to the frontend service;
  - the frontend service's checks ([2.1](#21-the-grpc-message-size-limit) to
    [2.3](#23-the-payload-size-limit)), and the frontend service trying the call again after a
    `history.rps` refusal ([2.4](#24-the-history-services-requests-per-second-limits));
  - the history service's RPS check, and the wait for the workflow lock;
  - a read of the mutable state from the database, when the history service's cache does not hold
    it.

  **[Heartbeat Call Latency (Frontend and
  History)](../observability/dashboards/server/temporal-server-readme.md#4-service-latencies)** shows
  all of these except the SDK worker and the network.
- **The database write comes after the history service records the heartbeat's time, so a slow
  write does not change this heartbeat's time.** The history service answers the SDK only once the
  write is done, and the SDK starts its next batching window only after that answer
  ([1.1](#11-the-batching-window)). So a slow write makes the next heartbeat later.

**Two panels show reads of the mutable state from the database.** **[Workflow Lock Hold
Duration](../observability/dashboards/server/temporal-server-readme.md#2-shard-and-workflow-lock-latencies)**
shows how long calls hold the workflow lock, these reads included. **[Write-Reject Loop Indicator
(cleared / cache
miss)](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
shows whether the reads come from refused database writes.
[History Task Processing](./history-task-processing-tuning.md#31-the-sequence-that-creates-the-loop)
explains how refused writes lead to repeated reads.

**Waiting for the workflow lock can make a heartbeat late enough for the activity to time out on its
heartbeat timeout.** For an activity with a 10-second heartbeat timeout:

- **The SDK sends heartbeats at most once per batching window, which is 80% of the heartbeat
  timeout**, so 8 seconds ([1.1](#11-the-batching-window)). The window starts when the server
  answers the previous heartbeat call.
- **The heartbeat call the SDK sends when the window ends has the remaining 2 seconds.** In that
  time the call reaches the history service, waits for the workflow lock and loads the mutable
  state, and the history service records the heartbeat time. The database write comes after that,
  but the next window starts only once the write is done, so the whole heartbeat call counts
  against the 2 seconds.

**The same holds for any heartbeat timeout.** Up to a heartbeat timeout of 75 seconds, a heartbeat
call has 20% of the heartbeat timeout. For a longer heartbeat timeout, the batching window stays at
60 seconds, so a heartbeat call has the heartbeat timeout minus 60 seconds.

**If heartbeat calls take longer than that time**, from the SDK sending them to the server
answering, the activity can time out on its heartbeat timeout. A timer task checks the heartbeat
timeout, and that task also needs the workflow lock:

- **If the late heartbeat call gets the lock first**, the history service records the new
  heartbeat time. The timer task then finds the new time, and the activity does not time out on
  its heartbeat timeout.
- **If the timer task gets the lock first**, the heartbeat timeout has passed, and the activity
  times out.
- **Each workflow run has one lock for all of its activities.** While one activity's heartbeat
  call holds the lock, through its database write, the heartbeat calls of the run's other
  activities and the run's timer tasks all wait. With many activities on one workflow run, those
  waits add up.

[Section 3](#3-how-the-server-works-out-and-checks-the-heartbeat-deadline) covers how the heartbeat
timeout is checked.

> <a id="rec-choose-heartbeat-timeout"></a>**Recommendation:** choose each activity's heartbeat timeout by weighing two things against each
> other:
>
> - **A shorter heartbeat timeout notices a crashed SDK worker sooner**, so the activity's next
>   attempt starts sooner.
> - **A longer heartbeat timeout leaves each heartbeat call more time** to reach the history service
>   and get the workflow lock. A 5-second heartbeat timeout leaves 1 second, a 10-second one 2
>   seconds, a 60-second one 12 seconds, and a 2-minute one 60 seconds.
>
> When heartbeat calls take close to the time they have, shown on **[Heartbeat Call Latency
> (Frontend and History)](../observability/dashboards/server/temporal-server-readme.md#4-service-latencies)**, a
> longer heartbeat timeout can stop the heartbeat timeouts. When noticing a crashed SDK worker quickly
> matters more, keep the heartbeat timeout short, and keep heartbeat calls fast instead, for example
> by spreading activities that heartbeat over several workflows.

> **Recommendation:** when a workflow runs many activities that heartbeat at the same time, watch
> these panels on the Temporal Server dashboard:
>
> - **[Heartbeat Call Latency (Frontend and
>   History)](../observability/dashboards/server/temporal-server-readme.md#4-service-latencies)**, against the time a
>   heartbeat call has. Heartbeat calls that take longer make heartbeats late.
> - **[Workflow Lock
>   Latency](../observability/dashboards/server/temporal-server-readme.md#2-shard-and-workflow-lock-latencies)**,
>   for how long calls wait for workflow locks. A rise there together with a rise in heartbeat call
>   latency can mean that the heartbeat calls are waiting for each other.
> - **[Timer Tasks Throttled by a Busy
>   Workflow](../observability/dashboards/server/temporal-server-readme.md#7-busy-workflow-throttling)**, for the
>   workflow's history tasks losing the workflow lock to the heartbeat calls. This panel moves
>   first, before any heartbeat fails.
> - **[Activity Heartbeat Timeout (Every
>   Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)**, for heartbeat
>   timeouts once heartbeat calls take longer than that time.

A heartbeat call that waits for the workflow lock can still keep the activity running, as long as
the call gets the lock before the timer task that checks the heartbeat timeout.

**A heartbeat call does not wait for the workflow lock forever.** The SDK sets a time limit on
every heartbeat call, the call's deadline. The call's deadline is not the heartbeat deadline from
[3.1](#31-the-heartbeat-deadline). The history service waits for the lock until half a second
before the call's deadline.
If the lock is still taken then, the history service refuses the heartbeat call with
`RESOURCE_EXHAUSTED`, cause `BusyWorkflow`, and the SDK retries the call
([1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails)). These refusals show on the
`history - BusyWorkflow` line of
**[Heartbeat Calls Refused by Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)**.

**How long the history service waits for the lock depends on the SDK:**

- **With Go SDK v1.51.0 and later, until half a second before the batching window ends**, and at
  most 9.5 seconds with the default 10-second RPC timeout.
- **With Go SDK releases before v1.51.0, until half a second before half the batching window has
  passed**, and at most 9.5 seconds. For a 10-second heartbeat timeout, that is about 3.5 seconds.
- **With the Java SDK, up to 9.5 seconds**, because the Java SDK sets the deadline to the client's
  RPC timeout, 10 seconds by default.

**For a heartbeat timeout under about 47 seconds, that wait is longer than the time a heartbeat
call has**, 20% of the heartbeat timeout. By the time the history service refuses such a heartbeat
call with `BusyWorkflow`, the heartbeat is already late.

**Heartbeat calls are rarely refused with `BusyWorkflow`, because heartbeat calls wait for the lock
much longer than the workflow run's history tasks do:**

- **A heartbeat call keeps waiting for the lock** until half a second before its deadline, several
  seconds.
- **A history task waits for the lock for only half a second**, set by
  `history.cacheNonUserContextLockTimeout`. If the lock is still taken, the history task stops
  waiting, is refused with `BusyWorkflow`, and is tried again later. A timer task refused this way
  shows on **[Timer Tasks Throttled by a Busy
  Workflow](../observability/dashboards/server/temporal-server-readme.md#7-busy-workflow-throttling)**.

So when a late heartbeat call and the timer task that checks the heartbeat timeout both wait for
the same lock, the heartbeat call gets the lock first, and the activity keeps running. The activity
times out on its heartbeat timeout in only two cases:

- **The timer task already holds the lock** when the heartbeat call arrives.
- **The history service processes the timer task before the heartbeat call arrives.**

> **Test cluster result:** heartbeat calls were rarely refused with `BusyWorkflow`, and the
> workflow run's history tasks often were.
>
> - **Goal:** with many activities heartbeating on one workflow run and a database made slower on
>   purpose, count how often heartbeat calls and the run's history tasks were refused with
>   `BusyWorkflow`.
> - **Setup:** server v1.31.0, 300 activities heartbeating on one workflow, and 50 milliseconds
>   added on purpose to every database write.
> - 1 of about 4,900 heartbeat calls was refused with `BusyWorkflow`.
> - The workflow run's history tasks were refused about 11,700 times in the same 8 minutes.

With the heartbeat's time recorded, the history service saves the heartbeat to the database.

### 2.6 What a heartbeat writes to the database, and the persistence QPS limits

**Each recorded heartbeat is one database write that saves the workflow run's mutable state, and
adds no event to the workflow's event history.** The write saves the activity's new heartbeat time
and heartbeat details. The heartbeat's write is the same database write as any other change to a
running workflow, the `UpdateWorkflowExecution` persistence operation. Unlike other changes, such
as a signal or a completed activity, a heartbeat adds no event to the event history and creates no
workflow task.

**Each recorded heartbeat replaces the heartbeat details the server holds.** The server keeps only
the details from the last recorded heartbeat, in the activity's record in mutable state, and keeps
none from earlier heartbeats. A heartbeat call without details clears the details saved earlier.
Each heartbeat writes the details in full, so large heartbeat details make every heartbeat's
database write bigger. The size of the heartbeat details shows on the Temporal Server dashboard in
**[Heartbeat Details Size](../observability/dashboards/server/temporal-server-readme.md#12-workflow-history-state-and-payload-sizes)**,
on server v1.30.0 and later.

**On a global namespace replicated to more than one cluster, each heartbeat also adds a replication
task.** On any other namespace, a heartbeat adds no task for the history service's task processing:

- **A heartbeat adds no task to check the heartbeat deadline.** That timer task already exists
  before the heartbeat, and [section 3](#3-how-the-server-works-out-and-checks-the-heartbeat-deadline)
  covers how it runs.
- **On a namespace whose cluster list has more than one cluster, a heartbeat adds one replication
  task.** The heartbeat's database write adds it. The replication task carries the activity's
  heartbeat time and heartbeat details to the other clusters of the namespace. The standby
  cluster's copy of the workflow then shows the same last heartbeat time as the active cluster's.

[Namespace Failover and Graceful
Handover](./namespace-failover-graceful-handover.md#what-your-clients-and-workers-experience)
covers what happens to heartbeat calls while such a namespace is handed over to another cluster.

> **Test cluster result:** heartbeats wrote replication tasks, and the checks of the heartbeat
> deadline did not.
>
> - **Goal:** check which heartbeat-related writes create replication tasks on a namespace that
>   replicates to a second cluster.
> - **Setup:** two clusters running server v1.31.0, and one workflow running one heartbeating
>   activity for 60 seconds.
> - The workflow's 8 heartbeats wrote 8 replication tasks.
> - The history service's 7 checks of the activity's heartbeat deadline, which
>   [section 3](#3-how-the-server-works-out-and-checks-the-heartbeat-deadline) describes, wrote none.

> **Recommendation:** keep heartbeat details small, as [2.1](#21-the-grpc-message-size-limit)
> recommends. Every heartbeat writes the details to the database again, and on a namespace that
> spans more than one cluster, replicates them again.

**Each heartbeat's database write counts against the history service's persistence QPS limits.**
These limits cap how many database calls a second the history service makes.
[History Persistence QPS Limits](./history-persistence-qps-limits.md) covers them and how to set
them. For heartbeats, two things matter:

- **Heartbeat writes have a higher priority than the timer tasks that check the heartbeat
  deadline.** The timer tasks that check activity timeouts, the heartbeat timeout among them, have
  a lower priority. Near a QPS limit, the history service refuses them before heartbeat writes. A
  refused timer task is tried again later, so it is processed late.
  [Section 3](#3-how-the-server-works-out-and-checks-the-heartbeat-deadline) shows why a timer task
  processed late never causes an early heartbeat timeout.
- **A heartbeat write that one of the history service's persistence QPS limits refuses is never
  sent to the database.** The limit refuses it at once, and the heartbeat call fails with
  `RESOURCE_EXHAUSTED`, cause `PersistenceLimit`. The heartbeat is not recorded, and the history
  service discards its in-memory copy of the mutable state. The next call on the workflow reads the
  mutable state from the database again. After the refusal:
  - **The history service does not retry the write.**
  - **The frontend service tries the heartbeat call again, only when the history service's
    overall limit refused the write**, `history.persistenceMaxQPS` or
    `history.persistenceGlobalMaxQPS`, as for `history.rps` in
    [2.4](#24-the-history-services-requests-per-second-limits).
  - **The frontend service does not retry a write refused by a namespace limit**,
    `history.persistenceNamespaceMaxQPS`, `history.persistenceGlobalNamespaceMaxQPS` or
    `history.persistencePerShardNamespaceMaxQPS`. The frontend service answers the SDK at once with
    `RESOURCE_EXHAUSTED`.
  - **The SDK then retries the heartbeat call**, as
    [1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails) describes.

  Refused heartbeat calls show in **[Heartbeat Calls Refused by
  Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)** under
  the history service.

> <a id="rec-persistence-limit"></a>**Recommendation:** when the `history - PersistenceLimit` line on **[Heartbeat Calls Refused by
> Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)** shows
> refusals, decide whether to raise the limit or lower the history service's database calls.
> [Section 4.6](./history-persistence-qps-limits.md#46-the-decision--what-to-actually-do) of the
> History Persistence QPS Limits playbook covers that decision.

A heartbeat's write that the persistence QPS limits let through can still fail in the database.

### 2.7 When the database write of a heartbeat fails

**A heartbeat whose write fails in the database is not recorded, and the heartbeat call fails.** The
history service discards its in-memory copy of the mutable state, and the SDK retries the heartbeat
call as [1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails) describes. What else happens
depends on whether the history service can tell that the write was not saved:

- **The database refused the write, so nothing was saved.** For example, the workflow's record in
  the database changed since the history service read it, or the write was too large. The history
  service only fails the heartbeat call.
- **The write timed out, or failed with another error, so the history service cannot tell whether
  the write was saved.** The history service then renews its ownership of the workflow's shard,
  which the next paragraph explains.

**When the history service cannot tell whether a write was saved, it renews its ownership of the
workflow's shard.** This happens mostly when a write times out:

- **A write that timed out could still reach the database later**, so reading the workflow right
  away could show the old state while the write is still on its way.
- **To rule that out, the history service writes a new ownership number for the shard to the
  database.** Every write is checked against that number, so a write still on its way with the old
  number is refused. After that, every read shows what was really saved.
- **While the history service renews its ownership, other calls for workflows on that shard fail
  for a short time.**

> **Test cluster result:** a heartbeat write that timed out made the history service renew its
> ownership of the shard, and other calls for workflows on that shard failed.
>
> - **Goal:** reproduce a heartbeat database write that times out, by making the workflow's
>   database writes slower than the heartbeat call on purpose, and see what the history service
>   does next.
> - **Setup:** server v1.31.0, Go SDK v1.48.0 and PostgreSQL. One activity with a 10-second
>   heartbeat timeout called the heartbeat API every 500 milliseconds. For 30 seconds, every
>   database write of its workflow was made 5 seconds slower on purpose, so each write took
>   longer than the call that made it.
> - **8 database writes of the workflow timed out**, 1 for a heartbeat and 7 for the history
>   service's checks of the activity's heartbeat deadline.
> - **The history service renewed its ownership of the workflow's shard 8 times**, and 25 other
>   calls for workflows on that shard failed.

> <a id="rec-slow-database-writes"></a>**Recommendation:** watch for slow and failing database writes on the Temporal Server dashboard:
>
> - **[Persistence
>   Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**,
>   for the `UpdateWorkflowExecution` persistence operation. Writes that take close to the
>   heartbeat call's deadline are about to time out.
> - **[Persistence Errors Total by
>   Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**,
>   for failed writes of the `UpdateWorkflowExecution` persistence operation.
>
> No panel shows the history service renewing its ownership of a shard. The history service logs
> each write that timed out with the message `Update workflow execution operation failed.` at log
> level ERROR. While it renews, other calls on that shard fail with `shard status unknown` in the
> message `service failures`, at log level ERROR.
>
> When the writes are slow or failing, the database is the cause, not the heartbeat settings.
> [History Persistence QPS Limits](./history-persistence-qps-limits.md) covers keeping the history
> service's database calls within what the database can take.
> [Shard IO Concurrency](./shard-io-concurrency.md) covers how many writes on one shard the history
> service makes at the same time with SQL persistence.

[History Persistence QPS Limits](./history-persistence-qps-limits.md) covers how to keep the
database within what it can handle.

**How the server receives and records a heartbeat is now covered.**
[Section 3](#3-how-the-server-works-out-and-checks-the-heartbeat-deadline) covers how the server
uses the recorded heartbeat time to decide when an activity has timed out.

---

## 3. How the server works out and checks the heartbeat deadline

**This section explains how the server decides that an activity attempt has timed out on its
heartbeat timeout.** [Section 2](#2-how-the-server-receives-and-records-a-heartbeat) ended with the
heartbeat's time recorded in the workflow's mutable state. This section covers:

- how the history service works out the **heartbeat deadline**, the time by which the next
  heartbeat must be recorded ([3.1](#31-the-heartbeat-deadline));
- the timer task that checks the heartbeat deadline, and what each timer task costs
  ([3.2](#32-how-a-timer-task-checks-the-heartbeat-deadline));
- why a timer task processed late never causes an early heartbeat timeout
  ([3.3](#33-a-timer-task-processed-late-never-causes-an-early-heartbeat-timeout));
- why a heartbeat timeout under about one second is not checked any sooner
  ([3.4](#34-heartbeat-timeouts-under-about-one-second));
- raising the heartbeat timeout of an activity that is already running
  ([3.5](#35-raising-the-heartbeat-timeout-of-an-activity-that-is-already-running)).

### 3.1 The heartbeat deadline

**The heartbeat deadline is the heartbeat timeout added to the later of two times**, the time the
activity attempt started and the last heartbeat time. The server does not store the heartbeat
deadline. The history service works it out from the activity's record in mutable state each time it
needs it, so each recorded heartbeat moves the heartbeat deadline later.

**Until an activity attempt has a recorded heartbeat, the attempt's start time counts**, so a new
activity attempt has a full heartbeat timeout from its start. A heartbeat time left over from an
earlier attempt does not count, because the new attempt's start time is later.

**An activity attempt starts when the history service records it as started, before the SDK worker
has the activity task.** Before the matching service passes the activity task on, it asks the history
service to record the start. The frontend service then answers the polling SDK worker with the
activity task. Everything after the recorded start counts against the first heartbeat timeout:

- the activity task's trip from the matching service, through the frontend service, to the SDK
  worker;
- the SDK worker starting the activity code, which can take longer on a busy SDK worker
  ([1.3](#13-why-an-sdk-worker-can-send-heartbeats-late));
- the first heartbeat call.

**The history service works out no heartbeat deadline for an activity that is not running.** An
activity waiting for an SDK worker to pick it up, or waiting before its next retry, has no heartbeat
deadline. Neither has an activity without a heartbeat timeout.

**The heartbeat timeout is never longer than the StartToClose timeout.** When the workflow schedules
the activity, the history service lowers a longer heartbeat timeout to the StartToClose timeout, and
to the workflow run timeout if the workflow has one. The Temporal CLI command
`temporal activity update-options` lowers it to the StartToClose timeout only, not to the workflow
run timeout. A heartbeat timeout set longer than the StartToClose timeout has no effect.

> <a id="rec-starttoclose-with-heartbeat-timeout"></a>**Recommendation:** to give an activity a heartbeat timeout longer than its StartToClose timeout,
> raise the StartToClose timeout too.

The history service checks the heartbeat deadline with a timer task, which
[3.2](#32-how-a-timer-task-checks-the-heartbeat-deadline) covers.

### 3.2 How a timer task checks the heartbeat deadline

**A timer task is a history task with a due time.** The history service saves it with the workflow
run. Once the due time comes, the history service loads the timer task from the database, queues it
in its task scheduler, and then **processes** it, which means it executes the timer task's work.
[History Task Processing](./history-task-processing-tuning.md#2-how-task-processing-works) explains
these steps. [2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late) explains history
tasks.

**However often an activity heartbeats, the history service keeps at most one timer task waiting to
check the activity's heartbeat deadline:**

- **Once the activity attempt has started**, the history service writes the timer task, due one
  heartbeat timeout after the start.
- **A heartbeat moves the heartbeat deadline later, and leaves the waiting timer task as it is.**
  The heartbeat does not update or remove the timer task, and writes no new one. For example, with a
  10-second heartbeat timeout, the timer task written at the start is due at 10 seconds. A heartbeat
  at 8 seconds moves the heartbeat deadline to 18 seconds, and the timer task stays due at 10
  seconds.
- **When the history service processes the timer task**, at 10 seconds in the example, it replaces
  the timer task with a new timer task due at 18 seconds, if the activity attempt is still running.
  The next paragraph explains this.

**When the history service processes the timer task, it works out the heartbeat deadline again:**

- **If the heartbeat deadline has passed**, the activity attempt times out on its heartbeat timeout.
  [Section 4](#4-what-happens-after-the-server-times-out-an-activity-on-its-heartbeat-timeout) covers what follows.
- **If a heartbeat moved the heartbeat deadline later**, nothing times out. The history service saves
  the workflow's mutable state and writes a new timer task, due one heartbeat timeout after the last
  recorded heartbeat. The new timer task goes into the database in the same write as the mutable
  state. The save is one more database write, the `UpdateWorkflowExecution` persistence operation.

**An activity that heartbeats steadily costs about two database writes per batching window**, one
for the heartbeat and one for the timer task. A heartbeat moves the heartbeat deadline about once per
batching window ([1.1](#11-the-batching-window)), so the timer task finds the heartbeat deadline
moved about once per batching window too.

> **Test cluster result:** one activity that heartbeated steadily caused about two database writes
> per batching window, one for the heartbeat and one for the timer task.
>
> - **Goal:** count the heartbeat writes and the timer task writes of one activity that heartbeats
>   steadily.
> - **Setup:** server v1.31.0, Go SDK v1.48.0 and PostgreSQL. One activity with a 10-second
>   heartbeat timeout called the heartbeat API every 500 milliseconds for 2 minutes.
> - **16 heartbeat writes**, one at the start of the attempt and then one about every 8 seconds.
> - **14 timer task writes**, one each time the timer task found the heartbeat deadline moved.
> - **30 writes in about 15 batching windows**, so about two per batching window. The workflow's
>   other 6 database writes were for starting and completing the workflow task and the activity.

**Each time the history service processes the timer task, it counts on [Timer Tasks Processed by
Type](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info) under
`TimerActiveTaskActivityTimeout`**, together with the checks of the activity's other timeouts. The
panel counts checks, not timeouts. Each activity that heartbeats steadily adds about one check per
batching window, so the line grows with the number of activities that heartbeat, even when none
times out. Heartbeat timeouts show on **[Activity Heartbeat Timeout (Every
Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)**.

**The timer task also needs the workflow lock**, and gets it after heartbeat calls
([2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late)). A timer task can therefore
be processed later than its due time, which
[3.3](#33-a-timer-task-processed-late-never-causes-an-early-heartbeat-timeout) covers.

### 3.3 A timer task processed late never causes an early heartbeat timeout

**The history service never processes a timer task before its due time.** The history service can
load a timer task from the database up to about one second before its due time, and then holds the
timer task until its due time.

**A timer task processed late gives the activity more time, never less.** The history service works
out the heartbeat deadline when it processes the timer task, from the last heartbeat time recorded
by then.

For example, with a 10-second heartbeat timeout, a timer task is due at 10 seconds, but the history
service processes it only at 14 seconds. A heartbeat recorded at 12 seconds moved the heartbeat
deadline to 22 seconds, so the timer task finds the heartbeat deadline still ahead, and the activity
does not time out on its heartbeat timeout.

**A timer task can be processed late for several reasons:**

- **The workflow lock is taken.** The timer task waits for the workflow lock for only half a second,
  and is then tried again later
  ([2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late)).
- **The history service is busy.** Activity timeout checks are in the middle of the history
  service's three priority classes for tasks, and get a slightly smaller share of task processing
  than the highest class. [History
  Task
  Processing](./history-task-processing-tuning.md#61-three-priority-classes-fixed-by-task-type)
  covers the priority classes.
- **A persistence QPS limit refuses the timer task's database calls.** The history service refuses
  timeout checks before heartbeat writes
  ([2.6](#26-what-a-heartbeat-writes-to-the-database-and-the-persistence-qps-limits)).
- **The workflow's shard moves to another history service instance**, which loads the shard's timer
  tasks again.

**A timer task processed late only makes a heartbeat timeout come later.** The server notices a
crashed SDK worker only through the heartbeat timeout. After the SDK worker crashes, no heartbeat
moves the heartbeat deadline, and the activity times out on its heartbeat timeout when the history
service processes the timer task.

For example, with a 10-second heartbeat timeout, an SDK worker that crashes right after a heartbeat
at 20 seconds leaves the heartbeat deadline at 30 seconds. If the history service processes the
timer task only at 34 seconds, the activity times out at 34 seconds instead of 30, and its next
attempt is scheduled 4 seconds later. **[Activity Heartbeat Timeout (Every
Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)**
counts the heartbeat timeout when it happens. **[Timer Task Delay After Due Time by
Type](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)**
shows how late the timer task was processed.

**Clock differences between history service instances are one exception, not measured.** The history
service records heartbeat times and checks heartbeat deadlines with its own clock. When a shard moves
to a history service instance whose clock is ahead of the previous owner's, a timer task can be
processed early in real time, by that difference.

> **Recommendation:** keep the clocks of the history service hosts in sync, and watch how late the
> history service checks activity deadlines in **[Timer Task Delay After Due Time by
> Type](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)**,
> under `TimerActiveTaskActivityTimeout`. A rise there means that crashed SDK workers are noticed
> later than their heartbeat timeout. The rise does not cause heartbeat timeouts.

> **Test cluster result:** with a busy workflow, the activity deadline checks were processed up to
> about 98 seconds late, and no activity timed out.
>
> - **Goal:** with many activities heartbeating on one workflow run and a database made slower on
>   purpose, see how late the history service checks the activities' deadlines.
> - **Setup:** server v1.31.0, 300 activities heartbeating on one workflow, and 50 milliseconds
>   added on purpose to every database write.
> - **Normally the activity deadline checks were processed about 0.24 seconds late at p99**, on
>   **[Timer Task Delay After Due Time by
>   Type](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)**.
> - **With the busy workflow, the activity deadline checks were processed up to about 98 seconds
>   late.**

A timer task due less than about one second after it is written is also processed later than its
due time, which [3.4](#34-heartbeat-timeouts-under-about-one-second) covers.

### 3.4 Heartbeat timeouts under about one second

**The history service fetches timer tasks from the database ahead of time**, so that each timer task
is already in memory when it comes due. Each fetch loads all timer tasks due within the next
`history.timerProcessorMaxTimeShift`, 1 second by default.

**A new timer task due sooner than the end of the last fetch is moved to just after it.** The last
fetch already loaded the timer tasks due within its second, so a timer task written after that fetch
would not be in memory. A timer task due less than about one second after it is written can
therefore be processed up to about one second after it is written.

**For a heartbeat timeout under one second, the timer task is processed about once a second.** With a
500-millisecond heartbeat timeout, each new timer task should be due 0.5 seconds after it is
written, but the history service processes it up to about 1 second after instead.

**A heartbeat timeout under one second notices a crashed SDK worker no sooner, and costs more.** The
SDK still sends a heartbeat every 80% of the heartbeat timeout ([1.1](#11-the-batching-window)). A
200-millisecond heartbeat timeout means about 6 heartbeat calls a second for one activity, and each
heartbeat is a database write.

> **Test cluster result:** heartbeat timeouts under one second were checked about once a second, and
> cost many more heartbeat calls.
>
> - **Goal:** see how often the history service checks very short heartbeat timeouts, and what they
>   cost.
> - **Setup:** server v1.31.0, Go SDK v1.48.0 and PostgreSQL. One activity per test, with heartbeat
>   timeouts of 200 milliseconds, 500 milliseconds and 2 seconds.
> - **The timer task was processed 31 times in 30 seconds with both 200 and 500 milliseconds**,
>   about once a second, and 19 times with 2 seconds.
> - **A 500-millisecond heartbeat timeout fired 0.86 and 0.89 seconds after the last heartbeat**,
>   about 0.4 seconds late. A 2-second heartbeat timeout fired 2 milliseconds late.
> - **A 200-millisecond heartbeat timeout made 181 heartbeat calls in 30 seconds**, against 19 for a
>   2-second heartbeat timeout.

> **Recommendation:** do not set any timer or timeout under one second. The server processes such a
> timer or timeout up to about one second late. That includes:
>
> - **the heartbeat timeout**, which also notices a crashed SDK worker no sooner, and makes each
>   activity send many more heartbeat calls and database writes;
> - the other timeouts in an activity's options, and the activity's retry interval;
> - timers in workflow code, such as `workflow.Sleep` in the Go SDK and `Workflow.sleep` in the Java
>   SDK.
>
> [2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late) covers how to choose a
> heartbeat timeout.

So far, this section has used the heartbeat timeout from the activity options that the workflow code
set when the workflow scheduled the activity. When an activity that is already running needs a
longer heartbeat timeout, its heartbeat timeout can be raised without waiting for new activities,
which
[3.5](#35-raising-the-heartbeat-timeout-of-an-activity-that-is-already-running) covers.

### 3.5 Raising the heartbeat timeout of an activity that is already running

**A new heartbeat timeout in the workflow code applies only to activities that the workflow code
starts after the new workflow code is deployed.** For activities that are already running, the
Temporal CLI command `temporal activity update-options` changes the heartbeat timeout without
restarting the activity. The Temporal CLI marks the command as experimental, up to Temporal CLI
v1.10.1, the latest release when this playbook was written.

```
temporal activity update-options \
    --namespace YourNamespace \
    --workflow-id YourWorkflowId \
    --activity-id YourActivityId \
    --heartbeat-timeout 60s
```

- **Check the current heartbeat timeout before changing it.** The heartbeat timeout may already have
  been changed, by another team for example. The Temporal CLI command
  `temporal workflow describe --output json` shows it for each pending activity, as
  `heartbeatTimeout` under `activityOptions`. In the Temporal Web UI, the Pending Activities tab
  shows it under Updated Activity Options, but only after a change. Before any change, it is the
  heartbeat timeout in the workflow's `ActivityTaskScheduled` event.
- **The new heartbeat timeout applies to the activity attempt that is already running, and to every
  later activity attempt.** This holds even when the running activity attempt has been sending
  heartbeats for a long time. The history service replaces the heartbeat timer of the activity
  attempt. From then on, the history service works out the heartbeat deadline from the last
  heartbeat and the new heartbeat timeout ([3.1](#31-the-heartbeat-deadline)).
- **The SDK worker keeps the batching window of the original heartbeat timeout, on every activity
  attempt.** The server gives each new activity attempt the heartbeat timeout from when the workflow
  scheduled the activity. Raising the heartbeat timeout therefore leaves each heartbeat call more
  time, but the SDK sends heartbeats as often as before.
- **Never lower the heartbeat timeout of a running activity.** The SDK worker keeps sending
  heartbeats at the batching window of the original heartbeat timeout, later than the new heartbeat
  timeout allows, so every activity attempt times out. When the new heartbeat timeout is shorter
  than the time since the last heartbeat, the running activity attempt times out at once.
- **`--restore-original-options` undoes a change.** It sets the heartbeat timeout, and the other
  activity options, back to the values from when the workflow scheduled the activity. An activity
  attempt that already timed out stays timed out.
- **The heartbeat timeout cannot go above the StartToClose timeout.** The server sets it to the
  StartToClose timeout instead, and the Temporal CLI shows no error.
- **`--query` changes every pending activity in every workflow that the query matches.** The query
  picks workflows, for example by workflow type, not activities. So other pending activities in
  those workflows get the new heartbeat timeout too, and time out on every activity attempt if it is
  shorter than theirs, or if they send no heartbeats.

> **Test cluster result:** lowering a running activity's heartbeat timeout timed out every activity
> attempt, and a heartbeat timeout above the StartToClose timeout was set to the StartToClose
> timeout.
>
> - **Goal:** find out what the Temporal CLI command
>   `temporal activity update-options --heartbeat-timeout` does to an activity that is running,
>   when it lowers the heartbeat timeout and when it raises it.
> - **Setup:** server v1.31.0, Go SDK v1.48.0 and Temporal CLI v1.7.0.
> - **Lowered from 30 seconds to 10 seconds, 12 seconds after the activity started, with a heartbeat
>   call every half second and maximum attempts 3:** the running activity attempt timed out at once.
>   Each of the next 2 activity attempts sent its first heartbeat at once and its next one 24
>   seconds later, the batching window of the original 30-second heartbeat timeout, and timed out.
>   The workflow failed.
> - **Raised to 2 hours on an activity with a 30-minute StartToClose timeout:** the server set the
>   heartbeat timeout to the 30-minute StartToClose timeout.

**`temporal activity update-options` needs a server and a Temporal CLI at these versions or later:**

- **Server v1.29.0 and later.** On server v1.26 to v1.28, the server refuses the command unless the
  server dynamic config setting `frontend.activityAPIsEnabled` is `true` for the namespace.
- **Temporal CLI v1.2.0 and later**, v1.5.0 and later for `--restore-original-options`, and v1.9.0
  and later for `--query`. With an older Temporal CLI, `--query` sends which options to change but
  not their new values, so do not use it.

> <a id="rec-raise-running-heartbeat-timeout"></a>**Recommendation:** when a longer heartbeat timeout is the fix
> ([2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late)), change the heartbeat
> timeout in two places:
>
> 1. **On the pending activities, with the Temporal CLI command `temporal activity update-options`.**
>    A change to the workflow code does not reach them. The server retries an activity with the
>    heartbeat timeout from when the workflow scheduled it, on every activity attempt. Without the
>    command, they keep timing out until their activity retries run out.
> 2. **In the workflow code**, so that activities scheduled after the deploy get the new heartbeat
>    timeout. Activities scheduled before the deploy finishes keep the old one, so run the command
>    again after the deploy.
>
> Before running the command, check the current heartbeat timeout, and never set a shorter heartbeat
> timeout. Use `--query` only when every pending activity in the matched workflows should get the
> new heartbeat timeout.

**How the server works out and checks the heartbeat deadline is now covered.**
[Section 4](#4-what-happens-after-the-server-times-out-an-activity-on-its-heartbeat-timeout) covers what the server does when the timer
task finds the heartbeat deadline passed.

---

## 4. What happens after the server times out an activity on its heartbeat timeout

**This section follows an activity attempt after the server has timed it out on its heartbeat
timeout.** [Section 3](#3-how-the-server-works-out-and-checks-the-heartbeat-deadline) ended with the
timer task finding the heartbeat deadline passed. The subsections cover what the server does next,
and what the SDK worker running the timed-out activity attempt sees:

- whether the server retries the activity, or ends it with the heartbeat timeout
  ([4.1](#41-whether-the-server-retries-the-activity-after-a-heartbeat-timeout));
- what the workflow and its event history show
  ([4.2](#42-what-the-workflow-and-its-event-history-show-after-a-heartbeat-timeout));
- what the next activity attempt receives, what happens to the timed-out attempt, and what
  activity retries do to an SDK worker short of CPU
  ([4.3](#43-the-next-activity-attempt-and-the-timed-out-activity-attempt));
- how long after the heartbeats stop the server times out the activity on its heartbeat timeout
  ([4.4](#44-how-long-after-the-heartbeats-stop-the-server-times-out-the-activity));
- which dashboard panels and which fields of the Temporal CLI command `temporal workflow describe`
  show heartbeat timeouts
  ([4.5](#45-where-heartbeat-timeouts-show-on-the-dashboard-and-in-temporal-workflow-describe)).

### 4.1 Whether the server retries the activity after a heartbeat timeout

**The server retries an activity after a heartbeat timeout, the same as after any other failure,
unless the activity's retry policy stops it.** Every activity that a workflow schedules has a
retry policy. The server fills any field the workflow does not set from the server's dynamic config
setting `history.defaultActivityRetryPolicy`. By default:

- **Initial interval** is 1 second, the wait before the first retry.
- **Backoff coefficient** is 2, so each later wait is twice the one before.
- **Maximum interval** is 100 seconds, so the wait never grows past 100 seconds.
  `history.defaultActivityRetryPolicy` sets it as `MaximumIntervalCoefficient`, 100 times the
  initial interval.
- **Maximum attempts** is 0, which means no limit on the number of attempts.

**The heartbeat timeout ends the activity only in four cases:**

- **The retry policy's maximum attempts is reached.**
- **The next activity attempt would start after the activity's ScheduleToClose timeout.** The
  server then ends the activity with a ScheduleToClose timeout instead of the heartbeat timeout. The
  `ActivityTaskTimedOut` event in the event history, and the failure the workflow receives, carry
  the failure message:

  `Not enough time to schedule next retry before activity ScheduleToClose timeout, giving up retrying`
- **The retry policy marks the heartbeat timeout as non-retryable**, with `TemporalTimeout:Heartbeat`
  in its non-retryable error types.
- **The workflow code has asked the server to cancel the activity.** Cancelling the workflow, for
  example with the Temporal CLI command `temporal workflow cancel`, can lead the workflow code to
  cancel its activities too. The Temporal CLI command `temporal activity cancel` cancels only
  Standalone Activities, not activities that a workflow scheduled.

> **Recommendation:** do not add `TemporalTimeout:Heartbeat` to an activity's non-retryable error
> types.

**So with the default retry policy, an activity whose heartbeats keep timing out is retried until
its ScheduleToClose timeout, or with no end when the activity has no ScheduleToClose timeout.**

> **Test cluster result:** heartbeat timeouts were retried with the default waits, and only the last
> one ended the activity.
>
> - **Goal:** reproduce an activity that stops heartbeating on every attempt, and see how the server
>   retries it after each heartbeat timeout.
> - **Setup:** server v1.31.0, Go SDK v1.48.0 and PostgreSQL. One activity with a 10-second heartbeat
>   timeout stopped calling the heartbeat API after 2 seconds of every attempt. The retry policy
>   allowed 5 attempts, with the other fields at their defaults.
> - **Every heartbeat timeout was seen 19 to 76 milliseconds after its heartbeat deadline**, never
>   before it.
> - **The waits before attempts 2 to 5 were 1, 2, 4 and 8 seconds.**
> - **Attempt 5's heartbeat timeout ended the activity**, because the retry policy allowed no more
>   attempts.

> <a id="rec-limit-activity-attempts"></a>**Recommendation:** for each activity that heartbeats, set the retry policy's maximum attempts or
> the activity's ScheduleToClose timeout. With the default retry policy and no ScheduleToClose
> timeout, the server retries the activity with no end. When every attempt times out for a cause that
> does not go away by itself, for example an SDK worker short of CPU, the activity then never
> finishes, and the workflow waits for it with no end. With a limit, the activity fails when the
> limit is reached:
>
> - **Make sure the workflow code handles the activity failure.** If the workflow code does not
>   handle it, the workflow fails, in the Go and Java SDKs. Before adding a limit to an activity
>   that had none, check what the workflow code does when the activity fails.
> - **Never set maximum attempts to 1.** The server marks an activity attempt as started before the
>   SDK worker receives it, in the response to the SDK worker's poll or, with eager activity
>   execution, in the response to the workflow task completion. If that response is lost on the
>   way, the activity attempt never runs and times out on its heartbeat timeout or StartToClose
>   timeout. Only an activity retry then runs the activity. With maximum attempts 1, the activity
>   fails without having run once.

The workflow learns about a heartbeat timeout only in those four cases, which
[4.2](#42-what-the-workflow-and-its-event-history-show-after-a-heartbeat-timeout) covers.

### 4.2 What the workflow and its event history show after a heartbeat timeout

**The server keeps an activity's retries out of the event history, because an activity can be
retried many times.** With the default retry policy, the number of attempts has no limit
([4.1](#41-whether-the-server-retries-the-activity-after-a-heartbeat-timeout)). Events for every
attempt could fill the event history up to its limit, `limit.historyCount.error`, 51,200 events by
default, and the server terminates a workflow whose event history passes that limit. So a retried
heartbeat timeout adds no event to the event history and creates no workflow task, the same as every
other activity retry.

**The server keeps the latest attempt's details in the activity's record in mutable state instead**
([section 2](#2-how-the-server-receives-and-records-a-heartbeat)):

- the attempt number;
- the last failure, for example the heartbeat timeout;
- the last heartbeat time and the last heartbeat details.

The Temporal CLI command `temporal workflow describe` shows them
([4.5](#45-where-heartbeat-timeouts-show-on-the-dashboard-and-in-temporal-workflow-describe)). The
SDK worker running the timed-out attempt learns about the heartbeat timeout separately, as
[4.3](#43-the-next-activity-attempt-and-the-timed-out-activity-attempt) describes.

**The `ActivityTaskStarted` event is written only when the activity finishes, not when an attempt
starts.** The history service keeps each attempt's start in mutable state, for the same reason it
keeps activity retries out of the event history. While the activity runs and retries, the
event history has only the `ActivityTaskScheduled` event. When the activity finishes, the history
service writes two events together:

- **`ActivityTaskStarted`, for the last attempt only.** The event's time is when that last attempt
  started, not when the event was written. The event carries the last attempt's number, and the
  last failure before that attempt, for example the heartbeat timeout of the attempt before.
- **The event for how the activity finished**, for example `ActivityTaskCompleted`,
  `ActivityTaskFailed` or `ActivityTaskTimedOut`.

**So the `ActivityTaskStarted` event is the only place in the event history where earlier heartbeat
timeouts show**, as its attempt number and last failure.

**When a heartbeat timeout ends the activity, the event history gets an `ActivityTaskTimedOut`
event**, and the history service creates a workflow task, so the workflow code receives the timeout
as the activity's failure. The event carries the last heartbeat details the server recorded, and the
retry state, for example `RETRY_STATE_MAXIMUM_ATTEMPTS_REACHED`.

**When the workflow ends while the activity waits for its next activity attempt, the event history
keeps no record of the activity's attempts.** The event history then has only the activity's
`ActivityTaskScheduled` event.

> **Test cluster result:** a workflow ended while the server was retrying the activity kept no record
> of the heartbeat timeouts.
>
> - **Goal:** see what the event history keeps of the heartbeat timeouts of an activity whose
>   workflow ends while the server is retrying the activity.
> - **Setup:** server v1.31.0, Go SDK v1.48.0 and PostgreSQL. One activity timed out on its heartbeat
>   timeout 7 times, and the workflow was terminated while the activity waited for attempt 8.
> - **The event history had only the activity's `ActivityTaskScheduled` event**, and nothing about
>   the 7 heartbeat timeouts.

**An event history without the activity's attempts cannot show why the activity did not finish**,
because the heartbeat timeouts of the earlier attempts, and the activity's last failure, are recorded
nowhere in it.

> **Recommendation:** in workflow code, wait for the result of every activity the workflow starts,
> whether it completed, failed or timed out, before the workflow completes. The event history then
> always records how each activity ended, with its last attempt number and last failure.
>
> To end a workflow early, use the Temporal CLI command `temporal workflow cancel`, not the Temporal
> CLI command `temporal workflow terminate`. A terminated workflow does not run its workflow code, so
> its pending activities stay unrecorded. A cancelled workflow runs its workflow code, which can
> cancel its pending activities and wait for them:
>
> - **An activity attempt still running** learns about the cancellation from its next heartbeat call.
>   The event history then gets its `ActivityTaskStarted` event, with the last failure, and an
>   `ActivityTaskCanceled` event.
> - **An activity waiting for its next activity attempt** is cancelled at once. The event history
>   gets `ActivityTaskCancelRequested` and `ActivityTaskCanceled` events, but the heartbeat timeouts
>   of its earlier attempts still do not show.

The next activity attempt starts after the retry wait, and the timed-out activity attempt can still
be running, which [4.3](#43-the-next-activity-attempt-and-the-timed-out-activity-attempt) covers.

### 4.3 The next activity attempt and the timed-out activity attempt

**The next activity attempt receives the last heartbeat details the server recorded**, so activity
code that saves its progress in the heartbeat details can continue where the timed-out attempt
stopped. Activity code that saves no progress starts the work again from the beginning.

**Resetting an activity keeps its heartbeat details, unless the reset clears them.** The Temporal
CLI command `temporal activity reset` restarts the activity as if it were first scheduled. The
attempt number goes back to 1, and an activity attempt still running gets a cancellation on its next
heartbeat call. The next activity attempt still receives the last heartbeat details, unless the
reset uses `--clear-heartbeat-details`. The Temporal CLI marks `temporal activity reset` as
experimental, up to Temporal CLI v1.10.1, the latest release when this playbook was written.

> **Recommendation:** when resetting an activity, add `--clear-heartbeat-details` when the saved
> progress should not be used, for example after a fix to how the activity code saves its progress.
> Without it, the reset activity continues from the last heartbeat details.

**The next activity attempt gets a full heartbeat timeout from its own start**
([3.1](#31-the-heartbeat-deadline)). Until the next activity attempt sends its first heartbeat, the
Temporal CLI command `temporal workflow describe` still shows the timed-out attempt's last heartbeat
time, older than the next activity attempt's start time.

**The timed-out activity attempt keeps running on its SDK worker.** The server does not tell the SDK
worker about the heartbeat timeout. The SDK worker learns about it only from the answer to its next
heartbeat call, as [1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails) describes, or when the
activity finishes and the server turns its result away. Until then, the timed-out attempt and the
next activity attempt can run at the same time, on the same SDK worker or on different ones.

> **Recommendation:** make the activity code, and the APIs and services it calls, idempotent, so that
> running the same activity twice, even at the same time, has the same effect as running it once.
> After a heartbeat timeout, the timed-out attempt and the next activity attempt can overlap. Save
> progress in the heartbeat details, so the next activity attempt continues instead of starting over.

**On an SDK worker short of CPU, activity retries after heartbeat timeouts make the SDK worker even
busier.** The timed-out activity attempt keeps running until its next heartbeat call. Meanwhile an
SDK worker polling the same task queue starts the next activity attempt, and that can be the same
SDK worker. Each heartbeat timeout then adds work to the SDK worker whose CPU shortage caused the
timeout ([1.3](#13-why-an-sdk-worker-can-send-heartbeats-late)).

> **Test cluster result:** on an SDK worker short of CPU, the activity retries kept the activities
> timing out, and fewer activities per SDK worker stopped the heartbeat timeouts.
>
> - **Goal:** find out whether the activity retries after heartbeat timeouts on an SDK worker short
>   of CPU settle by themselves or keep growing.
> - **Setup:** server v1.31.0, one Java SDK v1.40.0 worker limited to 1 CPU, and 200 workflows,
>   each running one activity with 1 second of CPU work. The activity made a heartbeat call after
>   each millisecond of work and had a 10-second heartbeat timeout. Each activity attempt started
>   the work again from the beginning.
> - **200 activities at the same time, default retry policy:** no workflow finished for almost 4
>   minutes, and all 200 finished after 531 seconds. The server counted 1,519 heartbeat timeouts,
>   and 63% of the CPU went to activity attempts that timed out.
> - **200 activities at the same time, the wait between attempts capped at 1 second:** no workflow
>   finished in 12 minutes, when the test was stopped. The server counted 5,493 heartbeat timeouts.
> - **20 activities at the same time on the same CPU, default retry policy:** all 200 workflows
>   finished in 190 seconds, with 1 heartbeat timeout.

> <a id="rec-retries-on-cpu-short-worker"></a>**Recommendation:**
>
> - **When heartbeat timeouts come from an SDK worker short of CPU, lower the number of activities
>   each SDK worker runs at the same time before anything else**, as
>   [1.3](#13-why-an-sdk-worker-can-send-heartbeats-late) describes. The total work stays the same,
>   but each activity gets enough CPU to send its heartbeats on time.
> - **Do not shorten the wait between activity attempts** for activities that can time out on an
>   SDK worker short of CPU. Keep the retry policy's backoff coefficient and maximum interval at
>   their defaults, 2 and 100 seconds
>   ([4.1](#41-whether-the-server-retries-the-activity-after-a-heartbeat-timeout)), or larger. With
>   the defaults, the wait doubles after each activity attempt, so the activity attempts spread out
>   over time. With the maximum interval at 1 second, the SDK worker stayed short of CPU and no
>   activity finished.
> - **Limit the activity attempts**, with the retry policy's maximum attempts or the activity's
>   ScheduleToClose timeout, as
>   [4.1](#41-whether-the-server-retries-the-activity-after-a-heartbeat-timeout) recommends.

How long after the heartbeats stop the server times out the activity is covered in
[4.4](#44-how-long-after-the-heartbeats-stop-the-server-times-out-the-activity).

### 4.4 How long after the heartbeats stop the server times out the activity

**How long the server takes to time out the activity depends on why the heartbeats stopped:**

- **The activity code stopped calling the heartbeat API, but the SDK worker kept running**, for
  example because the activity code hangs. The SDK still sends the last heartbeat call when the
  batching window ends ([1.1](#11-the-batching-window)), up to 0.8 heartbeat timeouts after the
  activity code made it. The server then waits one full heartbeat timeout from that heartbeat. So the
  server times out the activity between 1 and about 1.8 heartbeat timeouts after the activity code's
  last heartbeat call.
- **The SDK worker crashed.** Heartbeat calls the SDK was still holding back are never sent. The
  server times out the activity one heartbeat timeout after the last heartbeat the SDK sent.

For example, with a 10-second heartbeat timeout, an activity's code hangs after calling the heartbeat
API 1.5 seconds into the batching window. The SDK sends that heartbeat when the window ends, at 8
seconds, so the heartbeat deadline is 18 seconds. The server times out the activity 16.5 seconds
after the activity code's last heartbeat call.

**The server can time out the activity later still**, when the history service processes the timer
task late ([3.3](#33-a-timer-task-processed-late-never-causes-an-early-heartbeat-timeout)), and never
sooner than about one second after the timer task is written
([3.4](#34-heartbeat-timeouts-under-about-one-second)).

> **Recommendation:** when choosing an activity's heartbeat timeout, count on the server timing out a
> hung activity up to about 1.8 heartbeat timeouts after the activity code's last heartbeat call. An
> activity on a crashed SDK worker times out one heartbeat timeout after the last heartbeat the SDK
> sent.
> [2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late) covers the trade-off between
> a short and a long heartbeat timeout.

An operator can also change when the server times out the activity, by pausing the activity.

**Pausing an activity does not stop its heartbeat deadline.** While the activity is paused, the
activity does not time out on its heartbeat timeout, even when the history service processes a
timer task for it. The heartbeat deadline still stays at the last heartbeat time plus the heartbeat
timeout.
When the activity is unpaused after its heartbeat deadline has passed, the activity times out on its
heartbeat timeout at once, and the server retries it as
[4.1](#41-whether-the-server-retries-the-activity-after-a-heartbeat-timeout) describes.

> **Test cluster result:** an activity unpaused after its heartbeat deadline timed out on its
> heartbeat timeout at once.
>
> - **Goal:** see what happens to an activity whose heartbeat deadline passes while it is paused.
> - **Setup:** server v1.31.0, Go SDK v1.48.0 and PostgreSQL. One activity with a 30-second
>   heartbeat timeout stopped calling the heartbeat API, and was paused before its heartbeat deadline.
>   The activity was unpaused 38 seconds after its heartbeat deadline had passed.
> - **The activity timed out on its heartbeat timeout as soon as it was unpaused**, and the next
>   attempt started about 1.4 seconds later, with the last heartbeat details.

Which dashboard panels and which fields of the Temporal CLI command `temporal workflow describe`
show heartbeat timeouts is covered in
[4.5](#45-where-heartbeat-timeouts-show-on-the-dashboard-and-in-temporal-workflow-describe).

### 4.5 Where heartbeat timeouts show on the dashboard and in `temporal workflow describe`

**The Temporal Server dashboard counts heartbeat timeouts on two panels, in the SDK Workers Info
row:**

- **[Activity Heartbeat Timeout (Every
  Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)**
  counts every activity attempt that timed out on its heartbeat timeout, retried or not, by activity
  type, on server v1.29.0 and later. The panel counts the same heartbeat timeout again each time the
  database write that records it fails and is retried, so it can read high while the database has
  trouble.
- **[Activity Heartbeat Timeout (Final
  Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)**
  counts only the heartbeat timeouts that ended the activity
  ([4.1](#41-whether-the-server-retries-the-activity-after-a-heartbeat-timeout)). A heartbeat timeout
  that ended the activity because no time was left before the ScheduleToClose timeout also counts
  here, although the workflow receives it as a ScheduleToClose timeout.

**With the default retry policy, Activity Heartbeat Timeout (Final Attempt) can stay at zero while
many activity attempts time out**, because every heartbeat timeout is retried.

> **Test cluster result:** thousands of retried heartbeat timeouts showed on Activity Heartbeat
> Timeout (Every Attempt), and none on Activity Heartbeat Timeout (Final Attempt).
>
> - **Goal:** reproduce activities that keep timing out on their heartbeat timeout on an SDK worker
>   short of CPU on purpose, and see which panel counts the heartbeat timeouts.
> - **Setup:** server v1.31.0, Java SDK v1.40.0 worker limited to 1 CPU on purpose, 200 workflows
>   with one activity each, and a retry policy that waited 1 second between attempts.
> - **5,493 activity attempts timed out on their heartbeat timeout in 12 minutes**, all retried.
> - **Activity Heartbeat Timeout (Final Attempt) stayed at zero.**

> **Recommendation:** watch **[Activity Heartbeat Timeout (Every
> Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** for
> heartbeat timeouts, not the Final Attempt panel, unless the activities' retry policies limit the
> number of attempts.

**For one activity, the Temporal CLI command `temporal workflow describe` shows the pending activity
while the server retries the activity:**

- **`Attempt`**, the current attempt number, which goes up with each heartbeat timeout.
- **`LastFailure`**, the heartbeat timeout of the attempt before, with the message `activity
  Heartbeat timeout`.
- **`ScheduledTime`**, when the next activity attempt starts, while the activity waits for it. With
  `--output json`, the same time is `nextAttemptScheduleTime`. Do not read `currentRetryInterval`
  there, which shows a wrong value after a heartbeat timeout, about 56 years on a test cluster.
- **`LastHeartbeatTime` and `LastHeartbeatDetails`**, from the timed-out attempt until the next
  activity attempt sends its first heartbeat
  ([4.3](#43-the-next-activity-attempt-and-the-timed-out-activity-attempt)).

**What happens after the server times out an activity on its heartbeat timeout is now covered, which
completes the explanation of how a heartbeat travels.**
[Section 5](#5-how-to-find-where-the-heartbeats-were-delayed) uses [sections 1 to 4](#contents) to find, on the
dashboards, where the heartbeats of a timed-out activity were delayed.

---

## 5. How to find where the heartbeats were delayed

**Start here when activities are timing out on their heartbeat timeout now.** [Sections 1 to
4](#contents) explain how a heartbeat travels from the SDK worker to the server. This section goes
through the dashboard panels in order, and links each panel to the section that explains what it
shows:

- which activity types are timing out on their heartbeat timeout
  ([5.1](#51-which-activity-types-are-timing-out-on-their-heartbeat-timeout));
- heartbeat calls that the server refused
  ([5.2](#52-heartbeat-calls-that-the-server-refused));
- heartbeat calls that were slow on the server
  ([5.3](#53-heartbeat-calls-that-were-slow-on-the-server));
- heartbeat calls that the SDK worker sent late, or that failed before they reached the server
  ([5.4](#54-heartbeat-calls-that-the-sdk-worker-sent-late-or-that-failed-before-the-server));
- one activity in detail ([5.5](#55-one-activity-in-detail)).

### 5.1 Which activity types are timing out on their heartbeat timeout

**Open [Activity Heartbeat Timeout (Every
Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info) first.** The panel
counts every activity attempt that timed out on its heartbeat timeout, retried or not, by activity
type ([4.5](#45-where-heartbeat-timeouts-show-on-the-dashboard-and-in-temporal-workflow-describe)).
**[Activity Heartbeat Timeout (Final
Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)** counts only the
heartbeat timeouts that ended the activity, so it can stay at zero while many attempts time out.

**When several activity types heartbeat, whether one of them or many have heartbeat timeouts can
narrow down the cause:**

- **Only one of the activity types** can point at that activity's code, or at the SDK workers that
  run it ([5.4](#54-heartbeat-calls-that-the-sdk-worker-sent-late-or-that-failed-before-the-server)).
- **Many of the activity types at once** can point at something they share, such as the namespace's
  rate limit, the history service or the database
  ([5.2](#52-heartbeat-calls-that-the-server-refused),
  [5.3](#53-heartbeat-calls-that-were-slow-on-the-server)).

With only one activity type that heartbeats, the panel does not narrow down the cause. In every
case, go on to [5.2](#52-heartbeat-calls-that-the-server-refused), which checks whether the server
refused heartbeat calls.

### 5.2 Heartbeat calls that the server refused

**Open [Heartbeat Calls Refused by
Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits).** Each line is one
service and one cause:

- **`frontend - RpsLimit`** means that the namespace's rate limit in the frontend service refused
  heartbeat calls ([2.2](#22-the-namespace-rate-limit-in-the-frontend-service)), or the frontend
  service passed on a refusal from the history service.
  [2.4](#24-the-history-services-requests-per-second-limits) explains how to tell the two apart.
- **`history - RpsLimit`** means that the history service's requests-per-second limits refused
  heartbeat calls ([2.4](#24-the-history-services-requests-per-second-limits)).
- **`history - BusyWorkflow`** means that heartbeat calls waited for the workflow lock until half a
  second before the call's deadline, and were then refused
  ([2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late)). The call's deadline is
  the time limit that the SDK sets on every heartbeat call, at most 10 seconds by default, and is
  not the heartbeat deadline from [3.1](#31-the-heartbeat-deadline).
  - **A heartbeat call is refused only when the lock stays taken for that whole wait**, several
    seconds. The lock is usually freed sooner, so the heartbeat call gets the lock late instead of
    being refused.
  - **An empty `history - BusyWorkflow` line does not mean that the workflow lock is not the
    cause**, so check the workflow lock in [5.3](#53-heartbeat-calls-that-were-slow-on-the-server).
- **`history - PersistenceLimit`** means that the history service's persistence QPS limits refused
  heartbeat writes
  ([2.6](#26-what-a-heartbeat-writes-to-the-database-and-the-persistence-qps-limits)).

**On Heartbeat Calls Refused by Cause, a `frontend` line with the same cause as a `history` line
shows the same refusals.** The frontend service also counts the refusals that it passes on from the
history service. A `frontend - BusyWorkflow` or `frontend - PersistenceLimit` line on the panel
therefore repeats the `history - BusyWorkflow` or `history - PersistenceLimit` line, and needs no
check of its own.

**Check whether the heartbeat call refusals and the heartbeat timeouts happen at the same time.**
Open [Activity Heartbeat Timeout (Every
Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info) for the
same time range. When the refusals rise together with the heartbeat timeouts, the refusals are the
cause. Refusals with no heartbeat timeouts at the same time did not make heartbeats late, because
the SDK retried the refused calls in time ([1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails)).
When the refusals line up with the heartbeat timeouts, [6.1](#61-recommendations-for-heartbeat-calls-that-the-server-refused) lists the recommendation for each
line of the panel.

**When the server refused no heartbeat calls, or the heartbeat call refusals do not line up with the
heartbeat timeouts, go on to [5.3](#53-heartbeat-calls-that-were-slow-on-the-server).** A heartbeat
call that the server accepts can still be late, when the call spends too long in the history service
before the history service records the heartbeat, for example waiting for the workflow lock
([2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late)).
[5.3](#53-heartbeat-calls-that-were-slow-on-the-server) checks for heartbeat calls that were slow on
the server.

### 5.3 Heartbeat calls that were slow on the server

**Open [Heartbeat Call Latency (Frontend and
History)](../observability/dashboards/server/temporal-server-readme.md#4-service-latencies).** The
panel has two lines, `frontend` and `history`, for how long heartbeat calls take in each service. The
`frontend` line includes the time in the history service, because the frontend service waits for the
history service's answer. Compare both lines with the time a heartbeat call has, 20% of the heartbeat
timeout for a heartbeat timeout up to 75 seconds
([2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late)):

- **`frontend` and `history` both high and close together** means that the heartbeat calls spend
  their time in the history service.
- **`frontend` high and `history` low** means that the heartbeat calls spend their time in the
  frontend service, before the history service, for example on the frontend service's retry after a
  `history.rps` refusal ([2.4](#24-the-history-services-requests-per-second-limits)).
- **`frontend` and `history` both low while heartbeat timeouts happen** means that the server is not
  the cause, so look at the SDK workers
  ([5.4](#54-heartbeat-calls-that-the-sdk-worker-sent-late-or-that-failed-before-the-server)).

**When the `history` line on Heartbeat Call Latency (Frontend and History) is high, find out whether
the workflow lock or the database slowed the heartbeat calls:**

- **[Workflow Lock
  Latency](../observability/dashboards/server/temporal-server-readme.md#2-shard-and-workflow-lock-latencies) high,
  with [Timer Tasks Throttled by a Busy
  Workflow](../observability/dashboards/server/temporal-server-readme.md#7-busy-workflow-throttling) above zero** means that
  many calls wait for one workflow run's lock, often because many activities heartbeat at the same
  time on one workflow run ([2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late)).
- **[Workflow Lock Hold
  Duration](../observability/dashboards/server/temporal-server-readme.md#2-shard-and-workflow-lock-latencies)
  high** means that each call holds the workflow lock for a long time, often because of slow
  database calls.
- **[Persistence
  Latencies](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)
  high for the `UpdateWorkflowExecution` persistence operation, or failed writes on [Persistence
  Errors Total by
  Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors)**
  mean that the heartbeat writes are slow or failing
  ([2.6](#26-what-a-heartbeat-writes-to-the-database-and-the-persistence-qps-limits),
  [2.7](#27-when-the-database-write-of-a-heartbeat-fails)).

**The workflow lock and slow database writes also make the history service check heartbeat deadlines
late.** [Timer Task Delay After Due Time by
Type](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info),
under `TimerActiveTaskActivityTimeout`, shows how late the history service processes the timer tasks
that check heartbeat deadlines. A timer task processed late never causes a heartbeat timeout
([3.3](#33-a-timer-task-processed-late-never-causes-an-early-heartbeat-timeout)). It only makes the
server time out the activity later than its heartbeat deadline
([4.4](#44-how-long-after-the-heartbeats-stop-the-server-times-out-the-activity)). So a high reading
on that panel shows the server noticing heartbeat timeouts late, not a cause of them.

**When the workflow lock or database panels show what slowed the heartbeat calls, [6.2](#62-recommendations-for-heartbeat-calls-that-were-slow-on-the-server) lists the
recommendation for each cause.**

When the server panels in [5.2](#52-heartbeat-calls-that-the-server-refused) and
[5.3](#53-heartbeat-calls-that-were-slow-on-the-server) show nothing unusual, the heartbeats were
late before they reached the server, which
[5.4](#54-heartbeat-calls-that-the-sdk-worker-sent-late-or-that-failed-before-the-server) covers.

### 5.4 Heartbeat calls that the SDK worker sent late, or that failed before the server

**Open the SDK dashboard for the SDK that the SDK workers use**, and select the operation
`RecordActivityTaskHeartbeat` on these panels:

- **Request Latency**
  ([Go](../observability/dashboards/sdk/temporal-sdk-go-readme.md#1-grpc-requests-and-request-latencies),
  [Java
  Micrometer](../observability/dashboards/sdk/temporal-sdk-java-micrometer-readme.md#1-grpc-requests-and-request-latencies),
  [Java
  OpenTelemetry](../observability/dashboards/sdk/temporal-sdk-java-otel-readme.md#1-grpc-requests-and-request-latencies)).
  Heartbeat calls slow here, but fast on the server's [Heartbeat Call Latency (Frontend and
  History)](../observability/dashboards/server/temporal-server-readme.md#4-service-latencies), lose their time on
  the SDK worker or on the network.
- **Request Failures**, by status code
  ([Go](../observability/dashboards/sdk/temporal-sdk-go-readme.md#1-grpc-requests-and-request-latencies),
  [Java
  Micrometer](../observability/dashboards/sdk/temporal-sdk-java-micrometer-readme.md#1-grpc-requests-and-request-latencies),
  [Java
  OpenTelemetry](../observability/dashboards/sdk/temporal-sdk-java-otel-readme.md#1-grpc-requests-and-request-latencies)):
  - **`RESOURCE_EXHAUSTED` while [Heartbeat Calls Refused by
    Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits) shows nothing** means that
    the gRPC message size limit refused heartbeat calls over 4 MB, before the server counted them
    ([2.1](#21-the-grpc-message-size-limit)).
  - **`RESOURCE_EXHAUSTED` together with refusals on Heartbeat Calls Refused by Cause** are the
    refusals of [5.2](#52-heartbeat-calls-that-the-server-refused).
  - **`NOT_FOUND`** is the SDK worker learning that the server had already timed out the activity
    attempt ([1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails)).
- **Worker Task Slots Used**
  ([Go](../observability/dashboards/sdk/temporal-sdk-go-readme.md#2-worker-lifecycle), [Java
  Micrometer](../observability/dashboards/sdk/temporal-sdk-java-micrometer-readme.md#2-worker-lifecycle),
  [Java
  OpenTelemetry](../observability/dashboards/sdk/temporal-sdk-java-otel-readme.md#2-worker-lifecycle))
  for activity workers shows how many activities the task queue's SDK workers run at the same time,
  added up across all of them. The panel does not show how many each SDK worker runs, so one SDK
  worker that runs far more activities than the others does not stand out on the panel.
  [1.3](#13-why-an-sdk-worker-can-send-heartbeats-late) gives two reasons why an SDK worker running
  many activities at once sends heartbeats late:
  - **Many activities running while the SDK worker hosts' CPU is near its limit** means that the
    SDK worker is short of CPU. Check host CPU in your own host monitoring, because the Temporal
    dashboards do not show it.
  - **On the Java SDK, many activities running while Request Latency for
    `RecordActivityTaskHeartbeat` is high** means that the 4 heartbeat threads that all activities
    on the SDK worker share are all busy. The other activities' heartbeats wait for a free thread.

[6.3](#63-recommendations-for-heartbeat-calls-that-the-sdk-workers-sent-late-or-that-failed-before-the-server) lists the recommendation for what the SDK dashboard panels show.

**Tell a crashed SDK worker apart from late heartbeats before changing anything.** When an SDK
worker crashes, its heartbeats stop, and the server times out its activities one heartbeat timeout
after the last heartbeat that the SDK worker sent
([4.4](#44-how-long-after-the-heartbeats-stop-the-server-times-out-the-activity)). The heartbeat
timeout then does what it is for. A crashed SDK worker shows this way:

- **The crashed SDK worker's lines on the SDK dashboard stop**, because its metrics stop with it.
  Late heartbeats instead show as slow or failed heartbeat calls on the panels above.
- **[Activity Heartbeat
  Calls](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits) on the
  server dashboard drops**, and the heartbeat timeouts follow about one heartbeat timeout later.

When the heartbeat timeouts line up with a crashed SDK worker, find out why the SDK worker crashed,
for example from its restarts in your own host or container monitoring. The recommendations in
[section 6](#6-which-recommendations-apply-to-each-cause-of-late-heartbeats) are for late
heartbeats, not for crashes.

**The dashboards show heartbeat timeouts across all activities, so confirm the diagnosis on one
affected activity.** [5.5](#55-one-activity-in-detail) reads one activity in the Temporal Web UI
or with the Temporal CLI command `temporal workflow describe`. The Web UI and the Temporal CLI both
show whether the activity timed out on its heartbeat timeout, how many attempts it has had, and
whether the current attempt has recorded a heartbeat yet.

### 5.5 One activity in detail

**Look at one affected workflow in the Temporal Web UI, on its Pending Activities tab, or with the
Temporal CLI command `temporal workflow describe`.** Both show the same fields of the pending
activity, with different names
([4.5](#45-where-heartbeat-timeouts-show-on-the-dashboard-and-in-temporal-workflow-describe)):

- **Last Failure in the Web UI, `LastFailure` in the CLI**, with the message
  `activity Heartbeat timeout`, confirms that the activity timed out on its heartbeat timeout, not
  on its StartToClose timeout.
- **Attempt in the Web UI, `Attempt` in the CLI**, shows how many attempts the activity has had.
- **Last Heartbeat older than Last Started Time in the Web UI, `LastHeartbeatTime` older than
  `LastStartedTime` in the CLI**, shows that the current activity attempt has not recorded a
  heartbeat yet ([4.3](#43-the-next-activity-attempt-and-the-timed-out-activity-attempt)).
- **Next Retry in the Web UI, `ScheduledTime` in the CLI**, shows when the next activity
  attempt starts, while the activity waits for it.

**[Section 6](#6-which-recommendations-apply-to-each-cause-of-late-heartbeats) lists the
recommendations for what [5.2](#52-heartbeat-calls-that-the-server-refused) to
[5.5](#55-one-activity-in-detail) found**, by cause, with a link to each recommendation.

---

## 6. Which recommendations apply to each cause of late heartbeats

**[Sections 1 to 4](#contents) explain how heartbeats work, one step at a time, and give each
recommendation in the step it is about.** [Section 5](#5-how-to-find-where-the-heartbeats-were-delayed)
then checks the dashboard panels in order, to find where the heartbeats were delayed. This section
brings the recommendations together by what the dashboard panels showed, and links each cause to
every recommendation that applies.

> **Note:** one fix may not stop all the heartbeat timeouts, because heartbeats can be late for more
> than one reason at the same time. For example, the namespace's rate limit can refuse heartbeat
> calls while some SDK workers are also short of CPU. Raising the namespace's rate limit then stops
> only the heartbeat timeouts that the refused heartbeat calls caused. After each fix, go through
> [5.1](#51-which-activity-types-are-timing-out-on-their-heartbeat-timeout) to
> [5.4](#54-heartbeat-calls-that-the-sdk-worker-sent-late-or-that-failed-before-the-server) again,
> and act on whatever they still show.

**The subsections below follow [section 5](#5-how-to-find-where-the-heartbeats-were-delayed) in
order**, from [5.2](#52-heartbeat-calls-that-the-server-refused) to
[5.5](#55-one-activity-in-detail).

### 6.1 Recommendations for heartbeat calls that the server refused

**When [5.2](#52-heartbeat-calls-that-the-server-refused) found heartbeat calls that the server
refused**, follow the recommendation for each line of [Heartbeat Calls Refused by
Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits) that
shows refusals. Each line is labelled with the service that refused the heartbeat calls, then the
`RESOURCE_EXHAUSTED` cause it gave, for example `history - BusyWorkflow`. The frontend service also
counts the refusals that it passes on from the history service. A `frontend` line with the same
cause as a `history` line therefore shows the same refusals, and needs no other recommendation.

| Line on Heartbeat Calls Refused by Cause | What refused the heartbeat calls | Recommendation |
| --- | --- | --- |
| `frontend - RpsLimit`, with no refusals on `history - RpsLimit` at the same time | The namespace's rate limit | [Raise the namespace's rate limit, or lower the namespace's calls](#rec-namespace-rate-limit), in [2.2](#22-the-namespace-rate-limit-in-the-frontend-service) |
| `history - RpsLimit` | The history service's requests-per-second limits | [Find what uses up `history.rps`, and on server v1.32.0 and later limit the namespace with `history.namespaceRPS`](#rec-history-rps), in [2.4](#24-the-history-services-requests-per-second-limits) |
| `history - BusyWorkflow` | The workflow lock | [Keep down the activities that heartbeat at the same time on one workflow run](#rec-fewer-heartbeating-activities-per-run), in [2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late) |
| `history - PersistenceLimit` | The history service's persistence QPS limits | [Decide whether to raise the persistence QPS limits or lower the history service's database calls](#rec-persistence-limit), in [2.6](#26-what-a-heartbeat-writes-to-the-database-and-the-persistence-qps-limits) |

Heartbeat calls can also reach the server and be slow there without being refused, which
[6.2](#62-recommendations-for-heartbeat-calls-that-were-slow-on-the-server) covers.

### 6.2 Recommendations for heartbeat calls that were slow on the server

**When [5.3](#53-heartbeat-calls-that-were-slow-on-the-server) found heartbeat calls that were slow
on the server**, follow the recommendation for what slowed them:

| What slowed the heartbeat calls | Recommendation |
| --- | --- |
| The workflow lock, with many activities that heartbeat at the same time on one workflow run | [Keep down the activities that heartbeat at the same time on one workflow run](#rec-fewer-heartbeating-activities-per-run), in [2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late) |
| Slow or failing database writes | [Watch for slow and failing database writes](#rec-slow-database-writes), in [2.7](#27-when-the-database-write-of-a-heartbeat-fails) |
| The frontend service, while the history service is fast, for example the frontend service's retry after a `history.rps` refusal | [Find what uses up `history.rps`, and on server v1.32.0 and later limit the namespace with `history.namespaceRPS`](#rec-history-rps), in [2.4](#24-the-history-services-requests-per-second-limits) |
| Heartbeat calls that take close to the time they have, whatever the cause, where a longer heartbeat timeout helps | [Choose the heartbeat timeout by weighing a crashed SDK worker against late heartbeats](#rec-choose-heartbeat-timeout), in [2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late). [Raise the StartToClose timeout with it](#rec-starttoclose-with-heartbeat-timeout), in [3.1](#31-the-heartbeat-deadline). [Give activities that are already running the longer heartbeat timeout](#rec-raise-running-heartbeat-timeout), in [3.5](#35-raising-the-heartbeat-timeout-of-an-activity-that-is-already-running). |

When the server is fast but heartbeats are still late, the delay is on the SDK workers or on the
network, which [6.3](#63-recommendations-for-heartbeat-calls-that-the-sdk-workers-sent-late-or-that-failed-before-the-server)
covers.

### 6.3 Recommendations for heartbeat calls that the SDK workers sent late, or that failed before the server

**When [5.4](#54-heartbeat-calls-that-the-sdk-worker-sent-late-or-that-failed-before-the-server)
found heartbeat calls that the SDK workers sent late, or that failed before the server**, follow the
recommendation for each finding:

| What the SDK workers show | Recommendation |
| --- | --- |
| An SDK worker short of CPU, or Java SDK workers whose 4 heartbeat threads are all busy | [Give each SDK worker enough CPU, or run fewer activities on it at the same time](#rec-sdk-worker-cpu), in [1.3](#13-why-an-sdk-worker-can-send-heartbeats-late) |
| Heartbeat timeouts that keep repeating on an SDK worker short of CPU, because each activity retry adds work | [Run fewer activities per SDK worker first, keep the wait between activity attempts, and limit the attempts](#rec-retries-on-cpu-short-worker), in [4.3](#43-the-next-activity-attempt-and-the-timed-out-activity-attempt) |
| `RESOURCE_EXHAUSTED` on Request Failures while Heartbeat Calls Refused by Cause shows no refusals, for heartbeat details over 4 MB | [Keep heartbeat details small](#rec-small-heartbeat-details), in [2.1](#21-the-grpc-message-size-limit) |
| Heartbeat calls slow on Request Latency but fast on Heartbeat Call Latency (Frontend and History), and the SDK workers are neither short of CPU nor, on the Java SDK, short of free heartbeat threads | The network between the SDK workers and the frontend service, including any load balancer or proxy in front of the frontend service. Check the latency of each in your own network or load balancer monitoring. |

Whatever the cause, an activity without a limit on its attempts keeps timing out until the cause is
fixed, which [6.4](#64-recommendation-for-an-activity-whose-attempt-number-keeps-going-up) covers.

### 6.4 Recommendation for an activity whose attempt number keeps going up

**When [5.5](#55-one-activity-in-detail) shows an activity whose attempt number keeps going up, with
no limit on its attempts**, [set a limit on its attempts](#rec-limit-activity-attempts), as
[4.1](#41-whether-the-server-retries-the-activity-after-a-heartbeat-timeout) recommends. Without a
limit, the server retries the activity with no end, and the workflow waits for it with no end.

**[Section 7](#7-heartbeat-load-on-a-cluster-with-many-activities-that-heartbeat) covers the load that heartbeats put on the
cluster**, which grows with the number of activities that heartbeat.

---

## 7. Heartbeat load on a cluster with many activities that heartbeat

**Each activity that heartbeats adds a steady load on the cluster, even when no heartbeat is late.**
Every heartbeat is one call to the server and one database write. On a namespace replicated to more
than one cluster, each heartbeat also creates a replication task, which carries the heartbeat to the
namespace's standby clusters.

**How much the heartbeat load matters depends on how many activities heartbeat at the same time, and
how short their heartbeat timeouts are.** Heartbeats count against the namespace's rate limit, the
history service's limits and the database's capacity, like any other work on the cluster. With
enough activities that heartbeat at the same time, heartbeats can use up a noticeable share of
those limits.

This section works out how many heartbeat calls, database writes and replication tasks a second a
given number of activities that heartbeat adds, and how their heartbeat timeout changes those
numbers ([7.1](#71-how-much-load-heartbeats-put-on-the-cluster)). It then shows where the heartbeat
load appears on the Temporal Server dashboard
([7.2](#72-where-heartbeat-load-shows-on-the-temporal-server-dashboard)), and how to lower the heartbeat load when
it matters ([7.3](#73-lowering-heartbeat-load)).

### 7.1 How much load heartbeats put on the cluster

**An activity that heartbeats steadily costs the cluster the same four things in every batching
window**, as long as the activity code calls the heartbeat API at least once per batching window:

- **One heartbeat call**, which counts against the namespace's rate limit in the frontend service
  ([2.2](#22-the-namespace-rate-limit-in-the-frontend-service)) and against the history service's
  requests-per-second limits ([2.4](#24-the-history-services-requests-per-second-limits)).
- **One database write for the heartbeat**, which counts against the history service's persistence
  QPS limits ([2.6](#26-what-a-heartbeat-writes-to-the-database-and-the-persistence-qps-limits)).
- **One database write for the timer task that checks the heartbeat deadline**, when the timer task
  finds the heartbeat deadline moved
  ([3.2](#32-how-a-timer-task-checks-the-heartbeat-deadline)).
- **On a namespace replicated to more than one cluster, one replication task**, for the heartbeat
  only. The timer task's write adds no replication task
  ([2.6](#26-what-a-heartbeat-writes-to-the-database-and-the-persistence-qps-limits)).

**The heartbeat load of a cluster is the cost of one activity, from the list above, times the number
of activities that are heartbeating at the same time.** How often the activity code calls the
heartbeat API does not change the heartbeat load, because the SDK sends at most one heartbeat per
batching window ([1.1](#11-the-batching-window)).

For example, 1,000 activities that heartbeat at the same time, each with a 10-second heartbeat
timeout, have an 8-second batching window. Every 8 seconds, each activity costs one heartbeat call
and two database writes, so together they cost:

- **about 125 heartbeat calls a second**, 1,000 every 8 seconds;
- **about 250 database writes a second**, half for the heartbeats and half for the timer tasks;
- **about 125 replication tasks a second**, on a namespace replicated to more than one cluster.

These three rates stay the same whether the activity code calls the heartbeat API every 100
milliseconds or every 5 seconds. What changes them is the heartbeat timeout, because the heartbeat
timeout sets the batching window.

**The heartbeat timeout in each activity's options decides most of the heartbeat load.** It is the
heartbeat timeout that the workflow code sets when it schedules the activity, or that the Temporal
CLI command `temporal activity update-options` sets on a running activity
([3.5](#35-raising-the-heartbeat-timeout-of-an-activity-that-is-already-running)). Up to a heartbeat
timeout of 75 seconds, the batching window is 80% of the heartbeat timeout
([1.1](#11-the-batching-window)), so a longer heartbeat timeout lowers all four costs together.
Doubling the heartbeat timeout halves the heartbeat calls, the heartbeat writes, the timer task
writes and the replication tasks.

**Above a heartbeat timeout of 75 seconds, a longer heartbeat timeout lowers only the timer task
writes:**

- **The heartbeat calls, the heartbeat writes and the replication tasks stay at about one a minute
  per activity**, because the SDK's batching window stays at 60 seconds.
- **The timer task writes fall to about one per heartbeat timeout.** The history service makes these
  writes when it processes the timer task that checks the heartbeat deadline. The timer task is due
  one heartbeat timeout after the last recorded heartbeat
  ([3.2](#32-how-a-timer-task-checks-the-heartbeat-deadline)), so it finds the heartbeat deadline
  moved about once per heartbeat timeout, not once per batching window. With a 5-minute heartbeat
  timeout, the history service writes for the timer task about once every 5 minutes.

The fall in timer task writes above 75 seconds is worked out from how the timer task works, and no
test measured it.

> **Recommendation:** do not raise a heartbeat timeout only to lower the heartbeat load. A longer
> heartbeat timeout also makes the server notice a crashed SDK worker later. As
> [2.5](#25-how-waiting-for-the-workflow-lock-can-make-heartbeats-late) already recommends,
> [choose each heartbeat timeout for how quickly the activity needs a crash
> noticed](#rec-choose-heartbeat-timeout). [7.3](#73-lowering-heartbeat-load) covers the other ways
> to lower the heartbeat load.

**Worked example:** 100,000 activities that heartbeat at the same time, with the same heartbeat
timeout. The numbers in the table below are rounded, and come from the rules above, not from a
measurement.

| Heartbeat timeout | Batching window | Heartbeat calls and heartbeat writes per second | Timer task writes per second | Replication tasks written per second, on a namespace with standby clusters |
| --- | --- | --- | --- | --- |
| 10 seconds | 8 seconds | about 12,500 | about 12,500 | about 12,500 |
| 60 seconds | 48 seconds | about 2,100 | about 2,100 | about 2,100 |
| 5 minutes | 60 seconds | about 1,700 | about 330 | about 1,700 |

On a namespace with standby clusters, the active cluster writes one replication task per heartbeat,
however many standby clusters the namespace has, and each standby cluster receives it.

The rates in the table above hold while the history service runs steadily. Restarting a history
service instance, for example during a server deploy, briefly changes how often the history service
processes timer tasks.

**A history service restart makes the history service process more timer tasks for a while, but
adds no database writes.** When a history service instance stops, other instances take over its
shards and load their timer tasks again, so some timer tasks are processed twice.

> **Test cluster result:** a graceful restart of one history service instance re-ran about 20% more
> timer tasks for heartbeat deadlines, with no more database writes and no failed heartbeat calls.
>
> - **Goal:** find out what a history service restart costs activities that heartbeat.
> - **Setup:** server v1.31.0 with two history service instances, Go SDK v1.48.0 and PostgreSQL.
>   100 workflows, each running one activity with a 10-second heartbeat timeout that called the
>   heartbeat API every 500 milliseconds for 120 seconds. One history service instance was restarted
>   with a graceful stop 40 seconds into the run.
> - **All 100 activities finished, and all 1,600 heartbeat calls succeeded**, with no heartbeat
>   timeout. Heartbeat call latency at p99 reached 80 milliseconds for a few seconds while the shards
>   moved.
> - **The timer tasks that check activity deadlines ran about 1,797 times**, against 1,500 in the
>   same run without restarting a history service instance.
> - **Writes of the `UpdateWorkflowExecution` persistence operation did not rise**, about 3,400
>   against 3,600 without restarting a history service instance.
> - **Not tested:** a history service instance that crashes without a graceful stop.

**A rise in timer task processing during a history service deploy is therefore expected, and is not
a heartbeat problem.** On [Timer Tasks Processed by
Type](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info), the
`TimerActiveTaskActivityTimeout` line rises while the shards of the restarted instance are loaded
again. Heartbeat calls, database writes and heartbeat timeouts do not change.

> **Recommendation:** restart history service instances one at a time, each with a graceful stop, as
> the [rollout step](./server-upgrade.md#step-3-roll-out-the-new-version) of the Server Upgrade
> playbook describes. The test above covered only a graceful restart. A history
> service instance killed before its shutdown finished was not tested.

**The load from heartbeats is now worked out.**
[7.2](#72-where-heartbeat-load-shows-on-the-temporal-server-dashboard) shows where it appears on the Temporal Server
dashboard, so that the reader can see how much of the cluster's load comes from heartbeats.

### 7.2 Where heartbeat load shows on the Temporal Server dashboard

**The Temporal Server dashboard shows each cost from
[7.1](#71-how-much-load-heartbeats-put-on-the-cluster) on its own panel.** Each panel below shows the
namespace selected at the top of the dashboard, except the replication panel, which shows the whole
cluster:

- **[Activity Heartbeat
  Calls](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)** shows
  the heartbeat calls per second that reached the history service. It counts calls, not database
  writes. No panel counts heartbeat writes on their own, because a heartbeat write is an
  `UpdateWorkflowExecution` persistence operation, like every other write to a running workflow.
  The history service makes one write for each heartbeat it records, so the heartbeat writes are
  about as many as the heartbeat calls, minus any calls that the history service refused.
- **[Timer Tasks Processed by
  Type](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)**,
  under `TimerActiveTaskActivityTimeout`, shows how often the history service processes the timer
  tasks that check activity deadlines. The line also counts the checks of an activity's other
  timeouts, such as its StartToClose timeout, so it is somewhat higher than the timer task writes
  that heartbeats cause.
- **[Heartbeat Details
  Size](../observability/dashboards/server/temporal-server-readme.md#12-workflow-history-state-and-payload-sizes)**
  shows the bytes of heartbeat details per second, and the average per heartbeat call, on server
  v1.30.0 and later.
- **[Replication Tasks Sent per Second by Task
  Type](../observability/dashboards/server/temporal-server-readme.md#18-cluster-replication)** shows
  the replication tasks that the active cluster sends to each standby cluster. On server v1.30.0 and
  later, a heartbeat's replication task has the same task type as the replication tasks of other
  workflow changes, so the panel cannot show the heartbeats' share.

**To see what share of the namespace's database writes come from heartbeats, follow these steps for
the same time range:**

1. On [Activity Heartbeat
   Calls](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits), read
   the heartbeat calls per second.
2. Multiply that number by 2, when the heartbeat timeouts are 75 seconds or shorter. Each heartbeat
   call then causes about two database writes, one for the heartbeat and one for the timer task that
   checks the heartbeat deadline ([3.2](#32-how-a-timer-task-checks-the-heartbeat-deadline)). With
   longer heartbeat timeouts, the timer task writes to the database less often
   ([7.1](#71-how-much-load-heartbeats-put-on-the-cluster)). Multiply by 1.5 for a 2-minute
   heartbeat timeout, and by 1.2 for a 5-minute heartbeat timeout.
3. On [Persistence Requests per Namespace and
   Operation](../observability/dashboards/server/temporal-server-readme.md#3-persistence-requests-latencies-and-errors),
   read the writes per second of the `UpdateWorkflowExecution` persistence operation. The
   `UpdateWorkflowExecution` persistence operation counts every write to the namespace's running
   workflows.
4. Compare the number from step 2 with the number from step 3.

**For example:** Activity Heartbeat Calls shows 500 calls a second, so heartbeats cause about 1,000
writes a second. Persistence Requests per Namespace and Operation shows 1,500 writes a second of the
`UpdateWorkflowExecution` persistence operation. So about 1,000 of the namespace's 1,500 writes a
second come from heartbeats.

**Where heartbeat load shows on the Temporal Server dashboard is now covered.** When heartbeats cause
a large share of the cluster's work, [7.3](#73-lowering-heartbeat-load) covers how to lower the
heartbeat load, and what each way of lowering it costs.

### 7.3 Lowering heartbeat load

**Four changes lower the heartbeat load, and each change has a cost of its own.** They are listed in
order, from the change with the least effect on how activities that heartbeat behave, to the change
with the most:

- **Heartbeat only from activities that need heartbeats.** The [opening of this
  playbook](#activity-heartbeat-timeouts--finding-the-cause-and-tuning-playbook) gives two reasons
  an activity needs heartbeats. One is noticing a crashed SDK worker before the StartToClose timeout.
  The other is learning about a cancellation, pause or reset while the activity attempt runs. An
  activity that needs neither costs heartbeat load for nothing.
  - **An activity with no heartbeat timeout still sends heartbeats when its code calls the heartbeat
    API**, every 30 seconds by default, set by the worker option
    `DefaultHeartbeatThrottleInterval` in the Go and Java SDKs. Each of those heartbeats is still a
    heartbeat call and a database write, but the activity attempt never times out on a heartbeat
    timeout, because it has none.
- **Keep heartbeat details small.** Smaller heartbeat details make each heartbeat write and each
  replication task smaller, but do not lower the number of heartbeat calls, writes or replication
  tasks. [2.1](#rec-small-heartbeat-details) gives the recommendation.
- **Use a longer heartbeat timeout, where the activity can afford it.** Up to a heartbeat timeout of
  75 seconds, a longer heartbeat timeout lowers the heartbeat calls, heartbeat writes, timer task
  writes and replication tasks. Above 75 seconds, it lowers only the timer task writes
  ([7.1](#71-how-much-load-heartbeats-put-on-the-cluster)). It costs the activity:
  - **A crashed SDK worker is noticed later**, one heartbeat timeout after its last heartbeat
    ([4.4](#44-how-long-after-the-heartbeats-stop-the-server-times-out-the-activity)).
  - **Up to 75 seconds, a cancellation, pause or reset is noticed later too**, because the batching
    window grows with the heartbeat timeout ([1.1](#11-the-batching-window)).
- **Raise `MaxHeartbeatThrottleInterval` on the SDK workers, for activities with heartbeat timeouts
  above 75 seconds.** This worker option caps the batching window at 60 seconds by default
  ([1.1](#11-the-batching-window)). Raising it lets the batching window grow to 80% of the heartbeat
  timeout again, so the SDK sends fewer heartbeats. For example, with a 5-minute heartbeat timeout
  and `MaxHeartbeatThrottleInterval` at 4 minutes, the SDK sends a heartbeat every 4 minutes
  instead of every minute. It costs every activity on those SDK workers:
  - **A cancellation, pause or reset is noticed later**, up to one batching window later.
  - **More heartbeat details can be lost when an SDK worker crashes**, up to one batching window of
    them ([1.1](#11-the-batching-window)).

**When the heartbeat load already reaches one of the server's limits, the server refuses heartbeat
calls.** The server's limits for heartbeat calls are:

- the namespace's rate limit in the frontend service
  ([2.2](#22-the-namespace-rate-limit-in-the-frontend-service));
- the history service's requests-per-second limits
  ([2.4](#24-the-history-services-requests-per-second-limits));
- the history service's persistence QPS limits
  ([2.6](#26-what-a-heartbeat-writes-to-the-database-and-the-persistence-qps-limits)).

[5.2](#52-heartbeat-calls-that-the-server-refused) finds which of these limits refuses the heartbeat
calls, and [6.1](#61-recommendations-for-heartbeat-calls-that-the-server-refused) gives the
recommendation for each limit. For the persistence QPS limits, [History Persistence QPS
Limits](./history-persistence-qps-limits.md) covers how to decide whether to raise them or lower the
history service's database writes.

> **Recommendation:** to lower the heartbeat load, first stop heartbeats from activities that do not
> need them, and keep heartbeat details small. Change heartbeat timeouts or
> `MaxHeartbeatThrottleInterval` only for activities that can afford to have a crashed SDK worker,
> a cancellation, a pause or a reset noticed later.

**The heartbeat load on the cluster, and how to lower the heartbeat load, is now covered.**
[Section 8](#8-alerting-on-heartbeat-timeouts) covers alerts on heartbeat timeouts, so that a rise in
heartbeat timeouts is noticed when it starts, not when workflows are already late.

---

## 8. Alerting on heartbeat timeouts

**This section covers the alerts on heartbeat timeouts:**

- the server alert on heartbeat timeouts, and what to do when it fires
  ([8.1](#81-the-alert-on-heartbeat-timeouts));
- the SDK alerts that show the SDK workers' side of the same problem
  ([8.2](#82-sdk-alerts-for-the-sdk-workers-side-of-heartbeat-timeouts));
- why refused heartbeat calls, heartbeat load and late timer tasks have no alert
  ([8.3](#83-why-refused-heartbeat-calls-heartbeat-load-and-late-timer-tasks-have-no-alert)).

### 8.1 The alert on heartbeat timeouts

**Alert 55, [Activity Heartbeat
Timeouts](../observability/alerts/server/runbooks/55-activity-heartbeat-timeouts.md), fires when
activities in a namespace time out on their heartbeat timeout for at least 10 minutes, with no
gap of 5 minutes or more between heartbeat timeouts.** It is a
warning, one alert for each namespace. It counts the same heartbeat timeouts as [Activity Heartbeat
Timeout (Every
Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info), so it
counts every timed-out attempt, also the ones that the server retries
([4.5](#45-where-heartbeat-timeouts-show-on-the-dashboard-and-in-temporal-workflow-describe)). It
needs server v1.29.0 or later.

**A crashed SDK worker usually does not keep alert 55 firing.** When an SDK worker crashes, each
activity it was running times out once, and its next activity attempt runs on another SDK worker.
Heartbeat timeouts that go on for 10 minutes mean that heartbeats keep arriving late, which is what
[section 5](#5-how-to-find-where-the-heartbeats-were-delayed) finds the cause of.

> **Test cluster result:** alert 55 fired about 10 minutes after the first heartbeat timeout.
>
> - **Goal:** check that alert 55 fires on heartbeat timeouts that the server retries.
> - **Setup:** server v1.32.2, Go SDK v1.48.0 and PostgreSQL. One activity with a 10-second
>   heartbeat timeout stopped calling the heartbeat API 2 seconds into every attempt. Its retry
>   policy had no limit on attempts, with the other fields at their defaults.
> - **The first heartbeat timeout came 10 seconds after the last recorded heartbeat**, 18 seconds
>   after the activity started. The SDK sent that last heartbeat at the end of its 8-second batching
>   window. The waits between attempts then grew from 1 second to 100 seconds, as the default retry
>   policy sets them.
> - **Alert 55 went pending one minute later, and fired 10 minutes after that**, after 11 heartbeat
>   timeouts. No workflow failed during that time.
> - **Alert 55 cleared by itself about 4 minutes after the workflow was terminated**, once no more
>   heartbeat timeouts came.

**Alert 55 does not tell you where the heartbeats were delayed.** It sums the heartbeat timeouts of
all activity types in the namespace. [Section 5](#5-how-to-find-where-the-heartbeats-were-delayed)
finds which activity types time out, and where their heartbeats were delayed.
[Section 6](#6-which-recommendations-apply-to-each-cause-of-late-heartbeats) gives the
recommendation for each place where heartbeats were delayed.

**On servers older than v1.29.0, alert 55 never fires**, because servers before v1.29.0 do not
count heartbeat timeouts that the server retries. On servers before v1.29.0, read [Activity Heartbeat Timeout
(Final
Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info), which
counts only heartbeat timeouts that end the activity.

The SDK workers also see heartbeat timeouts, from their side, which
[8.2](#82-sdk-alerts-for-the-sdk-workers-side-of-heartbeat-timeouts) covers.

### 8.2 SDK alerts for the SDK workers' side of heartbeat timeouts

**Two SDK alerts fire on heartbeat calls that failed, as the SDK workers see them.** Both read
Request Failures on the SDK dashboards
([Go](../observability/dashboards/sdk/temporal-sdk-go-readme.md#1-grpc-requests-and-request-latencies),
[Java
Micrometer](../observability/dashboards/sdk/temporal-sdk-java-micrometer-readme.md#1-grpc-requests-and-request-latencies),
[Java
OpenTelemetry](../observability/dashboards/sdk/temporal-sdk-java-otel-readme.md#1-grpc-requests-and-request-latencies)).
Alerts 1c and 2d are warnings that fire when the failures go on for at least 5 minutes:

- **[Alert 1c, `NOT_FOUND` on
  `RecordActivityTaskHeartbeat`](../observability/alerts/sdk/runbooks/01c-not-found-heartbeat.md)**,
  fires when the server answers heartbeat calls with `NOT_FOUND`. The server answers `NOT_FOUND` in
  two cases:
  - **The activity attempt that sent the heartbeat call is no longer running.** The server has timed
    that activity attempt out on its heartbeat timeout or its StartToClose timeout
    ([1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails)).
  - **The workflow execution is no longer running.** It has completed, failed, timed out, been
    terminated or canceled, or continued as new.
- **[Alert 2d, `RESOURCE_EXHAUSTED` on other
  operations](../observability/alerts/sdk/alerts-index.md#alert-2d--request-failure-resource_exhausted-on-other-operations)**,
  fires when the server refuses SDK calls, heartbeat calls among them. It also covers many other
  operations, so check Request Failures for `RecordActivityTaskHeartbeat` before treating it as a
  heartbeat problem. The refused heartbeat calls are the ones that
  [5.2](#52-heartbeat-calls-that-the-server-refused) finds, or calls over 4 MB that the gRPC message
  size limit refused ([2.1](#21-the-grpc-message-size-limit)). Alert 2d is documented in the SDK
  alerts index, but the SDK alert rule files do not include it yet.

**Alert 1c and alert 55 do not catch the same activity attempts:**

- **Alert 1c misses heartbeat timeouts after which the timed-out activity attempt makes no more
  heartbeat calls**, for example because its SDK worker crashed, or its activity code stopped calling
  the heartbeat API. Alert 1c counts only `NOT_FOUND` answers to heartbeat calls.
- **Alert 1c can also fire with no heartbeat timeout at all.** A heartbeat call also gets
  `NOT_FOUND` when its activity attempt already timed out on its StartToClose timeout, or when its
  workflow execution is no longer running. When alert 1c fires while [Activity Heartbeat Timeout
  (Every Attempt)](../observability/dashboards/server/temporal-server-readme.md#14-sdk-workers-info)
  shows no heartbeat timeouts, the `NOT_FOUND` answers come from StartToClose timeouts or from
  workflow executions that are no longer running, which this playbook does not cover.
- **Alert 55 counts every heartbeat timeout on the server**, whichever SDK the SDK workers use.

> **Recommendation:** use alert 55 as the alert on heartbeat timeouts, and when you collect SDK
> metrics, also deploy alert 1c and build alert 2d from the [query in the SDK alerts
> index](../observability/alerts/sdk/alerts-index.md#alert-2d--request-failure-resource_exhausted-on-other-operations):
>
> - **Act on alert 55 first.** It counts every heartbeat timeout, whichever SDK the SDK workers use,
>   and [section 5](#5-how-to-find-where-the-heartbeats-were-delayed) starts from the panel it
>   reads.
> - **Keep alert 1c for what alert 55 does not show.** Alert 1c shows SDK workers still running
>   activity attempts that the server has already timed out, which run at the same time as the next
>   attempt ([1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails)). It also shows activity
>   attempts that timed out on their StartToClose timeout.
> - **Keep alert 2d for heartbeat calls over 4 MB, on SDKs that do not check the heartbeat details
>   before sending them ([2.1](#21-the-grpc-message-size-limit)).** Alert 55 fires on the heartbeat
>   timeouts that those calls cause, but no server panel shows the cause, because the gRPC message
>   size limit refuses the calls before the server counts them. Only alert 2d shows the refused
>   heartbeat calls. On SDKs that do check, neither alert fires, and
>   [2.3](#23-the-payload-size-limit) recommends alerting on the SDK workers' logs instead.
> - **On servers before v1.29.0, rely on alerts 1c and 2d**, because alert 55 never fires there.
>   Alert 1c still misses heartbeat timeouts after which the timed-out activity attempt makes no
>   more heartbeat calls.

Refused heartbeat calls, heartbeat load and late timer tasks have no alert of their own, which
[8.3](#83-why-refused-heartbeat-calls-heartbeat-load-and-late-timer-tasks-have-no-alert) explains.

### 8.3 Why refused heartbeat calls, heartbeat load and late timer tasks have no alert

**This playbook adds no alert on three things that it covers, on purpose, although a dashboard
panel shows each of them.** Refused heartbeat calls and heartbeat load matter only when they lead to
heartbeat timeouts, and alert 55 fires on those heartbeat timeouts. Late timer tasks never lead to a
heartbeat timeout:

- **Heartbeat calls that the server refused**, on [Heartbeat Calls Refused by
  Cause](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)
  ([5.2](#52-heartbeat-calls-that-the-server-refused)). The Go and Java SDKs retry a refused
  heartbeat call ([1.2](#12-what-the-sdks-do-when-a-heartbeat-call-fails)). A refusal alone does not
  time out the activity when the retry gets through before the heartbeat deadline. When refusals do
  make heartbeats late, the activities time out on their heartbeat timeout, and alert 55 fires.
- **Heartbeat load on the cluster**, on [Activity Heartbeat
  Calls](../observability/dashboards/server/temporal-server-readme.md#6-throttling-and-limits)
  ([section 7](#7-heartbeat-load-on-a-cluster-with-many-activities-that-heartbeat)). Heartbeat load
  causes no heartbeat timeouts until it reaches one of the server's limits. The server then refuses
  heartbeat calls, which the bullet above covers.
- **Timer tasks processed late**, on [Timer Task Delay After Due Time by
  Type](../observability/dashboards/server/temporal-server-readme.md#10-history-timer-task-info)
  ([3.3](#33-a-timer-task-processed-late-never-causes-an-early-heartbeat-timeout)). A timer task
  processed late gives the activity more time, never less, so it never causes a heartbeat timeout.

**When alert 55 fires, start at [section 5](#5-how-to-find-where-the-heartbeats-were-delayed)**,
which finds where the heartbeats were delayed. Then follow the recommendations that
[section 6](#6-which-recommendations-apply-to-each-cause-of-late-heartbeats) gives for what it found.
