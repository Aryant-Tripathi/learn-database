# Module 10: Project - Build a Replicated Key-Value Store with Raft

## Project Overview

In this project, you will build a distributed, replicated key-value store from scratch using the Raft consensus algorithm. The system will consist of multiple nodes that maintain a consistent state despite node failures and network partitions.

```mermaid
flowchart TB
    subgraph "Your System"
        Client["HTTP Client"]
        subgraph "Node 1"
            API1["HTTP API"]
            Raft1["Raft Module"]
            KV1["KV State Machine"]
        end
        subgraph "Node 2"
            API2["HTTP API"]
            Raft2["Raft Module"]
            KV2["KV State Machine"]
        end
        subgraph "Node 3"
            API3["HTTP API"]
            Raft3["Raft Module"]
            KV3["KV State Machine"]
        end
    end

    Client -->|"PUT/GET/DELETE"| API1
    Raft1 <-->|"RPC"| Raft2
    Raft1 <-->|"RPC"| Raft3
    Raft2 <-->|"RPC"| Raft3
    Raft1 --> KV1
    Raft2 --> KV2
    Raft3 --> KV3
```

## Learning Objectives

By completing this project, you will:
1. Implement Raft leader election with randomized timeouts
2. Implement Raft log replication with consistency checks
3. Build a replicated state machine (key-value store)
4. Support linearizable reads using the ReadIndex protocol
5. Test your system under simulated network partitions
6. Understand why consensus is hard and where edge cases lurk

---

## Prerequisites

- Go 1.21+ (recommended) or Rust
- Understanding of goroutines/channels (Go) or async/tokio (Rust)
- Familiarity with RPC (gRPC or simple HTTP/JSON)

---

## Part 1: Project Structure

Create the following directory layout:

```
raft-kv/
├── main.go                 # Entry point, starts a node
├── raft/
│   ├── raft.go             # Core Raft state machine
│   ├── log.go              # Log entry management
│   ├── rpc.go              # RPC message types
│   └── raft_test.go        # Unit tests
├── kv/
│   ├── store.go            # Key-value state machine
│   └── store_test.go       # KV tests
├── transport/
│   ├── http_transport.go   # HTTP-based RPC transport
│   └── sim_transport.go    # Simulated transport for testing
├── server/
│   ├── server.go           # HTTP API server
│   └── handler.go          # Request handlers
└── test/
    ├── partition_test.go   # Network partition tests
    └── linearizability_test.go  # Linearizability tests
```

---

## Part 2: Implement Raft Leader Election

### 2.1 Define Core Types

```mermaid
classDiagram
    class RaftNode {
        -id: NodeID
        -state: State
        -currentTerm: uint64
        -votedFor: NodeID
        -log: []LogEntry
        -commitIndex: uint64
        -lastApplied: uint64
        -nextIndex: map[NodeID]uint64
        -matchIndex: map[NodeID]uint64
        +Propose(data []byte) error
        +Step(msg Message) error
        +Tick()
        +Ready() Ready
        +Advance(rd Ready)
    }

    class LogEntry {
        +Term: uint64
        +Index: uint64
        +Data: []byte
    }

    class Message {
        +Type: MessageType
        +From: NodeID
        +To: NodeID
        +Term: uint64
        +LogTerm: uint64
        +LogIndex: uint64
        +Entries: []LogEntry
        +Commit: uint64
        +Reject: bool
    }

    class Ready {
        +Messages: []Message
        +CommittedEntries: []LogEntry
        +HardState: HardState
    }

    RaftNode --> LogEntry
    RaftNode --> Message
    RaftNode --> Ready
```

### 2.2 Implementation Tasks

**Task 1: Define the Raft state machine states.**

```go
type State int

const (
    StateFollower State = iota
    StateCandidate
    StateLeader
)

type RaftNode struct {
    mu sync.Mutex

    // Persistent state (must survive restarts)
    id          uint64
    currentTerm uint64
    votedFor    uint64
    log         []LogEntry

    // Volatile state
    state       State
    commitIndex uint64
    lastApplied uint64

    // Leader-only volatile state
    nextIndex  map[uint64]uint64
    matchIndex map[uint64]uint64

    // Configuration
    peers           []uint64
    electionTimeout int  // in ticks
    heartbeatTimeout int // in ticks

    // Internal counters
    electionElapsed  int
    heartbeatElapsed int
    randomizedElectionTimeout int

    // Votes received in current election
    votes map[uint64]bool

    // Output
    messages        []Message
    committedEntries []LogEntry
}
```

**Task 2: Implement the Tick function.**

The `Tick()` function is called at regular intervals (e.g., every 100ms). It drives election timeouts and heartbeat sending.

```mermaid
flowchart TD
    Tick["Tick() called"]
    State{"Current state?"}
    Follower["electionElapsed++<br/>Timeout?"]
    Candidate["electionElapsed++<br/>Timeout?"]
    Leader["heartbeatElapsed++<br/>Timeout?"]
    StartElection["Start new election"]
    SendHeartbeats["Send heartbeats<br/>to all peers"]

    Tick --> State
    State -->|Follower| Follower
    State -->|Candidate| Candidate
    State -->|Leader| Leader
    Follower -->|Yes| StartElection
    Candidate -->|Yes| StartElection
    Leader -->|Yes| SendHeartbeats
```

**Task 3: Implement leader election.**

When a follower's election timer expires:

1. Transition to candidate state.
2. Increment `currentTerm`.
3. Vote for self.
4. Reset election timer with a new random value.
5. Send `RequestVote` RPCs to all peers.

When processing a `RequestVote` request:

1. If the request's term > current term: update term, become follower.
2. Grant vote only if:
   - Have not voted in this term, OR already voted for this candidate.
   - Candidate's log is at least as up-to-date as own log.
3. Reset election timer if granting vote.

**Task 4: Handle vote responses.**

Track votes in the `votes` map. When a majority is reached:

1. Transition to leader state.
2. Initialize `nextIndex` for each peer to `len(log) + 1`.
3. Initialize `matchIndex` for each peer to 0.
4. Send initial empty `AppendEntries` (heartbeats) to all peers.

### 2.3 Testing Leader Election

Write tests for these scenarios:

```mermaid
flowchart TD
    subgraph "Test Scenarios"
        T1["Test 1: Basic Election<br/>3 nodes, one times out,<br/>becomes leader"]
        T2["Test 2: Split Vote<br/>2 candidates start simultaneously,<br/>new election resolves"]
        T3["Test 3: Stale Leader<br/>Leader with old term<br/>steps down on hearing new term"]
        T4["Test 4: Log Up-to-date Check<br/>Candidate with stale log<br/>denied votes"]
        T5["Test 5: Pre-existing Leader<br/>Candidate receives AppendEntries<br/>from valid leader, steps down"]
    end
```

```go
func TestBasicElection(t *testing.T) {
    // Create 3 nodes
    nodes := createTestCluster(3)

    // Advance time until one node times out and starts election
    for i := 0; i < 20; i++ {
        for _, n := range nodes {
            n.Tick()
        }
        deliverMessages(nodes)
    }

    // Verify exactly one leader exists
    leaders := countLeaders(nodes)
    if leaders != 1 {
        t.Fatalf("expected 1 leader, got %d", leaders)
    }

    // Verify all nodes agree on the term
    term := nodes[0].currentTerm
    for _, n := range nodes {
        if n.currentTerm != term {
            t.Fatalf("term mismatch: %d vs %d", n.currentTerm, term)
        }
    }
}
```

---

## Part 3: Implement Log Replication

### 3.1 AppendEntries RPC

```mermaid
sequenceDiagram
    participant L as Leader
    participant F as Follower

    L->>F: AppendEntries(term, prevLogIndex,<br/>prevLogTerm, entries, leaderCommit)

    alt Log matches at prevLogIndex
        F->>F: Append new entries
        F->>F: Update commitIndex
        F-->>L: Success=true, matchIndex
    else Log does not match
        F-->>L: Success=false,<br/>conflictTerm, conflictIndex
        L->>L: Decrement nextIndex[follower]
        L->>F: Retry with earlier prevLogIndex
    end
```

### 3.2 Implementation Tasks

**Task 5: Implement AppendEntries sending (leader side).**

For each peer, the leader:
1. Determines `prevLogIndex` and `prevLogTerm` from `nextIndex[peer]`.
2. Collects entries from `nextIndex[peer]` onward.
3. Sends an `AppendEntries` message.

**Task 6: Implement AppendEntries handling (follower side).**

1. Reject if `term < currentTerm`.
2. Reset election timer (valid leader exists).
3. Check log consistency at `prevLogIndex`.
4. If consistent: append entries, update `commitIndex`.
5. If inconsistent: reply with conflict information.

**Task 7: Implement commitment tracking.**

The leader updates `commitIndex` after receiving successful `AppendEntries` responses:

```go
func (rn *RaftNode) maybeAdvanceCommitIndex() {
    // Sort matchIndex values
    matches := make([]uint64, 0, len(rn.peers)+1)
    matches = append(matches, rn.lastLogIndex()) // leader's own match
    for _, peer := range rn.peers {
        matches = append(matches, rn.matchIndex[peer])
    }
    sort.Slice(matches, func(i, j int) bool {
        return matches[i] > matches[j]
    })

    // The median value is the highest index replicated to a majority
    majorityMatch := matches[len(matches)/2]

    // Only commit entries from the current term
    if majorityMatch > rn.commitIndex &&
        rn.log[majorityMatch-1].Term == rn.currentTerm {
        rn.commitIndex = majorityMatch
    }
}
```

**Critical rule:** The leader only advances `commitIndex` for entries from its **current term**. This prevents the commitment ambiguity described in the Raft paper (Figure 8).

---

## Part 4: Build the Key-Value State Machine

### 4.1 State Machine Interface

```mermaid
flowchart LR
    Raft["Raft Module"] -->|"Committed entries"| SM["State Machine<br/>(KV Store)"]
    SM -->|"Applied index"| Raft
    Client["Client"] -->|"Read request<br/>(after ReadIndex)"| SM
```

```go
type KVStore struct {
    mu   sync.RWMutex
    data map[string]string

    // Track applied index to prevent double-application
    appliedIndex uint64
}

type Command struct {
    Op    string `json:"op"`    // "put", "get", "delete"
    Key   string `json:"key"`
    Value string `json:"value"`
}

func (kv *KVStore) Apply(entry LogEntry) interface{} {
    kv.mu.Lock()
    defer kv.mu.Unlock()

    if entry.Index <= kv.appliedIndex {
        return nil // Already applied (idempotency)
    }

    var cmd Command
    json.Unmarshal(entry.Data, &cmd)

    var result interface{}
    switch cmd.Op {
    case "put":
        kv.data[cmd.Key] = cmd.Value
        result = "OK"
    case "delete":
        delete(kv.data, cmd.Key)
        result = "OK"
    case "get":
        result = kv.data[cmd.Key]
    }

    kv.appliedIndex = entry.Index
    return result
}

func (kv *KVStore) Get(key string) (string, bool) {
    kv.mu.RLock()
    defer kv.mu.RUnlock()
    v, ok := kv.data[key]
    return v, ok
}
```

### 4.2 Client-Facing API

**Task 8: Implement the HTTP API server.**

```go
// PUT /kv/:key
// Body: {"value": "..."}
func (s *Server) handlePut(w http.ResponseWriter, r *http.Request) {
    key := mux.Vars(r)["key"]
    var body struct{ Value string }
    json.NewDecoder(r.Body).Decode(&body)

    cmd := Command{Op: "put", Key: key, Value: body.Value}
    data, _ := json.Marshal(cmd)

    // Propose to Raft (only works on leader)
    resultCh, err := s.raft.Propose(data)
    if err != nil {
        if err == ErrNotLeader {
            // Redirect to leader
            http.Redirect(w, r, s.leaderURL()+r.URL.Path, http.StatusTemporaryRedirect)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    // Wait for the entry to be committed and applied
    select {
    case result := <-resultCh:
        json.NewEncoder(w).Encode(result)
    case <-time.After(5 * time.Second):
        http.Error(w, "timeout", http.StatusGatewayTimeout)
    }
}
```

---

## Part 5: Support Linearizable Reads

Linearizable reads must return the most recently committed value. There are two common approaches:

### 5.1 ReadIndex Protocol

```mermaid
sequenceDiagram
    participant Client
    participant Leader
    participant F1 as Follower 1
    participant F2 as Follower 2

    Client->>Leader: GET /kv/mykey
    Note over Leader: 1. Record readIndex = commitIndex
    Leader->>F1: Heartbeat (empty AppendEntries)
    Leader->>F2: Heartbeat (empty AppendEntries)
    F1-->>Leader: ACK (confirms leader)
    F2-->>Leader: ACK (confirms leader)
    Note over Leader: 2. Majority confirmed leadership
    Note over Leader: 3. Wait until appliedIndex >= readIndex
    Note over Leader: 4. Read from state machine
    Leader-->>Client: {"value": "result"}
```

**Task 9: Implement ReadIndex.**

```go
func (rn *RaftNode) ReadIndex() (uint64, error) {
    rn.mu.Lock()
    if rn.state != StateLeader {
        rn.mu.Unlock()
        return 0, ErrNotLeader
    }

    readIndex := rn.commitIndex
    rn.mu.Unlock()

    // Send heartbeats and wait for majority ACK
    ackCh := rn.broadcastHeartbeat()

    select {
    case <-ackCh:
        return readIndex, nil
    case <-time.After(2 * time.Second):
        return 0, ErrTimeout
    }
}
```

### 5.2 Lease-Based Reads (Advanced)

The leader holds a time-based lease. As long as the lease has not expired, the leader knows no other leader exists and can serve reads directly without the heartbeat round-trip.

```mermaid
flowchart TD
    Read["Read Request"]
    Check{"Lease valid?<br/>(now < lease_expiry)"}
    Direct["Read directly from<br/>state machine<br/>(fast path)"]
    ReadIndex["Fall back to<br/>ReadIndex protocol<br/>(slow path)"]

    Read --> Check
    Check -->|Yes| Direct
    Check -->|No| ReadIndex
```

**Trade-off:** Lease-based reads are faster (no heartbeat round-trip) but depend on clock accuracy. If clocks are skewed, two leaders could have overlapping leases.

---

## Part 6: Test with Network Partitions

### 6.1 Simulated Network

**Task 10: Build a simulated transport layer.**

```go
type SimulatedTransport struct {
    mu         sync.Mutex
    nodes      map[uint64]*RaftNode
    partitions map[uint64]map[uint64]bool
    latencyMs  int
    dropRate   float64  // probability of dropping a message
}

func (st *SimulatedTransport) Send(msg Message) {
    st.mu.Lock()
    defer st.mu.Unlock()

    // Check partition
    if st.isPartitioned(msg.From, msg.To) {
        return // message dropped
    }

    // Simulate random drops
    if rand.Float64() < st.dropRate {
        return
    }

    // Deliver with simulated latency
    go func() {
        time.Sleep(time.Duration(st.latencyMs) * time.Millisecond)
        if node, ok := st.nodes[msg.To]; ok {
            node.Step(msg)
        }
    }()
}

func (st *SimulatedTransport) Partition(a, b uint64) {
    st.mu.Lock()
    defer st.mu.Unlock()
    if st.partitions[a] == nil {
        st.partitions[a] = make(map[uint64]bool)
    }
    if st.partitions[b] == nil {
        st.partitions[b] = make(map[uint64]bool)
    }
    st.partitions[a][b] = true
    st.partitions[b][a] = true
}

func (st *SimulatedTransport) Heal(a, b uint64) {
    st.mu.Lock()
    defer st.mu.Unlock()
    delete(st.partitions[a], b)
    delete(st.partitions[b], a)
}
```

### 6.2 Partition Test Scenarios

```mermaid
flowchart TD
    subgraph "Scenario 1: Leader Isolation"
        S1A["5 nodes, Node 1 is leader"]
        S1B["Write key=X to Node 1"]
        S1C["Isolate Node 1 from all others"]
        S1D["Nodes 2-5 elect new leader"]
        S1E["Write key=Y to new leader"]
        S1F["Heal partition"]
        S1G["Verify: Node 1 steps down,<br/>all nodes have X and Y"]

        S1A --> S1B --> S1C --> S1D --> S1E --> S1F --> S1G
    end
```

```mermaid
flowchart TD
    subgraph "Scenario 2: Minority Partition"
        S2A["5 nodes, Node 1 is leader"]
        S2B["Partition: {1,2} vs {3,4,5}"]
        S2C["Node 1 cannot commit<br/>(only 2 of 5)"]
        S2D["Node 3,4,5 elect new leader"]
        S2E["New leader serves writes"]
        S2F["Heal partition"]
        S2G["Verify consistency"]

        S2A --> S2B --> S2C --> S2D --> S2E --> S2F --> S2G
    end
```

```mermaid
flowchart TD
    subgraph "Scenario 3: Stale Read Prevention"
        S3A["3 nodes, Node 1 is leader"]
        S3B["Partition Node 1 from {2,3}"]
        S3C["Node 1 gets read request"]
        S3D["Node 1's ReadIndex heartbeat fails<br/>(cannot reach majority)"]
        S3E["Read returns error, not stale data"]

        S3A --> S3B --> S3C --> S3D --> S3E
    end
```

**Task 11: Implement partition tests.**

```go
func TestLeaderIsolation(t *testing.T) {
    cluster := NewTestCluster(5)
    cluster.Start()
    defer cluster.Stop()

    // Wait for leader election
    leader := cluster.WaitForLeader(t, 5*time.Second)

    // Write a value
    err := cluster.Put(leader, "key1", "value1")
    require.NoError(t, err)

    // Isolate the leader
    cluster.IsolateNode(leader)

    // Wait for new leader in the majority partition
    newLeader := cluster.WaitForNewLeader(t, leader, 5*time.Second)
    require.NotEqual(t, leader, newLeader)

    // Write through the new leader
    err = cluster.Put(newLeader, "key2", "value2")
    require.NoError(t, err)

    // Heal the partition
    cluster.HealAll()

    // Wait for convergence
    time.Sleep(2 * time.Second)

    // Verify all nodes have both values
    for _, node := range cluster.Nodes {
        v1, ok := node.KV.Get("key1")
        require.True(t, ok)
        require.Equal(t, "value1", v1)

        v2, ok := node.KV.Get("key2")
        require.True(t, ok)
        require.Equal(t, "value2", v2)
    }

    // Verify only one leader exists
    require.Equal(t, 1, cluster.CountLeaders())
}
```

---

## Part 7: Snapshotting (Bonus)

As the log grows, it must be compacted. Implement snapshotting to truncate the log.

```mermaid
flowchart TD
    subgraph "Snapshotting Process"
        Check["Log exceeds<br/>size threshold"]
        Snapshot["Serialize state machine<br/>to snapshot file"]
        Compact["Truncate log entries<br/>before snapshot index"]
        Meta["Store snapshot metadata<br/>(last included index/term)"]
        Install["If follower is far behind,<br/>send InstallSnapshot RPC<br/>instead of log entries"]

        Check --> Snapshot --> Compact --> Meta
        Meta --> Install
    end
```

```go
type Snapshot struct {
    LastIncludedIndex uint64
    LastIncludedTerm  uint64
    Data              []byte // Serialized state machine
}

func (kv *KVStore) CreateSnapshot() Snapshot {
    kv.mu.RLock()
    defer kv.mu.RUnlock()

    data, _ := json.Marshal(kv.data)
    return Snapshot{
        LastIncludedIndex: kv.appliedIndex,
        LastIncludedTerm:  kv.appliedTerm,
        Data:              data,
    }
}

func (kv *KVStore) RestoreFromSnapshot(snap Snapshot) {
    kv.mu.Lock()
    defer kv.mu.Unlock()

    kv.data = make(map[string]string)
    json.Unmarshal(snap.Data, &kv.data)
    kv.appliedIndex = snap.LastIncludedIndex
}
```

---

## Milestones and Evaluation Criteria

### Milestone 1: Leader Election (Week 1)
- [ ] Nodes start as followers
- [ ] Election timeout triggers candidate transition
- [ ] RequestVote RPC sent and processed correctly
- [ ] Exactly one leader elected in a healthy cluster
- [ ] Split votes resolved with randomized timeouts
- [ ] Higher-term messages cause step-down

### Milestone 2: Log Replication (Week 2)
- [ ] Leader appends client proposals to log
- [ ] AppendEntries RPC replicates entries to followers
- [ ] Log consistency check works (prevLogIndex/prevLogTerm)
- [ ] Leader decrements nextIndex on rejection
- [ ] Commitment after majority replication
- [ ] Committed entries applied to state machine

### Milestone 3: KV Store + API (Week 3)
- [ ] HTTP API for PUT, GET, DELETE
- [ ] Redirect non-leader requests to leader
- [ ] Wait for commitment before responding to client
- [ ] Linearizable reads via ReadIndex

### Milestone 4: Partition Testing (Week 4)
- [ ] Simulated network transport
- [ ] Leader isolation test passes
- [ ] Minority partition test passes
- [ ] Stale read prevention test passes
- [ ] Concurrent client test under partitions

---

## Architecture Summary

```mermaid
flowchart TB
    subgraph "Complete System Architecture"
        subgraph "Client Layer"
            HTTP["HTTP Client<br/>PUT/GET/DELETE"]
        end

        subgraph "Server Layer"
            API["HTTP API Server<br/>Routes requests,<br/>redirects to leader"]
        end

        subgraph "Consensus Layer"
            RaftMod["Raft State Machine<br/>Leader election,<br/>log replication,<br/>commitment tracking"]
        end

        subgraph "State Machine Layer"
            KV["KV Store<br/>Applies committed<br/>log entries"]
        end

        subgraph "Transport Layer"
            Trans["HTTP/gRPC Transport<br/>or Simulated Transport<br/>for testing"]
        end

        subgraph "Storage Layer"
            WAL["Write-Ahead Log<br/>Persists Raft state"]
            Snap["Snapshot Storage<br/>Periodic state snapshots"]
        end
    end

    HTTP --> API
    API --> RaftMod
    RaftMod --> KV
    RaftMod <--> Trans
    RaftMod --> WAL
    KV --> Snap
```

---

## Stretch Goals

1. **Cluster membership changes:** Implement AddNode/RemoveNode using Raft configuration changes.
2. **Persistent storage:** Write Raft state (term, votedFor, log) to disk so nodes can restart.
3. **Log compaction:** Implement snapshotting and InstallSnapshot RPC.
4. **Follower reads:** Allow stale reads from followers for reduced latency.
5. **Benchmarking:** Measure throughput (ops/sec) and latency (p50, p99) under various partition scenarios.
6. **Jepsen-style testing:** Use a linearizability checker to verify your system's correctness under concurrent operations.

---

## References

- [The Raft Paper (Extended Version)](https://raft.github.io/raft.pdf)
- [Raft Visualization](https://raft.github.io/)
- [etcd Raft Library Source Code](https://github.com/etcd-io/raft)
- [Students' Guide to Raft (MIT 6.824)](https://thesquareplanet.com/blog/students-guide-to-raft/)
- [Designing Data-Intensive Applications, Chapter 9](https://dataintensive.net/)
