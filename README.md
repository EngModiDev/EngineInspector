# README.md

<p align="center">
  <img src="assets/logo.png" alt="EngineInspector Logo" width="220" height="220" />
</p>

<h1 align="center">EngineInspector</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Python-v3.10+-blue.svg?logo=python&logoColor=white" alt="Python v3.10+" />
  <img src="https://img.shields.io/badge/Rust-Edition%202021-orange.svg?logo=rust&logoColor=white" alt="Rust Edition 2021" />
  <img src="https://img.shields.io/badge/Static%20Analysis-AST%20%7C%20CFG%20%7C%20SSA-red.svg" alt="Static Analysis" />
  <img src="https://img.shields.io/badge/Security-Local%20Only-green.svg" alt="Local Only" />
  <img src="https://img.shields.io/badge/Performance-Zstd%20%7C%20Blake3-blueviolet.svg" alt="Performance" />
</p>

***

## Table of Contents
*   [1. Executive Summary](#1-executive-summary)
*   [2. Download Links](#2-download-links)
*   [3. Quick Start (Usage)](#3-quick-start-usage)
*   [4. Key Features](#4-key-features)
*   [5. Deep Technical Features & Architecture](#5-deep-technical-features--architecture)
*   [6. The Power of "Semantic Diffing"](#6-the-power-of-semantic-diffing)
*   [7. First-Run & Offline Integrity](#7-first-run--offline-integrity)
*   [8. Security & Transparency (False Positives)](#8-security--transparency-false-positives)
*   [9. Advanced FAQ](#9-advanced-faq)
*   [10. Contact & Support](#10-contact--support)

***

## 1. Executive Summary
EngineInspector is a static analysis and semantic diffing suite built on a high-performance hybrid architecture combining a Python orchestrator with a Rust performance core (`engineinspector_rust` FFI). By integrating dynamic Abstract Syntax Tree (AST) parsing with parallel control flow reconstruction and dataflow analysis, the tool offers deep structural and behavioral analysis of complex codebases. Every calculation, from BLAKE3 subtree fingerprinting to Static Single Assignment (SSA) solving, runs locally to safeguard your proprietary intellectual property.

***

## 2. Download Links

Choose the pre-compiled, standalone binary matching your operating system:

*   **[📥 Download EngineInspector for Windows (engineinspector.exe)](#)**
*   **[📥 Download EngineInspector for Linux (engineinspector)](#)**

***

## 3. Quick Start (Usage)

EngineInspector compiles down to a single native executable. Linux users must grant executable permissions before running:
```bash
chmod +x engineinspector
```

### Analyzing Individual Files
Compare two individual source files (supporting both C++ and Python) to generate a structural diff report:
```bash
# Windows
engineinspector.exe compare-files path/to/before.cpp path/to/after.cpp

# Linux
./engineinspector compare-files path/to/before.py path/to/after.py
```
*   Save the detailed analysis directly to a JSON report:
    ```bash
    ./engineinspector compare-files left.cpp right.cpp --out reports/diff_report.json
    ```

### Analyzing Complete Projects (Directory Trees)
Recursively match and inspect entire code repositories or directory structures:
```bash
./engineinspector compare-paths path/to/original_project/ path/to/modified_project/ --out-dir reports/
```
*   *Note: This recursively maps files by relative paths and exports individual structural JSON reports for every added, deleted, or modified module.*

### Initializing a Workspace
Scaffold a standard workspace environment with standard configuration templates:
```bash
./engineinspector init my_workspace/
```

### Version & Environment Audit
Verify local engine signatures, active dynamic parser modes, and check for updates:
```bash
./engineinspector version
```

***

## 4. Key Features

*   **🌐 Broad Multi-Language Support:** Full semantic parsing of C++ (using combined Tree-Sitter & Clang backends) and Python (utilizing built-in tokens and syntax trees), with platform support designed for Rust, Java, JavaScript, and TypeScript.
*   **⚡ Hybrid AST Parsing (libclang + Tree-Sitter):** Merges structural parser layouts with deep preprocessor macro evaluations, template arguments, and semantic typing.
*   **📊 Parallel CFG & SSA Compiler:** Reconstructs linearized Control Flow Graphs (CFG) across basic blocks in parallel, transforming variables into SSA representations using the Lengauer-Tarjan dominator tree algorithm.
*   **🔍 Interprocedural Call Graph & Points-To Analysis:** Traces function call paths across file boundaries. Resolves indirect, virtual, and dynamic dispatch calls utilizing points-to and type-based indexes.
*   **🌳 Incremental Semantic Diffing:** Analyzes changes structurally. prunes syntactic whitespace/comment-only diffs, and isolates renames or moved blocks.
*   **⚙️ Bytecode VM Rule Engine:** Features a sandboxed, budget-limited register virtual machine that executes precompiled bytecode rules for custom security and compliance validation.
*   **🔒 Local & Secure Offline Integrity:** Performs all processing, lexical parsing, AST hashing, and flow queries locally on your system.

***

## 5. Deep Technical Features & Architecture

<details>
<summary><b>🛠️ Click to expand: Deep Technical Features &amp; Architecture</b></summary>
<br>

### A. Parser Merging & Comment-Safe Normalization
*   **Hybrid Abstract Syntax Tree (AST) Parser:** Implemented in `ast_parser.py`, the `ASTParser` merges structural and semantic representations. It combines Tree-Sitter structure with the type-attribution of `libclang` cursors via `_merge_cpp_units`. Tree-Sitter builds the AST hierarchy, while `libclang` parses templates, preprocessor macros, and compiler-level semantic types to construct a comprehensive `ASTNodeRef`.
*   **Zero-Shift Comment Stripping:** Standard comment removal shifts code offsets, which breaks AST line mapping. Our `Normalizer` class (`normalizer.py`) uses a zero-shift replacement masking technique: it queries Tree-Sitter for comment ranges, then overwrites non-newline comment bytes with spaces (`32` in UTF-8 bytes) while keeping newlines (`10`) intact. This preserves the absolute line map for accurate diff referencing. Python streams are processed via the standard `tokenize` module.
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
*   **readability (Readability) Estimation:** Uses a heuristic based on physical line lengths, average line length, and token density per line to estimate code readability on a scale of 0 to 1.
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
*   **Sandboxed VM & custom Rule DSL:** The system includes a custom rule DSL parser (`rules/language.rs`) that compiles declarative assertions into a compact bytecode format (`OpCode`). This bytecode is executed inside a sandboxed register-based interpreter (`Vm` in `rules/engine.rs`). The VM prevents infinite loops and denial-of-service attempts by enforcing strict limits on instruction counts and execution times.

### E. Performance & Infrastructure
*   **Postcard Serialization & CRC64fast Checksums:** Located in `cfg/compress.rs`, this module serializes SSA and CFG representations into a version-controlled binary format (`ENGCFGv1`). It uses postcard serialization, validates data integrity with SIMD-accelerated `crc64fast` checksums, and applies optional `zstd` compression to reduce storage requirements.
*   **Atomic Replacement Auto-Updater:** The `EngineUpdater` (`core/updater.py`) queries a secure remote update manifest. It downloads new binaries, verifies their SHA-256 signatures, and performs atomic executable swaps on Windows by renaming active files (`active_exe.rename(backup_exe)`). It also includes fallback rollback handlers to restore previous versions if a crash occurs on first boot.
*   **Crash Boundary & State Harvest:** Uncaught runtime exceptions are captured by the boundary layer (`cli/boundary.py`). On failure, the system polls host diagnostics and active memory RSS parameters (`psutil`), generating a localized crash report (`crash_dump_*.json`) for debugging.
</details>

***

## 6. The Power of "Semantic Diffing"

<details>
<summary><b>💡 Click to expand: Git Diff vs. Semantic Diffing</b></summary>
<br>

Traditional line-by-line diffing utilities (such as `git diff`) operate purely on text sequences. Consequently, simple modifications like reformatting code, renaming a variable, or moving a block trigger large, misleading diff outputs. 

EngineInspector approaches diffing structurally using an abstract syntax tree parser and semantic matching:

```
    Source A (Before)                    Source B (After)
┌───────────────────────┐            ┌───────────────────────┐
│ void search() {       │            │ void search() {       │
│   int max_depth = 64; │            │   // Indent & rename  │
│   run(max_depth);     │            │   int limit = 64;     │
│ }                     │            │   run(limit);         │
│                       │            │ }                     │
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

## 7. First-Run & Offline Integrity

To keep the binary size minimal, EngineInspector does not bundle heavy platform-specific shared libraries (`libclang`). Instead, the tool uses an automated bootstrap process on its first run:

```
[ First Launch ] ──► [ Check Local 'libs/' ] ──► (Exists & Functional?)
                             │                              │
                             │ No                           ├──► Yes ──► [ Execute Offline ]
                             ▼                              │
                 [ Fetch Remote Hash ]                      │
                             │                              │
                             ▼                              │
                 [ Download main.zip ]                      │
                             │                              │
                             ▼                              │
                 [ Verify SHA-256 Signatures ]              │
                             │                              │
                             ▼                              │
                 [ Extract & Link libclang ] ───────────────┘
```

1.  **Platform Detection:** The `BinaryBootstrapper` (`core/bootstrap.py`) detects the host operating system and CPU architecture (e.g., Windows x86_64, Linux, or Darwin arm64) to find the correct library variant in the `BINARY_MANIFEST`.
2.  **Verification:** The bootstrapper queries the companion `.sha256` hash in our repository. It downloads the binary archive into a secure temporary file and validates its SHA-256 signature against the remote manifest before extracting it.
3.  **Extraction & Safe Placement:** Once validated, the library is extracted into the local `libs/` directory. On Windows, the bootstrapper extracts the contents and automatically cleans up nested directories.
4.  **Dynamic Shared Library Injection:** The `SharedLibraryInjector` (`core/linker.py`) loads the library. On Windows, it registers the path via `os.add_dll_directory` (with a fallback to prepend the path to the environment variable). On Unix and Darwin platforms, it maps the library into memory via `ctypes.CDLL(..., mode=ctypes.RTLD_GLOBAL)`. Once loaded, the engine is fully functional and operates entirely offline.

***

## 8. Security & Transparency (False Positives)

<details>
<summary><b>⚠️ Installation Notes &amp; Windows SmartScreen</b></summary>
<br>

As EngineInspector is an independently developed, closed-source utility, you may encounter security warnings when running the standalone executable for the first time.

### Why does this happen?
*   **Unsigned Binaries:** Standalone executable packages lack expensive enterprise code-signing certificates, which triggers precautionary warnings from protective filters like Microsoft Defender SmartScreen or Linux security layers.
*   **Low-Level Binaries:** The engine bundles and dynamically maps native library components (such as `libclang` and tree-sitter wrappers) into memory to maximize parsing performance. Security software often flags this low-level interaction as suspicious.

### How to proceed safely:
1.  **On Windows:** When the SmartScreen dialog appears, click **"More info"**, then select **"Run anyway"**.
2.  **Verify Hashes:** If your environment requires strict validation, you can inspect the SHA-256 signature of the binary against our official release manifests.
3.  **No Telemetry:** You can monitor your network interface; once the initial first-run `libclang` bootstrap is finished, EngineInspector will never open an external socket. Your intellectual property and code never leave your machine.
</details>

***

## 9. Advanced FAQ

<details>
<summary><b>💬 Click to expand: Advanced FAQ</b></summary>
<br>

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

EngineInspector is developed and maintained by **@EngModiDev**.

*   For enterprise support, customized parser adapters, or to report structural bugs, please open an issue in our official project repository.
*   Run `./engineinspector version` to verify your local installation signatures before submitting technical support tickets.
