<p align="center">
  <img src="input_file_0.png" alt="EngineInspector Logo" width="220" height="220" />
</p>

# EngineInspector

<p align="center">
  <img src="https://img.shields.io/badge/Python-v3.10+-blue.svg?logo=python&logoColor=white" alt="Python v3.10+" />
  <img src="https://img.shields.io/badge/Rust-Edition%202021-orange.svg?logo=rust&logoColor=white" alt="Rust Edition 2021" />
  <img src="https://img.shields.io/badge/Static%20Analysis-AST%20%7C%20CFG%20%7C%20SSA-red.svg" alt="Static Analysis" />
  <img src="https://img.shields.io/badge/Security-Local%20Only-green.svg" alt="Local Only" />
  <img src="https://img.shields.io/badge/Performance-Zstd%20%7C%20Blake3-blueviolet.svg" alt="Performance" />
</p>

***

## 1. Executive Summary
EngineInspector is a static analysis and semantic diffing suite designed to evaluate structural, dataflow, and behavioral changes in engine-like codebases. The project implements a hybrid architecture: a highly ergonomic Python orchestration layer handles OS-level boundary isolation, while a high-performance Rust core (`engineinspector_rust`) processes computationally intensive tasks over PyO3 FFI. By transforming source code into compact, memory-mapped syntax arenas and constructing Static Single Assignment (SSA) Control Flow Graphs, EngineInspector runs deep dataflow queries and custom VM rules completely locally to protect your proprietary intellectual property.

***

## 2. Deep Technical Features (The Core)

EngineInspector is structured into discrete, highly decoupled subsystems. The system architecture is mapped out below:

```
                  ┌──────────────────────────────────────────────┐
                  │                 Python CLI                   │
                  │             (main.py / CLI commands)         │
                  └──────────────────────┬───────────────────────┘
                                         │
                  ┌──────────────────────▼───────────────────────┐
                  │            Core Diff Engine Orchestrator      │
                  │                 (diff_engine.py)             │
                  └──────────┬───────────────────────┬───────────┘
                             │                       │
      ┌──────────────────────▼───────┐       ┌───────▼──────────────────────┐
      │      File Loader / Decoder   │       │   Lexical & AST Normalizer   │
      │        (file_loader.py)      │       │       (normalizer.py)        │
      └──────────────┬───────────────┘       └───────┬──────────────────────┘
                     │                               │
                     └───────────────┬───────────────┘
                                     │ (Normalized Content)
                                     ▼
                  ┌──────────────────────────────────────────────┐
                  │             Parser Interface Layer           │
                  │               (ast_parser.py)                │
                  └──────────┬───────────────────────┬───────────┘
                             │ (Tree-Sitter Mode)    │ (Clang Mode)
                             ▼                       ▼
                  ┌────────────────────┐   ┌────────────────────┐
                  │ Tree-Sitter AST    │   │  libclang Cursors  │
                  └──────────┬─────────┘   └─────────┬──────────┘
                             │                       │
                             └───────────┬───────────┘
                                         │ (Hybrid Merger)
                                         ▼
                  ┌──────────────────────────────────────────────┐
                  │          Opaque Handles via PyO3 FFI         │
                  │                (ffi/mod.rs)                  │
                  └──────────────────────┬───────────────────────┘
                                         │
  ┌──────────────────────────────────────┴──────────────────────────────────────┐
  │ Rust Core Backend (algorithms/rust/src)                                     │
  │                                                                             │
  │  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐  │
  │  │      AstArena        │  │     CfgBuilder       │  │      SsaBuilder      │  │
  │  │   (ast/arena.rs)     │  │    (cfg/builder.rs)  │  │     (dfa/ssa.rs)     │  │
  │  └──────────┬───────────┘  └──────────┬───────────┘  └──────────┬───────────┘  │
  │             │                         │                         │           │
  │             ▼                         ▼                         ▼           │
  │  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐  │
  │  │    Semantic Hasher   │  │    SymbolIndex       │  │     SparseSolver     │  │
  │  │ (diff/semantic_hash) │  │  (callgraph/index)   │  │  (dfa/sparse_solver) │  │
  │  └──────────┬───────────┘  └──────────┬───────────┘  └──────────┬───────────┘  │
  │             │                         │                                     │
  │             ▼                         ▼                                     │
  │  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐  │
  │  │   Incremental Diff   │  │  CallGraphExtractor  │  │     VM Rule VM       │  │
  │  │ (diff/incremental.rs)│  │ (callgraph/extractor)│  │   (rules/engine.rs)  │  │
  │  └──────────────────────┘  └──────────────────────┘  └──────────────────────┘  │
  └─────────────────────────────────────────────────────────────────────────────┘
```

### A. Core Parsing & Normalization
*   **Hybrid Abstract Syntax Tree (AST) Parser:** Located in `ast_parser.py`, the `ASTParser` merges structural and semantic representations. It combines the speed of Tree-Sitter with the deep semantic type-attribution of `libclang` cursors via `_merge_cpp_units`. Tree-Sitter generates a structural node hierarchy, while `libclang` extracts template parameters, C-style macros, and semantic compiler types to build an annotated `ASTNodeRef`.
*   **Zero-Shift Comment Normalization:** Traditional comment removal shifts character offsets, breaking AST line mapping. Our `Normalizer` class (`normalizer.py`) utilizes a zero-shift replacement masking technique: it queries Tree-Sitter to find comment blocks, then overwrites non-newline comment bytes with spaces (`32` in UTF-8 bytes) while keeping newlines (`10`) intact. This preserves the absolute line map for accurate diff referencing. For Python fallbacks, it leverages standard lexical `tokenize` streams.
*   **Indentation & Line Mapping:** The `Normalizer` standardizes indentation by expanding tabs to four spaces, tracking block scope shifts with an indentation stack, and constructing a stable `line_map` from the raw file to the normalized content.

### B. Dataflow & Analysis (Rust Core)
*   **High-Locality AST Arena (`AstArena`):** To avoid pointer chasing and memory fragmentation, the Rust core (`ast/arena.rs`) stores tree nodes contiguously as `NodeRecord` entries within a packed vector (`PackedVec` in `utils/memlayout.rs`). References between nodes are stored as lightweight `NodeId(u32)` indices. Out-of-line metadata (`NodeMeta`) keeps the memory footprint of individual nodes small, and the arena supports O(1) lookups. A `StringInterner` maps identifiers to `u32` keys to speed up comparisons.
*   **AstArena Compaction & Remapping:** The `compact_and_remap` method removes logically deleted nodes (marked via a tombstone vector) from the arena. It then reconstructs the contiguous nodes vector and maps remaining parent-child index references to their new positions.
*   **Parallel Control Flow Graph (CFG) Compiler:** The `CfgBuilder` (`cfg/builder.rs`) compiles AST nodes into basic blocks (`Block`) containing linearized lists of instructions (`Instr`). It utilizes Tokio tasks to generate CFG structures for independent functions in parallel, merging them into a sharded global `Cfg` container with minimal locking.
*   **Lengauer-Tarjan SSA Transformation:** The `SsaBuilder` (`dfa/ssa.rs`) translates compiled CFG blocks into Static Single Assignment (SSA) form. It computes immediate dominator trees (`idom`) via a deterministic implementation of the Lengauer-Tarjan algorithm, identifies dominance frontiers, and places $\phi$-nodes at iterated dominance frontiers.
*   **Sparse, Demand-Driven Dataflow Solver:** The `SparseSolver` (`dfa/sparse_solver.rs`) executes forward and backward dataflow queries over the SSA form. It avoids global iteration by propagating lattice values (such as the `ConstLattice` for constant propagation) only along use-def chains. It also features incremental invalidation via `invalidate_instrs` to re-run queries only on affected blocks.
*   **Interprocedural Call Graph Extractor:** The `CallGraphExtractor` (`callgraph/extractor.rs`) runs parallel analyses over function CFGs to resolve call targets. Direct calls are mapped straight to symbol IDs, while indirect calls are resolved using points-to analysis on variables in the global `SymbolIndex` (`callgraph/index.rs`).

### C. Advanced Metrics (Heuristics & Statistics)
Calculated in the Python analysis layer (`analysis/metrics.py`), the `Metrics` class provides structural and statistical insights into code modifications:
*   **Shannon Entropy over Token Frequency:** Measures information density within a file. It tokenizes the source with `_TOKEN_RE`, calculates individual token probability distributions, and evaluates Shannon entropy utilizing `math.log2(p)`, normalized against the total unique token count.
*   **Heuristic Cyclomatic Complexity:** Approximates structural branching complexity by counting occurrences of conditional, looping, and branching operators via precompiled regex structures (`_BRANCH_RE`), establishing a baseline score.
*   ** रीडबिलिटी (Readability) Estimation:** Uses a heuristic based on physical line lengths, average line length, and token density per line to estimate code readability on a scale of 0 to 1.
*   **Squashed Churn Scoring:** Combines normalized line deltas, token count changes, and cyclomatic complexity differentials into a single metric. The final score is normalized using a tanh-like mapping (`math.tanh(raw * 1.5)`) to produce a value between 0 and 1.

### D. Security, Risk & Behavior
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

### E. Infrastructure & Performance
*   **Contiguous Memory Layouts:** Built-in packed arrays (`PackedVec`) align records sequentially in memory. This layout maximizes CPU L1/L2 cache hit rates, allowing the semantic diffing engine to process thousands of nodes with minimal latency.
*   **Postcard Serialization & CRC64fast Checksums:** Located in `cfg/compress.rs`, this module serializes SSA and CFG representations into a version-controlled binary format (`ENGCFGv1`). It uses postcard serialization, validates data integrity with SIMD-accelerated `crc64fast` checksums, and applies optional `zstd` compression to reduce storage requirements.
*   **Crash Boundaries and Telemetry Harvest:** The exception boundary (`cli/boundary.py`) intercepts unhandled execution failures. It collects diagnostic system state (operating system, CPU architecture, active environment variables, and process memory rss metrics) and saves this data in a structured crash dump file (`crash_dump_*.json`) to assist with debugging.

***

## 3. The Power of "Semantic Diffing"

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

***

## 4. First-Run & Offline Integrity

EngineInspector is designed to run entirely locally, meaning it does not send your source code to external servers. To make deployment easy, the executable does not bundle heavy platform-specific shared libraries (`libclang`).

Instead, the tool uses an automated bootstrap process on its first run:

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

## 5. Security & Transparency

EngineInspector accesses low-level system APIs to ensure high-performance parsing. Because of these actions, security suites may sometimes flag the executable:

### Why security software might trigger alerts:
*   **Dynamic DLL Injection & Library Mapping:** To link `libclang` on the fly, the Python layer calls low-level FFI loaders (`ctypes.CDLL` and `os.add_dll_directory`). These APIs modify process search paths and map external binary code directly into the running process. Security software often flags this behavior because malware can use similar path mutations for DLL sideloading.
*   **Unsigned Binaries:** Standalone executables compiled for distribution do not contain expensive enterprise-level code signing certificates. As a result, Windows SmartScreen may display precautionary "Unknown Publisher" warnings on launch.
*   **Local Cryptographic Hashing:** The system runs local cryptographic hashing (BLAKE3, SHA-256, and CRC64fast) to verify downloaded libraries and index your AST subtrees. Heuristic analyzers in antivirus programs sometimes flag compiled hashing routines as suspicious.

### Why EngineInspector is completely safe:
*   **100% Local Processing:** The engine does not transmit your source code, file paths, or analysis results to any external server. 
*   **Sandboxed Environment:** The custom Rule VM enforces strict runtime boundaries, ensuring that executed analysis rules cannot access system resources or run arbitrary code.
*   **Verifiable Integrity:** The dynamic bootstrapper validates every downloaded file against our cryptographically signed SHA-256 manifests, protecting your system against man-in-the-middle attacks.

***

## 6. FAQ (Advanced)

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

***

## 7. Quick Start (Usage)

### Installation
EngineInspector is distributed as a pre-compiled, self-contained binary. Simply download the variant for your platform, place it in your execution path, and run the first-time setup with an active internet connection:

```bash
# Windows
engineinspector.exe version

# Linux / macOS
chmod +x engineinspector
./engineinspector version
```

### Basic Commands

#### Compare Two Files
Compare two individual source files and print a structural analysis report to the terminal:
```bash
./engineinspector compare-files path/to/before.cpp path/to/after.cpp
```
Write the diff report directly to a JSON file:
```bash
./engineinspector compare-files path/to/before.py path/to/after.py --out reports/diff.json
```

#### Compare Two Directories
Analyze and compare two entire directory structures recursively:
```bash
./engineinspector compare-paths path/to/original_v1/ path/to/modified_v2/ --out-dir reports/
```

#### Initialize a Workspace Project
Scaffold a standard workspace structure with template configurations:
```bash
./engineinspector init my_project_workspace/
```

***

## 8. Contact & Support

EngineInspector is developed and maintained by **@EngModiDev**.

*   For enterprise support, customized parser adapters, or to report structural bugs, please open an issue in our official project repository.
*   Run `./engineinspector version` to verify your local installation signatures before submitting technical support tickets.
