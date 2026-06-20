# AddressSanitizer & libFuzzer — Notes

A quick reference on two complementary tools for finding memory bugs in C/C++ code: **AddressSanitizer (ASan)**, a runtime bug detector, and **libFuzzer**, a coverage-guided fuzzing engine. They're most powerful when used together.

## AddressSanitizer (ASan)

ASan is a compiler instrumentation + runtime library built into Clang/LLVM. It inserts checks around memory operations to catch bugs as they happen, rather than letting them silently corrupt program state.

### Bugs it detects
- Out-of-bounds access (heap, stack, globals)
- Use-after-free
- Use-after-return
- Use-after-scope
- Double-free / invalid free
- Memory leaks (experimental, via LeakSanitizer)

### Basic usage
```bash
clang++ -O1 -g -fsanitize=address -fno-omit-frame-pointer program.cc -o program
./program
```

- `-O1` or higher: reasonable performance (ASan typically adds ~2x slowdown)
- `-g`: debug info for readable stack traces
- `-fno-omit-frame-pointer`: better stack traces

### Symbolizing reports
```bash
ASAN_SYMBOLIZER_PATH=/usr/local/bin/llvm-symbolizer ./program
```

### Useful runtime options
```bash
ASAN_OPTIONS=detect_leaks=1                  # enable leak detection (macOS; default on Linux)
ASAN_OPTIONS=detect_stack_use_after_return=0 # disable UAR checks
ASAN_OPTIONS=suppressions=MyASan.supp        # suppress known issues in external libs
```

### Notes
- ASan does **not** produce false positives — if you see a report, trust it.
- Not meant for production binaries; it's a testing/debugging tool, not hardened for security.
- Static linking of instrumented executables is not supported.

Full docs: https://clang.llvm.org/docs/AddressSanitizer.html

## libFuzzer

libFuzzer is a coverage-guided, in-process fuzzing engine. Instead of you writing test cases by hand, it mutates random input data and feeds it to a target function, using code coverage feedback to evolve inputs that explore new code paths.

### The fuzz target
```c
extern "C" int LLVMFuzzerTestOneInput(const uint8_t *Data, size_t Size) {
  DoSomethingWithData(Data, Size);
  return 0;
}
```

### Building and running
```bash
clang++ -g -fsanitize=address,fuzzer fuzz_target.cc -o fuzzer
./fuzzer
```

- `-fsanitize=fuzzer`: links the libFuzzer runtime and enables coverage instrumentation
- `-fsanitize=address`: pairs ASan with the fuzzer so memory bugs are caught and reported in detail

When a crash is found, libFuzzer writes a reproducer file to disk (e.g. `crash-<hash>`):
```bash
./fuzzer crash-<hash>   # replay the exact crashing input
```

### Improving fuzzing efficiency
| Technique | Flag / approach |
|---|---|
| Seed corpus | `./fuzzer CORPUS_DIR seeds/` — start from real sample inputs |
| Dictionaries | `-dict=file.dict` — supply known tokens/magic values (e.g. XML tags) |
| Parallel jobs | `-jobs=N -workers=M` — run multiple fuzzing processes |
| Corpus minimization | `-merge=1` — shrink corpus while preserving coverage |
| Crash minimization | `-minimize_crash=1 -runs=10000` — shrink a reproducer to a minimal case |

### Common failure modes to watch for
- **OOMs**: default RSS limit is 2GB (`-rss_limit_mb=N` to adjust)
- **Leaks**: undetected leaks eventually manifest as OOMs
- **Timeouts**: default per-input timeout is 1200s (`-timeout=N` to adjust)

Full docs: https://llvm.org/docs/LibFuzzer.html

## Why they're used together

| | AddressSanitizer | libFuzzer |
|---|---|---|
| Role | Detects memory bugs when they occur | Generates inputs that trigger bugs |
| Alone | Needs you to supply a triggering input | Without ASan, only catches hard crashes (segfaults), missing subtler corruption |

In practice: libFuzzer searches the input space automatically, and ASan explains exactly what went wrong (with a full stack trace) the moment a bad input is found. This combination underlies large-scale continuous fuzzing efforts like [OSS-Fuzz](https://github.com/google/oss-fuzz), and famously could have caught bugs like [Heartbleed](https://en.wikipedia.org/wiki/Heartbleed) (CVE-2014-0160).

### Typical compile line
```bash
clang++ -g -O1 -fsanitize=address,fuzzer -fno-omit-frame-pointer target.cc -o fuzz_target
```

## References
- AddressSanitizer: https://clang.llvm.org/docs/AddressSanitizer.html
- libFuzzer: https://llvm.org/docs/LibFuzzer.html
- libFuzzer tutorial: https://github.com/google/fuzzing/tree/master/tutorial/libFuzzer
- Sanitizers project: https://github.com/google/sanitizers
- OSS-Fuzz: https://github.com/google/oss-fuzz
