# Distributed Real-Time Drawing Board — Mini-RAFT

A fault-tolerant, real-time collaborative whiteboard backed by a custom
Mini-RAFT consensus protocol. Built with Node.js, Express, WebSockets,
and Docker. Three replica nodes maintain a shared stroke log through
leader election, log replication, and catch-up sync — with zero downtime
during failures and hot-reloads.

---

## Project Structure

```
miniraft/
├── docker-compose.yml          <- 1 gateway + 3 replicas + 1 frontend
├── README.md
├── kill-leader.sh              <- Auto-detect and kill current leader (failover demo)
├── blue-green-swap.sh          <- Zero-downtime rolling replica replacement
├── status.sh                   <- Live cluster state watcher
├── logs/
│   └── failover-events.log     <- Captured log demonstrating failover events
├── gateway/                    <- WebSocket server + leader routing
│   ├── Dockerfile
│   ├── package.json
│   ├── nodemon.json
│   └── index.js
├── replica1/                   <- RAFT replica node (Mini-RAFT implementation)
│   ├── Dockerfile
│   ├── package.json
│   ├── nodemon.json
│   └── index.js
├── replica2/                   <- Identical code, different env vars
├── replica3/
└── frontend/                   <- Static HTML canvas UI with cluster dashboard
    ├── Dockerfile
    └── index.html
```

---

## Port Reference

| Service             | Host Port |
|---------------------|-----------|
| Frontend (canvas)   | 8080      |
| Gateway (WS + HTTP) | 3000      |
| Replica 1           | 4001      |
| Replica 2           | 4002      |
| Replica 3           | 4003      |

---

## Prerequisites

```bash
# Install Docker and Docker Compose (Ubuntu)
sudo apt update
sudo apt install -y docker.io docker-compose-plugin curl python3
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# Log out and back in after the usermod step

# Verify
docker --version
docker compose version
```

---

## Quick Start

```bash
# 1. Unzip and enter the project
unzip miniraft.zip
cd miniraft

# 2. Make helper scripts executable
chmod +x kill-leader.sh blue-green-swap.sh status.sh

# 3. Build and start all containers
docker compose up --build
```

Open **http://localhost:8080** in your browser (or multiple tabs) to draw.

---

## Mini-RAFT Protocol Summary

### Node States
- **Follower** -- waits for leader heartbeats; resets election timer on each.
- **Candidate** -- increments term, requests votes from peers.
- **Leader** -- handles stroke replication and commits; sends heartbeats every 150 ms.

### Election
- Election timeout: random **500-800 ms**
- If a follower misses a heartbeat -> becomes candidate
- Candidate requests votes; becomes leader on **majority (>= 2 out of 3)**
- Higher term always wins; split votes retry automatically

### Log Replication
```
Client -> Gateway -> Leader
Leader:  append to log -> AppendEntries to followers -> await majority ACK -> commit -> broadcast
```

### Catch-Up (Restarted Node)
1. Restarted node starts as follower with empty log
2. AppendEntries prevLogIndex check fails -> follower replies with its log length
3. Leader calls /sync-log sending all committed entries from that index onward
4. Follower appends missing entries, updates commitIndex -> in sync

### Safety Rules Implemented
- Committed entries are **never overwritten** (wouldOverwriteCommitted guard)
- Higher term always causes immediate step-down (enforced on leaders too)
- Log up-to-date check in RequestVote (Raft section 5.4.1)
- Conflict-detection and safe rollback in AppendEntries (Raft section 5.3)
- Commit lock prevents concurrent replication races for the same index

---

## RPC Endpoints (per replica)

| Endpoint          | Method | Description                                      |
|-------------------|--------|--------------------------------------------------|
| /request-vote     | POST   | Vote request during election                     |
| /append-entries   | POST   | Log replication from leader                      |
| /heartbeat        | POST   | Periodic liveness + commitIndex update           |
| /sync-log         | POST   | Full catch-up sync for rejoining node            |
| /stroke           | POST   | Stroke ingestion (called by gateway)             |
| /state            | GET    | Node state: role, term, logLength, commitIndex   |
| /committed-log    | GET    | Full committed log (used for canvas replay)      |
| /health           | GET    | Health check                                     |

---

## Demo Scenarios

### 1. Multi-client drawing
Open http://localhost:8080 in two or more browser tabs. Strokes appear
instantly on all canvases. The sidebar shows live leader, term, log size,
and commit index for each replica.

### 2. Kill the leader (failover demo)
```bash
./kill-leader.sh
```
Identifies the current leader, stops it, waits 2 s for election, then restarts it.
Clients stay connected; the gateway re-routes to the new leader automatically.
Canvas state is preserved across failover.

### 3. Hot-reload (bind-mount demo)
While the system is running, edit any file in replica1/, replica2/, or replica3/
(e.g. add a comment). nodemon detects the change, gracefully restarts the container,
the surviving replicas maintain quorum, and the reloaded replica rejoins via catch-up sync.

```bash
# Example -- trigger a hot-reload on replica3
echo "// hot-reload test $(date)" >> replica3/index.js
```

### 4. Zero-downtime rolling replacement (blue-green swap)
```bash
# Swap all replicas one by one
./blue-green-swap.sh all

# Or swap a single replica
./blue-green-swap.sh replica2
```
Each replica is stopped, a new leader is elected on the remaining two, the
stopped replica restarts and catches up before the next one is swapped.

### 5. Live cluster status
```bash
./status.sh
```
Prints a table of ID | ROLE | TERM | LOG | COMMIT refreshed every second.

---

## Observability

All replicas log every state transition, vote, and commit in the format:
```
[HH:MM:SS.mmm][replicaN][ROLE][Tterm] message
```

Example output:
```
[08:00:00.612][replica2][FOLLOWER][T0] -> CANDIDATE (term=1)
[08:00:00.617][replica2][CANDIDATE][T1] -> LEADER
[08:00:05.218][replica2][LEADER][T1] Stroke committed at index 0
[08:00:10.607][replica1][CANDIDATE][T2] -> LEADER
[08:00:12.363][replica2][FOLLOWER][T2] sync-log: restored 2 entries [0..1] -> commitIndex=1
```

See logs/failover-events.log for a full annotated example covering startup,
failover, hot-reload, and blue-green swap.

---

## Docker Details

### How bind-mounts + nodemon work
Each replica mounts its source folder into the container:
```yaml
volumes:
  - ./replica1:/app
  - /app/node_modules   # anonymous volume keeps node_modules isolated
```
package.json starts nodemon with SIGTERM for graceful shutdown. Editing
any .js or .json file causes the container process to restart in place -- the
container itself stays up, so docker compose never sees a failure.

### Adding a fourth replica
1. Copy replica3/ to replica4/.
2. Add a replica4 service in docker-compose.yml with REPLICA_ID=replica4, REPLICA_PORT=4004.
3. Add http://replica4:4004 to the PEERS lists for all other replicas.
4. Add http://replica4:4004 to the gateway REPLICAS env var.
5. docker compose up --build replica4

The majority threshold (floor((N+1)/2)+1) auto-adjusts -- with 4 replicas it becomes 3.
