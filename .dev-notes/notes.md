# RaftKV — implementation notes

Personal reference document. Roadmaps, what to study at each phase, and reference links.

---

## Architecture

Two cleanly separated layers:

- **`raft/`** — pure state machine, no I/O, no network. Receives messages via `Step()`, returns a `Ready` struct describing what needs to happen next (entries to persist, messages to send, snapshots to apply). Fully testable without opening a TCP connection.
- **`node/`** — the driver. Consumes `Ready`: writes to WAL, sends messages over gRPC, applies committed entries to the state machine.
- **`kvstore/`** — the state machine. A replicated `map[string]string` with linearizable semantics on top of the node.

Inspired by the `etcd/raft` design: consensus core is fully decoupled from transport and storage.

---

## Raft roadmap

### Phase 1 — foundation

**What to build:**
- State enum: Follower, Candidate, Leader
- Persistent fields from the paper: `currentTerm`, `votedFor`, `log[]`
- Simple WAL to persist those fields with `fsync`
- Election timeout timer with randomized jitter (150–300ms)
- gRPC transport with `AppendEntries` and `RequestVote`
- Central event loop using `select`

**What to study:**
- Raft paper, sections 2 and 3 — read at least twice before writing a single line
- Figure 2 of the paper — the complete algorithm spec on one page, return to it whenever stuck
- `etcd/raft/doc.go` and `etcd/raft/raftpb/raft.proto` — understand the interface design and message types before defining your own
- Ongaro dissertation, chapter 3 — more detailed than the paper on data structures

**Books:**
- *Designing Data-Intensive Applications* (Kleppmann) — chapters 8 and 9. Read before phase 1. Chapter 8 covers everything that can go wrong in distributed systems; chapter 9 covers consistency, linearizability, and consensus. Gives you the mental model for why all of this is necessary before you write a line.
- *Database Internals* (Petrov) — part 2, chapters 13–15. Covers replication, consensus, and leader election from a systems perspective. Complements the Raft paper well on the fundamentals of replicated state machines.

---

### Phase 2 — leader election

**What to build:**
- `RequestVote` RPC: candidate increments term, votes for itself, fires requests to all peers
- Vote grant logic: only grant if `votedFor == null || == candidateId` AND candidate's log is at least as up-to-date
- Up-to-date check: compare `lastLogTerm` first, then `lastLogIndex` — order matters
- Convert to Follower on any RPC with a higher term

**What to study:**
- Paper, section 5.2 — leader election
- Paper, section 5.4.1 — election restriction and the precise definition of "at least as up-to-date"
- Interactive visualization at [raft.github.io](https://raft.github.io) — watch an election happen before implementing
- [The Secret Lives of Data](http://thesecretlivesofdata.com/raft/) — more guided, good starting point

**Books:**
- *Designing Data-Intensive Applications* (Kleppmann) — chapter 9, section on leader election and fencing tokens. Explains why a single leader matters and the subtle ways it can break.

---

### Phase 3 — log replication

**What to build:**
- `AppendEntries` RPC: leader sends entries with `prevLogIndex` and `prevLogTerm` — follower checks consistency before accepting
- `nextIndex` and `matchIndex` per follower
- `commitIndex` advance: only when a majority has `matchIndex >= N` AND `log[N].term == currentTerm`
- Heartbeat: empty `AppendEntries` periodically to suppress follower election timeouts
- Fast `nextIndex` rollback: follower returns `conflictTerm` and `conflictIndex` to skip entries in bulk

**What to study:**
- Paper, section 5.3 — log replication
- Paper, section 5.4.2 and Figure 8 — committing entries from previous terms, critical reading
- Dissertation, chapter 3, log matching section — the proof of the Log Matching Property explains why `prevLogIndex/Term` works
- Dissertation, section on accelerated log backtracking — the `conflictTerm` optimization

**Books:**
- *Database Internals* (Petrov) — chapter 14, replication and consistency. The WAL and log replication sections map directly to what you're building.

---

### Phase 4 — snapshots and log compaction

**What to build:**
- `InstallSnapshot` RPC: leader sends snapshot to lagging followers
- Snapshot interface: state machine exposes `Snapshot()` and `Restore()`
- Log truncation after snapshot, keeping `lastIncludedIndex` and `lastIncludedTerm`
- Boot restore: load snapshot then replay log entries after it
- Chunked transfer for large snapshots: `offset` + `done` flag in the RPC

**What to study:**
- Paper, section 7 — log compaction
- Dissertation, chapter 5 — more detailed, covers `InstallSnapshot` edge cases
- `etcd/raft/log.go` — how they separate the unstable log from persisted storage

**Books:**
- *Database Internals* (Petrov) — chapter 7, log-structured storage. Covers WAL, compaction strategies, and how databases manage growing logs on disk. Directly relevant to how your snapshot + log truncation should work.

---

### Phase 5 — linearizable reads and membership change

**What to build:**
- ReadIndex: leader records current `commitIndex`, sends a heartbeat round to confirm it's still leader, waits for `appliedIndex >= commitIndex`, then replies
- Single-server membership change: add or remove one server at a time — mathematically safe without joint consensus

**What to study:**
- Dissertation, chapter 6 — client interaction, ReadIndex, lease reads, and the trade-offs of each approach
- Dissertation, chapter 4 — single-server membership change (only in the dissertation, not the paper)
- Errata at [ongardie/dissertation](https://github.com/ongardie/dissertation) — there is a documented bug in single-server changes with the fix

**Books:**
- *Designing Data-Intensive Applications* (Kleppmann) — chapter 9, section on linearizability vs serializability. Clarifies the exact guarantee ReadIndex provides and why it's different from just reading locally.

---

## KV store roadmap

### Phase 1 — state machine and Raft interface

**What to build:**
- `PUT`, `GET`, `DELETE`, `CAS` commands as serializable structs (gob or protobuf)
- Apply loop: goroutine that reads from `applyCh` and applies commands to the map in order
- Submit and wait: submit via `Start()`, receive an index, wait for that index to appear in the apply loop
- Leader redirect: return `ErrNotLeader` with a hint pointing to the current leader

**What to study:**
- Dissertation, chapter 6 — client interaction, the complete flow of how a client talks to a Raft cluster
- MIT 6.824 Lab 3: [nil.csail.mit.edu/6.824/2022/labs/lab-kvraft.html](http://nil.csail.mit.edu/6.824/2022/labs/lab-kvraft.html) — the interface between KV server and Raft is a good design reference
- Eli Bendersky, part 4: [eli.thegreenplace.net/2024/implementing-raft-part-4-keyvalue-database](https://eli.thegreenplace.net/2024/implementing-raft-part-4-keyvalue-database/)

**Books:**
- *Designing Data-Intensive Applications* (Kleppmann) — chapter 9, section on linearizability. The definition of "every operation appears to take effect atomically at a single point in time" is exactly what your KV store needs to satisfy.

---

### Phase 2 — exactly-once semantics

**What to build:**
- Unique `ClientID` per client and monotonic `SeqNum` per request — both included in every command
- Dedup table on the server: `map[clientID]lastSeqNum`
- Cached last response per client to return on retry
- Cache GC: on receiving `SeqNum N`, discard all cached entries with `SeqNum < N`

**What to study:**
- Paper, section 8 — client interaction, the duplicate command problem is described here
- Dissertation, chapter 6, section "Implementing linearizable semantics"
- Eli Bendersky, part 5 — covers exactly this phase

**Books:**
- *Designing Data-Intensive Applications* (Kleppmann) — chapter 9, section on "implementing linearizable storage using total order broadcast". Explains exactly-once semantics and idempotency in the context of replicated state machines.

---

### Phase 3 — linearizable reads

**What to build:**
- `GET` via log (simple and correct): treat as a write — slow but linearizable
- ReadIndex (efficient): leader confirms majority with a heartbeat before replying

**What to study:**
- Dissertation, chapter 6, section "Read-only queries" — three approaches with trade-offs: via log, ReadIndex, and lease reads

**Books:**
- *Designing Data-Intensive Applications* (Kleppmann) — chapter 9, linearizability section. The distinction between linearizable reads and stale reads is explained with concrete examples that map directly to this phase.

---

### Phase 4 — KV store snapshots

**What to build:**
- Size-based trigger: when the Raft log exceeds N bytes, take a snapshot
- Serialization: map + dedup table together in the same snapshot
- Restore: replace the entire state and update `lastApplied`

**What to study:**
- MIT 6.824 Lab 3B — covers exactly the snapshot integration in the KV store

**Books:**
- *Database Internals* (Petrov) — chapter 7. Compaction and snapshot strategies in storage engines. The concepts map directly to how your KV state machine should serialize and restore state.

---

### Phase 5 — HTTP API and chaos tests

**What to build:**
- `PUT /keys/:k`, `GET /keys/:k`, `DELETE /keys/:k`, `CAS /keys/:k`
- Client with retry: follows `ErrNotLeader` hints, backoff on timeout
- History recorder: `{start_time, end_time, input, output}` per operation
- Chaos suite: 100 concurrent clients with random partitions and crashes, linearizability verified with Porcupine

**What to study:**
- [Porcupine](https://github.com/anishathalye/porcupine) — docs and KV model examples
- Author's blog post: [anishathalye.com/testing-distributed-systems-for-linearizability](https://anishathalye.com/testing-distributed-systems-for-linearizability/) — how to integrate the checker and why it catches bugs that ad-hoc tests miss

**Books:**
- *Designing Data-Intensive Applications* (Kleppmann) — chapter 8, section on detecting faults. Gives intuition for what your chaos suite is actually testing and what classes of bugs it should catch.

---

## References

### Required reading — in this order

| Document | Link | When to read |
|---|---|---|
| Raft paper (extended version) | [raft.github.io/raft.pdf](https://raft.github.io/raft.pdf) | Before everything. Read 3 times. |
| Ongaro dissertation | [web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf) | Chapters 3, 4, 6 — before each corresponding phase |
| Dissertation errata | [github.com/ongardie/dissertation](https://github.com/ongardie/dissertation) | Before implementing membership change |

### Tools

| Tool | Link | Purpose |
|---|---|---|
| Porcupine | [github.com/anishathalye/porcupine](https://github.com/anishathalye/porcupine) | Linearizability checker — the correctness proof artifact |
| Porcupine blog post | [anishathalye.com/testing-distributed-systems-for-linearizability](https://anishathalye.com/testing-distributed-systems-for-linearizability/) | How to integrate the checker |

### Implementation references

| Resource | Link | Purpose |
|---|---|---|
| MIT 6.824 Lab Raft | [pdos.csail.mit.edu/6.824/labs/lab-raft1.html](https://pdos.csail.mit.edu/6.824/labs/lab-raft1.html) | Reference interface and test suite |
| MIT 6.824 Lab KV | [nil.csail.mit.edu/6.824/2022/labs/lab-kvraft.html](http://nil.csail.mit.edu/6.824/2022/labs/lab-kvraft.html) | Interface between KV server and Raft |
| Eli Bendersky — part 4 | [eli.thegreenplace.net/2024/implementing-raft-part-4-keyvalue-database](https://eli.thegreenplace.net/2024/implementing-raft-part-4-keyvalue-database/) | KV store on top of Raft in Go |
| etcd/raft | [github.com/etcd-io/etcd/tree/main/raft](https://github.com/etcd-io/etcd/tree/main/raft) | Read `doc.go` and `raftpb/raft.proto` before starting — not during |

### Visualizations

| Resource | Link |
|---|---|
| RaftScope — interactive cluster | [raft.github.io](https://raft.github.io) |
| The Secret Lives of Data | [thesecretlivesofdata.com/raft](http://thesecretlivesofdata.com/raft/) |

### Books

Two books cover the entire project. You don't need to read them cover to cover — the relevant chapters are referenced in each phase above.

**Designing Data-Intensive Applications** — Martin Kleppmann
The best book on distributed systems for practitioners. Chapters 8 and 9 are the most relevant: chapter 8 covers everything that can go wrong in distributed systems (clocks, networks, process pauses), chapter 9 covers consistency models, linearizability, and consensus algorithms including Raft. Read chapters 8–9 before starting phase 1. Return to specific sections as needed during the KV store phases.
→ [dataintensive.net](https://dataintensive.net/)

**Database Internals** — Alex Petrov
Goes deeper into storage engine internals and distributed database algorithms than DDIA. Part 2 (chapters 12–16) covers replication, consensus, and leader election in detail. More technical and closer to what you're actually implementing. Particularly useful for phases 3 and 4 of the Raft roadmap (log replication, snapshots) and phase 4 of the KV store (snapshots, compaction).
→ [oreilly.com/library/view/database-internals](https://www.oreilly.com/library/view/database-internals/9781492040330/)
