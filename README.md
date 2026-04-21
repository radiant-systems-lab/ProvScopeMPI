# ProvScopeMPI

ProvScopeMPI is a tool for **recording, replaying, and analyzing non-determinism** in MPI applications. It intercepts MPI communication calls to capture execution traces, then replays those traces with a fixed communication order — allowing you to separate *communication-order non-determinism* from *input-based non-determinism* and pinpoint exactly where two execution runs diverge in their control flow.

---

## How It Works

ProvScopeMPI consists of two cooperating components:

1. **LLVM Instrumentation Pass** (`skeleton/`) — A compile-time pass that inserts a `printBBname()` call at every basic block entry. This produces a fine-grained execution trace (`.record<rank>.tr`) and a loop hierarchy graph (`loops.dot`) during runtime.

2. **MPI Interception Libraries** (`share/`) — Shared libraries that wrap MPI calls via PMPI. Three modes are available:
   - **`libmpirecord.so`** — Records every MPI call (source, destination, tag, request pointer, execution context) to `.record<rank>.txt`.
   - **`libmpireproduce.so`** — Replays execution using the recorded message order. At each MPI call, it validates the current execution context against the recording and reports divergences.
   - **`libmpiprovscope.so`** — Offline analysis variant.

### Record → Replay Workflow

```
Original Run                          Replay Run
─────────────────────────────────     ─────────────────────────────────────
instrumented_app + libmpirecord       instrumented_app + libmpireproduce
        │                                       │
        ▼                                       ▼
  mpirun -n N ./app [inputs]            mpirun -n N ./app [new_inputs]
        │                                       │
        ▼                                       ▼
  .record<rank>.txt  (messages)         Loads .record<rank>.txt
  .record<rank>.tr   (BB trace)         Validates execution context
  callLocations-<rank>.json             Reports divergence points
  loops.dot
```

---

## Dependencies

| Dependency | Purpose |
|---|---|
| OpenMPI or MPICH | MPI runtime |
| LLVM 11 (`llvm-11-dev`, `clang-11`) | Instrumentation pass |
| `wllvm` / `extract-bc` | Bitcode extraction |
| Graphviz C libraries (`libgraphviz-dev`) | Loop hierarchy parsing (`loops.dot`) |
| CMake ≥ 3.10 | Building the skeleton pass |
| C++20-capable compiler | Building the share libraries |

Install on Ubuntu 20.04+:
```bash
sudo apt-get install clang-11 llvm-11-dev libgraphviz-dev cmake
pip install wllvm
# Symlink if needed:
sudo ln -s /usr/bin/clang-11 /usr/bin/clang
```

---

## Building

### 1. LLVM Instrumentation Pass

```bash
cd skeleton
mkdir build && cd build
cmake ..
make
# Output: build/libSkeletonPass.so
```

### 2. MPI Interception Libraries

```bash
cd share
make
# Output: libmpirecord.so, libmpireproduce.so, libmpiprovscope.so
```

---

## Usage

### Step 1 — Instrument Your Application

```bash
# Compile with wllvm to preserve bitcode
CC=wllvm mpicc -o app app.c

# Extract bitcode
extract-bc app -o app.bc

# Run the skeleton pass to insert BB probes
opt -load skeleton/build/libSkeletonPass.so --bbprinter -o app_inst.bc < app.bc

# Link the instrumented bitcode back into an executable
mpicc -o app_record app_inst.bc share/record.cpp -lm -L./share -lmpirecord -I./share
mpicc -o app_replay  app_inst.bc share/record.cpp -lm -L./share -lmpireproduce -I./share
```

### Step 2 — Record the Original Run

```bash
mpirun -n 4 ./app_record [original_inputs]
```

Per-rank output files:
- `.record<rank>.txt` — pipe-delimited MPI call log
- `.record<rank>.tr` — basic block execution trace
- `callLocations-<rank>.json` — MPI call locations mapped to BB node counts
- `loops.dot` — loop hierarchy (shared across ranks)

### Step 3 — Replay and Detect Divergences

```bash
mpirun -n 4 ./app_replay [same_or_different_inputs]
```

During replay, `libmpireproduce.so` enforces the recorded message order and reports any divergence in execution context — i.e., where the control flow between the two runs differs.

---

## Supported MPI Calls

| Category | Calls |
|---|---|
| Blocking point-to-point | `MPI_Send`, `MPI_Recv`, `MPI_Probe` |
| Non-blocking | `MPI_Isend`, `MPI_Irecv`, `MPI_Irsend` |
| Completion | `MPI_Wait`, `MPI_Waitany`, `MPI_Waitall` |
| Polling | `MPI_Test`, `MPI_Testall`, `MPI_Testsome`, `MPI_Iprobe` |
| Request management | `MPI_Cancel`, `MPI_Request_free` |
| Persistent | `MPI_Send_init`, `MPI_Recv_init`, `MPI_Startall` |

`MPI_ANY_SOURCE` is handled specially: during replay a lookahead algorithm determines the actual sending rank from the recorded trace. Collective operations (`MPI_Bcast`, `MPI_Allreduce`, etc.) are not yet supported.

---

## Benchmark Programs

### `exs/sample/` — Minimal Non-determinism Demo
Three-rank program where rank 0 issues `MPI_Recv(..., MPI_ANY_SOURCE, ...)` and receives from either rank 1 or rank 2 in unpredictable order. Demonstrates both communication-order and input-based non-determinism.

```bash
cd exs/sample
make original    # Record with original inputs
make reproduced  # Replay with changed inputs
```

### `exs/oddEvenSort/` — Parallel Odd-Even Sort
Deterministic sorting benchmark. Useful for verifying correct alignment behavior on programs with no non-determinism.

### `SC24/particle/` — Particle Dynamics Simulation
N particles moving toward an attractor, distributed across ranks. Used as a SC24 conference benchmark.

### `SC24/laplace/` — Laplace Equation Solver
Grid-based PDE solver with MPI stencil communication.

### `SC24/gametheory/` — Game Theory Simulation
Domain-specific application exhibiting non-determinism patterns.

### Known Results from Real Applications

| Application | Non-determinism | Cause |
|---|---|---|
| MCB | ✅ Yes | `MPI_Testsome` |
| AMG2013 | ✅ Yes | `MPI_ANY_SOURCE` at `MPI_Irecv` |
| LULESH | ❌ No | Message reordering only, no CFG divergence |
| Hypre ParaSails | ❌ No | Message reordering only, no CFG divergence |

---

## Project Structure

```
ProvScopeMPI/
├── share/              # MPI interception and alignment libraries
│   ├── mpiRecord.cpp   # PMPI wrappers for recording
│   ├── mpiReproduce.cpp# Replay with online alignment
│   ├── alignment.cpp/h # Online and offline alignment algorithms
│   ├── loops.cpp/h     # Loop hierarchy parsing (loops.dot)
│   ├── messagePool.cpp # In-flight MPI buffer management
│   ├── utils.cpp/h     # String parsing, logging helpers
│   └── Makefile
├── skeleton/           # LLVM pass for basic block instrumentation
│   ├── skeleton.cpp    # FunctionPass: inserts printBBname() calls
│   ├── cfg.cpp/h       # Control flow graph utilities
│   ├── loops.cpp/h     # Loop detection and hierarchy output
│   ├── CMakeLists.txt
│   └── README.md
├── exs/                # Small example programs
│   ├── sample/         # Minimal non-determinism demo
│   └── oddEvenSort/    # Deterministic sorting baseline
└── SC24/               # SC24 benchmark suite
    ├── particle/
    ├── laplace/
    ├── gametheory/
    └── phloem-1.4.4/
```

---

## Output File Reference

| File | Created by | Contents |
|---|---|---|
| `.record<N>.txt` | `libmpirecord` | Pipe-delimited MPI call log per rank |
| `.record<N>.tr` | Instrumented code | Basic block execution trace per rank |
| `callLocations-<N>.json` | `libmpirecord` | MPI call site → node count mapping |
| `loops.dot` | Skeleton pass | Loop hierarchy as Graphviz digraph |

---

## Known Limitations

- Collective MPI operations are not yet intercepted
- `MPI_ANY_TAG` is not specially handled
- Mixing blocking and non-blocking calls in the same communication epoch has limited support
- Persistent requests (`MPI_Send_init` / `MPI_Recv_init`) are recorded but not extensively tested
