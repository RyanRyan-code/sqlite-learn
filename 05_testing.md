# SQLite testing

This note is for understanding how SQLite's tests are shaped, how to run the
Tcl tests locally, and what "unit test" means for internal functions such as
`btreeCreateTable()`.

## Where the tests live

The main Tcl tests are here:

```text
sqlite/test/*.test
```

Extension-specific Tcl tests are usually under:

```text
sqlite/ext/*/*.test
```

There are also multi-process tests under:

```text
sqlite/mptest
```

The C files named `src/test*.c` are generally not standalone unit-test files in
the usual xUnit sense. They are compiled into `testfixture` to expose extra Tcl
commands, fault-injection hooks, debug helpers, and test-only SQLite APIs.

So the rough structure is:

```text
Tcl .test script
  -> testfixture Tcl command
     -> SQL / sqlite3 API / test-only helper
        -> SQLite C code
```

## What Tcl is

Tcl is not SQLite-specific. It is a general-purpose scripting language whose
name means Tool Command Language.

It is in the same broad category as shell, Perl, Python, or Ruby: useful for
glue code, automation, embedding inside C programs, and writing compact test
scripts.

SQLite uses Tcl because Tcl is small, easy to embed in C, and was a natural
choice for C projects when SQLite was created. SQLite's tests are not plain Tcl
only; they are Tcl scripts running inside SQLite's custom `testfixture` shell.

That means a SQLite `.test` file is Tcl code that often contains SQL strings:

```tcl
do_execsql_test btree01-1.1 {
  CREATE TABLE t1(a INTEGER PRIMARY KEY, b BLOB);
  PRAGMA integrity_check;
} {ok}
```

In that example:

- `do_execsql_test` is a Tcl procedure from SQLite's test harness.
- The middle block is SQL text.
- `{ok}` is the expected result.

So Tcl is the scripting language. SQLite adds the testing vocabulary on top of
it.

## The `testfixture` binary

Most SQLite Tcl tests run under `testfixture`, not plain `sqlite3`.

`testfixture` is a custom Tcl shell linked with SQLite and SQLite's test-only C
helpers. It knows commands like:

```tcl
sqlite3 db test.db
do_execsql_test name { SQL } { expected result }
do_test name { Tcl script } { expected result }
```

It is built by the Makefile target:

```sh
make testfixture
```

In this checkout, the useful build directory is:

```text
/Users/albertw/Documents/source_code_libs/sqlite-debug
```

## `TCL_CONFIG_SH` on macOS

`testfixture` needs Tcl headers and linker flags. SQLite gets those from a file
named `tclConfig.sh`.

If `make testfixture sqlite3` fails with:

```text
TCL_CONFIG_SH must be set to point to a "tclConfig.sh"
```

then pass `TCL_CONFIG_SH` as a make variable:

```sh
cd /Users/albertw/Documents/source_code_libs/sqlite-debug
make TCL_CONFIG_SH=/Library/Developer/CommandLineTools/SDKs/MacOSX26.1.sdk/System/Library/Frameworks/Tcl.framework/tclConfig.sh testfixture sqlite3
```

This works in this checkout.

The subtle part: the configured `sqlite-debug/Makefile` has:

```make
TCL_CONFIG_SH =
```

Because of that, this shell-environment form may still fail:

```sh
TCL_CONFIG_SH=/path/to/tclConfig.sh make testfixture sqlite3
```

The command-line make-variable form overrides the Makefile assignment:

```sh
make TCL_CONFIG_SH=/path/to/tclConfig.sh testfixture sqlite3
```

The local system `tclsh` is Apple's Tcl 8.5:

```sh
/usr/bin/tclsh
```

SQLite's build accepts Tcl 8.5 or newer for `testfixture`.

## Run one Tcl test file

From the build directory:

```sh
cd /Users/albertw/Documents/source_code_libs/sqlite-debug
make TCL_CONFIG_SH=/Library/Developer/CommandLineTools/SDKs/MacOSX26.1.sdk/System/Library/Frameworks/Tcl.framework/tclConfig.sh testfixture sqlite3
./testfixture ../sqlite/test/btree01.test
```

That runs one Tcl script directly.

A btree-focused example:

```sh
./testfixture ../sqlite/test/btree02.test
```

If the test is an extension test, pass that file instead:

```sh
./testfixture ../sqlite/ext/session/session1.test
```

## Run test groups with `testrunner.tcl`

`testrunner.tcl` runs many Tcl tests in parallel.

Default, which means the `veryquick` set:

```sh
cd /Users/albertw/Documents/source_code_libs/sqlite-debug
../sqlite/test/testrunner.tcl
```

Equivalent:

```sh
../sqlite/test/testrunner.tcl veryquick
```

Run all normal Tcl tests:

```sh
../sqlite/test/testrunner.tcl full
```

Run the full suite plus permutations:

```sh
../sqlite/test/testrunner.tcl all
```

Run only tests whose file name matches a pattern:

```sh
../sqlite/test/testrunner.tcl btree%
../sqlite/test/testrunner.tcl 'btree*'
../sqlite/test/testrunner.tcl fts5%
```

The pattern matches the test script filename, not the test case name inside the
file. `%` is treated like Tcl's `*` wildcard.

## Make targets

These targets call into the same testing machinery:

```sh
cd /Users/albertw/Documents/source_code_libs/sqlite-debug
make testrunner
make devtest
make releasetest
make sdevtest
```

`make testrunner` builds `testfixture` and runs `testrunner.tcl`.

`make devtest` is the normal developer target. It does source-tree checks and
then runs a broader configured test set.

## Test logs

`testrunner.tcl` writes:

```text
testrunner.log
testrunner.db
```

Useful failure checks:

```sh
grep "^!" testrunner.log
grep failed testrunner.log
../sqlite/test/testrunner.tcl errors
../sqlite/test/testrunner.tcl errors -v
```

Check a running test:

```sh
../sqlite/test/testrunner.tcl status
../sqlite/test/testrunner.tcl status -d 2
```

Rerun failed or incomplete tests from the last run:

```sh
../sqlite/test/testrunner.tcl retest
```

## Is there a unit test for `btreeCreateTable()`?

Not in the narrow sense of:

```c
btreeCreateTable(...);
assert(...);
```

The function is `static`:

```c
static int btreeCreateTable(Btree *p, Pgno *piTable, int createTabFlags)
```

Because it is file-local to `btree.c`, the tests do not normally call it
directly.

The callable public wrapper is:

```c
int sqlite3BtreeCreateTable(Btree *p, Pgno *piTable, int flags)
```

That wrapper enters the btree mutex, calls the static implementation, then
leaves the mutex:

```text
sqlite3BtreeCreateTable()
  -> sqlite3BtreeEnter()
  -> btreeCreateTable()
  -> sqlite3BtreeLeave()
```

But even that wrapper is not usually tested as a direct C unit. It is mostly
exercised through SQL and VDBE execution.

## How SQL reaches `btreeCreateTable()`

For a normal rowid table:

```sql
CREATE TABLE t1(a INTEGER PRIMARY KEY, b);
```

the runtime path is:

```text
sqlite3_exec / sqlite3_step
  -> parser/code generator
  -> VDBE program
  -> OP_Transaction
  -> OP_CreateBtree
  -> sqlite3BtreeCreateTable()
  -> btreeCreateTable()
```

`OP_CreateBtree` calls:

```c
rc = sqlite3BtreeCreateTable(pDb->pBt, &pgno, pOp->p3);
```

The newly allocated root page number is returned through `pgno`, then stored in
the VDBE output register.

## Flag paths

`btreeCreateTable()` mainly chooses between two page formats:

```c
if( createTabFlags & BTREE_INTKEY ){
  ptfFlags = PTF_INTKEY | PTF_LEAFDATA | PTF_LEAF;
}else{
  ptfFlags = PTF_ZERODATA | PTF_LEAF;
}
zeroPage(pRoot, ptfFlags);
```

The important inputs are:

```text
BTREE_INTKEY   -> rowid table
BTREE_BLOBKEY  -> index or WITHOUT ROWID table
```

A normal table hits `BTREE_INTKEY`:

```sql
CREATE TABLE t1(a INTEGER PRIMARY KEY, b);
```

An index hits `BTREE_BLOBKEY`:

```sql
CREATE INDEX i1 ON t1(b);
```

A `WITHOUT ROWID` table also uses index-like btree storage:

```sql
CREATE TABLE t2(a PRIMARY KEY, b) WITHOUT ROWID;
```

So a small behavioral test for both branches would look like:

```tcl
do_execsql_test create-btree-flags-1 {
  CREATE TABLE t1(a INTEGER PRIMARY KEY, b);
  CREATE INDEX i1 ON t1(b);
  CREATE TABLE t2(a PRIMARY KEY, b) WITHOUT ROWID;
  PRAGMA integrity_check;
} {ok}
```

That is more SQLite-like than directly calling the static C function.

## What the btree tests look like

The files named `btree01.test` and `btree02.test` are btree-focused regression
tests, but they still test through SQL behavior.

`btree01.test` stresses btree balancing and integrity:

```sql
PRAGMA page_size=65536;
CREATE TABLE t1(a INTEGER PRIMARY KEY, b BLOB);
INSERT ...
UPDATE ...
PRAGMA integrity_check;
```

`btree02.test` stresses cursor save/restore behavior while modifying a table
during iteration.

This is typical SQLite testing style:

```text
find a real SQL-level behavior that reaches the internal code path
  -> run it through testfixture
  -> assert exact results, errors, or integrity_check output
```

## If you really wanted a direct test

A direct test of `btreeCreateTable()` would require changing the code shape,
because the function is `static`.

Possible approaches:

- Add a Tcl test command in a `src/test*.c` file that calls
  `sqlite3BtreeCreateTable()`.
- Expose a temporary test-only wrapper under a debug/test compile flag.
- Use existing SQL paths and add `testcase()` / coverage instrumentation instead
  of exposing the function.

SQLite usually prefers the last option. Internal functions remain private, and
tests prove behavior through stable public or semi-public paths.

## Source references

- `sqlite/doc/testrunner.md:33` describes `testrunner.tcl`.
- `sqlite/doc/testrunner.md:137` says binary tests expect an existing
  `testfixture`.
- `sqlite/doc/testrunner.md:151` explains where `*.test` files live.
- `sqlite/doc/testrunner.md:182` shows `veryquick`, `full`, pattern, and `all`
  commands.
- `sqlite/doc/testrunner.md:222` shows direct single-file execution with
  `./testfixture`.
- `sqlite/main.mk:1799` builds `testfixture`.
- `sqlite/main.mk:1836` defines the `testrunner` target.
- `sqlite/src/btree.c:10054` implements `btreeCreateTable()`.
- `sqlite/src/btree.c:10188` chooses `BTREE_INTKEY` versus index-like page
  flags.
- `sqlite/src/btree.c:10199` implements `sqlite3BtreeCreateTable()`.
- `sqlite/src/vdbe.c:7023` implements `OP_CreateBtree`.
- `sqlite/src/vdbe.c:7045` calls `sqlite3BtreeCreateTable()`.
- `sqlite/src/btree.h:87` declares `sqlite3BtreeCreateTable()`.
- `sqlite/src/btree.h:112` documents `BTREE_INTKEY` and `BTREE_BLOBKEY`.
- `sqlite/test/btree01.test:1` is a btree regression test.
- `sqlite/test/btree02.test:1` is another btree regression test.
