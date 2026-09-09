# `allocateBtreePage()`

This note explains SQLite's internal `allocateBtreePage()` function.

The job of this function is:

```text
choose a database page number
  -> obtain that page through the b-tree/pager layer
  -> mark it writable/dirty
  -> return both the MemPage pointer and page number
```

It is used by code such as `btreeCreateTable()` when SQLite needs a fresh page
for a new b-tree root, overflow page, split page, or other storage structure.

## Function shape

```c
static int allocateBtreePage(
  BtShared *pBt,         /* The btree */
  MemPage **ppPage,      /* Store pointer to the allocated page here */
  Pgno *pPgno,           /* Store the page number here */
  Pgno nearby,           /* Search for a page near this one */
  u8 eMode               /* BTALLOC_EXACT, BTALLOC_LT, or BTALLOC_ANY */
)
```

The outputs are:

```text
*ppPage  -> referenced MemPage for the allocated page
*pPgno   -> page number of that page
```

## Why `MemPage **ppPage`?

C passes arguments by value, including pointer arguments.

With `MemPage *pPage`, the callee receives a copy of the caller's pointer. It
can modify the shared page object through `pPage->...`, but this assignment only
changes the callee's local pointer variable:

```c
pPage = newlyAllocatedPage;  /* caller's pointer is unchanged */
```

`allocateBtreePage()` needs to choose a page and store that chosen `MemPage *`
back into the caller's pointer variable. So the caller passes the address of its
pointer:

```c
MemPage *pRoot = 0;
Pgno pgnoRoot = 0;

rc = allocateBtreePage(pBt, &pRoot, &pgnoRoot, 1, BTALLOC_ANY);
```

Inside the function, `ppPage` points at the caller's `pRoot` slot:

```text
before:
  caller pRoot = 0
  ppPage -------> caller's pRoot variable

assignment:
  *ppPage = selectedPage

after:
  caller pRoot -> selectedPage
```

Same idea for `pPgno`: `*pPgno = iTrunk` writes the chosen page number back into
the caller's `Pgno` variable.

The caller owns the returned page reference and must later unref/release it.

The returned page is already dirty:

```text
sqlite3PagerWrite() has already been called
```

So the caller can modify it without first asking the pager to make it writable.

## Allocation modes

At the top of `btree.c`:

```c
#define BTALLOC_ANY   0  /* Allocate any page */
#define BTALLOC_EXACT 1  /* Allocate exact page if possible */
#define BTALLOC_LE    2  /* Allocate any page <= the parameter */
```

The function comment calls the third mode `BTALLOC_LT`, but the actual local
constant in this checkout is `BTALLOC_LE`. The behavior is "less than or equal
to nearby."

The modes mean:

```text
BTALLOC_ANY
  choose any free/available page
  nearby is only a locality preference if it is nonzero

BTALLOC_EXACT
  if nearby is on the freelist, return exactly nearby
  used by autovacuum code that wants a specific page number

BTALLOC_LE
  choose a page <= nearby if possible
```

## `nearby`

`nearby` has two personalities.

With `BTALLOC_ANY`, it is a placement hint:

```c
allocateBtreePage(pBt, &pRoot, &pgnoRoot, 1, 0);
```

This means:

```text
allocate any page, but prefer one near page 1 if there is a choice
```

That makes sense for new root-ish pages in non-autovacuum databases.

With `BTALLOC_EXACT`, `nearby` becomes the target page number:

```c
allocateBtreePage(pBt, &pPageMove, &pgnoMove, pgnoRoot, BTALLOC_EXACT);
```

In `btreeCreateTable()`'s autovacuum path, `pgnoRoot` is the specific page
SQLite wants to become the new root page.

The picture is:

```text
page 1       database header + sqlite_schema root
page 2       first pointer-map page, if autovacuum is enabled
page 3+      ordinary pages/root-page candidates, except future pointer-map
             pages and the pending-byte page

root1
root2
...
pgnoRoot     desired next root-page slot
```

So in that case `nearby` is not just "close to here." It is "try to allocate
this exact page if it is free; otherwise allocate a page that can receive the
current occupant when we move it away."

## Preconditions and setup

```c
assert( sqlite3_mutex_held(pBt->mutex) );
assert( eMode==BTALLOC_ANY || (nearby>0 && IfNotOmitAV(pBt->autoVacuum)) );
```

The caller must hold the shared b-tree mutex.

The second assertion says constrained allocation modes are only expected with a
nonzero `nearby`, and, unless autovacuum code is omitted, with autovacuum active.
That matches the main reason for exact/less-than allocation: page relocation in
autovacuum mode.

```c
pPage1 = pBt->pPage1;
mxPage = btreePagecount(pBt);
```

Get page 1 and the current number of pages in the database image.

```c
n = get4byte(&pPage1->aData[36]);
```

Read the freelist page count from page 1. In SQLite's file format, page 1 offset
36 stores the total number of pages on the freelist.

`get4byte()` reads a 4-byte big-endian integer from the raw page bytes.
Big-endian means the most significant byte comes first.

For example, the 4-byte value:

```text
0x12345678
```

is stored big-endian as:

```text
offset +0: 0x12
offset +1: 0x34
offset +2: 0x56
offset +3: 0x78
```

So if `pPage1->aData[36..39]` contains:

```text
00 00 00 05
```

then:

```c
get4byte(&pPage1->aData[36])
```

returns `5`.

SQLite uses a fixed byte order in its database file format so the same database
file can be read consistently on machines with different CPU byte orders.

```c
if( n>=mxPage ){
  return SQLITE_CORRUPT_BKPT;
}
```

If the freelist claims at least as many free pages as the whole database has,
something is wrong. Page 1 itself cannot be on the freelist, so `n` must be less
than `mxPage`.

## Two big paths

The function splits here:

```c
if( n>0 ){
  /* There are pages on the freelist. Reuse one. */
  ...
}else{
  /* There are no pages on the freelist. Append one. */
  ...
}
```

So the mental model is:

```text
freelist has pages
  -> recycle one

freelist is empty
  -> grow the database image
```

## Freelist shape

SQLite's freelist is stored inside the database file.

Page 1 contains:

```text
offset 32: page number of first freelist trunk page
offset 36: total number of freelist pages
```

These are not C struct fields like `pBt->freelistCount`. They are persistent
database-header fields stored in the raw bytes of page 1:

```text
pPage1->aData[32..35]
  first freelist trunk page number

pPage1->aData[36..39]
  total number of freelist pages
```

The code reads them with:

```c
iTrunk = get4byte(&pPage1->aData[32]);
n = get4byte(&pPage1->aData[36]);
```

Each freelist trunk page contains:

```text
offset 0: page number of next trunk page, or 0
offset 4: number of leaf page pointers on this trunk
offset 8: first free leaf page number
offset 12: second free leaf page number
...
```

So a freelist looks like:

```text
page 1
  -> first trunk page
       -> leaf free page
       -> leaf free page
       -> next trunk page
            -> leaf free page
            -> ...
```

A trunk page is itself a free page, but it also stores pointers to more free
pages.

A leaf free page is just a free page named by a trunk page.

The words "trunk" and "leaf" here belong to the freelist, not the table/index
b-tree:

```text
freelist trunk page
  a free database page whose bytes store freelist bookkeeping

freelist leaf page
  a free database page listed by a freelist trunk
  not the same thing as a b-tree leaf page
```

If page 20 is the first freelist trunk, page 1 stores only the pointer to page
20. The freelist details then live inside page 20's own bytes:

```text
page 1:
  pPage1->aData[32..35] = 20

page 20:
  pTrunk->aData[0..3]   = next trunk page number
  pTrunk->aData[4..7]   = leaf count, called k
  pTrunk->aData[8..11]  = first freelist leaf page number
  pTrunk->aData[12..15] = second freelist leaf page number
  ...
```

The code loads page 20 as `pTrunk` and reads:

```c
k = get4byte(&pTrunk->aData[4]);
iPage = get4byte(&pTrunk->aData[8 + i*4]);
```

So the shape is:

```text
page 1 header
  -> first trunk page number

trunk page
  -> next trunk page number
  -> array of freelist leaf page numbers
```

## Reusing a freelist page

When `n>0`, SQLite will reuse a page from the freelist.

For exact allocation in autovacuum mode:

```c
if( eMode==BTALLOC_EXACT ){
  if( nearby<=mxPage ){
    rc = ptrmapGet(pBt, nearby, &eType, 0);
    if( eType==PTRMAP_FREEPAGE ){
      searchList = 1;
    }
  }
}
```

The pointer map can answer:

```text
is nearby currently a free page?
```

If yes, SQLite searches the entire freelist to find that exact page.

For `BTALLOC_LE`, it also searches:

```c
}else if( eMode==BTALLOC_LE ){
  searchList = 1;
}
```

because it needs to find a suitable page number less than or equal to `nearby`.

Before removing a page from the freelist, SQLite decrements the freelist count:

```c
rc = sqlite3PagerWrite(pPage1->pDbPage);
put4byte(&pPage1->aData[36], n-1);
```

Page 1 is made writable, then the count at offset 36 is reduced by one.

The loop then walks freelist trunk pages:

```c
do {
  ...
}while( searchList );
```

If `searchList` is false, the loop normally runs once. If `searchList` is true,
SQLite walks trunk by trunk until it finds the exact/suitable page.

## Case 1: use an empty trunk page

```c
if( k==0 && !searchList ){
  ...
}
```

`k` is the number of leaf pointers stored on the trunk page:

```c
k = get4byte(&pTrunk->aData[4]);
```

If the trunk has no leaves and SQLite is not searching for a special page, the
trunk page itself is removed from the freelist and returned as the allocation.

This updates page 1's first-trunk pointer:

```c
memcpy(&pPage1->aData[32], &pTrunk->aData[0], 4);
```

The first trunk becomes the next trunk.

## Case 2: search finds a trunk page

```c
}else if( searchList
      && (nearby==iTrunk || (iTrunk<nearby && eMode==BTALLOC_LE))
){
  ...
}
```

If the page SQLite wants is itself a trunk page, SQLite may allocate the trunk.

If that trunk has no leaves, unlinking it is simple: page 1 or the previous
trunk points to the next trunk.

If that trunk has leaf pointers, SQLite cannot simply discard those pointers.
So it promotes the first leaf page into a new trunk:

```text
old trunk page is allocated to caller
first leaf page becomes replacement trunk
remaining leaf pointers move into the replacement trunk
```

That is this part:

```c
Pgno iNewTrunk = get4byte(&pTrunk->aData[8]);
...
memcpy(&pNewTrunk->aData[0], &pTrunk->aData[0], 4);
put4byte(&pNewTrunk->aData[4], k-1);
memcpy(&pNewTrunk->aData[8], &pTrunk->aData[12], (k-1)*4);
```

## Case 3: extract a leaf page

```c
}else if( k>0 ){
  ...
}
```

If the trunk has free leaf pages, SQLite usually allocates one of those leaves.

If `nearby>0`, it chooses the closest leaf page to `nearby`:

```c
dist = sqlite3AbsInt32(get4byte(&aData[8]) - nearby);
...
if( d2<dist ){
  closest = i;
  dist = d2;
}
```

For `BTALLOC_LE`, it chooses a leaf page less than or equal to `nearby`.

Once it chooses a leaf page, SQLite removes that page number from the trunk's
leaf array by replacing it with the last leaf entry:

```c
if( closest<k-1 ){
  memcpy(&aData[8+closest*4], &aData[4+k*4], 4);
}
put4byte(&aData[4], k-1);
```

Then it fetches the chosen page:

```c
rc = btreeGetUnusedPage(pBt, *pPgno, ppPage, noContent);
```

and marks it writable:

```c
rc = sqlite3PagerWrite((*ppPage)->pDbPage);
```

## `noContent`

When SQLite allocates a page, it often does not need the old bytes currently on
that page. It is going to overwrite the page.

So it can ask the pager not to read old content from disk:

```c
PAGER_GET_NOCONTENT
```

For a freelist leaf:

```c
noContent = !btreeGetHasContent(pBt, *pPgno) ? PAGER_GET_NOCONTENT : 0;
```

The idea:

```text
if old content is irrelevant and not needed for rollback
  -> avoid reading it

if old content may matter
  -> let pager read/journal it
```

## Appending a new page

If the freelist is empty:

```c
}else{
  /* There are no pages on the freelist, so append a new page */
}
```

SQLite grows the database image:

```c
rc = sqlite3PagerWrite(pBt->pPage1->pDbPage);
pBt->nPage++;
```

If the new page number lands on the pending-byte page, SQLite skips it:

```c
if( pBt->nPage==PENDING_BYTE_PAGE(pBt) ) pBt->nPage++;
```

In an autovacuum database, the new page number might land on a pointer-map page:

```c
if( pBt->autoVacuum && PTRMAP_ISPAGE(pBt, pBt->nPage) ){
  ...
}
```

If so, SQLite allocates that page too, but it becomes a pointer-map page. Then
SQLite increments again and returns the next page to the caller:

```text
append page N
  if N is pointer-map page:
    initialize/write page N as pointer-map page
    append page N+1 for caller
```

Then page 1's database-size metadata is updated:

```c
put4byte(28 + (u8*)pBt->pPage1->aData, pBt->nPage);
```

Page 1 offset 28 stores the database size in pages.

Finally SQLite fetches and dirties the new page:

```c
*pPgno = pBt->nPage;
rc = btreeGetUnusedPage(pBt, *pPgno, ppPage, bNoContent);
rc = sqlite3PagerWrite((*ppPage)->pDbPage);
```

## Cleanup and invariants

Before returning:

```c
releasePage(pTrunk);
releasePage(pPrevTrunk);
```

Any temporary trunk-page references are released.

Then debug checks verify the output page is in the expected state:

```c
assert( rc!=SQLITE_OK || sqlite3PagerPageRefcount((*ppPage)->pDbPage)<=1 );
assert( rc!=SQLITE_OK || (*ppPage)->isInit==0 );
```

The returned page is referenced, but should not have extra unexpected
references.

The returned `MemPage` wrapper should not yet be initialized as a b-tree page.
The caller will decide the page shape later, often by calling `zeroPage()`.

## Compact flow

```text
allocateBtreePage()
  -> read freelist count from page 1 offset 36

  if freelist has pages:
    -> optionally search for nearby/exact page
    -> decrement freelist count
    -> remove a trunk or leaf page from the freelist
    -> fetch it as MemPage
    -> mark it writable
    -> return page + page number

  else:
    -> increment database page count
    -> skip pending-byte page
    -> in autovacuum, reserve pointer-map page if formula lands on one
    -> update page 1 database-size metadata at offset 28
    -> fetch new end page as MemPage
    -> mark it writable
    -> return page + page number
```

## Main mental model

`allocateBtreePage()` is the allocator for SQLite database pages.

It first tries to recycle space from the freelist. If no free page exists, it
extends the database image. In autovacuum mode, exact page choice matters because
root pages and pointer-map pages have layout rules. The `nearby` argument is
therefore sometimes a locality hint and sometimes the exact page number SQLite
is trying to make available.
