---
name: cpp-nav
description: >
  Navigate compiled C++ shared libraries (.so) back to source code using binary analysis tools.
  Use this skill whenever you need to find where a C++ function is defined, locate the source file
  for a symbol in a .so, understand a crash stack trace, map an error message string back to source
  code, find .so dependencies, or quickly navigate DV testbench C++ code structure.
  Also trigger when the user mentions .so files, libpwrv_lib_mid, nm, addr2line, readelf, objdump,
  C++ crash analysis, stack trace, SIGBUS, SIGSEGV, testdll, loaded_libs, or wants to find any
  C++ function/file without using slow find/grep across the whole source tree.
user-invocable: true
allowed-tools: Bash, Read, Glob, Grep
---

# cpp-nav — C++ Binary-to-Source Navigation

## Why This Skill Exists

In large DV environments, C++ source files are scattered across deep directory trees.
Using `find -name "*.cpp"` is slow and often finds the wrong copy. But every `.so` file
already knows where its source code lives — the debug info, symbol table, and build
directory encode this information. This skill teaches you to extract it efficiently.

## Core Tools

| Tool | What It Does | When to Use |
|------|-------------|-------------|
| `nm -C <.so>` | List all symbols (demangled) | Find if a function exists in a .so |
| `addr2line -e <.so> -f -C <addr>` | Address → source file:line | Map stack trace addresses to source |
| `readelf -d <.so>` | Show dependencies (NEEDED) | Find .so dependency chain |
| `strings <.so> \| grep "pattern"` | Search embedded strings | Find error messages back to source |
| `objdump -t <.so>` | Symbol table with types | Distinguish functions vs variables |
| `ldd <.so>` | Resolved dynamic dependencies | Check if all dependencies load |
| `stat <.so>` | File timestamps | Detect stale/rebuilt libraries |

## Finding .so Files in DV Runouts

Instead of searching the entire source tree, start from the simulation's own metadata:

### Method 1: sim.ini (fastest)
```bash
grep 'testdll' <runout>/*_sim.ini
# → -testdll=/path/to/libpwrv_lib_mid.so
```

### Method 2: loaded_libs.txt
```bash
cat <runout>/loaded_libs.txt
# Lists ALL .so files loaded by the simulation with their full paths
```

### Method 3: c_test_make.log
```bash
grep -E '\-I|\-o' <runout>/c_test_make.log | head -20
# Shows include paths (-I) and output paths (-o) from compilation
```

## From .so to Source Code

The key insight: a .so's path encodes its build directory, and the source is nearby.

```
/proj/.../out/.../common/tmp/src/test/tools/pwrv_lib_mid/libpwrv_lib_mid.so
                              ↑
                         build dir = .../tmp/src/test/tools/pwrv_lib_mid/
                         source    = same directory or ../src/
```

### Step-by-step:
```bash
# 1. Get .so path from sim.ini
SO_PATH=$(grep testdll runout/*_sim.ini | sed 's/.*testdll=//')

# 2. Derive build/source directory
SRC_DIR=$(dirname "$SO_PATH")

# 3. List source files there
ls "$SRC_DIR"/*.cpp "$SRC_DIR"/*.h 2>/dev/null
```

This replaces `find /proj/... -name "nbio_pcie_ip.cpp"` which can take minutes.

## Common Workflows

### Find where a function is defined
```bash
# Which .so has this function?
nm -C libpwrv_lib_mid.so | grep "Check_PwrBrk_Status"
# → 0000000000012345 T nbio_pcie_ip::Check_PwrBrk_Status(...)

# Get exact source file and line
addr2line -e libpwrv_lib_mid.so -f -C 0000000000012345
# → Check_PwrBrk_Status
#   /path/to/nbio_pcie_ip.cpp:356
```

### Analyze a crash stack trace
```bash
# From vcs_run.log:
#   #10 0x00001548c86b5212 in mid_init::run_soc_init() from libpwrv_lib_mid.so

# Get the .so path
SO=$(grep testdll runout/*_sim.ini | sed 's/.*testdll=//')

# Map address to source
addr2line -e "$SO" -f -C 0x00001548c86b5212
# → run_soc_init
#   mid_init.cpp:4455
```

### Find where an error message comes from
```bash
# Error: "Fail. BP_PWRBRK_OUT pad status 1, expected_val = 0"
strings libpwrv_lib_mid.so | grep "BP_PWRBRK_OUT"
# Confirms it's in this .so

nm -C libpwrv_lib_mid.so | grep "Check_PwrBrk"
# → nbio_pcie_ip::Check_PwrBrk_Status

# Then read the source at that function
```

### Detect library version mismatch
```bash
# Compare .so timestamp vs simulation start
stat -c '%Y %n' libpwrv_lib_mid.so    # .so modification time
stat -c '%Y %n' runout/vcs_run.log     # sim start time (first write)

# If .so is newer than sim start → mismatch!
# Also check post_sim_checker.log for explicit warnings
grep -i 'mismatch\|rebuild\|stale' runout/post_sim_checker.log
```

### List all functions in a .so
```bash
# All exported functions (T = text/code section)
nm -C libpwrv_lib_mid.so | grep ' T ' | head -30

# Filter by class
nm -C libpwrv_lib_mid.so | grep ' T .*nbio_pcie_ip::'
```

## DV Test Code Structure Patterns

Common C++ patterns in AMD DV testbenches:

| Pattern | Meaning |
|---------|---------|
| `TEST_MAIN()` | Test entry point |
| `thr_mp1_cpp()` | MP1 firmware thread |
| `mid_init::configure()` | Initialization phase |
| `SocPwr::Instance()->NBIO_PCIE->xxx()` | Power library function call |
| `RD_SMN_ADDR()` / `WR_SMN_ADDR()` | Register read/write via SMN |
| `simtool_getvalue("tb.path.signal")` | Read RTL signal value from C++ |
| `simtool_forcevalue("tb.path.signal", "1'b0")` | Force RTL signal |
| `dv_printf(dv_ERROR, "msg")` | Error reporting |
| `Wait(N)` | Wait N x 10ns |

## Tips

- `nm -C` is the demangled version — always use `-C` for C++ code
- `addr2line` needs debug info in the .so — if stripped, it won't work
- Multiple .so files may exist at different paths — use `loaded_libs.txt` to find the one actually loaded
- `.dpl` files are Perforce delta files — the actual compiled source may differ from depot
- `post_sim_checker.log` often contains library mismatch warnings that agents miss
- For `.so` symlinks, use `readlink -f` to find the real file
