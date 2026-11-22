# Choosing Sorting and Shortest Path Algorithms for a Food Delivery Platform

**Course:** CS-251 Design and Analysis of Algorithms, BSDS Semester 7, SEECS NUST
**Assignment:** 02
**Group members:** Affan Ahmad Basra (476173), Abdur Rafey (481971)
**Repository:** https://github.com/AffanBasra/foodroute-algo-eval

---

## 1. Scenario

We look at a food delivery platform running in Lahore (similar to foodpanda). Two parts of
its backend need algorithms that work well at scale.

**Task A: ordering pending orders (sorting).**
During the dinner rush, the dispatch system holds thousands of pending orders. Every few
seconds it sorts them by **promised delivery time** so the most urgent orders are handed to
riders first. Each record is `(promised_delivery_time, order_id)`, where the time is an
integer number of seconds since midnight (0 to 86,399).

Real orders have some useful properties:
- Keys are small whole numbers with a fixed range.
- Many orders share the same key, because the app promises delivery in 15 minute slots.
- Orders often arrive already close to time order.
- When two orders have the same promised time, the one placed first should be served first.
  This needs a **stable** sort.

**Task B: routing riders (single source shortest path).**
A rider at a junction needs the cheapest route to every other junction (to pick up and drop
off). We model the city as a directed graph. Junctions are nodes, roads are edges, and the
weight is the travel time in seconds. Each direction of a road gets its own time, since
traffic differs each way.

Some platforms also pay **rider bonuses** for passing pickup points in busy areas. If the
cost of an edge is "time minus bonus", some edges become **negative**. A negative cycle would
mean a rider could earn money forever by driving in a loop, so the platform must never allow
one.

### 1.1 Datasets

All data is generated with fixed random seeds so the results can be reproduced
(`src/generators.py`).

**Orders**, 5 input structures, sizes n = 1,000 / 5,000 / 10,000 / 50,000 / 100,000:

| Input | How it is made | Real meaning |
|---|---|---|
| random | uniform times over the day | orders from many restaurants mixed together |
| sorted | random times, sorted ascending | orders already queued in time order |
| reverse | sorted descending | worst case for some algorithms |
| duplicates | only 96 distinct keys (15 min slots) | slot based delivery promises |
| nearly sorted | sorted, then 1% of positions swapped | mostly in order, a few late orders |

**Road graphs**, 2 structures:

| Graph | Structure | Sizes | Real meaning |
|---|---|---|---|
| sparse | square grid, each junction links to up to 4 neighbours, E about 4V | V = 400 to 10,000 | city road blocks (like Johar Town) |
| dense | each pair of hubs connected with probability 0.5, E about V²/2 | V = 100 to 500 | network between restaurant hubs and dark kitchens |

Each graph is tested twice: once with normal weights (travel time, all positive) and once
with bonuses (negative edges). Travel times are 30 to 600 s per block on the grid and 60 to
3,600 s between hubs.

To add bonuses without making a negative cycle, each node `u` gets a random value `h[u]`
and every edge weight becomes `w(u,v) + h[u] - h[v]`. Around any cycle the `h` values cancel
out, so every cycle keeps its original positive cost. Single edges can still go negative:
about 15 to 16% of edges ended up negative in our graphs. This also gives us the exact correct
answer for checking: `d'(s,v) = d(s,v) + h[s] - h[v]`.

### 1.2 Evaluation criteria and weights

**Sorting:**

| Criterion | Weight | Why |
|---|---|---|
| Measured runtime | 35% | Orders are re-sorted every few seconds at peak, so speed matters most. |
| Fit for the data | 30% | How well the algorithm uses the data's properties (integer keys, small range, duplicates, near-sorted input) and how safe it is from bad cases. |
| Stability | 20% | Orders with the same promised time must stay in the order they were placed (first come, first served). Stability also allows multi-key sorting. |
| Memory use | 15% | The gap between algorithms is only a few MB for 100,000 orders, which is very small for a server. |

Memory gets the lowest weight because a dispatch server has gigabytes of RAM. The worst
algorithm here uses about 3 MB for 100,000 orders. If the sort ran on an edge device, such
as a rider's low-end phone or a scanner at the restaurant counter, memory would matter much
more and we would raise its weight.

**Shortest path:**

| Criterion | Weight | Why |
|---|---|---|
| Measured runtime | 40% | Routes are recomputed often as riders move and orders come in. |
| Correctness for our weights | 35% | A wrong route sends a rider the wrong way. |
| Memory use | 15% | Same reasoning as above, but road graphs can get large. |
| Extra features | 10% | For example, detecting negative cycles in the bonus setup. |

Each algorithm gets a score from 1 (poor) to 5 (best) on each criterion, and the weighted sum
gives the final ranking (Section 6).

---

## 2. Theoretical Analysis

### 2.1 Algorithms compared

**Sorting:** Insertion sort, Merge sort, Quick sort (random pivot, Hoare partition), Heap sort,
and LSD Radix sort (base 256, one stable counting/bucket pass per byte).

**Shortest path:** Dijkstra (binary heap, lazy deletion) and Bellman-Ford (with early exit
when a full pass changes nothing).

### 2.2 Complexity tables

n = number of orders, V = nodes, E = edges, d = number of digits, b = base.

| Algorithm | Best | Average | Worst | Extra space | Stable |
|---|---|---|---|---|---|
| Insertion | O(n) | O(n²) | O(n²) | O(1) | Yes |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | Yes |
| Quick | O(n log n) | O(n log n) | O(n²) | O(log n) stack | No |
| Heap | O(n log n)* | O(n log n) | O(n log n) | O(1) | No |
| Radix (LSD) | O(d(n + b)) | O(d(n + b)) | O(d(n + b)) | O(n + b) | Yes |

\* Heap sort is O(n) only in the special case where all keys are equal.

For our keys, the maximum is 86,399 < 2^17, so with base 256 Radix sort needs **d = 3**
passes. Since d and b are fixed, Radix sort is **O(n)** for this data.

Notes:
- **Insertion** best case is already-sorted input: each element is compared once and never moved.
- **Quick sort** worst case happens when the pivot is always the smallest or largest element.
  With a random pivot this is very unlikely on any input, including sorted and reverse.
  Recursing on the smaller side first keeps the stack at O(log n).
- **Hoare partition** splits runs of equal keys evenly, so many duplicates do not push Quick
  sort toward O(n²).

| Algorithm | Best | Average | Worst | Extra space | Negative weights |
|---|---|---|---|---|---|
| Dijkstra (binary heap) | O((V + E) log V) | O((V + E) log V) | O((V + E) log V) | O(V + E) | No |
| Bellman-Ford (early exit) | O(E) | O(k · E) | O(V · E) | O(V) | Yes, and detects negative cycles |

For Bellman-Ford, k is the number of passes needed. It is at most V - 1, and in practice it
equals one more than the largest number of edges on any shortest path. Our code also stores a
flat edge list, which adds O(E) memory (see Section 5).

Dijkstra's heap can hold up to E entries with lazy deletion, which is why the space is
O(V + E). On a sparse grid E is about 4V, so this is O(V log V). On a dense graph E is about
V², so it becomes O(V² log V).

### 2.3 Recurrence 1: Merge sort with the Master method

Merge sort splits the array into two halves, sorts each half, then merges them in linear time:

T(n) = 2T(n/2) + Θ(n)

Master method form: T(n) = aT(n/b) + f(n), with a = 2, b = 2, f(n) = Θ(n).

- Compute n^(log_b a) = n^(log_2 2) = n^1 = n.
- Compare f(n) with n: f(n) = Θ(n) = Θ(n^(log_b a)).
- This is **Case 2**, so T(n) = Θ(n^(log_b a) · log n) = **Θ(n log n)**.

### 2.4 Recurrence 2: Merge sort with a recursion tree

To check the same result another way, assume the merge costs cn:

| Level | Subproblems | Size of each | Work per level |
|---|---|---|---|
| 0 | 1 | n | cn |
| 1 | 2 | n/2 | 2 · c(n/2) = cn |
| 2 | 4 | n/4 | 4 · c(n/4) = cn |
| i | 2^i | n/2^i | cn |
| log₂ n | n | 1 | cn |

There are log₂ n + 1 levels and each costs cn, so the total is cn(log₂ n + 1) = **Θ(n log n)**.
This matches the Master method.

### 2.5 Recurrence 3: Quick sort worst case

If the pivot is always the smallest element, one side is empty and the other has n - 1 elements:

T(n) = T(n - 1) + cn

Unrolling: T(n) = cn + c(n - 1) + c(n - 2) + ... + c = c · n(n + 1)/2 = **Θ(n²)**.

With a random pivot, the expected split is balanced enough to give T(n) = Θ(n log n) on
average. This is why we use random pivots.

---

## 3. Experiment Design

**Machine:** Intel Core i7-11800H (8 cores, 16 threads, 2.3 GHz base), 32 GB RAM,
Windows 11 (build 26200), Python 3.14.2, numpy 2.5.4, matplotlib 3.11.2.

**Sorting** (`src/bench_sort.py`):
- Sizes: 1,000, 5,000, 10,000, 50,000, 100,000.
- 5 input structures (Section 1.1), each generated with seed 42.
- One warm-up run first (not counted), then 7 timed runs for n ≤ 10,000 and 5 for larger n.
  We report the mean, the median and the standard deviation.
- Insertion sort only goes up to 10,000 and uses 3 runs, because it is O(n²). At 100,000 it
  would take around 3 minutes per run on random input (projected from the 10,000 result).
- Time is measured with `time.perf_counter()`.
- Memory is measured in a **separate** run with `tracemalloc`, recording peak memory allocated
  during the sort. It is a separate run because `tracemalloc` slows the code down.
- Every algorithm first copies the input list (`list(arr)`) so all algorithms start from the
  same data. This copy costs 8 bytes per element (about 781 KB at n = 100,000) and is included
  in every memory figure.

**Shortest path** (`src/bench_graph.py`):
- Sparse grids with side 20, 40, 60, 80, 100 (V = 400 to 10,000).
- Dense graphs with V = 100, 200, 300, 400, 500.
- Each graph is run with positive weights and with bonus (negative) weights.
- 5 random source nodes per graph, and we report the mean time.
- Correctness: on positive graphs we compare both algorithms' outputs with each other. On
  negative graphs we compare against the exact formula from Section 1.1 and record the
  percentage of nodes where each algorithm returned the wrong distance.
- We also record how many passes Bellman-Ford needed before it stopped early.

**Correctness tests** (`src/tests.py`) check every algorithm on all input types and edge
cases (empty list, single element), check stability, and check that Bellman-Ford catches a
negative cycle. All tests passed before benchmarking.

---

## 4. Results

### 4.1 Sorting runtime

![Sorting runtime](../figures/sort_runtime.png)

Mean time in ms, **n = 10,000**:

| Algorithm | random | sorted | reverse | duplicates | nearly sorted |
|---|---|---|---|---|---|
| Insertion | 1948.9 | 1.1 | 3880.4 | 1963.1 | 51.4 |
| Merge | 21.0 | 15.8 | 13.8 | 16.7 | 16.1 |
| Quick | 15.1 | 14.1 | 14.3 | 13.7 | 13.9 |
| Heap | 22.0 | 22.8 | 18.0 | 20.2 | 19.5 |
| Radix | 4.2 | 2.6 | 2.7 | 3.1 | 2.5 |

Mean time in ms, **n = 100,000**:

| Algorithm | random | sorted | reverse | duplicates | nearly sorted |
|---|---|---|---|---|---|
| Merge | 318.8 | 284.7 | 234.8 | 315.4 | 268.6 |
| Quick | 293.0 | 290.6 | 229.2 | 297.0 | 216.6 |
| Heap | 369.7 | 362.1 | 341.2 | 418.1 | 361.5 |
| Radix | 72.9 | 43.3 | 55.8 | 58.2 | 54.8 |

### 4.2 Sorting memory

![Sorting memory](../figures/sort_memory.png)

Peak memory in KB, random input (includes the 8 bytes per element input copy):

| Algorithm | 1,000 | 5,000 | 10,000 | 50,000 | 100,000 |
|---|---|---|---|---|---|
| Insertion | 8 | 39 | 78 | - | - |
| Merge | 31 | 156 | 313 | 1,563 | 3,125 |
| Quick | 11 | 42 | 81 | 393 | 784 |
| Heap | 8 | 39 | 78 | 391 | 781 |
| Radix | 51 | 122 | 205 | 884 | 1,714 |

### 4.3 Stability check

From `tests.py`, on 2,000 orders with duplicate keys:

| Algorithm | Stable? |
|---|---|
| Insertion | Yes |
| Merge | Yes |
| Quick | No |
| Heap | No |
| Radix | Yes |

### 4.4 Shortest path runtime

![Shortest path runtime](../figures/graph_runtime.png)

| Graph | V | E | Dijkstra (ms) | Bellman-Ford (ms) | BF / Dijkstra | BF passes (max V - 1) |
|---|---|---|---|---|---|---|
| sparse | 400 | 1,520 | 0.35 | 2.61 | 7.5x | 17.2 (399) |
| sparse | 1,600 | 6,240 | 2.52 | 21.57 | 8.6x | 35 (1,599) |
| sparse | 3,600 | 14,160 | 3.05 | 63.57 | 20.8x | 48 (3,599) |
| sparse | 6,400 | 25,280 | 5.30 | 175.66 | 33.1x | 76.6 (6,399) |
| sparse | 10,000 | 39,600 | 8.45 | 351.72 | 41.6x | 103.4 (9,999) |
| dense | 100 | 5,069 | 0.34 | 2.36 | 7.0x | 4.8 (99) |
| dense | 200 | 20,012 | 1.10 | 11.11 | 10.1x | 4.8 (199) |
| dense | 300 | 45,095 | 2.45 | 25.63 | 10.5x | 4.4 (299) |
| dense | 400 | 79,940 | 3.99 | 47.24 | 11.8x | 5.2 (399) |
| dense | 500 | 124,845 | 6.70 | 81.45 | 12.2x | 5 (499) |

(Positive weights. Times with negative weights were almost the same; see
`results/summary_tables.md`.)

### 4.5 Correctness with negative weights

![Dijkstra errors](../figures/dijkstra_negative_errors.png)

With bonus edges, Dijkstra returned the wrong distance for **37% to 82%** of nodes.
Bellman-Ford was correct for **100%** of nodes in every case. With positive weights both
algorithms were always correct and matched each other.

### 4.6 Shortest path memory

| Graph | V | Dijkstra (KB) | Bellman-Ford (KB) |
|---|---|---|---|
| sparse | 10,000 | 466 | 3,679 |
| dense | 500 | 80 | 9,651 |

---

## 5. Theory vs Measurement

### 5.1 Where the theory matched

**Growth rates.** We fit a line to log(time) vs log(n). A slope of 1 means linear growth, 2
means quadratic, and n log n gives a slope a bit above 1.

| Algorithm | random | sorted | reverse | duplicates | nearly sorted |
|---|---|---|---|---|---|
| Insertion | 2.02 | 1.14 | 2.03 | 2.06 | 1.97 |
| Merge | 1.15 | 1.20 | 1.17 | 1.25 | 1.21 |
| Quick | 1.17 | 1.19 | 1.14 | 1.20 | 1.15 |
| Heap | 1.22 | 1.17 | 1.23 | 1.27 | 1.16 |
| Radix | 1.17 | 1.08 | 1.17 | 1.14 | 1.10 |

- **Insertion sort** has a slope of about 2.0 on random, reverse and duplicate input, which
  is exactly O(n²). On sorted input the slope drops to 1.14 and 10,000 orders take only
  1.1 ms, which is the O(n) best case. Reverse input took about 2x longer than random
  (3880 vs 1949 ms) because every element must shift past all earlier ones, while on random
  input it only shifts past about half of them on average.
- **Merge, Quick and Heap** all have slopes between 1.14 and 1.27, as expected for n log n.
- **Quick sort** stayed fast on sorted, reverse and duplicate input, so the random pivot and
  Hoare partition did their job. There is no sign of O(n²).
- **Stability** matched the theory exactly (Section 4.3).
- **Memory** matched the space complexity. Heap sort's memory is exactly the input copy
  (800,056 bytes at n = 100,000), so it uses O(1) extra. Quick sort adds only about 2.6 KB of
  recursion stack, matching O(log n). Merge sort uses about 4x the input copy because of its
  temporary array and the slice copies Python makes during merging, which is O(n). Radix sort
  sits in between, with O(n + b) for the buckets.
- **Bellman-Ford vs Dijkstra.** On the sparse grid the gap grows with size (7.5x to 41.6x),
  as expected when comparing roughly k · E against E log V with k growing.

### 5.2 Gaps and surprises

**1. Heap sort slows down per element as n grows.**
If time is c · n log n, then time / (n log₂ n) should stay flat. It did not for Heap sort:

| Algorithm | 1,000 | 5,000 | 10,000 | 50,000 | 100,000 |
|---|---|---|---|---|---|
| Merge (ns) | 149.6 | 135.7 | 158.3 | 141.6 | 192.0 |
| Quick (ns) | 112.6 | 125.7 | 113.7 | 109.0 | 176.4 |
| Heap (ns) | 127.3 | 152.7 | 165.6 | 175.4 | 222.6 |

Heap sort's constant grew by about 75%. The reason is **cache misses**: sift-down jumps from
index i to 2i + 1, so for large heaps each step lands far away in memory. Merge and Quick sort
scan memory in order, which the CPU cache handles well. Big-O ignores memory layout, but real
hardware does not. Merge and Quick also rise at 100,000 for the same reason, but less.

**2. Radix sort is O(n) but its slope was 1.08 to 1.17, not 1.0.**
At larger n, the bucket lists no longer fit in the fastest cache, and Python has to grow
the bucket lists more often. The work is still linear, but each element gets a bit more
expensive. Even so, Radix sort was 4x to 7x faster than the next best algorithm at 100,000.

**3. Merge and Quick sort ran faster on sorted and reverse input than on random input.**
Theory says the same O(n log n) for all inputs. On sorted input, Merge sort finishes each
merge early, because one half runs out before the other, so it does fewer comparisons.
More generally, the CPU predicts branches better when the data has a pattern. Random data
makes the `if a[i] <= a[j]` branch hard to predict.

**4. Bellman-Ford needed far fewer passes than V - 1.**
The worst case is V - 1 passes, but with early exit it stopped after only:
- about 5 passes on dense graphs (vs up to 499), because with so many direct routes, most
  shortest paths use only a few edges;
- 17 to 103 passes on the grid (vs up to 9,999). This is about the grid side length, because
  the longest shortest path has to cross the grid.

So the real cost was k · E with small k, not V · E. This is why Bellman-Ford was "only"
7 to 42x slower and not hundreds of times slower. The worst case is still possible, for
example on a long chain graph whose edges are listed in reverse order.

**5. Bellman-Ford used more memory than Dijkstra.**
In theory Bellman-Ford needs only O(V) extra space. Our version first builds a flat list of
all edges as (u, v, w) tuples, which is O(E). That made it use 3.7 MB vs 0.5 MB for Dijkstra
on the 10,000 node grid. This is an implementation choice, not part of the algorithm: looping
over the adjacency list directly would avoid it, but would make the inner loop slower in
Python.

**6. Dijkstra on negative weights did not crash, it was just wrong.**
Dijkstra assumes that once a node is taken from the heap, its distance is final. A negative
edge found later can break this, and the algorithm never goes back to fix it. The dangerous
part is that it fails **silently**. It returned a full set of distances and ran just as fast,
but 37 to 82% of them were wrong. The error rate was higher on the sparse grid, because grid
paths are long and one early mistake spreads to every node after it.

**7. Measurement noise.**
Some single runs were slowed by background tasks (for example, Merge sort on duplicates at
n = 50,000 took 193 ms vs 111 ms on random, while at 100,000 the two were close). We reduce
this with warm-up runs, repeated runs and reporting both the mean and median, but small
swings remain. The overall trends were the same in an earlier test run on a different
machine.

---

## 6. Ranking with Weighted Criteria

### 6.1 Sorting

Scores from 1 to 5. Runtime is based on the 100,000 order results; Insertion sort is scored
from its O(n²) growth.

| Algorithm | Runtime (35%) | Fit (30%) | Stability (20%) | Memory (15%) | **Weighted score** |
|---|---|---|---|---|---|
| **Radix** | 5 | 5 | 5 | 3 | **4.70** |
| Merge | 3 | 4 | 5 | 2 | 3.55 |
| Quick | 3 | 3 | 1 | 4 | 2.75 |
| Insertion | 1 | 2 | 5 | 5 | 2.70 |
| Heap | 2 | 3 | 1 | 5 | 2.55 |

How the scores were given:
- **Runtime:** Radix 43 to 73 ms; Quick and Merge 217 to 319 ms; Heap 341 to 418 ms;
  Insertion is far behind (minutes at 100,000).
- **Fit:** Radix uses the fact that our keys are small integers (3 passes). Merge is
  general and always O(n log n). Quick has an O(n²) worst case (unlikely but possible). Heap
  is general but slow in practice. Insertion only fits small or nearly sorted batches.
- **Stability:** stable = 5, not stable = 1.
- **Memory:** O(1) = 5, O(log n) = 4, O(n + b) = 3, O(n) with the highest measured use = 2.

### 6.2 Shortest path

| Algorithm | Runtime (40%) | Correctness (35%) | Memory (15%) | Features (10%) | **Weighted score** |
|---|---|---|---|---|---|
| **Dijkstra** | 5 | 3 | 5 | 2 | **4.00** |
| Bellman-Ford | 1 | 5 | 3 | 5 | 3.10 |

Dijkstra gets 3 for correctness, not 1: it is always correct for real travel times, which can
never be negative, but it fails if bonuses are mixed into edge weights.

---

## 7. Recommendation and Defense

### 7.1 Sorting: LSD Radix sort

**Choice:** Radix sort (LSD, base 256) for sorting orders by promised delivery time.

**Why:** It was the fastest on every input type and size, 4x to 7x faster than Quick and
Merge sort at 100,000 orders. It is stable, so orders in the same slot stay first come, first
served. Its running time does not depend on the input order, so there is no bad case to worry
about at peak hours.

**Weaknesses:**
1. It only works on non-negative integer keys of limited size. If the platform switched to
   sorting by a decimal "priority score", Radix sort would not work directly.
2. It needs O(n + b) extra memory (1.7 MB at 100,000), about twice as much as Quick or Heap
   sort. On a low-memory edge device this would matter.

**Defense against the strongest alternative, Merge sort:**
Merge sort is also stable and always O(n log n), and it works on any key type, which makes it
the safest general choice. But for this data, Radix sort was 3.9x to 6.6x faster at 100,000
orders and used about half the memory (1.7 MB vs 3.1 MB). Our keys will always be whole
seconds within one day, so Merge sort's flexibility gives us nothing here. If the key type
ever changes, Merge sort is the fallback.

### 7.2 Shortest path: Dijkstra

**Choice:** Dijkstra with a binary heap for rider routing on travel times.

**Why:** It was 7x to 42x faster than Bellman-Ford, and the gap grows on the large sparse
graphs that real road networks look like. It also used much less memory. Real travel times
are never negative, so Dijkstra is always correct for the main routing job.

**Weakness:** It gives wrong answers when there are negative edges, and it fails silently,
which we measured at 37 to 82% wrong distances. It also cannot detect negative cycles.

**Defense against the strongest alternative, Bellman-Ford:**
Bellman-Ford is the only one of the two that is correct with bonuses, and it can catch a
negative cycle. But it was up to 42x slower, and its worst case is O(VE), which would be too
slow for a city-wide map recalculated many times per minute. We do not need Bellman-Ford
for the main job:
- Routing should use **pure travel time**, which is never negative, so Dijkstra is correct.
- If bonuses must be part of the cost and they come from per-location values (as in our
  setup), the same trick from Section 1.1 can be used in reverse: run Dijkstra on the
  positive travel times, then adjust each answer with `+ h[s] - h[v]`. This is the idea behind
  **Johnson's algorithm**, which runs Bellman-Ford once to find such values and then uses
  Dijkstra for all the fast queries.
- Bellman-Ford is still useful as an occasional **check** whenever bonus rules change, to
  make sure no negative cycle (a money loop for riders) has been created.

---

## 8. Real-World Note

Large delivery platforms use the same ideas at a bigger scale: faster versions of Dijkstra
(such as A* and Contraction Hierarchies, which pre-process the map so each route takes
milliseconds), spatial indexes (geohash, H3, quadtrees) to find nearby riders, priority queues
for order queues, and optimisation methods for assigning orders to riders (a form of the
vehicle routing problem). Our results show why the basics matter: a small change in the
data (negative edges) or the hardware (cache behaviour) can change which algorithm is best.

---

## 9. Reproducing the Results

```
pip install -r requirements.txt
python src/tests.py
python src/bench_graph.py
python src/bench_sort.py
python src/plots.py
```

Raw results are in `results/sort_results.csv` and `results/graph_results.csv`. All tables in
this report come from `results/summary_tables.md`.

---

## 10. Individual Contribution Statement

| Member | Contribution |
|---|---|
| Affan Ahmad Basra | Implemented the sorting and shortest path algorithms, data generators, tests and benchmark scripts. Ran all experiments and produced the charts. |
| Abdur Rafey | Theoretical analysis (complexity tables and recurrences), evaluation criteria and weights, analysis of results, and writing the report. |

Both members reviewed the final code and report.

---

## 11. Declaration of AI Usage

[Edit this so it matches exactly what you did. Example:]

We used an AI assistant to help structure the code, suggest the experiment
design, and explain concepts such as the potential-based method for creating negative
weights. All code was reviewed and run by us, all benchmark results were produced on our
own machine, and we checked the analysis and conclusions against our measured data.
