# SQLite storage basics

This note builds the minimum mental model needed before tracing individual SQL
statements through SQLite storage.

The main idea:

```text
SQL tables
  -> b-trees
    -> fixed-size database pages
      -> raw bytes in a database file
```

SQLite mostly thinks in **page numbers**, not disk addresses and not C pointers.

## SQLite as an embedded database

SQLite is not a toy database and it is not a server process. It is an embedded C
library that an application links into its own process.

The basic shape is:

```text
application code
  -> calls SQLite library
    -> opens a local .db/.sqlite file
      -> runs SQL
        -> SQLite reads and writes ordinary files through the VFS/OS
```

That is why SQLite is widely used in places where the database belongs to one
application, device, user profile, or local workspace:

```text
mobile apps:
  app sandbox file such as Notes.db or Cache.db
  iOS can use SQLite directly or through Core Data
  Android can use SQLite directly or through Room

browsers:
  profile databases for history, cookies, downloads, permissions, autofill,
  site-storage metadata, and internal indexes

desktop apps:
  preferences, recent files, search indexes, message history, sync queues,
  cache metadata, and offline documents

Electron / Node apps:
  local app-data directory plus a native SQLite binding

Python:
  built-in sqlite3 module opens a database file directly
```

The common pattern is not:

```text
app -> network -> database server
```

It is:

```text
app -> in-process SQLite library -> local database file
```

This makes SQLite feel small, but the smallness is the point. It provides a
durable SQL file format with ACID transactions, no separate server, and very few
deployment requirements. Its weak spot is many concurrent writers; its strength
is reliable local structured storage almost anywhere a program can read and
write files.

## File vs disk

To a program, a file looks like one ordered byte stream:

```text
byte 0
byte 1
byte 2
...
```

But that does not mean the file is physically stored as one consecutive chunk on
disk. The operating system and filesystem map logical file offsets to physical
storage blocks.

So SQLite relies on this OS contract:

```text
read/write N bytes at logical file offset X
```

SQLite does not need to know where those bytes physically live on the SSD or
disk.

## Pages

SQLite divides a database file into fixed-size pages:

```text
page 1
page 2
page 3
...
```

Inside a single database file, every page has the same size. Common page sizes
are 4096 or 8192 bytes.

The page number maps to a file offset:

```text
file offset = (page_number - 1) * page_size
```

Example:

```text
page_size = 4096
page 1 -> offset 0
page 2 -> offset 4096
page 3 -> offset 8192
```

The database file itself can grow. It does not have a fixed size:

```text
small.db = 12 pages
large.db = 800000 pages
```

The fixed-size page is SQLite's storage unit. Rows are variable-size, but pages
are fixed-size.

## Why fixed-size pages?

Fixed-size pages make the storage layer simple and fast:

- page N can be found with arithmetic
- the pager can cache pages uniformly
- dirty-page tracking is page-based
- rollback journal and WAL records can refer to page numbers
- b-tree parent pages can store child page numbers
- freed pages can be reused through the freelist

The alternative, such as one file per page, would create many tiny files and make
atomic transactions, copying, caching, and path lookup much harder.

SQLite's file is conceptually:

```text
database = array of fixed-size pages
```

## Pager

The pager is the layer between the b-tree code and the OS/VFS:

```text
SQL -> VDBE -> b-tree -> pager -> VFS -> OS file
```

The b-tree layer asks for logical pages:

```text
give me page 42
mark page 42 writable
release page 42
```

The pager handles the connection between:

```text
logical page number
  <-> memory page buffer
  <-> database file bytes
```

It also owns the database-safety rules around that connection:

- page cache
- dirty-page tracking
- page reference counts
- rollback journal or WAL handling
- lock coordination between connections/processes
- fsync and durability ordering

So a good short definition is:

```text
The pager maps database page numbers to in-memory page buffers and makes changes
to those pages transactional.
```

The pager does not know whether page 42 is a table leaf, an index page, an
overflow page, or a freelist page. It mostly sees page numbers and raw bytes.

## Pager vs page cache

The pager is the manager. The page cache is one subsystem the pager uses.

```text
b-tree
  -> pager
    -> page cache
    -> rollback journal / WAL
    -> locks
    -> VFS / OS file
```

The page cache answers memory-cache questions:

```text
Do we already have page 42 in memory?
Can we keep this page around?
Which clean page can be evicted?
Which pages are dirty?
```

The pager answers transaction/storage questions:

```text
If page 42 is not cached, should I read it from the database file or WAL?
Before modifying page 42, has the old version been journaled?
Is this connection allowed to write right now?
When do dirty pages get flushed?
What happens if the transaction rolls back?
What happens after a crash?
```

The object relationship is roughly:

```text
Pager
  owns/uses PCache

PCache
  contains cached PgHdr pages

PgHdr / DbPage
  has pgno, pData, dirty flags, refcount, cache links
```

So when b-tree asks:

```c
sqlite3PagerGet(pPager, 42, &pDbPage, flags);
```

the pager checks the page cache. If page 42 is already there, the pager returns
the existing `PgHdr/DbPage`. If not, it fetches or allocates a cache entry and
fills `pData` from the database file, WAL, zeros, or another appropriate source.

Source:

- `sqlite/src/pcache.c:41` defines `struct PCache`.
- `sqlite/src/pcache.h:94` declares `sqlite3PcacheFetch()`.
- `sqlite/src/pcache.h:112` declares `sqlite3PcacheDirtyList()`.
- `sqlite/src/pager.c:5598` fetches a page from the pager's page cache.
- `sqlite/src/pager.c:5608` converts the cache object into a `PgHdr`.

## Page number vs memory pointer

A C pointer is only an in-memory address. It is not a durable storage address.

SQLite remembers persistent page locations using page numbers:

```text
Pgno        -> logical page number in the database file
file offset -> (Pgno - 1) * page_size
pointer     -> temporary RAM address for cached page bytes
```

In the b-tree layer:

```c
struct MemPage {
  Pgno pgno;        /* Page number for this page */
  u8 *aData;        /* Pointer to disk image of the page data */
  DbPage *pDbPage;  /* Pager page handle */
};
```

`MemPage.pgno` says which page this is. `MemPage.aData` points to the RAM copy
of that page's bytes.

`DbPage *pDbPage` is the b-tree layer's handle back to the pager-owned page
object. In this part of SQLite, `DbPage` is effectively the same type as
`PgHdr`:

```c
typedef struct PgHdr DbPage;
```

So `pDbPage` lets b-tree code ask the pager to do page-level work for this same
page:

```c
sqlite3PagerWrite(pPage->pDbPage); /* make writable / journal if needed */
sqlite3PagerUnref(pPage->pDbPage); /* release the page reference */
```

The same logical byte can be addressed two ways:

```text
disk/file identity:
  page number + offset inside page

memory access:
  base pointer + offset inside page
```

Example:

```text
page number = 7
page size   = 4096
cell offset = 3890

file offset = (7 - 1) * 4096 + 3890
memory addr = aData + 3890
```

Source:

- `sqlite/src/btreeInt.h:273` defines `struct MemPage`.
- `sqlite/src/btreeInt.h:295` has `u8 *aData`.
- `sqlite/src/btreeInt.h:301` has `DbPage *pDbPage`.
- `sqlite/src/pager.h:43` defines `DbPage` as an alias for `struct PgHdr`.

## Where page data lives in memory

The raw page bytes are stored by the pager/page-cache layer. The key structure is
`PgHdr`:

```c
struct PgHdr {
  sqlite3_pcache_page *pPage;
  void *pData;   /* Page data */
  void *pExtra;  /* Extra content */
  Pgno pgno;
};
```

`void *pData` means "pointer to raw bytes". SQLite usually casts it to `u8 *`,
where `u8` is an unsigned byte.

```text
PgHdr.pData  -> raw page bytes
MemPage.aData -> same raw page bytes
```

The b-tree layer converts a pager page into a `MemPage` like this:

```c
MemPage *pPage = (MemPage*)sqlite3PagerGetExtra(pDbPage);
pPage->aData = sqlite3PagerGetData(pDbPage);
pPage->pDbPage = pDbPage;
pPage->pgno = pgno;
```

So `MemPage` is mostly interpreted metadata. The page bytes themselves are in the
pager/cache page.

At least in memory, one page is a consecutive byte buffer:

```text
aData
  |
  v
address + 0
address + 1
address + 2
...
address + page_size - 1
```

That is why SQLite only needs the first byte address for the page. Every page
field, cell pointer, cell body, rowid, and payload is reached by adding an offset
to that base pointer:

```c
aData[0]
aData[100]
aData[cellOffset]
aData[cellOffset + 5]
```

Offsets are durable within a page. Pointers are temporary RAM addresses.

Source:

- `sqlite/src/pcache.h:25` defines `struct PgHdr`.
- `sqlite/src/pcache.h:27` has `void *pData`.
- `sqlite/src/btree.c:2308` has `btreePageFromDbPage()`.
- `sqlite/src/btree.c:2311` sets `pPage->aData = sqlite3PagerGetData(pDbPage)`.
- `sqlite/src/pager.c:7351` returns `pPg->pData` from `sqlite3PagerGetData()`.

## B-trees

A b-tree is a search tree designed for storage systems.

The central idea is to keep keys sorted, but group many keys together in each
tree node. Each node divides the key space into ranges.

A tiny conceptual b-tree might look like:

```text
root node:
  keys: 100, 250

  child A: keys <= 100
  child B: keys > 100 and <= 250
  child C: keys > 250
```

To find key `180`, search the root keys:

```text
180 is > 100 and <= 250
go to child B
```

Then repeat until reaching a leaf.

The reason databases like b-trees is fanout. One node can contain many keys and
many child pointers, so the tree is shallow:

```text
root
  -> interior
    -> leaf
```

Even a huge table might need only a few page reads to reach the right leaf.

Important b-tree invariants:

```text
keys are kept sorted
interior nodes route searches to child ranges
leaf nodes contain the final entries
all leaves are at the same depth
pages are kept neither overfull nor wastefully empty
```

This is different from a binary tree, where each node has one key and two
children. A database b-tree node may have hundreds of keys and child pointers.

That "hundreds" part matters. If each interior page can route to 200 child
pages, then height grows slowly:

```text
height 1: about 200 leaves
height 2: about 40,000 leaves
height 3: about 8,000,000 leaves
```

So lookup cost is not "scan all rows". It is closer to:

```text
search root page
search one interior page
search one leaf page
```

The searches inside each page are over sorted cells. The tree search chooses a
page; the page search chooses a cell.

## What self-balancing means

Self-balancing means writes repair the tree as part of the write operation.

It does not mean SQLite constantly moves every page into a perfect layout. It
means SQLite preserves the b-tree invariants after inserts and deletes:

```text
1. pages cannot stay overfull
2. pages should not stay too empty
3. every leaf remains the same distance from the root
4. parent separator keys continue to describe child key ranges
```

This is different from an AVL or red-black tree. Those are binary trees, and
they rebalance with rotations around individual nodes. SQLite's b-tree
rebalances whole pages of cells.

For an insert:

```text
1. Find the leaf page where the key belongs.
2. Insert the new cell into that page's sorted cell order.
3. If the page still fits, stop.
4. If the page overflows, rebalance that page with nearby sibling pages.
5. If a parent page must gain or change separator cells, update the parent.
6. If the parent overflows too, repeat upward.
7. If the root overflows, grow the tree by one level.
```

SQLite's implementation detail is neat: a page can temporarily contain
"overflow cells" in `MemPage.apOvfl[]`. That means "this cell belongs on this
page logically, but the page image does not have room for it yet." Then
`balance()` repairs the physical page layout.

For a delete:

```text
1. Remove the cell from the leaf page.
2. Free overflow pages if the cell's payload used any.
3. If the page is still dense enough, stop.
4. If the page is too empty, redistribute cells with siblings or merge pages.
5. If a parent separator changed or disappeared, update the parent.
6. If the root becomes unnecessary, shrink the tree by one level.
```

The important thing is that balancing is page-local first. SQLite does not
rebuild the whole tree after every write. It looks at the modified page, its
parent, and a few siblings, then may propagate the repair upward only if the
parent is affected.

Conceptually:

```text
before insert:

parent: [100 | 200]
children:
  A: keys <= 100
  B: keys 101..200
  C: keys > 200

insert key 175 into B

if B has room:
  only B changes

if B overflows:
  SQLite redistributes/splits B with siblings
  parent separator cells are adjusted
```

The "self" in self-balancing just means the b-tree code does this automatically
during `INSERT`, `DELETE`, and related structural edits. The user does not run a
separate repair step to keep lookups efficient.

Source:

- `sqlite/src/btree.c:9121` starts `balance()`, the central repair routine.
- `sqlite/src/btree.c:9142` skips rebalancing when the page is not overfull and
  still dense enough.
- `sqlite/src/btree.c:9151` calls `balance_deeper()` when the root page is
  overfull.
- `sqlite/src/btree.c:9191` uses `balance_quick()` for the common rightmost
  append case.
- `sqlite/src/btree.c:9210` calls `balance_nonroot()` to redistribute cells
  between a page and sibling pages.
- `sqlite/src/btree.c:9660` calls `balance()` after insert creates overflow
  cells.
- `sqlite/src/btree.c:9985` describes delete-time tree balancing.

## SQLite b-trees

A table is usually stored as a b-tree. In SQLite, b-tree nodes are database
pages:

```text
root page
  -> child page
    -> leaf page
```

The links between pages are page numbers, not C pointers:

```text
child page number = 42
```

When SQLite needs child page 42, the b-tree layer asks the pager for page 42.
The pager maps that to a file offset and returns a cached memory page.

The tree shape is page-to-page, not cell-to-cell:

```text
tree structure = pages linked by child page numbers
ordering inside a page = cells sorted by key
```

So a b-tree is not:

```text
cell -> child cell -> child cell
```

It is:

```text
page
  sorted cells
  child page numbers for key ranges
```

Interior pages use their cells as separators. Leaf pages hold the actual table
entries.

There are two common b-tree styles:

```text
table b-tree
  key   = integer rowid
  value = encoded SQL record

index b-tree
  key   = encoded index record
  value = usually no separate table payload
```

In `MemPage`:

```c
u8 intKey; /* True if table b-trees. False for index b-trees */
```

For a normal rowid table, `intKey` is true.

Source:

- `sqlite/src/btreeInt.h:275` documents `intKey`.

## Interior vs leaf pages

For a normal rowid table, the actual row payload lives in leaf pages.

```text
table leaf cell
  payload size
  rowid
  record payload
```

Interior table pages are navigation pages. They do not contain SQL record
payload. Each interior cell contains:

```text
4 bytes: left child page number
varint:  separator rowid
```

The page header also has a separate right-most child page number.

Conceptually:

```text
interior table page

cell 0:
  child page 12
  separator rowid 100

cell 1:
  child page 18
  separator rowid 250

right-most child:
  page 40
```

This means roughly:

```text
page 12 has rows <= 100
page 18 has rows > 100 and <= 250
page 40 has rows > 250
```

So the space in an interior table page is used by:

```text
page header
cell pointer array
small navigation cells
free space / freeblocks / fragments
```

No table row data is stored there.

Index b-trees are different: index keys are record-like payloads, so index
interior pages can carry key payload. This section is about ordinary rowid table
b-trees.

Source:

- `sqlite/src/btreeInt.h:134` describes the right child pointer.
- `sqlite/src/btreeInt.h:189` shows the general cell format.
- `sqlite/src/btreeInt.h:192` says the left child page number is omitted on leaf pages.
- `sqlite/src/btree.c:1242` implements table-interior cell parsing.
- `sqlite/src/btree.c:1253` reads child-page-plus-rowid separator size.
- `sqlite/src/btree.c:1254` sets table-interior payload size to 0.

## Page layout

Each b-tree page is a byte array with structure:

```text
page header
cell pointer array
unallocated space
cell content area
```

Page 1 is special because the first 100 bytes are the database file header:

```text
page 1:
  100-byte database file header
  b-tree page header
  cell pointer array
  cell content area

other pages:
  b-tree page header
  cell pointer array
  cell content area
```

SQLite's own source comment summarizes this layout in `btreeInt.h`.

Source:

- `sqlite/src/btreeInt.h:106` describes the three page sections.
- `sqlite/src/btreeInt.h:110` shows the page layout diagram.
- `sqlite/src/btreeInt.h:126` describes page header fields.

## Cells

A cell is one variable-length entry inside a b-tree page.

For a normal table leaf page, a cell is one table row entry:

```text
cell
  payload size
  rowid
  payload bytes
  optional overflow page pointer
```

The cell payload is the encoded SQL record. For:

```sql
CREATE TABLE t(a int);
INSERT INTO t(a) VALUES (123);
```

the table b-tree stores roughly:

```text
key / rowid = 1
payload     = encoded record containing a = 123
```

Important distinction:

```text
rowid is the b-tree key
column values are in the record payload
```

So the rowid is not stored inside the SQL record payload for a normal rowid
table. It is stored in the cell header as the integer key.

Source:

- `sqlite/src/btreeInt.h:165` says cells are variable length.
- `sqlite/src/btreeInt.h:186` says varints are used for rowids.
- `sqlite/src/btreeInt.h:189` shows the cell content format.
- `sqlite/src/btreeInt.h:193` has the data-size varint.
- `sqlite/src/btreeInt.h:194` has the key/rowid varint for `intKey` pages.

## Cell pointer array

Cells are not necessarily physically stored in sorted order. Instead, the page
has a sorted array of 2-byte offsets:

```text
cell pointer[0] -> offset of first cell
cell pointer[1] -> offset of second cell
cell pointer[2] -> offset of third cell
```

The offsets point into the same raw page byte array:

```text
pPage->aData
```

Conceptually:

```text
+-------------------------+
| page header             |
+-------------------------+
| cell pointer array      |  sorted by key / rowid
|  [0] -> offset 3890     |
|  [1] -> offset 3868     |
+-------------------------+
| free space              |
+-------------------------+
| cell content area       |
|  actual cell bytes      |
+-------------------------+
```

The sorted cell pointer array lets SQLite search by key without requiring the
cell bodies themselves to be physically sorted.

For a rowid table, the pointer array is sorted by rowid because rowid is the
b-tree key. Keeping it sorted is cheaper than it first sounds because SQLite
mostly moves 2-byte offsets, not whole row bodies.

When inserting a cell into the middle of a page, SQLite:

```text
finds the correct cell index
allocates space for the cell body somewhere in the cell content area
shifts the 2-byte cell pointers with memmove()
writes the new 2-byte offset into the pointer array
```

The relevant code is:

```c
pIns = pPage->aCellIdx + i*2;
memmove(pIns+2, pIns, 2*(pPage->nCell - i));
put2byte(pIns, idx);
```

So insertion inside one page is linear in the number of cells on that page, but
the data being shifted is tiny. The b-tree keeps the number of page reads small
by finding the target leaf in logarithmic page hops, then doing this local
pointer-array edit.

Source:

- `sqlite/src/btreeInt.h:142` describes the cell pointer array.
- `sqlite/src/btreeInt.h:149` says cell content grows from the end of the page.
- `sqlite/src/btreeInt.h:165` says cells may not be contiguous or in order.
- `sqlite/src/btree.c:7481` inserts into the cell pointer array.
- `sqlite/src/btree.c:7482` shifts existing 2-byte cell pointers.

## Delete and free space inside a page

Deleting a row does not necessarily scrub the old cell bytes immediately. SQLite
removes the cell pointer and marks the cell's old byte range as reusable free
space.

The local page operation is:

```text
dropCell(page, cell_index, cell_size)
  -> find the cell's byte offset
  -> freeSpace(page, offset, size)
  -> remove the 2-byte pointer from the cell pointer array
  -> decrement nCell
```

After deleting cell B:

```text
before:
  cell pointer array: A, B, C
  cell content area:  bytes for A, bytes for B, bytes for C

after:
  cell pointer array: A, C
  cell content area:  bytes for A, freeblock where B was, bytes for C
```

If a new cell fits in that freeblock, SQLite can reuse it. If not, it may:

```text
use another freeblock
use the unallocated gap between pointer array and cell content
defragment the page to make contiguous space
split or rebalance pages if the cell still will not fit
```

Freeblocks have their own tiny structure inside the page:

```text
2 bytes: offset of next freeblock
2 bytes: size of this freeblock
```

Adjacent freeblocks are coalesced, so deleting nearby cells can create a larger
usable region.

Source:

- `sqlite/src/btreeInt.h:152` describes freeblocks.
- `sqlite/src/btreeInt.h:161` shows freeblock layout.
- `sqlite/src/btree.c:1747` searches freeblocks for a slot.
- `sqlite/src/btree.c:1819` allocates space for a new cell.
- `sqlite/src/btree.c:1882` defragments if contiguous space is not available.
- `sqlite/src/btree.c:1918` implements `freeSpace()`.
- `sqlite/src/btree.c:1910` says adjacent freeblocks are coalesced.
- `sqlite/src/btree.c:7267` implements `dropCell()`.
- `sqlite/src/btree.c:7292` returns deleted cell bytes to page free space.
- `sqlite/src/btree.c:7305` removes the 2-byte cell pointer.

## From page bytes to row values

SQLite does not parse a page into all rows at once. It decodes only what it needs.

The path is:

```text
raw page bytes
  -> page header
  -> cell pointer array
  -> one cell
  -> CellInfo
  -> record payload
  -> SQL column value
```

First, b-tree page initialization reads page metadata from `aData`:

```text
page flags
number of cells
start of cell content area
freeblock information
```

Then, when a cursor points at cell `i`, SQLite finds the cell bytes through the
cell pointer array:

```c
#define findCell(P,I) \
  ((P)->aData + ((P)->maskPage & get2byteAligned(&(P)->aCellIdx[2*(I)])))
```

That gives a pointer to one cell inside the page's byte buffer.

For a table leaf page, `btreeParseCellPtr()` decodes that cell:

```text
read varint payload size
read varint rowid
set pInfo->nKey     = rowid
set pInfo->nPayload = payload size
set pInfo->pPayload = pointer to record bytes
```

So after b-tree cell parsing:

```text
CellInfo.nKey     = rowid
CellInfo.pPayload = encoded SQL record
```

When VDBE needs the rowid, it calls `sqlite3BtreeIntegerKey()`, which returns
`pCur->info.nKey`.

When VDBE needs a column, `OP_Column` fetches the payload and decodes SQLite's
record format:

```text
record header:
  header size
  serial type for column 0
  serial type for column 1
  ...

record body:
  encoded value for column 0
  encoded value for column 1
  ...
```

The key distinction:

```text
rowid        -> b-tree key, stored in the cell header
column data  -> SQL record payload, decoded by OP_Column
```

Source:

- `sqlite/src/btree.c:1166` defines `findCell`.
- `sqlite/src/btree.c:1259` implements `btreeParseCellPtr()`.
- `sqlite/src/btree.c:1330` stores the rowid in `CellInfo.nKey`.
- `sqlite/src/btree.c:1332` stores the payload pointer in `CellInfo.pPayload`.
- `sqlite/src/btree.c:4921` implements `sqlite3BtreeIntegerKey()`.
- `sqlite/src/vdbe.c:2975` implements `OP_Column`.
- `sqlite/src/vdbe.c:3042` asks b-tree for payload size.
- `sqlite/src/vdbe.c:3043` fetches the payload pointer.
- `sqlite/src/vdbe.c:3197` decodes one column with `sqlite3VdbeSerialGet()`.

## Overflow pages

If a row payload is too large to fit comfortably inside one b-tree page, SQLite
stores part of the payload locally in the cell and spills the rest to overflow
pages.

The cell then stores:

```text
local payload bytes
first overflow page number
```

Overflow pages form a linked list:

```text
overflow page
  next overflow page number
  payload bytes
```

Again, the links are page numbers, not memory pointers.

Source:

- `sqlite/src/btreeInt.h:196` mentions the first overflow page pointer.
- `sqlite/src/btreeInt.h:198` describes overflow pages as a linked list.
- `sqlite/src/btreeInt.h:202` shows overflow page layout.

## Disk reads

The number of reads still matters, but there are layers:

```text
SQLite page cache hit -> no OS read
OS file cache hit     -> no physical device read
cache miss            -> storage device read
```

SQLite thinks in logical page reads:

```text
read page 42
```

The OS turns that into filesystem/cache/device work.

For a b-tree lookup, SQLite may read:

```text
root page
interior page
leaf page
```

If those pages are already cached, this might cause zero physical disk reads. If
they are cold, each logical page might require I/O.

## Tiny example

For:

```sql
CREATE TABLE t(a int);
INSERT INTO t(a) VALUES (123);
```

SQLite's storage model is roughly:

```text
database file
  page 1
    sqlite_schema b-tree root
    row saying: table t has root page N

  page N
    table t b-tree root/leaf
    cell:
      rowid = 1
      payload = record(a = 123)
```

The persistent identity of the table is the root page number recorded in
`sqlite_schema`. The in-memory pointers only exist while pages are cached.

## Short glossary

```text
Pager
  Layer that reads/writes fixed-size pages and handles journaling/cache rules.

Page
  Fixed-size chunk of the database file.

Pgno
  Logical page number in the database file.

DbPage / PgHdr
  Pager/page-cache object for one page. Holds pData, the raw bytes.

MemPage
  B-tree layer's interpreted wrapper around one page.

aData / pData
  Pointer to raw page bytes in memory.

B-tree
  Tree of database pages used to store tables and indexes.

Cell
  Variable-length entry inside a b-tree page.

Rowid
  Integer key for a normal table b-tree cell.

Payload
  Encoded SQL record stored in the cell.

Overflow page
  Extra page used when a cell payload is too large.
```
