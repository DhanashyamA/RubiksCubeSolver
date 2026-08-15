# IDA* Solver + Pattern Database + Cube Scanner — Explained

---

## Part 1: The Pattern Database (the "brain" behind IDA*)

Before understanding IDA*, you need to understand how it knows **how far away** a cube is from being solved. That's the pattern database.

### 1.1 The Big Picture

A **Corner Pattern Database** is a precomputed lookup table. For every possible configuration of the 8 corners of a Rubik's cube, it stores the **minimum number of moves** needed to solve just the corners. This acts as a **heuristic** — an optimistic estimate of how many moves the full cube needs.

The cube has 8 corners, each with:
- **8! = 40,320** possible permutations (which corner is in which slot)
- **3⁷ = 2,187** possible orientations (each corner can be twisted 3 ways; the 8th is determined by the other 7)

Total entries: **40,320 × 2,187 = 88,179,840** — but the code uses **100,179,840** as the database size (a rounded superset).

### 1.2 NibbleArray — Space-Efficient Storage

Since move counts fit in 4 bits (0–15), the [`NibbleArray`](file:///c:/Users/Dhanashyam/Desktop/rubiks-cube-solver/PatternDatabases/NibbleArray.h) packs **two entries per byte**, halving memory usage. Each entry is a "nibble" (4 bits).

### 1.3 PermutationIndexer — Ranking Corner Permutations

[`PermutationIndexer.h`](file:///c:/Users/Dhanashyam/Desktop/rubiks-cube-solver/PatternDatabases/PermutationIndexer.h) converts a permutation of 8 items into a unique integer (its **Lehmer code rank**):

| Line | What it does |
|------|-------------|
| **17** | `onesCountLookup` — precomputed table: how many 1-bits are in each number up to `2⁸-1`. Used for O(1) "how many elements have I already seen?" |
| **21** | `factorials` — precomputed partial factorials: `P(N-1-i, K-1-i)` for weighted digit conversion. |
| **43–76** | `rank()` — converts a permutation to its Lehmer code, then to a base-10 index. For each element, it counts how many smaller unseen elements exist before it (using the bitset + ones-count lookup). |

### 1.4 CornerPatternDatabase — The Index Function

[`CornerPatternDatabase.cpp`](file:///c:/Users/Dhanashyam/Desktop/rubiks-cube-solver/PatternDatabases/CornerPatternDatabase.cpp) combines permutation rank + orientation into a single flat index:

```
getDatabaseIndex(cube):
    cornerPerm = [getCornerIndex(0), ..., getCornerIndex(7)]   // which corner is in each slot
    rank = permIndexer.rank(cornerPerm)                         // → 0..40319

    cornerOrientations = [getCornerOrientation(0), ..., getCornerOrientation(6)]  // 0, 1, or 2 each
    orientationNum = o[0]*729 + o[1]*243 + o[2]*81 + o[3]*27 + o[4]*9 + o[5]*3 + o[6]
                                                                // base-3 number → 0..2186

    return rank * 2187 + orientationNum                         // unique flat index
```

This maps every possible corner configuration to a unique integer in `[0, 88,179,839]`.

### 1.5 CornerDBMaker — Building the Database via BFS

[`CornerDBMaker.cpp`](file:///c:/Users/Dhanashyam/Desktop/rubiks-cube-solver/PatternDatabases/CornerDBMaker.cpp) populates the database using a **BFS from the solved state**:

| Line | Code | What it does |
|------|------|-------------|
| **19** | `RubiksCubeBitboard cube;` | Start from the **solved** cube. |
| **21** | `q.push(cube);` | Seed BFS with solved state. |
| **22** | `cornerDB.setNumMoves(cube, 0);` | Solved state needs **0 moves**. |
| **24** | `while (!q.empty())` | Level-by-level BFS. |
| **25–26** | `n = q.size(); curr_depth++;` | Process entire current level, then increment depth. |
| **27** | `if (curr_depth == 9) break;` | Stop at depth 8 (moves 0 through 8 are stored). |
| **31–38** | For each state, try all 18 moves. If the neighbor hasn't been seen at a shallower depth (`getNumMoves > curr_depth`), record `curr_depth` and enqueue it. |
| **43** | `cornerDB.toFile(fileName);` | Serialize the entire database to a binary file for reuse. |

> [!IMPORTANT]
> This BFS is a **one-time offline computation**. The resulting binary file is loaded at runtime by the IDA* solver, so the solver never needs to rebuild it.

### 1.6 PatternDatabase Base Class — Storage Operations

[`PatternDatabase.cpp`](file:///c:/Users/Dhanashyam/Desktop/rubiks-cube-solver/PatternDatabases/PatternDatabase.cpp):

| Method | What it does |
|--------|-------------|
| `setNumMoves(ind, numMoves)` | Only updates if `numMoves < oldMoves` (keeps the minimum). Tracks count of filled entries via `numItems`. |
| `getNumMoves(cube)` | Calls `getDatabaseIndex(cube)` → looks up the nibble array → returns the stored move count. |
| `toFile()` / `fromFile()` | Binary serialization. Writes/reads the raw nibble array to/from disk. Validates file size on load. |
| `isFull()` | Returns `true` when all entries have been filled. |

---

## Part 2: The IDA* Solver — Line by Line

[`IDAstarSolver.h`](file:///c:/Users/Dhanashyam/Desktop/rubiks-cube-solver/Solver/IDAstarSolver.h) implements **Iterative Deepening A\***, combining the memory efficiency of DFS with the optimality guarantees of A*.

### 2.1 Private Members (Lines 16–19)

| Line | Member | Purpose |
|------|--------|---------|
| **16** | `CornerPatternDatabase cornerDB` | The precomputed heuristic database loaded from disk. |
| **17** | `vector<MOVE> moves` | Final solution path. |
| **18** | `unordered_map<T, MOVE, H> move_done` | Back-pointer map (same as BFS): records which move reached each state. |
| **19** | `unordered_map<T, bool, H> visited` | Tracks visited states within a single iteration to avoid cycles. |

### 2.2 Node Struct (Lines 21–27)

```cpp
struct Node {
    T cube;       // the cube state
    int depth;    // g(n): actual moves taken so far
    int estimate; // h(n): heuristic estimate from pattern DB
};
```

The key value in A* is **f(n) = g(n) + h(n)** = `depth + estimate`.

### 2.3 compareCube — Priority Queue Ordering (Lines 29–36)

```cpp
// Returns true when p1 should have LOWER priority than p2
if (f(n1) == f(n2)):
    prefer the one with smaller estimate (i.e., deeper actual depth)
else:
    prefer the one with smaller f(n)
```

This is a **min-heap** on f-value. When f-values tie, it prefers states that are deeper (more actual moves, smaller heuristic) — these are "closer to being solved" and more worth exploring.

### 2.4 The Core `IDAstar(int bound)` Function (Lines 46–80)

This runs a **bounded A\* search** — it explores all states with `f(n) ≤ bound`:

| Line | Code | What it does |
|------|------|-------------|
| **48** | `priority_queue<...> pq;` | Min-heap ordered by f-value (via `compareCube`). |
| **49** | `Node start = Node(rubiksCube, 0, cornerDB.getNumMoves(rubiksCube));` | Create start node: depth=0, estimate=heuristic lookup from pattern DB. |
| **50** | `pq.push(make_pair(start, 0));` | Push start node with a dummy move (0). |
| **51** | `int next_bound = 100;` | Tracks the **smallest f-value that exceeded the current bound**. This becomes the bound for the next iteration. |
| **52–55** | Pop the node with the lowest f-value from the priority queue. | |
| **57** | `if (visited[node.cube]) continue;` | Skip if already visited (cycle prevention). |
| **59** | `visited[node.cube] = true;` | Mark as visited. |
| **60** | `move_done[node.cube] = MOVE(p.second);` | Record the move that reached this state (for path reconstruction). |
| **62** | `if (node.cube.isSolved()) return make_pair(node.cube, bound);` | **Goal found!** Return the solved cube and the current bound (signals success). |
| **63** | `node.depth++;` | Increment depth for children. |
| **64–75** | **For each of the 18 moves**: |
| **66** | `node.cube.move(curr_move);` | Apply move. |
| **67** | `if (!visited[node.cube])` | Skip visited states. |
| **68** | `node.estimate = cornerDB.getNumMoves(node.cube);` | **Look up the heuristic** from the pattern database — "at least this many moves to solve the corners". |
| **69** | `if (estimate + depth > bound)` | If f(n) exceeds the bound, **don't explore** — but **record it** as a candidate for the next iteration's bound. |
| **70** | `next_bound = min(next_bound, estimate + depth);` | Track the tightest next bound. |
| **72** | `pq.push(make_pair(node, i));` | Within bound → push to priority queue for exploration. |
| **75** | `node.cube.invert(curr_move);` | Undo the move. |
| **79** | `return make_pair(rubiksCube, next_bound);` | No solution found within this bound → return the next bound to try. |

### 2.5 The `solve()` Outer Loop (Lines 90–109)

This implements the **iterative deepening** part:

| Line | Code | What it does |
|------|------|-------------|
| **91** | `int bound = 1;` | Start with a tight bound (f ≤ 1). |
| **92** | `auto p = IDAstar(bound);` | Run bounded A*. |
| **93** | `while (p.second != bound)` | If `p.second == bound`, a solution was found. If `p.second > bound`, we need to increase the bound and try again. |
| **94** | `resetStructure();` | Clear visited/move_done maps for the new iteration. |
| **95–96** | `bound = p.second; p = IDAstar(bound);` | Set bound to `next_bound` and re-run. |
| **98–107** | **Path reconstruction** — identical to the BFS solver: walk back from solved state using `move_done`, then reverse. |

### 2.6 How IDA* + Pattern Database Work Together — Visual

```
Iteration 1: bound = 1
  Start → h=5, f=0+5=5 > 1 → PRUNED.  next_bound = 5

Iteration 2: bound = 5
  Start → f=5 ≤ 5 → explore
    → move L → h=4, f=1+4=5 ≤ 5 → explore
      → move R → h=4, f=2+4=6 > 5 → PRUNED. next_bound = 6
      → move F → h=3, f=2+3=5 ≤ 5 → explore
        ...
  No solution found. next_bound = 6

Iteration 3: bound = 6
  ... explores deeper ...

Eventually: bound = actual solution length → FOUND!
```

The pattern database provides the `h` values. Without it, IDA* would degenerate to IDDFS (no pruning intelligence). With it, vast branches are **pruned** because the heuristic proves they can't lead to a solution within the current bound.

---

## Part 3: The Cube Scanner — Line by Line

[`CubeScanner.cpp`](file:///c:/Users/Dhanashyam/Desktop/rubiks-cube-solver/Scanner/CubeScanner.cpp) uses OpenCV to scan a physical Rubik's Cube via webcam and populate the internal cube model.

### 3.1 Color Map (Lines 12–20)

```cpp
const map<RubiksCube::COLOR, Scalar> CubeScanner::colorMap = {
    {WHITE,   Scalar(255, 255, 255)},   // BGR format
    {RED,     Scalar(0, 0, 255)},
    {ORANGE,  Scalar(0, 165, 255)},
    {YELLOW,  Scalar(0, 255, 255)},
    {GREEN,   Scalar(0, 255, 0)},
    {BLUE,    Scalar(255, 0, 0)},
    {UNKNOWN, Scalar(50, 50, 50)},
};
```

Maps each cube color to an OpenCV BGR `Scalar` for drawing detected faces on screen.

### 3.2 Constructor / Destructor (Lines 22–30)

| Line | What it does |
|------|-------------|
| **23** | `cap.open(camIndex)` — Opens the webcam (default camera index 0). |
| **24** | Throws an exception if the webcam can't be opened. |
| **28–29** | Destructor: releases the camera and closes all OpenCV windows. |

### 3.3 `classifyColor()` — Color Detection (Lines 32–53)

This is the core color-recognition logic:

| Line | What it does |
|------|-------------|
| **33–36** | Convert the BGR pixel to **HSV** (Hue-Saturation-Value) color space. HSV is much better for color classification than RGB because hue is independent of brightness. |
| **37** | Extract `h` = hue value (0–180 in OpenCV). |
| **39–44** | **White detection**: All three BGR channels > 200, and all channels within 30 of each other. White doesn't have a distinct hue, so it's detected by high brightness + low color variance. |
| **46** | `h ∈ [160, 190]` → **Red** (red wraps around 180° in the hue circle). |
| **47** | `h ∈ [3, 19]` → **Orange**. |
| **48** | `h ∈ [20, 30]` → **Yellow**. |
| **49** | `h ∈ [60, 90]` → **Green**. |
| **50** | `h ∈ [100, 120]` → **Blue**. |
| **52** | Fallback → **White** (if no hue range matches). |

Hue ranges visualized on the color wheel:
```
  0°────────────────────────────────────180°
  |  Orange  |Yellow|    Green   | Blue  | Red  |
  |  3─19    |20─30 |   60─90    |100─120|160─190|
```

### 3.4 `medianColor()` — Noise Reduction (Lines 55–69)

Instead of reading a single pixel (which could be noisy), this samples a **region of pixels** around the center and takes the **median** of each channel:

| Line | What it does |
|------|-------------|
| **56** | `half = region / 2` — defines a square region (default 5×5 pixels). |
| **58–65** | Collect all B, G, R values from the region into separate vectors. Bounds-checks each pixel. |
| **66** | Sort each channel's values independently. |
| **67–68** | Return the **median** (middle value) of each channel → a robust representative color. |

### 3.5 `captureFace()` — Webcam Face Capture (Lines 71–104)

This is the interactive scanning loop for **one face**:

| Line | What it does |
|------|-------------|
| **75–89** | **Live preview loop**: Continuously reads frames from webcam. Draws a **3×3 grid overlay** centered on the frame. Displays `"Press SPACE to capture"`. Breaks when spacebar (ASCII 32) is pressed. |
| **78–79** | `startX`, `startY` = top-left corner of the 3×3 grid, centered in the frame. |
| **81–84** | Draw 4 horizontal + 4 vertical white lines forming the 3×3 grid. |
| **91–103** | **After capture**: Read a fresh frame. For each of the 9 cells in the 3×3 grid: |
| **98–99** | Calculate the center pixel `(x, y)` of cell `[i][j]`. |
| **100** | Get the `medianColor` at that point (noise-resistant). |
| **101** | Classify it into a `RubiksCube::COLOR`. |
| **103** | Return the 3×3 color grid. |

### 3.6 `drawColorFace()` — Visual Feedback (Lines 106–120)

Draws a **colored 3×3 grid** showing what the scanner detected for one face. Each cell is filled with the detected color. Shows `"Press [R] to rescan or [N] for next"` at the bottom.

### 3.7 `drawFullCube()` — Cube Net Visualization (Lines 122–145)

Draws all 6 scanned faces in a **cross/net layout**:

```
         [Face 0]
[Face 1] [Face 2] [Face 3] [Face 4]
         [Face 5]
```

| Line | Layout position |
|------|----------------|
| **140** | Face 0 at row 0, col 3 (top) |
| **141** | Faces 1–4 at row 3, cols 0, 3, 6, 9 (middle row) |
| **142** | Face 5 at row 6, col 3 (bottom) |

### 3.8 `scan()` — The Main Scanning Orchestrator (Lines 147–174)

| Line | What it does |
|------|-------------|
| **148** | Initialize a 6×3×3 grid with all WHITE. |
| **150** | Loop over all **6 faces**. |
| **151** | Inner `while(true)` — allows **rescanning** a face. |
| **152** | `captureFace()` — capture the current face from webcam. |
| **155–159** | Draw the detected face and the full cube net so far. Show both windows. |
| **161** | `waitKey(0)` — wait for a keypress. |
| **162–167** | If **'N'** is pressed: accept this face → call `cube.setColor(face, i, j, color)` for all 9 cells → close the face window → `break` to move to next face. |
| **(implicit)** | If **'R'** is pressed (or any other key): the `while(true)` loop continues → re-capture this face. |
| **172–173** | After all 6 faces are scanned: release camera and close all windows. |

### 3.9 Full Scanning Flow

```
┌─────────────────────────────────────────────────┐
│  For each of the 6 faces:                       │
│                                                 │
│  1. Show live webcam with 3×3 grid overlay      │
│  2. User aligns face → presses SPACE            │
│  3. Capture frame → sample 9 center points      │
│  4. medianColor() → classifyColor() per cell    │
│  5. Show detected colors + cube net so far       │
│  6. User presses N (accept) or R (rescan)       │
│  7. If accepted → write colors to cube model    │
│                                                 │
│  After all 6 faces:                             │
│  → cube object is fully populated               │
│  → passed to IDA* solver                        │
└─────────────────────────────────────────────────┘
```

---

## End-to-End Pipeline (from [`main.cpp`](file:///c:/Users/Dhanashyam/Desktop/rubiks-cube-solver/main.cpp))

```cpp
CubeScanner scanner(0);           // Open webcam
RubiksCubeBitboard cube;
scanner.scan(cube);               // Scan all 6 faces into cube

IDAstarSolver solver(cube, fileName);  // Load corner DB from file
auto moves = solver.solve();           // IDA* with pattern DB heuristic

// Print solution
for (auto move : moves) cout << cube.getMove(move) << " ";
```

1. **Scanner** reads the physical cube → populates `cube`
2. **IDA\* solver** loads the precomputed corner pattern database from disk
3. Solver runs iterative deepening A* with increasing f-value bounds
4. Pattern database provides the heuristic `h(n)` at each node
5. Solution is reconstructed via back-pointers and returned
