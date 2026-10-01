# x48 safety audit and testing plan

This fork's review baseline is
[`887dd80e9472124064686e9006429421948cdea6`](https://github.com/doolin/x48/tree/887dd80e9472124064686e9006429421948cdea6).
Source line numbers below refer to that commit. Audit recorded September 30, 2026.
Keep this baseline available for reproducing failures and comparing compatible
behavior. The historical feature list remains in [doc/TODO](doc/TODO).

This document records findings and proposed work. No production fixes or permanent
tests accompany it. The first implementation work should establish reproducible
tests, followed by separate commits fixing the code and updating those tests.
UI replacement and broad emulator redesign are outside this plan.

## Evidence and scope

The review covered compiler warnings, Clang static analysis, source inspection,
and small AddressSanitizer (ASan) / UndefinedBehaviorSanitizer (UBSan) probes.
Exploratory probes ran on macOS arm64 with synthetic memory and isolated files;
they are not yet a checked-in test suite. Most exercised existing C modules. The
window-manager probe used an extracted function with Xlib stubs; its permanent
replacement must exercise the production source directly.

Each item distinguishes a reproduced failure from a source finding. Priorities
describe correctness and remediation urgency, not a claim of remote exploitability.
Passing the existing build does not demonstrate that these paths are safe.

## Findings to reproduce and fix

### T01 — High: RPL array decoding indexes before its allocation

- [ ] Commit a reproducer, then fix dimension unwinding and add boundary tests.

**Location:** [src/rpl.c](src/rpl.c), `dec_array()` / `any_array()`, especially
lines 1126–1130. When the outermost dimension finishes, `d--` makes `d == -1`,
then `dims[d]--` accesses memory before the allocation.

**Evidence:** ASan heap-buffer-overflow with a one-element REAL array. In synthetic
nibble memory, place length 36 at offset 0, `DOREAL` at 5, dimension count 1 at 10,
dimension length 1 at 15, and a 16-nibble real body at 20–35 (all zero except nibble
34 set to 1). Call `dec_array()` with address 0 and a large output buffer.

**Regression contract:** Decode ordinary one- and multidimensional arrays with
balanced brackets and the correct final address. Cover the last element, empty
dimensions, and linked arrays; never index a negative dimension. Define rejection
of malformed dimensions explicitly.

### T02 — High: unchecked RPL dimension multiplication overflows

- [ ] Commit an overflow reproducer, then validate dimension arithmetic and lengths.

**Location:** [src/rpl.c](src/rpl.c), `any_array()`, line 1077:
`elems *= dim_lens[i]`. Allocation failures and consistency with the object length
also need validation in this function.

**Evidence:** UBSan signed overflow for four dimensions of `0xfffff` on LP64.
The exploratory object used length 35 and unknown type `0x12345`, isolating the
dimension product before element decoding.

**Regression contract:** Reject dimensions whose product or storage requirement
cannot be represented or cannot fit the declared object. Test zero, one, boundary,
and oversized dimensions without trying to allocate huge buffers. Use checked
arithmetic before allocation, iteration, or address advancement.

### T03 — High: LCD menu writes can exceed the display buffers

- [ ] Commit the memory-write-path reproducer, then bound menu drawing to visible rows.

**Location:** [src/lcd.c](src/lcd.c), `menu_draw_nibble()`, lines 476–480;
[src/memory.c](src/memory.c), menu range calculation around lines 399–405 and
`write_nibble_sx()` → `menu_draw_nibble()`. Both `disp_buf` and `lcd_buffer` have
64 rows of 36 nibbles. The menu address range reserves eight rows even when fewer
rows remain visible; the drawing function does not check `y < 64`.

**Evidence:** UBSan reports row index 64 through the actual SX memory write path.
Allocate SX RAM; set RAM controller 1 to base `0x70000`, mask `0xf0000`;
set `display.lines = 62`, `nibs_per_line = 34`, display range
`[0x70000, 0x70000 + 34*63)`, menu range `[0x71000, 0x71110)`,
`disp.mapped = 1`, `shm_flag = 0`, and `device.display_touched = 0`.
Then call `write_nibble_sx(0x71022, 1)`.

**Regression contract:** Exercise all supported display/menu splits, including
zero and one visible menu row, with guard regions and sanitizer checks. Retain
correct RAM writes even when their LCD region is invisible. Check ordinary and
shared-memory rendering configurations separately.

### T04 — High: RPL formatting can overflow debugger output buffers

- [ ] Commit a long-list reproducer, then propagate output capacity through decoding.

**Location:** [src/rpl.c](src/rpl.c), `dec_list()` (line 459), `dec_char()`, and
other recursive decoders; [src/debugger.c](src/debugger.c), `get_stack()` and
`do_stack()` use 65,536-byte local buffers (lines 1541 and 1603). Truncating output
after decoding cannot protect the original buffer. LCD right-click handling in
[src/x48_x11.c](src/x48_x11.c), around line 4343, also calls `get_stack()`.

**Evidence:** ASan stack-buffer-overflow when `dec_list()` formats 30,000 `DOCHAR`
objects containing `A`, terminated by `SEMI`, into a 65,536-byte buffer. The input
occupies only about 210,000 nibbles and fits the synthetic address space.

**Regression contract:** Every decoder respects the caller's capacity, terminates
output when capacity permits, and returns an explicit truncation/error result.
Test exact fit, one byte too small, long strings/lists/arrays, deep nesting, and
malformed or cyclic references. Bound input traversal and recursion as well as
output size; preserve formatting for normal objects.

### T05 — High: window-manager command saving has a heap off-by-one

- [ ] Commit a production-source reproducer, then correct argument-vector sizing.

**Location:** [src/x48_x11.c](src/x48_x11.c), `save_command_line()`, lines
3447–3490, called by the `WM_SAVE_YOURSELF` event handler. It allocates
`saved_argc + 5` pointers, but can append four geometry arguments, `-iconic`, and
the terminating NULL: up to `saved_argc + 6` pointers are needed.

**Evidence:** ASan heap-buffer-overflow with `saved_argc = 1`, argv containing
only `x48`, and stubbed `XGetWindowAttributes()` returning `IsUnmapped`.

**Regression contract:** Verify mapped/unmapped windows, existing geometry/iconic
options, argument contents/count, and the final NULL. Capture the arguments in a
stubbed `XSetCommand()`; no X server is necessary. Check temporary allocation
ownership during repeated calls.

### T06 — High: dump2rom accepts records beyond the ROM buffer

- [ ] Commit the malformed-input reproducer, then validate whole record spans.

**Location:** [src/dump2rom.c](src/dump2rom.c), allocation at line 145 and writes
at lines 184–193. A 16-nibble record can start too close to the end of the
`0x100000`-byte nibble buffer.

**Evidence:** ASan heap-buffer-overflow from input `FFFFF:0000000000000000\n`.
Fifteen writes would fall beyond the allocation if execution continued.

**Regression contract:** Accept the final complete record at `FFFF0`; reject
`FFFF1` and `FFFFF`, bad hex, truncated records, and malformed separators. Invalid
input must return failure without publishing a seemingly valid `rom.dump`.
Run every tool test in its own temporary working directory.

### T07 — High: configuration strings overflow fixed stack buffers

- [ ] Commit separate path, resource-name, and serial-path cases; fix their bounds.

**Locations:** [src/init.c](src/init.c), `get_home_directory()` (line 1101) and
callers with 1,024-byte path/filename arrays;
[src/resources.c](src/resources.c), `get_string_resource_from_db()` (lines
140–147), unbounded concatenation into `full_name[1024]` / `full_class[1024]`;
[src/serial.c](src/serial.c), `serial_init()` (line 300), `sprintf()` of
`serialLine` into `tty_dev_name[128]`.

**Evidence:** ASan reproduced the home-directory overflow with a 2,048-character
absolute path, and the resource overflow with a 2,048-character `res_name`.
The serial-path issue is source-confirmed; it still needs an executable proof.

**Regression contract:** Validate the complete constructed length, including
separators, filename suffixes, and NUL. Test empty, normal, exact-boundary, and
oversized inputs, including a relative home directory appended to the user's
home path. Reject overlong paths clearly or allocate sufficient storage; silent
truncation must not redirect reads or state saving to a different path.

### T08 — High: absent expansion cards can be dereferenced

- [ ] Commit an empty-slot reproducer, then specify and implement absent-card reads.

**Location:** [src/memory.c](src/memory.c), `read_nibble_sx()` (lines 990–1015)
and corresponding CRC/GX paths. Memory-controller configuration can select a
card region while `saturn.port1` or `saturn.port2` is NULL.

**Evidence:** UBSan null access with `saturn.mem_cntl[2].config[0] = 0xc0000`,
`saturn.port1 = NULL`, and `read_nibble_sx(0xf07f0)`. The interactive SX recovery
softkey crash also reached this path through display refresh. Creating two RAM
card files allowed that session to continue; that workaround does not fix reads
from absent slots.

**Regression contract:** Test neither/one/both cards present, card removal,
mapping edges, and GX bank selection, for ordinary and CRC reads. Derive the
absent-slot value and CRC effect from the emulated hardware/memory contract;
do not simply select an arbitrary return value to avoid the crash. Review writes
and allocation failure handling alongside reads.

### T09 — Medium: ROM header access precedes size validation

- [ ] Commit a short-ROM reproducer, then validate sizes before allocation/access.

**Location:** [src/romio.c](src/romio.c), `read_rom_file()`: file-size narrowing
and doubling around lines 82–97, unchecked initial allocation at line 114, and
`(*mem)[0x29]` at line 190 before model/size validation.

**Evidence:** ASan heap-buffer-overflow for a four-byte file containing
`02 03 06 09`. Excessive-size narrowing and allocation failure are additional
source findings, not reproduced huge-file tests.

**Regression contract:** Reject short, invalid, or unrepresentable sizes before
indexing or allocation. Cover lengths through `0x29`, supported packed/unpacked
ROM forms, valid SX/GX sizes, and failed allocation/read. Simulate excessive file
sizes rather than allocating enormous fixtures. Define ownership and outputs on
failure so callers cannot use partial data.

### T10 — Medium: signed shifts and host-sized words undermine 32-bit operations

- [ ] Commit signed-shift reproducers and format fixtures before changing types.

**Location:** [src/init.c](src/init.c), `read_32()` and `read_u_long()` (lines
689 and 712), shift promoted signed `int` values by 24;
[src/memory.c](src/memory.c), `write_dev_mem()` (line 422), shifts a high timer
nibble by 28. [src/hp48.h](src/hp48.h), line 103, defines `word_32` as `long`,
which is 64 bits on LP64.

**Evidence:** UBSan reports undefined signed shifts for `read_32()` on bytes
`80 00 00 00` and `write_dev_mem(0x13f, 8)`.

**Regression contract:** Use defined-width unsigned assembly, then explicitly
apply the required signed timer interpretation. Test `0x7fffffff`, `0x80000000`,
and `0xffffffff`, all timer nibbles, round trips, and timer underflow. Preserve the
four-byte state-file representation and existing byte order. Audit arithmetic,
format specifiers, and pointer casts before changing a shared typedef; avoid a
blanket type replacement mixed into a small shift fix.

### T11 — Medium: repeated T1 reads count the same elapsed time again

- [ ] Commit the frozen-clock test, then correct timer accumulation.

**Location:** [src/timer.c](src/timer.c), `get_t1_t2()`, lines 443–454. It adds
`stop - timers[T1_TIMER].start` to the accumulated value without advancing the
start timestamp.

**Evidence:** A fake clock starting at second 1000, followed by
`restart_timer(T1_TIMER)` and a move to 250,000 microseconds, produces T1 readings
2 and then 4 with no further clock advancement. The probe allocated SX RAM and
set `in_debugger = 1` to isolate this path.

**Regression contract:** Reading twice at the same instant gives the same elapsed
count. Cover restart/start/stop/reset, later clock advancement, second boundaries,
and backwards clock movement. Confirm the intended tick conversion as part of
the tests; do not rely on real sleeps.

### T12 — Medium: timer adjustment truncates a wide value through abs(int)

- [ ] Commit the large-adjustment test, then use width-safe magnitude handling.

**Location:** [src/timer.c](src/timer.c), `get_t1_t2()`, line 506:
`delta = abs(adj_time)` where `adj_time` is a `word_64`.

**Evidence:** A fake-clock/RAM-access-time fixture producing an adjustment of
`2^32` left `set_0_time` at zero instead of applying that adjustment.

**Regression contract:** Test both signs around the `0x3c000` adjustment threshold,
values exceeding `INT_MAX`, and the most negative representable value. Avoid
replacing one overflow with signed negation of the minimum value. Verify both
the threshold decision and resulting timer/time-offset values.

### T13 — Medium: legacy pseudo-terminal discovery can spin forever on macOS

- [ ] Commit a bounded fake-open failure case, then make discovery terminate.

**Location:** [src/serial.c](src/serial.c), `serial_init()`, lines 225–252.
The legacy loop increments `c`, but repeatedly probes master names built with
the fixed `/dev/ptyp%x` pattern. Persistent errors other than `ENOENT` can keep
the loop running indefinitely.

**Evidence:** The interactive launch consumed approximately one CPU core in
serial initialization. Disabling terminal support with `+terminal` avoided it.
An automated syscall-fake reproducer still needs to be committed.

**Regression contract:** Simulate success, exhausted candidates, `ENOENT`,
`EACCES`, `EBUSY`, and slave-open failure. Assert a finite number of attempts,
appropriate error reporting, and descriptor cleanup. Evaluate the supported
platform PTY API after preserving the current failure in a test.

### T14 — Medium: failed state saving can destroy the previous saved state

- [ ] Commit isolated write-failure tests before changing the save protocol.

**Location:** [src/init.c](src/init.c), `write_files()`, line 1648 onward,
and `write_mem_file()`. Opening `hp48` with `"w"` truncates it before success is
known; the state writer ignores primitive write results and `fclose()` failure.
State, RAM, and card files are then saved separately.

**Evidence:** Source finding. Disk-full/interruption tests have not been run
against the user's state and must use disposable fixtures.

**Regression contract:** Inject short writes, `ENOSPC`, flush/close failure, and
publication failure. Return failure accurately and retain a recoverable,
consistent previous save. Stage checked writes in temporary files before
publication. Replacing individual files with `rename()` is not an atomic
transaction across state/RAM/cards: explicitly test recovery between each
publication step and choose a compatible generation/backup scheme in a separate
design step. Preserve legacy load compatibility.

### T15 — Lower priority: the first throttle comparison reads an uninitialized time

- [ ] Establish a deterministic/static-analysis proof, then initialize the timestamp.

**Location:** [src/emulate.c](src/emulate.c), `emulate()`, local `tv2` near line
2434 and comparison near line 2467. A pressed key or `throttle` can reach the
comparison before `tv2` has been assigned.

**Evidence:** Clang static-analysis finding, supported by source inspection.
ASan/UBSan alone do not establish absence of uninitialized reads.

**Regression contract:** Cover the first iteration with a pressed key and with
throttling enabled, using a controlled clock and bounded instruction execution.
Check the initialization path with static analysis; use an uninitialized-memory
detector in a supported environment if a reliable dynamic probe is needed.

### T16 — Lower priority: SIMPLE_64 skips speed-calibration time conversion

- [ ] Commit a fake-runtime calibration test, then implement the missing conversion.

**Location:** [src/emulate.c](src/emulate.c), `schedule()`, around line 2355.
Assignments to `s_1` and `s_16` exist only under `#ifndef SIMPLE_64`, with no
corresponding native-64-bit branch. [src/hp48.h](src/hp48.h) enables `SIMPLE_64`
when `HAVE_STDINT_H` is defined.

**Evidence:** Source finding. The missing assignments leave the calibration time
counters unchanged and select fallback instruction-per-tick values of 8192/16.

**Regression contract:** Inject known `RUN_TIMER` values and instruction counts,
then assert elapsed-time conversion and resulting calibration. Compare both
64-bit representations where those build configurations remain supported.

## Existing runtime observations

The SX session worked with the X server at `DISPLAY=:0`, terminal support disabled
(`+terminal`), X shared memory disabled (`+xshm`), and two RAM cards present. This
is a useful manual smoke-test baseline, not an acceptance condition for fixing
empty slots or serial initialization.

- [ ] Investigate the blank LCD seen with X shared memory enabled. Disabling it
  restored display output, but this does not yet isolate a code defect versus
  an X-server/platform interaction. Record server capabilities and compare the
  shared-memory and ordinary update paths before proposing a fix.

## Unit-testing approach

### Recommended starting point

Use **Unity for C assertions, the existing Automake build, and separate child
processes for crash/overflow/hang probes**. Unity's core is one C source file and
two headers, so it can be pinned and vendored with its license under a test-only
directory. It does not require adopting a different application build system.
See the [official Unity overview](https://www.throwtheswitch.org/unity).

The root [Makefile.am](Makefile.am) declares `AUTOMAKE_OPTIONS = dejagnu`, but
the reviewed tree has no accompanying test suite. This is not a reason to add
DejaGnu; use ordinary Automake test programs and test drivers for the first cases.

| Option | What it offers | Fit for this fork |
| --- | --- | --- |
| [Unity](https://www.throwtheswitch.org/unity) | Small C assertion framework | Recommended first step; hand-written fakes and explicit runners keep adoption small. |
| [cmocka](https://cmocka.org/) | C tests with mocking and fixtures | Reasonable alternative if its mocking facilities reduce fixture code. |
| [Criterion](https://github.com/Snaipe/Criterion) | Automatic test registration and process isolation, including crash/signal tests | Attractive if built-in isolation is preferred over a small subprocess driver; adds a framework dependency to install/build. |
| [Ceedling](https://www.throwtheswitch.org/ceedling) | Ruby-based build/test orchestration using Unity and CMock | Consider later if generated mocks become useful; another build layer is unnecessary initially. |

The offered C++ system with RSpec-style descriptions is also viable. Compile
production `.c` files with the C compiler and link those objects into its C++
runner through C-linkage declarations or a small C bridge. Some existing headers
use C++ keywords such as `class` as parameter names, so `extern "C"` alone may
not make them includable. Keep fixtures independent of the assertion framework
so changing runners does not require rewriting the emulator tests. No custom
framework is needed to begin.

### Commit the proof before the fix

For each finding, preserve this reviewable sequence:

1. **Proof commit:** Add the smallest fixture/reproducer against unchanged
   production code, plus a normal neighboring case where possible. Record the
   TODO ID, input, baseline commit, compiler/architecture, command, and observed
   failure. For a source-only finding, establish the failure before calling it
   reproduced. Infrastructure and test-only fakes may accompany this commit.
2. **Fix commit:** Change only the relevant implementation and update that same
   test to require the correct result. Remove its known-bug expectation, add
   boundary cases, and run both ordinary and sanitizer checks. The earlier commit
   remains executable evidence of the defect.
3. **Optional cleanup commit:** Refactor or broaden coverage after the regression
   is fixed. Keep mechanical cleanup separate from the behavioral correction.

There are two useful kinds of proof. For deterministic wrong results, such as
T11's readings of 2 then 4, assert the intended result and record the observed
one. The proof commit classifies that specific assertion failure as XFAIL;
the fix commit promotes the same test to an ordinary passing regression.
For memory corruption, execute a sanitizer-instrumented child and check the
specific diagnostic rather than trying to inspect state after undefined behavior.

Keep a small, explicit known-bugs manifest for unresolved sanitizer/failing
regression probes. Its driver should distinguish:

- **XFAIL:** The documented failure occurred, with the expected diagnostic class
  and relevant function/source location, or the exact documented assertion.
- **XPASS:** The case unexpectedly succeeded; fail the wrapper so a stale
  expectation cannot silently survive a fix.
- **ERROR:** Compilation, fixture setup, an unrelated crash, or another diagnostic
  prevented the intended test. Never count this as reproduction.
- **SKIP:** The platform or instrumentation is explicitly unsupported for this
  case. Report the reason and retain at least one supported CI job for each proof.

A generic nonzero exit is insufficient evidence. Do not match unstable addresses
or an entire compiler-specific backtrace. A hang probe should use fake syscalls
and an attempt counter; an external timeout bounds the child but does not by
itself prove the intended bug. Correctness regressions belong in the normal suite
after fixing; known-bug expectations are temporary and each must name a TODO ID.

### Fixtures and seams

Start with tests that call actual functions in `romio.c`, `memory.c`, `rpl.c`,
`timer.c`, and the small command-line tools. Use synthetic nibble arrays, generated
small files, and fake clocks/syscalls. Fixtures should describe their object
layout and expected output rather than depend on a personal ROM/state snapshot.

Global state includes `saturn`, memory-function pointers, card masks/sizes,
`display`, `disp`, `device`, timer arrays/offsets, scheduling counters, resource
strings, and Xlib handles. Resetting just `saturn` is not enough. Initially use a
fresh process for each stateful scenario, with explicit fixture initialization
and ownership of allocations. Keep pure arithmetic tests grouped where safe.

Link production objects with narrow Xlib/OS fakes, or redirect a dependency such
as `gettimeofday` only in the test build. For otherwise inaccessible static code,
a test translation unit can include the original `.c` file and replace external
dependencies; do not copy the implementation into the test. Prefer public entry
paths, such as `write_nibble_sx()` for T03, to preserve realistic call relationships.

If a production seam is genuinely necessary, make the smallest behavior-preserving
change in its own reviewed commit and document why the original code could not
be exercised directly. Do not make a new core API or eliminate all globals as a
prerequisite for adding tests. Headless tests may initially need X11 headers or
libraries at build/link time while never opening an X display.

Proposed layout (not yet created):

```text
tests/
  unit/              normal assertions and fixed regressions
  repro/             unresolved bug probes, named by TODO ID
  support/           memory builders, fake clock/Xlib/I/O, child-process driver
  fixtures/          small documented binary/text compatibility fixtures
  known-bugs.json    exact expected failure classification for each open probe
  vendor/unity/      pinned release sources, version/provenance, and license
```

A small Python 3 standard-library driver is a practical way to capture child
status/diagnostics and enforce deadlines on macOS and Linux; it would be an
explicit test-only dependency. Ordinary assertion suites remain C executables.
Keep application builds independent of test-runner dependencies.

### First implementation batches

- [ ] Bootstrap Unity/Automake integration and a single isolated sanitizer probe.
  Start with T06 (`dump2rom`) or T09 (short ROM): both provide a small file-based
  demonstration of the proof-commit/fix-commit workflow.
- [ ] Apply that workflow to T08 (empty cards), T01/T02 (arrays), T03 (LCD bounds),
  T04 (formatting), T05 (saved argv), and T07 (configuration strings). Each item
  gets its own proof and focused fix; do not bundle all changes into one patch.
- [ ] Add fake-clock fixtures for T11/T12, then T15/T16; cover T10's integer and
  serialization boundaries before shared type changes.
- [ ] Add fake PTY/I/O fixtures for T13 and failure-injected state saves for T14.
  Specify the multi-file recovery contract before implementing save publication.

### Build and execution policy

Proposed targets, to be implemented with the harness:

| Target | Contract |
| --- | --- |
| `make check` | Normal assertions and fixed regressions; no unresolved expected failures hidden as successes. |
| `make check-known-bugs` | Run and classify every supported unresolved proof explicitly as XFAIL, XPASS, ERROR, or skip. |
| `make check-sanitize` | Build instrumented production objects and run fixed regressions plus separately reported known-bug probes. |

Use a separate build directory or disposable checkout so sanitizer artifacts do
not replace the interactive emulator. Compile **and link** the relevant production
objects and test executable with sanitizer flags; instrumenting only the harness
misses errors inside the emulator. A starting Clang configuration is:

```text
-O1 -g -fno-omit-frame-pointer -fsanitize=address,undefined
-fno-sanitize-recover=all
```

ASan checks memory safety; UBSan covers cases such as signed overflow and invalid
shifts. These are complementary to assertions, not replacements for behavioral
tests. See the official [ASan](https://clang.llvm.org/docs/AddressSanitizer.html)
and [UBSan](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html) guides.
Make symbolization and diagnostic matching part of validating the driver.

Run normal optimized tests as well as sanitizer builds. Record the C language
mode explicitly, since it can affect UB diagnostics. Use Clang/GCC on Linux and
Clang on macOS where available; include LP64 and, when practical, a 32-bit build
to reveal assumptions about `long`. Cover supported XShm and 64-bit representation
configurations. Keep unsupported configurations visibly skipped.

Retain targeted static analysis and warnings (`-Wall -Wextra -Wformat=2`, with
conversion warnings reviewed separately). Avoid imposing blanket `-Werror` on
all legacy code before triaging existing warnings. Intentional emulated integer
wraparound needs a documented contract, not indiscriminate suppression.

Default tests must not need an X server, proprietary ROM downloads, real serial
devices, or the user's calculator state. Use temporary directories for all writes.
An optional GUI/ROM smoke test should select an explicit disposable `-home`
directory and be separate from unit tests; Linux Xvfb can support that layer.

### Broader coverage after the initial regressions

| Area | Tests to grow from the first fixtures |
| --- | --- |
| CPU/registers (`emulate.c`, `actions.c`, `register.c`) | Instruction vectors, carry/borrow, BCD versus hexadecimal arithmetic, field limits, shifts, return stack, and PC wrapping. |
| Memory (`memory.c`) | SX/GX mapping, boundaries, controller configuration, RAM/ROM/card permissions, bank selection, and CRC effects. |
| Timers/device state (`timer.c`, `device.c`) | Deterministic elapsed time, interrupt conditions, stop/restart, wraparound, and clock discontinuities. |
| Keyboard (`actions.c`, `x48_x11.c`) | Matrix press/release transitions, repeated events, simultaneous keys, and ON-key behavior; test X-event translation separately. |
| LCD (`lcd.c`) | Nibble-to-pixel expectations, bit order, line offset, menu boundaries, and unchanged/dirty regions using fake rendering calls. |
| RPL/debugger (`rpl.c`, `debugger.c`) | Bounded formatting, nested/malformed objects, and valid-object output compatibility. |
| Files (`init.c`, `romio.c`, tool programs) | Golden byte order/format fixtures, round trips, truncation, malformed sizes, and save/reload after injected failures. |
| Serial (`serial.c`) | Open/read/write errors, partial operations, device state transitions, and cleanup; optional PTY integration separately. |

Compare valid inputs with the pinned fork baseline when preserving behavior.
Undefined behavior, crashes, and data loss are not compatibility requirements;
derive their corrected result from a stated contract. Hardware documentation or
independent known vectors should resolve cases where the baseline is itself wrong.

- [ ] Once bounded parser interfaces exist, add resource-limited fuzz targets for
  ROM/dump input, RPL decoding, and state loading. Preserve minimized findings as
  named regressions using the same proof-then-fix workflow.
- [ ] Mark an item complete only when its historical proof is committed, its fix
  passes normal and applicable sanitizer checks, its known-bug entry is removed,
  and the relevant valid-input compatibility cases still pass. Add proof/fix commit
  references to the item as work lands.
