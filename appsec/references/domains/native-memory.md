# Domain: native, memory safety, binary, kernel

Use when the target processes untrusted bytes in a memory-unsafe or privileged context: C/C++/Objective-C, Rust `unsafe`, FFI (CGo, JNI, Python C-ext, N-API), kernel modules, drivers, parsers/decoders, network daemons, firmware, binary loaders, language runtimes, JIT. Protocol authz logic and state machines: [protocols-messaging.md](protocols-messaging.md).

## Discipline

- Recompute every bound and lifetime from attacker input and from **every** caller. Check against the worst case accepted, not typical test vectors.
- A panic, sanitizer warning, or crash proves a bug only when real untrusted input can reach it. Do not infer memory corruption, code execution, or shared-availability impact from the label alone.
- Verify with a local harness with sanitizers (ASan, UBSan, TSan, MSan), deterministic concurrency tests, existing fuzz targets. Stop once you have proven the invariant is violated and the consequence is observable; **do not** develop exploitation techniques past corruption.
- Assembly, JIT code, custom allocators, intra-object accesses, and external libraries may lie outside what the sanitizer tracks. Determine whether the relevant instructions are instrumented.
- ABI, allocator, architecture, build features, or reachability not visible: Needs validation.

## Bounds, integers, representation

| Layer | What to hunt |
|---|---|
| Out-of-bounds read/write | Length, offset, index, terminator reaching a fixed/allocated buffer with a wrong bound. Recompute the space left after prefix, alignment, padding, terminator. Check both source and destination capacity; is short input read before the declared length is trusted |
| Overflow, underflow, truncation, sign | Attacker-influenced arithmetic before allocation, copy, loop, index, pointer: `a - b` when `b > a`; `count * size`; addition near max; negative to unsigned; 64-bit length truncated to 32-bit; sentinel `-1` becoming a huge size. Which representation was checked, which one is used |
| Unit confusion, pointer depth | Mixing bytes, elements, code units, pages, words, wire units, nested element sizes. A bound check that uses the same wrong unit as the allocation is still wrong |
| Format string, native injection | Attacker-controlled format string (`printf`, `syslog`, logging wrappers, `%n`), or input reaching `system`/`popen`/`exec*` through a native wrapper | The format or command string itself is attacker-controlled, not just an argument |
| Uninitialized data | Buffers, padding, struct fields, vector capacity that are returned, compared, hashed, serialized, or sent across a trust boundary before initialization. Requires an observable consumer and the actual output length |

## Lifetime, type, concurrency

| Layer | What to hunt |
|---|---|
| Use-after-free, stale view, double free | The owner is freed while a callback, wait queue, timer, iterator, borrowed slice, or cached raw pointer is still in use. Examine every error, cancel, close, realloc branch |
| Type confusion, wrong downcast | The tag, vtable, union discriminator, object kind, or external handle that was checked differs from the representation read later. Unchecked dynamic cast, stale tag after reuse, validated element differs from the element used |
| Refcount, ownership races | Non-atomic retain/release, check-then-use without a lock, ownership rules that disagree between threads |
| Shared-state races, TOCTOU | Parallel stream parsers, global caches, lazy init, signal handlers, teardown that invalidate an established bound/policy/pointer. Verify with a deterministic schedule, barriers, or TSan; an interleaving hypothesis with no security-relevant state transition: Needs validation |
| Lock order, deadlock | External operations take locks in inconsistent order, hold a lock across a callback/blocking I/O. Only an availability issue when bounded input can stall a shared process |

## FFI and ABI

| Layer | What to hunt |
|---|---|
| Pointer, length, ownership contract | Caller and callee disagree on who allocates, frees, pins, modifies the buffer, how long the pointer lives, whether the length is bytes or elements. Trace both sides of every `extern`, CGo, JNI, binding, generated wrapper. Null, zero length, aliasing, retained callbacks |
| Layout, alignment, enums | Struct, bitfield, packed, callback signature, integer width, enum, calling convention differing by architecture or build flag. Check `repr(C)`, packing, alignment, endianness |
| Unwind, exception, threads | An exception/panic crossing an ABI that forbids unwinding; a callback running after teardown; an API meant for a single runtime thread called from another |

## Binary loading and runtime

| Layer | What to hunt |
|---|---|
| Library, plugin search order | A privileged process loads a library, plugin, or helper from a path writable by a lower-trust principal; resolves a bare name through the working directory or an influenced ENV (`LD_LIBRARY_PATH`, DLL search order). A user loading their own plugin into their own process: not a boundary |
| Check one file, map another | The loader verifies one file/metadata but maps a different image because of path resolution, file replacement, architecture slice, or embedded resources not bound to the check |
| Corrupt binary metadata | Offsets, counts, sections, relocations, symbols, bytecode, debug info trusted before range, overlap, and representation checks. Separate memory corruption from a corrupt file that is safely rejected |
| JIT, generated code | The validator, interpreter, optimizer, and generated code disagree on types, bounds, side effects, lifetimes. Compare the optimized and unoptimized paths on the same input |
| Unload, reload, teardown | Function pointers, callbacks, threads, data views that outlive a module unload or runtime reset |

## Kernel and privileged interfaces

| Layer | What to hunt |
|---|---|
| Copy from user, re-read | A syscall, ioctl, or driver derives truth from user memory, then re-reads the same modifiable address (double fetch). Copy the whole request once or re-validate the later copy. Check size, direction, permission at each copy primitive |
| Privileged object lifecycle, dispatch | Mismatched retain/release, teardown that does not drain observers, unchecked selector table index, duplicated compatibility paths missing a guard |
| Powerful interface without authz | Device node, admin socket, helper, management API validates shape but does not check the caller's permission on the resource. Permissions/sandbox outside the repo: Needs validation |

## Common moves

- Sweep patches and duplicate paths with the same source→sink shape: a check in one caller, architecture, protocol role, feature flag, or compatibility path that does not protect its siblings.
- Build a table for every parser/FFI boundary: length/type accepted, representation checked, who owns the allocation, consumer, thread, teardown. Most native bugs are a single mismatch in this table.
- Use existing corpora and small boundary fixtures; keep the raw sanitizer output and the properties of the failing input.

## Verification before reporting

1. State the real untrusted entry point and the operation that violates the bound, type, lifetime, ABI, concurrency, loader, or privilege invariant.
2. Classify the observable consequence: wrong read, wrong write, stale alias, wrong object, uninitialized output, unauthorized image load, deadlock, safe process termination. Do not claim a consequence stronger than observed.
3. Concurrency: deterministic schedule or sanitizer trace. Binary loading: show that the verified identity differs from the mapped identity and name the low-trust writer.
