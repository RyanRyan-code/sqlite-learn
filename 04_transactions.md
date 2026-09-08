# SQLite transactions

This note is for building a durable mental model of SQLite transactions from
the source code.

The main idea:

```text
SQL transaction
  -> VDBE transaction opcodes
    -> btree transaction state
      -> pager transaction state
        -> journal/WAL state
          -> VFS file locks
```

The word "transaction" appears at several layers. Those layers are related, but
they are not the same object.

## Layer map

At the SQL level:

```sql
BEGIN;
CREATE TABLE t(a int);
INSERT INTO t VALUES (1);
COMMIT;
```

But an explicit `BEGIN` is not required for a transaction to exist. A standalone
statement like this also runs inside a transaction:

```sql
CREATE TABLE t(a int);
```

SQLite starts an implicit transaction for the statement if one is not already
open, then commits it when the statement finishes successfully.

Conceptually:

```sql
CREATE TABLE t(a int);
```

acts like:

```sql
BEGIN;
CREATE TABLE t(a int);
COMMIT;
```

The implementation does not literally prepend and append SQL text. The VDBE and
pager machinery open and close the transaction.

At the VDBE level:

```text
OP_Transaction
OP_CreateBtree / OP_OpenWrite / OP_Insert / ...
OP_AutoCommit or commit/halt cleanup
```

At the btree level:

```text
sqlite3BtreeBeginTrans()
sqlite3BtreeCommitPhaseOne()
sqlite3BtreeCommitPhaseTwo()
sqlite3BtreeRollback()
```

At the pager level:

```text
sqlite3PagerSharedLock()
sqlite3PagerBegin()
sqlite3PagerWrite()
sqlite3PagerCommitPhaseOne()
sqlite3PagerCommitPhaseTwo()
sqlite3PagerRollback()
```

At the VFS / OS level:

```text
sqlite3OsLock()
sqlite3OsUnlock()
sqlite3OsRead()
sqlite3OsWrite()
sqlite3OsSync()
```

## Transaction state is layered

The btree layer tracks transaction state with:

```text
Btree.inTrans
BtShared.inTransaction
```

The pager layer tracks transaction state with:

```text
Pager.eState
Pager.eLock
```

A useful distinction:

```text
Btree/BtShared transaction state:
  "What does the btree layer believe this handle/shared btree is doing?"

Pager transaction state:
  "What file-lock, journal/WAL, cache, and dirty-page state exists?"
```

## Read transaction outline

A read transaction begins by getting enough access to safely read page 1 and
database contents:

```text
sqlite3BtreeBeginTrans(..., wrflag=0, ...)
  -> btreeBeginTrans(...)
     -> sqlite3BtreeEnter(p)
     -> lockBtree(pBt)
        -> sqlite3PagerSharedLock(pPager)
           -> pager_wait_on_lock(..., SHARED_LOCK)
              -> pagerLockDb(..., SHARED_LOCK)
                 -> sqlite3OsLock(fd, SHARED_LOCK)
        -> btreeGetPage(... page 1 ...)
     -> set btree read transaction state
     -> sqlite3BtreeLeave(p)
```

The key point:

```text
read transaction
  -> SHARED_LOCK in rollback-journal mode
  -> snapshot/read coordination in WAL mode
```

## BtreeEnter vs lockBtree vs PagerBegin

These names look similar when reading the call chain, but they live at different
conceptual levels.

```text
sqlite3BtreeEnter(p)
  enter the in-memory BtShared mutex
  protect C structs inside SQLite
  does not read page 1
  does not take a database file lock

lockBtree(pBt)
  start access to the actual btree database file
  obtain pager read access / SHARED_LOCK
  read page 1
  initialize pBt->pPage1 and page-size/page-count state

sqlite3PagerBegin(pPager, exFlag, ...)
  turn the pager's read transaction into a write transaction
  obtain writer permission
```

The call shape in `btreeBeginTrans()` is:

```text
sqlite3BtreeBeginTrans()
  -> btreeBeginTrans()
     -> sqlite3BtreeEnter(p)
        "this thread may safely touch pBt fields"

     -> lockBtree(pBt)
        -> sqlite3PagerSharedLock(pPager)
           -> pager_wait_on_lock(..., SHARED_LOCK)
              -> pagerLockDb(..., SHARED_LOCK)
                 -> sqlite3OsLock(fd, SHARED_LOCK)
        -> btreeGetPage(pBt, 1, &pPage1, 0)
        "this connection can read the database and has page 1"

     -> sqlite3PagerBegin(pPager, exFlag, ...)
        rollback-journal mode:
          -> pagerLockDb(..., RESERVED_LOCK)
          -> maybe pager_wait_on_lock(..., EXCLUSIVE_LOCK)
        WAL mode:
          -> sqlite3WalBeginWriteTransaction(...)
        "this connection has writer permission"
```

The compact distinction:

```text
sqlite3BtreeEnter:
  memory mutex boundary

lockBtree:
  read/open database access and load page 1

sqlite3PagerBegin:
  upgrade from reader to writer
```

## Write transaction outline

A write transaction first has to be a reader, then it becomes a writer:

```text
sqlite3BtreeBeginTrans(..., wrflag=1 or 2, ...)
  -> btreeBeginTrans(...)
     -> sqlite3BtreeEnter(p)
     -> lockBtree(pBt)
        -> sqlite3PagerSharedLock(pPager)
     -> sqlite3PagerBegin(pPager, exFlag, ...)
        rollback-journal mode:
          -> pagerLockDb(..., RESERVED_LOCK)
          -> maybe pager_wait_on_lock(..., EXCLUSIVE_LOCK)
        WAL mode:
          -> sqlite3WalBeginWriteTransaction(...)
     -> set btree write transaction state
     -> sqlite3BtreeLeave(p)
```

The key point:

```text
opening the write transaction is when SQLite gets writer permission
```

Later operations such as `btreeCreateTable()` and `sqlite3BtreeInsert()` assume
that permission already exists.

## sqlite3PagerWrite

`sqlite3PagerWrite()` is easy to misread as "lock the database for writing".
Usually, by the time btree code calls it, the write transaction is already open.

Its job is closer to:

```text
before modifying this page:
  make sure the old content is recoverable
  mark the page dirty
  arrange rollback/commit bookkeeping
```

So in a path like:

```text
OP_CreateBtree
  -> sqlite3BtreeCreateTable()
     -> allocateBtreePage()
        -> sqlite3PagerWrite(page 1)
        -> sqlite3PagerWrite(new root page)
```

the file-lock permission was obtained earlier by `OP_Transaction`.

## File lock levels

Rollback-journal mode uses database file lock levels:

```text
SHARED:
  many readers

RESERVED:
  one connection intends to write, while readers may still exist

PENDING:
  writer is waiting to exclude readers; existing readers may remain, but new
  readers are blocked

EXCLUSIVE:
  writer has exclusive access for database-file writes that require it
```

These are not mutex types. They are VFS/OS file locks.

## What the file lock locks

SQLite's lock names are higher-level protocol states. The operating system does
not usually know about a "SQLite RESERVED lock" as a special object. SQLite maps
those states onto ordinary file-locking primitives supplied by the VFS.

The useful split is:

```text
OS/VFS provides:
  read/shared locks
  write/exclusive locks
  byte ranges inside a file

SQLite defines:
  which byte or byte range means SHARED, RESERVED, PENDING, or EXCLUSIVE
  which transitions are allowed
  when readers are allowed to keep going
  when new readers are blocked
  when the writer may overwrite the database file
```

So it is still a file lock. The slightly surprising part is that Unix and
Windows file-locking APIs can lock byte ranges inside a file, and SQLite uses a
few byte ranges as coordination markers. The bytes are not rows, btree pages, or
useful database content. They are agreed-upon lock coordinates.

Another way to say it: SQLite treats these byte positions like a tiny
cross-process lock register bank attached to the database file. The contents of
the bytes are not the signal. The lock state on those byte positions is the
signal.

When saying "SQLite locks the database file", be precise:

```text
What the OS sees:
  this process has a read/write lock on this byte range of this file

What SQLite means:
  this connection is in SHARED/RESERVED/PENDING/EXCLUSIVE state for the
  database file
```

So the OS may only be locking one designated byte or a 510-byte shared-lock
range, not every byte of the `.db` file. Because every SQLite connection follows
the same lock protocol, those tiny byte-range locks coordinate access to the
whole database file.

On Unix, these locks are usually advisory. A non-cooperating program could
ignore them and write the file anyway. SQLite correctness depends on SQLite
connections using the same protocol and on the filesystem implementing the lock
operations correctly.

SQLite's default rollback-lock byte layout is:

```text
PENDING_BYTE:
  first lock byte, normally just past the 1GB boundary

RESERVED_BYTE:
  PENDING_BYTE + 1

SHARED_FIRST .. SHARED_FIRST + SHARED_SIZE - 1:
  shared-lock range, 510 bytes by default
```

The simplified mapping is:

```text
SHARED_LOCK:
  temporarily read-lock PENDING_BYTE
  then read-lock the shared-lock range
  then release PENDING_BYTE

RESERVED_LOCK:
  requires an existing SHARED_LOCK
  write-lock RESERVED_BYTE

PENDING_LOCK:
  not requested directly by pager code
  happens while trying to get EXCLUSIVE_LOCK
  write-lock PENDING_BYTE
  blocks new SHARED_LOCK attempts, but old readers may remain

EXCLUSIVE_LOCK:
  write-lock the whole shared-lock range
  can succeed only after other shared/read locks are gone
```

This is why `SHARED_LOCK` is "shared": many readers can hold compatible
read-locks on the database file's shared-lock range. It is not a row/page/table
lock. Its practical meaning is:

```text
I am reading this database-file image.
Do not let a writer overwrite the database file until readers release it.
```

A writer can still get `RESERVED_LOCK` while readers exist, because that uses a
different lock byte. But before the writer writes changed pages back into the
main database file in rollback-journal mode, it needs `EXCLUSIVE_LOCK`, which
conflicts with readers' shared-range locks.

The `PENDING_LOCK` step is the drain-the-readers step:

```text
writer has RESERVED_LOCK
  -> wants EXCLUSIVE_LOCK
  -> takes PENDING_BYTE as a write-lock
  -> new SHARED_LOCK attempts fail
  -> existing SHARED_LOCK holders finish naturally
  -> writer retries and gets EXCLUSIVE_LOCK
```

## Why not row locks?

It is tempting to think SQLite could avoid this by locking rows instead of the
database file. But at the pager/VFS layer, rows are not the thing being changed.
SQLite is coordinating safe access to database files, pages, journals, and WAL
state.

A single logical row update can touch more than one physical place:

```text
table btree page containing the row
overflow pages for large records
index btree pages for indexed columns
page headers and cell pointer arrays
parent/internal btree pages after page splits or merges
freelist pages
rollback journal or WAL records
schema pages for DDL
```

Rows are also not stable byte ranges. A record can move because of inserts,
deletes, vacuuming, page splits, defragmentation inside a page, or overflow-page
changes. The operating system cannot know which bytes are "row 42" or which
index entries and metadata belong to that row. It only knows files, offsets,
lengths, and read/write locks.

To make row locks work as a general concurrency feature, SQLite would need a
much larger concurrency subsystem:

```text
lock manager
shared lock table across connections/processes
deadlock detection
row/page/intention lock hierarchy
predicate or range locks for index scans
more complex visibility and recovery rules
```

That is the kind of machinery a server database can centralize. SQLite avoids a
server process, so it uses the filesystem and VFS locks as the shared
coordination point.

The compact tradeoff:

```text
SQLite:
  no server
  one portable database file
  many readers
  one writer at a time
  rollback-journal commits may need brief exclusive database-file access
  WAL improves reader/writer overlap, but still keeps one writer

PostgreSQL:
  server process
  shared memory and lock tables
  MVCC and WAL managed by the server
  row-level locks for normal row updates
  many concurrent writers when they do not conflict
```

Postgres also has table locks, internal page latches, predicate locks in some
isolation modes, and advisory locks. But as a mental model:

```text
SQLite coordinates writes at the database-file level.
Postgres can coordinate ordinary row changes at the row level.
```

## WAL comparison with Postgres

In terms of reader/writer blocking, SQLite WAL mode is closer to Postgres than
SQLite rollback-journal mode is.

The rough shape:

```text
SQLite rollback journal:
  writer eventually overwrites the main database file
  readers need a stable main database file
  writer may need to wait for readers before commit can write the db file

SQLite WAL:
  writer appends changes to the WAL file
  readers keep their snapshot using the database file plus WAL state
  commit usually does not force existing readers out
  writers are still serialized

Postgres:
  writer appends WAL records for recovery
  writer also changes shared-buffer pages and later data files
  readers use MVCC snapshots to decide which tuple versions are visible
  ordinary readers usually do not block writers
  many writers can run unless they conflict on rows or higher-level objects
```

So the useful lesson is:

```text
WAL helps avoid "writer must rewrite the main file while readers are using it."
MVCC is what lets Postgres make reader/writer overlap the normal case.
```

SQLite WAL has better reader/writer overlap than rollback-journal mode, but it
does not turn SQLite into a many-writer row-locking system. It still relies on
embedded-file coordination and a single writer at a time. Postgres can afford a
more detailed lock model because the server owns shared memory, lock tables,
deadlock detection, snapshots, and recovery.

## Questions to trace next

- How does explicit `BEGIN` differ from the implicit transaction around one
  standalone statement?
- When exactly does rollback-journal mode upgrade from `RESERVED_LOCK` to
  `EXCLUSIVE_LOCK`?
- How does WAL mode replace database-file writer locks with WAL-index/write
  locks?
- What do `CommitPhaseOne` and `CommitPhaseTwo` each guarantee?
- When are dirty pages written to the database file versus journal or WAL?
- How do savepoints and statement journals fit into the pager model?

Source starting points:

- `sqlite/src/vdbe.c:4107` implements `OP_Transaction`.
- `sqlite/src/btree.c:3580` begins the `btreeBeginTrans()` discussion.
- `sqlite/src/btree.c:3607` enters the btree mutex in `btreeBeginTrans()`.
- `sqlite/src/btree.c:3717` calls `sqlite3PagerBegin()` for write transactions.
- `sqlite/src/btree.c:3805` implements `sqlite3BtreeBeginTrans()`.
- `sqlite/src/pager.c:5300` implements `sqlite3PagerSharedLock()`.
- `sqlite/src/pager.c:5968` implements `sqlite3PagerBegin()`.
- `sqlite/src/pager.c:1157` calls `sqlite3OsLock()` through `pagerLockDb()`.
- `sqlite/src/os.h:83` documents VFS file lock levels.
- `sqlite/src/os.h:112` documents the lock-byte/range layout.
- `sqlite/src/os.h:160` defines `PENDING_BYTE`, `RESERVED_BYTE`,
  `SHARED_FIRST`, and `SHARED_SIZE`.
- `sqlite/src/os_unix.c:1846` describes allowed lock transitions.
- `sqlite/src/os_unix.c:1866` implements `unixLock()`.
