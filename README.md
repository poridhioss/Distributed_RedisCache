# Distributed_Cache_Lab-03
**Consistent Hashing evolution**, using **exactly 4 virtual nodes per physical server** for the final example (small number chosen for clarity — real systems use 50–500+).

### 1. Starting Point: Naive Modulo-Based Sharding
We initially used a simple hash function:  
`server_index = hash(key) % num_servers`

**Problem**: Adding or removing even one server forces massive data movement.  
Example: 3 servers → add 4th server  
- Keys stay on the same server only if `hash(key) % 3 == hash(key) % 4`  
- Probability a key stays = **1/(n+1)** = **25%** (for n=3)  
→ **~75% of all data must be moved/reshuffled** — very expensive and slow.

### 2. Better Approach: Basic Consistent Hashing Ring (Fixed Size, e.g., 0–99)
We switched to a **ring** (circular hash space):  
- Fixed virtual ring of size **100** (positions 0 to 99)  
- Each physical server is placed at one position: `hash(server_name) % 100`  
- For a key: compute `pos = hash(key) % 100`, then find the **first server clockwise** (≥ pos, wrap around)

**Textual Ring Example** (3 servers placed at positions 20, 50, 80):

```
      0 ───────────── 99/0
     /                  \
   90                    10
  /                        \
80   ← redis-2             20   ← redis-0
  \                        /
   70                    30
     \                  /
      60 ───── 50 ───── 40
                ↑
             redis-1
```

**Improvement**: When adding a new server (e.g., redis-3 at 65), **only the keys between the previous owner and the new position move** → typically **~1/(n+1)** fraction of data moves (much better than 75%).

**Still Problems**:
- Uneven load: one server might own a large arc (e.g., 40% of the ring) → hotspots  
- **Cascading failure risk**: If redis-1 dies, its entire large arc suddenly goes to the next server → that server overloads → chain reaction

### 3. Final & Best Solution: Consistent Hashing with Virtual Nodes
To fix imbalance and cascading issues, we introduce **multiple virtual nodes** (replicas) per physical server, **randomly scattered** around the ring.

**How it works**:
1. For each physical server, create **k virtual nodes** (e.g., k=4)  
   Virtual node position = `hash(physical_server_name + "-" + i) % 100`  (i = 0 to 3)
2. Place all virtual nodes on the ring
3. Each ring position belongs to the **nearest clockwise virtual node**
4. That virtual node maps back to its **physical server**

**Example with 3 physical servers, 4 virtual nodes each** (12 virtual nodes total on ring 0–99):

| Virtual Node          | Hash Position (example) | Physical Server |
|-----------------------|--------------------------|-----------------|
| redis-0-v0            | 7                        | redis-0         |
| redis-0-v1            | 41                       | redis-0         |
| redis-0-v2            | 84                       | redis-0         |
| redis-0-v3            | 97                       | redis-0         |
| redis-1-v0            | 14                       | redis-1         |
| redis-1-v1            | 35                       | redis-1         |
| redis-1-v2            | 56                       | redis-1         |
| redis-1-v3            | 78                       | redis-1         |
| redis-2-v0            | 2                        | redis-2         |
| redis-2-v1            | 28                       | redis-2         |
| redis-2-v2            | 49                       | redis-2         |
| redis-2-v3            | 71                       | redis-2         |

→ Each physical server owns ~33 positions, but **scattered** → no large consecutive blocks.

**Simplified Ring View** (positions with owners):

```
0    2(redis-2)          7(redis-0)     14(redis-1)
          28(redis-2)               35(redis-1)
                    41(redis-0)           49(redis-2)
                              56(redis-1)
                                        71(redis-2)
                                                  78(redis-1)
                                                            84(redis-0)
                                                                      97(redis-0)
```

**Key Advantages**:
- **Excellent load balance**: With enough virtual nodes, each physical server gets almost equal share (variance drops dramatically)
- **No cascading failure**: If redis-1 dies, its 4 virtual nodes disappear → its ~33 positions are taken by **different neighboring virtual nodes** → load spreads across **all remaining servers**
- **Smooth scaling**: Adding redis-3 → create 4 new virtual nodes for it → only those ~4% of positions move → very little data reshuffling
- **Minimal disruption** on failure/add/remove

### Summary Table: Evolution of Approaches

| Approach                     | Data Movement on Add/Remove | Load Balance | Failure Isolation | Use Case Fit                  |
|------------------------------|------------------------------|--------------|-------------------|-------------------------------|
| Simple hash % n              | ~75% (n=3→4)                | Good         | Good              | Tiny/static clusters          |
| Basic ring (1 node/server)   | ~25% (1/(n+1))              | Poor         | Poor              | Small clusters, no hotspots   |
| Ring + Virtual Nodes (k=4+)  | ~1/(n+1) or better          | Excellent    | Excellent         | Production (Redis, Cassandra, Dynamo) |

This is why **consistent hashing with virtual nodes** became the standard in distributed caches (Redis Cluster), NoSQL databases (Cassandra, DynamoDB), and many load balancers.
