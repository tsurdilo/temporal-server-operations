# Temporal Server Metrics Reference

This document lists all OSS Temporal server metrics defined in `metrics_defs.go`, organized by category. Metric types: **Counter**, **Gauge**, **Timer** (latency histogram), **Histogram** (dimensionless or bytes).

---

## Table of Contents

- [Common / Service Base](#common--service-base)
- [RPC & Client](#rpc--client)
- [Persistence](#persistence)
- [Visibility Persistence](#visibility-persistence)
- [Frontend](#frontend)
- [History](#history)
- [Task Processing](#task-processing)
- [Replication](#replication)
- [Matching](#matching)
- [Worker / Archival](#worker--archival)
- [Schedules](#schedules)
- [Worker Versioning & Deployments](#worker-versioning--deployments)
- [Workflow Execution](#workflow-execution)
- [Caching](#caching)
- [Namespace Registry](#namespace-registry)
- [Nexus](#nexus)
- [Deadlock Detector](#deadlock-detector)
- [Elasticsearch / Bulk Processor](#elasticsearch--bulk-processor)
- [Delete Namespace / Executions](#delete-namespace--executions)
- [Runtime / Go Process](#runtime--go-process)

---

## Common / Service Base

| Metric | Type | Description |
|--------|------|-------------|
| `service_requests` | Counter | Number of RPC requests received by the service. |
| `service_pending_requests` | Gauge | Currently pending RPC requests. |
| `service_errors` | Counter | Number of unexpected service request errors. |
| `service_panics` | Counter | Number of service panics. |
| `service_error_with_type` | Counter | All service request errors, keyed by `error_type`. |
| `service_grpc_conn_accepted` | Counter | Number of gRPC TCP connections accepted. |
| `service_grpc_conn_closed` | Counter | Number of gRPC TCP connections closed. |
| `service_grpc_conn_active` | Gauge | Current number of active gRPC TCP connections. |
| `service_dial_latency` | Timer | Latency of establishing a new TCP connection. |
| `service_dial_success` | Counter | TCP dial attempts that successfully established a connection. |
| `service_dial_error` | Counter | TCP dial attempts that failed to establish a connection. |
| `service_latency` | Timer | Service request latency. |
| `service_latency_nouserlatency` | Timer | Service latency excluding user latency. |
| `service_latency_userlatency` | Timer | User-contributed latency portion of service requests. |
| `service_errors_invalid_argument` | Counter | Errors due to invalid arguments. |
| `service_errors_namespace_not_active` | Counter | Errors due to namespace not active. |
| `service_errors_resource_exhausted` | Counter | Errors due to resource exhaustion. |
| `service_errors_entity_not_found` | Counter | Errors due to entity not found. |
| `service_errors_execution_already_started` | Counter | Errors due to execution already started. |
| `service_errors_context_timeout` | Counter | Errors due to context timeout. |
| `service_errors_retry_task` | Counter | Retry task errors. |
| `service_errors_incomplete_history` | Counter | Errors due to incomplete history. |
| `service_errors_nondeterministic` | Counter | Non-determinism errors. |
| `service_errors_unauthorized` | Counter | Unauthorized errors. |
| `service_errors_authorize_failed` | Counter | Authorization failure errors. |
| `service_errors_shard_ownership_lost` | Counter | Shard ownership lost errors. |
| `service_authorization_latency` | Timer | Latency of authorization checks. |
| `namespace_rate_limit_poll_wait_latency` | Timer | Latency waiting due to namespace rate limiting. |
| `action` | Counter | Generic action counter. |
| `operation` | Counter | Generic operation counter. |
| `certificates_expired` | Gauge | Number of expired TLS certificates. |
| `certificates_expiring` | Gauge | Number of TLS certificates expiring soon. |
| `dynamic_config_update_failure` | Gauge | Dynamic config update failures. |
| `restarts` | Counter | Service restart count. |
| `lock_requests` | Counter | Lock request count. |
| `lock_latency` | Timer | Lock acquisition latency. |
| `semaphore_requests` | Counter | Semaphore request count. |
| `semaphore_failures` | Counter | Semaphore failure count. |
| `semaphore_latency` | Timer | Semaphore acquisition latency. |
| `total_namespaces` | Gauge | Total number of namespaces. |
| `read_namespace_errors` | Counter | Errors reading namespace data. |
| `host_rps_limit` | Gauge | Host-level RPS limit. |
| `namespace_host_rps_limit` | Gauge | Per-namespace host RPS limit. |
| `handover_wait_latency` | Timer | Latency waiting for namespace handover. |

---

## RPC & Client

| Metric | Type | Description |
|--------|------|-------------|
| `client_requests` | Counter | Requests sent by the client to an individual service, keyed by `service_role` and `operation`. |
| `client_errors` | Counter | Client-side errors. |
| `client_latency` | Timer | Client request latency. |
| `client_redirection_requests` | Counter | Requests that were redirected by the DC redirection client. |
| `client_redirection_errors` | Counter | Errors from redirected requests. |
| `client_redirection_latency` | Timer | Latency of redirected requests. |
| `http_service_requests` | Counter | Number of HTTP requests received by the service. |

---

## Persistence

| Metric | Type | Description |
|--------|------|-------------|
| `persistence_requests` | Counter | Persistence requests, keyed by `operation`. |
| `persistence_errors` | Counter | Persistence errors. |
| `persistence_error_with_type` | Counter | Persistence errors, keyed by `error_type`. |
| `persistence_latency` | Timer | Persistence latency, keyed by `operation`. |
| `persistence_shard_rps` | Histogram | Per-shard persistence RPS. |
| `persistence_errors_resource_exhausted` | Counter | Persistence resource exhausted errors. |
| `cassandra_init_session_latency` | Timer | Cassandra session initialization latency. |
| `cassandra_session_refresh_failures` | Counter | Cassandra session refresh failures. |
| `persistence_session_refresh_failures` | Counter | Persistence session refresh failures. |
| `persistence_session_refresh_attempts` | Counter | Persistence session refresh attempts. |
| `persistence_sql_max_open_conn` | Gauge | SQL persistence max open connections. |
| `persistence_sql_open_conn` | Gauge | SQL persistence open connections. |
| `persistence_sql_idle_conn` | Gauge | SQL persistence idle connections. |
| `persistence_sql_in_use` | Gauge | SQL persistence connections in use. |

---

## Visibility Persistence

| Metric | Type | Description |
|--------|------|-------------|
| `visibility_persistence_requests` | Counter | Visibility persistence requests. |
| `visibility_persistence_error_with_type` | Counter | Visibility persistence errors by type. |
| `visibility_persistence_errors` | Counter | Visibility persistence errors. |
| `visibility_persistence_resource_exhausted` | Counter | Visibility persistence resource exhausted errors. |
| `visibility_persistence_latency` | Timer | Visibility persistence latency. |

---

## Frontend

| Metric | Type | Description |
|--------|------|-------------|
| `add_search_attributes_workflow_success` | Counter | Successful AddSearchAttributes workflow completions. |
| `add_search_attributes_workflow_failure` | Counter | Failed AddSearchAttributes workflow completions. |
| `version_check_success` | Counter | Successful version check count. |
| `version_check_failed` | Counter | Failed version checks. |
| `version_check_request_failed` | Counter | Version check requests that failed. |
| `version_check_latency` | Timer | Version check latency. |

---

## History

| Metric | Type | Description |
|--------|------|-------------|
| `event_blob_size` | Histogram (bytes) | Size of event blobs. |
| `blob_size_error` | Counter | Requests failing due to blob size exceeding limits. |
| `header_size` | Histogram (bytes) | Size of request headers passed by clients. |
| `state_transition_count` | Histogram | Number of state transitions per workflow execution. |
| `history_size` | Histogram (bytes) | Workflow history size. |
| `history_count` | Histogram | Number of history events. |
| `search_attributes_size` | Histogram (bytes) | Search attributes size. |
| `memo_size` | Histogram (bytes) | Workflow memo size. |
| `history_event_notification_queueing_latency` | Timer | Latency for queueing history event notifications. |
| `history_event_notification_fanout_latency` | Timer | Latency for fanning out history event notifications. |
| `history_event_notification_inflight_message_gauge` | Gauge | In-flight history event notification messages. |
| `history_event_notification_fail_delivery_count` | Counter | History event notification delivery failures. |
| `host_health` | Gauge | History host health gauge. |
| `archival_task_invalid_uri` | Counter | Archival tasks with invalid URIs. |
| `archiver_archive_latency` | Timer | Archiver archive latency. |
| `archiver_archive_target_latency` | Timer | Archiver target archive latency. |
| `shard_closed_count` | Counter | Shard context closed events. |
| `sharditem_created_count` | Counter | Shard context creation events. |
| `sharditem_removed_count` | Counter | Shard context removal events. |
| `sharditem_acquisition_latency` | Timer | Shard acquisition latency. |
| `shardinfo_immediate_queue_lag` | Histogram | Lag between pending immediate task ID and last generated task ID, per shard. |
| `shardinfo_scheduled_queue_lag` | Timer | Lag between earliest scheduled pending task time and now, per shard. |
| `syncshard_remote_count` | Counter | Remote shard sync count. |
| `syncshard_remote_failed` | Counter | Failed remote shard syncs. |
| `finalizer_runs` | Counter | Number of finalizer runs. |
| `finalizer_run_timeouts` | Counter | Number of finalizer run timeouts. |
| `finalizer_items_completed` | Counter | Finalizer items completed successfully. |
| `finalizer_items_unfinished` | Counter | Finalizer items aborted before completion. |
| `finalizer_latency` | Timer | Finalizer latency. |
| `wf_too_many_pending_child_workflows` | Counter | Workflow tasks failed due to too many pending child workflows. |
| `wf_too_many_pending_activities` | Counter | Workflow tasks failed due to too many pending activities. |
| `wf_too_many_pending_cancel_requests` | Counter | Workflow tasks failed due to too many pending cancel requests. |
| `wf_too_many_pending_external_workflow_signals` | Counter | Workflow tasks failed due to too many pending external signals. |
| `mutable_state_size` | Histogram (bytes) | Size of an individual workflow execution's in-memory state. |
| `persisted_mutable_state_size` | Histogram (bytes) | Size of persisted workflow execution state in DB. |
| `execution_info_size` | Histogram (bytes) | Execution info size. |
| `execution_state_size` | Histogram (bytes) | Execution state size. |
| `activity_info_size` | Histogram (bytes) | Activity info size. |
| `timer_info_size` | Histogram (bytes) | Timer info size. |
| `child_info_size` | Histogram (bytes) | Child execution info size. |
| `request_cancel_info_size` | Histogram (bytes) | Request cancel info size. |
| `signal_info_size` | Histogram (bytes) | Signal info size. |
| `signal_request_id_size` | Histogram (bytes) | Signal request ID size. |
| `buffered_events_size` | Histogram (bytes) | Buffered events size. |
| `chasm_total_size` | Histogram (bytes) | CHASM total size. |
| `activity_info_count` | Histogram | Activity info count. |
| `timer_info_count` | Histogram | Timer info count. |
| `child_info_count` | Histogram | Child execution info count. |
| `signal_info_count` | Histogram | Signal info count. |
| `request_cancel_info_count` | Histogram | Request cancel info count. |
| `signal_request_id_count` | Histogram | Signal request ID count. |
| `buffered_events_count` | Histogram | Buffered events count. |
| `task_count` | Histogram | Task count. |
| `total_activity_count` | Histogram | Total activity count. |
| `total_user_timer_count` | Histogram | Total user timer count. |
| `total_child_execution_count` | Histogram | Total child execution count. |
| `total_request_cancel_external_count` | Histogram | Total external cancel request count. |
| `total_signal_external_count` | Histogram | Total external signal count. |
| `total_signal_count` | Histogram | Total signal count. |
| `ack_level_update` | Counter | Ack level update events. |
| `ack_level_update_failed` | Counter | Ack level update failures. |
| `command` | Counter | Workflow commands processed. |
| `request_workflow_update_message` | Counter | Request workflow update messages. |
| `accept_workflow_update_message` | Counter | Accept workflow update messages. |
| `respond_workflow_update_message` | Counter | Respond workflow update messages. |
| `reject_workflow_update_message` | Counter | Reject workflow update messages. |
| `workflow_update_registry_size` | Histogram (bytes) | Update registry size per workflow execution. |
| `workflow_update_registry_size_limited` | Counter | Times the update registry size was limited. |
| `workflow_update_request_rate_limited` | Counter | Update requests that were rate limited. |
| `business_id_reuse_rate_limited` | Counter | Business ID reuse rate limited events. |
| `workflow_update_request_too_many` | Counter | Update requests rejected due to too many pending updates. |
| `workflow_update_aborted` | Counter | Aborted workflow updates. |
| `workflow_update_sent_to_worker` | Counter | Updates sent to worker. |
| `workflow_update_sent_to_worker_again` | Counter | Updates re-sent to worker. |
| `workflow_update_wait_stage_accepted` | Counter | Updates waiting at accepted stage. |
| `workflow_update_wait_stage_completed` | Counter | Updates waiting at completed stage. |
| `workflow_update_client_timeout` | Counter | Update requests that timed out on the client side. |
| `workflow_update_server_timeout` | Counter | Update requests that timed out on the server side. |
| `speculative_workflow_task_commits` | Counter | Speculative workflow task commits. |
| `speculative_workflow_task_rollbacks` | Counter | Speculative workflow task rollbacks. |
| `activity_eager_execution` | Counter | Activity eager execution count. |
| `workflow_eager_execution` | Counter | Eager workflow start requests. |
| `workflow_eager_execution_denied` | Counter | Eager workflow starts that fell back to standard dispatch. Tagged with `reason`. |
| `start_workflow_request_deduped` | Counter | Deduplicated start workflow requests. |
| `empty_completion_commands` | Counter | Empty completion commands. |
| `multiple_completion_commands` | Counter | Multiple completion commands. |
| `failed_workflow_tasks` | Counter | Failed workflow tasks. |
| `workflow_task_attempt` | Histogram | Workflow task attempt distribution. |
| `stale_mutable_state` | Counter | Stale mutable state detections. |
| `auto_reset_points_exceed_limit` | Counter | Auto-reset points exceeding the limit. |
| `auto_reset_point_corruption` | Counter | Corrupted auto-reset points. |
| `batchable_task_batch_count` | Gauge | Batchable task batch count. |
| `concurrency_update_failure` | Counter | Concurrency update failures. |
| `heartbeat_timeout` | Counter | Activity heartbeat timeouts. |
| `schedule_to_start_timeout` | Counter | Schedule-to-start timeouts. |
| `start_to_close_timeout` | Counter | Start-to-close timeouts. |
| `schedule_to_close_timeout` | Counter | Schedule-to-close timeouts. |
| `new_timer_notifications` | Counter | New timer notification count. |
| `state_machine_timer_processing_failures` | Counter | State machine timer processing failures. |
| `state_machine_timer_skips` | Counter | State machine timer skips. |
| `acquire_shards_count` | Counter | Shard acquisition attempts. |
| `acquire_shards_latency` | Timer | Shard acquisition latency. |
| `membership_changed_count` | Counter | Membership change events. |
| `numshards_gauge` | Gauge | Number of shards owned. |
| `get_engine_for_shard_errors` | Counter | Errors getting engine for a shard. |
| `get_engine_for_shard_latency` | Timer | Latency getting engine for a shard. |
| `remove_engine_for_shard_latency` | Timer | Latency removing engine for a shard. |
| `complete_workflow_task_sticky_enabled_count` | Counter | Workflow task completions with sticky enabled. |
| `complete_workflow_task_sticky_disabled_count` | Counter | Workflow task completions with sticky disabled. |
| `workflow_task_heartbeat_timeout_count` | Counter | Workflow task heartbeat timeouts. |
| `signal_with_start_skip_delay_count` | Counter | SignalWithStart requests that skipped the delay. |
| `duplicate_replication_events` | Counter | Duplicate replication events. |
| `acquire_lock_failed` | Counter | Failed lock acquisition attempts. |
| `workflow_context_cleared` | Counter | Workflow context cleared events. |
| `mutable_state_dirty` | Counter | Dirty mutable state detections. |
| `mutable_state_checksum_mismatch` | Counter | Mutable state checksum mismatches. |
| `mutable_state_checksum_invalidated` | Counter | Invalidated mutable state checksums. |
| `closed_workflow_buffer_event_counter` | Counter | Buffer events on closed workflows. |
| `out_of_order_buffered_events` | Counter | Out-of-order buffered events. |
| `shard_linger_success` | Timer | Successful shard linger durations. |
| `shard_linger_timeouts` | Counter | Shard linger timeouts. |
| `dynamic_rate_limit_multiplier` | Gauge | Current dynamic rate limiter multiplier. |
| `dlq_writes` | Counter | Messages enqueued to the DLQ. |
| `dlq_message_count` | Gauge | Current number of messages in DLQ. |
| `data_loss_errors` | Counter | Data loss errors (high cardinality; tagged with namespace, workflowID, runID). |
| `rate_limited_task_runnable_wait_time` | Timer | Wait time for rate-limited tasks to become runnable. |
| `circuit_breaker_executable_blocked` | Counter | Executables blocked by circuit breaker. |
| `direct_query_dispatch_latency` | Timer | Direct query dispatch latency. |
| `direct_query_dispatch_sticky_latency` | Timer | Sticky direct query dispatch latency. |
| `direct_query_dispatch_non_sticky_latency` | Timer | Non-sticky direct query dispatch latency. |
| `direct_query_dispatch_sticky_success` | Counter | Successful sticky direct query dispatches. |
| `direct_query_dispatch_non_sticky_success` | Counter | Successful non-sticky direct query dispatches. |
| `direct_query_dispatch_clear_stickiness_latency` | Timer | Latency clearing stickiness during query dispatch. |
| `direct_query_dispatch_clear_stickiness_success` | Counter | Successful stickiness clears during query dispatch. |
| `direct_query_dispatch_timeout_before_non_sticky` | Counter | Query dispatch timeouts before falling back to non-sticky. |
| `workflow_task_query_latency` | Timer | Latency for workflow task queries. |
| `consistent_query_timeout` | Counter | Consistent query timeouts. |
| `query_buffer_exceeded` | Counter | Query buffer exceeded events. |
| `query_registry_invalid_state` | Counter | Invalid state in query registry. |
| `workflow_task_timeout_overrides` | Counter | Workflow task timeout overrides. |
| `workflow_run_timeout_overrides` | Counter | Workflow run timeout overrides. |
| `event_reapply_skipped_count` | Counter | Events skipped during reapplication. |
| `tasks_per_shardinfo_update` | Histogram | Tasks completed per shard info update. |
| `time_between_shardinfo_update` | Timer | Time between shard info updates. |
| `execution_time_skipping_transitioned_count` | Counter | Execution time skipping transitioned count. |
| `execution_time_skipping_transitioned_error_count` | Counter | Errors during execution time skipping transitions. |
| `history_workflow_execution_cache_latency` | Timer | Workflow execution cache operation latency. |
| `history_workflow_execution_cache_lock_hold_duration` | Timer | Duration workflow execution cache lock is held. |

---

## Task Processing

| Metric | Type | Description |
|--------|------|-------------|
| `task_requests` | Counter | Number of history tasks processed. |
| `task_latency_load` | Timer | Latency from task generation to loading into memory. |
| `task_latency_schedule` | Timer | Latency from task loading to start of processing. |
| `task_latency_processing` | Timer | Latency to process a history task one time. |
| `task_processing_no_persistence_latency` | Timer | Task processing latency excluding persistence. |
| `task_latency` | Timer | Total task processing latency across all attempts, excluding workflow lock and user quota latency. |
| `task_latency_queue` | Timer | End-to-end latency from task generation to completion. |
| `task_attempt` | Histogram | Attempts taken to complete a history task. |
| `task_errors` | Counter | Unexpected task processing errors. |
| `task_terminal_failures` | Counter | Tasks that failed terminally and were sent to DLQ. |
| `task_dlq_failures` | Counter | Failures sending tasks to the DLQ. |
| `task_dlq_latency` | Timer | Latency of the final attempt to send a task to DLQ. |
| `task_errors_discarded` | Counter | Discarded tasks. |
| `task_skipped` | Counter | Skipped tasks. |
| `task_errors_version_mismatch` | Counter | Tasks with version mismatches. |
| `task_dependency_task_not_completed` | Counter | Tasks whose dependency task is not yet completed. |
| `task_errors_standby_retry_counter` | Counter | Standby task retry errors. |
| `task_errors_workflow_busy` | Counter | Task errors due to failing to acquire workflow lock within timeout. |
| `task_errors_not_active_counter` | Counter | Task errors due to inactive namespace. |
| `task_errors_namespace_handover` | Counter | Task errors during namespace handover. |
| `task_errors_internal` | Counter | Internal task errors. |
| `task_errors_throttled` | Counter | Task errors caused by resource exhaustion (excluding workflow busy). |
| `task_errors_corruption` | Counter | Task corruption errors. |
| `chasm_pure_task_requests` | Counter | CHASM pure tasks executed. |
| `chasm_pure_task_errors` | Counter | Errors during CHASM pure task execution. |
| `task_schedule_to_start_latency` | Timer | Task schedule-to-start latency. |
| `task_batch_complete_counter` | Counter | Batch task completions. |
| `task_rescheduler_pending_tasks` | Histogram | Pending tasks in the rescheduler. |
| `pending_tasks` | Histogram | In-memory pending history tasks per shard. |
| `task_scheduler_throttled` | Counter | Throttled task scheduler events. |
| `queue_latency_schedule` | Timer | Latency for scheduling 100 tasks in one task channel. |
| `queue_reader_count` | Histogram | Queue reader count. |
| `queue_slice_count` | Histogram | Queue slice count. |
| `queue_actions` | Counter | Queue action count. |
| `execution_queue_scheduler_queue_count` | Gauge | Number of active queues in execution queue scheduler. |
| `execution_queue_scheduler_tasks_submitted` | Counter | Tasks submitted to execution queue scheduler. |
| `execution_queue_scheduler_tasks_completed` | Counter | Tasks completed by execution queue scheduler. |
| `execution_queue_scheduler_tasks_failed` | Counter | Tasks failed in execution queue scheduler. |
| `execution_queue_scheduler_submit_rejected` | Counter | Task submissions rejected by execution queue scheduler. |
| `execution_queue_scheduler_task_latency` | Timer | Task latency in execution queue scheduler. |
| `execution_queue_scheduler_queue_wait_time` | Timer | Queue wait time in execution queue scheduler. |
| `dynamic_worker_pool_scheduler_buffer_size` | Gauge | Buffer size of dynamic worker pool scheduler. |
| `dynamic_worker_pool_scheduler_active_workers` | Gauge | Active workers in dynamic worker pool scheduler. |
| `dynamic_worker_pool_scheduler_enqueued_tasks` | Counter | Tasks enqueued to dynamic worker pool scheduler. |
| `dynamic_worker_pool_scheduler_dequeued_tasks` | Counter | Tasks dequeued from dynamic worker pool scheduler. |
| `dynamic_worker_pool_scheduler_rejected_tasks` | Counter | Tasks rejected by dynamic worker pool scheduler. |

---

## Activity

| Metric | Type | Description |
|--------|------|-------------|
| `activity_end_to_end_latency` | Timer | **Deprecated.** Duration of an activity attempt. Use `activity_start_to_close_latency` instead. |
| `activity_start_to_close_latency` | Timer | Duration of a single activity attempt, excluding retries and backoffs. |
| `activity_schedule_to_close_latency` | Timer | Duration from scheduled time to terminal state, including retries and backoffs. |
| `activity_success` | Counter | Activities that succeeded (not counting retries). |
| `activity_fail` | Counter | Activities that failed with no retries remaining. |
| `activity_task_fail` | Counter | Activity task failures (includes retries). |
| `activity_cancel` | Counter | Cancelled activities. |
| `activity_terminate` | Counter | Terminated activities. |
| `activity_task_timeout` | Counter | Activity task timeouts (including retries). |
| `activity_timeout` | Counter | Terminal activity timeouts. |
| `activity_payload_size` | Counter | Activity payload sizes in bytes. |
| `paused_activities` | Counter | Paused activity count. |
| `activity_pause_requests` | Counter | Activity pause requests. |
| `activity_unpause_requests` | Counter | Activity unpause requests. |
| `activity_reset_requests` | Counter | Activity reset requests. |
| `activity_update_options_requests` | Counter | Activity update options requests. |
| `external_payload_upload_size` | Histogram (bytes) | Sizes of uploaded external payloads. |

---

## Replication

| Metric | Type | Description |
|--------|------|-------------|
| `replication_stream_panic` | Counter | Replication stream panics. |
| `replication_stream_error` | Counter | Replication stream errors. |
| `replication_service_error` | Counter | Replication service errors. |
| `replication_stream_stuck` | Counter | Stuck replication streams. |
| `replication_stream_channel_full` | Counter | Replication stream channel full events. |
| `replication_tasks_send` | Counter | Replication tasks sent. |
| `replication_task_send_attempt` | Histogram | Attempts per replication task send. |
| `replication_task_send_error` | Counter | Replication task send errors. |
| `replication_task_generation_latency` | Timer | Latency to generate a replication task. |
| `replication_task_load_latency` | Timer | Latency to load replication tasks. |
| `replication_task_load_size` | Histogram | Number of replication tasks loaded per batch. |
| `replication_task_send_latency` | Timer | Latency to send a replication task. |
| `replication_task_send_backlog` | Histogram | Replication task send backlog size. |
| `replication_tasks_recv` | Counter | Replication tasks received. |
| `replication_tasks_recv_backlog` | Histogram | Replication task receive backlog. |
| `replication_tasks_skipped` | Counter | Replication tasks skipped. |
| `replication_tasks_applied` | Counter | Replication tasks successfully applied. |
| `replication_tasks_failed` | Counter | Replication tasks that failed. |
| `replication_tasks_back_fill` | Counter | Replication tasks backfilled. |
| `replication_tasks_back_fill_latency` | Timer | Backfill replication task latency. |
| `replication_orphaned_history_branch` | Counter | History branches orphaned to avoid deleting successfully written history. |
| `replication_tasks_lag` | Histogram | Heuristic for how far behind the remote DC is (in tasks). |
| `replication_delete_execution_task_generation_failure` | Counter | Failures generating delete execution replication tasks. |
| `replication_tasks_fetched` | Histogram | Tasks fetched by the replication poller per batch. |
| `replication_latency` | Timer | End-to-end replication latency. |
| `replication_task_queue_latency` | Timer | Replication task queue latency. |
| `replication_task_processing_latency` | Timer | Replication task processing latency. |
| `replication_task_transmission_latency` | Timer | Replication task transmission latency. |
| `replication_tasks_attempt` | Histogram | Attempts per replication task. |
| `replication_tasks_error_by_type` | Counter | Replication task errors by type. |
| `replication_dlq_enqueue_failed` | Counter | Replication DLQ enqueue failures. |
| `replication_dlq_max_level` | Gauge | Max level of replication DLQ. |
| `replication_dlq_ack_level` | Gauge | Ack level of replication DLQ. |
| `replication_dlq_non_empty` | Counter | Times replication DLQ was found non-empty. |
| `replication_outlier_namespace` | Counter | Replication outlier namespaces. |
| `replication_duplicated_task` | Counter | Duplicated replication tasks. |
| `replication_sender_rate_limit_latency` | Timer | Replication sender rate limit latency. |
| `replication_task_cleanup_count` | Counter | Replication task cleanup count. |
| `replication_task_cleanup_failed` | Counter | Replication task cleanup failures. |
| `namespace_replication_task_ack_level` | Gauge | Namespace replication task ack level. |
| `namespace_dlq_ack_level` | Gauge | Namespace DLQ ack level. |
| `namespace_dlq_max_level` | Gauge | Namespace DLQ max level. |
| `namespace_replication_dlq_enqueue_requests` | Counter | Namespace replication DLQ enqueue requests. |

---

## Matching

| Metric | Type | Description |
|--------|------|-------------|
| `forwarded` | Counter | Forwarded matching requests. |
| `invalid_task_queue_name` | Counter | Requests with invalid task queue names. |
| `invalid_task_queue_partition` | Counter | Requests with invalid task queue partitions. |
| `syncmatch_latency` | Timer | Synchronous match latency per task queue. |
| `asyncmatch_latency` | Timer | Asynchronous match latency per task queue. |
| `poll_success` | Counter | Successful polls per task queue. |
| `poll_timeouts` | Counter | Poll timeouts per task queue. |
| `poll_success_sync` | Counter | Successful synchronous polls per task queue. |
| `poll_latency` | Timer | Poll latency per task queue. |
| `lease_requests` | Counter | Lease requests per task queue. |
| `lease_failures` | Counter | Lease failures per task queue. |
| `condition_failed_errors` | Counter | Condition failed errors per task queue. |
| `respond_query_failed` | Counter | Failed query task responses. |
| `respond_nexus_failed` | Counter | Failed Nexus task responses. |
| `nexus_task_requests` | Counter | Nexus task poll and respond requests in matching. |
| `sync_throttle_count` | Counter | Sync throttle events per task queue. |
| `buffer_throttle_count` | Counter | Buffer throttle events per task queue. |
| `tasks_expired` | Counter | Expired tasks per task queue. |
| `forwarded_per_tl` | Counter | Forwarded tasks per task queue. |
| `priority_backlog_forwarded` | Counter | Priority backlog tasks forwarded per task queue. |
| `forward_task_errors` | Counter | Task forwarding errors per task queue. |
| `local_to_local_matches` | Counter | Local-to-local task matches. |
| `local_to_remote_matches` | Counter | Local-to-remote task matches. |
| `remote_to_local_matches` | Counter | Remote-to-local task matches. |
| `remote_to_remote_matches` | Counter | Remote-to-remote task matches. |
| `loaded_task_queue_family_count` | Gauge | Loaded task queue family count. |
| `loaded_task_queue_count` | Gauge | Loaded task queue count. |
| `loaded_task_queue_partition_count` | Gauge | Loaded task queue partition count. |
| `force_loaded_task_queue_partitions_count` | Counter | Force-loaded task queue partition count. |
| `force_loaded_task_queue_partition_unnecessarily_count` | Counter | Task queue partitions force-loaded unnecessarily. |
| `loaded_physical_task_queue_count` | Gauge | Loaded physical task queue count. |
| `task_queue_started` | Counter | Task queues started. |
| `task_queue_stopped` | Counter | Task queues stopped. |
| `task_write_throttle_count` | Counter | Task write throttle events per task queue. |
| `task_write_latency` | Timer | Task write latency per task queue. |
| `task_rewrites` | Counter | Tasks rewritten to persistence after failing to process. |
| `task_lag_per_tl` | Gauge | Task lag per task queue. |
| `no_poller_tasks` | Counter | Tasks with no recent pollers. |
| `unknown_build_polls` | Counter | Polls from unknown build IDs. |
| `unknown_build_tasks` | Counter | Tasks dispatched to unknown build IDs. |
| `task_dispatch_latency` | Timer | Task dispatch latency per task queue. |
| `approximate_backlog_count` | Gauge | Approximate backlog task count. |
| `approximate_backlog_age_seconds` | Gauge | Approximate backlog age in seconds. |
| `physical_approximate_backlog_count` | Gauge | Physical approximate backlog count. |
| `physical_approximate_backlog_age_seconds` | Gauge | Physical approximate backlog age in seconds. |
| `non_retryable_tasks` | Counter | Non-retryable matching tasks dropped due to errors. |
| `task_completed_dropped` | Counter | Tasks completed after being dropped from the matcher. |
| `task_retry_transient` | Counter | Tasks retried immediately after a transient error. |
| `reachability_exit_point_count` | Counter | Reachability analysis exit point count. |
| `partition_cache_size` | Gauge | Client-side matching partition cache size. |
| `worker_registry_eviction_blocked_by_age` | Gauge | Worker registry entries that could not be evicted due to minEvictAge policy. |
| `worker_registry_capacity_utilization` | Gauge | Ratio of total entries to maxItems in worker registry. |
| `worker_registry_workers_added` | Counter | Workers registered in the worker registry. |
| `worker_registry_workers_removed` | Counter | Workers removed from the worker registry. |
| `worker_registry_activity_slots_used` | Histogram | Activity slots in use per worker. |
| `worker_plugin_name` | Gauge | Set if worker is configured with a plugin. Dimensions: namespace, plugin_name. |
| `worker_storage_driver_type` | Gauge | Set if worker is configured with a storage driver. Dimensions: namespace, storage_driver_type. |
| `poller_autoscaling_heartbeat_count` | Counter | Worker heartbeats with poller autoscaling enabled. Dimensions: namespace, taskqueue, task_type. |

---

## Nexus

| Metric | Type | Description |
|--------|------|-------------|
| `nexus_requests` | Counter | Nexus requests received by the service. |
| `nexus_request_preprocess_errors` | Counter | Nexus requests for which pre-processing failed. |
| `nexus_latency` | Timer | Nexus request latency. |
| `nexus_completion_requests` | Counter | Nexus completion (callback) requests received. |
| `nexus_completion_latency` | Timer | Nexus completion request latency. |
| `nexus_completion_request_preprocess_errors` | Counter | Nexus completion requests for which pre-processing failed. |

---

## Worker / Archival

| Metric | Type | Description |
|--------|------|-------------|
| `executor_done` | Counter | Executor tasks successfully completed. |
| `executor_err` | Counter | Executor task errors. |
| `executor_deferred` | Counter | Executor tasks deferred. |
| `executor_dropped` | Counter | Executor tasks dropped. |
| `started` | Counter | Worker started count. |
| `stopped` | Counter | Worker stopped count. |
| `task_processed` | Gauge | Tasks processed. |
| `task_deleted` | Gauge | Tasks deleted. |
| `taskqueue_processed` | Gauge | Task queues processed. |
| `taskqueue_deleted` | Gauge | Task queues deleted. |
| `taskqueue_outstanding` | Gauge | Outstanding task queues. |
| `history_archiver_archive_non_retryable_error` | Counter | Non-retryable history archiver errors. |
| `history_archiver_archive_transient_error` | Counter | Transient history archiver errors. |
| `history_archiver_archive_success` | Counter | Successful history archivals. |
| `history_archiver_total_upload_size` | Histogram (bytes) | Total upload size for history archiver. |
| `history_archiver_history_size` | Histogram (bytes) | Archived history size. |
| `history_archiver_duplicate_archivals` | Counter | Duplicate history archival attempts. |
| `history_archiver_blob_exists` | Counter | Blob already exists during archival. |
| `history_archiver_blob_size` | Histogram (bytes) | History archiver blob size. |
| `visibility_archiver_archive_non_retryable_error` | Counter | Non-retryable visibility archiver errors. |
| `visibility_archiver_archive_transient_error` | Counter | Transient visibility archiver errors. |
| `visibility_archiver_archive_success` | Counter | Successful visibility archivals. |
| `scavenger_success` | Counter | Successful scavenger runs. |
| `scavenger_errors` | Counter | Scavenger errors. |
| `scavenger_skips` | Counter | Scavenger skips. |
| `executions_outstanding` | Gauge | Outstanding executions count. |
| `scavenger_validation_requests` | Counter | Scavenger validation requests. |
| `scavenger_validation_failures` | Counter | Scavenger validation failures. |
| `scavenger_validation_skips` | Counter | Scavenger validation skips. |
| `add_search_attributes_failures` | Counter | AddSearchAttributes failures. |
| `worker_commands_sent` | Counter | Worker command dispatches, tagged by outcome (e.g. success, no_poller, rpc_error). |

---

## Schedules

| Metric | Type | Description |
|--------|------|-------------|
| `schedule_missed_catchup_window` | Counter | Times a schedule missed an action due to the catchup window. |
| `schedule_rate_limited` | Counter | Times a schedule action was delayed >1s due to rate limiting. |
| `schedule_buffer_overruns` | Counter | Schedule actions dropped due to the action buffer being full. |
| `schedule_action_success` | Counter | Schedule actions successfully taken. |
| `schedule_action_errors` | Counter | Failed attempts from starting schedule actions. |
| `schedule_cancel_workflow_errors` | Counter | Errors trying to cancel a previous scheduled run. |
| `schedule_terminate_workflow_errors` | Counter | Errors trying to terminate a previous scheduled run. |
| `schedule_action_delay` | Timer | Delay between when scheduled actions should vs. actually happen. |
| `schedule_payload_size` | Counter | Size in bytes of a schedule customer payload. |
| `schedule_migration_started` | Counter | Schedule migrations started. |
| `schedule_migration_completed` | Counter | Schedule migrations completed successfully. |
| `schedule_migration_failed` | Counter | Schedule migrations that failed. |

---

## Worker Versioning & Deployments

| Metric | Type | Description |
|--------|------|-------------|
| `worker_deployment_created` | Counter | Worker deployments created. |
| `worker_deployment_version_created` | Counter | Worker deployment versions created. |
| `worker_deployment_version_created_managed_by_controller` | Counter | Deployment versions created managed by controller. |
| `worker_deployment_version_visibility_query_count` | Counter | Visibility queries for worker deployment versions. |
| `worker_deployment_versioning_override_count` | Counter | Worker deployment versioning overrides. |
| `start_deployment_transition_count` | Counter | Deployment transitions started. |
| `versioning_data_propagation_latency` | Timer | Latency for versioning data propagation. |
| `slow_versioning_data_propagation` | Counter | Slow versioning data propagation events. |
| `workflow_target_version_changed_count` | Counter | Times a workflow's target version changed. |

---

## Workflow Execution

| Metric | Type | Description |
|--------|------|-------------|
| `workflow_backoff_timer` | Counter | Workflow backoff timer events. |
| `workflow_retry_backoff_timer` | Counter | Workflow retry backoff timer events. |
| `workflow_cron_backoff_timer` | Counter | Workflow cron backoff timer events. |
| `workflow_delayed_start_backoff_timer` | Counter | Workflow delayed-start backoff timer events. |
| `workflow_cleanup_delete` | Counter | Workflow cleanup deletes. |
| `workflow_success` | Counter | Successful workflow completions. |
| `workflow_cancel` | Counter | Cancelled workflows. |
| `workflow_failed` | Counter | Failed workflows. |
| `workflow_timeout` | Counter | Timed-out workflows. |
| `workflow_terminate` | Counter | Terminated workflows. |
| `workflow_continued_as_new` | Counter | Workflows continued-as-new. |
| `workflow_schedule_to_close_latency` | Timer | Total workflow schedule-to-close latency. |
| `workflow_continue_as_new_count` | Counter | Continue-as-new count. |
| `workflow_suggest_continue_as_new_count` | Counter | Suggested continue-as-new count. |
| `workflow_reset_count` | Counter | Workflow resets. |
| `workflow_query_success_count` | Counter | Successful workflow queries. |
| `workflow_query_failure_count` | Counter | Failed workflow queries. |
| `workflow_query_timeout_count` | Counter | Timed-out workflow queries. |
| `workflow_tasks_completed` | Counter | Workflow tasks completed. |

---

## Caching

| Metric | Type | Description |
|--------|------|-------------|
| `cache_requests` | Counter | Cache requests. |
| `cache_errors` | Counter | Cache errors. |
| `cache_latency` | Timer | Cache operation latency. |
| `cache_miss` | Counter | Cache misses. |
| `cache_size` | Gauge | Cache size. |
| `cache_usage` | Gauge | Cache usage. |
| `cache_pinned_usage` | Gauge | Pinned cache usage. |
| `cache_ttl` | Timer | Cache TTL distribution. |
| `cache_entry_age_on_get` | Timer | Age of cache entries at get time. |
| `cache_entry_age_on_eviction` | Timer | Age of cache entries at eviction time. |

---

## Namespace Registry

| Metric | Type | Description |
|--------|------|-------------|
| `namespace_registry_watch_reconnections` | Counter | Namespace registry watch reconnections. |
| `namespace_registry_watch_start_failures` | Counter | Namespace registry watch start failures. |
| `namespace_registry_slow_callbacks` | Counter | Slow namespace registry callbacks. |
| `namespace_registry_refresh_failures` | Counter | Namespace registry refresh failures. |
| `namespace_registry_refresh_latency` | Timer | Namespace registry refresh latency. |

---

## Deadlock Detector

| Metric | Type | Description |
|--------|------|-------------|
| `dd_suspected_deadlocks` | Counter | Suspected deadlocks detected. |
| `dd_current_suspected_deadlocks` | Gauge | Current number of suspected deadlocks. |
| `dd_cluster_metadata_lock_latency` | Timer | Cluster metadata lock latency. |
| `dd_cluster_metadata_callback_lock_latency` | Timer | Cluster metadata callback lock latency. |
| `dd_shard_controller_lock_latency` | Timer | Shard controller lock latency. |
| `dd_shard_lock_latency` | Timer | Shard lock latency. |
| `dd_shard_io_semaphore_latency` | Timer | Shard I/O semaphore latency. |
| `dd_namespace_registry_lock_latency` | Timer | Namespace registry lock latency. |

---

## Elasticsearch / Bulk Processor

| Metric | Type | Description |
|--------|------|-------------|
| `elasticsearch_bulk_processor_requests` | Counter | Elasticsearch bulk processor requests. |
| `elasticsearch_bulk_processor_queued_requests` | Histogram | Queued requests in Elasticsearch bulk processor. |
| `elasticsearch_bulk_processor_errors` | Counter | Elasticsearch bulk processor errors. |
| `elasticsearch_bulk_processor_corrupted_data` | Counter | Corrupted data in Elasticsearch bulk processor. |
| `elasticsearch_bulk_processor_duplicate_request` | Counter | Duplicate Elasticsearch bulk processor requests. |
| `elasticsearch_bulk_processor_request_latency` | Timer | Elasticsearch bulk processor request latency. |
| `elasticsearch_bulk_processor_commit_latency` | Timer | Elasticsearch bulk processor commit latency. |
| `elasticsearch_bulk_processor_wait_add_latency` | Timer | Elasticsearch bulk processor wait-add latency. |
| `elasticsearch_bulk_processor_wait_start_latency` | Timer | Elasticsearch bulk processor wait-start latency. |
| `elasticsearch_bulk_processor_bulk_size` | Histogram | Elasticsearch bulk request size. |
| `elasticsearch_bulk_processor_bulk_request_took_latency` | Timer | Elasticsearch bulk request execution latency. |
| `elasticsearch_document_parse_failures_counter` | Counter | Elasticsearch document parse failures. |
| `elasticsearch_document_generate_failures_counter` | Counter | Elasticsearch document generation failures. |
| `elasticsearch_custom_order_by_clause_counter` | Counter | Elasticsearch custom ORDER BY clause usage. |

---

## Delete Namespace / Executions

| Metric | Type | Description |
|--------|------|-------------|
| `reclaim_resources_namespace_delete_success` | Counter | Namespaces successfully deleted by ReclaimResources workflow. |
| `reclaim_resources_namespace_delete_failure` | Counter | ReclaimResources workflows completing without deleting a namespace. |
| `reclaim_resources_delete_executions_success` | Counter | Executions successfully deleted during ReclaimResources. |
| `reclaim_resources_delete_executions_failure` | Counter | Executions with errors during ReclaimResources. |
| `delete_executions_success` | Counter | Executions successfully deleted by DeleteExecutions workflow. |
| `delete_executions_failure` | Counter | Executions with errors during DeleteExecutions. |
| `delete_executions_not_found` | Counter | Executions not found during DeleteExecutions. |
| `batcher_processor_requests` | Counter | Workflow execution tasks successfully processed by the batch processor. |
| `batcher_processor_errors` | Counter | Batch processor errors. |
| `batcher_operation_errors` | Counter | Batch operation errors. |
| `catchup_ready_shard_count` | Gauge | Shards ready for catch-up. |
| `handover_ready_shard_count` | Gauge | Shards ready for handover. |
| `replicator_messages` | Counter | Replicator messages processed. |
| `replicator_errors` | Counter | Replicator errors. |
| `replicator_latency` | Timer | Replicator latency. |
| `replicator_dlq_enqueue_fails` | Counter | Replicator DLQ enqueue failures. |
| `parent_close_policy_processor_requests` | Counter | Successful parent close policy processor requests. |
| `parent_close_policy_processor_errors` | Counter | Parent close policy processor errors. |

---

## Force Replication

| Metric | Type | Description |
|--------|------|-------------|
| `encounter_zombie_workflow_count` | Counter | Zombie workflows encountered during force replication. |
| `encounter_not_found_workflow_count` | Counter | Workflows not found during force replication. |
| `encounter_pass_retention_workflow_count` | Counter | Workflows past retention encountered during force replication. |
| `generate_replication_tasks_latency` | Timer | Latency to generate replication tasks. |
| `verify_replication_task_success` | Counter | Successfully verified replication tasks. |
| `verify_replication_task_not_found` | Counter | Replication tasks not found during verification. |
| `verify_replication_task_failed` | Counter | Replication task verification failures. |
| `verify_replication_tasks_latency` | Timer | Replication task verification latency. |
| `verify_describe_mutable_state_latency` | Timer | DescribeMutableState latency during verification. |

---

## Runtime / Go Process

| Metric | Type | Description |
|--------|------|-------------|
| `num_goroutines` | Gauge | Number of goroutines. |
| `gomaxprocs` | Gauge | GOMAXPROCS value. |
| `memory_allocated` | Gauge | Total allocated memory. |
| `memory_heap` | Gauge | Heap memory in use. |
| `memory_heap_objects` | Gauge | Number of allocated heap objects. |
| `memory_heapidle` | Gauge | Idle heap memory. |
| `memory_heapinuse` | Gauge | In-use heap memory. |
| `memory_heapreleased` | Gauge | Released heap memory. |
| `memory_stack` | Gauge | Stack memory in use. |
| `memory_mallocs` | Gauge | Total malloc calls. |
| `memory_frees` | Gauge | Total free calls. |
| `memory_num_gc` | Histogram (bytes) | GC run count. |
| `memory_gc_pause_ms` | Timer | GC pause duration. |
| `memory_num_gc_last` | Gauge | Last `runtime.MemStats.NumGC` value. |
| `memory_pause_total_ns_last` | Gauge | Last `runtime.MemStats.PauseTotalNs` value. |

---

## Common Tags Reference

| Tag Name | Description |
|----------|-------------|
| `operation` | Operation name. |
| `service_role` | Service role (e.g. `history`, `matching`, `frontend`, `admin`). |
| `cache_type` | Cache type (e.g. `mutablestate`, `events`). |
| `failure` | Failure indicator. |
| `failure_source` | Source of the failure. |
| `task_category` | Task category. |
| `task_type` | Task type. |
| `task_priority` | Task priority. |
| `queue_reader_id` | Queue reader ID. |
| `queue_action` | Queue action. |
| `queue_type` | Queue type. |
| `error_type` | Error type. |
| `outcome` | Outcome (e.g. `success`, `no_poller`, `rpc_error`). |
| `versioned` | Whether the entity is versioned. |
| `resource_exhausted_cause` | Cause of resource exhaustion. |
| `resource_exhausted_scope` | Scope of resource exhaustion. |
| `partition` | Partition identifier. |
| `priority` | Priority level. |
| `db_kind` | Database kind. |
| `worker_plugin_name` | Worker plugin name. |
| `worker_storage_driver_type` | Worker storage driver type. |
| `archetype` | Archetype identifier. |
| `chasm_task_type` | CHASM task type. |
| `nexus_endpoint` | Nexus endpoint name. |
| `nexus_service` | Nexus service name. |
| `nexus_operation` | Nexus operation name. |
| `schedule_action` | Schedule action type. |
| `scheduler_backend` | Scheduler backend (`chasm`, `legacy`, `workflow`). |
| `schedule_migration_direction` | Schedule migration direction (`to_chasm`, `to_workflow`). |