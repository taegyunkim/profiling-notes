# musl vs glibc: Fork Safety Comparison

## Executive Summary

**Yes, musl libc implements a similar mechanism**, but with a fundamentally different philosophy and implementation. Both make malloc/free work after fork in multithreaded programs (as a non-POSIX extension), but:

- **glibc**: Evolved organically, had multiple critical bugs, prioritizes backward compatibility
- **musl**: Designed from scratch for correctness, comprehensive lock management, cleaner architecture

## Timeline Comparison

| Date | glibc | musl |
|------|-------|------|
| Pre-2013 | Basic fork handlers, hidden bugs | Simple implementation, deadlock issues |
| 2013-2016 | Deadlock bugs discovered | No major changes |
| 2016 (glibc 2.24) | Major redesign: malloc handlers called directly from fork() | — |
| 2018-2022 (glibc 2.28-2.35) | **REGRESSION**: atfork/dlclose deadlock | — |
| Jan 2021 (musl 1.2.2) | — | **Major redesign**: Comprehensive MT-fork support |
| Aug 2022 (glibc 2.36) | Fixed atfork/dlclose deadlock | — |

## Key Differences

### 1. Design Philosophy

**glibc**:
- Evolved incrementally over decades
- Fixes bugs reactively as they're discovered
- Prioritizes backward compatibility
- "If it mostly works, ship it"

**musl**:
- Designed comprehensively from scratch
- Proactive correctness guarantees
- "Get it right or don't ship it"
- Explicitly documents trade-offs

### 2. Implementation Approach

#### glibc (2016-present)

**Strategy**: Special-case malloc only

```c
// Before fork (in parent)
__malloc_fork_lock_parent() {
    // Lock ONLY malloc arenas
    for (arena in all_arenas)
        __libc_lock_lock(arena->mutex);
}

// After fork (in child)
__malloc_fork_unlock_child() {
    // Reinitialize ONLY malloc locks
    for (arena in all_arenas)
        __libc_lock_init(arena->mutex);
}
```

**Characteristics**:
- Malloc handlers run **directly from fork()**, not via pthread_atfork
- Runs "as late as possible" to avoid deadlocks with other subsystems
- Only makes **malloc** safe; other libc components may deadlock
- Quote from glibc discussion: "doesn't make anything but malloc work in the MT-forked child"

#### musl (2021-present)

**Strategy**: Lock **all** libc subsystems

From [musl commit 167390f0](https://git.musl-libc.org/cgit/musl/commit/?id=167390f05564e0a4d3fcb4329377fd7743267560):

```c
// Before fork (in parent)
fork() {
    if (need_locks) {  // Only if multithreaded
        // Lock ALL subsystems in order
        __ldso_atfork(-1);        // Dynamic linker
        __pthread_key_atfork(-1);  // Thread-local storage
        __malloc_atfork(-1);       // Malloc
        __aio_atfork(-1);          // Async I/O
        __stdio_atfork(-1);        // Standard I/O
        __locale_atfork(-1);       // Locale
        __random_atfork(-1);       // PRNG
        __vmlock_atfork(-1);       // VM locks
        // ... more subsystems
    }

    pid = _Fork();  // Actual syscall

    if (pid == 0) {  // Child
        // Reset ALL locks
        __vmlock_atfork(0);
        __random_atfork(0);
        __locale_atfork(0);
        // ... etc
    } else {  // Parent
        // Unlock ALL locks
        __vmlock_atfork(1);
        __random_atfork(1);
        // ... etc
    }
}
```

**Characteristics**:
- Comprehensive lock management across **all** libc subsystems
- Each subsystem exports locks via `fork_impl.h`
- Makes **all** standard library functions safe in child
- Only locks if parent is multithreaded (preserves async-signal-safety for single-threaded)

### 3. The _Fork vs fork Split

**musl's Innovation**: Separate functions for different use cases

#### `_Fork()` - Async-Signal-Safe
```c
// Raw syscall wrapper
// Safe to call from signal handlers
// Does NOT run pthread_atfork handlers
// Does NOT lock anything
pid_t _Fork(void) {
    return syscall(SYS_clone, SIGCHLD, 0);
}
```

#### `fork()` - Full Library Support
```c
// Full featured, NOT async-signal-safe
// Runs pthread_atfork handlers
// Locks all libc subsystems
// Makes child environment fully usable
pid_t fork(void) {
    // Complex lock management shown above
}
```

From [musl 1.2.2 release notes](https://musl.libc.org/releases.html):
> "The release adds the `_Fork` function from the upcoming edition of POSIX and
> takes advantage of the interpretation dropping the async-signal-safety
> requirement from `fork` to provide a consistent execution environment."

**Key insight**: POSIX now allows `fork()` to NOT be async-signal-safe, so musl makes it comprehensive but non-AS-safe. For signal handlers, use `_Fork()` instead.

**glibc**: Doesn't have this split. `fork()` tries to be both, succeeding at neither perfectly.

### 4. Lock Ordering and Deadlock Avoidance

#### glibc's Approach

**Problem**: Lock ordering between subsystems causes deadlocks

From [Red Hat BZ #906468](https://bugzilla.redhat.com/show_bug.cgi?id=906468) (2013):
- Thread A: fork() → locks malloc → tries to lock stdio
- Thread B: I/O operation → locks stdio → calls malloc
- **Result**: Deadlock

**Solution** (2016): Run malloc fork handler "as late as possible"
- Moved malloc locking directly into fork() implementation
- Happens after other locks are released
- Band-aid solution, doesn't prevent all deadlocks

#### musl's Approach

**Solution**: Explicit, documented lock ordering

From [musl MT fork discussion](https://www.openwall.com/lists/musl/2020/11/09/6):
> "we do the malloc_atfork code last so that the lock isn't held while other
> libc components' locks are being taken"

**Key differences**:
1. **Comprehensive**: All subsystems participate
2. **Ordered**: Explicit acquisition/release order
3. **Documented**: Lock order defined in `fork_impl.h`
4. **Verified**: Single code path, easier to reason about

Quote from discussion:
> glibc "works" primarily because fork operations rarely coincide with
> concurrent subsystem usage—"only with 0.01% probability."

musl prioritizes correctness over "usually works."

## Specific Technical Differences

### malloc Lock Management

#### glibc
- Uses arena-based locking (multiple locks)
- Locks acquired in `__malloc_fork_lock_parent()`
- Child reinitializes via `__libc_lock_init()`
- **Only malloc is handled**

#### musl
- Uses per-size-class locks (fine-grained)
- Locks acquired in `__malloc_atfork(-1)`
- Child resets via `__malloc_atfork(0)`
- **Part of comprehensive subsystem management**

### pthread_atfork Integration

#### glibc (2016+)
- Malloc **does not** use `pthread_atfork()`
- Called directly from fork() implementation
- User atfork handlers run before malloc locking
- Ensures malloc is available in user handlers

#### musl
- Malloc **uses** the unified atfork system
- Integrated with all other subsystems
- Explicit ordering via `fork_impl.h`
- More predictable behavior

### Application malloc Interposition

**Critical issue**: What if an application provides its own malloc?

#### glibc Behavior
From [musl discussion](https://www.openwall.com/lists/musl/2020/11/09/6):
> "glibc can only take the internal locks of its own malloc implementation;
> any interposed malloc has the same issue as musl's after fork."

If you use a custom malloc (jemalloc, tcmalloc, etc.):
- glibc's fork handlers don't help
- Custom malloc must register its own pthread_atfork handlers
- Still vulnerable to deadlocks

#### musl Behavior
Same issue, but more honestly documented:
> "Calling malloc after fork in a pthread_atfork handler in the child process
> is undefined behavior when the parent is multi-threaded."

musl doesn't pretend interposed malloc is safe.

## Real-World Issues

### glibc Issues (Still Occurring)

1. **strongSwan VPN** ([Bug #990](https://wiki.strongswan.org/issues/990)):
   - Deadlocks with malloc after fork
   - Even with glibc's special handling
   - "may self-deadlock and hang indefinitely"

2. **OpenVPN + Smartcard** (2019-2022):
   - glibc 2.28-2.35 regression
   - Fork handler called dlclose()
   - Complete hang for 4 years

### musl Issues (Historical)

1. **QEMU User Mode** ([Ubuntu Bug #1813398](https://bugs.launchpad.net/qemu/+bug/1813398)):
   - Before musl 1.2.2
   - Malloc after fork in multithreaded processes
   - Fixed in 1.2.2 (January 2021)

2. **Perl Testsuite** ([Perl Issue #18159](https://github.com/Perl/perl5/issues/18159)):
   - Deadlocks in musl 5.32.0 testsuite
   - Fork-related synchronization issues
   - Addressed in later releases

**Key difference**: musl's issues were resolved comprehensively with 1.2.2, while glibc continues to have sporadic problems.

## Benchmarks and Performance

### Memory Overhead

**glibc**:
- Arena-based allocation: higher memory usage
- Multiple arenas per thread
- Trade-off for better multithreaded performance

**musl**:
- Fine-grained locking: lower memory usage
- More CPU overhead for locking
- Trade-off for correctness and simplicity

### Fork Performance

**glibc** (multithreaded parent):
- Fast: Only locks malloc
- Risk: Other subsystems may deadlock

**musl** (multithreaded parent):
- Slower: Locks all subsystems
- Safe: Comprehensive consistency guarantee

**musl** (single-threaded parent):
- Very fast: Skips all locking
- Preserves async-signal-safety

## Portability Considerations

### What Works Where

| Operation | glibc 2.36+ | musl 1.2.2+ | POSIX Guarantee |
|-----------|-------------|-------------|-----------------|
| malloc() after fork | ✅ Yes* | ✅ Yes* | ❌ No (undefined) |
| stdio after fork | ⚠️ Maybe | ✅ Yes | ❌ No (undefined) |
| dlopen() after fork | ❌ Risky | ✅ Yes | ❌ No (undefined) |
| locale after fork | ⚠️ Maybe | ✅ Yes | ❌ No (undefined) |
| fork() from signal handler | ❌ No | Use _Fork() | ❌ No (as of POSIX-2024) |
| _Fork() from signal handler | N/A | ✅ Yes | ✅ Yes (POSIX-2024) |

*Only if multithreaded parent; not guaranteed by POSIX

### Code Portability

If you write code that works on musl, it will likely work on glibc.
If you write code that works on glibc, it may fail on musl (before 1.2.2) or other libcs.

**Recommendation**: Follow POSIX strictly, or test on musl for stricter correctness.

## Implications for Gunicorn

### With glibc

**What works**:
```python
def post_fork(server, worker):
    # Safe: glibc special-cases malloc
    pool = create_connection_pool()
    cache = {}
    buffer = bytearray(1024)
```

**What may deadlock**:
```python
def post_fork(server, worker):
    # Risky: stdio locks might be held
    sys.stdout.write("Worker started\n")

    # Risky: locale functions
    locale.setlocale(locale.LC_ALL, 'en_US.UTF-8')

    # Risky: dynamic loading
    import importlib
    importlib.import_module('mymodule')
```

### With musl 1.2.2+

**What works**:
```python
def post_fork(server, worker):
    # Safe: musl locks ALL subsystems
    pool = create_connection_pool()
    sys.stdout.write("Worker started\n")  # Actually safe!
    locale.setlocale(locale.LC_ALL, 'en_US.UTF-8')  # Safe!
    import importlib  # Safe!
```

**What still doesn't work**:
```python
def post_fork(server, worker):
    # Still unsafe: OpenSSL threads from parent
    import requests
    requests.get("https://api.example.com")

    # Still unsafe: OpenBLAS threads from parent
    import numpy as np
    result = np.dot(matrix1, matrix2)
```

**Key point**: musl makes **libc** safe, but not **native extensions**. You still need post_fork hooks for OpenSSL, OpenBLAS, database clients, etc.

## Design Philosophy Comparison

### glibc: Pragmatic Evolution

**Strengths**:
- Extensive real-world testing (decades of production use)
- Performance optimizations for common cases
- Backward compatibility

**Weaknesses**:
- Reactive bug fixing ("patch what breaks")
- Complex codebase with historical baggage
- Deadlocks still occur in edge cases

**Quote**: "works primarily because fork operations rarely coincide with concurrent subsystem usage—only with 0.01% probability"

Translation: "We know it's broken, but it doesn't hit often enough to care."

### musl: Correctness First

**Strengths**:
- Comprehensive correctness guarantees
- Clean, auditable codebase
- Explicit documentation of limitations

**Weaknesses**:
- Smaller installed base (less battle-tested)
- Sometimes slower (correctness over speed)
- Stricter adherence to standards (can break buggy code)

**Quote**: "we shouldn't hold implementation locks while executing an external callback"

Translation: "If we can't guarantee correctness, we won't do it."

## Recommendations

### For Application Developers

1. **Check your libc version**:
   ```bash
   ldd --version  # glibc
   ldd 2>&1 | grep musl  # musl
   ```

2. **If using glibc < 2.36**:
   - Avoid `--preload` if possible
   - Implement comprehensive post_fork hooks
   - Test extensively under load

3. **If using glibc 2.36+**:
   - `--preload` is relatively safe for malloc
   - Still need post_fork hooks for native extensions
   - Monitor for rare deadlocks

4. **If using musl 1.2.2+**:
   - `--preload` is safer for all libc operations
   - Still need post_fork hooks for native extensions
   - Trust the guarantees more

### For Library Developers

1. **Don't rely on malloc working after fork**:
   - It works on modern glibc/musl, but it's undefined per POSIX
   - Other platforms (BSD, macOS, embedded) may not support it

2. **If you use threads, provide fork-safety hooks**:
   ```c
   void mylib_reinit_after_fork(void) {
       // Detect fork via PID check
       // Reinitialize thread pools
       // Reset mutexes
   }
   ```

3. **Document fork-safety characteristics**:
   ```c
   /**
    * Fork safety: This library is NOT fork-safe when using
    * multithreading. Call mylib_reinit_after_fork() in the
    * child process after fork().
    */
   ```

## Conclusion

**Does musl implement similar mechanisms?**

**Yes**, but better:
- ✅ More comprehensive (all libc subsystems, not just malloc)
- ✅ Better designed (explicit lock ordering, cleaner architecture)
- ✅ Fewer bugs (comprehensive approach prevents classes of issues)
- ✅ Honest documentation (explicitly states limitations)
- ✅ Separates concerns (_Fork for signals, fork for full environment)

**Trade-offs**:
- ⚠️ Slower fork in multithreaded programs (more locks to acquire)
- ⚠️ Smaller installed base (less real-world testing)
- ⚠️ Stricter correctness (may expose bugs in applications)

**For Gunicorn with `--preload`**:
- **glibc**: Mostly works, occasional edge-case deadlocks
- **musl 1.2.2+**: Should work better, but still need to handle native extensions

**Bottom line**: Neither libc can solve the fundamental issue that **native extension threads don't survive fork()**. You still need post_fork hooks to reinitialize OpenSSL, OpenBLAS, database clients, etc. But musl makes the libc part more reliable.

## References

### musl Resources
- [musl 1.2.2 release notes](https://musl.libc.org/releases.html) (January 15, 2021)
- [musl MT fork commit](https://git.musl-libc.org/cgit/musl/commit/?id=167390f05564e0a4d3fcb4329377fd7743267560)
- [musl fork.c source](https://git.musl-libc.org/cgit/musl/tree/src/process/fork.c)
- [musl MT fork discussion](https://www.openwall.com/lists/musl/2020/11/09/6) (November 2020)
- [musl design concepts](https://wiki.musl-libc.org/design-concepts.html)

### glibc Resources
- [Red Hat BZ #906468](https://bugzilla.redhat.com/show_bug.cgi?id=906468) (2013 deadlock)
- [glibc BZ #19431 patch](https://patchwork.ozlabs.org/project/glibc/patch/570E5362.7070300@redhat.com/) (2016 redesign)
- [glibc BZ #24595](https://bugzilla.redhat.com/show_bug.cgi?id=1974550) (2019 regression)
- [Red Hat: pthread_atfork unforeseen use case](https://developers.redhat.com/articles/2022/12/14/how-we-addressed-unforeseen-use-case-pthreadatfork)

### Issues
- [QEMU malloc after fork bug](https://bugs.launchpad.net/qemu/+bug/1813398)
- [Perl deadlock with musl](https://github.com/Perl/perl5/issues/18159)
- [strongSwan malloc fork issue](https://wiki.strongswan.org/issues/990)
