# Pager journal and savepoint notes

## `sqlite3PagerWrite()` does not initialize the page

Despite its name, `sqlite3PagerWrite()` does not immediately write new page
contents to the database file. It prepares a cached page to be changed:

```text
sqlite3PagerWrite(page)
  -> preserve the old contents in a journal when required
  -> mark the cached page dirty
  -> set PGHDR_WRITEABLE
```

The caller must check for `SQLITE_OK` before changing any page bytes. The
`PGHDR_WRITEABLE` flag is deliberately set only after the required journaling
has succeeded.

`zeroPage()` has a different job. It formats an already-writable `MemPage` as
an empty b-tree page by setting its page flags, cell count, free-space offsets,
and in-memory metadata. It requires the pager preparation to be complete:

```c
assert( sqlite3PagerIswriteable(pPage->pDbPage) );
```

For a newly allocated b-tree root, the order is therefore:

```text
allocateBtreePage()
  -> sqlite3PagerWrite()   journal/track the page and make it writable

zeroPage()
  -> give the page its empty table-btree or index-btree format
```

Allocation does not always lead to `zeroPage()`. The allocated page might
instead become an overflow page or another structure with a different format.

## Why savepoints need a sub-journal

The main rollback journal preserves page images from the beginning of the
transaction. A savepoint may need a later, intermediate image:

```text
transaction begins:  page 7 = A
modify page 7:        page 7 = B    main journal saves A

SAVEPOINT s
modify page 7:        page 7 = C    sub-journal saves B

ROLLBACK TO s:        page 7 = B
ROLLBACK transaction: page 7 = A
```

Each `PagerSavepoint` records:

```text
nOrig             database page count when the savepoint opened
pInSavepoint      pages whose before-images have already been preserved
```

Since page numbers begin at 1, `nOrig == 10` means pages 1 through 10 existed
in the savepoint's logical database image.

`subjRequiresPage()` requires a sub-journal record when at least one active
savepoint satisfies both conditions:

```text
page number <= savepoint.nOrig
page not already recorded for that savepoint
```

The first condition asks whether the page existed when that savepoint began.
The second prevents SQLite from saving the same before-image repeatedly.

## Why appended pages usually need no sub-journal record

Suppose a savepoint opens when the database contains 10 pages, then SQLite
appends page 11:

```text
SAVEPOINT s: nOrig = 10
append page 11
ROLLBACK TO s
```

Page 11 has no pre-savepoint contents to restore because it did not exist in
that database image. Rollback restores the database size to `nOrig`:

```text
dbSize = 10
```

That makes page 11 disappear from the logical database. Sub-journaling its
bytes would not provide anything useful.

An existing page is different:

```text
modify page 7  -> rollback must restore old bytes, so save its before-image
append page 11 -> rollback only needs to remove it by restoring dbSize
```

The rule is evaluated separately for nested savepoints:

```text
SAVEPOINT s1: nOrig = 10
append page 11
SAVEPOINT s2: nOrig = 11
modify page 11
```

Page 11 needs no before-image for `s1`; rolling back to `s1` removes it. It
does need one for `s2`; page 11 existed when `s2` began, so rolling back to
`s2` must restore its contents from that moment.

## When the sub-journal is updated in the followed bytecode path

The standalone bytecode for:

```sql
CREATE TABLE t(a int);
```

contains `OP_Transaction` but no `OP_Savepoint`. In the normal autocommit
execution, `pPager->nSavepoint` is zero, so its `sqlite3PagerWrite()` calls do
not update a sub-journal.

An explicit `SAVEPOINT` is a separate SQL statement with its own VDBE program:

```text
program for SAVEPOINT s:
  OP_Savepoint                 update connection savepoint state

later program for CREATE TABLE:
  OP_Transaction
    -> sqlite3BtreeBeginTrans()
       -> sqlite3PagerOpenSavepoint()
  OP_CreateBtree
    -> allocateBtreePage()
       -> sqlite3PagerWrite()
          -> subjournalPageIfRequired()
```

`sqlite3PagerOpenSavepoint()` creates pager bookkeeping such as `nOrig`, the
bit-vector, and the starting sub-journal record number. It does not eagerly
copy every database page. The before-image is written lazily by the first
later `sqlite3PagerWrite()` that needs it.

During the append path, this distinction applies page by page:

```text
sqlite3PagerWrite(page 1)    page 1 existed, so preserve it before changing
                             the database-size header

sqlite3PagerWrite(page 11)   if nOrig was 10, page 11 needs no before-image
```

SQLite builds a statement's complete opcode array during `sqlite3_prepare()`
and executes it later during `sqlite3_step()`. The compiler does not bake the
current pager savepoint count into `CREATE TABLE` bytecode. `OP_Transaction`
and `sqlite3PagerWrite()` consult the connection and pager state at runtime.

## WAL mode

WAL mode replaces the main rollback journal, but not pager savepoint
bookkeeping or the sub-journal:

```text
rollback-journal mode:
  main journal    transaction-start images
  sub-journal     savepoint-start images

WAL mode:
  WAL frames      transaction changes
  WAL savepoint   position/state to rewind
  sub-journal     page images needed for savepoint rollback
```

On WAL savepoint rollback, SQLite calls `sqlite3WalSavepointUndo()` and then
plays back relevant sub-journal records. Therefore `subjRequiresPage()`,
`nOrig`, and `pInSavepoint` still apply in WAL mode.

## The already-writable fast path

`sqlite3PagerWrite()` avoids repeating `pager_write()` when:

```c
(pPg->flags & PGHDR_WRITEABLE)!=0 && pPager->dbSize>=pPg->pgno
```

The page has already been journaled as required for the transaction, marked
dirty, and included in the logical database size. If a savepoint is active,
SQLite still checks the sub-journal because the savepoint may have opened
after the page first became writable:

```text
write page 7       pager_write() preserves A; page becomes writable; A -> B
SAVEPOINT s
write page 7       skip pager_write(), but sub-journal B before B -> C
ROLLBACK TO s      restore B
```

The `dbSize>=pgno` part matters because savepoint rollback can shrink the
logical image while a cached page beyond the new end still carries
`PGHDR_WRITEABLE`. Reusing that page must run `pager_write()` again so it can
extend `dbSize` and perform the current bookkeeping.
