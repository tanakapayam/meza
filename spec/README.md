# meza Specification

**Spec version:** 1.0

## 1. Purpose

`meza` is a Unix-style structural editor for tables.

This specification defines the table model, addressing rules, transformations, input/output behavior, and conformance requirements independently of any particular implementation.

## 2. Table Model

A table consists of one header row, zero or more data rows, one stub column, and zero or more data columns.

The header row is row **0**. Data rows are numbered **1..N**, and are unique or empty.

The stub column is column **0**. Data columns are numbered **1..N**, and are unique or empty.

The stub is structural metadata for rows. Ordinary column transformations operate on data columns and must not modify column 0 unless explicitly specified.

A table is always rectangular: every row has the same number of columns.

Column headers and cell contents are strings.

## 3. Addressing

### 3.1 Column references

Column references may use numeric indices, column names, comma-separated lists, and half-closed (inclusive-start, exclusive-finish) ranges.

Examples:

- `COLS`
```
name
name,email
1
1,3,5
```
- `COL_RANGE`
```
name
1
2:4 ← [2,4) — from 2 onward, stops before 4
3:  ← from 3 onward
:4  ← everything before 4
:   ← all columns
```

Numeric data-column indices begin at 1. Column 0 is the stub.

### 3.2 Row references

Row references may use numeric indices, stub values, comma-separated lists, and half-closed ranges.

Numeric data-row indices begin at 1. Row 0 is the header.

### 3.3 Cell references

Cell-specific operations may identify cells using row and column references.

### 3.4 Escaping DSL Characters

The following need to be escaped to be included as part of keys and values:

```
, ← \,
; ← \;
: ← \:
= ← \=
\ ← \\
' ← \' — inside single-quoted strings
" ← \" — inside double-quoted strings
```

## 4. Operation Vocabulary

Unless explicitly qualified with `-row(s)`, `-cell(s)`, or `-table(s)`, structural operations operate on **columns**.

Stub head and columns are explicitly qualified.

* Canonical column operations:
```
--select
--drop
--duplicate
--move
--swap
--rename
--update
--prepend
--append
--insert-blank
--prune-blank
--zip
--unzip
--enquote
--unquote
--urlencode
--urldecode
```

* Canonical row operations:
```
--select-rows
--drop-rows
--duplicate-rows
--move-rows
--swap-rows
--rename-row
--update-rows
--prepend-rows
--append-rows
--insert-blank-row
--prune-blank-rows
--zip-rows
--unzip-row
--enquote-rows
--unquote-rows
--urlencode-rows
--urldecode-rows
```

* Canonical stub head, stub column and header row operations:
```
--get-stub-column
--set-stub-column
--assert-stub-column
--get-header-row
--set-header-row
--assert-header-row
--align-stub-head
--bold-stub-column
```

* Canonical cell operations:
```
--update-cells
--prepend-cells
--append-cells
--match-cells
--find-cells --replace-cells
```

* Canonical table operations:
```
--describe-table
--normalize-table
--no-normalize-table
--transpose-table
--rotate-table-90
--generate-blank-table
--paste-tables
--cat-tables
--cut-table
--split-tables
--add-tables
--extract-tables
```

The UNIX utilities `paste`, `cat`, `cut` and `split` were the inspiration for the names of the table operations.

## 5. Select

```
--select=COLS|COL_RANGE
```

Retains the selected data columns in the order specified by the reference. The stub column remains part of the table. Duplicate selections are errors.

### Row Select

Equivalent row operation:

```
--select-rows=ROWS|ROW_RANGE
```

## 6. Drop

```
--drop=COLS|COL_RANGE
```

Removes the selected data columns. Column 0 cannot be dropped.

### Row Drop

Equivalent row operation:

```
--drop-rows=ROWS|ROW_RANGE
```

## 7. Duplicate

```
--duplicate=COLS|COL_RANGE
```

Duplicates each selected column immediately after the original. Selected columns are processed left-to-right.

Duplicate column names are not permitted. Column 0 cannot be duplicated.

To ensure there are no duplicate column names, the name of the duplicate column follows these rules:

- If source column name has no counter, `-N` is added, where N is smallest number greater than 1, so that new name is not in conflict with any other header name.
- Otherwise, take the next higher counter.

Example:

```
A B C C-2 D
```

with `--duplicate=2:3` produces:

```
A B B-2 C C-3 C-2 D
```

### Row Duplicate

Equivalent row operation, following equivalent logic as header name for stub value:

```
--duplicate-rows=ROWS|ROW_RANGE
```

## 8. Move

```
--move=COL_RANGE,DEST
```

Moves the selected contiguous column block immediately before the destination column.

`DEST` may be a column reference or a numeric column position. The destination column must not be part of the moved block. Column 0 cannot be moved.

Numeric destinations are interpreted after removing the source block.

Example:

```
A B C D E
```

`--move=2:3,5` produces:

```
A D B C E
```

### Row Move

Equivalent row operation:

```
--move-rows=ROW_RANGE,DEST
```

## 9. Swap

```
--swap=COL_RANGE_A,COL_RANGE_B
```

Swaps two column blocks. The ranges must not overlap and may have different lengths. Column 0 cannot participate.

Example:

```
A B C D E
```

`--swap=2:3,5` produces:

```
A E D B C
```

### Row Swap

```
--swap-rows=ROW_RANGE_A,ROW_RANGE_B
```

These are equivalent.

## 10. Rename

```
--rename=COL=NEW_NAME,...
```

Renames selected column headers.

### Row Rename

Equivalent row operation:

```
--rename-row=ROW=NEW_VALUE,...
```

## 11. Update

```
--update=COLS|COL_RANGE=VALUE
```

Replaces every data cell in the selected column(s) with `VALUE`. An empty value is valid. The value is everything after the first `=`.

### Row Update

Equivalent row operation:

```
--update-rows=ROWS|ROW_RANGE=NEW_VALUE
```

### Cell Update

Equivalent cell operation:

```
--update-cells=ROW_RANGE,COL_RANGE=NEW_VALUE
```

## 12. Prepend and Append

```
--prepend=COLS|COL_RANGE=VALUE
```

Prepends `VALUE` to every data cell in the selected column(s).

```
--append=COLS|COL_RANGE=VALUE
```

Appends `VALUE` to every data cell in the selected column(s).

The first `=` separates the column reference from the value. The value is not stripped or normalized; whitespace is significant.

Examples:

```
--prepend='2= hello '
--append='3:5= world '
```

### Row Prepend and Append

Equivalent row operations:

```
--prepend-rows=ROWS|ROW_RANGE=VALUE
--append-rows=ROWS|ROW_RANGE=VALUE
```

### Cell Prepend and Append

Equivalent cell operations:

```
--prepend-cells=ROW_RANGE,COL_RANGE=VALUE
--append-cells=ROW_RANGE,COL_RANGE=VALUE
```

## 13. Insert and Prune Blank Columns

```
--insert-blank=POSITION
```

Inserts an empty data column immediately before the specified position. The inserted column has an empty header.

```
--prune-blank
```

Removes empty data columns.

### Insert and Prune Blank Rows

Equivalent row operations:

```
--insert-blank-row=POSITION
--prune-blank-rows
```

## 14. Zip

```
--zip=COL_RANGE;SEPARATOR
```

Collapses a contiguous range of columns into one column. For every data row, the selected cell values are joined using `SEPARATOR`.

`SEPARATOR` is a literal string, not a regular expression.

If all selected source headers are identical modulo numerical suffix, the resulting header is that common header; otherwise, the resulting header is the source headers joined using the same separator.

Thus:

```
first | middle | last
```

with separator `, ` becomes:

```
first, middle, last
```

while:

```
name | name-2 | name-3
```

becomes:

```
name
```

Arbitrary non-contiguous column lists are not valid zip sources.

For compatible tables where the separator does not create ambiguity in cell values:

```
unzip(zip(T)) = T
```

The reverse composition is not generally guaranteed:

```
zip(unzip(T)) != T
```

because `unzip` may create repeated headers.

### Row Zip

Equivalent row operation:

```
--zip-rows=ROW_RANGE;SEPARATOR
```

## 15. Unzip

```
--unzip=COL;SEPARATOR
```

Splits one data column into multiple columns using the literal `SEPARATOR`.

If the source header contains the separator, the header is split using the same separator and the resulting fields become the new headers.

If the source header does not contain the separator, the source header is repeated with a numerical increment suffix for each resulting column.

The number of resulting columns is determined by the data.

Every row must produce the same number of fields; otherwise, the operation is an error.

For compatible tables:

```
unzip(zip(T)) = T
```

### Row Unzip

Equivalent row operation:

```
--unzip-row=ROW;SEPARATOR
```

## 16. Enquote

```
--enquote=COLS|COL_RANGE[;SPEC]
```

Wraps every data cell in the selected columns with a marker or delimiter pair.

The default marker is `"`.

Supported single markers include:

```
"
'
`
/
-
_
:
*
```

Supported delimiter pairs include:

```
[]
()
{}
<>
«»
»«
„“
¡!
¿?
```

Examples:

```
--enquote=2
--enquote='2;"'
--enquote="2;'"
--enquote='2;[]'
```

The header and stub are not modified.

The operation is intentionally non-idempotent: applying it twice quotes the result again. Existing quotation is not interpreted specially.

The `;` is part of the `meza` argument grammar. Shell quoting is separate.

### Row Enquote

Equivalent row operation:

```
--enquote-rows=ROWS|ROW_RANGE[;SPEC]
```

## 17. Unquote

```
--unquote=COLS|COL_RANGE[;SPEC]
```

Strips matching outer markers or delimiter pairs from every data cell in the selected columns.

The default marker is `"` and the same marker/pair syntax as `--enquote` is used.

Stripping occurs only when the configured outer delimiters are both present.

For marker `"`:

```
"foo"  -> foo
foo    -> foo
"foo   -> "foo
foo"   -> foo"
```

For delimiter pair `[]`:

```
[foo]  -> foo
foo    -> foo
[foo   -> [foo
foo]   -> foo]
```

The header and stub are not modified.

For values produced by `--enquote`, the corresponding `--unquote` operation restores the original values.

### Row Unquote

Equivalent row operation:

```
--unquote-rows=ROWS|ROW_RANGE[;SPEC]
```

## 18. URL-Encode

```
--urlencode=COLS|COL_RANGE
```

### Row URL-Encode

Equivalent row operation:

```
--urlencode-rows=ROWS|ROW_RANGE
```

## 19. URL-Decode

```
--urldecode=COLS|COL_RANGE
```

### Row URL-Decode

Equivalent row operation:

```
--urldecode-rows=ROWS|ROW_RANGE
```

## 20. Stub Column

```
--get-stub-column
--set-stub-column=VALUE_1,VALUE_2,...
```

Get and set the unique values of stub column.

```
--assert-stub-column=VALUE_1,VALUE_2,...
```

Affects exit code: Exits 0 or 1 on stub column matching assertion or not.

## 21. Header Row

```
--get-header-row
--set-header-row=NAME_1,NAME_2,...
```

Get and set the unique values of header row.

```
--assert-header-row=NAME_1,NAME_2,...
```

Affects exit code: Exits 0 or 1 on header row matching assertion or not.

## 22. Stub Formatting

```
--align-stub-head=:--
--align-stub-head=:-:
--align-stub-head=--:
```

select left, center, or right stub-header alignment syntax in Markdown output.

```
--bold-stub-column
```

makes the stub column bold in Markdown and Rich output.

These options affect presentation and do not change the logical table.

## 23. Match Cells

```
--match-cells=/REGEX/
```

Produces another table with the same dimensions, headers, and stub values.

Each data cell is replaced with `1` if the regular expression finds a match anywhere in the cell, or `0` otherwise.

The header and stub are not matched.

The `/.../` delimiters are CLI syntax and are not part of the pattern. Matching uses search semantics rather than full-string matching.

Invalid regular expressions are errors.

## 24. Find and Replace Cells

```
--find-cells=/REGEX/ --replace-cells=LITERAL
```

Produces another table with the same dimensions, headers, and stub values.

Each data cell is replaced with `LITERAL` if the regular expression finds a match anywhere in the cell, or is untouched otherwise. The header and stub are not matched.

The `/.../` delimiters are CLI syntax and are not part of the pattern. Matching uses search semantics rather than full-string matching.

Invalid regular expressions are errors.

## 25. Describe Table

```
--describe-table
```

Reports structural information including input format, row count, column count, stub, column indices, and column names.

## 26. Normalization of Table

Normalization is enabled by default.

```
--normalize-table
```

is the explicit form.

```
--no-normalize-table
```

disables normalization.

If an operation requires normalization (e.g. `transpose`), invocation of `--no-normalize-table` results in an incompatible-mix error.

## 27. Transpose Table

```
--transpose-table
```

Transposes the table while treating the first column as row labels.

Given:

```
|     | a | b | c |
|-----|---|---|---|
| A   | 1 | 2 | 3 |
| B   | 4 | 5 | 6 |
```

the result is:

```
|     | A | B |
|-----|---|---|
| a   | 1 | 4 |
| b   | 2 | 5 |
| c   | 3 | 6 |
```

Transpose requires normalization.

## 28. Rotate Table 90 Degrees Clockwise

```
--rotate-table-90
```

Rotates the table clockwise, including header row and stub column.

Given:

```
|     | a | b  | c  | d  |
|-----|---|----|----|----|
| A   | 1 |  2 |  3 |  4 |
| B   | 5 |  6 |  7 |  8 |
| C   | 9 | 10 | 11 | 12 |
```

the result is:

```
|     | C  | B | A |
|-----|----|---|---|
| a   |  9 | 5 | 1 |
| b   | 10 | 6 | 2 |
| c   | 11 | 7 | 3 |
| d   | 12 | 8 | 4 |
```

Rotate requires normalization.

## 29. Generate Blank Table

```
--generate-blank-table=ROW_SIZE,COL_SIZE
```

Generates a blank `ROW_SIZE`x`COL_SIZE` table, not counting header row and stub column.

Header row is a sequential list of letters: `A`..`Z`,`AA`..`AZ`,...

Stub column is a sequential list of numbers: 1..`ROW_SIZE`

## 30. Paste Tables

```
--paste-tables FILE [FILE ...]
```

Pastes (joins) tables side-by-side.

All inputs must have the same number of rows and exactly the same stub values in the same order.

Resulting data-column headers must not conflict.

No implicit sorting, coercion, schema reconciliation, or arbitrary key join is performed.

### Cat Tables

```
--cat-tables FILE [FILE ...]
```

Catenates (joins) tables sequentially.

All inputs must have the same number of columns and exactly the same header values in the same order.

Resulting stub-row values must not conflict.

No implicit sorting, coercion, schema reconciliation, or arbitrary key join is performed.

## 31. Cut Table

```
--cut-table=COLS;FILENAME_PATTERN FILE.ext
```

Cuts table at columns into new tables with new, non-conflicting filenames based on `FILENAME_PATTERN`, which can have one `#` symbol for numbering scheme.

### Split Tables

```
--split-table=ROWS;FILENAME_PATTERN FILE
```

Splits table at rows into new tables with new, non-conflicting filenames based on `FILENAME_PATTERN`, which can have one `#` symbol for numbering scheme.

## 32. Add Tables

```
--add-tables FILE [FILE ...]
```

Given tables of exact same dimensions, `--add-tables` produces a new table of same dimensions with containing cell-wise catenation of the cells of the original tables.

For example:

- `table-a.md`
```
|   | a | b | c | d |
|---|---|---|---|---|
| 1 | A | B | C | D |
| 2 | E | F | G | H |
| 3 | I | J | K | L |
| 4 | M | N | O | P |
```
- `table-b.md`
```
|   | a | b | c | d |
|---|---|---|---|---|
| 1 | α | β | γ | δ |
| 2 | ε | ζ | η | θ |
| 3 | ι | κ | λ | μ |
| 4 | ν | ξ | ο | π |
```
- `--add-tables table-a.md table-b.md`
```
|   |  a |  b |  c |  d |
|---|----|----|----|----|
| 1 | Aα | Bβ | Cγ | Dδ |
| 2 | Eε | Fζ | Gη | Hθ |
| 3 | Iι | Jκ | Kλ | Lμ |
| 4 | Mν | Nξ | Oο | Pπ |
```

## 33. Extract Tables

```
--extract-tables=FILENAME_PATTERN FILE [FILE ...]
```

Extracts valid tables from Markdown files and saves them with non-conflicting filenames based on `FILENAME_PATTERN`, which can have one `#` symbol for numbering scheme.

## 34. Input Formats

Input format is detected from the first valid row of input:

- Comma-separated-values → CSV
- Pipe-delimited-values → Markdown

Input sources can be:

- Stdin
```
meza -
```
- Local File
```
meza table.md
```
- Remote URI
```
meza 'https://docs.google.com/spreadsheets/d/{SPREADSHEET_ID}/export?format=csv&gid={SHEET_ID}'
```

## 35. Output Formats

Available output formats:

```
--markdown
--csv
--rich
```

Without an explicit output format, output uses the input format.

`--rich` is an output format only.

## 36. Markdown

Markdown input consists of a header row, a separator row, and zero or more data rows.

Normalization may canonicalize whitespace and alignment.

Canonical Markdown output is rectangular and consistently padded.

Transposed Markdown output includes the empty top-left corner.

## 37. CSV

CSV input and output use standard CSV quoting rules.

CSV comment lines are lines beginning exactly with `#`.

For transformations and conversions from CSV to another format, comment lines are ignored.

CSV input with no transformation and CSV output is an exact byte-preserving
no-op.

## 38. Errors

Invalid operations must report an error and must not emit a partially transformed table.

Examples include:

- nonexistent column references
- duplicate references where prohibited
- attempts to modify the stub with a data-column-only operation
- invalid regular expressions
- invalid ranges
- overlapping swap ranges
- invalid move destinations
- non-contiguous zip sources
- inconsistent unzip cardinality
- incompatible join schemas
- malformed input tables
- incompatibile mix of options

Error wording is implementation-defined unless a fixture explicitly requires a particular error condition.

## 39. Conformance

A conformance suite should test logical table transformations separately from serialization where possible.

A normative fixture consists of:

- a specification version
- an input table
- an operation
- an expected table or expected error

Implementations may use any internal representation. Observable behavior must conform to this specification.

## 40. Versioning

The specification version is independent of the implementation version.

A patch version may clarify wording without changing semantics.

A minor version may add backward-compatible operations or behavior.

A major version may change existing semantics or remove operations.

Fixtures should declare the specification version they target.

## 41. Non-goals

Version 1 does not define:

- SQL execution
- arbitrary relational joins
- implicit type coercion
- sorting
- filtering with SQL `WHERE` expressions
- formulas
- database connectivity
- spreadsheet formatting semantics
- arbitrary Markdown extensions
- a required programming language or implementation architecture

The specification defines table transformations, not a database engine.
