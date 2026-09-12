# Page-cache relationships and dirty lists

## `Pager`, `PCache`, and `PgHdr`

A `Pager` manages one database image: its file access, transaction state,
journal or WAL, logical database size, and page cache.

A `PgHdr` describes one database page currently held in that cache. The
high-level relationship is:

```text
Pager
  -> PCache
       -> pCache: lower cache containing all cached pages
       -> pDirty: linked subset of dirty PgHdr objects
```

The page points back to both owners:

```c
struct PgHdr {
  void *pData;       /* Raw bytes of this cached page */
  PCache *pCache;    /* Cache containing the page */
  Pager *pPager;     /* Pager managing the database */
  Pgno pgno;         /* Database page number */
  u16 flags;         /* CLEAN, DIRTY, WRITEABLE, etc. */
};
```

At the b-tree layer, a `MemPage` interprets the same bytes as a b-tree page:

```text
MemPage -> PgHdr/DbPage -> Pager
             |
             -> pData: raw database-page bytes
```

The compact distinction is:

```text
Pager   manager for the database image
PCache  collection of cached pages
PgHdr   cache metadata for one page
MemPage b-tree interpretation of one page
```

## Where all cached pages and their bytes live

`PCache.pDirty` is not the storage for the whole cache. The opaque
`PCache.pCache` handle leads to the lower cache implementation, which performs
lookup and owns every cached page, clean or dirty:

```text
PCache
  |-- pCache -> lower cache
  |              |-- page 1, clean
  |              |-- page 2, dirty
  |              |-- page 5, clean
  |              `-- page 7, dirty
  |
  `-- pDirty -> page 2 <-> page 7
```

A dirty page therefore remains in the lower cache and is additionally linked
into `PCache.pDirty` for commit, rollback, and spilling.

At the lower interface, one cached page has this public shape:

```c
struct sqlite3_pcache_page {
  void *pBuf;    /* Actual page-byte buffer */
  void *pExtra;  /* Storage requested by SQLite core */
};
```

`pcache.c` places a `PgHdr` in `pExtra` and wires its data pointer to the lower
buffer:

```c
pPgHdr = (PgHdr *)pPage->pExtra;
pPgHdr->pPage = pPage;
pPgHdr->pData = pPage->pBuf;
```

Thus the page bytes are accessed through `PgHdr.pData`, but that pointer aliases
the buffer allocated by the lower cache:

```text
sqlite3_pcache_page.pBuf <-----+
                               |
PgHdr.pData -------------------+
```

For an ordinary PCache-backed page, this is an identity:

```c
pPgHdr->pData == pPgHdr->pPage->pBuf
```

They are two names for one buffer, not two buffers with a copy between them.
The lower cache calls it `pBuf`; pager and b-tree code reach it through
`PgHdr.pData`, and `MemPage.aData` later points at those same bytes for b-tree
interpretation. `BtShared` is input context used to reach the pager; it is not
where the fetched page content is stored.

`readDbPage(pPg)` reads database or WAL bytes directly into this shared buffer.
The exception is a memory-mapped page: it has no lower-cache page object
(`pPg->pPage==0`), carries `PGHDR_MMAP`, and `pPg->pData` points directly at the
mapped file memory.

## `sqlite3_pcache_methods2` and the `pcache1` implementation

`sqlite3_pcache_methods2` is a C vtable: a struct of function pointers defining
the lower-cache interface. `pcache1.c` is SQLite's built-in implementation:

```text
interface function       default implementation
xCreate                  pcache1Create
xFetch                   pcache1Fetch
xUnpin                   pcache1Unpin
xRekey                   pcache1Rekey
xTruncate                pcache1Truncate
xDestroy                 pcache1Destroy
```

`sqlite3PCacheSetDefault()` installs these pointers into
`sqlite3GlobalConfig.pcache2`. Calls from `pcache.c` then dispatch indirectly:

```c
sqlite3GlobalConfig.pcache2.xFetch(pCache->pCache, pgno, eCreate);
```

There is intentionally no definition of `struct sqlite3_pcache`. It is an
incomplete type used only as an opaque handle. The built-in backend allocates
its private concrete type and casts it to that handle:

```c
struct PCache1 { /* private hash table, limits, LRU state, ... */ };

PCache1 *pCache = sqlite3MallocZero(sizeof(PCache1));
return (sqlite3_pcache *)pCache;
```

Each `pcache1` callback casts the handle back before accessing fields:

```c
PCache1 *pCache = (PCache1 *)p;
```

An application may install another `sqlite3_pcache_methods2` table, but normal
SQLite initialization selects `pcache1`. This is composition through an opaque
pointer and function table, not C struct inheritance.

## Pager getter dispatch with a function pointer

SQLite is C, so `Pager` does not have C++-style methods. It does have an `xGet`
field that stores a function pointer:

```c
int (*xGet)(Pager *, Pgno, DbPage **, int);
```

`sqlite3PagerGet()` is a small dispatch wrapper:

```c
return pPager->xGet(pPager, pgno, ppPage, flags);
```

The call is equivalent to `(*pPager->xGet)(...)`. During `sqlite3PagerOpen()`,
`setGetterMethod()` assigns a valid implementation before returning the new
Pager:

```text
normal access   -> getPageNormal
memory mapping  -> getPageMMap
pager error     -> getPageError
```

SQLite may call `setGetterMethod()` again when those conditions change. This is
C-style dynamic dispatch: the function receives `pPager` explicitly instead of
an implicit C++ `this` pointer.

The `#if 0` branch inside `sqlite3PagerGet()` is compile-time-disabled tracing.
Changing it to `#if 1` compiles extra page-number and error logging; normal
builds contain only the direct `xGet` call.

## What `PgHdr.nRef` counts

`nRef` counts active lifetime claims that keep a cached page pinned. It does
not count C pointer variables:

```c
PgHdr *p = fetchedPage;  /* one acquired reference */
PgHdr *q = p;            /* pointer copy; nRef does not change */
```

SQLite changes the count only through its reference APIs:

```text
PagerGet / PagerRef       nRef++
PagerUnref / releasePage  nRef--
```

Repeated gets from the same Pager do not normally copy the page bytes:

```text
first PagerGet(page 5)
  -> cache miss: allocate/fill one cache entry, nRef becomes 1

second PagerGet(page 5)
  -> cache hit: return the same PgHdr and pData, nRef becomes 2
```

A later get may allocate and refill a new buffer only after the old cache entry
has been evicted. Separate Pager/cache instances may also hold separate copies.

The page records only the count, not who owns each reference. That is why
`releasePage(pPage)` needs no cursor or caller identifier. Higher-level code is
responsible for balancing each acquired reference with one release.

```text
nRef == 0  no active holder keeps the page pinned
nRef == 1  one active reference
nRef > 1   multiple active references
```

For `btreeGetUnusedPage()`, the fetch itself creates one reference. A result of
`nRef > 1` means something else was already holding a page that the freelist
claims is unused, so SQLite reports corruption.

## What `sqlite3PcacheMakeDirty()` does

`pager_write()` calls `sqlite3PcacheMakeDirty()` before page data is changed:

```text
sqlite3PagerWrite()
  -> pager_write()
     -> sqlite3PcacheMakeDirty()
```

`sqlite3PcacheMakeDirty()` performs two matching updates:

```text
1. replace PGHDR_CLEAN with PGHDR_DIRTY on the page
2. add the page to PCache's dirty-page list
```

The flag answers whether one particular page is dirty. The list lets the
pager find all pages that need commit, rollback, or cache-spill processing.
An already-dirty page is not added a second time.

`sqlite3PcacheMakeClean()` performs the reverse operation: it removes the
page from the dirty list, clears dirty/writeable/sync flags, and sets
`PGHDR_CLEAN`.

## `pcacheManageDirtyList()`

This private helper changes only the in-memory dirty-list links. Its second
argument is a bitmask:

```c
PCACHE_DIRTYLIST_REMOVE = 1;  /* binary 01 */
PCACHE_DIRTYLIST_ADD    = 2;  /* binary 10 */
PCACHE_DIRTYLIST_FRONT  = 3;  /* binary 11: remove, then add */
```

The main dirty list is doubly linked and kept in LRU order:

```text
PCache.pDirty
    |
    v
 newest page <-> ... <-> oldest page
                            ^
                    PCache.pDirtyTail
```

It is linked through each page's `pDirtyNext` and `pDirtyPrev` fields.
`FRONT` removes a page from its current position and adds it at the head.

### How the LRU order is maintained

There is no separate call that later sorts this list into LRU order. Assuming
the callers satisfy each operation's preconditions, every call leaves the list
in LRU order:

```text
ADD     insert a newly dirty page at the front; it is the newest
REMOVE  unlink one page; the relative order of the others is unchanged
FRONT   remove an existing dirty page and add it back as the newest
```

The caller identifies the event; `pcacheManageDirtyList()` performs the
corresponding pointer changes. For example, when the last user of a dirty page
releases it, its reference count reaches zero and the caller requests `FRONT`:

```text
sqlite3PcacheRelease(page)
  -> nRef becomes 0
     -> pcacheManageDirtyList(page, PCACHE_DIRTYLIST_FRONT)
```

For example:

```text
before using page 1:
  page 7 <-> page 3 <-> page 1
  newest                    oldest

after using and releasing page 1:
  page 1 <-> page 7 <-> page 3
  newest                    oldest
```

This is an approximate LRU based on when active use finishes. SQLite does not
reorder the list for every byte access. Because the list is doubly linked,
each operation updates only a few known pointers and takes constant time; no
LRU sort or full-list scan is required.

## The two different `pDirty` names

`PCache.pDirty` and `PgHdr.pDirty` do not form one directly linked chain.

`PCache.pDirty` is the head of the persistent LRU dirty list, which continues
through `PgHdr.pDirtyNext`:

```text
PCache.pDirty -> page 9 -> page 4 -> page 2
                 using each page's pDirtyNext
```

`PgHdr.pDirty` is a separate temporary link. `sqlite3PcacheDirtyList()` uses
it to return the same dirty pages sorted by page number for pager/WAL work:

```text
returned list -> page 2 -> page 4 -> page 9
                 using each page's pDirty
```

The same `PgHdr` objects can therefore participate in both lists:

```text
pDirtyNext/pDirtyPrev  persistent cache LRU list
PgHdr.pDirty           temporary page-number-sorted list
```

The two orders are independent because they use separate link fields. Changing
the temporary `PgHdr.pDirty` chain does not rearrange `pDirtyNext` or
`pDirtyPrev`, and changing the LRU chain does not determine page-number order:

```text
LRU order:          page 9 <-> page 2 <-> page 7
page-number order:  page 2  -> page 7  -> page 9
```

### When and why page-number sorting happens

For a file-backed autocommit `CREATE TABLE`, the temporary sorted list is
normally built when `OP_Halt` commits the completed statement:

```text
OP_Halt
  -> sqlite3VdbeHalt()
     -> vdbeCommit()
        -> sqlite3BtreeCommitPhaseOne()
           -> sqlite3PagerCommitPhaseOne()
              -> sqlite3PcacheDirtyList()
                 -> pcacheSortDirtyList()
```

The resulting ascending page numbers correspond to ascending file offsets:

```text
page 1 -> offset 0
page 2 -> offset pageSize
page 7 -> offset 6 * pageSize
```

This turns an LRU order such as `7, 1, 4, 2` into the write order
`1, 2, 4, 7`. Sequential file offsets improve write locality and reduce
seeking. In WAL mode the pager likewise passes a page-number-sorted list to
the WAL-frame writer.

The two orders therefore solve different problems:

```text
LRU order          choose a dirty page to spill under cache pressure
page-number order  flush a batch of dirty pages efficiently
```

`EXPLAIN CREATE TABLE ...` only displays bytecode and does not execute the
schema change. A pure `:memory:` database also normally has no disk-write
reason to build this sorted commit list.

A clearer mental renaming is:

```text
PCache.pDirty     dirtyLruHead
PgHdr.pDirtyNext dirtyLruNext
PgHdr.pDirtyPrev dirtyLruPrevious
PgHdr.pDirty     temporarySortedNext
```
