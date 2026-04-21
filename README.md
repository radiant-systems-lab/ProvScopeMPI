# ProvScopeMPI

ProvScopeMPI is a tool for **recording, replaying, and analyzing non-determinism** in MPI applications. It intercepts MPI communication calls to capture execution traces, then replays those traces with a fixed communication order — letting you separate *communication-order non-determinism* from *input-based non-determinism* and pinpoint exactly where two execution runs diverge in control flow.

---

## How It Works

ProvScopeMPI has two cooperating components:

1. **LLVM Instrumentation Pass** (`skeleton/`) — A compile-time pass that inserts a `printBBname()` call at the entry of every basic block. At runtime this produces a fine-grained execution trace (`.record<rank>.tr`) and a loop hierarchy graph (`loops.dot`).

2. **MPI Interception Libraries** (`share/`) — Shared libraries built on PMPI that wrap MPI calls. Three modes:
   - **`libmpirecord.so`** — Records every MPI call (source, destination, tag, request, execution context) to `.record<rank>.txt`.
   - **`libmpireproduce.so`** — Replays execution using the recorded message order. At each MPI call it validates the current execution context against the recording and reports divergences.
   - **`libmpiprovscope.so`** — Offline analysis variant.

### Record → Replay Workflow

```
Original Run                         Replay Run
────────────────────────────────     ──────────────────────────────────────
instrumented_app + libmpirecord      instrumented_app + libmpireproduce
        │                                      │
        ▼                                      ▼
  mpirun -n N ./app [inputs]           mpirun -n N ./app [new_inputs]
        │                                      │
        ▼                                      ▼
  .record<rank>.txt  (messages)        Loads .record<rank>.txt
  .record<rank>.tr   (BB trace)        Validates execution context
  callLocations-<rank>.json            Reports divergence points
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
| [nlohmann/json](https://github.com/nlohmann/json) | JSON output in the record/reproduce libraries |
| CMake ≥ 3.10 | Building the skeleton pass |
| C++20-capable compiler | Building the share libraries |

Install on Ubuntu 20.04+:

```bash
sudo apt-get install openmpi-bin openmpi-common libopenmpi-dev \
                     clang-11 llvm-11 llvm-11-dev libgraphviz-dev cmake

pip install wllvm

# If clang-11 is not on PATH as 'clang', symlink it:
sudo ln -s /usr/bin/clang-11 /usr/bin/clang
sudo ln -s /usr/bin/llvm-link-11 /usr/bin/llvm-link
sudo ln -s /usr/bin/opt-11 /usr/bin/opt

# wllvm needs to wrap mpicc/mpicxx:
export MPICC=wllvm
export MPICXX=wllvm++
```

nlohmann/json is a header-only library. Either install it system-wide or drop `json.hpp` into `share/`:

```bash
sudo apt-get install nlohmann-json3-dev   # Ubuntu 20.04+
# or manually: place single-header json.hpp at /usr/include/nlohmann/json.hpp
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
# Output: libmpirecord.so  libmpireproduce.so  libmpiprovscope.so
```

---

## Usage

### Step 1 — Instrument Your Application

```bash
# Compile with wllvm so the bitcode is embedded
CC=wllvm mpicc -o app app.c

# Extract bitcode
extract-bc app -o app.bc

# Run the skeleton pass to insert BB probes
opt -load skeleton/build/libSkeletonPass.so --bbprinter -o app_inst.bc < app.bc

# Recompile: record variant
mpicc -o app_record app_inst.bc share/record.cpp \
      -lm -L./share -lmpirecord -I./share

# Recompile: replay variant
mpicc -o app_replay app_inst.bc share/record.cpp \
      -lm -L./share -lmpireproduce -I./share
```

> `share/record.cpp` provides the `printBBname()` stub that writes the basic-block trace file. It must be linked into both variants.

### Step 2 — Record the Original Run

```bash
mpirun -n 4 ./app_record [original_inputs]
```

This produces per-rank output files:

| File | Contents |
|---|---|
| `.record<N>.txt` | Pipe-delimited MPI call log |
| `.record<N>.tr` | Basic block execution trace |
| `callLocations-<N>.json` | MPI call site → node count mapping |
| `loops.dot` | Loop hierarchy as a Graphviz digraph (shared) |

### Step 3 — Replay and Detect Divergences

```bash
mpirun -n 4 ./app_replay [same_or_different_inputs]
```

`libmpireproduce.so` enforces the recorded message order and prints any divergence in execution context — i.e., where control flow between the two runs first differs.

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

## Example Programs

### `exs/sample/` — Minimal Non-determinism Demo

Three-rank program where rank 0 calls `MPI_Recv(..., MPI_ANY_SOURCE, ...)` and receives from rank 1 or rank 2 in unpredictable order. Rank 1 reads a value from a file, so changing that file also changes execution paths. Demonstrates both communication-order and input-based non-determinism.

```bash
cd exs/sample

# Build (plain, without instrumentation)
make

# Record with original inputs
make original      # runs: mpirun -n 3 ./main origInput.txt

# Replay with changed inputs (input non-determinism exposed)
make reproduced    # runs: mpirun -n 3 ./main reproducedInput.txt
```

To run the full record → replay pipeline with instrumentation:

```bash
cd exs/sample

# 1. Compile with wllvm
CC=wllvm mpicc -o main main.c

# 2. Extract bitcode and instrument
extract-bc main -o main.bc
opt -load ../../skeleton/build/libSkeletonPass.so --bbprinter -o main.mod.bc < main.bc

# 3. Link record and replay executables
mpicc -o main_record main.mod.bc ../../share/record.cpp \
      -lm -L../../share -lmpirecord -I../../share
mpicc -o main_replay  main.mod.bc ../../share/record.cpp \
      -lm -L../../share -lmpireproduce -I../../share

# 4. Record
mpirun -n 3 ./main_record origInput.txt

# 5. Replay with different input (exposes divergence)
mpirun -n 3 ./main_replay reproducedInput.txt
```

### `exs/oddEvenSort/` — Parallel Odd-Even Sort

Deterministic sorting benchmark. Useful for verifying correct alignment behavior on programs with no non-determinism.

```bash
cd exs/oddEvenSort
make
mpirun -n 4 ./oddEvenSort
```

### `exs/globalAlignment/` — Global Alignment Example

Minimal example illustrating the global alignment algorithm used when multiple ranks must synchronize their replay context.

### `SC24/particle/` — Particle Dynamics Simulation

N particles moving toward an attractor, distributed across ranks. SC24 conference benchmark.

```bash
cd SC24/particle
make
mpirun -n 4 ./mpi-particle init.txt.default
```

### `SC24/laplace/` — Laplace Equation Solver

Grid-based PDE solver with MPI stencil communication.

```bash
cd SC24/laplace
make
mpirun -n 4 ./laplace_mpi
```

### `SC24/gametheory/` — Game Theory Simulation

Domain-specific application exhibiting non-determinism patterns.

```bash
cd SC24/gametheory
make
mpirun -n 4 ./mpi-gametheory
```

### Known Results from Real Applications

| Application | Non-determinism | Cause |
|---|---|---|
| MCB | Yes | `MPI_Testsome` |
| AMG2013 | Yes | `MPI_ANY_SOURCE` at `MPI_Irecv` |
| LULESH | No | Message reordering only, no CFG divergence |
| Hypre ParaSails | No | Message reordering only, no CFG divergence |

---

## Project Structure

```
ProvScopeMPI/
├── share/                  # MPI interception and alignment libraries
│   ├── mpiRecord.cpp       # PMPI wrappers for recording
│   ├── mpiReproduce.cpp    # Replay with online alignment
│   ├── mpiProvScope.cpp    # Offline analysis variant
│   ├── alignment.cpp/h     # Online and offline alignment algorithms
│   ├── alignmentUtils.cpp/h
│   ├── loops.cpp/h         # Loop hierarchy parsing (loops.dot → loopNode tree)
│   ├── messagePool.cpp/h   # In-flight MPI buffer management
│   ├── messageTools.cpp/h  # Message utilities
│   ├── record.cpp          # printBBname() stub (linked into app, not the lib)
│   ├── utils.cpp/h         # String parsing, logging, MPI_ASSERT
│   └── Makefile
├── skeleton/               # LLVM pass for basic block instrumentation
│   ├── skeleton.cpp        # FunctionPass: inserts printBBname() calls
│   ├── cfg.cpp/h           # Control flow graph utilities
│   ├── loops.cpp/h         # Loop detection and hierarchy output
│   ├── tools.cpp/h
│   └── CMakeLists.txt
├── exs/                    # Small example programs
│   ├── sample/             # Minimal non-determinism demo (MPI_ANY_SOURCE + file input)
│   ├── oddEvenSort/        # Deterministic sorting baseline
│   ├── globalAlignment/    # Global alignment demo
│   └── buffers/
└── SC24/                   # SC24 benchmark suite
    ├── particle/           # N-body particle simulation
    ├── laplace/            # Laplace PDE solver
    ├── gametheory/         # Game theory simulation
    └── phloem-1.4.4/
```

---

## Output File Reference

| File | Created by | Contents |
|---|---|---|
| `.record<N>.txt` | `libmpirecord` | Pipe-delimited MPI call log per rank |
| `.record<N>.tr` | `record.cpp` via skeleton pass | Basic block execution trace per rank |
| `callLocations-<N>.json` | `libmpirecord` | MPI call site → node count mapping |
| `loops.dot` | Skeleton pass | Loop hierarchy as Graphviz digraph |

---

## Known Limitations

- Collective MPI operations are not yet intercepted
- `MPI_ANY_TAG` is not specially handled
- Mixing blocking and non-blocking calls in the same communication epoch has limited support
- Persistent requests (`MPI_Send_init` / `MPI_Recv_init`) are recorded but not extensively tested
