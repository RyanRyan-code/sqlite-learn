# Pointer-map notes

## What a pointer-map page stores

Pointer-map pages exist for autovacuum. They let SQLite look backwards:

```text
normal page pointer:  parent -> child
pointer-map entry:    child  -> type + parent
```

A pointer-map page is a fixed array, not a b-tree or a freelist trunk/leaf
structure. Each entry is 5 bytes:

```text
byte 0      page type
bytes 1..4  parent/backlink page number, big-endian
```

The target page number is not stored in the entry. SQLite uses the target page
number, called `key`, to calculate the pointer-map page and entry offset.

```text
pointer-map page 2

offset 0     entry for page 3
offset 5     entry for page 4
offset 10    entry for page 5
...
```

Because each target page has a fixed slot, SQLite does not insert or delete
entries. If a page is freed or reused, it overwrites that slot with the new type
and parent/backlink.

## Parent page

The parent is the page containing the pointer to the target page. For example:

```text
page 3 contains a child pointer to page 7

entry for page 7 = { PTRMAP_BTREE, parent 3 }
```

The field's exact meaning depends on the type:

```text
PTRMAP_BTREE      parent b-tree page
PTRMAP_OVERFLOW1  b-tree page containing the owning cell
PTRMAP_OVERFLOW2  previous overflow page
PTRMAP_ROOTPAGE   no parent
PTRMAP_FREEPAGE   no parent
```

The pointer map does not need trunk and leaf pages. It updates fixed entries.
The freelist separately tracks which pages are available for allocation.

## Reading an entry

The important part of `ptrmapGet()` is:

```c
iPtrmap = PTRMAP_PAGENO(pBt, key);
offset = PTRMAP_PTROFFSET(iPtrmap, key);

*pEType = pPtrmap[offset];
if( pPgno ) *pPgno = get4byte(&pPtrmap[offset+1]);
```

`key` identifies the page being looked up. The entry position identifies that
page, while the five bytes store its type and backlink.

`pEType` is required. `pPgno` is an optional output: passing `0` means the
caller only wants the type. The four parent bytes still exist in the entry.

## `0` and null pointers

In C, `0` used as a pointer is a null pointer. SQLite commonly uses `0` where
other code uses `NULL`:

```c
assert( pEType!=0 );
assert( pEType!=NULL );  /* equivalent */
```

This checks the pointer `pEType`, not the value `*pEType`.

Similarly:

```c
ptrmapGet(pBt, nearby, &eType, 0);
```

The final `0` means `pPgno` is null, so the caller does not want the parent-page
output.

## Array indexing and pointers

In C, array indexing is pointer arithmetic:

```c
pPtrmap[0]         == *pPtrmap
pPtrmap[n]         == *(pPtrmap + n)
&pPtrmap[offset+1] == pPtrmap + offset + 1
```

Therefore `get4byte(&pPtrmap[offset+1])` receives the address of the first
parent-number byte and reads bytes `offset+1` through `offset+4`.

## Why the entry must fit

An entry begins at `offset` and occupies five bytes, so its last valid starting
position is `usableSize-5`:

```c
assert( offset <= (int)pBt->usableSize-5 );
```

Together with the earlier `offset<0` check, the intended range is:

```text
0 <= offset <= usableSize - 5
```

## What `sqlite3PagerUnref(pDbPage)` releases

In `ptrmapGet()`, `pDbPage` is the in-memory handle for the pointer-map page. It
is not the target page `key` and not the parent page returned through `pPgno`.

`sqlite3PagerGet()` acquires a reference to a page's in-memory image.
`sqlite3PagerUnref()` balances that operation:

```text
sqlite3PagerGet()    "I am using this cached page"
sqlite3PagerUnref()  "I am finished with this cached page"
```

This reference count is different from the pointer-map parent relationship:

```text
pointer-map parent  on-disk relationship between database pages
pager reference     number of active users of an in-memory page image
```

When the in-memory reference count reaches zero:

```text
clean page  may remain cached or become recyclable
dirty page  must be retained until it is safely handled
```

`sqlite3PagerUnref()` does not itself write the page to disk. It updates the
in-memory ownership state so the pager can safely coordinate the page cache
with the database file, rollback journal, or WAL.

For `ptrmapGet()`, the compact flow is:

```text
load the pointer-map page
  -> read the entry
  -> copy the requested outputs
  -> release the in-memory page reference
```
