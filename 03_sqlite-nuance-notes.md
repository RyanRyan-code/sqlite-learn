# SQLite nuance notes

Small SQLite details that are easy to misunderstand but useful while reading the
source.

## assert, ALWAYS, and NEVER

This section captures the difference between SQLite's plain `assert()` checks
and the `ALWAYS()` / `NEVER()` macros.

The short version:

```text
assert(X)      -> debug-only invariant check
ALWAYS(X)      -> release-time defensive check, debug assert if violated
NEVER(X)       -> release-time defensive check, debug assert if violated
```

`assert(X)` is not rewritten into `ALWAYS(X)` or `NEVER(X)`. They are separate
tools with different meanings.

### assert comes from C

SQLite gets `assert()` from the standard C header:

```c
#include <assert.h>
```

The standard `assert()` macro is controlled by `NDEBUG`. If `NDEBUG` is defined,
`assert(X)` becomes a no-op.

SQLite makes release builds opt out of assertions by default:

```c
#if !defined(NDEBUG) && !defined(SQLITE_DEBUG)
# define NDEBUG 1
#endif
#if defined(NDEBUG) && defined(SQLITE_DEBUG)
# undef NDEBUG
#endif
```

So in a normal SQLite release build:

```text
SQLITE_DEBUG is not defined
  -> SQLite defines NDEBUG
  -> assert(X) is disabled
```

In a debug build:

```text
SQLITE_DEBUG is defined
  -> SQLite undefines NDEBUG
  -> assert(X) is active
```

An `assert(X)` means SQLite believes `X` is a proven invariant. It is useful for
debugging and reasoning, but it is not part of the release-build defense path.

### ALWAYS and NEVER are SQLite macros

`ALWAYS(X)` and `NEVER(X)` are defined by SQLite itself in `sqliteInt.h`:

```c
#if defined(SQLITE_OMIT_AUXILIARY_SAFETY_CHECKS)
# define ALWAYS(X)      (1)
# define NEVER(X)       (0)
#elif !defined(NDEBUG)
# define ALWAYS(X)      ((X)?1:(assert(0),0))
# define NEVER(X)       ((X)?(assert(0),1):0)
#else
# define ALWAYS(X)      (X)
# define NEVER(X)       (X)
#endif
```

In a normal release build, they are pass-through expressions:

```c
#define ALWAYS(X) (X)
#define NEVER(X)  (X)
```

So this:

```c
if( ALWAYS(p!=0) ){
  use(p);
}
```

becomes this:

```c
if( p!=0 ){
  use(p);
}
```

That check still runs in release builds. It is not a dummy marker.

The point is defensive behavior: SQLite believes the condition should always be
true, or should never be true, but it still wants the release binary to avoid
doing something unsafe if that belief is wrong.

### Why assert(0),0?

The debug definitions use C's comma operator:

```c
(assert(0), 0)
```

That means:

```text
run assert(0)
then evaluate the whole expression as 0
```

In an active assertion build, `assert(0)` normally prints an assertion failure
and calls `abort()`, so the program stops there. The trailing `0` is not there
because SQLite expects execution to continue after the failed assertion. It is
there so the macro still has a valid expression value and type shape.

So:

```c
#define ALWAYS(X) ((X)?1:(assert(0),0))
```

means:

```text
if X is true:
  return 1
else:
  fail an assertion
  return 0
```

And:

```c
#define NEVER(X) ((X)?(assert(0),1):0)
```

means:

```text
if X is true:
  fail an assertion
  return 1
else:
  return 0
```

The value after the comma keeps the macro usable as an expression in an `if`
condition.

### Performance model

`assert(X)` has no release-build runtime cost because it is disabled when
`NDEBUG` is defined.

`ALWAYS(X)` and `NEVER(X)` can have release-build runtime cost because they still
evaluate `X`. SQLite uses them selectively where the defensive check is worth
keeping.

The mental split is:

```text
assert(X)
  "This is proven. Remove the check in release builds."

ALWAYS(X)
  "This should be true, but keep the runtime guard in release builds."

NEVER(X)
  "This should be false, but keep the runtime guard in release builds."
```

Coverage and mutation-testing builds are special: SQLite can force
`ALWAYS(X)` to `1` and `NEVER(X)` to `0` so unreachable defensive branches do not
count against coverage.

Source:

- `sqlite/src/sqliteInt.h:438` explains the `NDEBUG` / `SQLITE_DEBUG` relation.
- `sqlite/src/sqliteInt.h:449` defines `NDEBUG` for normal builds.
- `sqlite/src/sqliteInt.h:453` undefines `NDEBUG` for `SQLITE_DEBUG`.
- `sqlite/src/sqliteInt.h:533` defines `ALWAYS(X)` for omitted auxiliary safety
  checks.
- `sqlite/src/sqliteInt.h:536` defines debug-build `ALWAYS(X)`.
- `sqlite/src/sqliteInt.h:537` defines debug-build `NEVER(X)`.
- `sqlite/src/sqliteInt.h:539` defines release-build `ALWAYS(X)`.
- `sqlite/src/sqliteInt.h:540` defines release-build `NEVER(X)`.
- `sqlite/src/sqliteInt.h:637` includes `<assert.h>`.

## C memory ownership while reading SQLite

SQLite is C code, not C++ code, so there is no language-level RAII inside the
SQLite implementation. If SQLite allocates heap memory, SQLite code must arrange
for the matching cleanup path.

A local pointer variable and the heap block it points to have different
lifetimes:

```c
void example(void){
  int *p = malloc(100 * sizeof(int));
}
```

When `example()` returns, the local variable `p` is gone. The heap allocation is
not automatically freed. If nothing saved the pointer and no `free(p)` happens,
the process has leaked that block until process exit.

The mental split is:

```text
stack/local variable:
  lifetime is tied to the function scope

heap allocation:
  lifetime is tied to an explicit free path or process exit
```

That matters when reading SQLite because many public APIs hide allocation behind
opaque handles. Caller code often does not call `malloc()` directly:

```c
sqlite3_stmt *pStmt = 0;
sqlite3_prepare_v2(db, zSql, -1, &pStmt, 0);
sqlite3_step(pStmt);
sqlite3_finalize(pStmt);
```

`sqlite3_prepare_v2()` takes a `sqlite3_stmt **` output parameter. Passing a
pointer-to-pointer lets SQLite store the resulting handle in the caller's
variable, but it does not remove the need for storage. The prepared statement
object itself is created inside SQLite and must outlive the prepare call so the
caller can later step it, reset it, bind values, read columns, and finalize it.

The shape is:

```text
caller stack:
  sqlite3_stmt *pStmt

SQLite heap/internal allocation:
  Vdbe object, exposed publicly as sqlite3_stmt

ownership pair:
  sqlite3_prepare_v2() creates/returns the handle
  sqlite3_finalize() destroys it
```

So this is not garbage collection. It is disciplined ownership through API
pairs. In C++ wrapper libraries, destructors often call these cleanup APIs
automatically, which makes the calling code look RAII-style. The underlying
SQLite C API is still explicit: finalize statements, close database handles, and
free SQLite-owned returned buffers with the documented matching function.

The rule of thumb is:

```text
fixed-size and short-lived:
  stack storage plus output parameters can be enough

variable-size or must outlive the function:
  someone needs heap allocation, and someone needs the matching cleanup path
```

Source:

- `sqlite/src/prepare.c:688` implements the internal `sqlite3Prepare()`.
- `sqlite/src/prepare.c:695` receives `sqlite3_stmt **ppStmt` as an output
  parameter.
- `sqlite/src/prepare.c:941` implements `sqlite3_prepare_v2()`.
- `sqlite/src/prepare.c:957` calls `sqlite3LockAndPrepare()` with `ppStmt`.
- `sqlite/src/vdbeaux.c:11` says `Vdbe` is known externally as
  `sqlite3_stmt`.
- `sqlite/src/vdbeaux.c:25` implements `sqlite3VdbeCreate()`.
- `sqlite/src/vdbeaux.c:28` allocates the `Vdbe` with
  `sqlite3DbMallocRawNN()`.
- `sqlite/src/vdbeapi.c:105` implements `sqlite3_finalize()`.
- `sqlite/src/vdbeapi.c:119` calls `sqlite3VdbeDelete()`.
- `sqlite/src/vdbeaux.c:3778` implements `sqlite3VdbeDelete()`.
- `sqlite/src/vdbeaux.c:3790` frees the `Vdbe` with `sqlite3DbNNFreeNN()`.
- `sqlite/src/sqlite.h.in:3251` documents `sqlite3_malloc()`.
- `sqlite/src/sqlite.h.in:3262` documents `sqlite3_free()`.

## BtreeEnter, mutexes, and file locks

`sqlite3BtreeEnter(Btree *p)` sounds like it might lock the database, but it
does not. It enters a btree-layer critical section for a shared in-memory object.

The implementation lives in `btmutex.c`:

```c
void sqlite3BtreeEnter(Btree *p){
  ...
  if( !p->sharable ) return;
  p->wantToLock++;
  if( p->locked ) return;
  btreeLockCarefully(p);
}
```

The actual mutex it enters is:

```c
p->pBt->mutex
```

where `p->pBt` is a `BtShared`.

### Btree vs BtShared

A database connection has a `Btree` handle for each attached database file:

```text
sqlite3 connection
  -> Btree for main
  -> Btree for temp
  -> Btree for each ATTACHed database
```

The `Btree` object is per connection. It points at a `BtShared` object:

```c
struct Btree {
  sqlite3 *db;
  BtShared *pBt;
  u8 sharable;
  u8 locked;
  int wantToLock;
  ...
};
```

`BtShared` contains the backend state that can be shared by multiple `Btree`
handles in shared-cache mode:

```c
struct BtShared {
  Pager *pPager;
  sqlite3 *db;
  BtCursor *pCursor;
  MemPage *pPage1;
  void *pSchema;
  sqlite3_mutex *mutex;
  ...
};
```

The name `BtShared` means "the shareable part of a btree handle". It does not
mean all connections to the same database file always share one `BtShared`.

### `Btree.pNext` and `pPrev` do not describe tree pages

The name `Btree` is easy to misread. A `Btree` object is not a node in an
on-disk b-tree. It is a per-connection handle for one database file. One
connection may therefore have several such handles:

```text
sqlite3 connection
  |-- Btree for main.db
  |-- Btree for aux.db
  `-- Btree for another attached database
```

Its `pNext` and `pPrev` fields link the **sharable `Btree` handles belonging to
that same connection**:

```c
Btree *pNext;  /* Other sharable Btrees from the same db connection */
Btree *pPrev;
```

SQLite keeps this handle list sorted by `pBt` address. `sqlite3BtreeEnter()`
uses that ordering when acquiring multiple `BtShared.mutex` objects, so every
connection takes them in a consistent order and avoids deadlock. These pointers
are connection and locking bookkeeping; they say nothing about b-tree shape.

Actual tree relationships are represented differently:

```text
interior page bytes  -> child page numbers in cells + rightmost child
leaf page bytes      -> records only; no sibling next/previous pointers
BtCursor             -> apPage[]/aiIdx[] stack used to navigate through parents
```

Therefore a `MemPage` does not need general tree `pNext`/`pPrev` fields. Some
special page formats do encode their own links in page bytes—for example, an
overflow page's next-page number and a freelist trunk's next-trunk number—but
those are format-specific chains, not the `Btree` handle list.

### Pointer fields that are really linked-list heads

This SQLite field is a good C-language trap:

```c
BtCursor *pCursor;    /* A list of all open cursors */
```

The type only says "`pCursor` points to a `BtCursor`". C does not know whether
that address is one object, the first element of an array, or the first node in
a linked list. The surrounding code defines the convention.

Here, `pCursor` is the head of a linked list. Each cursor contains another
pointer field:

```c
struct BtCursor {
  ...
  BtCursor *pNext;    /* Forms a linked list of all cursors */
  ...
};
```

Code walks the list by following `pNext` until it reaches a null pointer:

```c
for(p=pBt->pCursor; p; p=p->pNext){
  ...
}
```

So the end marker is not hidden in the pointer type. It is the explicit
`NULL`/`0` value stored in the final node's `pNext`.

This is different from an actual fixed-size array declaration:

```c
BtCursor cursors[10];
```

and different from a function parameter written with array syntax:

```c
void f(BtCursor cursors[], size_t n);
```

In a function parameter, `BtCursor cursors[]` adjusts to `BtCursor *cursors`.
The function still needs `n`, a sentinel, or some other convention to know how
many elements are valid.

### Connections do not always share BtShared

In normal private-cache use, two connections to the same database file usually
look like this:

```text
conn1 -> Btree A -> BtShared A -> Pager A -> file.db
conn2 -> Btree B -> BtShared B -> Pager B -> file.db
```

They do not share `BtShared`. They coordinate through pager/VFS file locks.

In shared-cache mode, they can look like this:

```text
conn1 -> Btree A \
                  -> BtShared X -> Pager X -> file.db
conn2 -> Btree B /
```

That is when `BtShared.mutex` matters for cross-connection shared memory.

`sqlite3BtreeOpen()` only searches for an existing `BtShared` when the open uses
`SQLITE_OPEN_SHAREDCACHE`:

```c
if( vfsFlags & SQLITE_OPEN_SHAREDCACHE ){
  p->sharable = 1;
  ...
  for(pBt=sqlite3SharedCacheList; pBt; pBt=pBt->pNext){
    if( same filename and same VFS ){
      p->pBt = pBt;
      pBt->nRef++;
      break;
    }
  }
}
```

If no existing `BtShared` is reused, SQLite allocates a new one and opens a new
pager:

```c
pBt = sqlite3MallocZero( sizeof(*pBt) );
sqlite3PagerOpen(..., &pBt->pPager, ...);
p->pBt = pBt;
```

### Mutex vs file lock

A mutex and a database file lock are different tools.

Mutex:

```text
Protects in-memory C data structures inside one process.
Usually comes from a threading library or platform primitive.
SQLite wraps it as sqlite3_mutex.
```

File lock:

```text
Protects the actual database file across connections and processes.
Implemented through the pager and VFS.
```

SQLite mutex types are things like:

```text
SQLITE_MUTEX_FAST
SQLITE_MUTEX_RECURSIVE
SQLITE_MUTEX_STATIC_MAIN
SQLITE_MUTEX_STATIC_OPEN
```

They are not the same as database file lock levels:

```text
NO_LOCK
SHARED_LOCK
RESERVED_LOCK
PENDING_LOCK
EXCLUSIVE_LOCK
```

So this:

```c
sqlite3_mutex_enter(pBt->mutex);
```

means:

```text
enter this in-memory critical section
```

It does not mean:

```text
take a SHARED/RESERVED/EXCLUSIVE database file lock
```

### What sqlite3BtreeEnter protects

`sqlite3BtreeEnter(p)` protects shared btree-layer memory such as:

```text
BtShared.pCursor
BtShared.pPage1
BtShared.pSchema
BtShared.inTransaction
BtShared.pLock
BtShared.pPager access from the btree layer
```

The mental model is:

```text
sqlite3BtreeEnter(p)
  "I am about to read or mutate shared in-memory btree state."
```

It is not:

```text
sqlite3BtreeEnter(p)
  "I am locking the database file."
```

### Where the file gets locked

The database file is locked in the pager/VFS path.

For example, the pager helper says:

```c
static int pagerLockDb(Pager *pPager, int eLock){
  ...
  rc = pPager->noLock ? SQLITE_OK : sqlite3OsLock(pPager->fd, eLock);
  ...
}
```

`eLock` is one of the database file lock levels:

```text
SHARED_LOCK
RESERVED_LOCK
EXCLUSIVE_LOCK
```

SQLite describes those levels like this:

```text
SHARED:
  any number of processes may hold a SHARED lock simultaneously

RESERVED:
  a single process may hold a RESERVED lock; other processes may still hold
  and obtain SHARED locks

PENDING:
  existing SHARED locks may persist, but no new SHARED locks may be obtained

EXCLUSIVE:
  excludes all other locks
```

So a read path can begin with:

```text
btree
  -> sqlite3PagerSharedLock()
    -> pager_wait_on_lock(..., SHARED_LOCK)
      -> pagerLockDb(..., SHARED_LOCK)
        -> sqlite3OsLock(..., SHARED_LOCK)
```

And a write path can later involve:

```text
RESERVED_LOCK
EXCLUSIVE_LOCK
```

depending on transaction state, rollback journal mode, WAL mode, and commit
phase.

### Why rollback-journal mode has more reader/writer blocking

Rollback-journal mode writes changed pages back into the main database file.
Before doing that safely, SQLite first saves the old page images in the
rollback journal. The journal protects recovery, but the main database file is
still the file readers are using.

That is why rollback-journal mode needs stronger coordination at commit time:

```text
reader:
  holds SHARED_LOCK to read a stable database image

writer:
  may hold RESERVED_LOCK while preparing changes
  must prevent new readers with PENDING_LOCK near commit
  must obtain EXCLUSIVE_LOCK before writing changed pages to the database file
```

The conflict is not "a writer exists", exactly. A writer can hold
`RESERVED_LOCK` while existing and new readers still take `SHARED_LOCK`. The
blocking happens when the writer needs to finish the transaction and update the
main database file without readers observing a half-old, half-new image.

WAL mode moves the dangerous overlap to a different shape. The writer appends
new page versions to the WAL file instead of overwriting the database file
during the transaction:

```text
reader:
  keeps reading from its snapshot of the database plus WAL frames up to its
  snapshot point

writer:
  appends newer frames to the WAL
```

So readers and one writer can overlap better in WAL mode. SQLite still allows
only one writer at a time, and checkpoints still have their own coordination
rules, but normal read/write concurrency is better because the writer is not
forcing the main database image to change underneath active readers.

### Why BtreeEnter remains even though shared cache is discouraged

Shared-cache mode is discouraged for most application use, but SQLite still
supports it. The btree mutex code remains for that configuration.

SQLite avoids the real mutex cost when shared cache is not used:

```c
if( !p->sharable ) return;
```

So in normal private-cache use:

```text
p->sharable == 0
  -> sqlite3BtreeEnter(p) returns
  -> no BtShared mutex is entered
```

If SQLite is compiled with shared-cache support omitted entirely, `btree.h`
turns the enter/leave calls into macro no-ops:

```c
#define sqlite3BtreeEnter(X)
#define sqlite3BtreeEnterAll(X)
```

The cost model is:

```text
SQLITE_OMIT_SHARED_CACHE:
  sqlite3BtreeEnter() is compiled away

shared-cache supported but not used:
  small function call and branch, no real BtShared mutex lock

shared-cache used:
  actual BtShared mutex lock
```

In debug builds, SQLite may mark persistent btrees sharable even when the open
did not request shared cache. That exercises the locking code and helps
`assert(sqlite3_mutex_held(...))` catch mutex-discipline bugs.

Source:

- `sqlite/src/btmutex.c:1` says the file implements mutexes on `Btree` objects.
- `sqlite/src/btmutex.c:20` enters `p->pBt->mutex` and sets `p->locked`.
- `sqlite/src/btmutex.c:56` explains the recursive interface over a
  non-recursive mutex.
- `sqlite/src/btmutex.c:71` implements `sqlite3BtreeEnter()`.
- `sqlite/src/btmutex.c:88` returns immediately when `p->sharable` is false.
- `sqlite/src/btmutex.c:143` implements `sqlite3BtreeLeave()`.
- `sqlite/src/btreeInt.h:345` defines `struct Btree`.
- `sqlite/src/btreeInt.h:425` defines `struct BtShared`.
- `sqlite/src/btree.c:2591` starts the shared-cache lookup path.
- `sqlite/src/btree.c:2595` checks `SQLITE_OPEN_SHAREDCACHE`.
- `sqlite/src/btree.c:2627` scans `sqlite3SharedCacheList`.
- `sqlite/src/btree.c:2677` allocates a new `BtShared`.
- `sqlite/src/btree.c:2682` opens a new pager.
- `sqlite/src/btree.h:390` declares `sqlite3BtreeEnter()` when shared cache is
  compiled in.
- `sqlite/src/btree.h:396` makes `sqlite3BtreeEnter()` a no-op macro when
  shared cache is omitted.
- `sqlite/src/os.h:83` explains database file lock levels.
- `sqlite/src/pager.c:1157` calls `sqlite3OsLock()` for file locking.

## Mutex mental model

A mutex is not a database lock and not a file lock. It is a synchronization
object used by threads to coordinate access to shared memory.

The usual shape is:

```c
sqlite3_mutex_enter(mutex);

/* critical section: read or mutate shared memory here */

sqlite3_mutex_leave(mutex);
```

`enter` means "acquire the mutex" or "enter the critical section protected by
this mutex". It does not mean entering, reading, or using the struct itself.

Conceptually:

```text
sqlite3_mutex_enter(m)
  if no thread owns m:
    mark m as owned by this thread
    continue
  otherwise:
    wait until the owner releases m

sqlite3_mutex_leave(m)
  release ownership of m
  wake a waiting thread if needed
```

The important operation is atomic:

```text
check whether the mutex is free + claim it
```

Those two steps must happen as one indivisible operation. Otherwise two threads
could both see "free" and both enter the protected code.

### How mutex waiting works

`sqlite3_mutex_enter()` does not mean SQLite runs a manual busy loop like this:

```c
while( mutex_is_locked ){
  /* keep checking */
}
```

SQLite delegates the real wait to the configured mutex implementation. On the
Unix implementation used on Linux/macOS, that eventually reaches:

```c
pthread_mutex_lock(&p->mutex);
```

The usual implementation shape is:

```text
fast path:
  try to claim the lock with an atomic operation in user space
  if it succeeds, enter the critical section without a syscall

slow path:
  if another thread owns the mutex, ask the OS/threading runtime to block this
  thread
  the waiting thread is descheduled and does not consume CPU polling the mutex

unlock path:
  release the lock
  if there are waiters, wake one or more blocked threads through the runtime/OS
```

On Linux, pthread mutexes are commonly implemented with futexes: the uncontended
path is a user-space atomic operation, and the contended path can park the
thread in a kernel wait queue until another thread wakes it. That futex detail is
below SQLite; SQLite just calls the pthread API.

There can still be loops inside a real mutex implementation, but the important
distinction is where the time is spent:

```text
spinning:
  repeatedly check the lock while still running on CPU

blocking:
  sleep in the OS/runtime and retry only after being woken
```

General-purpose mutexes normally avoid burning CPU for long waits. Some
implementations spin briefly before sleeping, and explicit spinlocks are a
different primitive intended for very short critical sections.

A blocked mutex waiter is also not a Java-style listener object. A listener is
usually just an object in memory whose method is called directly when some other
code decides to notify it:

```java
for (Listener l : listeners) {
  l.onChange(newState);
}
```

That loop is ordinary application dispatch. The listener is not necessarily a
parked thread. A blocked mutex waiter is a thread that tried to acquire a lock
and could not continue until the runtime/OS makes it runnable again.

### What a thread "holding" a mutex means

A thread does not literally hold a struct. "Thread A holds mutex M" means:

```text
Thread A successfully acquired M and has not released it yet.
```

The mutex object is just memory plus OS/runtime bookkeeping. It does not
automatically protect nearby variables or fields. The protection comes from a
codebase convention:

```text
before touching this shared state, acquire this specific mutex
```

For example:

```c
sqlite3_mutex_enter(pBt->mutex);

/* safe to touch selected BtShared fields here */

sqlite3_mutex_leave(pBt->mutex);
```

This only works if all code that touches those selected `BtShared` fields follows
the same rule.

### SQLite's mutex abstraction

SQLite wraps platform mutex APIs behind `sqlite3_mutex`.

The public wrapper is:

```c
void sqlite3_mutex_enter(sqlite3_mutex *p){
  if( p ){
    sqlite3GlobalConfig.mutex.xMutexEnter(p);
  }
}
```

The configured mutex implementation is a table of function pointers:

```c
struct sqlite3_mutex_methods {
  int (*xMutexInit)(void);
  int (*xMutexEnd)(void);
  sqlite3_mutex *(*xMutexAlloc)(int);
  void (*xMutexFree)(sqlite3_mutex *);
  void (*xMutexEnter)(sqlite3_mutex *);
  int (*xMutexTry)(sqlite3_mutex *);
  void (*xMutexLeave)(sqlite3_mutex *);
  int (*xMutexHeld)(sqlite3_mutex *);
  int (*xMutexNotheld)(sqlite3_mutex *);
};
```

So on Unix/macOS the path is:

```text
sqlite3_mutex_enter(p)
  -> sqlite3GlobalConfig.mutex.xMutexEnter(p)
    -> pthreadMutexEnter(p)
      -> pthread_mutex_lock(&p->mutex)
```

On Windows, the same SQLite call routes to the Windows mutex implementation. In
single-thread/no-op builds, it can route to a no-op implementation.

### pthreads

`pthreads` means POSIX threads, the common C threading API on Unix-like systems
such as Linux, macOS, and BSD.

SQLite's Unix mutex implementation includes:

```c
#include <pthread.h>
```

The SQLite mutex object contains a pthread mutex:

```c
struct sqlite3_mutex {
  pthread_mutex_t mutex;
  ...
};
```

The real lock/unlock calls are:

```c
pthread_mutex_lock(&p->mutex);
pthread_mutex_unlock(&p->mutex);
```

`pthread_mutex_t` is opaque application-facing storage. User code should not
inspect fields inside it. The pthread library and OS use that object, plus any
needed platform/kernel state, to coordinate ownership and waiting.

### Java comparison

Java's:

```java
synchronized (obj) {
  ...
}
```

means:

```text
acquire the JVM monitor associated with obj
run the block
release the monitor
```

It does not automatically protect all fields inside `obj`. It just uses `obj` as
the identity/key for a hidden JVM-managed lock.

That is similar to:

```c
pthread_mutex_lock(&m);
...
pthread_mutex_unlock(&m);
```

Modern Java code often uses an explicit private lock object because it makes the
intent clearer:

```java
private final Object lock = new Object();

synchronized (lock) {
  ...
}
```

The same core rule applies in Java and C:

```text
the lock protects shared state only if all code uses the same lock before
touching that state
```

Source:

- `sqlite/src/mutex.c:319` says `sqlite3_mutex_enter()` blocks until the mutex
  can be obtained.
- `sqlite/src/mutex.c:322` implements the public `sqlite3_mutex_enter()`
  wrapper.
- `sqlite/src/mutex_unix.c:31` defines SQLite's Unix `sqlite3_mutex` wrapper
  around `pthread_mutex_t`.
- `sqlite/src/mutex_unix.c:270` calls `pthread_mutex_lock()`.
- `sqlite/src/mutex_unix.c:360` calls `pthread_mutex_unlock()`.
- `sqlite/src/mutex_unix.c:373` installs the pthread mutex method table.
- `sqlite/src/sqlite.h.in:8525` defines `sqlite3_mutex_methods`.

### CREATE TABLE file-lock timing

For `CREATE TABLE`, `btreeCreateTable()` is not the place that normally obtains
the database file lock. It assumes a write transaction is already open.

The useful call chain is:

```text
sqlite3_step()
  -> sqlite3VdbeExec()
    -> OP_Transaction
       -> sqlite3BtreeBeginTrans(pBt, pOp->p2, &iMeta)
          -> btreeBeginTrans(...)
             -> sqlite3BtreeEnter(p)
             -> lockBtree(pBt)
                -> sqlite3PagerSharedLock(pPager)
                   -> pager_wait_on_lock(..., SHARED_LOCK)
                      -> pagerLockDb(..., SHARED_LOCK)
                         -> sqlite3OsLock(fd, SHARED_LOCK)
             -> sqlite3PagerBegin(pPager, exFlag, ...)
                rollback-journal mode:
                  -> pagerLockDb(..., RESERVED_LOCK)
                     -> sqlite3OsLock(fd, RESERVED_LOCK)
                WAL mode:
                  -> sqlite3WalBeginWriteTransaction(...)
    -> OP_CreateBtree
       -> sqlite3BtreeCreateTable(...)
          -> sqlite3BtreeEnter(p)
          -> btreeCreateTable(...)
             -> allocateBtreePage(...)
             -> sqlite3PagerWrite(...)
             -> zeroPage(...)
          -> sqlite3BtreeLeave(p)
```

So the split is:

```text
OP_Transaction:
  get permission to read/write the database

OP_CreateBtree / btreeCreateTable:
  allocate and initialize the new root page after permission exists
```

There are two different lock lifetimes in that call chain:

```text
BtShared mutex:
  sqlite3BtreeEnter(p)
  protects in-memory btree state for this btree-layer call
  released by sqlite3BtreeLeave(p) before returning to the VDBE opcode loop

pager/file transaction state:
  sqlite3PagerSharedLock(...)
  sqlite3PagerBegin(...)
  SHARED_LOCK / RESERVED_LOCK / WAL write-lock state
  remains part of the active transaction until commit, rollback, or cleanup
```

So `OP_Transaction` can briefly enter the `BtShared` mutex and then leave it
while still leaving the database transaction open. Later, `OP_CreateBtree` enters
the `BtShared` mutex again because the earlier mutex critical section has ended.
The transaction-level pager/file permission is still active; the in-memory
btree mutex is not.

The mental model is:

```text
sqlite3BtreeEnter():
  short-lived mutex around shared in-memory btree structs

sqlite3PagerBegin():
  transaction-duration pager/file or WAL permission
```

`OP_Transaction` calls:

```c
rc = sqlite3BtreeBeginTrans(pBt, pOp->p2, &iMeta);
```

For a write transaction, `btreeBeginTrans()` eventually calls:

```c
rc = sqlite3PagerBegin(pPager, wrflag>1, sqlite3TempInMemory(p->db));
```

In rollback-journal mode, `sqlite3PagerBegin()` obtains a writer-intent file
lock:

```c
rc = pagerLockDb(pPager, RESERVED_LOCK);
```

If an exclusive transaction is requested, it may immediately upgrade:

```c
rc = pager_wait_on_lock(pPager, EXCLUSIVE_LOCK);
```

The final OS/VFS call is inside `pagerLockDb()`:

```c
rc = pPager->noLock ? SQLITE_OK : sqlite3OsLock(pPager->fd, eLock);
```

By the time `OP_CreateBtree` runs, `btreeCreateTable()` asserts that the btree is
already in a write transaction:

```c
assert( pBt->inTransaction==TRANS_WRITE );
```

Then it allocates the new root page:

```c
rc = allocateBtreePage(pBt, &pRoot, &pgnoRoot, 1, 0);
```

or, in auto-vacuum mode, chooses a root-page number and may move an existing page
to make room:

```c
rc = allocateBtreePage(pBt, &pPageMove, &pgnoMove, pgnoRoot, BTALLOC_EXACT);
...
rc = relocatePage(pBt, pRoot, eType, iPtrPage, pgnoMove, 0);
```

`allocateBtreePage()` marks page-allocation metadata writable. If it reuses a
freelist page, it decrements the freelist count on page 1:

```c
rc = sqlite3PagerWrite(pPage1->pDbPage);
put4byte(&pPage1->aData[36], n-1);
```

If it appends a new page to the database image, it also marks page 1 writable,
increments `pBt->nPage`, and writes the database-size field:

```c
rc = sqlite3PagerWrite(pBt->pPage1->pDbPage);
pBt->nPage++;
put4byte(28 + (u8*)pBt->pPage1->aData, pBt->nPage);
```

Then it gets the new page and marks that page writable:

```c
rc = btreeGetUnusedPage(pBt, *pPgno, ppPage, bNoContent);
rc = sqlite3PagerWrite((*ppPage)->pDbPage);
```

Back in `btreeCreateTable()`, the root page is initialized as a leaf table or
index page:

```c
zeroPage(pRoot, ptfFlags);
```

The role of `sqlite3PagerWrite()` in this path is not mainly "take the file
lock". The write transaction has already done that. Its role is:

```text
before modifying this page, make sure the pager has journaled/tracked enough
state for rollback and commit
```

The mental model is:

```text
file lock:
  obtained before btreeCreateTable(), during OP_Transaction

page allocation:
  done inside btreeCreateTable()

page journaling / dirty tracking:
  done by sqlite3PagerWrite() before page bytes are changed
```

Source:

- `sqlite/src/vdbe.c:4107` implements `OP_Transaction`.
- `sqlite/src/vdbe.c:4128` calls `sqlite3BtreeBeginTrans()`.
- `sqlite/src/btree.c:3607` enters the btree mutex in `btreeBeginTrans()`.
- `sqlite/src/btree.c:3660` comments that transactions imply a read-lock on
  page 1.
- `sqlite/src/btree.c:3717` calls `sqlite3PagerBegin()` for write transactions.
- `sqlite/src/btree.c:3282` implements `lockBtree()`.
- `sqlite/src/btree.c:3290` calls `sqlite3PagerSharedLock()`.
- `sqlite/src/pager.c:5318` waits for `SHARED_LOCK`.
- `sqlite/src/pager.c:5968` implements `sqlite3PagerBegin()`.
- `sqlite/src/pager.c:6002` obtains `RESERVED_LOCK` in rollback-journal mode.
- `sqlite/src/pager.c:6004` may wait for `EXCLUSIVE_LOCK`.
- `sqlite/src/pager.c:1157` calls `sqlite3OsLock()`.
- `sqlite/src/vdbe.c:7032` implements `OP_CreateBtree`.
- `sqlite/src/vdbe.c:7045` calls `sqlite3BtreeCreateTable()`.
- `sqlite/src/btree.c:10054` implements `btreeCreateTable()`.
- `sqlite/src/btree.c:10063` asserts the write transaction is already active.
- `sqlite/src/btree.c:10181` allocates the new root page in the non-auto-vacuum
  path.
- `sqlite/src/btree.c:10155` marks a moved auto-vacuum root page writable.
- `sqlite/src/btree.c:10193` initializes the new root page with `zeroPage()`.
- `sqlite/src/btree.c:10199` wraps `btreeCreateTable()` with
  `sqlite3BtreeEnter()` / `sqlite3BtreeLeave()`.
- `sqlite/src/btree.c:6514` implements `allocateBtreePage()`.
- `sqlite/src/btree.c:6570` marks page 1 writable before freelist metadata
  changes.
- `sqlite/src/btree.c:6776` marks page 1 writable before appending a new page.
- `sqlite/src/btree.c:6792` marks the allocated page writable.
