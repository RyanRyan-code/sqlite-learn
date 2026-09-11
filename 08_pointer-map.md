# Pointer-map pages and `ptrmapGet()` / `ptrmapPut()`

This note explains SQLite's pointer-map pages and the functions that read and
write them.

The short version is:

```text
ordinary reference:  parent page -> child page
pointer-map entry:    child page  -> type + parent page
```

The pointer map is a reverse-reference table used by autovacuum. When SQLite
moves a page, it needs to find and update the page containing the pointer to
that page. The pointer map provides that lookup without scanning every b-tree
page in the database.

## When pointer-map pages exist

Pointer-map pages exist only in databases with autovacuum enabled:

```text
PRAGMA auto_vacuum = FULL
```

or:

```text
PRAGMA auto_vacuum = INCREMENTAL
```

At runtime this is represented by:

```c
pBt->autoVacuum
```

If autovacuum is disabled, these page numbers are available for ordinary
database content instead.

Manual `VACUUM` is a different mechanism. It can rebuild a database without
requiring pointer-map pages.

## A pointer-map page is a fixed array

A pointer-map page is not a b-tree page. It has no b-tree page header, cell
pointer array, or cells.

It is also not organized like the freelist. It has no next-trunk pointer and no
array of freelist leaf page numbers.

Instead, its usable bytes contain a dense array of fixed-size entries:

```text
one entry = 5 bytes

byte 0      entry type
bytes 1..4  parent/reference page number, big-endian
```

The page being described is not stored inside the entry. Its identity comes
from the entry's position, like an array index:

```text
pointer-map page 2

offset 0     entry for database page 3
offset 5     entry for database page 4
offset 10    entry for database page 5
offset 15    entry for database page 6
offset 20    entry for database page 7
...
```

Therefore SQLite does not search the entries for a page number. Given target
page `key`, it calculates the pointer-map page and byte offset directly.

Because every covered database page has a permanent slot, entries are not
inserted or deleted. SQLite overwrites a slot when the corresponding page
changes role.

## What "parent page" means

Here, parent means the database page containing the pointer that refers to the
target page.

For example, suppose b-tree page 3 contains a child pointer to page 7:

```text
b-tree direction:
  page 3 -> page 7

pointer-map direction:
  entry for page 7 = { PTRMAP_BTREE, parent 3 }
```

This is physical page ownership, not a SQL foreign-key relationship and not a
relationship between table rows.

The word "parent" is exact for b-tree children, but the four-byte field is more
generally a backlink or owner page. Its meaning depends on the entry type.

## Entry types

SQLite defines five valid types:

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
  Target is a b-tree root page.
  It has no parent, so the four-byte field is unused.

PTRMAP_FREEPAGE
  Target is currently free.
  It has no parent, so the four-byte field is unused.

PTRMAP_OVERFLOW1
  Target is the first page of an overflow chain.
  The parent/owner is the b-tree page containing the cell whose payload
  points to this overflow page.

PTRMAP_OVERFLOW2
  Target is the second or a later page of an overflow chain.
  The parent is the previous overflow page.

PTRMAP_BTREE
  Target is a non-root b-tree page.
  The parent is the b-tree page containing its child-page pointer.
```

For `PTRMAP_ROOTPAGE` and `PTRMAP_FREEPAGE`, SQLite normally writes `0` into the
unused parent field.

## Direct lookup: page number to slot

SQLite first calculates which pointer-map page covers `key`:

```c
static Pgno ptrmapPageno(BtShared *pBt, Pgno pgno){
  int nPagesPerMapPage;
  Pgno iPtrMap, ret;
  assert( sqlite3_mutex_held(pBt->mutex) );
  if( pgno<2 ) return 0;
  nPagesPerMapPage = (pBt->usableSize/5)+1;
  iPtrMap = (pgno-2)/nPagesPerMapPage;
  ret = (iPtrMap*nPagesPerMapPage) + 2;
  if( ret==PENDING_BYTE_PAGE(pBt) ){
    ret++;
  }
  return ret;
}
```

The related macros are:

```c
#define PTRMAP_PAGENO(pBt, pgno) ptrmapPageno(pBt, pgno)
#define PTRMAP_PTROFFSET(pgptrmap, pgno) (5*(pgno-pgptrmap-1))
#define PTRMAP_ISPAGE(pBt, pgno) \
  (PTRMAP_PAGENO((pBt),(pgno))==(pgno))
```

`PTRMAP_PAGENO()` answers:

```text
Which pointer-map page describes target page key?
```

`PTRMAP_PTROFFSET()` answers:

```text
At which byte offset in that pointer-map page is key's entry?
```

For example, with `key = 7` and pointer-map page 2:

```text
offset = 5 * (key - pointerMapPage - 1)
       = 5 * (7 - 2 - 1)
       = 20
```

SQLite reads the type at offset 20 and the backlink at offsets 21 through 24.

## Pointer-map page placement

The number of ordinary pages covered by one map page is:

```text
entriesPerMapPage = usableSize / 5
```

The placement interval also includes the pointer-map page itself:

```c
nPagesPerMapPage = (pBt->usableSize/5) + 1;
```

With 4096 usable bytes:

```text
4096 / 5 = 819 entries
placement interval = 819 + 1 = 820 pages
```

Ignoring the pending-byte-page exception for the moment:

```text
pointer-map page 2    describes pages 3..821
pointer-map page 822  describes pages 823..1641
pointer-map page 1642 describes pages 1643..2461
```

Pointer-map pages themselves are reserved structural pages. They do not have
entries describing themselves. Page 1 also has no pointer-map entry.

If a normally calculated pointer-map page would be the pending-byte page,
SQLite moves that pointer-map page to the following page. The pending-byte page
is reserved for the file-locking layout and is never used as an ordinary
database page.

## `ptrmapGet()`

The function shape is:

```c
static int ptrmapGet(
  BtShared *pBt,
  Pgno key,
  u8 *pEType,
  Pgno *pPgno
)
```

The arguments mean:

```text
pBt      shared b-tree/database state
key      page whose pointer-map entry should be read
pEType   required output for its entry type
pPgno    optional output for its parent/backlink page number
```

The important distinction is:

```text
key identifies the target page
entry position is calculated from key
the entry stores type + parent, not key itself
```

The lookup is:

```c
iPtrmap = PTRMAP_PAGENO(pBt, key);
rc = sqlite3PagerGet(pBt->pPager, iPtrmap, &pDbPage, 0);
...
offset = PTRMAP_PTROFFSET(iPtrmap, key);
...
*pEType = pPtrmap[offset];
if( pPgno ) *pPgno = get4byte(&pPtrmap[offset+1]);
```

Step by step:

```text
1. Compute the pointer-map page number from key.
2. Ask the pager for that page.
3. Compute key's byte offset within it.
4. Read the one-byte type.
5. If requested, read the four-byte parent/backlink.
6. Release the pager reference.
7. Reject type values outside 1..5 as corruption.
```

### `0` as a null pointer

In C, `0` used as a pointer means a null pointer. SQLite commonly uses `0`
where other code uses `NULL`, so these are equivalent here:

```c
assert( pEType!=0 );
assert( pEType!=NULL );
```

The assertion checks that `pEType` is an address, not that `*pEType` already
contains a nonzero value.

### Why is `pPgno` optional?

Only the output pointer is optional. The four bytes still exist in every
on-disk entry.

A caller that only needs to test the page type can pass `0`:

```c
u8 eType;
rc = ptrmapGet(pBt, nearby, &eType, 0);
```

The final `0` is a null pointer for `pPgno`, meaning that the caller does not
want the parent-page output.

For example, `allocateBtreePage()` uses this to ask whether an exact target is
free:

```c
if( eType==PTRMAP_FREEPAGE ){
  searchList = 1;
}
```

It already knows the target page number (`nearby`) and does not need that
page's parent, so retrieving the parent would add no useful information.

This does not make the stored parent field optional, and it does not mean the
type is found by using the parent as an index.

## `ptrmapPut()`

The write function is:

```c
static void ptrmapPut(
  BtShared *pBt,
  Pgno key,
  u8 eType,
  Pgno parent,
  int *pRC
)
```

It calculates the same pointer-map page and offset:

```c
iPtrmap = PTRMAP_PAGENO(pBt, key);
offset = PTRMAP_PTROFFSET(iPtrmap, key);
```

Then it updates the fixed slot:

```c
if( eType!=pPtrmap[offset]
 || get4byte(&pPtrmap[offset+1])!=parent
){
  *pRC = rc = sqlite3PagerWrite(pDbPage);
  if( rc==SQLITE_OK ){
    pPtrmap[offset] = eType;
    put4byte(&pPtrmap[offset+1], parent);
  }
}
```

SQLite first compares the requested value with the current value. If nothing
changed, it avoids dirtying the pointer-map page. Otherwise it asks the pager
to make the page writable before changing the five bytes, so normal rollback
journal or WAL transaction handling applies.

`pRC` follows SQLite's accumulated-error pattern:

```c
if( *pRC ) return;
```

If an earlier operation already failed, `ptrmapPut()` is a no-op. If this write
fails, it stores the error in `*pRC` for the caller.

## What happens when a page is freed or reused?

Suppose page 7 is currently a child of b-tree page 3:

```text
entry for page 7 = { PTRMAP_BTREE, 3 }
```

When page 7 is freed, SQLite overwrites the same slot:

```text
entry for page 7 = { PTRMAP_FREEPAGE, 0 }
```

When page 7 is later reused as the first overflow page owned by b-tree page 12,
SQLite overwrites it again:

```text
entry for page 7 = { PTRMAP_OVERFLOW1, 12 }
```

Nothing is inserted into or deleted from the pointer-map page. The slot always
corresponds to page 7.

## Relationship with the freelist

The pointer map and freelist answer different questions:

```text
pointer map:
  What is page N, and which page points to it?

freelist:
  Which pages are available for allocation?
```

When a page becomes free, SQLite performs both kinds of bookkeeping:

```text
1. Add/represent the page in the freelist structure.
2. Change its fixed pointer-map entry to PTRMAP_FREEPAGE.
```

When that page is allocated again:

```text
1. Remove/claim it through the freelist structure.
2. Overwrite its fixed pointer-map entry with its new type and backlink.
```

The pointer-map entry does not say whether a free page is a freelist trunk or a
freelist leaf. Both are simply `PTRMAP_FREEPAGE`. The freelist structure holds
the information needed to enumerate and allocate free pages.

## How this helps autovacuum move a page

Imagine that page 7 is a free hole and page 100 is the last live page in the
database:

```text
... page 7 [free] ... page 100 [live] EOF
```

Autovacuum wants to move page 100 into page 7, then truncate the old last page.
Before moving it, SQLite reads page 100's pointer-map entry:

```text
entry for page 100 = { PTRMAP_BTREE, 42 }
```

Now SQLite knows page 42 contains the child pointer to page 100. It can:

```text
1. Copy/relocate page 100 to page 7.
2. Change the child pointer in page 42 from 100 to 7.
3. Record page 7's correct pointer-map entry.
4. Update backlinks for pages referenced by the moved page when necessary.
5. Remove/truncate the old end page when it is safe to do so.
```

Without the reverse lookup, SQLite would have to search b-tree and overflow
structures to discover who points to page 100.

## Mental model

The compact mental model is:

```text
The target page number chooses the slot.
The slot stores the target page's role and reverse pointer.
```

Or, using page 7 again:

```text
key = 7
  -> calculate pointer-map page
  -> calculate page 7's fixed 5-byte slot
  -> read/write { type, parent-or-owner }
```

`pPgno` being optional in `ptrmapGet()` only means a caller may decline the
second output. It does not change the on-disk entry layout.
