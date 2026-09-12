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

## Local variables

At the start of the function:

```c
MemPage *pPage1;
int rc;
u32 n;     /* Number of pages on the freelist */
u32 k;     /* Number of leaves on the trunk of the freelist */
MemPage *pTrunk = 0;
MemPage *pPrevTrunk = 0;
Pgno mxPage;     /* Total size of the database file */
```

The `= 0` initializers on `pTrunk` and `pPrevTrunk` are necessary because
automatic local variables in C are not initialized by default. Without an
initializer:

```c
MemPage *pTrunk;  /* indeterminate pointer value, not automatically NULL */
```

Reading that value in `if( pTrunk )` or passing it to `releasePage(pTrunk)`
would be undefined behavior. Initializing it with `0` creates a null pointer
that safely means "no page has been acquired yet".

Other locals can omit an initializer only when every path assigns them before
their first read. Initialization is about preventing an indeterminate value
from being used, not a requirement that every local declaration contain `= 0`.

`pPage1` is a local pointer to page 1:

```c
pPage1 = pBt->pPage1;
```

Page 1 contains database-header fields, including the first freelist trunk page
and the total freelist page count.

`rc` is the usual SQLite return-code variable. The function stores errors there
and returns early when something fails.

`n` is the total number of pages on the freelist. It is read from page 1 offset
36:

```c
n = get4byte(&pPage1->aData[36]);
```

`k` is the number of freelist leaf page numbers stored on the current freelist
trunk page:

```text
trunk page
  offset 0  -> next trunk page
  offset 4  -> k, number of leaf pointers
  offset 8  -> first leaf page number
  offset 12 -> second leaf page number
```

`pTrunk` points to the current freelist trunk page being inspected.
`pPrevTrunk` points to the previous trunk page, which matters if SQLite removes
a trunk from the middle of the freelist and has to relink the chain.

```text
pPrevTrunk        pTrunk
    |               |
    v               v
 trunk 20  -----> trunk 35 -----> trunk 50
```

`mxPage` is the current database size in pages:

```c
mxPage = btreePagecount(pBt);
```

SQLite uses it for corruption checks, for example to reject freelist page
numbers that are beyond the end of the database.

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

For normal non-root pages, `nearby` is usually a related page number:

```text
overflow page
  -> prefer a page near the previous overflow page

page split
  -> prefer a page near the page being split
```

This is only a locality hint. Nearby page numbers mean nearby offsets inside the
database file, not guaranteed nearby physical disk sectors. The benefit is
practical and probabilistic: the operating system, filesystem, SQLite page
cache, and storage device may handle nearby file offsets with better readahead
and cache behavior. This especially helps overflow chains and range scans across
neighboring b-tree leaf pages.

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

Reading the `||` explicitly makes the mode requirements easier to see:

```text
eMode == BTALLOC_ANY
    OR
(nearby > 0 AND pBt->autoVacuum)
```

In a build that includes autovacuum support, this gives:

| Allocation mode | Required value of `pBt->autoVacuum` | Other requirement |
| --- | --- | --- |
| `BTALLOC_ANY` | Either true or false | None from this assertion |
| `BTALLOC_EXACT` | True | `nearby > 0` |
| `BTALLOC_LE` | True | `nearby > 0` |

For `BTALLOC_ANY`, the left side of `||` is true, so C short-circuits and the
right side does not impose any requirement. For either constrained mode, the
left side is false, so the entire right side must be true:

```text
BTALLOC_EXACT or BTALLOC_LE
  -> nearby > 0
  -> pBt->autoVacuum == true
```

Thus, it is not only `BTALLOC_EXACT` that requires autovacuum. `BTALLOC_LE`
also requires it because it chooses an earlier free destination for a page move;
after the move, autovacuum can truncate the old page at the end of the file.
Some source comments use the older name `BTALLOC_LT`; the constant in this
checkout is `BTALLOC_LE`, meaning less than or equal to `nearby`.

### Compile-time support versus runtime mode

These two checks answer different questions:

```c
#ifndef SQLITE_OMIT_AUTOVACUUM
```

asks whether this SQLite binary was compiled with autovacuum support. In
contrast:

```c
assert( pBt->autoVacuum );
```

asks whether the particular database currently being accessed is an
autovacuum database. One SQLite binary can support autovacuum while opening
both kinds of database:

```text
SQLite build includes autovacuum support
  database A: pBt->autoVacuum = 1
  database B: pBt->autoVacuum = 0
```

When `pBt->autoVacuum` is true, the database contains pointer-map pages. A
useful way to remember the relationship is:

```text
autovacuum may move pages
  -> moving a page requires finding who points to it
  -> pointer-map pages provide that reverse lookup
```

Ordinary b-tree links mainly give SQLite the forward relationship:

```text
parent -> child
```

The pointer map supplies information in the reverse direction:

```text
child -> parent/type
```

This is why the memorable rule is: **autovacuum moves pages; pointer maps make
pages movable**. The separate `pBt->incrVacuum` flag distinguishes incremental
autovacuum from full autovacuum.

Autovacuum's page movement is also why `allocateBtreePage()` has constrained
allocation modes. Ordinary allocation usually needs `BTALLOC_ANY`. Autovacuum
may need `BTALLOC_EXACT` to remove or claim one particular page, or `BTALLOC_LE`
to obtain a free page below a truncation boundary:

```text
before:  [used] [free] [used] [last used page]
                       ^
          move the last page into an earlier free slot
          then truncate the old end of the file
```

With `SQLITE_OMIT_AUTOVACUUM` defined, `IfNotOmitAV(expr)` expands to `0`:

```c
#ifndef SQLITE_OMIT_AUTOVACUUM
# define IfNotOmitAV(expr) (expr)
#else
# define IfNotOmitAV(expr) 0
#endif
```

The allocation assertion then effectively requires `eMode==BTALLOC_ANY`, and
the preprocessor removes the exact/less-than freelist searches and pointer-map
page handling. `allocateBtreePage()` itself is still required because every
database needs to allocate ordinary b-tree and overflow pages.

Inside the compiled autovacuum branch, SQLite repeats the runtime assertion
immediately before using the pointer map:

```c
if( eMode==BTALLOC_EXACT ){
  ...
  assert( pBt->autoVacuum );
  rc = ptrmapGet(pBt, nearby, &eType, 0);
}
```

The earlier allocation assertion already implies this condition, so the local
assertion is logically redundant. It is still valuable because it documents
and checks the requirement exactly where `ptrmapGet()` depends on pointer-map
pages being present. `#ifndef SQLITE_OMIT_AUTOVACUUM` alone cannot establish
that runtime condition; it only establishes that support exists in the binary.

### Manual `VACUUM` is separate

Disabling autovacuum for a database does not prevent:

```sql
VACUUM;
```

The mechanisms are different:

```text
VACUUM
  explicitly rebuilds the whole database into a compact database image

auto_vacuum
  uses pointer maps to relocate individual pages and reclaim pages from the end
```

Therefore, `PRAGMA auto_vacuum=NONE` and a later manual `VACUUM` are compatible.
Likewise, `SQLITE_OMIT_AUTOVACUUM` removes automatic and incremental autovacuum
support, but does not by itself remove the SQL `VACUUM` command. That command
has its own compile-time omission path, `SQLITE_OMIT_VACUUM` (and in this source
is also unavailable when attach support is omitted).

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

`pPage1->aData` is a pointer to page 1's bytes in memory. The pager has already
made the database page available as a contiguous memory view, so ordinary array
indexing can access each byte:

```text
pPage1->aData       address of page 1's first byte in memory
aData[36]           byte at offset 36 in that memory buffer
&aData[36]          address of that byte
get4byte(...)       read bytes 36, 37, 38, and 39
```

The address passed to `get4byte()` is therefore a memory address, not a disk
address. Conceptually, the pager provides this mapping:

```text
logical database file             page 1 memory view

file offset 0    ----------------> aData[0]
file offset 1    ----------------> aData[1]
...
file offset 36   ----------------> aData[36]
file offset 37   ----------------> aData[37]
file offset 38   ----------------> aData[38]
file offset 39   ----------------> aData[39]
```

The bytes are consecutive in SQLite's logical database file even if the
filesystem stores the underlying blocks in different physical locations. The
filesystem hides that physical layout, and SQLite asks for data using logical
file offsets. The pager then presents the requested page through contiguous
virtual memory, either from its page cache or through an equivalent mapped
view.

SQLite page numbers start at 1. In general, byte offset `x` within page `N`
corresponds to this logical file position:

```text
file offset = (N - 1) * page_size + x
```

Because page 1 starts at file offset 0, `pPage1->aData[36]` corresponds directly
to logical database-file offset 36.

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

Separately, SQLite's C code may define `SQLITE_BYTEORDER` at build time:

```c
#if defined(__BYTE_ORDER__) && __BYTE_ORDER__==__ORDER_LITTLE_ENDIAN__
# define SQLITE_BYTEORDER 1234
#endif
```

Macros like `__BYTE_ORDER__` are usually predefined by the compiler. The CPU or
target architecture determines the endian convention; the compiler reports that
target information through macros; SQLite reads those macros during
preprocessing. No runtime CPU check is needed for this path.

The `1234` value means little-endian, and `4321` means big-endian. This is about
how a multi-byte value is laid out in memory, not how the number exists inside a
CPU register:

```text
number:          0x12345678
big-endian:      12 34 56 78
little-endian:   78 56 34 12
```

Little-endian feels backwards because humans write the most significant part
first. The practical convenience is that the lowest-address byte is also the
least significant byte:

```text
memory at p:  78 56 34 12

32-bit value at p -> 0x12345678
16-bit value at p -> 0x5678
 8-bit value at p -> 0x78
```

That can be convenient for sub-word access and arithmetic, where carries start
from the least significant side. Big-endian has the human-friendly property that
memory order matches written numeric order. SQLite's file format chooses one
fixed order; `SQLITE_BYTEORDER` describes the CPU's native order so SQLite knows
when it can read directly and when it must swap bytes.

The implementation of `sqlite3Get4byte()` uses that information to choose a
fast path when possible:

```c
u32 sqlite3Get4byte(const u8 *p){
#if SQLITE_BYTEORDER==4321
  u32 x;
  memcpy(&x,p,4);
  return x;
#elif SQLITE_BYTEORDER==1234 && GCC_VERSION>=4003000
  u32 x;
  memcpy(&x,p,4);
  return __builtin_bswap32(x);
#elif SQLITE_BYTEORDER==1234 && MSVC_VERSION>=1300
  u32 x;
  memcpy(&x,p,4);
  return _byteswap_ulong(x);
#else
  testcase( p[0]&0x80 );
  return ((unsigned)p[0]<<24) | (p[1]<<16) | (p[2]<<8) | p[3];
#endif
}
```

The branches mean:

```text
known big-endian
  copy the four file bytes directly into the u32

known little-endian with a supported compiler builtin
  copy the bytes, then reverse them with one byte-swap operation

anything else
  construct the numeric value explicitly from four separate byte values
```

The final branch does not assume that the CPU is big-endian. For input bytes
`12 34 56 78`, each array access first reads a one-byte number, which has no
endianness:

```text
p[0] = 0x12  ->  p[0] << 24 = 0x12000000
p[1] = 0x34  ->  p[1] << 16 = 0x00340000
p[2] = 0x56  ->  p[2] <<  8 = 0x00005600
p[3] = 0x78                  = 0x00000078
                                      OR
                              0x12345678
```

Shifts and bitwise OR operate on numeric values, so this produces the same
`u32` value on either endian architecture. If that `u32` is later stored in
memory, a little-endian CPU will lay out its bytes as `78 56 34 12`, but its
numeric value remains `0x12345678`.

Therefore, the direct `memcpy()` branch and the shift expression are equivalent
on a big-endian CPU. On a little-endian CPU, direct `memcpy()` alone would yield
the wrong numeric value, so SQLite either swaps the copied value or uses the
portable shift expression. The cast to `unsigned` before shifting `p[0]` also
makes a high top byte, such as `0x80`, safe to shift into the highest eight bits.

Byte-swapping and copying are different operations. A compiler builtin such as
`__builtin_bswap32(x)` reverses the byte order of a 32-bit value:

```text
0x12345678 -> 0x78563412
```

That is useful for endian conversion. `memcpy()` does not do that. It copies raw
bytes in the same order:

```text
source:       12 34 56 78
destination:  12 34 56 78
```

SQLite has a test-only `SQLITE_INLINE_MEMCPY` option that replaces `memcpy()`
with a simple byte-at-a-time loop:

```c
int xxn = N;
while( xxn-- > 0 ){
  *(xxd++) = *(xxs++);
}
```

That loop is slow because it copies one byte per iteration. Its purpose is
profiling, not production speed. Normal `memcpy()` may be a compiler builtin or
a highly optimized libc routine, so Cachegrind may attribute the work to
compiler/libc internals. The inline macro makes the copy cost appear at the
SQLite source line that called `memcpy()`, which helps measure where SQLite is
doing copies.

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

`iTrunk` is the integer database page number identifying the current trunk. It
is not a pointer and not an array index. `pTrunk` is the corresponding pointer
to the in-memory page object:

```text
iTrunk = 20              database page number/identifier

btreeGetUnusedPage(pBt, iTrunk, &pTrunk, 0)
                              |
                              -> pTrunk now points to the MemPage for page 20
```

This follows a common SQLite naming distinction: `i...` holds an integer
identifier or position, while `p...` holds a pointer.

### Why `btreeGetUnusedPage()` is not just `btreeGetPage()`

`btreeGetUnusedPage()` first fetches the requested page through
`btreeGetPage()`, then enforces that an allocation candidate is safe to
repurpose:

```text
pager refcount == 1  only this fetch holds the page
pager refcount > 1   another user still holds it; report corruption
```

A page identified by the freelist is supposed to be free. If another cursor or
operation is using it, the freelist and b-tree disagree, so SQLite must not
overwrite it. The function also sets `isInit = 0` so stale b-tree metadata is
not reused when the caller gives the page a new role.

Thus "unused" does not mean "absent from the cache". It means that the page
has no other active user and is safe to reformat or overwrite.

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

It is not the next-trunk page number. These are separate fields:

```text
pTrunk->aData[0..3]  next trunk page number
pTrunk->aData[4..7]  k, number of leaf page numbers
```

Therefore `k==0` only means that this trunk has no leaves. Its offset-0 field
may still point to another trunk.

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
