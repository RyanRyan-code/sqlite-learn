# `btreeCreateTable()`

This note explains SQLite's internal `btreeCreateTable()` function.

The function does not parse SQL and does not insert a row into
`sqlite_schema`. It works at the storage layer. Its job is narrower:

```text
allocate one empty b-tree root page
  -> initialize that page as either a table root or index root
  -> return the root page number to the caller
```

For a statement like:

```sql
CREATE TABLE t(a int);
```

the runtime path reaches this function through the VDBE opcode:

```text
OP_CreateBtree
  -> sqlite3BtreeCreateTable()
    -> btreeCreateTable()
```

## Function shape

```c
static int btreeCreateTable(Btree *p, Pgno *piTable, int createTabFlags)
```

The arguments are:

- `p`: the database b-tree handle.
- `piTable`: output pointer. The function writes the new root page number here.
- `createTabFlags`: says what page shape the new b-tree root should use.

The return value is an SQLite result code:

```text
SQLITE_OK        success
SQLITE_NOMEM     allocation failed
SQLITE_IOERR     pager/file operation failed
SQLITE_CORRUPT   database invariants were broken
...
```

The common `createTabFlags` values are:

```text
BTREE_INTKEY | BTREE_LEAFDATA   normal SQL rowid table
BTREE_ZERODATA                  SQL index
```

## Local variables

```c
BtShared *pBt = p->pBt;
```

Get the shared b-tree state. SQLite may have more than one `Btree` handle
pointing at the same underlying database file. `BtShared` is the shared part.

```c
MemPage *pRoot;
```

Will point to the in-memory representation of the page that becomes the new root
page.

```c
Pgno pgnoRoot;
```

Will hold the page number of the new root page.

```c
int rc;
```

The usual SQLite return-code variable.

```c
int ptfFlags;
```

The page-type flags used when initializing the new root page.

## Preconditions

```c
assert( sqlite3BtreeHoldsMutex(p) );
assert( pBt->inTransaction==TRANS_WRITE );
assert( (pBt->btsFlags & BTS_READ_ONLY)==0 );
```

These are debug-only checks. They say:

- the caller already holds the b-tree mutex
- the b-tree is already inside a write transaction
- the database is not read-only

The read-only assertion uses bitwise `&`, not logical `&&`:

```c
(pBt->btsFlags & BTS_READ_ONLY)==0
```

`BTS_READ_ONLY` is a constant bit mask. The expression checks whether that
specific bit is set inside `pBt->btsFlags`. If the bitwise `&` result is zero,
the read-only flag is absent.

This function assumes the transaction and locking setup has already happened.
It is allowed to allocate and modify pages.

## Simple allocation path

When autovacuum is omitted at compile time, or compiled in but disabled for this
database, the function can allocate a page directly:

```c
rc = allocateBtreePage(pBt, &pRoot, &pgnoRoot, 1, 0);
if( rc ){
  return rc;
}
```

This asks the b-tree layer for a writable page. On success:

```text
pRoot     -> in-memory page object
pgnoRoot  -> page number of that page
```

The `1` is a nearby/preferred page hint. The final `0` means ordinary
allocation mode.

## Why autovacuum is more complicated

With autovacuum enabled, SQLite keeps pointer-map pages. These pages record
"who points to this page?" information so pages can be moved safely when the
database shrinks or reorganizes itself.

Because pages may move, root pages are special. SQLite tracks the largest root
page created so far and tries to allocate new root pages in increasing page
number order.

That explains the larger branch:

```c
if( pBt->autoVacuum ){
  ...
}
```

There are two different autovacuum switches involved:

```text
SQLITE_OMIT_AUTOVACUUM   compile-time SQLite build option
pBt->autoVacuum          runtime per-database state
```

If SQLite is compiled with `SQLITE_OMIT_AUTOVACUUM`, this source branch is not
included in the binary. The function always uses the simple allocation path.

If autovacuum support is compiled in, each database can still have autovacuum
off or on. That database-level setting is exposed through SQLite's SQL-facing
`PRAGMA` command:

```sql
PRAGMA auto_vacuum;
PRAGMA auto_vacuum = FULL;
PRAGMA auto_vacuum = INCREMENTAL;
PRAGMA auto_vacuum = NONE;
```

`PRAGMA` is SQLite-specific SQL-interface syntax for reading or changing
internal settings. It comes in through the SQL front door, but SQLite handles it
with special code in `pragma.c`, not like an ordinary user table query.

For `auto_vacuum`, the rough path is:

```text
PRAGMA auto_vacuum = ...
  -> pragma.c
    -> sqlite3BtreeSetAutoVacuum()
      -> pBt->autoVacuum / pBt->incrVacuum
      -> database metadata when allowed
```

For an existing database, `pBt->autoVacuum` is normally loaded from the database
header/page-1 metadata. It can be changed while configuring a new or compatible
database, but SQLite will not casually flip an existing initialized database
between autovacuum-capable and non-autovacuum-capable layouts.

## Pointer-map pages

Pointer-map pages are SQLite's relocation lookup table for autovacuum.

When SQLite moves a page from one page number to another, it must update the
page that points to it. The pointer map lets SQLite find that parent/reference
quickly without scanning the entire database.

The source comment describes the logical entry shape as:

```text
1 byte   pointer-map type
4 bytes  parent page number
```

So pointer-map pages are normal database pages filled with 5-byte entries. The
valid entry types are:

```c
#define PTRMAP_ROOTPAGE  1
#define PTRMAP_FREEPAGE  2
#define PTRMAP_OVERFLOW1 3
#define PTRMAP_OVERFLOW2 4
#define PTRMAP_BTREE     5
```

Their meanings are:

```text
PTRMAP_ROOTPAGE
  the page is a b-tree root page; parent page number is unused

PTRMAP_FREEPAGE
  the page is free; parent page number is unused

PTRMAP_OVERFLOW1
  the page is the first overflow page for a cell; parent points to the b-tree
  page containing that cell

PTRMAP_OVERFLOW2
  the page is later in an overflow chain; parent points to the previous
  overflow page

PTRMAP_BTREE
  the page is a non-root b-tree page; parent points to its parent b-tree page
```

In memory, SQLite handles pointer-map pages through the pager as `DbPage *` plus
raw bytes:

```c
DbPage *pDbPage;  /* The pointer map page */
u8 *pPtrmap;      /* The pointer map data */
```

`DbPage` is a typedef for the pager/cache page header:

```c
typedef struct PgHdr DbPage;
```

And `PgHdr` contains:

```c
struct PgHdr {
  sqlite3_pcache_page *pPage;
  void *pData;
  void *pExtra;
  ...
};
```

Then:

```c
pPtrmap = (u8 *)sqlite3PagerGetData(pDbPage);
```

`sqlite3PagerGetData()` returns `pPg->pData`:

```c
void *sqlite3PagerGetData(DbPage *pPg){
  assert( pPg->nRef>0 || pPg->pPager->memDb );
  return pPg->pData;
}
```

So the representation is:

```text
on disk:
  fixed-size database page
  5-byte pointer-map entries

in memory:
  DbPage * / PgHdr * from the pager
  PgHdr.pData as raw u8 * bytes
```

This is different from a normal table/index b-tree page:

```text
normal b-tree page:
  DbPage from pager
  + PgHdr.pData raw b-tree page bytes
  + PgHdr.pExtra initialized as a MemPage wrapper

pointer-map page:
  DbPage from pager
  + PgHdr.pData raw pointer-map entries
  + PgHdr.pExtra must not be initialized as a b-tree MemPage
```

`ptrmapPut()` checks for this corruption case:

```c
if( ((char*)sqlite3PagerGetExtra(pDbPage))[0]!=0 ){
  /* The first byte of the extra data is the MemPage.isInit byte.
  ** If that byte is set, it means this page is also being used
  ** as a btree page. */
  *pRC = SQLITE_CORRUPT_BKPT;
}
```

That means: if the pager page's extra area says `MemPage.isInit` is set, SQLite
believes the same page is also being used as a b-tree page. A pointer-map page
must not also be a b-tree page.

## Choosing the new root page number

```c
invalidateAllOverflowCache(pBt);
```

Open cursors may cache overflow-page information. Since creating a new table may
move pages around, these caches are invalidated first.

```c
sqlite3BtreeGetMeta(p, BTREE_LARGEST_ROOT_PAGE, &pgnoRoot);
```

Read the metadata value for the largest root page created so far.

```c
if( pgnoRoot>btreePagecount(pBt) ){
  return SQLITE_CORRUPT_PGNO(pgnoRoot);
}
```

If metadata claims that the largest root page is beyond the end of the database,
the database is corrupt.

```c
pgnoRoot++;
```

The candidate root page is the next page after the largest old root page.

```c
while( pgnoRoot==PTRMAP_PAGENO(pBt, pgnoRoot) ||
    pgnoRoot==PENDING_BYTE_PAGE(pBt) ){
  pgnoRoot++;
}
```

Some page numbers cannot be root pages:

- pointer-map pages are used for autovacuum bookkeeping
- the pending-byte page is reserved for SQLite's locking protocol

The loop skips over those page numbers.

```c
assert( pgnoRoot>=3 );
```

In this path, the chosen root page should be page 3 or later. Page 1 is the
database header page, and page 2 has special cases elsewhere.

## Allocating room for the root page

```c
rc = allocateBtreePage(pBt, &pPageMove, &pgnoMove, pgnoRoot, BTALLOC_EXACT);
```

This asks SQLite to allocate exactly `pgnoRoot`.

The subtle part: if `pgnoRoot` is currently occupied, SQLite may allocate a
different page, `pgnoMove`, to receive the page that is currently sitting at
`pgnoRoot`.

So after this call:

```text
pgnoRoot  -> the page number SQLite wants for the new root
pgnoMove  -> where the old occupant of pgnoRoot may need to move
```

If:

```c
pgnoMove==pgnoRoot
```

then no move is needed. The desired page was allocated directly.

If:

```c
pgnoMove!=pgnoRoot
```

then the current page at `pgnoRoot` has to be relocated to `pgnoMove`.

## Moving the old page away

```c
u8 eType = 0;
Pgno iPtrPage = 0;
```

These hold pointer-map information for the page currently at `pgnoRoot`:

- `eType`: what kind of page it is
- `iPtrPage`: which page points to it

```c
rc = saveAllCursors(pBt, 0, 0);
releasePage(pPageMove);
```

Before moving pages, SQLite saves open cursor positions. This protects cursors
that may be holding direct page references.

Then it releases the in-memory reference to `pPageMove`. The page allocation is
still valid; SQLite is just letting go of this `MemPage *` reference.

```c
rc = btreeGetPage(pBt, pgnoRoot, &pRoot, 0);
```

Load the page currently at `pgnoRoot`.

At this moment, `pRoot` is a confusing name. It does not yet point to the new
root page. It points to the old page that must be moved away.

```c
rc = ptrmapGet(pBt, pgnoRoot, &eType, &iPtrPage);
```

Read pointer-map information for the page being moved.

```c
if( eType==PTRMAP_ROOTPAGE || eType==PTRMAP_FREEPAGE ){
  rc = SQLITE_CORRUPT_PGNO(pgnoRoot);
}
```

SQLite expected a normal movable page. If the page is already a root page or a
free page, the database state is inconsistent, so the function reports
corruption.

```c
rc = relocatePage(pBt, pRoot, eType, iPtrPage, pgnoMove, 0);
releasePage(pRoot);
```

Move the old page from `pgnoRoot` to `pgnoMove`.

`relocatePage()` also updates references and pointer-map metadata so anything
that used to point to the old location now points to the new one.

After this, `pgnoRoot` is available for the new table or index root page.

```c
rc = btreeGetPage(pBt, pgnoRoot, &pRoot, 0);
rc = sqlite3PagerWrite(pRoot->pDbPage);
```

Load the now-available `pgnoRoot` page and mark it writable through the pager.

From here on, `pRoot` really is the new root page.

## When no move was needed

```c
}else{
  pRoot = pPageMove;
}
```

If the exact requested page was allocated, no relocation was needed. The
allocated page object is already the new root page.

## Updating autovacuum metadata

```c
ptrmapPut(pBt, pgnoRoot, PTRMAP_ROOTPAGE, 0, &rc);
```

Record in the pointer map that `pgnoRoot` is now a root page.

```c
assert( sqlite3PagerIswriteable(pBt->pPage1->pDbPage) );
rc = sqlite3BtreeUpdateMeta(p, 4, pgnoRoot);
```

Update database metadata so `BTREE_LARGEST_ROOT_PAGE` now stores the new root
page number.

The comment in the source says this update should not fail because page 1 was
already made writable while allocating the root page.

```c
if( NEVER(rc) ){
  releasePage(pRoot);
  return rc;
}
```

`NEVER(rc)` documents an "expected impossible" case while still handling it
defensively.

## Initializing the new page

At this point, all branches converge here:

```c
assert( sqlite3PagerIswriteable(pRoot->pDbPage) );
```

The new root page must be writable.

```c
if( createTabFlags & BTREE_INTKEY ){
  ptfFlags = PTF_INTKEY | PTF_LEAFDATA | PTF_LEAF;
}else{
  ptfFlags = PTF_ZERODATA | PTF_LEAF;
}
```

This chooses the b-tree page type.

For a normal rowid table:

```text
PTF_INTKEY    keys are integer rowids
PTF_LEAFDATA  table row data lives on leaf pages
PTF_LEAF      this brand-new root page is also a leaf page
```

For an index:

```text
PTF_ZERODATA  no separate data payload
PTF_LEAF      this brand-new root page is also a leaf page
```

The root starts as a leaf because an empty b-tree has only one page. If rows are
inserted later and the tree grows, SQLite can split pages and turn the root into
an internal page.

```c
zeroPage(pRoot, ptfFlags);
```

Initialize the page as an empty b-tree page with the selected flags.

This is the moment the allocated page becomes a valid empty table/index root.

```c
sqlite3PagerUnref(pRoot->pDbPage);
```

Release this function's reference to the page.

```c
assert( (pBt->openFlags & BTREE_SINGLE)==0 || pgnoRoot==2 );
```

Debug check for a special single-btree mode. If that mode is active, the root
page must be page 2.

```c
*piTable = pgnoRoot;
```

Return the new root page number to the caller through the output pointer.

```c
return SQLITE_OK;
```

The new empty b-tree exists.

## Compact flow

Without autovacuum:

```text
allocate page
  -> zero it as table/index leaf root
  -> return page number
```

With autovacuum:

```text
read largest-root metadata
  -> choose next legal root page
  -> allocate exact root page if possible
  -> move old page away if needed
  -> mark page as PTRMAP_ROOTPAGE
  -> update largest-root metadata
  -> zero it as table/index leaf root
  -> return page number
```

## The main mental model

`btreeCreateTable()` is not "create table" in the SQL sense.

It is "create the empty storage root that a table or index will use."

The schema row is handled elsewhere. This function only makes sure there is a
real b-tree root page on disk, initialized with the right kind of page header,
and gives its page number back to the VDBE.
