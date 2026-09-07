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
