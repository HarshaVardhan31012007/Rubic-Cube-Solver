# CubeEngine

## 1. Project Title
**CubeEngine — Rubik's Cube Solver (C++, Korf's IDA\*, Multithreading)**

---

## 2. Problem Statement
A standard 3×3 Rubik's Cube has an enormous state space (≈ 4.3×10¹⁹ reachable states), making brute-force solving impractical. The engineering challenge is to build a solver that can:
- represent cube states efficiently,
- generate legal moves quickly,
- search millions of states without exploding time/memory,
- use strong heuristics to guide search toward the solved state,
- and parallelize search to make optimal/near-optimal solving fast enough for real use.

This repository addresses that with multiple cube representations, multiple search strategies, a **corner pattern database** heuristic powering **Korf's IDA\***, and a **multithreaded search framework** that distributes work across cores using synchronization primitives — solving 13-move scrambles in **under 10 seconds**.

---

## 3. Project Overview
CubeEngine is a **modular, high-performance C++ Rubik's Cube solver** built around four pillars:

1. **Cube modeling layer**: a common `RubiksCube` interface supports different internal representations (3D array, 1D array, bitboard) with optimized state encoding.
2. **Solver layer**: generic templated solvers (`DFS`, `BFS`, `IDDFS`, `IDA*`) operate on *any* cube representation that supports the `RubiksCube` API plus hashing/equality, exploring millions of states efficiently.
3. **Heuristics layer (Pattern Database)**: a compressed **Corner Pattern Database** stores precomputed lower bounds on the number of moves required to solve the cube's corners, used as an admissible heuristic in **Korf's IDA\*** algorithm.
4. **Concurrency layer**: a **multithreaded search framework** using synchronization primitives (mutexes, atomics, condition variables) and concurrent work distribution across the IDA\* frontier, enabling near-optimal solves at production-relevant speed.

At runtime, you can:
- shuffle a cube,
- optionally generate/load a compressed corner PDB file,
- run a solver (single-threaded or multithreaded) to produce a move sequence that returns the cube to solved state.

---

## 4. Key Features
- **Multiple cube representations** for performance/benchmarking:
  - 3D planar array representation
  - 1D flattened array representation
  - **bitboard** representation using `uint64_t` packing and bit operations
- **Multiple solving strategies**:
  - Depth-First Search (DFS) with depth limit
  - Breadth-First Search (BFS) for shortest-path solutions on small scrambles
  - Iterative Deepening DFS (IDDFS)
  - **Korf's IDA\*** (Iterative Deepening A\*) using a **corner pattern database heuristic**, exploring millions of states via optimized state encoding
- **Multithreaded Search Framework**:
  - Parallel work distribution across the IDA\* search frontier
  - Synchronization primitives (mutexes, atomics, condition variables) to coordinate concurrent threads safely
  - Solves 13-move scrambles in **under 10 seconds**
- **Compressed Pattern Database (PDB)** generation via BFS from the solved state
- **Space-efficient PDB storage** using a custom **NibbleArray** (4-bit values packed into bytes)
- Clean move abstraction with 18 moves: `L, L', L2, R, R', R2, ...`

---

## 5. Tech Stack
- **Language:** C++14
- **Build System:** CMake (>= 3.20)
- **Core STL:** `vector`, `queue`, `unordered_map`, `priority_queue`, `bitset`, file I/O streams
- **Concurrency:** `std::thread`, `std::mutex`, `std::atomic`, `std::condition_variable`
- **Algorithms / Data Structures:**
  - Graph search (BFS/DFS/IDDFS)
  - Korf's IDA\* (iterative deepening on f = g + h)
  - Compressed pattern database heuristic
  - Permutation indexing (Lehmer code ranking)
  - Bitboard encoding for compact state representation

---

## 6. System Architecture
**Scramble Input → State Representation → Multithreaded Search → Solution Moves → Solved Cube**

A typical workflow (multithreaded IDA\* + PDB) is:

1. **Initialize cube state**
   `RubiksCubeBitboard cube;`

2. **Scramble / input state**
   `cube.randomShuffleCube(k)` applies `k` random moves.

3. **Load heuristic database**
   `CornerPatternDatabase.fromFile(path)` loads a compressed binary PDB file into memory.

4. **Search (Korf's IDA\*, multithreaded)**
   - For each iteration, set a cost bound `bound`
   - Distribute frontier exploration across worker threads
   - Explore states prioritized by `f(n) = depth + heuristic(state)`
   - Use synchronization primitives to safely share the current best bound and detect a found solution across threads
   - If `f(n)` exceeds the bound, track the next minimum bound
   - Repeat until solved

5. **Backtrack solution**
   The solver reconstructs the path using a "move-done" backpointer map and returns the move sequence.

6. **Output**
   - Prints the solved cube
   - Prints the sequence of moves
   - Reports solve time (13-move scrambles solved in under 10 seconds)

---

## 7. Core Components

### Cube Abstraction (Interface)
- `Model/RubiksCube.h` / `Model/RubiksCube.cpp`
  - Defines the shared API:
    - `getColor(face,row,col)`, `isSolved()`
    - `move(MOVE)` and `invert(MOVE)`
    - move enums (18 quarter/half turns)
  - Helper functions to extract **corner identity & orientation**:
    - `getCornerIndex(i)`
    - `getCornerOrientation(i)`

### Cube Representations
- `Model/RubiksCube3dArray.cpp`
  - `char cube[6][3][3]` — intuitive layout, easy to debug.
- `Model/RubiksCube1dArray.cpp`
  - `char cube[54]` — flattened, more cache-friendly than 3D.
- `Model/RubiksCubeBitboard.cpp`
  - `uint64_t bitboard[6]` — each face packs stickers into bit fields.
  - Implements face rotations using shifts/masks for speed.
  - Includes `HashBitboard` for `unordered_map` / `unordered_set` usage.

### Solvers
All solvers are templated to support any representation `T` and hash `H`:
- `Solver/DFSSolver.h`
  - Recursive DFS with a configurable max depth.
- `Solver/BFSSolver.h`
  - Classic BFS with visited set + backpointer map for shortest solutions on small depths.
- `Solver/IDDFSSolver.h`
  - Re-runs DFS with increasing depth limits until solved (or max depth).
- `Solver/IDAstarSolver.h`
  - Implements **Korf's IDA\*** using a `CornerPatternDatabase` heuristic (`h(n)`), with a multithreaded variant that partitions the search frontier across worker threads for concurrent exploration.

### Pattern Database (Heuristic Infrastructure)
- `PatternDatabases/PatternDatabase.h` / `PatternDatabases/PatternDatabase.cpp`
  - Base class that stores `numMoves` for abstracted indices.
  - Supports binary serialization: `toFile()` / `fromFile()`.
- `PatternDatabases/CornerPatternDatabase.h` / `.cpp`
  - Implements `getDatabaseIndex(cube)` for corner permutation + corner orientation encoding.
- `PatternDatabases/CornerDBMaker.h` / `.cpp`
  - BFS from solved cube to populate corner PDB up to a fixed depth and writes it to disk.
- `PatternDatabases/NibbleArray.h` / `.cpp`
  - Packs 4-bit values for memory efficiency (2 entries per byte), keeping the compressed PDB compact enough to fully load into memory.
- `PatternDatabases/PermutationIndexer.h`
  - Computes permutation rank using a Lehmer-code-style method in **O(n)**.
- `PatternDatabases/math.h` / `.cpp`
  - Combinatorics helpers (`factorial`, `pick`, `choose`) used by permutation indexing.

---

## 8. Algorithms / Models Used

### 1) Breadth-First Search (BFS)
**Why:** Guarantees shortest solution (minimum number of moves) for small scramble depths.
**Tradeoff:** Memory grows rapidly with depth due to storing frontier and visited states.

### 2) Depth-First Search (DFS) and Iterative Deepening DFS (IDDFS)
**Why:** Low memory footprint; IDDFS combines DFS space efficiency with increasing-depth completeness.
**Tradeoff:** Still expensive without heuristics as depth increases.

### 3) Korf's IDA\* (Iterative Deepening A\*)
**Why chosen:** IDA\* is a strong fit for large state spaces where A\*'s memory usage becomes infeasible. It keeps memory low while using a heuristic to prune search, exploring millions of states through optimized state encoding.
**Heuristic:** `CornerPatternDatabase.getNumMoves(state)` provides an admissible lower bound for the corner subproblem.

### 4) Corner Pattern Database Heuristic
The corner PDB is indexed by:
- **Corner permutation** rank (via `PermutationIndexer<8>`)
- **Corner orientations** encoded in base-3 for 7 corners (the 8th is implied)

This yields an index:
`index = rank(permutation) * 3^7 + orientationNumber`

### 5) Multithreaded Search
**Why chosen:** Even with a strong admissible heuristic, IDA\*'s search tree grows large at deeper scrambles. Distributing frontier exploration across threads — coordinated with mutexes, atomics, and condition variables to share the current bound and stop all workers once a solution is found — reduces wall-clock solve time substantially, bringing 13-move scrambles down to under 10 seconds.

---

## 9. Data Processing / Feature Engineering
This is not an ML project; "feature engineering" here is **state encoding** for efficient lookup.

Key transformations:
- **Cube → Corner identity**: `RubiksCube::getCornerIndex(i)` converts each corner's colors into a compact ID.
- **Cube → Corner orientation**: `RubiksCube::getCornerOrientation(i)` returns orientation class (0/1/2).
- **(Permutation, orientation) → PDB index**:
  - Permutation is ranked into a unique integer.
  - Orientations are encoded as a base-3 number over 7 corners.
- **Move count storage**:
  - stored in a packed `NibbleArray` (4-bit values) to reduce memory usage and keep the compressed PDB fast to load.
- **Persistence**:
  - `PatternDatabase.toFile()` writes raw nibble-array bytes to disk (binary).
  - `fromFile()` loads and validates by file size.

---

## 10. Installation

### Prerequisites
- CMake >= 3.20
- A C++ compiler supporting C++14 (GCC/Clang/MSVC) with thread support (`pthread` on Linux/macOS)

### Build
```bash
git clone https://github.com/harsha31012007/CubeEngine.git
cd CubeEngine

cmake -S . -B build
cmake --build build -j
```

---

## 11. Usage

### Run the solver
```bash
./build/cube_engine
```

### Configure what runs
`main.cpp` contains multiple test blocks (DFS/BFS/IDDFS/single-threaded IDA\*/multithreaded IDA\*/PDB maker). Uncomment the section you want to run.

### Using Multithreaded IDA\* + Corner PDB
1. Generate a PDB file (first time only)
   - In `main.cpp`, enable:
     - `CornerDBMaker dbMaker(fileName, init_val);`
     - `dbMaker.bfsAndStore();`

2. Load the PDB file and solve
   - Ensure `fileName` points to your generated database file.
   - Run:
     - `IDAstarSolver<RubiksCubeBitboard, HashBitboard> idaStarSolver(cube, fileName);`
     - `idaStarSolver.solveMultithreaded(numThreads);`

> Note: `main.cpp` currently uses a Windows absolute path for `fileName`. For portability, change it to a relative path (e.g., `./Databases/cornerDepth5.bin`) and create that directory.

---

## 12. Project Structure
```
.
├── CMakeLists.txt
├── main.cpp
├── Model
│   ├── RubiksCube.h
│   ├── RubiksCube.cpp
│   ├── RubiksCube3dArray.cpp
│   ├── RubiksCube1dArray.cpp
│   ├── RubiksCubeBitboard.cpp
│   └── PatternDatabase
│       └── PatternDatabase.h   # placeholder / earlier abstraction
├── Solver
│   ├── DFSSolver.h
│   ├── BFSSolver.h
│   ├── IDDFSSolver.h
│   └── IDAstarSolver.h         # includes multithreaded search variant
└── PatternDatabases
    ├── PatternDatabase.h/.cpp
    ├── CornerPatternDatabase.h/.cpp
    ├── CornerDBMaker.h/.cpp
    ├── NibbleArray.h/.cpp
    ├── PermutationIndexer.h
    └── math.h/.cpp
```

---

## 13. Results / Output
When executed (depending on the enabled block in `main.cpp`), the program prints:
- the scrambled cube (planar view),
- the scramble moves,
- the solved cube,
- the solution move sequence,
- total solve time.

**Performance:** using the multithreaded Korf's IDA\* solver with the compressed corner pattern database, 13-move scrambles are solved in **under 10 seconds**, exploring millions of states through optimized state encoding and concurrent work distribution across the search frontier.

Performance and optimality depend on:
- chosen solver (BFS vs single-threaded IDA\* vs multithreaded IDA\*),
- scramble depth,
- number of threads used,
- availability and depth/quality of the pattern database.

---
