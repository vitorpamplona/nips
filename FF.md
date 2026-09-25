NIP-FF
======

Nostr Query Language (NQL)
--------------------------

`draft` `optional` `relay`

This NIP defines NQL, a small, strictly typed, read-only query language over Nostr events, and the relay commands that run it. NQL's syntax is a subset of SQL's `SELECT`, but it is not a general database language: its only data are the events a relay holds, seen through two fixed sources, `events` and `tags`. There are no user tables, no schema, no writes and no extensions.

A relay that supports this NIP MUST implement **everything** defined here, and nothing it answers may differ from what this document specifies. Clients can therefore send the same query to any supporting relay. The language is closed: it grows only by revising this NIP.

## Motivation

`REQ` returns events matching a filter; `COUNT` ([NIP-45](45.md)) returns how many. Neither can group, aggregate, join an event to the events its tags reference, read a tag beyond its first value, or compute over tag values. Clients answer those questions by downloading every matching event: all reactions to count them per note, all kind `10002` lists to collect relay urls, all zap receipts to sum their amounts. NQL moves that work to the relay and returns only the answer.

## Data model

A query ranges over the events the relay would serve this connection for a `REQ` with no filter conditions and no limit: every access rule, [NIP-42](42.md) requirement, [NIP-09](09.md) deletion and [NIP-40](40.md) expiration that applies to `REQ` applies here too. Relays MUST NOT apply ranking, search, trust or result caps to that set. What a `REQ` would hide, NQL hides; what it would serve, NQL sees in full.

These events are visible through exactly two sources.

### `events`: one row per event

| Column       | Type    | Value                               |
|--------------|---------|-------------------------------------|
| `id`         | TEXT    | the event's `id` (64 lowercase hex) |
| `pubkey`     | TEXT    | the event's `pubkey`                |
| `created_at` | INTEGER | the event's `created_at`            |
| `kind`       | INTEGER | the event's `kind`                  |
| `content`    | TEXT    | the event's `content`               |
| `sig`        | TEXT    | the event's `sig`                   |

None of these columns is ever NULL.

### `tags`: one row per tag of every event

| Column       | Type    | Value                                                     |
|--------------|---------|-----------------------------------------------------------|
| `event_id`   | TEXT    | the `id` of the event holding the tag                     |
| `idx`        | INTEGER | the tag's 0-based position in that event's `tags` array   |
| `name`       | TEXT    | `tag[0]`                                                  |
| `value`      | TEXT    | `tag[1]`, or NULL when the tag has fewer than 2 elements  |
| `v2`         | TEXT    | `tag[2]`, or NULL when absent                             |
| `v3`         | TEXT    | `tag[3]`, or NULL when absent                             |
| `v4`         | TEXT    | `tag[4]`, or NULL when absent                             |
| `created_at` | INTEGER | the holding event's `created_at`                          |
| `kind`       | INTEGER | the holding event's `kind`                                |
| `pubkey`     | TEXT    | the holding event's `pubkey`                              |

A tag that is an empty array has no row. Elements past `tag[4]` are not visible. `created_at`, `kind` and `pubkey` repeat the holding event's fields so that most tag questions need no join.

## Types and values

Every value has one of four types:

- **INTEGER**: a signed 64-bit integer.
- **REAL**: an IEEE 754 binary64 number, including ±Infinity. NQL never produces NaN.
- **TEXT**: a sequence of Unicode code points. Lengths, positions and ordering are in code points.
- **BOOLEAN**: `TRUE` or `FALSE`.

`NULL` is the absence of a value and belongs to every type. INTEGER and REAL are the *numeric* types.

Every expression has a single static type, determined before execution by the rules below. A query that breaks a typing rule is invalid and MUST be refused before any row is produced.

The only implicit conversion is **numeric promotion**: where an operator or function accepts two numeric operands of different types, the INTEGER operand is converted to REAL. Every other conversion requires `CAST`.

A `NULL` literal, and a parameter whose value is `null`, takes its type from its context: the other operand of an operator, or the common type of a `CASE`, `coalesce`, `IN` list or `UNION` column. Where nothing fixes it, as in `SELECT NULL`, its type is TEXT.

## Syntax

A query is a single statement, optionally followed by one `;`. Keywords and identifiers are case-insensitive. Whitespace separates tokens; there are no comments.

```
query         = select-core { "UNION" [ "ALL" ] select-core } [ order-by ] [ limit ]
select-core   = "SELECT" [ "DISTINCT" | "ALL" ] result-column { "," result-column }
                [ "FROM" from ]
                [ "WHERE" expr ]
                [ "GROUP" "BY" expr { "," expr } [ "HAVING" expr ] ]
result-column = "*" | name "." "*" | expr [ [ "AS" ] name ]
from          = source { ( "," | "CROSS" "JOIN" ) source
                       | ( [ "INNER" ] "JOIN" | "LEFT" [ "OUTER" ] "JOIN" ) source "ON" expr }
source        = ( "events" | "tags" ) [ [ "AS" ] name ]
              | "(" query ")" [ "AS" ] name
order-by      = "ORDER" "BY" order-term { "," order-term }
order-term    = expr [ "ASC" | "DESC" ] [ "NULLS" ( "FIRST" | "LAST" ) ]
limit         = "LIMIT" expr [ "OFFSET" expr ]

expr          = or-expr
or-expr       = and-expr { "OR" and-expr }
and-expr      = not-expr { "AND" not-expr }
not-expr      = "NOT" not-expr | equality
equality      = relation { equality-op relation
                         | "IS" [ "NOT" ] relation
                         | [ "NOT" ] "IN" "(" ( query | expr { "," expr } ) ")"
                         | [ "NOT" ] "LIKE" relation [ "ESCAPE" relation ]
                         | [ "NOT" ] "BETWEEN" relation "AND" relation }
equality-op   = "=" | "!=" | "<>"
relation      = additive { ( "<" | "<=" | ">" | ">=" ) additive }
additive      = multiplicative { ( "+" | "-" ) multiplicative }
multiplicative= concat { ( "*" | "/" | "%" ) concat }
concat        = unary { "||" unary }
unary         = ( "-" | "+" ) unary | primary
primary       = literal | parameter | column | call | cast | case
              | "(" expr ")" | "(" query ")" | "EXISTS" "(" query ")"
column        = [ name "." ] name
call          = function-name "(" [ [ "DISTINCT" ] expr { "," expr } | "*" ] ")"
cast          = "CAST" "(" expr "AS" ( "INTEGER" | "REAL" | "TEXT" ) ")"
case          = "CASE" [ expr ] "WHEN" expr "THEN" expr { "WHEN" expr "THEN" expr }
                [ "ELSE" expr ] "END"
literal       = integer | real | string | "NULL" | "TRUE" | "FALSE"
parameter     = "?" | ":" name
```

Binary operators on the same line of the grammar are left-associative. The `AND` inside `BETWEEN` belongs to `BETWEEN`; its operands are `relation`s, so `x BETWEEN a AND b AND c` means `(x BETWEEN a AND b) AND c`.

Lexical rules:

- **Names** match `[A-Za-z_][A-Za-z0-9_]*` and MUST NOT be a keyword. The keywords are: `ALL AND AS ASC BETWEEN BY CASE CAST CROSS DESC DISTINCT ELSE END ESCAPE EXISTS FALSE FIRST FROM GROUP HAVING IN INNER INTEGER IS JOIN LAST LEFT LIKE LIMIT NOT NULL NULLS OFFSET ON OR ORDER OUTER REAL SELECT TEXT THEN TRUE UNION WHEN WHERE`.
- **Integers** are one or more decimal digits and MUST fit in a signed 64-bit integer. The one exception is `9223372036854775808`, which is valid only directly after a unary `-`.
- **Reals** are `digits "." [digits] [exponent]`, `"." digits [exponent]` or `digits exponent`, where `exponent = ("e" | "E") ["+" | "-"] digits`. They round to the nearest binary64 value.
- **Strings** are enclosed in single quotes; `''` inside a string stands for one quote. There are no other escapes.
- **Parameters** are either all `?` or all `:name` within one query. The n-th `?` takes the n-th positional parameter; `:name` takes the named one.

## Semantics

### Names and scopes

Each `source` in a `FROM` has a name: its alias, or `events`/`tags` when it has none. Names within one `FROM` MUST be unique. A subquery in `FROM` MUST have an alias; its columns are its result columns, named as defined under [Result columns](#result-columns).

A qualified column `s.c` refers to column `c` of source `s`. An unqualified column `c` refers to the only source in the innermost enclosing `FROM` that has a column `c`. If no source there has one, the next enclosing query's `FROM` is searched (a correlated reference). If two sources at the same level have it, the query is invalid. A reference that resolves nowhere makes the query invalid.

### Evaluation

A `select-core` behaves as follows:

1. The `FROM` sources are combined left to right:
   - `,` and `CROSS JOIN` pair every row with every row.
   - `JOIN … ON` keeps the pairs whose `ON` is TRUE.
   - `LEFT JOIN … ON` also keeps each left row that matched nothing, paired with NULLs.
2. `WHERE` keeps the rows for which it is TRUE.
3. Rows are grouped (see [Grouping](#grouping-and-aggregates)).
4. `HAVING` keeps the groups for which it is TRUE.
5. The result columns are computed, and `DISTINCT` removes duplicate rows.

`SELECT` without `FROM` yields exactly one row. `UNION ALL` concatenates results; `UNION` also removes duplicate rows. Then `ORDER BY` sorts the whole result, and `LIMIT`/`OFFSET` select a slice of it.

`WHERE`, `HAVING`, `ON`, `NOT`, `AND`, `OR` and `CASE … WHEN` conditions MUST be BOOLEAN.

Without `ORDER BY`, the row order is unspecified. With it, the order of rows that compare equal on every term is unspecified.

Two values are **duplicates** (for `DISTINCT`, `UNION` and `GROUP BY`) when they are equal or both NULL.

### Result columns

A query MUST NOT have more than one `*` result column that expands to the same source.

- `*` expands to every column of every `FROM` source, in `FROM` order and table order.
- `s.*` expands to every column of source `s`.

Each result column is named:

- by its alias, if it has one;
- otherwise by the column's name, if the expression is a bare column;
- otherwise by the expression's text exactly as written in the query, from its first to its last character.

In a `UNION`, all `select-core`s MUST have the same number of columns. Each column's type is the common type of that column in every core (see [Common type](#common-type)). The names come from the first core.

### Common type

Where the rules ask for the *common type* of several expressions:

- If all are NULL literals, the common type is TEXT.
- Otherwise the NULL literals take the type of the others.
- If all the others are numeric and at least one is REAL, the common type is REAL.
- Otherwise all of them MUST have the same type.

### Operators

In the tables below, *N* is any numeric type.

| Operator | Operands | Result | Notes |
|---|---|---|---|
| `+` `-` `*` | N, N | INTEGER if both are INTEGER, else REAL | |
| `/` | N, N | INTEGER if both are INTEGER, else REAL | INTEGER division truncates toward zero |
| `%` | INTEGER, INTEGER | INTEGER | the result has the sign of the left operand |
| unary `-`, `+` | N | same type | |
| `\|\|` | TEXT, TEXT | TEXT | concatenation |
| `=` `!=` `<>` | any common type | BOOLEAN | |
| `<` `<=` `>` `>=` | numeric, or TEXT, TEXT | BOOLEAN | |
| `IS`, `IS NOT` | any common type | BOOLEAN | never NULL (see below) |
| `IN`, `NOT IN` | any common type | BOOLEAN | |
| `BETWEEN` | numeric, or all TEXT | BOOLEAN | |
| `LIKE` | TEXT, TEXT [, `ESCAPE` TEXT] | BOOLEAN | |
| `NOT` | BOOLEAN | BOOLEAN | |
| `AND`, `OR` | BOOLEAN, BOOLEAN | BOOLEAN | |

Numeric rules:

- **Division or remainder by zero** (INTEGER or REAL) yields NULL.
- **INTEGER overflow**: an INTEGER `+`, `-`, `*`, unary `-` or `abs()` whose exact result does not fit in 64 bits yields NULL.
- **Infinity**: REAL arithmetic follows IEEE 754, so it may produce ±Infinity but never NaN.

NULL and comparison rules:

- Every operator except `IS`, `IS NOT`, `AND` and `OR` yields NULL when any operand is NULL.
- `a IS b` is TRUE when `a` and `b` are equal or both NULL, and FALSE otherwise; `IS NOT` is its negation.
- `AND` and `OR` use three-valued logic: `FALSE AND NULL` is FALSE and `TRUE OR NULL` is TRUE; every other combination with NULL is NULL.
- TEXT compares code point by code point, and a string sorts before any longer string that starts with it. Numbers compare by value, after numeric promotion. BOOLEAN supports only equality, `IS` and `IN`.

`IN`, `BETWEEN` and `LIKE`:

- **`x IN (…)`** is TRUE if `x` equals any element. It is NULL if no element equals `x` and either `x` or some element is NULL. It is FALSE otherwise.
  - `IN (query)`: the query MUST have exactly one column, and its elements are that column's values.
  - `NOT IN` is the negation of `IN`, with NULL staying NULL.
- **`x BETWEEN a AND b`** is `x >= a AND x <= b`.
- **`x LIKE p`** is TRUE if `x` matches the whole of pattern `p`:
  - `%` matches any sequence of code points, and `_` matches exactly one.
  - ASCII letters match regardless of case; every other code point matches only itself.
  - With `ESCAPE e`, `e` MUST be a string literal or a parameter of exactly one code point; anything else makes the query invalid. In the pattern, `e` followed by any code point matches that code point literally.

### `CAST`

| From \ To | INTEGER | REAL | TEXT |
|---|---|---|---|
| INTEGER | itself | the nearest REAL | decimal digits, with a leading `-` if negative |
| REAL | toward zero; ±Infinity and out-of-range values saturate to the INTEGER limits | itself | *invalid* |
| TEXT | see below | see below | itself |
| BOOLEAN | `TRUE` → 1, `FALSE` → 0 | *invalid* | *invalid* |

NULL casts to NULL.

**TEXT to INTEGER:**
1. Skip leading whitespace.
2. Read an optional `+` or `-`, then the longest run of decimal digits.
3. If there are no digits, the result is 0.
4. A value beyond the 64-bit range saturates to the nearest limit.
5. Everything after the digits is ignored, so `'12abc'` → 12, `'3.9'` → 3, `'1e3'` → 1 and `'abc'` → 0.

**TEXT to REAL:**
1. Skip leading whitespace.
2. Read the longest prefix that forms an optionally signed `integer` or `real` literal.
3. The result is its value, or 0.0 when there is none. `'1.5e2x'` → 150.0.

REAL to TEXT is not part of the language.

### `CASE`

`CASE WHEN c1 THEN r1 … [ELSE e] END` yields the first `r` whose `c` is TRUE. If none is, it yields `e`, or NULL when there is no `ELSE`.

`CASE x WHEN v1 THEN r1 …` is the same as `CASE WHEN x = v1 THEN r1 …`, except that `x` is evaluated once.

The `THEN` and `ELSE` results MUST have a common type, which is the type of the `CASE`.

### Subqueries

- A subquery used as a value MUST have exactly one column, and its type is that column's type. It yields the value in the first row of its result, or NULL when there is no row.
- `EXISTS (query)` is TRUE if the query yields at least one row, and FALSE otherwise.
- Subqueries may refer to the columns of enclosing queries.

### Grouping and aggregates

A `select-core` is **grouped** if it has `GROUP BY`, or if its result columns or `HAVING` contain an aggregate call.

- **Groups:** with `GROUP BY`, rows are grouped by the values of the `GROUP BY` expressions, with duplicates as defined above. Without it, all rows form one group, and that group exists even when there are no rows.
- **Where aggregates may appear:** only in result columns, `HAVING` and `ORDER BY`, and never inside another aggregate.
- **Grouping terms:** a `GROUP BY` term is resolved like an `order-term` (see below): an integer literal *k* is the *k*-th result column, a bare name that is a result column's name is that column, and anything else is an expression over the `FROM` sources.
- **What a grouped expression may reference:** outside an aggregate's arguments, an expression MUST refer only to the `GROUP BY` expressions (matched as written, ignoring case and whitespace) or to columns listed bare in `GROUP BY`.
- `HAVING` requires `GROUP BY`.

The aggregates, where *x* is any expression:

| Aggregate | Argument | Result | Value |
|---|---|---|---|
| `count(*)` | none | INTEGER | the number of rows in the group |
| `count(x)` | any | INTEGER | the number of non-NULL `x` |
| `sum(x)` | N | same type as `x` | the sum of non-NULL `x`, or NULL if there are none. An INTEGER sum that overflows fails the query (`error:`). A REAL sum uses compensated (Kahan–Babuška–Neumaier) summation |
| `avg(x)` | N | REAL | the compensated sum of non-NULL `x` divided by their count, or NULL if there are none |
| `min(x)`, `max(x)` | N or TEXT | same type as `x` | the least or greatest non-NULL `x`, or NULL if there are none |

With `DISTINCT` (for example `count(DISTINCT x)`), the aggregate considers each distinct non-NULL value once.

Row order within a group is unspecified, so a REAL `sum` or `avg` may differ in the last bits between relays. INTEGER results are exact.

### `ORDER BY`, `LIMIT`, `OFFSET`

An `order-term` refers to one of three things:

- an integer literal *k*, meaning the *k*-th result column (1-based);
- a bare name that is a result column's name, meaning that column;
- any other expression, evaluated per row.

After a `UNION` or `DISTINCT`, only the first two forms are allowed. Terms MUST be numeric or TEXT.

A bare name that is both a result column's name and a column of a `FROM` source makes the query invalid, in `ORDER BY` and `GROUP BY` alike, unless the result column is that same source column.

`ASC` is the default. NULLs sort first for `ASC` and last for `DESC`, unless `NULLS FIRST` or `NULLS LAST` says otherwise.

`LIMIT` and `OFFSET` MUST be INTEGER expressions that use only literals and parameters. Their values MUST be non-negative, or the query is invalid.

## Functions

These are all the functions; no other name is callable. In the signatures, *N* is any numeric type. Every function returns NULL when any argument is NULL, except `coalesce`, `ifnull` and `nullif`. An argument of the wrong type makes the query invalid.

### Text

| Function | Result | Value |
|---|---|---|
| `length(TEXT)` | INTEGER | the number of code points |
| `lower(TEXT)`, `upper(TEXT)` | TEXT | ASCII letters converted; every other code point unchanged |
| `trim(TEXT [, TEXT])`, `ltrim(…)`, `rtrim(…)` | TEXT | removes, from both ends / the start / the end, every code point that appears in the second argument (a space when omitted) |
| `replace(x TEXT, from TEXT, to TEXT)` | TEXT | `x` with every non-overlapping occurrence of `from`, scanned left to right, replaced by `to`; `x` unchanged when `from` is empty |
| `instr(x TEXT, y TEXT)` | INTEGER | the 1-based position of the first occurrence of `y` in `x`: 0 if there is none, 1 if `y` is empty |
| `substr(x TEXT, start INTEGER [, len INTEGER])` | TEXT | see below |

`substr` counts in code points. With `L` = `length(x)`, `p` = `start`, and `n` = `len` (unbounded when omitted), it computes as follows and returns code points `[p, p + n)` of `x` (0-based), clipped to `[0, L)`:

```
neg = n < 0;   if neg: n = -n
if p < 0:      p = p + L;  if p < 0: n = max(0, n + p); p = 0
else if p > 0: p = p - 1
else if n > 0: n = n - 1        # start 0 selects one fewer code point
if neg:        p = p - n;  if p < 0: n = n + p; p = 0
```

So `substr('abcdef', 2, 3)` = `'bcd'`, `substr('abcdef', 0, 2)` = `'a'`, `substr('abcdef', -2)` = `'ef'`, `substr('abcdef', 3, -2)` = `'ab'` and `substr('abcdef', -10, 3)` = `''`.

### Conditional

| Function | Result | Value |
|---|---|---|
| `coalesce(x, y, …)` (2 or more arguments) | common type | the first non-NULL argument, or NULL |
| `ifnull(x, y)` | common type | `coalesce(x, y)` |
| `nullif(x, y)` | type of `x` | NULL if `x = y` is TRUE, otherwise `x`; `x` and `y` MUST have a common type |

### Numeric

| Function | Result | Value |
|---|---|---|
| `abs(N)` | same type | the absolute value; `abs` of the smallest INTEGER is NULL |
| `sign(N)` | INTEGER | -1, 0 or 1 |
| `round(N [, INTEGER])` | REAL | the argument rounded to the given number of decimal places (default 0; values below 0 count as 0 and above 30 as 30). Ties go away from zero, judged on the exact binary value, so `round(2.675, 2)` = 2.67 because the stored value is just below 2.675 |
| `ceil(N)`, `ceiling(N)`, `floor(N)`, `trunc(N)` | same type | toward +∞, toward -∞, toward zero; INTEGER arguments are returned unchanged |
| `mod(N, N)` | REAL | the remainder of truncating division, with the sign of the first argument (`fmod`); NULL when the second argument is 0 |
| `pi()` | REAL | π |
| `sqrt(N)` | REAL | the square root |
| `exp(N)` | REAL | *e*^x |
| `pow(N, N)`, `power(N, N)` | REAL | x^y |
| `ln(N)` | REAL | the natural logarithm |
| `log(N)`, `log10(N)` | REAL | the base-10 logarithm |
| `log(b N, x N)` | REAL | the base-*b* logarithm of *x* |
| `log2(N)` | REAL | the base-2 logarithm |
| `degrees(N)`, `radians(N)` | REAL | angle conversion |
| `sin` `cos` `tan` `asin` `acos` `atan` `sinh` `cosh` `tanh` `asinh` `acosh` `atanh` (N) | REAL | the trigonometric and hyperbolic functions, in radians |
| `atan2(y N, x N)` | REAL | the angle of the point (x, y), in radians |

Every REAL function yields NULL outside its mathematical domain, and wherever the IEEE 754 result would be NaN. Examples: `sqrt(-1)`, `ln(0)`, `ln(-1)`, `acos(2)`, `acosh(0.5)`, `pow(-8, 1.0/3)`, `log(1, 5)`, `log(0, 5)`.

Overflow and poles yield ±Infinity, as IEEE 754 does: `exp(1000)`, `pow(10, 400)`, `pow(0, -1)`, `atanh(1)`.

Results of `exp`, `ln`, `log*`, `pow`, `sqrt` and the trigonometric functions MUST be within one unit in the last place of the exact result. Clients MUST NOT expect them to be bit-identical across relays.

## Protocol

### Running a query

```
["NQL", <query_id>, <query>, <options>?]
```

- `query_id` is an arbitrary non-empty string, with the same scoping as a `REQ` subscription id.
- `query` is the query text.
- `options` is an optional object with two optional fields:
  - `params`: a JSON array of positional parameters, or an object of named parameters.
  - `page`: the maximum number of rows to send first.

Each parameter's type comes from its JSON value:

| JSON value | Type |
|---|---|
| an integer within the 64-bit range | INTEGER |
| any other number | REAL |
| a string | TEXT |
| `true` / `false` | BOOLEAN |
| `null` | NULL, compatible with any type |

A query that references a parameter it was not given is invalid.

When the relay accepts the query, it replies with the result's column names and types, then the first page of rows:

```
["NQL-COLS", <query_id>, [[<name>, <type>], ...]]
["NQL-ROWS", <query_id>, [[<value>, ...], ...], "more" | "done"]
```

- `type` is `"INTEGER"`, `"REAL"`, `"TEXT"` or `"BOOLEAN"`.
- Values are JSON integers, numbers, strings, `true`/`false` or `null`.
- ±Infinity, which JSON cannot carry, is sent as the string `"Infinity"` or `"-Infinity"` in a REAL column.
- Clients MUST parse INTEGER values as 64-bit integers.

The first page holds at most `page` rows. Without `page`, the relay chooses the size, which SHOULD be the default `limit` it applies to `REQ`.

### Paging

`"done"` means there are no more rows; the relay has discarded the query, and `query_id` can be reused. `"more"` means rows remain. The client then either asks for the next page or discards the query:

```
["NQL-FETCH", <query_id>, <max_rows>]
["NQL-CLOSE", <query_id>]
```

`NQL-FETCH` is answered with another `NQL-ROWS`. A relay MAY discard a query that waits too long for its next `NQL-FETCH`; it then sends `CLOSED` with the `closed:` prefix.

Sending `NQL` with an id that is still open replaces the previous query. Closing the connection discards all of its queries.

### Refusals and errors

A relay that does not run a query, or stops running one, answers with `CLOSED`:

```
["CLOSED", <query_id>, "<prefix>: <human-readable reason>"]
```

| Prefix | Meaning |
|---|---|
| `invalid:` | the query breaks this NIP: its syntax, a typing rule, an unknown name or function, or a missing parameter |
| `auth-required:`, `restricted:` | the same access rules as `REQ` |
| `unsupported:` | the relay declines this valid query because of its cost; see [Cost](#cost) |
| `error:` | evaluation failed, e.g. an INTEGER `sum` overflow, or the relay failed internally |
| `closed:` | the relay discarded the query, e.g. after an idle timeout |

## Cost

A relay MAY decline any valid query it judges too expensive with `unsupported:`, for example:

- a query that reads a source without a condition on `id`/`event_id`, `pubkey`, `kind` or a tag `name`/`value`;
- a query that exceeds the relay's time or memory budget.

Relays SHOULD state the condition that would make the query acceptable. A declined query is not a wrong answer. A relay that answers MUST answer exactly as this NIP specifies.

Clients get the best service from queries that constrain every `events`/`tags` source by `kind`, `pubkey`, `id` or a tag `name`/`value`, and that use `ORDER BY created_at DESC` with a `LIMIT` for listings.

## Conformance

[`FF-conformance.json`](FF-conformance.json) holds this NIP's test vectors:
- `events`: a corpus of signed events;
- `cases`: queries run against that corpus.

To check a relay:
1. Start it empty, with every access restriction off.
2. Publish every event in `events`.
3. Run each case's `query` with its `params`.

Each case either expects an answer or a refusal:
- **An answer:** the relay MUST answer with exactly the case's `columns` (names and types) and `rows`.
  - Rows are compared in order only when `ordered` is `true`; otherwise as a multiset.
  - With `tolerance`, a REAL value may differ from the expected one by up to `tolerance` × max(1, |expected|).
- **A refusal:** the relay MUST send `CLOSED` whose reason starts with the case's `error` value followed by `:`.

A case with `sqlite_differs` states where SQLite on its own gives a different answer; see the implementation notes.

A relay conforms when every case passes, apart from queries it declines with `unsupported:` for cost.

## Advertising

Relays that implement this NIP include its number in `supported_nips` ([NIP-11](11.md)).

## Examples

Zap totals per zapped note, in sats:

```sql
SELECT target.value AS note, count(*) AS zaps, sum(CAST(amount.value AS INTEGER)) / 1000 AS sats
FROM tags AS target JOIN tags AS amount ON amount.event_id = target.event_id AND amount.name = 'amount'
WHERE target.kind = 9735 AND target.name = 'e'
GROUP BY target.value ORDER BY sats DESC LIMIT 20
```

Write relays named in [NIP-65](65.md) lists, by how many authors use them:

```sql
SELECT value AS relay, count(DISTINCT pubkey) AS authors
FROM tags
WHERE kind = 10002 AND name = 'r' AND (v2 IS NULL OR v2 = 'write')
GROUP BY value ORDER BY authors DESC LIMIT 100
```

Reactions per note of one author, including notes with none:

```sql
SELECT notes.id, count(reactions.event_id) AS total
FROM events AS notes LEFT JOIN tags AS reactions
  ON reactions.value = notes.id AND reactions.kind = 7 AND reactions.name = 'e'
WHERE notes.kind = 1 AND notes.pubkey = ?
GROUP BY notes.id ORDER BY total DESC
```

The `created_at` of each author's newest kind `0`:

```sql
SELECT pubkey, max(created_at) FROM events WHERE kind = 0 AND pubkey IN (?, ?, ?) GROUP BY pubkey
```

The geometric mean of zap amounts in a time window:

```sql
SELECT exp(avg(ln(CAST(value AS REAL)))) FROM tags
WHERE kind = 9735 AND name = 'amount' AND created_at BETWEEN :since AND :until
```

## Implementation notes

This section is non-normative.

**SQLite.** NQL is designed so that any conforming query runs unchanged on SQLite 3.35 or later, built with math functions and executed over views of an event store shaped like `events` and `tags`. SQLite and NQL agree on every rule above except these, which an implementation built on SQLite MUST account for:

| Rule | SQLite's behavior | What the implementation does |
|---|---|---|
| Static types and BOOLEAN | SQLite is dynamically typed and has no BOOLEAN | Type-check before executing, and map 0/1 to `false`/`true` in BOOLEAN result columns |
| INTEGER overflow in `+ - *` | SQLite switches to REAL | Emit `iif(typeof(r) = 'integer', r, NULL)` around the operation |
| `abs` of the smallest INTEGER | SQLite fails the statement | Emit `CASE WHEN x = -9223372036854775808 THEN NULL ELSE abs(x) END` |
| Mixed-type comparisons | SQLite allows them | Rejected by the type check |
| Numeric promotion in `CASE`, `coalesce`, `ifnull` and `UNION` | SQLite keeps an INTEGER value INTEGER even where the result is REAL, so `(CASE WHEN c THEN 1 ELSE 2.5 END) / 2` divides as integers | Wrap the INTEGER branches in `CAST(… AS REAL)` |
| Infinity in results | SQLite prints REAL infinity as `Inf` | Map it to the spelling defined under [Protocol](#protocol) |

**Stores that are not SQLite.** Such a store can still answer the language:
1. Push each source's constant conditions (`kind`, `pubkey`, `id`, tag `name`/`value`, time bounds) down as a NIP-01 filter.
2. Load the matching events into an in-memory SQLite shaped like `events` and `tags`.
3. Run the query there.

An engine that computes the same results natively is equally conforming.
