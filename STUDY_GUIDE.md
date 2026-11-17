# Study guide: A* search notebook (AM6020)

This guide goes with `astar_pathfinding.ipynb`. It explains each section in simple English, points to the lines of code you
must be able to explain, and gives the main result with the real numbers from the run (seed 42). Run times in ms change a
little from run to run; everything else (paths, costs, node counts) is the same every time.

---

## Section 0: Title and overview

**Summary.** The first cell says what the notebook is for, lists the sections, explains how to run it, and says where the
outputs go. It also lists the only references we cite. Read it once before the presentation; it is your table of contents.

**Key point.** The search code is ours. NetworkX only *stores* graphs, and is used as a reference answer in Section 3.

---

## Section 1: Setup

**Summary.** Imports the libraries, fixes the random seed, creates the folders `figures/`, `tables/` and `data/`, sets one
plot style for every figure, and defines the two saving helpers. It also saves a table of software versions. Everything later
depends on this section.

**Key lines.**
- `SEED = 42` and `rng = np.random.default_rng(SEED)`: the *only* source of randomness. Every random map and every random
  start–goal pair comes from `rng`, so a Restart & Run All always gives the same results. (Re-running one cell on its own
  moves `rng` forward, so always use Restart & Run All for the final numbers.)
- `save_fig(fig, name)`: writes `figures/name.pdf` (vector PDF, no title; the caption goes in LaTeX).
- `save_table(df, name)`: writes `tables/name.tex` with `df.to_latex(...)`; pandas already uses `booktabs` rules
  (`\toprule`, `\midrule`, `\bottomrule`). We replace `±` with `$\pm$` so LaTeX prints it correctly.
- `"pdf.fonttype": 42`: embeds real fonts in the PDF so text stays sharp and selectable on Overleaf.

**Main result.** Python 3.11.0, NumPy 2.4.6, pandas 2.2.3, NetworkX 3.6.1, Matplotlib 3.10.1, OSMnx 2.1.1.

---

## Section 2: Implementation

**Summary.** Builds everything the experiments need: the `SearchResult` record, the grid builder, the heuristics, the main
function `best_first_search`, the named wrappers and BFS. One function does Dijkstra, A*, Weighted A* and greedy search;
only the priority `f` changes. A quick test on an empty 20×20 grid checks the answer against one we can work out by hand.
(The grid builder lives here, not in Section 5, because the quick test and Section 3 already need grids.)

**Key lines.**
- `f = g + w*h` (in `priority`): Dijkstra is `h = 0`; A* is `w = 1`; Weighted A* is `w > 1`; greedy uses `f = h` only.
- Heap entries `(f, tie, counter, g, node)`: Python compares tuples left to right, so the smallest `f` comes out first; `tie`
  breaks equal `f`; `counter` is unique, so two nodes are never compared directly.
- `if g_value > best_g[node]: continue`: skip a **stale** entry.
- `if node == goal:` comes **after** the pop: goal test on pop.
- `if g_new < best_g.get(neighbour, math.inf):`: only update when strictly cheaper.
- `octile`: `max(dr, dc) + (√2 − 1)·min(dr, dc)`, the exact cost on an empty 8-connected grid (Amit Patel, Red Blob Games).
- `make_haversine(G, max_speed_kph)`: great-circle distance in metres; divided by top speed (in m/s) it becomes a time in seconds.
- In `add_grid_edges`, the diagonal is added only if both side cells are free (no corner cutting, as in Sturtevant's benchmarks).

**Main result.** From (0, 0) to (19, 19) on an empty 8-connected grid, every method finds the optimal cost
19√2 = 26.87. Dijkstra expands 399 nodes; A* with octile expands only 19, exactly the cells on the path.

---

## Section 3: Correctness check

**Summary.** Compares our Dijkstra and A* with NetworkX's `dijkstra_path_length` and `astar_path_length` on four 30×30 maps
(4- and 8-connected, empty and with 30% walls), 150 pairs with a path on each. It compares **costs**, not paths, because two
different paths can have the same cost. It also checks that `path_cost(path)` equals the cost we report, and that both
libraries agree when there is no path.

**Key lines.**
- `assert abs(ours_astar.cost - ref_astar) <= TOLERANCE`: the notebook stops if any answer differs by more than 1e-9.
- `reference_cost(...)` turns NetworkX's "no path" exception into `math.inf`, which is what our code returns.

**Main result.** **600/600 pairs match NetworkX** for both Dijkstra and A*. 12 and 7 pairs had no path (on the two maps with
walls) and both libraries agreed on that.

---

## Section 4: Worked example

**Summary.** A 6-node graph (S, A, B, C, D, G) with coordinates. The heuristic is straight-line distance to G. Every edge weight
is at least the straight-line distance, so the heuristic is consistent, and the notebook checks this. `astar_traced` follows the
same rules as `best_first_search` but prints one row per pop.

**Key lines.**
- The consistency check `worked_h(a, "G") > data["weight"] + worked_h(b, "G")` is the definition `h(u) ≤ c(u,v) + h(v)`.
- `describe_open_list` marks stale entries, so you can point at the stale D in the table.

**Main result.** Path S → B → D → G with cost 8. A* expanded 4 of 6 nodes (S, A, B, D); C was never expanded because
f(C) = 8.10 > 8. D was first reached through A (g = 6.6, f = 9.10), then through B more cheaply (g = 5, f = 7.5), which left a
stale D entry in the heap.

---

## Section 5: Test environments

**Summary.** Builds nine grids (random walls 25%, 4- and 8-connected on the same walls; and mazes, 4-connected) in three sizes,
plus the Cork road network from OpenStreetMap via OSMnx. The Cork graph is converted from a `MultiDiGraph` to a simple
`DiGraph`, keeping the shortest of any parallel edges. The road heuristics are checked for consistency on every road segment.

**Key lines.**
- Why convert: in a `MultiDiGraph`, `G[u][v]` is `{0: {...}, 1: {...}}` (keyed by edge number), so `G[u][v]["length"]` fails.
  `to_simple_digraph` keeps the minimum length, because a shortest-path search would always choose it anyway.
- `CORK_MAX_SPEED_KPH = max(...)` reads the top speed from the graph's own edges (120 km/h).
- `blocked_maze`: a stack-based random depth-first search; "knock down the wall between two rooms, move on; at a dead end, step
  back". Then 10% of inner walls are removed to add loops.

**Main result.** Grids from 75 to 7,495 free cells; Cork has 10,161 junctions and 21,878 one-way segments (394 parallel edges
dropped). Both road heuristics have **0 consistency violations** out of 21,878 edges.

---

## Section 6: Demonstrations

**Summary.** On all 10 environments, runs BFS, Dijkstra, greedy and A* on random pairs (50 per grid, 30 on Cork) and records
cost ratio, nodes expanded, peak open-list size and run time (median of 3). It checks that A* is always optimal. It draws the
explored cells in a maze, and the explored junctions for a long Cork route.

**Key lines.**
- `timed_run`: runs 3 times, returns the median time (less sensitive to random slow-downs).
- `"cost ratio": result.cost / optimal_cost`, where the optimum is Dijkstra's cost.
- `assert astar_optimal`: the notebook would stop if A* ever returned a costlier path.

**Main result.** A* was optimal on **480/480** pairs and never expanded more nodes than Dijkstra. On the 100×100 random
4-connected grid A* expanded 374 nodes against Dijkstra's 3,144 (about 12%). In the 50×50 maze (start (1,1), goal (47,47)):
Dijkstra 951 expanded, A* 663, greedy 487 but cost 200 instead of 176. Long Cork route (15.25 km straight line, 17.10 km by
road): Dijkstra 9,822 junctions, A* 2,999.

Two things you might be asked about:
- BFS and Dijkstra expand exactly the same nodes on 4-connected grids. With every edge costing 1 and ties first-in-first-out,
  Dijkstra pops nodes in the same order as BFS.
- A*'s peak open list is sometimes *larger* than Dijkstra's. Section 8 explains why (stale entries).

---

## Section 7: Parameter study (most important for marks)

**Summary.** Changes one thing at a time: (a) the heuristic, (b) the weight `w`, (c) 4 vs 8 moves, (d) tie-breaking, (e) road
length vs travel time. Each part saves a table or figure.

**Key lines.**
- 7a: `ADMISSIBLE_ON_8` marks Manhattan as *not* admissible on 8-connected grids.
- 7b: `assert` that `cost ratio <= w` for every run (the Weighted A* bound).
- 7c: the same `walls` array builds both grids, so only the move set changes.
- 7d: `astar(..., tie_break=rule)` with `rule` in `high_g`, `low_g`, `fifo`.
- 7e: `cork_h_time = make_haversine(cork, max_speed_kph=CORK_MAX_SPEED_KPH)` is admissible; `cork_h_length` used with
  `weight="travel_time"` is the deliberately **wrong** heuristic.

**Main results.**
- **7a.** Expanded (100×100, 8-conn, 100 pairs): zero 3,930; Chebyshev 1,119; Euclidean 835; octile 662; all optimal.
  Manhattan expanded 340 but was **suboptimal on 58%** of pairs (mean +1.5%, worst +7.7%).
- **7b.** w = 1 → 531 expanded; w = 1.25 → 151 (only +1.7% cost); w = 5 → 76 expanded, mean +9.6%, worst +41%.
  The bound (≤ w × optimal) held on every run.
- **7c.** 8 moves: mean cost 63.6 vs 71.6, steps 57.8 vs 71.6. A* expands 10% of Dijkstra's nodes on 4-connected (Manhattan)
  and 17% on 8-connected (octile).
- **7d.** Empty 4-connected grid: high_g 33.9 expanded on average (= path length), low_g and fifo 300; same costs. For (5,5) →
  (44,44): 78 vs 1,599 expanded.
- **7e.** Length: Dijkstra 4,659 vs A* 1,215. Time: Dijkstra 4,731 vs A* (haversine/120 km/h) 2,998, both optimal. Wrong
  heuristic: 155 expanded but slower route on 93% of pairs (mean +29%, worst +110%). On the long route: shortest 17.10 km /
  20.1 min; fastest 21.21 km / 16.6 min; wrong heuristic 26.36 km / 22.7 min.

---

## Section 8: Limitations

**Summary.** Measures memory as the grid grows (n = 20 … 100), explains why A*'s open list can be bigger than Dijkstra's, and
tries the ALT landmark heuristic on Cork (Goldberg & Harrelson, 2005).

**Key lines.**
- `open_list_at_peak`: counts stale entries (`entry[3] > best_g[entry[4]]`) when the heap is largest.
- `all_distances(G, source, weight, reverse=True)` follows edges backwards (`G.pred`) to get distances *to* a landmark.
- ALT heuristic: `max(d(L,t) − d(L,u), d(u,L) − d(t,L))` over landmarks: both are lower bounds by the triangle inequality.

**Main result.** At n = 100 Dijkstra generates 3,355 entries vs A* 828, but peak open list is 121 vs 140. At the peak A* has
76 stale entries (Dijkstra 1.6), while live entries are similar (111 vs 118). ALT (8 landmarks, about 1.2–1.3 s set-up per weight):
length 393 expanded vs 1,215 haversine; travel time 592 vs 2,998, all optimal.

---

## Section 9: Animation

**Summary.** A GIF of Dijkstra and A* filling the same 50×50 maze at the same speed, for the slides. A* finishes first (663
expansions) and shows its path while Dijkstra is still spreading (951).

**Key lines.** `draw_frame(frame)` colours the first `frame * nodes_per_frame` expanded cells; `PillowWriter(fps=12)` saves the GIF.

---

## Section 10: Summary of saved files

**Summary.** Lists every file in `figures/` and `tables/` with the section that made it and the report chapter it belongs to,
and checks that nothing is missing. **Main result:** all 21 expected files are present.

---

# Walkthrough of `best_first_search` (using the Section 4 graph)

Call: `astar(worked_graph, "S", "G", worked_h)`, i.e. `best_first_search(..., h=worked_h, w=1, greedy=False, tie_break="high_g")`.
Heuristic values: h(S)=6.08, h(A)=4.03, h(B)=4.92, h(C)=2.50, h(D)=2.50, h(G)=0.

| Code line | What it does | In the example |
|---|---|---|
| `if h is None: h = zero` | No heuristic means Dijkstra. | `h` is given, so this is A*. |
| `best_g = {start: 0.0}` | Cheapest known cost to each node. | `{S: 0}` |
| `parent = {start: None}` | Where each node was reached from. | `{S: None}` |
| `counter = 0` | Insertion number, the last tie-breaker. | 0 |
| `f_start = priority(0.0, h(start, goal), w, greedy)` | f of the start = 0 + 6.08. | 6.08 |
| `open_heap = [(f_start, tie_value(0.0, ...), counter, 0.0, start)]` | The open list holds one entry (rule 1). | `[(6.08, -0, 0, 0, S)]` |
| `expanded, generated, peak_open, order = 0, 1, 1, []` | Counters. | |
| `while open_heap:` | Keep going while there is something to expand. | |
| `f, tie, count, g_value, node = heapq.heappop(open_heap)` | Take the entry with smallest f. | Pop S (f 6.08). |
| `if g_value > best_g[node]: continue` | Rule 2: skip stale entries. | 0 > 0? No. |
| `if node == goal:` | Rule 3: goal test on pop. | S ≠ G. |
| `expanded += 1` | Count the expansion. | expanded = 1 |
| `for neighbour, edge_data in G[node].items():` | Look at each edge out of S. | A (2.6), B (2.0) |
| `g_new = g_value + edge_data[weight]` | Cost through S. | A: 2.6; B: 2.0 |
| `if g_new < best_g.get(neighbour, math.inf):` | Rule 4: strictly better? | Both unseen (∞), so yes. |
| `best_g[...] = g_new; parent[...] = node` | Record the better route. | A: 2.6 from S; B: 2.0 from S |
| `f_new = priority(...)` and `heappush(...)` | Push with f = g + h. | A: 6.63; B: 6.92 |
| `peak_open = max(peak_open, len(open_heap))` | Track memory. | 2 |
| *(next loop)* pop A (6.63) | A has smaller f than B, so A comes first. | Push C (g 5.6, f 8.10), D (g 6.6, f 9.10). S again: 5.2 < 0? No. |
| pop B (6.92) | | D through B: g = 5.0 < 6.6, so **update**: push D (f 7.50). Old D (9.10) is now stale. |
| pop D (7.50) | | G: g = 8, f = 8. A through D: 9 < 2.6? No. B: 8 < 2? No. |
| pop G (8.00) | `node == goal` is true. | `reconstruct_path` gives S → B → D → G; return cost 8, expanded 4. |
| *(never reached)* | C (8.10) and the stale D (9.10) are still in the heap. | If we had popped the stale D, rule 2 would skip it. |
| `return SearchResult([], math.inf, ...)` | Only if the heap empties without reaching the goal. | Not used here. |

Things to say aloud:
- G was **generated** at the D step with g = 8, but we only stopped when it was **popped**. At that moment no open entry has a smaller f, so (with an admissible h) no cheaper path can exist.
- We never edit an entry in the heap (`heapq` cannot do that cheaply). We push a new one and skip the old one later. That is what "stale" means.

---

# Likely exam questions (short answers)

**1. What happens if h overestimates?**
A* may return a path that is not the cheapest, because it can pop the goal before exploring the true best route. Our example:
Manhattan on 8-connected grids gave a costlier path on 58% of pairs (worst +7.7%), and metres-as-seconds on Cork was up to 2.1×
slower. Overestimating usually expands fewer nodes, so it trades optimality for speed (that is also what Weighted A* does, but in a
controlled way).

**2. Why test for the goal when it is popped, not when it is generated?**
When the goal is generated we have *a* path to it, not necessarily the cheapest one. Another open node might lead to a cheaper
route (in Section 4, G was first generated with g = 8 while C, with f = 8.10, was still open). When the goal is popped, its f is the
smallest in the open list; with an admissible h every other open node has f ≤ its true best cost, so none can lead to a cheaper path.

**3. Admissible vs consistent?**
Admissible: h(n) ≤ true cost from n to the goal, for every n. Consistent: h(u) ≤ c(u, v) + h(v) for every edge, and h(goal) = 0.
Consistent implies admissible; the reverse is not always true. With a consistent h, f never decreases along a path, so a node is
never expanded twice. Our code has no closed set: if a cheaper route is found later the node is simply pushed again. So it is still
optimal for an admissible but inconsistent h; it just may re-expand some nodes (Pearl, 1984; Russell & Norvig, 2020).

**4. Why skip stale heap entries?**
`heapq` cannot change the priority of an entry already in the heap. When we find a cheaper route we push a new entry, and the old
one stays behind with a larger g. When it is eventually popped, `g_value > best_g[node]` tells us it is out of date, and expanding it
would only waste work. Cost: extra memory (Section 8: 76 stale entries in A*'s heap at its peak).

**5. What does A* become when h = 0?**
Dijkstra's algorithm: f = g, so nodes are expanded in order of distance from the start. In the notebook, `dijkstra()` is literally
`best_first_search(h=None)`, and `h=None` is replaced with `zero`.

**6. Weighted A* and its bound?**
f = g + w·h with w > 1 (Pohl, 1970). It trusts the heuristic more, so it expands fewer nodes, but the path cost is guaranteed to be
at most w times the optimal (if h is admissible). In Section 7b, w = 1.25 cut work from 531 to 151 nodes for +1.7% cost; at w = 5 the
worst path was only 1.41× optimal, far inside the bound of 5×.

**7. Why does Manhattan fail on 8-connected grids?**
It assumes a diagonal step costs 2 (one across plus one down), but a diagonal move really costs √2 ≈ 1.41. So when the best path
uses diagonals, Manhattan overestimates and is not admissible. Octile distance is the correct heuristic: exact on an empty 8-connected grid.

**8. Why does the travel-time heuristic divide by the maximum speed?**
The heuristic must be in seconds, like the edge weights, and must never overestimate. Haversine distance is the shortest possible
distance; dividing by the *fastest* speed on any road (120 km/h in Cork) gives the shortest possible time. No real trip can be faster.
Dividing by an average speed would give a stronger but possibly overestimating (inadmissible) heuristic. The downside of the
top-speed version is that it is weak in a city (2,998 vs 4,731 expanded), and ALT does much better (592).

**9. What about negative edge weights?**
Dijkstra and A* assume all edges cost ≥ 0. With a negative edge, a node already popped (or the goal) could later be reached more
cheaply, so "goal test on pop" and "a popped node is final" both break. Road lengths, times and grid moves are never negative, so this
is not a problem here. Graphs with negative weights need a different algorithm (for example Bellman–Ford, which is outside this
report's references).

**10. Time and memory complexity?**
With a binary heap, Dijkstra runs in O((V + E) log V) time; our "push a new entry" version stores up to O(E) heap entries, so
O(E log E) ≈ O(E log V). A* with a consistent h has the same worst case (h = 0 *is* Dijkstra), but a good h expands far fewer nodes in
practice. Memory is O(V) for `best_g` and `parent` plus the heap: A* keeps every generated node, and on large maps memory, not time,
is usually the limit (Russell & Norvig, 2020, Ch. 3). Section 8 shows the generated count growing roughly with the number of cells (n²).

**11. (Bonus) Why compare costs and not paths in the correctness check?**
Two different paths can have exactly the same cost; both are correct, and the two implementations may break ties differently.

**12. (Bonus) Does the tie-breaking rule change the answer?**
No, only the work. All three rules gave the same costs; high_g expanded 78 cells vs 1,599 on the empty-grid example.

---

# Figure and table map

LaTeX: put `\usepackage{booktabs}` in the preamble; include tables with `\input{tables/name.tex}` and figures with
`\includegraphics[width=\linewidth]{figures/name.pdf}`.

| File | Report chapter | Caption sentence you can use |
|---|---|---|
| `tables/versions.tex` | Appendix: software | Software versions used for all experiments. |
| `tables/correctness.tex` | Method: validation | Our Dijkstra and A* costs agree with NetworkX on all 600 random start–goal pairs (tolerance $10^{-9}$). |
| `tables/worked_example.tex` | Background / Method | Step-by-step trace of A* on the example graph: the node popped, its $g$, $h$ and $f$ values, and the open list after each step. |
| `figures/worked_graph.pdf` | Background / Method | Example graph with edge weights and straight-line heuristic values $h$; the A* path S–B–D–G (cost 8) is shown in orange. |
| `tables/environments.tex` | Experimental setup | Size of each test environment and the heuristic used on it. |
| `figures/environments.pdf` | Experimental setup | Example environments: (a) 50×50 grid with 25% random walls, (b) 50×50 maze, (c) Cork road network. |
| `tables/results.tex` | Results | Mean ± standard deviation of cost ratio, nodes expanded, peak open-list size and run time for each algorithm and environment. |
| `figures/explored_grid.pdf` | Results | Cells expanded in a 50×50 maze by (a) Dijkstra, (b) greedy best-first and (c) A*; the path found is drawn in black. |
| `figures/cork_route.pdf` | Results | Junctions expanded on a 17.1 km Cork route by (a) Dijkstra and (b) A* with the haversine heuristic. |
| `tables/heuristics.tex` | Results: parameter study (a) | Effect of the heuristic on an 8-connected grid: the inadmissible Manhattan distance expands fewest nodes but returns suboptimal paths. |
| `tables/weighted_astar.tex` | Results: parameter study (b) | Weighted A*: mean nodes expanded and mean/worst cost ratio for each weight $w$. |
| `figures/weighted_astar.pdf` | Results: parameter study (b) | Weighted A*: nodes expanded (left axis) and cost ratio (right axis) against $w$; the dashed line is the guaranteed bound cost $\le w \times$ optimal. |
| `tables/connectivity.tex` | Results: parameter study (c) | 4- versus 8-connected moves on the same 100×100 map: path cost, steps and nodes expanded. |
| `tables/tie_breaking.tex` | Results: parameter study (d) | Tie-breaking rules on empty 50×50 grids: identical costs, very different numbers of nodes expanded. |
| `figures/tie_breaking.pdf` | Results: parameter study (d) | Cells expanded by A* on an empty grid with tie-breaking (a) high $g$, (b) low $g$, (c) first-in-first-out. |
| `tables/cork_weights.tex` | Results: parameter study (e) | Cork network with length and travel-time weights, including an inadmissible heuristic that mixes metres and seconds. |
| `figures/cork_length_vs_time.pdf` | Results: parameter study (e) | Shortest (length) and fastest (travel time) routes between the same two junctions, and the route found with the wrong heuristic. |
| `figures/memory_growth.pdf` | Discussion: limitations | Memory against grid size: (a) peak open-list size, (b) total entries generated, for Dijkstra and A*. |
| `tables/open_list_peak.tex` | Discussion: limitations | Open list at its largest, split into stale and live entries: A*'s extra memory is stale duplicates. |
| `tables/alt.tex` | Discussion: extensions | ALT landmark heuristic (8 landmarks) compared with haversine A* and Dijkstra on 30 Cork routes. |
| `figures/dijkstra_vs_astar.gif` | Slides only | (Animation) Dijkstra and A* filling the same maze at the same speed. |
