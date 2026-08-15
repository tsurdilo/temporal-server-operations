# Temporal HANDOVER Replication State - Implementation Research

## 1. ENUM DEFINITION

**File:** `/Users/tsurdilo/devel/temporal/api/temporal/api/enums/v1/namespace.proto`

ReplicationState enum with three states:
- REPLICATION_STATE_UNSPECIFIED = 0
- REPLICATION_STATE_NORMAL = 1
- REPLICATION_STATE_HANDOVER = 2

**Import:** `enumspb "go.temporal.io/api/enums/v1"`

---

## 2. CORE HANDOVER TRACKING

### HandoverTracker Interface
**File:** `/Users/tsurdilo/devel/temporal/temporal/service/history/shard/handover_tracker.go`

- Tracks namespace handover state on a shard
- Manages mapping of namespaces to replication watermarks during handover
- Key methods:
  - `UpdateHandoverState(ns *namespace.Namespace, deletedFromDB bool)` - Process namespace state changes
  - `IsInHandover(namespaceName namespace.Name, workflowID string) bool` - Check if namespace is in handover
  - `GetHandoverNamespaces() map[string]*historyservice.HandoverNamespaceInfo` - Get handover info for RPC
  - `ResolvePendingTaskIDs(maxReplicationTaskID int64)` - Replace sentinel watermarks with real values

### Handover Data Structure
**In:** `context_impl.go` line 162

```go
namespaceHandOverInfo struct {
    MaxReplicationTaskID int64
    NotificationVersion  int64
}
```

### Handover Tracker Implementation
Lines 78-113 in `handover_tracker.go`:

- Checks if namespace is global AND active in current cluster AND in HANDOVER state
- Tracks max replication task ID at time of handover transition
- Uses PendingMaxReplicationTaskID sentinel value until shard state acquired
- Updates handover info when notification version increases

---

## 3. REPLICATION STATE IN NAMESPACE

**File:** `/Users/tsurdilo/devel/temporal/temporal/common/namespace/replication_resolver.go`

```go
type ReplicationResolver interface {
    ReplicationState(businessID string) enumspb.ReplicationState
    ...
}
```

Implementation (line 91):
```go
func (r *defaultReplicationResolver) ReplicationState(_ string) enumspb.ReplicationState {
    if r.replicationConfig == nil {
        return enumspb.REPLICATION_STATE_UNSPECIFIED
    }
    return r.replicationConfig.State
}
```

State stored in: `persistencespb.NamespaceReplicationConfig.State`

---

## 4. HANDOVER WORKFLOW (Migration)

**File:** `/Users/tsurdilo/devel/temporal/temporal/service/worker/migration/handover_workflow.go`

Workflow steps:
1. **Step 1:** Get Cluster Metadata
2. **Step 2:** Get current replication status
3. **Step 3:** Wait for Remote Cluster to catch-up on Replication Tasks
4. **Step 4:** Transition to HANDOVER state (WARNING: Namespace cannot serve traffic)
5. **Step 5:** Wait for Remote Cluster to completely drain Replication Tasks
6. **Step 6:** Update Namespace Active Cluster to Remote
7. **Final Step:** Reset namespace state from HANDOVER → NORMAL

Key parameters:
- `AllowedLaggingSeconds`: How far behind remote cluster can be (5-120 sec default)
- `HandoverTimeoutSeconds`: Max time to wait for handover to complete (max 30 sec)

State transitions (lines 119, 146):
```go
NewState: enumspb.REPLICATION_STATE_NORMAL  // Step 3 - final reset
NewState: enumspb.REPLICATION_STATE_HANDOVER // Step 4 - enters handover
```

---

## 5. HANDOVER VALIDATION & CONSTRAINTS

**File:** `/Users/tsurdilo/devel/temporal/temporal/service/frontend/namespace_handler.go`

Lines 1144-1159 - Validation when setting HANDOVER state:
- HANDOVER can only be set for GLOBAL namespaces
- Namespace must have 2+ replication clusters
- Returns error: `"REPLICATION_STATE_HANDOVER require more than one replication clusters"`

Lines 1173-1175 - State update blocking:
- Cannot update namespace State (REGISTERED/DEPRECATED/DELETED) while in HANDOVER
- Returns error: `"cannot update namespace state while its replication state in REPLICATION_STATE_HANDOVER"`

---

## 6. HANDOVER INTERCEPTOR (Request Processing)

**File:** `/Users/tsurdilo/devel/temporal/temporal/common/rpc/interceptor/namespace_handover.go`

- Type: `NamespaceHandoverInterceptor`
- Intercepts WorkflowService requests when namespace in HANDOVER state
- Waits for namespace state change from HANDOVER → normal state
- Allowed methods during HANDOVER are whitelisted
- Uses state change callbacks to monitor for handover completion
- Max wait time = cache refresh interval (configurable)

Key check (line 135):
```go
if namespaceData.ReplicationState(businessID) == enumspb.REPLICATION_STATE_HANDOVER {
    // Wait for state change callback
}
```

State change validation (lines 142-145):
- Stop waiting if: namespace is deleting, state changed from HANDOVER, or namespace is not global

---

## 7. REPLICATION TASK FILTERING

**Files:** 
- `/Users/tsurdilo/devel/temporal/temporal/service/history/transfer_queue_active_task_executor.go`
- `/Users/tsurdilo/devel/temporal/temporal/service/history/outbound_queue_active_task_executor.go`
- `/Users/tsurdilo/devel/temporal/temporal/service/history/timer_queue_active_task_executor.go`
- `/Users/tsurdilo/devel/temporal/temporal/service/history/visibility_queue_task_executor.go`

Pattern (all task executors):
```go
if replicationState == enumspb.REPLICATION_STATE_HANDOVER {
    // Skip task execution during handover
    return queues.ExecuteResponse{...}
}
```

---

## 8. NAMESPACE TRANSMISSION (Replication)

**File:** `/Users/tsurdilo/devel/temporal/temporal/common/namespace/nsreplication/transmission_task_handler.go`

Key property (lines 112-114):
```go
if replicationConfig.State == enumspb.REPLICATION_STATE_NORMAL {
    task.NamespaceTaskAttributes.ReplicationConfig.State = replicationConfig.State
}
```

**CRITICAL:** HANDOVER state is NOT replicated to other clusters
- Only NORMAL state is included in replication task
- HANDOVER is local-only state on active cluster
- Comment in test (line 453): "HANDOVER is local-only and must not be propagated"

---

## 9. HANDOVER NAMESPACE INFO (API)

**File:** `/Users/tsurdilo/devel/temporal/temporal/api/historyservice/v1/request_response.pb.go`

Proto message: `HandoverNamespaceInfo`
- Field: `HandoverReplicationTaskId int64` - Max replication task ID when namespace transitioned to HANDOVER

Used in: `ShardReplicationStatus.HandoverNamespaces` map

---

## 10. ERROR HANDLING

**File:** `/Users/tsurdilo/devel/temporal/temporal/common/util.go` line 129

```go
ErrNamespaceHandover = serviceerror.NewUnavailablef(
    "Namespace replication in %s state.",
    enumspb.REPLICATION_STATE_HANDOVER,
)
```

---

## 11. KEY SEARCH RESULTS

Grep results confirming HANDOVER state usage:

Total files with HANDOVER references:
- `replication_resolver_test.go` - Test validation
- `transmission_task_handler_test.go` - Test: HANDOVER not replicated (line 453)
- `namespace_handler.go` - Validation and state management
- `namespace_handover.go` - Request interception during handover
- `handover_tracker.go` - Shard-level handover tracking
- `handover_workflow.go` - Migration workflow orchestration
- `transfer_queue_active_task_executor.go` - Task filtering
- `history_service.pb.go` - Handover namespace info message
- Multiple task queue executors - Task execution blocking

---

## STATE MACHINE TRANSITIONS

Normal Failover Path:
```
NORMAL → (user initiates handover workflow)
  ↓
HANDOVER → (remote cluster catches up)
  ↓
NORMAL (with active cluster switched)
```

Key constraints:
1. Only accessible from NORMAL state
2. Requires 2+ replication clusters
3. Global namespace only
4. Blocks most RPC operations
5. Not propagated to passive clusters
6. Reverted back to NORMAL after handover completes
