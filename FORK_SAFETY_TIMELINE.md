# malloc/free Fork Safety: Version-Specific Timeline

## Executive Summary

**You were right to be skeptical** - calling malloc()/free() in pthread_atfork handlers is technically undefined behavior per POSIX, but glibc has implemented special handling to make it work. However, this implementation has had **multiple critical bugs across different versions**, and the behavior has changed significantly over time.

## Timeline of glibc Fork Safety Evolution

### Pre-2013: Early Implementation
- glibc's ptmalloc registered fork handlers using `pthread_atfork()`
- Basic locking mechanism: lock all arenas before fork, unlock after
- Hidden deadlock potential with other subsystems

### 2013: First Major Deadlock Discovered

**Bug**: [Red Hat BZ #906468](https://bugzilla.redhat.com/show_bug.cgi?id=906468) - "Deadlock in glibc between fork and malloc"
- **Filed**: January 31, 2013
- **Affected**: glibc 2.17 and earlier
- **Problem**: Lock ordering issue between malloc, fork, and stdio
  - Thread A: fork() → locks malloc → tries to lock stdio
  - Thread B: I/O operation → locks stdio → calls malloc()
  - Result: Circular dependency deadlock

### 2016: Major Fork Handler Redesign

**Patch**: [malloc: Run fork handler as late as possible [BZ #19431]](https://patchwork.ozlabs.org/project/glibc/patch/570E5362.7070300@redhat.com/)
- **Submitted**: April 13, 2016 (Florian Weimer)
- **Released**: glibc 2.24 (August 7, 2016)

**Major Changes**:
1. Renamed functions for clarity:
   - `ptmalloc_lock_all` → `__malloc_fork_lock_parent`
   - `ptmalloc_unlock_all` → `__malloc_fork_unlock_parent`
   - `ptmalloc_unlock_all2` → `__malloc_fork_unlock_child`

2. **Stopped using pthread_atfork()** for malloc's own handlers:
   - Previously: Registered via `thread_atfork()` during `ptmalloc_init()`
   - New: Called directly from `fork()` implementation
   - Why: To run "as late as possible" and avoid deadlocks with other fork handlers

3. Key insight from commit message:
   > "This commit makes the malloc subsystem available to fork handlers"

**Impact**: This change means malloc's fork handlers run **after** all user-registered pthread_atfork handlers, ensuring malloc is safe for use in those handlers.

### 2017: Fix Released to RHEL

- **Fixed In**: glibc-2.17-162.el7 (RHEL 7)
- **Released**: August 1, 2017 (RHSA-2017:1916)
- **Note**: Backported the 2016 fix to older RHEL versions

### 2018-2022: glibc 2.28 Regression

**Bug**: [BZ #24595](https://bugzilla.redhat.com/show_bug.cgi?id=1974550) - "Deadlock in atfork handler which calls dlclose"
- **Reported**: May 2019 (Jeremy Drake - OpenVPN + Gnuk smartcard hang)
- **Affected**: glibc 2.28 (released August 1, 2018) and later
- **Root Cause**: Refactoring of atfork handler management introduced new deadlock

**Technical Details** (from [Red Hat article, December 14, 2022](https://developers.redhat.com/articles/2022/12/14/how-we-addressed-unforeseen-use-case-pthreadatfork)):

The problem:
```
1. Component A registers fork handler that calls dlclose() in child
2. dlclose() tries to deregister fork handlers from Component B
3. But fork() holds a lock while executing handlers
4. dlclose() tries to acquire same lock → deadlock
```

Quote from the article:
> "calling `dlclose()` during the execution of a fork handler means that while
> _one_ handler is _running_, another needs to be removed from the list"

The glibc 2.28 implementation used "a simple lock, not allowing reentrancy. This leads to deadlocks for the aforementioned scenario."

**This means glibc 2.28 through 2.35 had a REGRESSION** - fork handlers were less safe than in 2.24-2.27!

### 2021: Malloc Hooks Removed

**Release**: glibc 2.34 (August 2, 2021)
- Removed malloc hooks (`__malloc_hook`, `__free_hook`) from API
- Fork handler implementation remained similar
- Did **NOT** fix BZ #24595 (dlclose deadlock)

### 2022: Current Implementation

**Release**: glibc 2.36 (August 3, 2022)
- **Fixed**: [BZ #24595](https://sourceware.org/pipermail/glibc-cvs/2019q3/067318.html) - atfork/dlclose deadlock
- **Solution**: Changed from array-based to doubly-linked list with lock release
  - Release lock **before** executing each handler
  - Similar to atexit() handler strategy
- **Also Fixed**: [BZ #27054] - pthread_atfork handlers calling pthread_atfork

Quote from fix:
> "we shouldn't hold implementation locks while executing an external callback"

## Version-Specific Behavior Matrix

| glibc Version | malloc/free Safe After Fork? | Known Issues | Recommended? |
|---------------|------------------------------|--------------|--------------|
| < 2.24 (pre-2016) | Mostly* | Deadlock with malloc+stdio+fork | ❌ No |
| 2.24-2.27 (2016-2018) | Yes* | None known | ✅ Yes |
| 2.28-2.35 (2018-2022) | Yes* | **REGRESSION**: dlclose in fork handler deadlocks | ⚠️ Caution |
| 2.36+ (2022+) | Yes* | None currently known | ✅ Yes |

*With important caveats (see below)

## The POSIX vs glibc Reality

### What POSIX Says

[POSIX pthread_atfork(3)](https://pubs.opengroup.org/onlinepubs/9799919799/functions/pthread_atfork.html):

> **If a fork() call in a multi-threaded process leads to a child fork handler
> calling any function that is not async-signal-safe, the behavior is undefined.**

malloc() and free() are **explicitly NOT** async-signal-safe functions.

Therefore, calling them after fork() in a multithreaded process is **undefined behavior** per POSIX.

### What glibc Implements

From [Red Hat Customer Portal article](https://access.redhat.com/articles/2921161) (analyzing glibc behavior):

> "Existing application usage requires that memory allocation functions such as
> malloc and free must work in the new subprocess created by fork, even in the
> case where the original process was multi-threaded."

glibc **chooses** to support this as an **extension beyond POSIX**, but:
- It's not guaranteed by the standard
- Not portable to other libc implementations (musl, BSD libc, etc.)
- The implementation has had multiple serious bugs
- Still technically undefined behavior that happens to work

### Why It "Works" in glibc

From [glibc arena.c source](https://codebrowser.dev/glibc/glibc/malloc/arena.c.html):

**Before fork** (prepare handler - runs in parent before fork):
```c
__malloc_fork_lock_parent() {
    // Acquire ALL malloc arena locks
    // Ensures no malloc operation is in progress
    __libc_lock_lock(list_lock);
    for (arena in all_arenas)
        __libc_lock_lock(arena->mutex);
}
```

**After fork in child**:
```c
__malloc_fork_unlock_child() {
    // REINITIALIZE locks (don't just unlock!)
    for (arena in all_arenas)
        __libc_lock_init(arena->mutex);  // Reset to unlocked state
    __libc_lock_init(list_lock);

    // Reconstruct free_list
    // Only keep main_arena attached to current thread
    // Move all others to free_list for reuse
}
```

**Critical point**: `__libc_lock_init()` **reinitializes** the mutex rather than unlocking it. This is safe because:
1. All other threads are gone in the child
2. The calling thread never held these locks (the parent's other threads did)
3. Reinitialization creates fresh, unlocked mutexes

## Why This Still Doesn't Make It Safe

### 1. Only Works for glibc's malloc

Other libraries (OpenSSL, OpenBLAS, database clients) don't have this special handling:
- Their mutexes are copied in whatever state they were in
- If another thread held a lock during fork, it's stuck forever
- No automatic reinitialization happens

### 2. Temporal Coupling

Whether your pthread_atfork handler can safely call malloc depends on:
- **When** your handler is registered (LIFO order for prepare, FIFO for child)
- **When** glibc's malloc handler runs
- **Whether** glibc even uses pthread_atfork (it stopped in 2.24!)

In glibc 2.24+, malloc handlers run **from fork() itself**, not via pthread_atfork, so they execute **after** all user pthread_atfork handlers. This makes malloc "safe" in user handlers, but it's an implementation detail.

### 3. Has Failed Multiple Times

Despite special handling, malloc fork safety has had critical bugs:
- **2013**: Deadlock with stdio (fixed 2016)
- **2019-2022**: Deadlock with dlclose (fixed 2022)

Even with glibc's extensive effort, the fork-without-exec model is fundamentally fragile.

### 4. Not Portable

This behavior is **glibc-specific**. Other implementations:
- **musl libc**: Different approach, may not work the same way
- **BSD libc**: No guarantees about malloc after fork
- **Android Bionic**: Has its own fork handling

Code that relies on malloc in fork handlers is non-portable.

## Real-World Production Issues

### strongSwan VPN (Ongoing)

[strongSwan Bug #990](https://wiki.strongswan.org/issues/990) - "unsafe use of malloc after fork":

> "The current implementation may self-deadlock under such circumstances and
> the process invoking fork will hang indefinitely."

Despite glibc's efforts, edge cases still cause production hangs.

### OpenVPN + Smartcard (glibc 2.28-2.35)

The [BZ #24595](https://developers.redhat.com/articles/2022/12/14/how-we-addressed-unforeseen-use-case-pthreadatfork) case:
- OpenVPN with Gnuk smartcard
- Fork handler called dlclose()
- Complete hang in glibc 2.28-2.35
- Affected **4 years** of glibc releases (2018-2022)

## What This Means for Gunicorn

### Your Original Question

> "does this mean that the child process is in a state that is not safe to call
> any function that's not async signal safe?"

**Yes**, according to POSIX. But glibc makes an exception for malloc/free **only**.

### For Native Extensions

When gunicorn forks workers with `--preload`:

#### ✅ Safe (due to glibc extension):
```python
def post_fork(server, worker):
    # These work because glibc special-cases malloc
    mylist = []  # Allocates memory
    mydict = {}  # Allocates memory
    connection_pool = create_pool()  # May use malloc
```

#### ❌ Unsafe (no special handling):
```python
def post_fork(server, worker):
    # If OpenSSL threads existed in parent - DEADLOCK RISK
    import requests
    requests.get("https://example.com")

    # If OpenBLAS threads existed - DEADLOCK RISK
    import numpy as np
    np.dot(matrix1, matrix2)

    # If DB connection pool threads existed - CORRUPTION RISK
    db_connection.execute("SELECT 1")
```

### Version-Specific Recommendations

**If you're on glibc < 2.24 (RHEL 6, old Ubuntu 14.04)**:
- Avoid `--preload` entirely
- Or ensure no malloc happens during fork (impractical)

**If you're on glibc 2.24-2.27 (RHEL 7, Ubuntu 16.04-17.10)**:
- `--preload` is relatively safe for malloc
- Still dangerous for other threaded libraries

**If you're on glibc 2.28-2.35 (Ubuntu 18.04-21.10, RHEL 8 without updates)**:
- **REGRESSION**: Don't call dlclose() in fork handlers
- Check if your native extensions dynamically load/unload libraries
- Safer to avoid `--preload` if using complex native extensions

**If you're on glibc 2.36+ (Ubuntu 22.04+, RHEL 9, current Debian)**:
- Most malloc/fork issues resolved
- Still must handle other libraries' threads properly
- Use post_fork hooks to reinitialize threaded components

## Best Practices (Version-Independent)

### 1. Verify Your glibc Version
```bash
ldd --version
```

### 2. Use post_fork Hooks
```python
# gunicorn_config.py
import os

def post_fork(server, worker):
    # Close inherited connections (always safe)
    from myapp.database import engine
    engine.dispose()

    # Disable threading in numeric libraries
    os.environ['OMP_NUM_THREADS'] = '1'
    os.environ['OPENBLAS_NUM_THREADS'] = '1'

    # Reinitialize anything that used threads
    # (Check your extension docs for fork-safety)
```

### 3. Test Aggressively
```bash
# Run with thread sanitizer
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libtsan.so.0 gunicorn --preload myapp:app

# Stress test fork paths
for i in {1..1000}; do
    curl http://localhost:8000 &
    kill -HUP $GUNICORN_PID  # Force worker restart
done
```

### 4. Monitor for Symptoms
- Workers hanging indefinitely
- High restart rates
- Intermittent connection errors
- Random deadlocks under load

## Conclusion

**Were you just lucky?**

Sort of. You were relying on:
1. ✅ glibc's non-standard extension (makes malloc work)
2. ⚠️ A specific glibc version without known bugs
3. ❌ Native extensions not using threads (or getting lucky with timing)
4. ❌ Undefined behavior per POSIX

**The safest approach**: Only use async-signal-safe functions in fork handlers, as POSIX requires. But if you must use malloc/free:
- Know your glibc version
- Understand its limitations
- Test thoroughly
- Have monitoring in place
- Don't rely on it for other libraries

The fact that malloc "works" is more "glibc tries really hard despite multiple bugs" than "it's actually safe."

## References

### Bug Reports & Fixes
- [Red Hat BZ #906468: Deadlock in glibc between fork and malloc](https://bugzilla.redhat.com/show_bug.cgi?id=906468) (Filed: Jan 2013, Fixed: Aug 2017)
- [BZ #19431: malloc: Run fork handler as late as possible](https://patchwork.ozlabs.org/project/glibc/patch/570E5362.7070300@redhat.com/) (Apr 2016)
- [BZ #24595: Deadlock in atfork handler which calls dlclose](https://bugzilla.redhat.com/show_bug.cgi?id=1974550) (Reported: May 2019, Fixed: glibc 2.36)
- [strongSwan Bug #990: unsafe use of malloc after fork](https://wiki.strongswan.org/issues/990)

### Release Announcements
- [glibc 2.24 released](https://sourceware.org/legacy-ml/libc-alpha/2016-03/msg00766.html) (August 7, 2016)
- [glibc 2.28 released](https://lists.gnu.org/archive/html/info-gnu/2018-08/msg00000.html) (August 1, 2018)
- [glibc 2.34 released](https://sourceware.org/pipermail/libc-alpha/2021-August/129718.html) (August 2, 2021)
- [glibc 2.36 released](https://lists.gnu.org/archive/html/info-gnu/2022-08/msg00000.html) (August 3, 2022)

### Technical Articles
- [Red Hat: Using the fork function in signal handlers](https://access.redhat.com/articles/2921161)
- [Red Hat: How we addressed an unforeseen use case in pthread_atfork()](https://developers.redhat.com/articles/2022/12/14/how-we-addressed-unforeseen-use-case-pthreadatfork) (December 14, 2022)
- [Red Hat: Why glibc 2.34 removed libpthread](https://developers.redhat.com/articles/2021/12/17/why-glibc-234-removed-libpthread)

### Source Code
- [glibc arena.c - malloc fork handlers](https://codebrowser.dev/glibc/glibc/malloc/arena.c.html)

### Standards
- [POSIX pthread_atfork specification](https://pubs.opengroup.org/onlinepubs/9799919799/functions/pthread_atfork.html)
- [POSIX signal-safety requirements](https://man7.org/linux/man-pages/man7/signal-safety.7.html)
