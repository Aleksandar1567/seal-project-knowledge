# Fuzzing & MPC Compiler Testing — Notes

Personal notes from working through the warm-up task: understanding AddressSanitizer, libFuzzer, and how to build LLM-generated fuzzing harnesses for the MPC compiler testing project (BabelFuzz / EzPC).

---

## 1. AddressSanitizer (ASan)

A Clang-built memory error detector. Compiles instrumentation into the binary that catches bugs at the moment they happen.

**Detects:**
- Out-of-bounds access (heap, stack, globals)
- Use-after-free / use-after-return / use-after-scope
- Double-free
- Memory leaks

**Basic usage:**
```bash
clang++ -O1 -g -fsanitize=address -fno-omit-frame-pointer program.cc -o program
```

- ~2x runtime slowdown — a testing tool, never for production binaries.
- Does not produce false positives — if it reports something, trust it.

---

## 2. libFuzzer — core mechanics

A coverage-guided, in-process fuzzing engine. It mutates raw bytes and feeds them to a target function, using code-coverage feedback to evolve inputs that explore new code paths.

### Key terms

| Term | Meaning |
|---|---|
| **Fuzz target** | The function `LLVMFuzzerTestOneInput` — the entry point libFuzzer calls repeatedly with different byte arrays. |
| **API under test** | The actual library/function being fuzzed — not part of libFuzzer itself. |
| **Bytes (`Data`, `Size`)** | Raw, meaningless `uint8_t` array generated/mutated by libFuzzer. No structure, no format. |
| **Harness** | The glue code you write (inside the fuzz target) that converts bytes into something the API under test understands, then calls the API. |

### Minimal fuzz target
```c
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  DoSomethingInterestingWithMyAPI(Data, Size);
  return 0;
}
```

The fuzz target signature is a convention, not a libFuzzer-specific dependency — the same function can be driven by AFL or Radamsa too. Only *how* `Data`/`Size` are generated differs between engines.

### Build & run
```bash
clang++ -g -fsanitize=address,fuzzer fuzz_target.cc -o fuzzer
./fuzzer
```
- `-fsanitize=fuzzer`: links libFuzzer runtime + coverage instrumentation
- `-fsanitize=address`: pairs ASan so memory bugs are caught and explained

Crashes get saved to disk automatically (`crash-<hash>`), replayable with `./fuzzer crash-<hash>`.

### One harness per function, not per file
For a library with ~100 functions, you don't cram them into one `LLVMFuzzerTestOneInput`. Each function (or logical group) gets its own small `.cc` file, compiled into its own fuzzer binary:
```
fuzz_parse_header.cc      → testira parse_header()
fuzz_decompress.cc        → testira decompress()
```
This keeps crash attribution clear and coverage signal focused.

### Converting bytes into meaningful input — the spectrum

| Function signature | Conversion approach |
|---|---|
| `f(const uint8_t*, size_t)` | Pass `Data`/`Size` directly — no conversion needed |
| `f(const std::string&)` | `std::string s(reinterpret_cast<const char*>(Data), Size);` |
| `f(int, float, bool, string...)` | Use `FuzzedDataProvider` to carve typed values out of the byte stream |
| `f(structured input — AST, JSON, IR program)` | **Structure-aware fuzzing**: bytes drive *decisions* in a generator that builds the structure directly, instead of being parsed as text |

`FuzzedDataProvider` example:
```cpp
#include <fuzzer/FuzzedDataProvider.h>
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  FuzzedDataProvider fdp(Data, Size);
  int x = fdp.ConsumeIntegral<int>();
  bool flag = fdp.ConsumeBool();
  std::string label = fdp.ConsumeRandomLengthString();
  compute(x, flag, label);
  return 0;
}
```

### Why naive byte→text doesn't work for structured input (e.g. MPC programs)

Feeding raw bytes directly as source code to a compiler almost always produces a syntax error — coverage never gets past the parser. Two problems:
1. Astronomically low odds random bytes form valid syntax.
2. Even when parsing succeeds, the program may contain undefined behavior (div by zero, out-of-bounds array access), causing false positives.

**Fix — structure-aware fuzzing:** bytes don't *become* the program; they *choose* which AST node to build next. The generator guarantees syntactic (and optionally semantic) validity by construction, then serializes the resulting structure to source text only at the end.

---

## 3. AST (Abstract Syntax Tree)

A way to represent a program's structure as a **tree of objects in memory**, instead of raw text.

`inp = inp * 2` becomes:
```
Assignment
├── Variable: inp
└── Multiply
    ├── Variable: inp
    └── Constant: 2
```

Why it matters for fuzzing: a generator that builds this tree directly **cannot** produce invalid syntax — you simply can't construct a `Multiply` node with the wrong number of children. This is why BabelFuzz uses a Python AST as its intermediate representation (IR): generate the tree, then serialize it into whichever target DSL.

---

## 4. Complexity levels for harness generation (project's "Next steps")

Three levels of increasing constraint strength a generator must satisfy:

| Level | Constraint | Example |
|---|---|---|
| **1 — Syntactic only** | Output parses without error. Meaning is irrelevant. | `x / 0` is syntactically fine — division by zero isn't a grammar rule. |
| **2 — Syntactic + semantic** | Output parses *and* respects meaning: types match, no undefined behavior, indices in bounds. | This is what BabelFuzz's `RunSnippet` enforces before accepting a generated snippet. |
| **3 — Protocol specification** | Stateful correctness — validity depends on prior history, not just the current message. | TLS handshake ordering, HTTP/2 stream states. For MPC, this would live at the network-execution level, not IR generation. |

**Where MPC IR generation (BabelFuzz-style) sits:** Level 2. A single generated program must be syntactically valid AND semantically well-defined (no div-by-zero, no out-of-bounds, correct secrecy/type usage).

### Level 1 example — grammar only, no semantics
```
expr := NUMBER | VARIABLE | expr '+' expr | expr '*' expr | expr '/' expr | '(' expr ')'
```
Valid by this grammar but semantically broken: `x / 0`, `(((1)))`, `y * y * y`. The grammar has no concept of "division by zero is a problem" — that's a semantic rule, not a syntax rule.

### Real Level 1 harness (MP-SPDZ, Python-based generator)
Key patterns that intentionally violate semantics while staying syntactically valid:
- `sint.Array(0)` — array size 0 is syntactically fine, semantically empty.
- `idx = rng.randint(0, arr_size + 2)` — deliberately may exceed bounds.
- Reassigning a variable that may not exist yet (use-before-define).
- Calling `.reveal()` on `cint` (public) values — syntactically valid method call, semantically meaningless (`reveal()` is for secret `sint` values).

These represent the **baseline** — weakest form of test generation, useful as a lower bound for comparison against smarter (semantic-aware) generators.

---

## 5. BabelFuzz — overview

Paper: *Cost-Effective Testing of MPC Compilers* (Watzinger, Wüstholz, Garg, Christakis — TU Wien / MPI-SWS).

**Problem it solves:** MPC compilers may contain *logic bugs* — they compile and run, but silently produce the wrong output. No single standardized DSL exists across compilers, so traditional differential testing (run two compilers on the same input) doesn't directly apply.

**Core idea:**
1. Generate a well-defined program in an **intermediate representation (IR)** — a restricted Python AST.
2. Translate that IR to (a) the target MPC compiler's DSL and (b) a plain Python "**oracle implementation**" with concrete hardcoded values instead of MPC placeholders.
3. Compile/run the MPC version, run the oracle in plain Python.
4. Compare outputs — any mismatch is a logic bug in the MPC compiler (oracle is assumed correct).

This is **differential testing using a single, always-available reference implementation** instead of needing a second real compiler.

**Two modes:**
- **DT (differential testing)** — compare DSL output vs oracle output. Faster, found nearly all bugs.
- **MT (metamorphic testing)** — apply semantics-preserving transformations (e.g. `x = e` → `x = e * 1`) to the IR, compare original vs transformed DSL output. Useful for features hard to model in the oracle, but ~3-10x slower at finding the same bugs.

**Results:** 27 new logic bugs found across 4 compilers (MP-SPDZ, EMP Toolkit, EzPC, Silph); rediscovers all previously known bugs from the only prior tool (MT-MPC), usually within an hour.

**Tool:** https://github.com/Rigorous-Software-Engineering/BabelFuzz

---

## 6. The four MPC compilers BabelFuzz tests

| Compiler | Maker | Language style | Notes |
|---|---|---|---|
| **MP-SPDZ** | Alan Turing Institute | Python-like | Most popular (~1.1K GitHub stars), 30+ protocols, supports secret array indices, fixed-point arithmetic |
| **EzPC** | Microsoft Research India | C-like | Hybrid arithmetic/boolean compilation; no Boost dependency → faster Docker build; does **not** allow secret values in conditionals |
| **EMP Toolkit** | Xiao Wang et al. | C++ library (no separate DSL) | Garbled-circuits based; focus on semi-honest two-party module |
| **Silph** | Cornell | C-subset | Hybrid MPC like EzPC, but **does** allow secret conditionals and secret array indices; automatic low-level protocol partitioning |

**Why EzPC as a starting point:** purely an engineering convenience — no Boost dependency means faster Docker image builds. It is not "semantically simpler" as a compiler; its hybrid arithmetic/boolean conversion logic is actually a rich source of bugs (see below).

---

## 7. Boost

A large general-purpose C++ library (not MPC-specific) providing functionality C++ lacked natively: smart pointers, regex, filesystem access, threading, networking (Asio), parsing (Spirit), and more. Many features later entered the C++ standard after originating in Boost.

Relevance here: some MPC toolchains depend on Boost as part of their C++ infrastructure. Building a project with a Boost dependency means downloading and compiling a large, template-heavy library — which adds real time to every Docker image build. EzPC's lack of a Boost dependency is purely why it builds faster, unrelated to compiler design quality.

---

## 8. ABY framework

A C++ framework implementing the actual **two-party secure computation cryptography** underneath compilers like EzPC. EzPC compiles your C-like source into C++ code that calls into ABY; ABY does the real protocol work.

**Name origin** — three sharing schemes it supports and converts between:
- **A**rithmetic sharing — value split as numbers that sum to the original; fast for add/multiply.
- **B**oolean sharing — value split bitwise via XOR; fast for comparisons/bitwise ops.
- **Y**ao (garbled circuits) — whole logical circuit is "garbled" so no party sees intermediate values.

### Layer stack
```
EzPC DSL code               (what you write)
        ↓ compiled to
C++ code calling ABY        (generated by EzPC compiler)
        ↓
ABY framework                (arithmetic / boolean / Yao sharing + conversions)
        ↓
Network                      (two parties exchange data)
```

Conversions between schemes show up directly in generated code, e.g.:
```cpp
x + _al( z << y )   // _al = convert boolean-shared result back to arithmetic sharing
```

**Why this matters for fuzzing:** BabelFuzz found several EzPC bugs specifically at these A/B/Y conversion boundaries (e.g. "modulo fails for arithmetic shares", "signed types behave like unsigned"). These conversions are a structurally complex part of the compiler — generating programs with lots of mixed bit-shift/modulo/comparison operations is a good way to stress exactly this part.

---

## 9. Working without Docker/EzPC installed

Without the EzPC Docker build, you cannot execute a real `.mpc`/`.ezpc` program — there's no compiler available to run it.

**What you CAN still do:**
1. Generate the IR program and translate it to (a) the target DSL text and (b) a plain Python **oracle** implementation.
2. Run the oracle directly: `python3 prog_0000_oracle.py` → gives the "correct" expected output.
3. Manually/visually inspect the generated DSL text for syntactic plausibility.
4. Validate your *generator* (harness) logic — does the oracle run without errors, does it produce sensible values?

**What you CANNOT do without the compiler:**
- Actually compile/execute the `.mpc` file.
- Automatically compare compiler output vs oracle output — the core bug-detection step of differential testing requires both sides to exist.

```
prog_0000.mpc           ← needs real compiler (Docker) to run
prog_0000_oracle.py      ← pure Python, runs immediately
```

Once Docker + EzPC is available, the comparison becomes:
```python
oracle_result = run_python(prog_0000_oracle.py)
ezpc_result   = compile_and_run_ezpc(prog_0000.mpc)
if oracle_result != ezpc_result:
    print("BUG FOUND:", oracle_result, "vs", ezpc_result)
```

### Docker persistence notes
- Docker installed on your own machine persists permanently — `docker build` once, then `docker run -it <image>` afterward without rebuilding (unless the Dockerfile changes).
- Mount a local folder to keep generated files outside the container's ephemeral filesystem:
  ```bash
  docker run -it -v /local/path:/workspace ezpc bash
  ```
- Useful commands: `docker images` (list built images), `docker ps -a` (list containers including stopped ones).

---

## 10. Open threads / next steps

- Build out a Level 2 (syntax + semantic) harness for MPC IR generation, comparable to BabelFuzz's own `GenerateSnippet`/`RunSnippet` approach — likely the core deliverable for the MPC compiler track.
- Decide whether Level 1 and Level 3 domains are covered personally or split across team members (other papers in the broader project cover SMT solvers, JIT compilers, database systems, etc., which may naturally fit Level 1 or Level 3 better).
- Once Docker + EzPC is confirmed working, wire up the full generate → translate → compile/execute → compare-with-oracle loop.
- Explore whether libFuzzer's coverage-guided mutation can sit on top of a structure-aware generator (mutating AST-building decisions encoded in the byte stream) rather than mutating raw program text — this is the bridge between libFuzzer mechanics and BabelFuzz-style IR generation.