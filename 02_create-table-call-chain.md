# CREATE TABLE call chain: VDBE execution

This note follows a tiny statement through the runtime half of SQLite:

```sql
CREATE TABLE t(a int);
```

The parser/build phase has already produced a VDBE program. Now the question is:
when `sqlite3_step()` runs that program, which opcodes execute, and where do they
touch storage?

## The useful mental model

`CREATE TABLE` does not immediately store user rows. It does two storage-facing
things:

1. Allocates an empty b-tree root page for the new table.
2. Inserts/updates a row in `sqlite_schema` describing that table.

So the important runtime path is:

```text
sqlite3_step()
  -> sqlite3Step()
    -> sqlite3VdbeExec()
      -> OP_Transaction
      -> OP_CreateBtree
      -> OP_OpenWrite / OP_NewRowid / OP_Insert
      -> OP_MakeRecord / OP_Insert
      -> OP_SetCookie / OP_ParseSchema
      -> OP_Halt
        -> sqlite3VdbeHalt()
          -> vdbeCommit()
            -> sqlite3BtreeCommitPhaseOne()
            -> sqlite3BtreeCommitPhaseTwo()
```

## How prepare() adds this opcode

Before `sqlite3_step()` can run anything, `sqlite3_prepare_v2()` compiles the SQL
text into the VDBE program shown below:

```text
sqlite3_prepare_v2()
  -> sqlite3LockAndPrepare()
    -> sqlite3Prepare()
      -> sqlite3RunParser()
        -> parse.y CREATE TABLE actions
          -> sqlite3StartTable()
             emits OP_CreateBtree
          -> sqlite3EndTable()
             emits the final sqlite_schema update
```

The `CREATE TABLE` grammar action in `parse.y` calls `sqlite3StartTable()` when
it has seen:

```sql
CREATE TABLE t
```

Then the closing `)` action calls `sqlite3EndTable()` after the column list is
known.

The actual `OP_CreateBtree` instruction is appended in `sqlite3StartTable()`:

```c
pParse->u1.cr.addrCrTab =
   sqlite3VdbeAddOp3(v, OP_CreateBtree, iDb, reg2, BTREE_INTKEY);
```

That one call chooses the operands:

- `P1 = iDb`: which attached database gets the new b-tree.
- `P2 = reg2`: which VM register will receive the new root page number.
- `P3 = BTREE_INTKEY`: create a rowid-table b-tree.

`sqlite3VdbeAddOp3()` copies those values into the next `VdbeOp` slot. They are
not runtime scratch values; they are part of the prepared statement's bytecode.
Later compile-time helpers may patch them before execution. For example,
`convertToWithoutRowidTable()` changes this opcode's `P3` from `BTREE_INTKEY` to
`BTREE_BLOBKEY` for a `WITHOUT ROWID` table.

`sqlite3EndTable()` then uses the saved registers from `sqlite3StartTable()`:

- `pParse->u1.cr.regRoot`: the register where `OP_CreateBtree` will put the
  root page.
- `pParse->u1.cr.regRowid`: the rowid of the placeholder `sqlite_schema` row.

It emits an internal schema update with `sqlite3NestedParse()` so the final
`sqlite_schema` record gets the real table name, root page, and SQL text.

Source:

- `sqlite/src/prepare.c:937` implements `sqlite3_prepare_v2()`.
- `sqlite/src/prepare.c:836` implements `sqlite3LockAndPrepare()`.
- `sqlite/src/prepare.c:779` calls `sqlite3RunParser()`.
- `sqlite/src/parse.y:209` calls `sqlite3StartTable()` for `CREATE TABLE`.
- `sqlite/src/parse.y:224` calls `sqlite3EndTable()` after the column list.
- `sqlite/src/build.c:1206` implements `sqlite3StartTable()`.
- `sqlite/src/build.c:1374` emits `OP_CreateBtree`.
- `sqlite/src/vdbeaux.c:272` implements `sqlite3VdbeAddOp3()`.
- `sqlite/src/vdbeaux.c:288` stores `p1`, `p2`, and `p3` into the opcode.
- `sqlite/src/build.c:2376` patches `P3` for `WITHOUT ROWID`.
- `sqlite/src/build.c:2637` implements `sqlite3EndTable()`.
- `sqlite/src/build.c:2909` emits the final schema-table update through
  `sqlite3NestedParse()`.

## Where r[1], r[2], r[3] live

The `P1`, `P2`, and `P3` values are fields in a `VdbeOp`. They are instruction
operands. The `r[1]`, `r[2]`, and `r[3]` values are different: they are runtime
registers in the VM's `Mem *aMem` array.

At execution time, `sqlite3VdbeExec()` keeps a local copy:

```c
Mem *aMem = p->aMem;
```

So an opcode operand that names a register is just an integer index into
`aMem[]`. For example, `CreateBtree 0 2 1` has `P2 = 2`, so `out2Prerelease()`
returns `&p->aMem[pOp->p2]`, meaning register `r[2]`.

For the early `CREATE TABLE` bytecode:

```text
CreateBtree 0 2 1   -> writes the new root page into r[2]
NewRowid    0 1 0   -> writes the sqlite_schema rowid into r[1]
Blob        6 3 0   -> writes the placeholder blob into r[3]
Insert      0 3 1   -> reads record data from r[3], rowid key from r[1]
```

So:

```text
P2 = 2       fixed bytecode operand
r[2] = pgno  runtime value stored in p->aMem[2]
```

The register numbers are picked during prepare/code generation. `sqlite3StartTable()`
stores the important ones in parse state:

```text
pParse->u1.cr.regRoot   -> register that will receive the table root page
pParse->u1.cr.regRowid  -> register that will receive the schema rowid
```

Later code generation emits more opcodes that refer to those same register
numbers. The values themselves are produced only when `sqlite3_step()` runs the
VM.

Source:

- `sqlite/src/vdbeInt.h:474` stores the VM register array as `Vdbe.aMem`.
- `sqlite/src/vdbe.c:865` copies `p->aMem` into local `aMem` in
  `sqlite3VdbeExec()`.
- `sqlite/src/vdbe.c:672` implements `out2Prerelease()`.
- `sqlite/src/vdbe.c:676` resolves an output register with `&p->aMem[pOp->p2]`.
- `sqlite/src/vdbe.c:5601` uses `out2Prerelease()` for `OP_NewRowid`.
- `sqlite/src/vdbe.c:5757` reads `OP_Insert` data from `aMem[pOp->p2]`.
- `sqlite/src/vdbe.c:5770` reads `OP_Insert` rowid/key from `aMem[pOp->p3]`.

### Why numbered registers are not as risky as they look

At first glance, `r[1]`, `r[2]`, and `r[3]` look like a fragile way to pass
values around. It feels like normal C code manually sharing global variables:
one opcode writes `r[2]`, another opcode later reads `r[2]`, and a mistake could
overwrite something useful.

The important distinction is that these registers are not global variables and
not hardware CPU registers. They are per-statement VM slots:

```text
Vdbe *p
  -> p->aMem[1]  == r[1]
  -> p->aMem[2]  == r[2]
  -> p->aMem[3]  == r[3]
```

Each prepared statement has its own `Vdbe` object and its own `aMem` array. So
`r[2]` in one running statement is different storage from `r[2]` in another
running statement.

The register numbers are also not chosen casually during execution. They are
assigned by SQLite's code generator during prepare. The code generator is
responsible for knowing when a register value is still live and when the slot
can be reused. In that sense, `r[2]` is closer to a compiler temporary than to a
source-level variable.

A friendlier high-level form might be:

```text
rootPage = CreateBtree(db=main, flags=BTREE_INTKEY)
OpenWrite(cursor=1, root=rootPage)
```

But an efficient interpreter normally lowers that kind of symbolic value into a
small numbered slot anyway. SQLite keeps the lowered form visible because the
VDBE is a compact bytecode interpreter written in portable C.

So the safety contract is:

- `P1`, `P2`, and `P3` are fixed integer operands in the bytecode instruction.
- `r[N]` is a runtime `Mem` slot inside the current `Vdbe`.
- The SQL compiler/code generator owns register allocation and reuse.
- If bytecode overwrites a live register, that is a code-generation bug, not
  normal VM behavior.

When `CreateBtree 0 2 1` runs, the VM does not pass a named `rootPage` variable.
It resolves operand `P2 = 2` into `&p->aMem[2]`, then stores the new root page
number there. Later opcodes use the same register number because the code
generator deliberately emitted them that way.

## The bytecode spine

From:

```sh
sqlite-debug/sqlite3 :memory: "EXPLAIN CREATE TABLE t(a int);"
```

The core program looks like this:

```text
addr  opcode       p1  p2  p3  p4                          p5
0     Init         0   30  0                               0
1     ReadCookie   0   3   2                               0
2     If           3   5   0                               0
3     SetCookie    0   2   4                               0
4     SetCookie    0   5   1                               0
5     CreateBtree  0   2   1                               0
6     OpenWrite    0   1   0   5                           0
7     NewRowid     0   1   0                               0
8     Blob         6   3   0   <six-byte null record>       0
9     Insert       0   3   1                               8
10    Close        0   0   0                               0
11    Close        0   0   0                               0
12    Null         0   4   5                               0
14    OpenWrite    1   1   0   5                           0
17    SeekRowid    1   19  1                               0
20    String8      0   6   0   table                       0
21    String8      0   7   0   t                           0
22    String8      0   8   0   t                           0
23    Copy         2   9   0                               0
24    String8      0   10  0   CREATE TABLE t(a int)        0
25    MakeRecord   6   5   4   BBBDB                       0
26    Insert       1   4   5                               0
27    SetCookie    0   1   1                               0
28    ParseSchema  0   0   0   tbl_name='t' AND type!='trigger'
29    Halt         0   0   0                               0
30    Transaction  0   1   0   0                           1
31    Goto         0   1   0                               0
```

The slightly surprising bit: execution starts at `Init`, jumps to
`Transaction`, then `Goto` jumps back to opcode 1. SQLite puts transaction setup
at the end and jumps there first.

## Step entry

The public API is `sqlite3_step()` in `sqlite/src/vdbeapi.c`.

```text
sqlite3_step()
  -> sqlite3Step()
    -> sqlite3VdbeExec()
```

`sqlite3Step()` prepares the VM for execution:

- sets `p->pc = 0`
- changes the VDBE state to `VDBE_RUN_STATE`
- increments active/read/write VM counters
- calls `sqlite3VdbeExec(p)`

Source:

- `sqlite/src/vdbeapi.c:777` has `sqlite3Step()`.
- `sqlite/src/vdbeapi.c:869` calls `sqlite3VdbeExec(p)`.
- `sqlite/src/vdbe.c:846` is the VDBE interpreter entry point.

## OP_Transaction

`OP_Transaction` starts the b-tree transaction before any page mutation.

For this statement:

```text
Transaction 0 1 0 0 1
```

Meaning:

- `P1 = 0`: operate on the main database.
- `P2 = 1`: start a write transaction.
- `P5 = 1`: check schema version/generation while starting.

At runtime, `OP_Transaction` calls:

```text
sqlite3BtreeBeginTrans(pBt, pOp->p2, &iMeta)
```

If the write lock cannot be acquired, the opcode returns `SQLITE_BUSY` and saves
the program counter so execution can retry later.

Source:

- `sqlite/src/vdbe.c:4102` implements `OP_Transaction`.
- `sqlite/src/vdbe.c:4128` calls `sqlite3BtreeBeginTrans()`.
- `sqlite/src/vdbe.c:4164` checks schema metadata when `P5` is set.

## OP_ReadCookie / OP_SetCookie

SQLite database metadata lives in page 1. VDBE refers to pieces of that metadata
as cookies.

Early bytecode:

```text
ReadCookie 0 3 2
If         3 5 0
SetCookie 0 2 4
SetCookie 0 5 1
```

This initializes file format and text encoding if needed. Later:

```text
SetCookie 0 1 1
```

bumps the schema version cookie, telling other prepared statements that the
schema changed.

Runtime calls:

```text
OP_ReadCookie -> sqlite3BtreeGetMeta()
OP_SetCookie  -> sqlite3BtreeUpdateMeta()
```

Source:

- `sqlite/src/vdbe.c:4215` implements `OP_ReadCookie`.
- `sqlite/src/vdbe.c:4228` calls `sqlite3BtreeGetMeta()`.
- `sqlite/src/vdbe.c:4249` implements `OP_SetCookie`.
- `sqlite/src/vdbe.c:4261` calls `sqlite3BtreeUpdateMeta()`.

## OP_CreateBtree

This is the first opcode that directly creates table storage:

```text
CreateBtree 0 2 1
```

Meaning:

- `P1 = 0`: main database.
- `P2 = 2`: store the new root page number in register 2.
- `P3 = 1`: `BTREE_INTKEY`, meaning a rowid table b-tree.

Runtime path:

```text
OP_CreateBtree
  -> sqlite3BtreeCreateTable()
    -> btreeCreateTable()
      -> allocateBtreePage()
      -> zeroPage(... PTF_INTKEY | PTF_LEAFDATA | PTF_LEAF)
```

The new page is not filled with rows. It is initialized as an empty leaf table
b-tree page. The resulting root page number goes into `r[2]`.

Source:

- `sqlite/src/vdbe.c:7032` implements `OP_CreateBtree`.
- `sqlite/src/vdbe.c:7045` calls `sqlite3BtreeCreateTable()`.
- `sqlite/src/btree.c:10199` wraps `btreeCreateTable()`.
- `sqlite/src/btree.c:10183` allocates the root page in the non-autovacuum case.
- `sqlite/src/btree.c:10188` chooses table-page flags for `BTREE_INTKEY`.
- `sqlite/src/btree.c:10193` calls `zeroPage()`.
- `sqlite/src/btree.c:6514` implements `allocateBtreePage()`.

## allocateBtreePage()

`allocateBtreePage()` decides where the new page comes from.

There are two broad cases:

1. Reuse a page from the freelist.
2. Append a new page to the database image.

In the append case it:

```text
sqlite3PagerWrite(page 1)
pBt->nPage++
write new page count into page 1 header
btreeGetUnusedPage(new-page-number)
sqlite3PagerWrite(new page)
```

That `sqlite3PagerWrite()` call matters. It makes the page writable and ensures
rollback/journaling rules are satisfied before b-tree code changes the bytes.

Source:

- `sqlite/src/btree.c:6535` reads the freelist count from page 1.
- `sqlite/src/btree.c:6570` marks page 1 writable before freelist edits.
- `sqlite/src/btree.c:6755` starts the append-new-page path.
- `sqlite/src/btree.c:6776` marks page 1 writable before increasing page count.
- `sqlite/src/btree.c:6806` marks the newly allocated page writable.
- `sqlite/src/pager.c:6280` implements `sqlite3PagerWrite()`.

## First sqlite_schema insert: placeholder row

The parser emitted an early schema-table placeholder before it knew the complete
final `CREATE TABLE ...` SQL text:

```text
OpenWrite 0 1 0 5
NewRowid  0 1 0
Blob      6 3 0 <six-byte null record>
Insert    0 3 1
Close     0
```

This opens cursor 0 on root page 1, which is `sqlite_schema`, generates a rowid
into `r[1]`, creates a tiny placeholder record in `r[3]`, and inserts it.

The generated rowid is important: later bytecode reopens `sqlite_schema`, seeks
back to that same rowid, and overwrites the placeholder with the real schema
record.

Source:

- `sqlite/src/vdbe.c:4331` documents `OP_OpenWrite`.
- `sqlite/src/vdbe.c:4387` handles `OP_OpenWrite`.
- `sqlite/src/vdbe.c:4453` opens the b-tree cursor with `sqlite3BtreeCursor()`.
- `sqlite/src/vdbe.c:5589` implements `OP_NewRowid`.
- `sqlite/src/vdbe.c:5748` implements `OP_Insert`.
- `sqlite/src/vdbe.c:5816` calls `sqlite3BtreeInsert()`.

## OP_Insert into a table b-tree

`OP_Insert` prepares a `BtreePayload`:

```text
x.nKey  = rowid from register P3
x.pData = record blob from register P2
x.nData = record size
```

Then it calls:

```text
sqlite3BtreeInsert(pC->uc.pCursor, &x, flags, seekResult)
```

For a rowid table b-tree, `sqlite3BtreeInsert()`:

- moves/seeks the cursor to the target integer key if needed
- encodes the new b-tree cell with `fillInCell()`
- calls `sqlite3PagerWrite()` on the destination page
- inserts or overwrites the cell
- may split/balance pages if the target page cannot fit the new cell

Source:

- `sqlite/src/vdbe.c:5774` copies the integer key into `x.nKey`.
- `sqlite/src/vdbe.c:5806` points the payload at the record bytes.
- `sqlite/src/vdbe.c:5816` calls `sqlite3BtreeInsert()`.
- `sqlite/src/btree.c:9409` implements `sqlite3BtreeInsert()`.
- `sqlite/src/btree.c:9513` seeks rowid-table inserts with `sqlite3BtreeTableMoveto()`.
- `sqlite/src/btree.c:9601` encodes the cell with `fillInCell()`.
- `sqlite/src/btree.c:9614` marks the page writable before overwrite.

## Final sqlite_schema overwrite

After the placeholder, SQLite builds the final schema record:

```text
OpenWrite  1 1 0 5
SeekRowid  1 19 1
String8    table      -> r[6]
String8    t          -> r[7]
String8    t          -> r[8]
Copy       2 9        -> r[9] = root page from OP_CreateBtree
String8    CREATE TABLE t(a int) -> r[10]
MakeRecord 6 5 4 BBBDB
Insert     1 4 5
```

This constructs the five columns of a `sqlite_schema` row:

```text
type     = 'table'
name     = 't'
tbl_name = 't'
rootpage = r[2]
sql      = 'CREATE TABLE t(a int)'
```

`OP_MakeRecord` packs those register values into SQLite's on-disk record format:

```text
| header-size | serial-type... | payload... |
```

Then the second `OP_Insert` writes that packed record back into `sqlite_schema`,
overwriting the placeholder row.

Source:

- `sqlite/src/vdbe.c:3469` implements `OP_MakeRecord`.
- `sqlite/src/vdbe.c:3485` documents the record layout.
- `sqlite/src/vdbe.c:5748` implements the final `OP_Insert`.

## OP_ParseSchema

After changing the schema table, SQLite reloads the new object into the
connection's in-memory schema:

```text
ParseSchema 0 0 0 "tbl_name='t' AND type!='trigger'"
```

This opcode runs a query against `sqlite_schema`, feeds matching rows through
`sqlite3InitCallback()`, and reparses the saved SQL text while `db->init.busy`
is set. During that reparse, builder functions reconstruct the in-memory
`Table` object, but do not create another b-tree or write another schema row.

Source:

- `sqlite/src/vdbe.c:7114` implements `OP_ParseSchema`.
- `sqlite/src/vdbe.c:7152` builds the `SELECT * FROM sqlite_schema ...` query.
- `sqlite/src/vdbe.c:7163` calls `sqlite3_exec(... sqlite3InitCallback ...)`.

## Halt and commit

The visible program ends at:

```text
Halt 0 0 0
```

`OP_Halt` calls `sqlite3VdbeHalt()`. For a successful auto-commit statement,
`sqlite3VdbeHalt()` calls `vdbeCommit()`, which drives b-tree commit phase one
and phase two. That is where dirty pages move from the pager/cache/journal world
toward a durable transaction result.

Runtime path:

```text
OP_Halt
  -> sqlite3VdbeHalt()
    -> vdbeCommit()
      -> sqlite3BtreeCommitPhaseOne()
      -> sqlite3BtreeCommitPhaseTwo()
```

Source:

- `sqlite/src/vdbe.c:1293` implements `OP_Halt`.
- `sqlite/src/vdbe.c:1354` calls `sqlite3VdbeHalt()`.
- `sqlite/src/vdbeaux.c:3315` implements `sqlite3VdbeHalt()`.
- `sqlite/src/vdbeaux.c:3421` calls `vdbeCommit()` for successful auto-commit.
- `sqlite/src/vdbeaux.c:2919` implements `vdbeCommit()`.
- `sqlite/src/vdbeaux.c:3005` calls `sqlite3BtreeCommitPhaseOne()`.
- `sqlite/src/vdbeaux.c:3020` calls `sqlite3BtreeCommitPhaseTwo()`.
- `sqlite/src/btree.c:4313` implements `sqlite3BtreeCommitPhaseOne()`.
- `sqlite/src/btree.c:4402` implements `sqlite3BtreeCommitPhaseTwo()`.

## Storage summary

For `CREATE TABLE t(a int)`, the storage-relevant state changes are:

1. `OP_Transaction` opens a write transaction on the main database b-tree.
2. `OP_CreateBtree` allocates and zeroes a new empty table root page.
3. The first `OP_Insert` writes a placeholder row to `sqlite_schema`.
4. The second `OP_Insert` overwrites it with:

   ```text
   ('table', 't', 't', rootpage, 'CREATE TABLE t(a int)')
   ```

5. `OP_SetCookie` bumps the schema version.
6. `OP_ParseSchema` reloads the new table definition into memory.
7. `OP_Halt` commits the transaction through the VDBE/b-tree/pager commit path.

The core bridge from VM to storage is small and beautiful:

```text
VDBE opcode
  -> btree API
    -> pager API
      -> database pages and journal/WAL discipline
```
