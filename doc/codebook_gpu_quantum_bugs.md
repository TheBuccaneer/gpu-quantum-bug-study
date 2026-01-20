# Codebook – GPU-Accelerated Quantum Stack Bugs

## 1. Bug Types (What)

| ID | Bug Type                           | Short Description                                                                 | Typical CTClass Tendency                                         |
|---:|------------------------------------|----------------------------------------------------------------------------------|------------------------------------------------------------------|
| 1  | Config-/Environment Bug            | Incorrect/incompatible environment: drivers, CUDA versions, missing libraries, container/cluster config. | Often C; sometimes B (e.g., if better preflight/build checks help). |
| 2  | Build-/Install-/Packaging Bug      | Errors in build scripts/installer/packaging: dependency declarations, wheels/sdists, linker issues (pip/uv/conda). | Often B; occasionally A or C (depends on the nature of the problem). |
| 3  | Backend-/Framework Integration Bug | Interaction problems between framework and backends: wrong target selection, GPU-backend init, incompatible interfaces. | Often B (A/B borderline); partly C if strongly runtime/hardware-dependent. |
| 4  | API-/Usage-/Logic Bug (High-Level) | Misuse of high-level APIs: wrong preconditions, typical logic/algorithm errors in user code. | More A or B, since many errors are catchable via types/contracts. |
| 5  | Performance-/Numerics Bug          | Performance degradation, unexpected complexity, numerical instability, precision/rounding issues. | Mostly B or C: often analysis/design or runtime questions.       |
| 6  | Other / Uncategorized              | Does not fit the above categories; used initially to keep the codebook extensible. | Depends on the specific case; keep open initially.               |




### Overview of the Bug Categories 

1. Config-Environment Bug
   → Problems caused by an incorrect or incompatible environment, drivers, CUDA versions,
   missing libraries, faulty container/cluster configs, etc.

2. Build-Install-Packaging Bug
   → Errors in the build system, during installation, or in packaging
   (e.g., incorrect or dynamic dependency declarations, issues with
   `pip`/`uv`/wheels, incorrectly linked libraries).

3. Backend-Framework Integration Bug
   → Problems in the interaction between framework and backend(s), e.g.
   wrong selection of targets/simulators, faulty initialization of
   GPU backends, inconsistencies between Python/C++ API and internal
   implementations.

4. API-Usage-Logic Bug (High-Level)
   → Misuse of high-level APIs, incorrect assumptions about semantics,
   typical logic or algorithm errors in quantum programs that do not
   primarily depend on the environment.

5. Performance-Numerics Bug
   → Bugs where the primary cause lies in performance degradation, incorrect
   complexity, or numerical problems (e.g., instabilities, overflow,
   unexpected precision loss).

6. Other - Uncategorized
   → Cases that would not fit well into the above categories.



## 1.1 Config-/Environment-Bug

**Definition:**  
A bug where the primary cause lies in the system environment, installation, or
configuration – e.g. incompatible CUDA/driver versions, missing
libraries, incorrectly chosen container images, or not enabled features
(MPI, GPU support, etc.).

**Typical indicators:**
- Error messages explicitly mention CUDA/driver/runtime versions
  (e.g. „unsatisfied condition: cuda>=…“, missing `libcublas.so.X`, etc.). 
- The problem occurs only in specific environments / images / installation paths
  (e.g. Docker image, specific Linux release).  
- Workarounds consist of „fixing the environment“ (updating drivers, using a different CUDA-
  version, choosing a different package/install tool).

**Note for coding:**  
If it is unclear whether this is a pure config/env bug or a
build/install bug, it is first checked whether the problem can be solved solely by
adjusting the environment (driver, CUDA version, package choice, flags).
If yes, it is coded as a config/environment bug.


### 1.2 Build-/Install-/Packaging-Bug

**Definition:**  
A bug where the primary cause lies in the build or installation process or in
packaging – e.g. faulty or dynamically determined dependency declarations, 
problems with package managers (`pip`, `uv`, `conda`, …),
inconsistent binary/library versions, or metapackages that do not install all
required components.

**Typical indicators:**
- The error already occurs during installation or on the first import of a package
  (e.g. missing modules/libraries, incomplete wheels/sdists).
- Different package managers or installation paths (e.g. `pip` vs. `uv`,
  `pip` vs. `conda`) lead to different or broken installations.
- Workarounds consist of “installing differently” (different package manager, explicitly
  installing missing dependencies, using an alternative channel/index), without changing
  the actual program logic.

**Distinction from 1.1 Config-/Environment bug:**  
If the problem primarily stems from the *environment* not matching the
requirements (driver too old, wrong CUDA version, missing system
library), we code it as a config/environment bug (1.1).  
If, however, the *build/install process itself* is broken or incomplete
(metapackage assembled incorrectly, dynamic dependencies, broken wheels),
we code it as a build/install/packaging bug (1.2).


### 1.3 Build-/Install-/Packaging-Bug

**Definition:**  
A bug where the primary cause lies in the interplay between the framework and
backend(s). Typically, each component works “correctly” on its own, but the
integration—i.e., how backends are selected, initialized, configured, or invoked—
fails. Examples include wrong default targets, inconsistent interfaces between the
Python and C++ layers, or errors during GPU-backend initialization.

**Typical indicators:**
- The error occurs only when a specific backend (e.g., GPU simulator,
  hardware backend, custatevec) is active, but not with another
  backend (e.g., CPU/qpp simulator).
- Workarounds consist of “choosing a different backend” or “setting an environment
  variable to force a different target,” without changing the actual
  algorithm.
- Error messages originate from the backend layer but are triggered by the
  framework (e.g., during backend selection or initialization).

**Distinction:**  
- If the cause lies in the environment or installation (driver, CUDA,
  missing libraries), the bug is coded as a config/environment or build/
  install bug (1.1 / 1.2).
- If the cause lies in pure user logic or API usage (wrong parameters, wrong
  use of gradient APIs, etc.), the bug falls into category 1.4 “API-/Usage-/Logic bug (high-level)”.
- 1.3 is used when the problem lies “in between”: the framework
  delegates to a backend, but the integration (target selection,
  initialization, interface) is faulty or insufficiently
  aligned.


### 1.4 API-/Usage-/Logic-Bug (High-Level) 

**Definition:**  
A bug where the primary cause lies in the use of high-level APIs or
in the program logic of the quantum program. This includes typical
misuse of functions (e.g. wrong return types, missing
`expval` calls), misunderstood semantics (e.g. broadcast behavior,
shape conventions of gradients), or classic logic/algorithm errors
in (hybrid) quantum code.

**Typical indicators:**
- The error can be traced back to incorrect or inappropriate use of an API
  (e.g. wrong combination of measurements, gradients, interfaces,
  or return types).
- The environment/installation is correct; the error is reproducible even in “clean”
  setups as long as the same code/workflow is used.
- Error messages explicitly point to API contracts (e.g. “Grad only applies
  to real scalar-output functions”) or to shape/type mismatches in user
  code (e.g. unexpected tensor shapes in gradients/Jacobians).

**Distinction:**  
- If the same code is semantically correct with a different backend (or on different hardware)
  but merely slower or more unstable, it may be a performance/numerics bug (1.5)
  rather than an API/usage bug.
- If the error would be recognizable as a “classic
  logic/API error” even without any quantum-specific context (wrong return type, shape error,
  wrong function used), 1.4 is usually appropriate.
- If, however, only the interplay between framework and
  backend fails (e.g. only a specific target is affected, other targets
  work), 1.3 “Backend/Framework integration bug”
  should be chosen instead.


### 1.5 Performance-/Numerik-Bug 

**Definition:**  
A bug where the primary cause lies in performance- or numerics-related
properties of the system—e.g. severe slowdowns, memory leaks,
unexpectedly high resource usage, or numerically unstable or inaccurate
results (e.g. due to floating-point limits, mixed precision, rounding errors).

**Typical indicators:**
- Metrics such as runtime, memory consumption, throughput, or scaling
  behave significantly worse than expected (e.g. regression relative to a
  previous version, exponential growth instead of linear).
- Repeated execution of code leads to continuously increasing
  memory usage (memory leak) until an out-of-memory/allocation error
  occurs.
- Results are numerically unstable or obviously wrong, although the
  logic itself appears correct (e.g. only at very small angles, at
  certain qubit counts, under mixed-precision training).

**Distinction:**
- If the main focus is on “runs at all vs. does not run,” and the
  cause is clearly installation/config, prefer 1.1 / 1.2.
- If the cause lies in API misuse or logic (wrong algorithm,
  wrong formula), prefer 1.4.
- 1.5 is used when the code basically “works,” but
  performance/resource or numerics characteristics are the problem.

### 1.6 Other - Uncategorized

**Working definition:**  
Catch-all category for bugs that do not meaningfully fit into any of the
previous bug types (1.1–1.5). This category is used primarily in the early
phase of coding to mark borderline cases and later decide whether new
categories are needed or existing ones should be extended.

**Typical indicators:**
- The issue mainly concerns topics such as debugging/logging infrastructure,
  test frameworks, very framework-specific edge cases, or exotic
  tooling problems that do not allow a clear assignment to config, build, integration,
  API/usage, or performance/numerics.
- The bug is extremely specific (e.g. only at exactly one specific
  qubit count or only in combination with a particular external tool)
  and would require its own tiny category.

**Distinction and usage note:**
- 1.6 should be used **sparingly**: only when a meaningful
  assignment to 1.1–1.5 is not possible even after careful consideration.
- Cases in 1.6 are particularly interesting for further development of the
  codebook: if similar “other” bugs accumulate, this can lead to a new
  regular category.




## 2. Stack-Layer (Where)

### 2.1 Build/Deploy/Environment 

**Definition:**  
This layer covers everything related to installation, packaging, build systems,
containers, driver setup, and the runtime environment (e.g. Docker images, installers,
Python wheels, system libraries, environment variables).

**Typical artifacts:**
- Installer scripts, Dockerfiles, container images
- Package manager configuration (pip, uv, conda, apt, ...)
- System libraries and runtime environment (CUDA runtimes, drivers, MPI)

**Questions when coding:**
- Is it primarily about how CUDA-Q/cuQuantum is installed or started?
- Does the issue revolve around missing or incompatible libraries/versions
  (e.g. specific CUDA or driver versions)?
- Does the error occur before a specific quantum kernel or a
  high-level API function is executed?

**Distinction:**  
If the actual error only occurs during execution of a specific
kernel or algorithmic logic and is not primarily triggered by setup
or installation, a different layer is usually chosen
(e.g. framework integration or high-level API).



### 2.2 Backend-Library

**Definition:**  
This category covers bugs whose primary cause lies in the **backend-close implementation**—i.e., libraries that provide the actual compute kernels, linear algebra, statevector/tensor operations, or code generation. Examples include **cuStateVec, cuTensorNet, qpp, MLIR/LLVM codegen paths** or comparable low-level backends.

**Typical indicators:**

- The error occurs **independently of the high-level API** as soon as a particular backend is used.
- The issue explicitly refers to a backend component (e.g. `custatevec`, “statevector backend”, “tensornet backend”, “MLIR lowering”).
- It involves wrong results, crashes, or exceptions that can be traced back to **numerical cores, low-level kernels, or code generation**.

**Distinction:**

- If the problem occurs only with **one specific backend** even though the high-level calls are correct → more likely **backend library** than high-level API.
- If the cause lies in wrong backend selection or initialization (e.g. wrong device, backend not loaded), that is more **2.3 framework integration**.

**Examples (schematic):**

- Incorrect simulation results only in the `custatevec` backend, while other backends produce correct values.
- Crash in a specific MLIR lowering pass with valid input.



### 2.3 Framework-Integration

**Definition:**  
This category covers bugs in the **interaction between the high-level framework (Python/C++ API)** and the underlying backends. It concerns errors in **backend selection, initialization, configuration, and invocation of backends** by the framework.

**Typical indicators:**

- The bug occurs only when a **specific backend is selected via the framework API** (e.g. `--target`, `set_backend(...)`).
- Error messages indicate **failed initialization**, wrong device selection, or mismatching options between framework and backend.
- The backend itself works correctly, but the framework passes **wrong parameters, options, or data types** to the backend.

**Distinction:**

- If the problem lies directly in the user-facing API semantics (kernel invocation, parameter checking, measurement logic), it is more likely **2.4 High-Level API / framework logic**.
- If drivers, CUDA versions, paths, or the system environment are incorrect → more likely **2.1 build/deploy/environment**.

**Examples (schematic):**

- The framework attempts to use a GPU backend even though only CPU is available, and does not handle the error robustly.
- Incorrect translation of framework options (`shots`, `qubits`, `precision`) into backend-specific flags, leading to runtime errors.


### 2.4 High-Level-API / Framework-Logic 

**Definition:**  
This category covers bugs in the **user-facing API and framework logic**—i.e., kernel semantics, parameter validation, type/shape handling, result processing, transformation and measurement logic. The cause typically lies in the **frontend** (e.g. Python API, decorators, high-level constructs).

**Typical indicators:**

- The API does not fulfill its **own contract** (according to the documentation something should work, but it does not).
- Errors in handling **types, shapes, containers, or default values** of parameters.
- Incorrect or misleading high-level results (e.g. wrong measurement indices, wrong aggregation of shots), even though the backend works correctly.
- Bugs in transformation pipelines (e.g. QNode transforms, pass pipelines) that are used directly by users.

**Distinction:**

- If the problem arises due to **incorrect use** of the API, but the API behaves correctly according to the specification, it may be a **usage/user error** (if coded separately).
- If the API is correct, but a specific backend computes incorrectly internally → more likely **2.2 backend library**.
- If the cause lies in the environment/installation → **2.1 build/deploy/environment**.

**Examples (schematic):**

- `list[list[int]]` is supported as a kernel argument according to the documentation, but leads to an internal type inference error.
- A high-level function for measurement aggregation returns a wrongly sorted or wrongly indexed result.


### 2.5 Runtime-/Framework-Runtime 

**Definition:**  
This category covers bugs that primarily appear in the **runtime phase of the framework**, typically in the area of **resource management, caching, scheduling, and performance**. This includes, for example, memory leaks, inefficient JIT caches, faulty lazy-evaluation behavior, or severe performance regressions despite otherwise correct results.

**Typical indicators:**

- The program produces **correct results**, but shows **memory leaks**, unusually high GPU/CPU utilization, or inconsistent runtime behavior.
- JIT/cache mechanisms behave incorrectly (e.g. incorrect reuse of compiled kernels, caches not being invalidated).
- Performance problems are clearly attributable to **framework runtime logic**, not to general hardware limitations.

**Distinction:**

- If the problem is primarily caused by incorrect configuration/installation (e.g. wrong BLAS/CUDA version) → **2.1 build/deploy/environment**.
- If incorrect results (not just performance/resource behavior) are affected, it may be **2.2 backend library** or **2.4 high-level API / framework logic** depending on localization.

**Examples (schematic):**

- Repeated execution of the same kernel leads to continuously increasing GPU memory usage (leak in the runtime layer).
- A JIT cache stores too many variants and is never cleaned up → strongly increasing compile time and/or RAM consumption.




## 3. Compile-Time Avoidability (CTClass A/B/C)

To classify to what extent a bug could in principle be avoided by compile-time mechanisms
(e.g. types, typestate, static analyses), we use a three-level scale CTClass A/B/C.
This classification is an informed assessment, not a proof.


### 3.1 CTClass A – directly compile-time avoidable

**Definition:**  
Bugs that could, with high probability, be prevented by comparatively “simple”
compile-time mechanisms, e.g. by stronger type systems, more precise API
contracts, or basic static checks
(null/option checks, dimension/shape checks, required parameters).

**Typical examples:**
- Wrong types or dimensions of parameters that lead to immediate
  error messages (“Grad only for scalar outputs”, shape mismatches
  in gradients).
- Clear API contract violations (e.g. “this function expects exactly one
  expectation value, not a list”), which can be expressed via stronger types or
  contracts.

**Heuristic questions:**
- Could a static type checker with sufficiently expressive types
  find this error?
- Would a static API contract (“Function f: Input → Output”) already indicate
  violated preconditions before the code runs?
- If yes, A is a plausible classification.


### 3.2 CTClass B – potentially compile-time avoidable (advanced analysis)

**Definition:**  
Bugs for which compile-time-based avoidance appears possible in principle,
but which would require advanced analyses—e.g.
resource and lifetime analysis, complex data-flow analyses, inter-
procedural analysis, domain-specific static checks.

**Typical examples:**
- Certain build/install/packaging problems where an analysis of
  dependency graphs or packaging metadata could detect inconsistencies.
- Integration errors between framework and backend where certain
  sequences of API calls or configurations could be statically recognized as
  “incompatible”.
- Performance regressions that result from clearly identifiable structural
  changes (e.g. inefficient data structures, obvious jumps in complexity).

**Heuristic questions:**
- Could a very “smart” static analysis/verification tool (with
  domain knowledge) detect this error without executing the program?
- Does this require knowledge across multiple modules, data flows, or special
  invariants (e.g. loop-unrolling limitations on hardware backends)?
- If yes, the classification tends more toward B than A.



### 3.2.1 Subtypes within CTClass B (B1/B2)

**Motivation (brief):** CTClass B groups cases that are, in principle, avoidable before execution,
but not “trivial” like A. For greater precision, we distinguish two mechanisms.

**B1 – statically/metadata-based avoidable (Compatibility/Constraints)**
Definition: Avoidability via checks over *static artifacts* (build/install metadata, version and feature matrices,
architecture/compute capability, declared limits, configuration flags).
Typical mechanisms: dependency resolver rules, compatibility audits, build-time guards, capability-matrix checks.

Heuristic questions:
- Is the required information present in metadata/constraints/support matrices (versions, arch, flags, limits)?
- Could a preflight/resolver correctly “fail fast” without executing the workload?

**B2 – contract/guard-based avoidable (Contracts/Feature-Gating/Typestate idea)**
Definition: Avoidability via *API/framework contracts*, guards, or feature gates that make unsafe states un-reachable
(e.g. forbidden combination states, result-typed error propagation, safe/unsafe modes).
Typical mechanisms: preflight validators in the API, explicit “unsafe” modes, state-machine/typestate design, Result/Option contracts.

Heuristic questions:
- Does the error arise because the framework allows an “unsafe state” at all?
- Could a contract/guard prevent the faulty combination before start or deterministically block it?

**Coding rule:** Assign a subtype only when CTClass=B; otherwise leave empty.
Optional: Confidence {high/medium/low} for uncertain reports.



### 3.3 CTClass C – not meaningfully compile-time avoidable

**Definition:**  
Bugs that primarily depend on the runtime environment, hardware, drivers, external
configuration, or highly dynamic conditions, for which compile-time-based prevention is
realistically not feasible (or only with disproportionate effort).

**Typical examples:**
- GPU/driver/CUDA version conflicts (e.g. container requires CUDA >= X,
  driver is older; a certain GPU architecture is not supported by a backend library).
- Numerical stability problems that strongly depend on concrete hardware properties,
  floating-point implementations, or mixed-precision paths.
- Memory leaks and resource problems that arise from complex runtime behavior
  (JIT caches, allocator strategies, interaction of multiple libraries).

**Heuristic questions:**
- Does the bug depend substantially on specific hardware, driver version, cluster
  configuration, or runtime load?
- Would even an extremely powerful type system be unable to know the relevant
  environmental parameters at compile time?
- If yes, C is the obvious classification.


## 4 Example Cases (Representative Issues)

This chapter provides a small set of representative labeled issues extracted from the two annotated datasets (CUDA-Q and Qiskit Aer GPU).

### 4.1 Selected Examples (cross-category)

| Dataset 	| Project | Issue | Title | BugType | StackLayer | CTClass | SubType | URL | Rationale |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| CUDA-Q | NVIDIA/cuda-quantum | #2185 | `nvidia` target: simulation errors when setting `CUDAQ_MAX_CPU_MEMORY_GB` beyond system memory capacity | Config-/Environment-Bug | Runtime-/Framework-Runtime | C |  | https://github.com/NVIDIA/cuda-quantum/issues/2185 | Documented environment variable does not behave robustly for "unlimited" or too large values and leads to runtime errors / incorrect results. |
| Aer GPU | Qiskit/qiskit-aer | #2351 | 'Simulation device "GPU" is not supported on this system ' for qiskit-aer-gpu 0.15.1, CUDA 12.7 | Config-/Environment-Bug | Build/Deploy/Environment | B | B1 static | https://github.com/Qiskit/qiskit-aer/issues/2351 | The problem is the incorrect detection of qiskit i.e. a configuration issue. |
| CUDA-Q | NVIDIA/cuda-quantum | #3433 | Can not pip install cudaq on Ubuntu 22.04 | Build-/Install-/Packaging-Bug | Build/Deploy/Environment | B | B1 | https://github.com/NVIDIA/cuda-quantum/issues/3433 | The PyPI cudaq source distributions are published with broken package metadata where the project name is reported as unknown |
| Aer GPU | Qiskit/qiskit-aer | #2231 | CMake error with CMake 3.20 | Build-/Install-/Packaging-Bug | Build/Deploy/Environment | B | B1 static | https://github.com/Qiskit/qiskit-aer/issues/2231 | The error occurs during building or installation (CMake - Conan) before any simulation is even run. |
| CUDA-Q | NVIDIA/cuda-quantum | #349 | IR produced when compiling for Quantinuum and IonQ backends is not spec compliant | Backend-/Framework-Integrations-Bug | Framework-Integration | B | B2 | https://github.com/NVIDIA/cuda-quantum/issues/349 | The issue explicitly states that the names of the QIR functions for T- and S-adjoint (__tdg, __sdg) do not comply with the QIR Base Profile and must be changed to __t__adj/__s__... |
| Aer GPU | Qiskit/qiskit-aer | #2353 | Program exit unexpectedly without error when qubits>15 | Backend-/Framework-Integrations-Bug | Framework-Integration | B | B2 contractually | https://github.com/Qiskit/qiskit-aer/issues/2353 | The error affects the integration of MPI with GPU-based Aer execution, not the algorithm itself. |
| CUDA-Q | NVIDIA/cuda-quantum | #2627 | List of list in kernel argument gives error | API-/Usage-/Logic-Bug (High-Level) | High-Level-API / Framework-Logic | A |  | https://github.com/NVIDIA/cuda-quantum/issues/2627 | When the kernel is declared with a nested Python type annotation and called with a list-of-lists argument, CUDA-Q fails at runtime with RuntimeError |
| Aer GPU | Qiskit/qiskit-aer | #2336 | Incompatibility of qiskit-aer-gpu 0.15.1 with Qiskit 2.0: qiskit.providers.convert_to_target() function removed. | API-/Usage-/Logic-Bug (High-Level) | High-Level-API / Framework-Logic | B | B1 static | https://github.com/Qiskit/qiskit-aer/issues/2336 | The import version error is caused by incompatible high-level API package versions. |
| CUDA-Q | NVIDIA/cuda-quantum | #2937 | Wrong simulation result with the tensornet/tensornet-mps target | Performance-/Numerik-Bug | Backend-Library | C |  | https://github.com/NVIDIA/cuda-quantum/issues/2937 | For this specific highly structured 20-qubit circuit, the tensornet and tensornet-mps targets return a sample distribution that only contains 8 bitstrings instead of the 16 bits... |
| Aer GPU | Qiskit/qiskit-aer | #2340 | Cacheblocking (with multiple GPUs) results in unexpected measurement samples | Performance-/Numerik-Bug | Backend-Library | B | B2 contractually | https://github.com/Qiskit/qiskit-aer/issues/2340 | The core symptom is incorrect samples due to a faulty blocking process. |
| CUDA-Q | NVIDIA/cuda-quantum | #1804 | [python] Combining `cudaq` module with `multiprocesssing` module can result in deadlock | Config-/Environment-Bug | Runtime-/Framework-Runtime | C |  | https://github.com/NVIDIA/cuda-quantum/issues/1804 | Using the default fork startup method with a multithreaded CUDA-Q process leads to deadlock. |
| Aer GPU | Qiskit/qiskit-aer | #2030 | qiskit-aer-gpu with MPI takes longer to run code on distributed nodes | Performance-/Numerik-Bug | Runtime-/Framework-Runtime | C |  | https://github.com/Qiskit/qiskit-aer/issues/2030 | Multi-node MPI becomes significantly slower, although results remain correct—a pure performance and scaling effect. |

### 4.2 Quick index (one example per main category)

- **1.1 Config/Env:** CUDA-Q #2185; Aer GPU #2351
- **1.2 Build/Install/Packaging:** CUDA-Q #3433; Aer GPU #2231
- **1.3 Backend/Framework Integration:** CUDA-Q #349; Aer GPU #2353
- **1.4 API/Usage/Logic (High-Level):** CUDA-Q #2627; Aer GPU #2336
- **1.5 Performance/Numerics:** CUDA-Q #2937; Aer GPU #2340
- **2.5 Runtime/Framework-Runtime (extra):** CUDA-Q #1804; Aer GPU #2030

## 5 Borderline Cases (Adjudication Log)

The following table documents borderline cases (especially A/B/C and B1/B2), including the conservative decision rule used.

| Issue                                                                                   | Alternative Labels (before adjudication) | Why it is borderline                                                                                                                                                                                                                                                                             | Final decision + rule (conservative)                                                                                                                                                |
| --------------------------------------------------------------------------------------- | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **CUDA-Q #920 — “MPI is not enabled in cudaq Python wheels”**                           | **B1 ↔ C**                               | Packaging advertises/ships an API surface, but the functionality is absent (stubs/feature missing in the wheel). A pure metadata/support-matrix check (B1) is brittle if the artifact exposes the API while not providing the implementation.                                                    | **C** — treat “API present but non-functional” as runtime/artefact-level failure; avoid optimistic “install-time checkable” claims.                                                 |
| **CUDA-Q #2628 — “cudaErrorIllegalAddress at mqpu”**                                    | **B2 ↔ C**                               | One could argue for stronger preflight/guards (B2) around IDs, resource selection, and safety checks. However, illegal-address faults in multi-GPU/asynchronous execution are often timing/context dependent and manifest only at runtime.                                                       | **C** — if the failure is driven by asynchronous runtime/hardware behavior rather than invalid static inputs, classify as runtime-only (conservative).                              |
| **CUDA-Q #1464 — “CNOT degradation / control semantics lost in MLIR lowering”**         | **B ↔ C** (not “simple B2”)              | A “B” argument would require semantic verification of compiler transformations (e.g., IR-level equivalence/invariants across lowering). In practice, user code is valid and the miscompile is silent; without heavyweight semantic checking, it is not realistically prevented before execution. | **C** — when prevention would require semantic equivalence checking in the compiler pipeline, adjudicate as C to avoid overstating compile-time checkability.                       |
| **CUDA-Q #2279 — “Mid-circuit measurements: missing state collapse / wrong semantics”** | **A ↔ C**                                | The program is syntactically valid; the issue is a backend/simulator semantic mismatch (wrong-results). Static typing or surface-level contracts do not capture the physical/semantic modeling choice inside the backend.                                                                        | **C** — backend semantic/wrong-results issues are treated as runtime/back-end dependent (conservative).                                                                             |
| **CUDA-Q #875 — “State preparation broadcasting / ASTBridge spec adherence”**           | **B2 ↔ C**                               | A B2 argument would be “add API/parameter checks for shape/qubit count.” But the fix indicates the bridge/lowering logic violated the specified mapping while the user followed the documented API; the failure is in internal translation semantics.                                            | **C** — if user input is spec-conform and the internal bridge/translation violates the spec, classify as deep framework/compiler logic (C), not as a preventable usage/guard issue. |

