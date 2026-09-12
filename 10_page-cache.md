# Page-cache relationships and dirty lists

## `Pager`, `PCache`, and `PgHdr`

A `Pager` manages one database image: its file access, transaction state,
journal or WAL, logical database size, and page cache.

A `PgHdr` describes one database page currently held in that cache:

```text
Pager
  -> PCache
       -> PgHdr for page 1
       -> PgHdr for page 2
       -> PgHdr for page 3
       -> ...
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

A clearer mental renaming is:

```text
PCache.pDirty     dirtyLruHead
PgHdr.pDirtyNext dirtyLruNext
PgHdr.pDirtyPrev dirtyLruPrevious
PgHdr.pDirty     temporarySortedNext
```
