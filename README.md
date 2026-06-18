***

```markdown
<p align="center">
  <img src="assets/logo.png" alt="EngineInspector Logo" width="220" height="220" />
</p>

<h1 align="center">EngineInspector</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Python-v3.10+-blue.svg?logo=python&logoColor=white" alt="Python v3.10+" />
  <img src="https://img.shields.io/badge/Rust-Edition%202021-orange.svg?logo=rust&logoColor=white" alt="Rust Edition 2021" />
  <img src="https://img.shields.io/badge/Static%20Analysis-AST%20%7C%20CFG%20%7C%20SSA-red.svg" alt="Static Analysis" />
  <img src="https://img.shields.io/badge/Security-100%25%20Local%20Only-green.svg" alt="100% Local Only" />
  <img src="https://img.shields.io/badge/Performance-Zstd%20%7C%20Blake3-blueviolet.svg" alt="Performance" />
</p>

***

## Executive Summary
EngineInspector is a high-performance static analysis and semantic diffing suite built on a fast hybrid architecture (Python orchestration + Rust computation core). It parses source code into memory-safe arenas, reconstructs control flow, and compiles variables into Static Single Assignment (SSA) representations—allowing you to audit structural risks and track behavioral changes completely locally.

> **Proprietary yet 100% Free:** EngineInspector is closed-source to safeguard its highly optimized analysis algorithms, but it is **100% free** for both community developers and commercial workspaces.

***

## Table of Contents
*   [1. Download Binaries](#1-download-binaries)
*   [2. Quick Start (Usage)](#2-quick-start-usage)
*   [3. Key Features](#3-key-features)
*   [4. Security & Offline Integrity](#4-security--offline-integrity)
*   [5. Architectural Deep Dive](#5-architectural-deep-dive)
*   [6. The Power of "Semantic Diffing"](#6-the-power-of-the-semantic-diff)
*   [7. FAQ & Advanced Details](#7-faq--advanced-details)
*   [8. Community & Fast-Response Support](#8-community--fast-response-support)

***

## 1. Download Binaries

Get the pre-compiled, standalone executable for your operating system:

*   **[📥 Download EngineInspector for Windows (engineinspector.exe)](#)**
*   **[📥 Download EngineInspector for Linux (engineinspector)](#)**

***

## 2. Quick Start (Usage)

EngineInspector is distributed as a single native binary. For Linux environments, make sure to grant executable permissions before launching:
```bash
chmod +x engineinspector
```

### Analyzing Individual Files
Compare two source modules (C++ or Python) and print a structural analysis report to the console:
```bash
# Windows
engineinspector.exe compare-files path/to/before.cpp path/to/after.cpp

# Linux
./engineinspector compare-files path/to/before.py path/to/after.py
```
Save the semantic analysis directly to a JSON report:
```bash
./engineinspector compare-files left.cpp right.cpp --out reports/diff_report.json
```

### Analyzing Complete Projects (Directory Trees)
Recursively compare two workspace directories to isolate semantic modifications:
```bash
./engineinspector compare-paths path/to/original_project/ path/to/modified_project/ --out-dir reports/
```

### Workspace Scaffolding & Auditing
```bash
# Initialize a standard project workspace directory
./engineinspector init my_project/

# Verify local engine status and check for updates
./engineinspector version
```

***

## 3. Key Features

*   **🌐 C++ and Python Analysis:** Provides out-of-the-box support for both C++ (via libclang/Tree-Sitter) and Python (via native lexical tokenization & Tree-Sitter) source structures.
*   **📊 Parallel Control Flow Graph (CFG) Compiler:** Operates on multi-threaded Tokio workers to build linearized basic blocks and SSA mappings across massive codebases.
*   **🌳 Precise Rename & Move Tracking:** Identifies structural renames and moved subtrees, ignoring cosmetic whitespace changes and file reformatting.
*   **⚙️ Bytecode Rule Engine VM:** Executes precompiled bytecode checks within a sandboxed virtual machine under strict, customizable instruction-level budgets.
*   **📈 Behavioral Shift & Heuristic Detection:** Automatically tracks search depth alterations, thread management patterns, evaluation score thresholds, and aspiration windows.
*   **🔒 Local execution with first-run bootstrap protection:** Secures your system's integrity by fetching external dependencies only from verified repositories on the very first run.

***

## 4. Security & Offline Integrity

### 🛡️ One-Time Verified Bootstrap
To maintain a small file size, the executable does not bundle heavy platform-specific shared libraries (`libclang`). On its **very first launch**, EngineInspector performs an automated dependency download. 

*   **Authenticity Guaranteed:** These dependency binaries are fetched exclusively from our official, cryptographically validated project repository (`EngModiDev`).
*   **Cryptographic Verification:** Every downloaded asset is validated against a secure companion `.sha256` signature before it is mapped into memory.
*   **100% Offline Execution:** Once this initial bootstrap is completed, the engine locks all external sockets. The tool operates 100% offline, ensuring your proprietary code and file structures never leave your machine.

***

## 5. Architectural Deep Dive

<details>
<summary><b>🛠️ Click to expand: Deep Technical Features &amp; Architecture</b></summary>
<br>

### A. Parser Merging & Comment-Safe Normalization
*   **Hybrid AST Parser:** Implemented in `ast_parser.py`, the `ASTParser` class combines Tree-Sitter structural trees with the type-attribution of `libclang` cursors via `_merge_cpp_units`. Tree-Sitter builds the AST hierarchy, while `libclang` parses templates, preprocessor macros, and compiler-level semantic types to construct a comprehensive `ASTNodeRef`.
*   **Zero-Shift Comment Stripping:** Traditional comment removal shifts code offsets, which breaks AST line mapping. Our `Normalizer` class (`normalizer.py`) uses a zero-shift replacement masking technique: it queries Tree-Sitter for comment ranges, then overwrites non-newline comment bytes with spaces (`32` in UTF-8 bytes) while keeping newlines (`10`) intact. This preserves the absolute line map for accurate diff referencing. Python streams are processed via the standard `tokenize` module.
*   **Unify Indentation and Stable Line Mapping:** The normalizer expands tabs, normalizes newlines, tracks block scopes with an indentation stack, and maintains a stable `line_map` mapping the raw file to the normalized content.

### B. Control Flow, SSA & Dataflow Solver (Rust Core)
*   **O(1) Arena-Backed AST Storage (`AstArena`):** In `ast/arena.rs`, the Rust core stores tree nodes contiguously as `NodeRecord` entries within a packed vector (`PackedVec` in `utils/memlayout.rs`). References between nodes are stored as compact `NodeId(u32)` indices to maximize CPU L1/L2 cache hit rates and eliminate pointer chasing. Out-of-line metadata (`NodeMeta`) keeps individual node footprints small. A `StringInterner` maps identifiers to `u32` keys to speed up comparisons.
*   **AstArena Compaction & Remapping:** The `compact_and_remap` method removes logically deleted nodes (marked via tombstones) from the arena, reconstructs the packed nodes vector, and updates remaining parent-child index references.
*   **Parallel Control Flow Graph (CFG) Compiler:** The `CfgBuilder` (`cfg/builder.rs`) compiles AST nodes into basic blocks (`Block`) containing linearized instructions (`Instr`). It utilizes Tokio tasks to generate CFG structures for independent functions in parallel, merging them into a sharded global `Cfg` container with minimal lock contention.
*   **Lengauer-Tarjan SSA Transformation:** The `SsaBuilder` (`dfa/ssa.rs`) translates compiled CFG blocks into Static Single Assignment (SSA) form. It computes immediate dominator trees (`idom`) via a deterministic implementation of the Lengauer-Tarjan algorithm, identifies dominance frontiers, and places $\phi$-nodes at iterated dominance frontiers.
*   **Sparse, Demand-Driven Dataflow Solver:** The `SparseSolver` (`dfa/sparse_solver.rs`) executes forward and backward dataflow queries over the SSA form. It avoids global iteration by propagating lattice values (such as the `ConstLattice` for constant propagation) only along use-def chains. It also features incremental invalidation via `invalidate_instrs` to re-run queries only on affected blocks.
*   **Interprocedural Call Graph Extractor:** The `CallGraphExtractor` (`callgraph/extractor.rs`) runs parallel analyses over function CFGs to resolve call targets. Direct calls are mapped straight to symbol IDs, while indirect calls are resolved using points-to analysis on variables in the global `SymbolIndex` (`callgraph/index.rs`).

### C. Advanced Heuristics & Metrics
Calculated in the Python analysis layer (`analysis/metrics.py`), the `Metrics` class provides structural and statistical insights into code modifications:
*   **Shannon Entropy over Token Frequency:** Measures information density within a file. It tokenizes the source with `_TOKEN_RE`, calculates individual token probability distributions, and evaluates Shannon entropy utilizing `math.log2(p)`, normalized against the total unique token count.
*   **Heuristic Cyclomatic Complexity:** Approximates structural branching complexity by counting occurrences of conditional, looping, and branching operators via precompiled regex structures (`_BRANCH_RE`), establishing a baseline score.
*   **Readability (Readability) Estimation:** Uses a heuristic based on physical line lengths, average line length, and token density per line to estimate code readability on a scale of 0 to 1.
*   **Squashed Churn Scoring:** Combines normalized line deltas, token count changes, and cyclomatic complexity differentials into a single metric. The final score is normalized using a tanh-like mapping (`math.tanh(raw * 1.5)`) to produce a value between 0 and 1.

### D. Security, Risk and Custom DSL
*   **Preprocessor-Aware Comment Stripping:** To identify true changes, `RiskClassifier` (`analysis/risk_classifier.py`) implements `_is_comment_only_change`. Traditional comment strippers can accidentally strip C/C++ preprocessor directives starting with `#` (such as `#define MAX_DEPTH 64`), which are critical macro definitions. Our preprocessor-aware comment stripper uses a negative lookahead regex to preserve these directives while stripping genuine Python/shell-style `#` comments:
    ```python
    re.sub(
        r"#(?!\s*(include|define|undef|if|ifdef|ifndef|elif|else|endif|pragma|error|warning|line)\b).*?$",
        "",
        s,
        flags=re.MULTILINE
    )
    ```
*   **Unbiased Sigmoid Risk Scoring:** Standard sigmoids map a neutral inputs (raw score = 0) to 0.5, which falsely categorizes minor, safe changes as medium-risk. The `RiskClassifier` corrects this bias in its `_compute_risk` pipeline by utilizing an offset exponential curve:
    $$\text{Score} = 1.0 - 2^{-\max(0.0, \text{raw})}$$
    This maps a raw score of 0 directly to 0 (categorized as low-risk/info), while maintaining a smooth saturation curve toward 1.0 as risk signals accumulate.
*   **Heuristic Engine-Behavior Detector:** The `BehaviorDetector` (`analysis/behavior_detector.py`) checks for semantic changes in performance-critical code. It uses precompiled regex matrices to track changes in stopping criteria (e.g., search timeouts, ponder states), time management clocks, move ordering, aspiration windows, and multithreading primitives (`std::mutex`, `std::atomic`).
*   **Sandboxed VM & custom Rule DSL:** The system includes a custom rule DSL parser (`rules/language.rs`) that compiles declarative assertions into a compact bytecode format (`OpCode`). The bytecode is executed inside a sandboxed register-based interpreter (`Vm` in `rules/engine.rs`). The VM prevents infinite loops and denial-of-service attempts by enforcing strict limits on instruction counts and execution times.

### E. Performance & Infrastructure
*   **Postcard Serialization & CRC64fast Checksums:** Located in `cfg/compress.rs`, this module serializes SSA and CFG representations into a version-controlled binary format (`ENGCFGv1`). It uses postcard serialization, validates data integrity with SIMD-accelerated `crc64fast` checksums, and applies optional `zstd` compression to reduce storage requirements.
*   **Atomic Replacement Auto-Updater:** The `EngineUpdater` (`core/updater.py`) queries a secure remote update manifest. It downloads new binaries, verifies their SHA-256 signatures, and performs atomic executable swaps on Windows by renaming active files (`active_exe.rename(backup_exe)`). It also includes fallback rollback handlers to restore previous versions if a crash occurs on first boot.
*   **Crash Boundary & State Harvest:** Uncaught runtime exceptions are captured by the boundary layer (`cli/boundary.py`). On failure, the system polls host diagnostics and active memory RSS parameters (`psutil`), generating a localized crash report (`crash_dump_*.json`) for debugging.
</details>

***

## 6. The Power of "Semantic Diffing"

<details>
<summary><b>🌳 Click to expand: Git Diff vs. Semantic Diffing</b></summary>
<br>

Traditional line-by-line diffing utilities (such as `git diff`) operate purely on text sequences. Consequently, simple modifications like reformatting code, renaming a variable, or moving a block trigger large, misleading diff outputs. 

EngineInspector approaches diffing structurally using an abstract syntax tree parser and semantic matching:

```
    Source A (Before)                    Source B (After)
┌───────────────────────┐            ┌───────────────────────┐
│ void search() {       │            │ void search() {       │
│   int max_depth = 64; │            │   // Indent & rename  │
│   run(max_depth);     │            │   int limit = 64;     │
│   run(limit);         │
│ }                     │            │ }                     │
└──────────┬────────────┘            └──────────┬────────────┘
           │                                    │
           ▼ (AST Construction)                 ▼ (AST Construction)
┌───────────────────────┐            ┌───────────────────────┐
│ AstArena [A]          │            │ AstArena [B]          │
│  - Fn: search         │            │  - Fn: search         │
│  - VarDecl: max_depth │            │  - VarDecl: limit     │
│  - Call: run          │            │  - Call: run          │
└──────────┬────────────┘            └──────────┬────────────┘
           │                                    │
           ▼ (Subtree Hashing)                  ▼ (Subtree Hashing)
┌───────────────────────┐            ┌───────────────────────┐
│ Blake3 Fingerprints   │            │ Blake3 Fingerprints   │
│  - Leaf: 0x4fbc72...  │            │  - Leaf: 0x4fbc72...  │
│                       │            │    (Exact Match!)     │
└──────────┬────────────┘            └──────────┬────────────┘
           │                                    │
           └─────────────────┬──────────────────┘
                             ▼
               ┌───────────────────────────┐
               │    Matcher Heuristics     │
               │ (Rename / Move Detection) │
               └─────────────┬─────────────┘
                             │
                             ▼
               ┌───────────────────────────┐
               │     Stable Edit Script    │
               │  Rename: max_depth->limit │
               │   (Indentation Ignored)   │
               └───────────────────────────┘
```

1.  **BLAKE3 Subtree Fingerprinting:** The `SemanticHasher` (`diff/semantic_hash.rs`) traverses the `AstArena` from the bottom up, generating a cryptographically stable 256-bit BLAKE3 hash for every node and its children. It uses canonical sorting to ensure that children reordered during compilation yield identical signatures.
2.  **Greedy Ast Matching:** The `Matcher` (`core/matcher.py`) aligns nodes using their semantic fingerprints and kinds. Nodes with identical signatures are matched immediately in O(1) time. For modified subtrees, it falls back to a structural similarity check that evaluates node sizes and child structures.
3.  **Strict Identity Verification:** To prevent mismatched pairings, the matcher verifies node identity. Functions or variables with completely different signatures are never paired together as a "modify" chunk, ensuring that renames are distinguished from unrelated code edits.
4.  **Rename & Move Detection:** When two matching nodes of the same kind contain different text, the system calculates a weighted confidence score. This score evaluates span overlap, parent context similarity, and Levenshtein token similarity. If the score exceeds a threshold (typically 0.7), the change is classified as a rename. Similarly, nodes that appear under a different parent are classified as moves rather than deletes/inserts.
</details>

***

## 7. Security & Transparency (False Positives)

<details>
<summary><b>⚠️ Installation Notes, Windows SmartScreen & Antivirus Heuristics</b></summary>
<br>

As EngineInspector is an independently developed, closed-source utility, you may encounter security warnings when running the standalone executable for the first time.

### Why do these warnings occur?
*   **Unsigned Standalone Binaries:** Standalone executable packages lack expensive enterprise code-signing certificates, which triggers precautionary warnings from protective filters like Microsoft Defender SmartScreen or Linux security layers.
*   **Low-Level Shared Memory Allocations:** To maximize execution speed, our shared library loader (`core/linker.py`) dynamically maps dynamic libraries (`libclang`) into the active process memory using standard platform APIs (`ctypes.CDLL` / `os.add_dll_directory`). These low-level dynamic allocations are flagged by heuristic antivirus software as suspicious patterns (historically mapped to dynamic sideloading risks).

### Reassuring Core Metrics:
Our dynamic bootstrapper validates all assets via cryptographically signed SHA-256 signatures before they are allowed to bind with process memory. Your source code and parsed structural metrics are stored entirely in local processes; EngineInspector does not transmit any of your code or environment paths.
</details>

***

## 8. Advanced FAQ

<details>
<summary><b>💬 Click to expand: Advanced FAQ</b></summary>
<br>

### Q: Why do I see `"semantic_change": null` in some JSON reports?
A: If a structural diff contains `"semantic_change": null`, **this is not a bug**. We are currently designing a deterministic semantic classification engine to identify highly precise syntax modifications without triggering false positives. We choose to return `null` instead of guessing or returning unstable classifications. This feature is actively scheduled for the next release, matching the strict performance standards of the rest of the application.

### Q: How does the PyO3 FFI manage memory without GIL-based overhead?
A: To prevent Python's Global Interpreter Lock (GIL) from slowing down performance, the Rust core operates on thread-safe, atomically referenced structures (`Arc`). The Python layer interacts with the Rust backend using opaque `u64` handles managed by a global, mutex-protected `HandleManager`. When Python triggers a heavy task (such as `build_callgraph_from_cfg_py`), it passes these handles across the FFI boundary, releases the GIL, and allows the Rust core to run parallel processes on Rust-managed memory threads.

### Q: What bytecode structure is executed by the Rule VM?
A: The Rule VM executes a stack-based instruction set defined by the `OpCode` enum. When a rule is loaded, the parser extracts literals into a structured `ConstantPool` to avoid string allocations during VM execution. Comparison operations push constant indices and evaluate variables against the current node's AST properties using a lightweight `HostApi` trait. The VM runs on a strict evaluation budget to protect against performance degradation or infinite loops:
```rust
let check_budget = |instr_count: u64, start: Instant| {
    if instr_count > self.budget.max_instructions {
        return Err("instruction budget exceeded");
    }
    if start.elapsed() > self.budget.max_duration {
        return Err("time budget exceeded");
    }
    Ok(())
};
```

### Q: How does EngineInspector integrate into CI/CD pipelines?
A: EngineInspector is designed to integrate into automated pipelines. The CLI commands return standard, reliable exit codes (`0` for clean execution, `1` for failures, and `130` for user interrupts). The `--out` flag writes structured reports directly to JSON, which can be easily parsed by downstream step processors to enforce commit validation rules:
```json
{
  "summary": {
    "added": 12.0,
    "removed": 2.0,
    "modified": 5.0,
    "matches": 142,
    "chunks": 19,
    "duration_seconds": 0.142
  },
  "metadata": {
    "language": "cpp",
    "node_counts": { "left": 1402, "right": 1548 },
    "ast_depth": { "left": 18, "right": 21 }
  }
}
```

### Q: How does the incremental diff engine prevent recursive rehashing?
A: When a code change is made, recalculating hashes for the entire AST tree would be inefficient. The `SemanticHasher` (`diff/semantic_hash.rs`) solves this by building a reverse parent map (`NodeId -> parents`). During an update, it places only the modified nodes into a queue and walks upward through their ancestors, recalculating hashes only along that specific path. This targeted recalculation updates the affected tree signatures in $O(d \log n)$ time (where $d$ is the tree depth).
</details>

***

## 10. Contact & Support

This project is actively maintained under the care of **@EngModiDev**.

I am committed to providing highly responsive support and am open to discussing any issues, optimizations, or feature ideas directly with the developer community. 

*   **Feedback & Bug Reports:** If you run into issues, find structural edge cases, or have optimization advice, please open an issue in our repository.
*   **Updates:** Run `./engineinspector version` to let the auto-updater dynamically sync with verified updates before opening a ticket.
```
