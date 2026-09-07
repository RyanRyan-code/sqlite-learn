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
