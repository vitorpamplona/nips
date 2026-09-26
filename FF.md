NIP-FF
======

Nostr Query Language (NQL)
--------------------------

`draft` `optional` `relay`

This NIP defines NQL, a small, strictly typed, read-only query language over the events a relay holds, and one command to run it. NQL looks like SQL's `SELECT` and means what SQL would mean, but it is closed: its only data are two fixed views of Nostr events, `events` and `tags`, and it has no user tables, no writes and no extensions.

A relay that supports this NIP MUST implement all of it, and every answer it gives MUST be the one this document specifies, so a client can send the same query to any supporting relay.

## Motivation

`REQ` returns events matching a filter, and `COUNT` ([NIP-45](45.md)) returns how many. Neither can group, aggregate, join an event to the events its tags point at, read a tag past its first value, or compute over tag values. Clients answer such questions by downloading every matching event: all reactions to count them per note, all kind `10002` lists to collect relay urls, all zap receipts to add up their amounts. NQL moves that work to the relay and returns only the answer.

## Overview

The client sends a query; the relay answers with typed columns and rows:

```
["NQL", "q1", "SELECT t1 AS relay, count(DISTINCT pubkey) AS authors FROM tags WHERE kind = 10002 AND t0 = 'r' GROUP BY t1 ORDER BY authors DESC LIMIT 2"]
["NQL", "q1", {"columns": [["relay", "TEXT"], ["authors", "INTEGER"]], "rows": [["wss://relay.damus.io", 91234], ["wss://nos.lol", 80112]], "truncated": false}]
```

More queries:

```sql
-- Zap totals per zapped note, in sats
SELECT target.t1 AS note, count(*) AS zaps, sum(CAST(amount.t1 AS INTEGER)) / 1000 AS sats
FROM tags AS target JOIN tags AS amount ON amount.event_id = target.event_id AND amount.t0 = 'amount'
WHERE target.kind = 9735 AND target.t0 = 'e'
GROUP BY target.t1 ORDER BY sats DESC LIMIT 20

-- Reactions per note of one author, including notes with none
SELECT notes.id, count(reactions.event_id) AS total
FROM events AS notes LEFT JOIN tags AS reactions
  ON reactions.t1 = notes.id AND reactions.kind = 7 AND reactions.t0 = 'e'
WHERE notes.kind = 1 AND notes.pubkey = ?
GROUP BY notes.id ORDER BY total DESC

-- The newest kind 0 of each of three authors
SELECT pubkey, max(created_at) AS newest FROM events WHERE kind = 0 AND pubkey IN (?, ?, ?) GROUP BY pubkey
```

Readers who know SQL need the [data model](#data-model), the [protocol](#protocol) and [NQL for SQL users](#nql-for-sql-users). The [language reference](#language-reference) defines each rule precisely for implementers, and the [conformance vectors](#conformance) test them.

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

| Column       | Type    | Value                                                   |
|--------------|---------|---------------------------------------------------------|
| `event_id`   | TEXT    | the `id` of the event holding the tag                   |
| `idx`        | INTEGER | the tag's 0-based position in that event's `tags` array |
| `t0`         | TEXT    | `tag[0]`, the tag's name                                |
| `t1`         | TEXT    | `tag[1]`, or NULL when absent                           |
| `t2`         | TEXT    | `tag[2]`, or NULL when absent                           |
| `t3`         | TEXT    | `tag[3]`, or NULL when absent                           |
| `t4`         | TEXT    | `tag[4]`, or NULL when absent                           |
| `created_at` | INTEGER | the holding event's `created_at`                        |
| `kind`       | INTEGER | the holding event's `kind`                              |
| `pubkey`     | TEXT    | the holding event's `pubkey`                            |

A tag that is an empty array has no row, and elements past `tag[4]` are not visible. A NIP-01 filter's `#x` condition is `t0 = 'x' AND t1 IN (…)`. `created_at`, `kind` and `pubkey` repeat the holding event's fields so that most tag questions need no join.

## Protocol

```
["NQL", <query_id>, <query>, <params>?]
```

- `query_id` is an arbitrary non-empty string, scoped like a `REQ` subscription id. A client MUST NOT reuse it until the query is answered.
- `query` is the query text.
- `params`, when present, is a JSON array holding the value of each `?` in the query, in order. It MUST have exactly as many elements as the query has `?`.

A parameter's type comes from its JSON value:

| JSON value | Type |
|---|---|
| a number without a fraction or exponent, within the 64-bit range | INTEGER |
| any other number | REAL |
| a string | TEXT |
| `true` / `false` | BOOLEAN |
| `null` | NULL, typed by its context as a `NULL` literal is |

The relay answers with one message:

```
["NQL", <query_id>, {"columns": [[<name>, <type>], ...], "rows": [[<value>, ...], ...], "truncated": <boolean>}]
```

- `type` is `"INTEGER"`, `"REAL"`, `"TEXT"` or `"BOOLEAN"`.
- Values are JSON numbers, strings, `true`/`false` or `null`. Clients MUST read INTEGER values as 64-bit integers; a REAL value is written with enough digits to read back as the same binary64 value.
- A relay MAY cap the number of rows it sends. It then sends the first rows of the result, in the query's `ORDER BY` order, and sets `truncated` to `true`. To read further, clients repeat the query with a condition past the last row received (for example `created_at < ?`) or with `OFFSET`.

A relay that does not answer a query sends `CLOSED` instead:

```
["CLOSED", <query_id>, "<prefix>: <human-readable reason>"]
```

| Prefix | Meaning |
|---|---|
| `invalid:` | the query breaks this NIP: its syntax, a typing rule, an unknown name or function, or the wrong number of parameters |
| `auth-required:`, `restricted:` | the same access rules as `REQ` |
| `unsupported:` | the relay declines this valid query because of its cost; see [Cost](#cost) |
| `error:` | evaluation failed (see [Errors](#errors)), or the relay failed internally |

## NQL for SQL users

NQL is the part of SQL's `SELECT` that every engine agrees on, plus these rules:

- **Types are strict.** INTEGER (64-bit), REAL (binary64), TEXT and BOOLEAN. INTEGER mixes with REAL in arithmetic and comparisons; every other mix needs `CAST`. `'1' + 1`, `'a' || 1` and `WHERE kind` are invalid.
- **Arithmetic never silently degrades.** Overflow, division by zero and math outside a function's domain fail the query. INTEGER division truncates: `7 / 2` is 3.
- **`CAST` from TEXT never fails:** text that is not exactly a number is NULL, so `CAST(t1 AS INTEGER)` skips malformed tag values.
- **Text is compared by code point**, and `LIKE` is case-sensitive with no escape character.
- **NULLs sort last** in ascending order and first in descending order.
- **Every computed result column needs an `AS` name**, and names are reported in lower case.
- **Only `JOIN … ON` and `LEFT JOIN … ON`** combine sources. There is no `UNION`, `WITH`, window function or `CASE x WHEN`, and parameters are only `?`.
- **About 25 functions** exist; see [Functions](#functions).

## Language reference

### Types and values

- **INTEGER**: a signed 64-bit integer.
- **REAL**: a finite IEEE 754 binary64 number. NQL never produces ±Infinity or NaN.
- **TEXT**: a sequence of Unicode code points. Lengths, positions and ordering count code points.
- **BOOLEAN**: `TRUE` or `FALSE`.

`NULL` is the absence of a value and belongs to every type. INTEGER and REAL are the *numeric* types.

Every expression has one static type, fixed before execution by the rules below. A query that breaks a typing rule is invalid, and the relay MUST refuse it before evaluating anything.

The only implicit conversion is **numeric promotion**: where an arithmetic or comparison operator has one INTEGER and one REAL operand, the INTEGER operand is converted to REAL. Nothing else converts without `CAST`: the branches of a `CASE`, the arguments of `coalesce` and `nullif` MUST have the same type.

A `NULL` literal, or a parameter whose value is `null`, takes the type of its context: the other operand of an operator, the other elements of an `IN` list, or the other branches or arguments of `CASE`, `coalesce` or `nullif`. Where nothing fixes it, as in `SELECT NULL AS x`, its type is TEXT.

### Syntax

A query is exactly one `query`. Keywords and names are case-insensitive. Whitespace separates tokens. There are no comments and no trailing `;`.

```
query         = "SELECT" [ "DISTINCT" ] results
                [ "FROM" source { [ "LEFT" ] "JOIN" source "ON" expr } ]
                [ "WHERE" expr ]
                [ "GROUP" "BY" expr { "," expr } [ "HAVING" expr ] ]
                [ "ORDER" "BY" order-term { "," order-term } ]
                [ "LIMIT" count [ "OFFSET" count ] ]
results       = "*" | expr [ "AS" name ] { "," expr [ "AS" name ] }
source        = ( "events" | "tags" ) [ "AS" name ] | "(" query ")" "AS" name
order-term    = expr [ "ASC" | "DESC" ]
count         = integer | "?"

expr          = and-expr { "OR" and-expr }
and-expr      = not-expr { "AND" not-expr }
not-expr      = "NOT" not-expr | predicate
predicate     = sum [ ( "=" | "<>" | "<" | "<=" | ">" | ">=" ) sum
                    | "IS" [ "NOT" ] "NULL"
                    | [ "NOT" ] "IN" "(" ( query | expr { "," expr } ) ")"
                    | [ "NOT" ] "LIKE" sum
                    | [ "NOT" ] "BETWEEN" sum "AND" sum ]
sum           = product { ( "+" | "-" | "||" ) product }
product       = unary { ( "*" | "/" | "%" ) unary }
unary         = "-" unary | primary
primary       = literal | "?" | [ name "." ] name | call | cast | case
              | "(" expr ")" | "(" query ")" | "EXISTS" "(" query ")"
call          = name "(" [ "*" | [ "DISTINCT" ] expr { "," expr } ] ")"
cast          = "CAST" "(" expr "AS" ( "INTEGER" | "REAL" | "TEXT" ) ")"
case          = "CASE" "WHEN" expr "THEN" expr { "WHEN" expr "THEN" expr } [ "ELSE" expr ] "END"
literal       = integer | real | string | "NULL" | "TRUE" | "FALSE"
```

Binary operators are left-associative. A `predicate` holds at most one comparison, so `a < b = c` is invalid; write `(a < b) = c`.

Lexical rules:

- **Names** match `[A-Za-z_][A-Za-z0-9_]*` and MUST NOT be one of the keywords `AND AS ASC BETWEEN BY CASE CAST DESC DISTINCT ELSE END EXISTS FALSE FROM GROUP HAVING IN INTEGER IS JOIN LEFT LIKE LIMIT NOT NULL OFFSET ON OR ORDER REAL SELECT TEXT THEN TRUE WHEN WHERE`.
- **Integers** are decimal digits and MUST fit in a signed 64-bit integer. The one exception is `9223372036854775808`, which is valid only directly after a unary `-`.
- **Reals** are `digits "." digits [exponent]` or `digits exponent`, where `exponent = ("e" | "E") ["+" | "-"] digits`. The value is rounded to the nearest binary64. A real that rounds to infinity, or to zero although it is not zero, is invalid.
- **Strings** are enclosed in single quotes, and `''` stands for one quote. There are no other escapes.

### Names

Each source has a name: its alias, or `events`/`tags` when it has none. Names of sources joined in one `FROM` MUST differ.

A qualified column `s.c` refers to column `c` of source `s`. An unqualified column `c` refers to the one source in the innermost enclosing `FROM` that has a column `c`. If none has it, the next enclosing query is searched (a correlated reference). If two sources at the same level have it, or none anywhere does, the query is invalid.

**Result columns.** `*` stands for every column of every source, in `FROM` order and then in the order of the tables above. A result column is named by its alias, or, if it is a bare column, by that column's name. Any other result column without an alias makes the query invalid. Names are reported in lower case, and those of the outermost query and of a subquery in `FROM` MUST be unique. A subquery in `FROM` exposes its result columns under those names.

### Evaluation

A query is evaluated in this order:

1. `FROM` combines its sources left to right. `JOIN … ON` keeps each pair of rows for which `ON` is TRUE. `LEFT JOIN … ON` also keeps each left row that matched nothing, paired with NULLs. Without `FROM`, there is exactly one row.
2. `WHERE` keeps the rows for which it is TRUE.
3. The rows are grouped (see [Grouping and aggregates](#grouping-and-aggregates)), and `HAVING` keeps the groups for which it is TRUE.
4. The result columns are computed, and `DISTINCT` removes duplicate rows.
5. `ORDER BY` sorts the rows, and `LIMIT`/`OFFSET` keep a slice of them.

`ON`, `WHERE`, `HAVING`, `CASE` conditions and the operands of `NOT`, `AND` and `OR` MUST be BOOLEAN.

Two rows are **duplicates** when each pair of values is equal or both NULL. The same holds for the values `GROUP BY` groups on.

### Operators

*N* is either numeric type.

| Operator | Operands | Result |
|---|---|---|
| `+` `-` `*` | N, N | INTEGER if both are INTEGER, else REAL |
| `/` | N, N | INTEGER if both are INTEGER (truncated toward zero), else REAL |
| `%` | INTEGER, INTEGER | INTEGER, with the sign of the left operand |
| unary `-` | N | the same type |
| `\|\|` | TEXT, TEXT | TEXT, the concatenation |
| `=` `<>` | two numbers, two TEXT or two BOOLEAN | BOOLEAN |
| `<` `<=` `>` `>=`, `BETWEEN` | numbers, or all TEXT | BOOLEAN |
| `IN` | as `=` between `x` and each element | BOOLEAN |
| `LIKE` | TEXT, TEXT | BOOLEAN |
| `IS NULL`, `IS NOT NULL` | any | BOOLEAN, never NULL |
| `NOT`, `AND`, `OR` | BOOLEAN | BOOLEAN |

- Every operator except `IS [NOT] NULL`, `AND` and `OR` yields NULL when an operand is NULL.
- `AND` and `OR` use three-valued logic: `FALSE AND NULL` is FALSE, `TRUE OR NULL` is TRUE, and every other combination with NULL is NULL.
- TEXT compares code point by code point, and a string sorts before any longer string that starts with it. Numbers compare by value.
- `x IN (…)` is TRUE if `x` equals an element, NULL if none does and `x` or an element is NULL, and FALSE otherwise. `IN (query)` takes the elements from the query's single column. `NOT IN` negates `IN`, leaving NULL as NULL.
- `x BETWEEN a AND b` is `x >= a AND x <= b`.
- `x LIKE p` is TRUE if `x` matches all of pattern `p`, where `%` matches any sequence of code points, `_` matches exactly one, and every other code point matches only itself. Matching is case-sensitive.

### Errors

These conditions fail the whole query with `error:`:

- an INTEGER `+`, `-`, `*`, `/`, unary `-`, `abs` or `sum` whose exact result does not fit in 64 bits;
- `/` or `%` by zero, INTEGER or REAL;
- a REAL result that is too large for binary64 (overflow), or that is not zero but rounds to zero (underflow);
- a numeric function applied outside its domain (see [Numeric](#numeric));
- a scalar subquery that yields more than one row.

An expression is evaluated only for rows that reach it: result columns, `HAVING` and `ORDER BY` only for rows and groups that `WHERE` and `HAVING` kept, and a `CASE` result only when its condition selects it. The order in which the operands of other operators are evaluated is unspecified, so guard a computation that may fail with `CASE`, not with `AND`.

### `CAST`

| From \ To | INTEGER | REAL | TEXT |
|---|---|---|---|
| INTEGER | itself | the nearest REAL | decimal digits, with a leading `-` if negative |
| REAL | truncated toward zero; NULL if that does not fit | itself | *invalid* |
| TEXT | see below | see below | itself |
| BOOLEAN | `TRUE` → 1, `FALSE` → 0 | *invalid* | *invalid* |

`CAST` never fails, and NULL casts to NULL.

- **TEXT to INTEGER:** if the whole text is an optional `+` or `-` followed by one or more decimal digits, and its value fits in 64 bits, the result is that value. Otherwise it is NULL: `'12abc'`, `' 42'`, `'3.9'`, `'1e3'` and `''` all give NULL.
- **TEXT to REAL:** if the whole text is an optional `+` or `-` followed by an `integer` or `real` literal, and that literal would be valid, the result is its value. Otherwise it is NULL, which also covers `'.5'`, `'Infinity'` and `'1e999'`.

### `CASE`

`CASE WHEN c1 THEN r1 … [ELSE e] END` yields the first `r` whose `c` is TRUE. If there is none, it yields `e`, or NULL when there is no `ELSE`. All `r` and `e` MUST have the same type, which is the type of the `CASE`.

### Subqueries

- A subquery used as a value MUST have one column; its type is that column's type. It yields that column's value in its only row, or NULL when it has no row. More than one row is an [error](#errors).
- `EXISTS (query)` is TRUE if the query yields at least one row, and FALSE otherwise.
- A subquery may refer to the columns of the queries that enclose it.

### Grouping and aggregates

A query is **grouped** if it has `GROUP BY`, or if an aggregate appears in its result columns, `HAVING` or `ORDER BY`.

- With `GROUP BY`, rows are grouped by the values of its terms. Without it, all rows form one group, which exists even when there are no rows.
- A `GROUP BY` term is an expression over the sources, or the bare name of a result column's alias.
- Outside aggregate arguments, the result columns, `HAVING` and `ORDER BY` of a grouped query may use a source column only as part of an expression that is written the same as a `GROUP BY` term (ignoring case and whitespace), or as a column that is a `GROUP BY` term itself.
- Aggregates appear only in result columns, `HAVING` and `ORDER BY`, and never inside another aggregate. `HAVING` requires `GROUP BY`.

| Aggregate | Argument | Result | Value over the group's non-NULL arguments |
|---|---|---|---|
| `count(*)` | none | INTEGER | the number of rows in the group |
| `count(x)` | any | INTEGER | how many there are |
| `count(DISTINCT x)` | any | INTEGER | how many distinct values there are |
| `sum(x)` | N | the type of `x` | their sum, or NULL if there are none |
| `avg(x)` | N | REAL | their mean, or NULL if there are none |
| `min(x)`, `max(x)` | N or TEXT | the type of `x` | the least or greatest, or NULL if there are none |

An INTEGER `sum` is exact, and only its final value must fit in 64 bits. A REAL `sum` or `avg` may add values in any order, so relays may differ in its last bits.

### `ORDER BY`, `LIMIT`, `OFFSET`

- An `order-term` is the bare name of a result column, or an expression over the sources. After `DISTINCT`, only result columns may be used.
- A bare name that is both a result column's alias and a source column is invalid in `ORDER BY` and `GROUP BY`, unless the alias names that same column.
- Terms MUST be numeric or TEXT. `ASC` is the default. NULLs sort after every value, so they come last in ascending order and first in descending order.
- The order of rows that compare equal on every term is unspecified, as is the order of rows without `ORDER BY`.
- `LIMIT` and `OFFSET` take a non-negative INTEGER: a literal, or a `?` whose value is one.

### Functions

These are all the functions. Every function yields NULL when an argument is NULL, except `coalesce` and `nullif`. An argument of the wrong type makes the query invalid.

#### Text

| Function | Result | Value |
|---|---|---|
| `length(TEXT)` | INTEGER | the number of code points |
| `lower(TEXT)`, `upper(TEXT)` | TEXT | ASCII letters converted; every other code point unchanged |
| `trim(x TEXT [, chars TEXT])`, `ltrim(…)`, `rtrim(…)` | TEXT | `x` without the code points in `chars` (a space when omitted) at both ends, the start or the end |
| `replace(x TEXT, from TEXT, to TEXT)` | TEXT | `x` with each non-overlapping `from`, scanned left to right, replaced by `to`; `x` itself when `from` is empty |
| `instr(x TEXT, y TEXT)` | INTEGER | the 1-based position of the first `y` in `x`; 0 if there is none, and 1 if `y` is empty |
| `substr(x TEXT, start INTEGER [, len INTEGER])` | TEXT | the code points of `x` at positions `start` through `start + len - 1`, counting from 1, that exist; without `len`, through the end. NULL if `len` is negative |

So `substr('abcdef', 2, 3)` is `'bcd'`, `substr('abcdef', 0, 2)` is `'a'`, and `substr('abcdef', 10)` is `''`.

#### Conditional

| Function | Result | Value |
|---|---|---|
| `coalesce(x, y, …)` (2 or more arguments) | their type | the first non-NULL argument, or NULL |
| `nullif(x, y)` | their type | NULL if `x = y` is TRUE, otherwise `x` |

#### Numeric

| Function | Result | Value | Error when |
|---|---|---|---|
| `abs(N)` | same type | the absolute value | the INTEGER result does not fit |
| `round(N)` | REAL | the nearest integer, ties to even: `round(2.5)` = 2.0 | |
| `ceil(N)`, `floor(N)`, `trunc(N)` | REAL | toward +∞, toward −∞, toward zero | |
| `sqrt(N)` | REAL | the square root | the argument is negative |
| `exp(N)` | REAL | *e*^x | overflow or underflow |
| `ln(N)`, `log10(N)` | REAL | the natural and base-10 logarithm | the argument is zero or negative |
| `pow(x N, y N)` | REAL | x^y | `x` is zero and `y` negative, `x` is negative and `y` not an integer, or overflow or underflow |

`sqrt` is correctly rounded. The others MUST be within a relative error of 10⁻¹² and exact where the exact result is representable, as in `pow(2, 10)`, `exp(0)`, `ln(1)` and `log10(1000)`.

## Cost

A relay MAY decline any valid query it judges too expensive with `unsupported:`, for example:

- a query that reads a source without a condition on `id`/`event_id`, `pubkey`, `kind` or a tag's `t0`/`t1`;
- a query that exceeds the relay's time or memory budget.

Relays SHOULD state the condition that would make the query acceptable. A declined query is not a wrong answer. A relay that answers MUST answer exactly as this NIP specifies.

Clients get the best service from queries that constrain every source by `kind`, `pubkey`, `id` or a tag's `t0`/`t1`, and from listings that use `ORDER BY created_at DESC` with a `LIMIT`.

## Conformance

[`FF-conformance.json`](FF-conformance.json) holds this NIP's test vectors:
- `events`: a corpus of signed events;
- `cases`: queries to run against that corpus.

To check a relay:
1. Start it empty, with every access restriction off.
2. Publish every event in `events`.
3. Run each case's `query` with its `params`.

Each case either expects an answer or a refusal:
- **An answer:** the relay MUST answer with exactly the case's `columns` (names and types) and `rows`.
  - Rows are compared in order only when `ordered` is `true`; otherwise as a multiset.
  - With `tolerance`, a REAL value may differ from the expected one by up to `tolerance` × max(1, |expected|).
- **A refusal:** the relay MUST send `CLOSED` whose reason starts with the case's `error` value followed by `:`.

A relay conforms when every case passes, apart from queries it declines with `unsupported:` for cost.

## Advertising

Relays that implement this NIP include its number in `supported_nips` ([NIP-11](11.md)). A relay that caps result rows SHOULD publish the cap as `max_nql_rows` in its NIP-11 `limitation` object.

## Implementation notes

This section is non-normative.

In every design, parse and type-check the query yourself, and never hand the client's text to a database. The type check also gives each result column its type.

### PostgreSQL

Expose the relay's events as `events` and `tags` relations with `int8` and `text` columns under the `C` collation (for example, create the database with `LOCALE 'C'`). Then translate the checked query into SQL:

- Emit every expression fully parenthesized, INTEGER literals as `n::int8`, REAL literals as `x::float8`, and the types INTEGER and REAL as `int8` and `float8`.
- Bind the parameters with their types.
- Quote each alias as its lower-case name.
- Run the query in a read-only transaction with a `statement_timeout`.

PostgreSQL then applies these rules on its own: NULL ordering, `round`, errors for overflow, underflow, division by zero, domain and too many subquery rows, INTEGER division, and grouping. What remains is a handful of rewrites:

| NQL | PostgreSQL |
|---|---|
| `CAST(x AS INTEGER)`, `x` TEXT | `CASE WHEN x ~ '^[+-]?[0-9]+$' AND pg_input_is_valid(x, 'int8') THEN x::int8 END` |
| `CAST(x AS REAL)`, `x` TEXT | `CASE WHEN x ~ '^[+-]?[0-9]+(\.[0-9]+)?([eE][+-]?[0-9]+)?$' AND pg_input_is_valid(x, 'float8') THEN x::float8 END` |
| `CAST(x AS INTEGER)`, `x` REAL | `CASE WHEN x >= -9223372036854775808::float8 AND x < 9223372036854775808::float8 THEN trunc(x)::int8 END` |
| `CAST(x AS INTEGER)`, `x` BOOLEAN | `x::int4::int8` |
| `x LIKE p` | `x LIKE p ESCAPE ''` |
| `instr(x, y)` | `strpos(x, y)` |
| `substr(x, s, n)` | `CASE WHEN n >= 0 THEN substr(x, s, n) END`, with `s` and `n` clamped into the `int4` range |
| `sum(x)`, `x` INTEGER | `sum(x)::int8` |
| `avg(x)` | `avg(x)::float8` |
| an untyped `NULL` | `NULL::text` |

### Key-value stores (LMDB and similar)

A relay whose events sit in LMDB, RocksDB or a similar store keeps the indexes it already has for `REQ` and evaluates the query itself:

1. Read each `events` or `tags` source through the index that its constant conditions (`id`/`event_id`, `pubkey`, `kind`, `t0` with `t1`, and `created_at` bounds) select, exactly as it would read a NIP-01 filter. Decline with `unsupported:` when no index applies.
2. Evaluate everything else in a small interpreter over typed values:
   - joins by index lookups on the join key (an event by `id`, tag rows by `t0`/`t1` or `event_id`);
   - grouping in a hash map;
   - ordering by a sort, or a bounded heap when there is a `LIMIT`.

The rules that most often go wrong when written by hand:
- **Code-point order.** It is not UTF-16 order: in Java, Kotlin or JavaScript, compare code points rather than `String.compareTo`, or compare UTF-8 bytes.
- **64-bit INTEGER arithmetic** must detect overflow (`Math.addExact` and similar), and an INTEGER `sum` must accumulate exactly (in 128 bits or a big integer).
- **REAL results** must be checked for overflow and underflow.
- **`round`** rounds ties to even (`Math.rint`).
- **`CAST` from TEXT** matches the whole text, not a prefix.
- **`LIKE` and `substr`** count code points.

The conformance vectors cover each of these.

### SQLite

The same approach works with SQLite's `events`/`tags` views, but SQLite differs from NQL on more rules than PostgreSQL does:
- `LIKE` is case-insensitive unless `PRAGMA case_sensitive_like` is on.
- NULLs sort first in ascending order; emit `NULLS LAST` and `NULLS FIRST`.
- Overflow becomes REAL, and division by zero and domain errors become NULL.
- `CAST` reads a numeric prefix.
- `round` rounds ties away from zero, and `ceil`, `floor` and `trunc` keep INTEGER arguments INTEGER.
- A scalar subquery takes its first row.
- Result names keep their case.

An implementation on SQLite registers its own functions for arithmetic, `CAST` and the math functions, emitted in place of SQLite's.
