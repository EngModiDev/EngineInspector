# EngineInspector Security, Privacy & License Agreement (EI-FIP-1.0)



This document establishes a binding legal commitment and licensing agreement between you (the "Licensee," whether an individual developer or an entity) and the maintainers of **EngineInspector** ("we," "us," or "the developers"). 



Please read this document carefully before installing or executing the software. By downloading, compiling, or executing EngineInspector, you agree to be bound by the security terms and licensing conditions defined herein.



***



## Part 1: The Absolute Offline & Privacy Guarantee



EngineInspector is engineered to operate as a completely self-contained, air-gapped static analysis tool. We understand that your source code represents your most valuable intellectual property. Therefore, we provide this unconditional, legally binding guarantee regarding your data privacy:



### 1. Zero Outbound Telemetry

Once the initial one-time dependency bootstrap process is complete, EngineInspector **NEVER**:

*   Transmits, relays, uploads, or caches any portion of your source code, syntactic trees, variable names, or structural hashes to any remote server or third-party service.

*   Collects or transmits telemetry, anonymous usage statistics, tracking metrics, execution times, metadata, or diagnostic indicators of any kind.

*   Initiates background network connections, with the sole exception of the secure, cryptographically validated auto-updater, which runs **only** when you explicitly execute the `version` command in an internet-enabled terminal.



### 2. Transitive Dependency & Zero-Leak Commitment

We recognize that modern software relies on third-party libraries (such as Tree-Sitter grammars and Clang wrappers). We guarantee that:

*   We do not bundle any obfuscated, hidden, or untrusted network-enabled libraries.

*   No transitive dependency within EngineInspector has the capability or configuration to establish outbound network requests during execution.

*   If this software is ever found to transmit even a single byte of your source code or analysis metadata to an external network—whether by design, unintentional error, or transitive dependency behavior—such an occurrence will be treated as a critical security defect and a breach of this agreement.



### 3. Permission to Audit

We actively encourage, authorize, and permit individuals and organizations to independently verify this pledge. You are fully authorized to:

*   Perform active network traffic analysis (using packet analyzers such as **Wireshark**, **tcpdump**, or **sysdig**) while running EngineInspector.

*   Monitor system-level calls and process bounds (using **strace**, **dtrace**, **Process Monitor**, or **eBPF** tracing) to confirm that the execution is entirely local and restricted to your local file descriptor maps.

*   Configure aggressive firewall policies to block all outbound traffic for the `engineinspector` binary, verifying that the analyzer performs identically with zero degradation of performance or capabilities.



***



## Part 2: First-Run Bootstrap & Dependency Integrity



Because EngineInspector utilizes dynamic shared libraries (`libclang`) for hybrid semantic parsing, it requires these external libraries to run. To ensure absolute safety during their acquisition, the following protocol is enforced:



1.  **Public Library Repository:** On the very first run, if the `libs/` directory is missing, the integrated `BinaryBootstrapper` fetches the pre-compiled platform libraries directly from our dedicated, publicly visible GitHub repository: [EngineInspector-libs](https://github.com/EngModiDev/EngineInspector-libs).

2.  **Auditability:** Every library, file, and helper script in the `EngineInspector-libs` repository is public. You can manually inspect, verify, and audit every asset before it is placed on your machine.

3.  **Manual Offline Ingestion:** If you prefer to keep your machine completely air-gapped from the first launch, you may download the required libraries manually from the repository, unpack them into a folder named `libs` placed directly adjacent to the EngineInspector binary, and disable network access entirely. The engine will detect the directory and execute without any internet initialization.



***



## Part 3: License Agreement (EI-FIP-1.0)



EngineInspector is released under a proprietary custom license designed to encourage developer adoption while protecting our specialized algorithmic design from commercial exploitation.



### 1. Permitted Uses (The "Free" Tier)

*   **Individual & Personal Use:** EngineInspector is **100% free** for individual software developers, hobbyists, security researchers, and academic researchers for personal, non-commercial learning, and study.

*   **Internal Corporate Evaluation:** Organizations and companies are permitted to run, evaluate, and utilize EngineInspector internally on their proprietary codebases free of charge, provided such usage complies with the restrictions below.



### 2. Restricted Uses (The "Proprietary" Boundaries)

*   **No Commercial Reselling:** You are strictly prohibited from reselling, sub-licensing, renting, or leasing EngineInspector (or its pre-compiled binaries) to any third party.

*   **No Product Bundling or White-Labeling:** You may not bundle, package, embed, or white-label the EngineInspector binary, its Rust FFI, or its analysis CLI within any commercial software product, proprietary IDE extension, or third-party continuous integration (CI) service without written authorization from the copyright holder.

*   **No Hosted Analysis Services (No SaaS):** You may not offer EngineInspector's parsing, static analysis, or semantic diffing capabilities as a managed service, hosted API, or software-as-a-service (SaaS) utility.



***



## Part 4: Open Community Feedback & Accountability



EngineInspector is developed with a deep commitment to technical excellence and professional accountability. 



*   **Constructive Critique Welcomed:** While the core AST, CFG, and SSA engines remain closed-source to protect our proprietary matching math, we operate with maximum transparency. We welcome any security feedback, architectural critique, performance warnings, or suggestions.

*   **Fast-Response Maintenance:** We are committed to maintaining a highly stable and secure static analyzer. If you discover a parsing bug, a regression, or a security concern, please file a detailed Issue in our GitHub repository. We actively monitor these reports and aim to provide transparent, high-priority responses and resolutions.



***



*Copyright (c) 2026 @EngModiDev. All rights reserved.*

```