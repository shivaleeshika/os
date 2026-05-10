# os
# OS Lab — Master Reference Document

All 4 programs: Producer-Consumer, Banker's Algorithm, CPU Scheduling, Page Replacement.
Each section: concept → fully commented code → dry run.

---

# 1. PRODUCER-CONSUMER (Semaphore + Mutex Simulation)

## Concept

A shared circular buffer of size MAX sits between a producer and consumer.
- **Producer** puts items into the buffer
- **Consumer** takes items out
- **Semaphores** control access — `emp` counts empty slots, `full` counts filled slots
- **Mutex** is a lock — ensures only one operation touches the buffer at a time

**Why mutex on top of semaphores?**
Semaphores tell you *if* you can enter. Mutex ensures no two operations *overlap* inside the buffer. Without mutex, two producers could write to the same slot simultaneously.

**Critical rule — order of operations:**
```
Producer:  wait(emp) → lock → WRITE → unlock → signal(full)
Consumer:  wait(full) → lock → READ  → unlock → signal(emp)
```
Lock always comes AFTER the semaphore wait. Reversing this causes deadlock.

## Code

```cpp
#include <iostream>
using namespace std;

// Buffer setup — circular array of size MAX
// 'in' = next position to write (producer moves this forward)
// 'out' = next position to read (consumer moves this forward)
const int MAX = 5;
int buffer[MAX];
int in = 0, out = 0;

// Counting semaphores (simulated as integers)
// full  = how many slots currently have data  (starts at 0, max MAX)
// emp   = how many slots are currently empty  (starts at MAX, min 0)
int full = 0;
int emp = MAX;

// Mutex — binary lock (true = someone is in critical section, false = free)
// In real threading this would be pthread_mutex_t or std::mutex
bool mutex = false;

// wait(s) — tries to decrement s
// If s == 0, nothing to consume/produce — print blocked message
// In real OS this would suspend the thread; here we just warn and skip
void wait(int &s) {
    if (s > 0) s--;
    else cout << "WAITING (operation not allowed now)\n";
}

// signal(s) — increments s
// Tells waiting threads that a resource is now available
void signal(int &s) { s++; }

// lock() — acquire the mutex before entering critical section
// If already locked, another operation is inside — print blocked message
void lock()   { if (!mutex) mutex = true; else cout << "MUTEX LOCKED\n"; }

// unlock() — release the mutex after leaving critical section
void unlock() { mutex = false; }

// PRODUCER
// Step 1: wait(emp)  — check if there's an empty slot, claim it
// Step 2: lock()     — enter critical section (touch buffer safely)
// Step 3: write item into buffer[in], advance 'in' circularly
// Step 4: unlock()   — leave critical section
// Step 5: signal(full) — tell consumer one more item is ready
void producer(int item) {
    if (emp == 0) { cout << "Buffer FULL! Produce not possible.\n"; return; }

    wait(emp);   // claim an empty slot
    lock();      // enter critical section

    buffer[in] = item;          // write item at current write position
    in = (in + 1) % MAX;        // advance 'in' circularly (wraps 4→0 if MAX=5)

    unlock();    // leave critical section
    signal(full); // one more slot is now filled

    cout << "Produced: " << item << endl;
}

// CONSUMER
// Step 1: wait(full)  — check if there's a filled slot, claim it
// Step 2: lock()      — enter critical section
// Step 3: read item from buffer[out], advance 'out' circularly
// Step 4: unlock()    — leave critical section
// Step 5: signal(emp) — tell producer one more slot is free
void consumer() {
    if (full == 0) { cout << "Buffer EMPTY! Consume not possible.\n"; return; }

    wait(full);   // claim a filled slot
    lock();       // enter critical section

    int item = buffer[out];     // read item at current read position
    out = (out + 1) % MAX;      // advance 'out' circularly

    unlock();    // leave critical section
    signal(emp); // one more slot is now empty

    cout << "Consumed: " << item << endl;
}

// DISPLAY — show current buffer contents from out to out+full
void display() {
    cout << "Buffer: ";
    if (full == 0) { cout << "EMPTY\n"; return; }
    int temp = out;
    for (int i = 0; i < full; i++) {
        cout << buffer[temp] << " | ";
        temp = (temp + 1) % MAX;  // walk circularly through filled slots
    }
    cout << endl;
}

int main() {
    int choice, item;
    while (1) {
        cout << "\n1. Produce\n2. Consume\n3. Display\n4. Exit\nChoose: ";
        cin >> choice;
        switch (choice) {
            case 1: cout << "Enter item: "; cin >> item; producer(item); break;
            case 2: consumer(); break;
            case 3: display();  break;
            case 4: return 0;
            default: cout << "Invalid choice!\n";
        }
    }
}
```

## Dry Run

**Setup:** MAX=5, buffer=[_, _, _, _, _], in=0, out=0, emp=5, full=0

**Step 1: Produce(10)**
- emp=5 > 0, so allowed
- wait(emp) → emp becomes 4
- lock() → mutex=true
- buffer[0]=10, in becomes 1
- unlock() → mutex=false
- signal(full) → full becomes 1
- State: buffer=[10,_,_,_,_], in=1, out=0, emp=4, full=1

**Step 2: Produce(20)**
- wait(emp) → emp=3
- buffer[1]=20, in=2
- signal(full) → full=2
- State: buffer=[10,20,_,_,_], in=2, out=0, emp=3, full=2

**Step 3: Consume()**
- full=2 > 0, allowed
- wait(full) → full=1
- lock() → mutex=true
- item = buffer[0] = 10, out becomes 1
- unlock() → mutex=false
- signal(emp) → emp=4
- Output: "Consumed: 10"
- State: buffer=[10,20,_,_,_], in=2, out=1, emp=4, full=1
  *(buffer[0] still holds 10 but out has moved past it — it's effectively gone)*

**Step 4: Consume() again**
- item = buffer[1] = 20, out becomes 2
- State: emp=5, full=0

**Step 5: Consume() on empty**
- full=0 → "Buffer EMPTY! Consume not possible."

---

# 2. BANKER'S ALGORITHM (Deadlock Avoidance)

## Concept

Before giving resources to any process, the OS checks: "If I grant this, can the system still reach a state where every process eventually finishes?"

Three matrices:
- `maxNeed[i][j]` — maximum units of resource j that process i will ever ask for
- `alloc[i][j]`   — what process i currently holds
- `need[i][j]`    — what process i still needs = maxNeed - alloc

**Safety Algorithm:**
1. Set `work[] = avail[]` (simulate available resources)
2. Find any unfinished process whose `need[] <= work[]`
3. If found: pretend it finishes, add its `alloc[]` back to `work[]`, mark done
4. Repeat until all done (SAFE) or no progress possible (UNSAFE)

## Code

```cpp
#include <iostream>
using namespace std;

int main() {
    int p, r;
    // p = number of processes, r = number of resource types
    int avail[10];        // avail[j] = available units of resource j right now
    int maxNeed[10][10];  // maxNeed[i][j] = max resource j that process i can ever need
    int alloc[10][10];    // alloc[i][j] = resource j currently given to process i
    int need[10][10];     // need[i][j] = how much more of j process i might still request
                          //            = maxNeed[i][j] - alloc[i][j]

    cout << "Enter number of processes and resources: ";
    cin >> p >> r;

    // Read currently available resources
    cout << "Enter Available resources (" << r << " values): ";
    for (int j = 0; j < r; j++) cin >> avail[j];

    // Read maximum demand matrix row by row (each row = one process)
    cout << "Enter Maximum matrix (" << p << " x " << r << "):\n";
    for (int i = 0; i < p; i++)
        for (int j = 0; j < r; j++) cin >> maxNeed[i][j];

    // Read allocation matrix and compute need matrix on the fly
    // need[i][j] = maxNeed[i][j] - alloc[i][j]
    // (how much more does process i still need of resource j)
    cout << "Enter Allocation matrix (" << p << " x " << r << "):\n";
    for (int i = 0; i < p; i++)
        for (int j = 0; j < r; j++) {
            cin >> alloc[i][j];
            need[i][j] = maxNeed[i][j] - alloc[i][j];
        }

    // --- SAFETY ALGORITHM STARTS HERE ---

    bool done[10] = {false};  // done[i] = true when process i has "finished" in simulation
    int work[10];             // work[] = simulated available resources (copy of avail)
    for (int j = 0; j < r; j++) work[j] = avail[j];

    int safeSeq[10];  // stores the safe execution order
    int count = 0;    // how many processes have been safely "finished" so far

    // Keep scanning all processes until no more progress can be made
    while (count < p) {
        bool progress = false;  // did we finish at least one process this pass?

        for (int i = 0; i < p; i++) {
            if (done[i]) continue;  // already finished, skip

            // Check if process i's remaining need can be satisfied by work[]
            // i.e., need[i][j] <= work[j] for ALL resource types j
            bool canRun = true;
            for (int j = 0; j < r; j++)
                if (need[i][j] > work[j]) { canRun = false; break; }

            if (canRun) {
                // Simulate process i finishing:
                // it releases all its currently allocated resources back to work[]
                for (int j = 0; j < r; j++) work[j] += alloc[i][j];

                done[i] = true;          // mark as finished
                safeSeq[count++] = i;    // add to safe sequence
                progress = true;         // we made progress, keep scanning
            }
        }

        // If no process could run this entire pass, we're stuck — UNSAFE
        if (!progress) break;
    }

    // If all processes finished in simulation → system is in safe state
    if (count == p) {
        cout << "\nSAFE STATE\nSafe Sequence: ";
        for (int i = 0; i < p; i++)
            cout << "P" << safeSeq[i] + 1 << (i < p-1 ? " -> " : "\n");
    } else {
        cout << "\nNOT SAFE — Deadlock possible\n";
    }

    return 0;
}
```

## Dry Run

**Input:**
- 3 processes (P1, P2, P3), 2 resource types (A, B)
- avail = [3, 2]

| Process | Max  | Alloc | Need |
|---------|------|-------|------|
| P1      | 4, 3 | 1, 0  | 3, 3 |
| P2      | 2, 2 | 2, 1  | 0, 1 |
| P3      | 3, 1 | 0, 1  | 3, 0 |

**Start:** work = [3, 2], done = [F, F, F]

**Pass 1:**
- P1: need=[3,3], work=[3,2] → 3>2 ✗ cannot run
- P2: need=[0,1], work=[3,2] → 0≤3 and 1≤2 ✓ can run
  - work = [3+2, 2+1] = [5, 3], done[1]=true, safeSeq=[P2]
- P3: need=[3,0], work=[5,3] → 3≤5 and 0≤3 ✓ can run
  - work = [5+0, 3+1] = [5, 4], done[2]=true, safeSeq=[P2, P3]

**Pass 2:**
- P1: need=[3,3], work=[5,4] → 3≤5 and 3≤4 ✓ can run
  - work = [5+1, 4+0] = [6, 4], done[0]=true, safeSeq=[P2, P3, P1]

**Result:** SAFE STATE → Safe Sequence: P2 -> P3 -> P1

---

# 3. CPU SCHEDULING (FCFS, SJF, Priority, Round Robin)

## Concept

CPU has one core. Multiple processes arrive and need CPU time. The OS must decide the order.

Key terms:
- `at` = arrival time (when process enters system)
- `bt` = burst time (how long it needs CPU)
- `pr` = priority (lower number = higher priority)
- `ft` = finish time (when it completes)
- `TAT` = Turnaround Time = ft - at (total time in system)
- `WT`  = Waiting Time = TAT - bt (time spent waiting, not running)

**FCFS** — run in arrival order. No choice, just queue.
**SJF**  — always pick the shortest job available. Minimizes average WT.
**Priority** — pick highest priority (lowest number) available.
**RR** — give each process a fixed time slice (quantum), cycle through.

Note: SJF and Priority are identical code — only the comparison changes (`bt` vs `pr`).

## Code

```cpp
#include <iostream>
#include <queue>
using namespace std;

// Global arrays — all processes share these
// id[i] = process ID, at[i] = arrival time, bt[i] = burst time, pr[i] = priority
int n, q;  // n = number of processes, q = time quantum for RR
int id[10], at[10], bt[10], pr[10];

// stats() — prints WT, TAT for each process and averages
// order[] = the index of which process finished at position i
// ft[]    = finish time of process at order[i]
// cnt     = how many processes to print (= n for non-RR)
void stats(int order[], int ft[], int cnt) {
    cout << "PID\tWT\tTAT\n";
    float sw = 0, st = 0;
    for (int i = 0; i < cnt; i++) {
        int o = order[i];                    // o is the index into global at[], bt[]
        int tat = ft[i] - at[o];            // TAT = finish - arrival
        int wt  = tat - bt[o];              // WT  = TAT - burst
        cout << "P" << id[o] << "\t" << wt << "\t" << tat << "\n";
        sw += wt; st += tat;
    }
    cout << "Avg WT=" << sw/n << " Avg TAT=" << st/n << "\n";
}

// FCFS — First Come First Served
// Sort all processes by arrival time, then run them one by one in that order.
// If CPU is idle when next process arrives, jump clock to its arrival time.
void fcfs() {
    // Selection sort by arrival time (at[])
    for (int i = 0; i < n-1; i++)
        for (int j = i+1; j < n; j++)
            if (at[j] < at[i]) {
                // swap all arrays together so data stays aligned
                swap(at[i],at[j]); swap(bt[i],bt[j]);
                swap(id[i],id[j]); swap(pr[i],pr[j]);
            }

    int ct = 0;       // ct = current clock time
    int order[10], ft[10];
    cout << "\n=== FCFS ===\nGantt: ";
    for (int i = 0; i < n; i++) {
        if (ct < at[i]) ct = at[i];  // CPU idle gap — jump clock to arrival
        cout << "P" << id[i] << "(" << ct << "-";
        ct += bt[i];                  // run process to completion
        cout << ct << ") ";
        order[i] = i; ft[i] = ct;    // record finish time
    }
    cout << "\n";
    stats(order, ft, n);
}

// SJF — Shortest Job First (Non-Preemptive)
// At each step, from all arrived and unfinished processes, pick the one
// with the smallest burst time. Run it to completion.
void sjf() {
    bool done[10] = {};  // done[i] = process i has finished
    int ct = at[0], cnt = 0, order[10], ft[10];
    // start clock at earliest arrival (not necessarily at[0] after sorting)
    for (int i = 1; i < n; i++) if (at[i] < ct) ct = at[i];

    cout << "\n=== SJF ===\nGantt: ";
    while (cnt < n) {
        int idx = -1;
        for (int i = 0; i < n; i++)
            if (!done[i] && at[i] <= ct)           // must have arrived
                if (idx == -1 || bt[i] < bt[idx])  // pick shortest burst
                    idx = i;

        if (idx == -1) { ct++; continue; }  // no process ready, advance clock

        cout << "P" << id[idx] << "(" << ct << "-";
        ct += bt[idx];  // run selected process to completion
        cout << ct << ") ";
        order[cnt] = idx; ft[cnt++] = ct;
        done[idx] = true;
    }
    cout << "\n";
    stats(order, ft, n);
}

// PRIORITY — Non-Preemptive Priority Scheduling
// Identical to SJF — only difference: pick lowest pr[i] instead of lowest bt[i]
void priority() {
    bool done[10] = {};
    int ct = at[0], cnt = 0, order[10], ft[10];
    for (int i = 1; i < n; i++) if (at[i] < ct) ct = at[i];

    cout << "\n=== PRIORITY ===\nGantt: ";
    while (cnt < n) {
        int idx = -1;
        for (int i = 0; i < n; i++)
            if (!done[i] && at[i] <= ct)
                if (idx == -1 || pr[i] < pr[idx])  // pick lowest priority number
                    idx = i;

        if (idx == -1) { ct++; continue; }

        cout << "P" << id[idx] << "(" << ct << "-";
        ct += bt[idx];
        cout << ct << ") ";
        order[cnt] = idx; ft[cnt++] = ct;
        done[idx] = true;
    }
    cout << "\n";
    stats(order, ft, n);
}

// ROUND ROBIN — Time-sliced scheduling
// Each process gets at most q time units. If not done, goes back to end of queue.
// New arrivals during a slice are added to the queue in arrival order.
void rr() {
    // Sort by arrival so queue is seeded in correct order
    for (int i = 0; i < n-1; i++)
        for (int j = i+1; j < n; j++)
            if (at[j] < at[i]) {
                swap(at[i],at[j]); swap(bt[i],bt[j]);
                swap(id[i],id[j]); swap(pr[i],pr[j]);
            }

    int rem[10], ft[10] = {};   // rem[i] = remaining burst time of process i
    bool inq[10] = {};          // inq[i] = has process i entered the queue yet
    for (int i = 0; i < n; i++) rem[i] = bt[i];

    queue<int> rq;   // ready queue holds indices of processes
    int ct = at[0];  // start clock at first arrival
    rq.push(0); inq[0] = true;  // seed queue with first process

    cout << "\n=== ROUND ROBIN (q=" << q << ") ===\nGantt: ";
    while (!rq.empty()) {
        int i = rq.front(); rq.pop();  // take next process from front

        int use = min(rem[i], q);      // run for min(remaining, quantum)
        cout << "P" << id[i] << "(" << ct << "-";
        ct += use; rem[i] -= use;      // advance clock, reduce remaining
        cout << ct << ") ";

        // Enqueue any process that arrived during this slice (and not yet queued)
        for (int j = 0; j < n; j++)
            if (!inq[j] && at[j] <= ct && rem[j] > 0)
                { rq.push(j); inq[j] = true; }

        if (rem[i] == 0) ft[i] = ct;  // done — record finish time
        else rq.push(i);               // not done — go back to end of queue
    }
    cout << "\n";

    // RR stats — ft[] is indexed directly by process index (not order[])
    cout << "PID\tWT\tTAT\n";
    float sw = 0, st = 0;
    for (int i = 0; i < n; i++) {
        int tat = ft[i] - at[i], wt = tat - bt[i];
        cout << "P" << id[i] << "\t" << wt << "\t" << tat << "\n";
        sw += wt; st += tat;
    }
    cout << "Avg WT=" << sw/n << " Avg TAT=" << st/n << "\n";
}

int main() {
    cout << "Processes: "; cin >> n;
    cout << "Enter id at bt pr:\n";
    for (int i = 0; i < n; i++) cin >> id[i] >> at[i] >> bt[i] >> pr[i];

    int ch;
    cout << "\n1.FCFS 2.SJF 3.Priority 4.RR\nChoice: "; cin >> ch;
    if (ch == 4) { cout << "Quantum: "; cin >> q; }

    if (ch==1) fcfs();
    if (ch==2) sjf();
    if (ch==3) priority();
    if (ch==4) rr();
}
```

## Dry Run

**Input:** 3 processes, all arrive at t=0

| PID | AT | BT | PR |
|-----|----|----|-----|
| P1  | 0  | 4  | 2  |
| P2  | 0  | 3  | 1  |
| P3  | 0  | 2  | 3  |

**FCFS** (run in arrival order P1→P2→P3):
```
Gantt: P1(0-4) P2(4-7) P3(7-9)

P1: TAT=4-0=4,  WT=4-4=0
P2: TAT=7-0=7,  WT=7-3=4
P3: TAT=9-0=9,  WT=9-2=7
Avg WT=3.67  Avg TAT=6.67
```

**SJF** (shortest burst first: P3(bt=2) → P2(bt=3) → P1(bt=4)):
```
Gantt: P3(0-2) P2(2-5) P1(5-9)

P3: TAT=2,  WT=0
P2: TAT=5,  WT=2
P1: TAT=9,  WT=5
Avg WT=2.33  Avg TAT=5.33
```

**Priority** (lowest pr# first: P2(pr=1) → P1(pr=2) → P3(pr=3)):
```
Gantt: P2(0-3) P1(3-7) P3(7-9)

P2: TAT=3,  WT=0
P1: TAT=7,  WT=3
P3: TAT=9,  WT=7
Avg WT=3.33  Avg TAT=6.33
```

**Round Robin** (q=2):
```
Queue seed: [P1]
t=0: P1 runs 2 → rem[P1]=2, enqueue P2,P3 → queue=[P2,P3,P1]
t=2: P2 runs 2 → rem[P2]=1, queue=[P3,P1,P2]
t=4: P3 runs 2 → rem[P3]=0, ft[P3]=6, queue=[P1,P2]
t=6: P1 runs 2 → rem[P1]=0, ft[P1]=8, queue=[P2]
t=8: P2 runs 1 → rem[P2]=0, ft[P2]=9

Gantt: P1(0-2) P2(2-4) P3(4-6) P1(6-8) P2(8-9)

P1: TAT=8, WT=4
P2: TAT=9, WT=6
P3: TAT=6, WT=4
Avg WT=4.67  Avg TAT=7.67
```

---

# 4. PAGE REPLACEMENT (FIFO, LRU, Optimal)

## Concept

RAM has limited frames. When a new page is needed and all frames are full, one page must be evicted. Which one?

- **FIFO** — evict the page that has been in memory the longest (oldest loaded)
- **LRU** — evict the page that was used least recently (furthest back in time)
- **Optimal** — evict the page that will be used furthest in the future (theoretical best)

**Page Fault** = requested page is not in any frame → must load it (and possibly evict).
**Hit** = page already in a frame → do nothing.

**Common logic across all three:**
1. Check if page is in frames (hit → skip)
2. If miss: find empty frame first
3. If no empty frame: apply algorithm's eviction rule to find victim

## Code

```cpp
#include <iostream>
using namespace std;

// pages[]  = the reference string (sequence of page numbers requested)
// frames[] = current contents of RAM frames (-1 = empty)
// n  = length of reference string
// nf = number of frames
int pages[50], frames[10], n, nf;

// inFrame(p) — check if page p is currently in any frame
// Returns the frame index if found, -1 if not (page fault)
int inFrame(int p) {
    for (int i = 0; i < nf; i++)
        if (frames[i] == p) return i;
    return -1;
}

// show(fault) — print current frame contents and whether this was a fault
void show(bool fault) {
    for (int i = 0; i < nf; i++)
        cout << (frames[i] == -1 ? 0 : frames[i]) << " ";
    cout << (fault ? "<-- FAULT" : "") << "\n";
}

// FIFO — First In First Out
// ptr is a circular pointer. Every fault loads new page at frames[ptr],
// then moves ptr forward. So ptr always points to the oldest loaded page.
void fifo() {
    for (int i = 0; i < nf; i++) frames[i] = -1;  // empty all frames
    int ptr = 0, faults = 0;

    cout << "\n=== FIFO ===\n";
    for (int i = 0; i < n; i++) {
        cout << "Ref " << pages[i] << ": ";
        if (inFrame(pages[i]) != -1) { show(false); continue; }  // hit

        frames[ptr] = pages[i];      // load page into oldest slot
        ptr = (ptr + 1) % nf;        // advance pointer circularly
        faults++; show(true);
    }
    cout << "Faults: " << faults << "\n";
}

// LRU — Least Recently Used
// last[i] = the time step when frames[i] was last accessed
// On fault: victim = frame with smallest last[] value (used longest ago)
// On hit: just update last[idx] = current time
void lru() {
    for (int i = 0; i < nf; i++) frames[i] = -1;
    int last[10] = {}, faults = 0;  // last[] starts at 0 (long ago)

    cout << "\n=== LRU ===\n";
    for (int t = 0; t < n; t++) {
        cout << "Ref " << pages[t] << ": ";
        int idx = inFrame(pages[t]);
        if (idx != -1) { last[idx] = t; show(false); continue; }  // hit, update time

        // Find victim: prefer empty frame, else frame with smallest last[] (oldest use)
        int v = 0;
        for (int i = 0; i < nf; i++) {
            if (frames[i] == -1) { v = i; break; }   // empty slot found, use it
            if (last[i] < last[v]) v = i;             // track least recently used
        }
        frames[v] = pages[t]; last[v] = t;  // load page, record access time
        faults++; show(true);
    }
    cout << "Faults: " << faults << "\n";
}

// OPTIMAL — Optimal Page Replacement
// On fault: for each frame, scan forward in pages[] to find next use of that page.
// Evict the frame whose page is used farthest in the future (or never again).
// 'n' as next-use means "never used again" — those get evicted first.
void optimal() {
    for (int i = 0; i < nf; i++) frames[i] = -1;
    int faults = 0;

    cout << "\n=== OPTIMAL ===\n";
    for (int t = 0; t < n; t++) {
        cout << "Ref " << pages[t] << ": ";
        if (inFrame(pages[t]) != -1) { show(false); continue; }  // hit

        // Prefer empty frame first
        int v = -1;
        for (int i = 0; i < nf; i++)
            if (frames[i] == -1) { v = i; break; }

        if (v == -1) {
            // All frames full — find which page is used farthest in future
            int far = -1;
            for (int i = 0; i < nf; i++) {
                int nx = n;  // default: assume this page never appears again (= n, beyond end)
                for (int j = t+1; j < n; j++)
                    if (pages[j] == frames[i]) { nx = j; break; }  // found next use
                if (nx > far) { far = nx; v = i; }  // farther = better victim
            }
        }
        frames[v] = pages[t];
        faults++; show(true);
    }
    cout << "Faults: " << faults << "\n";
}

int main() {
    cout << "Number of pages: "; cin >> n;
    cout << "Reference string: ";
    for (int i = 0; i < n; i++) cin >> pages[i];
    cout << "Frames: "; cin >> nf;

    int ch;
    cout << "\n1.FIFO 2.LRU 3.Optimal 4.All\nChoice: "; cin >> ch;

    if (ch==1||ch==4) fifo();
    if (ch==2||ch==4) lru();
    if (ch==3||ch==4) optimal();
}
```

## Dry Run

**Input:** Reference string: 1 2 3 4 1 2 5 1 2 3 4 5, Frames = 3

**FIFO** (ptr cycles 0→1→2→0→...):

| Ref | Frame 0 | Frame 1 | Frame 2 | Fault? |
|-----|---------|---------|---------|--------|
| 1   | 1       | -       | -       | FAULT (ptr→1) |
| 2   | 1       | 2       | -       | FAULT (ptr→2) |
| 3   | 1       | 2       | 3       | FAULT (ptr→0) |
| 4   | 4       | 2       | 3       | FAULT evict 1 (ptr→1) |
| 1   | 4       | 1       | 3       | FAULT evict 2 (ptr→2) |
| 2   | 4       | 1       | 2       | FAULT evict 3 (ptr→0) |
| 5   | 5       | 1       | 2       | FAULT evict 4 (ptr→1) |
| 1   | 5       | 1       | 2       | HIT |
| 2   | 5       | 1       | 2       | HIT |
| 3   | 5       | 3       | 2       | FAULT evict 1 (ptr→2) |
| 4   | 5       | 3       | 4       | FAULT evict 2 (ptr→0) |
| 5   | 5       | 3       | 4       | HIT |

**Total FIFO Faults: 9**

**LRU** (evict page unused for longest):

| Ref | Frames    | last[]   | Fault? |
|-----|-----------|----------|--------|
| 1   | [1,-,-]   | [0,-,-]  | FAULT |
| 2   | [1,2,-]   | [0,1,-]  | FAULT |
| 3   | [1,2,3]   | [0,1,2]  | FAULT |
| 4   | [4,2,3]   | [3,1,2]  | FAULT — evict 1 (last used at t=0, oldest) |
| 1   | [4,1,3]   | [3,4,2]  | FAULT — evict 2 (last used at t=1) |
| 2   | [4,1,2]   | [3,4,5]  | FAULT — evict 3 (last used at t=2) |
| 5   | [5,1,2]   | [6,4,5]  | FAULT — evict 4 (last used at t=3) |
| 1   | [5,1,2]   | [6,7,5]  | HIT — update last[1]=7 |
| 2   | [5,1,2]   | [6,7,8]  | HIT |
| 3   | [5,3,2]   | [6,9,8]  | FAULT — evict 5 (last used at t=6) |
| 4   | [4,3,2]   | [10,9,8] | FAULT — evict 5→4 |
| 5   | [4,3,5]   | [10,9,11]| FAULT — evict 2 (last at t=8) |

**Total LRU Faults: 10**

**Optimal** (evict page used farthest in future):

| Ref | Frames  | Next use of each frame's page | Evict | Fault? |
|-----|---------|-------------------------------|-------|--------|
| 1   | [1,-,-] | —                             | —     | FAULT |
| 2   | [1,2,-] | —                             | —     | FAULT |
| 3   | [1,2,3] | —                             | —     | FAULT |
| 4   | [1,2,4] | 1→t=4, 2→t=5, 3→never        | 3     | FAULT |
| 1   | [1,2,4] | —                             | —     | HIT |
| 2   | [1,2,4] | —                             | —     | HIT |
| 5   | [1,2,5] | 1→t=7, 2→t=8, 4→never        | 4     | FAULT |
| 1   | [1,2,5] | —                             | —     | HIT |
| 2   | [1,2,5] | —                             | —     | HIT |
| 3   | [3,2,5] | 1→never, 2→never, 5→t=11     | 1     | FAULT |
| 4   | [3,4,5] | 3→never, 2→never, 5→t=11     | 2(3)  | FAULT |
| 5   | [3,4,5] | —                             | —     | HIT |

**Total Optimal Faults: 7**

**Summary for same input:**
```
FIFO    = 9 faults
LRU     = 10 faults  (FIFO anomaly — more frames can cause more faults)
Optimal = 7 faults   (theoretical minimum, always best or equal)
```

---

## Quick Formula Sheet

```
TAT = Finish Time - Arrival Time
WT  = TAT - Burst Time
need[i][j] = maxNeed[i][j] - alloc[i][j]    (Banker's)

FIFO:    victim = frames[ptr],  ptr = (ptr+1) % nf
LRU:     victim = frame with min(last[])
Optimal: victim = frame with max(next use position)

Producer: wait(emp) → lock → write → unlock → signal(full)
Consumer: wait(full) → lock → read  → unlock → signal(emp)
```
