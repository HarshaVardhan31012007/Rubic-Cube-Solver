# Rubik's Cube Solver

## 1. Project Title

**Intelligent Rubik's Cube Solver** – An advanced algorithmic solution for solving Rubik's Cubes using optimized search techniques and pattern databases.

---

## 2. Problem Statement

Solving a Rubik's Cube optimally is a computationally complex problem with **43 quintillion possible states**. While brute-force approaches are infeasible, finding solutions that minimize the number of moves requires sophisticated algorithms that balance:

- **Computational efficiency** – solving quickly within reasonable time
- **Solution optimality** – finding near-optimal or optimal move sequences
- **Memory constraints** – handling the search space without excessive resource consumption

This project addresses these challenges by implementing advanced search algorithms enhanced with pattern databases, a heuristic technique that dramatically reduces search time for complex puzzle states.

---

## 3. Project Overview

This is a **high-performance C++ implementation** of a Rubik's Cube solver that combines:

1. **Efficient state representation** – compact encoding of cube configurations
2. **Advanced search algorithms** – A* search with intelligent heuristics
3. **Pattern databases** – precomputed databases for fast heuristic evaluation
4. **Optimized data structures** – minimal memory footprint for performance

The solver accepts an arbitrary Rubik's Cube scramble and outputs a sequence of moves that restores it to the solved state, optimizing for move count and computation time.

**Data Flow:**
