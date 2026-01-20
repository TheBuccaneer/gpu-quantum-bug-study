# Empirical Study: Bugs in GPU-Accelerated Quantum Stacks

## 1. Research Questions

RQ1: What types of bugs occur in GPU-accelerated quantum stacks
(e.g. cuQuantum, CUDA-Q, PennyLane Lightning GPU)?

Motivation: We want to create a clear bug profile for GPU-based quantum stacks
and later compare it with existing studies on Qiskit, Cirq, etc.
This allows us to see whether GPU stacks exhibit different or similar bug patterns.


RQ2: In which layers of the stack (backend library, framework integration,
high-level API, build/deploy/environment) do these bugs occur?

Motivation: The different layers of the stack (low-level library,
framework integration, high-level API, build/deploy/environment) have very
different responsibilities. We want to understand where problems
concentrate in order to apply targeted improvements and tools.


RQ3: What proportion of these bugs is, in principle, avoidable at compile time
(e.g. through stronger types, typestate, or static analyses)?

Motivation: This question directly links the empirical analysis with
compile-time safety (e.g. types, typestate, static analyses). We want to
estimate what potential there is to catch bugs before execution at all.
In addition, we will categorize all bugs into three CTClass categories:
A (directly compile-time avoidable), B (potentially avoidable only with advanced static
analysis), and C (not compile-time avoidable).



## 2. Systems / Projects

In this study, we consider three central systems from the area of
GPU-accelerated quantum stacks:

### System 1 – cuQuantum / cuStateVec (NVIDIA)

cuQuantum (in particular cuStateVec and cuTensorNet) is a low-level GPU
library for simulating quantum states and circuits. It primarily provides
C and Python APIs that can be integrated into various frameworks.

For our study, cuQuantum is relevant as an example of a
backend library in which bugs typically affect GPU-close functionality,
e.g. memory management, CUDA kernel calls, performance and
numerics issues, or special hardware configurations.

### System 2 – CUDA-Q (cuda-quantum)

CUDA-Q (formerly cuda-quantum) is a framework for quantum programming with
C++ and Python frontends and multiple backends (including CPU simulators and
GPU-based backends such as cuStateVec). It combines high-level programming
with execution on different targets.

For our study, CUDA-Q is an example of a framework at the
framework-integration layer. Relevant bugs here include, among others, the
connection to backends, API contracts, types and parameters, build and
compile problems, as well as the interaction between host code and quantum
kernels.


## 3. Timeframe & Sample Size

In this study, we consider bugs from a clearly delimited time period and
use a fixed sample size to keep the analysis manageable and
replicable.

### Timeframe

Planned observation period:

- Bugs that were created in the period **01.01.2023 – 19.11.2025**.
- Focus on modern versions of cuQuantum, CUDA-Q, and Lightning GPU, in which
  GPU support is already established.
- The exact cutoff date (date of the data pull) will be documented in the methods section
  in case it deviates slightly from the period stated here.

### Sample Size

The goal is a sample of a total of about **100–120 bugs** that are classified as
“GPU-relevant” (see the definitions section).

Rough target distribution across the three systems:

- about 60–70% of bugs from Cuda-Q
- about 30% of bugs from Qskit

This distribution can be adjusted during actual data collection
(e.g. if individual repositories contain significantly fewer suitable bugs in the
period), but the **overall size** of the sample (≈ 100–120 bugs)
should be maintained.


## 4. Out of Scope

In this study, we deliberately consider only a subset of the overall
ecosystem. The following aspects are explicitly **not** part of the scope:

1. Classic CPU-only simulators
   - Bugs that exclusively concern pure CPU simulators or backends without
     GPU relevance are not considered.
   - Example: Bugs in purely CPU-based PennyLane or Qiskit backends
     without using cuQuantum or comparable GPU libraries.

2. Pure feature requests and design discussions
   - Issues in which no malfunction is discussed, but only new features, API
     extensions, or general design proposals, do not count as bugs in the sense of this study.

3. Documentation and writing errors
   - Issues that exclusively concern documentation errors, typos, or missing
     examples in the documentation are excluded, provided that no
     actual runtime malfunction is described.

4. Hardware/driver problems without a clear software reference
   - Cases in which problems are attributed exclusively to specific hardware defects
     or external driver/system configurations and no meaningful reference to the libraries/frameworks
     studied can be established are not included in the sample.



## 5. Definitions

In this section, we define key terms that are important for selecting and
later classifying the bugs.

### Definition: Bug

For this study, we understand a “bug” to be an issue or report
in which at least one of the following points is met:

- Incorrect behavior, a crash, or an error message is described in
  connection with one of the systems considered, or
- a concrete fix, workaround, or patch is discussed that addresses a
  malfunction.

Pure improvement suggestions, design ideas, or documentation requests do
not count as a bug (see out of scope).

### Definition: GPU-relevant bug

A bug is classified as “GPU-relevant” if at least one of the following
conditions is met:

- The issue explicitly mentions GPU/CUDA terms (e.g. “GPU”, “CUDA”, “device”,
  “kernel”, “stream”, “GPU backend”), or
- the problem occurs only when using a GPU backend or a
  GPU-specific device, or
- the bug is clearly tied to the configuration or availability of GPU resources
  (e.g. CUDA version, driver, GPU memory).

### Definition: compile-time avoidable bug (CTClass A/B/C – working version)

We use a rough three-way split to classify the potential for compile-time
avoidability:

- **CTClass A – directly compile-time avoidable**  
  Bugs that can, with high probability, be prevented by stronger types,
  typestate, or simple static checks
  with high probability could be prevented we
