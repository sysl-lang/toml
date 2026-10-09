# toml

TOML v1.0.0, read into a sysl value.

```sysl
import sh.sysl.toml.parse
import sh.sysl.parsing.source_of

val src = source_of("config.toml", text)

parse(src) match
    Ok(v) -> print(v.get("server").unwrap().get("port").unwrap().as_int().unwrap())
    Err(d) -> print(d.render(src))
```

```
dependencies {
  toml { git = "github.com/sysl-lang/toml", version = "0.1.2" }
}
```

The coordinate names the package, `toml`; the module a program imports is `sh.sysl.toml`. Naming the
coordinate alone reaches `sh.sysl.parsing` too, since imports are transitive — which a caller needs,
because `parse` takes a `Source` and answers with a `Diagnostic`.

## What it is

A reader. `parse` takes the whole document and answers with a `Value`, which is always a table, or
with the first thing wrong with it. There is no writer yet.

The tree owns itself: a table is a `Buf[Entry]` and an array a `Buf[Value]`, with no reference
anywhere, so every walk over a document is an ordinary `match` and nothing has to be dereferenced
first. That is `sysl-lang/json`'s shape, deliberately — the two formats are read for the same reason,
and a program that handles both should not have to learn two idioms.

```sysl
enum Value
    Str(s: string)
    Int(n: long)
    Float(x: real)
    Bool(b: bool)
    OffsetDateTime(dt: LocalDateTime, offset: Offset)
    DateTime(dt: LocalDateTime)
    Date(d: LocalDate)
    Time(t: LocalTime)
    Arr(items: Buf[Value])
    Table(entries: Buf[Entry])
```

Entries keep the order they were written in, so `get` is a linear scan. That is the right trade for
the sizes TOML is used at — it is a configuration format, and nobody writes a thousand keys into one
table.

## The dates are `sysl.time`'s, and there are four of them

TOML has four date and time types and this keeps them apart, because collapsing them would answer a
question the document did not ask: an offset date-time knows where on the planet it was written, and
a local date-time is not a point on the timeline until somebody supplies a zone.

`OffsetDateTime` keeps the offset it was written with rather than folding to an `Instant`.
`1979-05-27T00:32:00-07:00` and `1979-05-27T07:32:00Z` are the same instant and are not the same
document. `to_instant` is one call away for the other question.

## v1.0.0, and not the draft

This reads **TOML v1.0.0**, the released specification, and nothing else. TOML 1.1 has been a draft
for years — `toml-test` ships it as a separate list of files for that reason — and a package that
pinned to it would be promising something the specification has not settled. If 1.1 is released the
delta is small: optional seconds, newlines and trailing commas in inline tables, and the `\e` and
`\xHH` escapes, which `sh.sysl.parsing` already reads.

## Conformance

Every document in `toml-test`'s TOML 1.0.0 corpus, on the tag this was released at:

| | |
|---|---|
| valid documents accepted | **210 / 210** |
| …parsing to the expected value, compared field by field | **210 / 210** |
| invalid documents refused | **501 / 501** |

The value comparison is the half worth stating separately. Accepting a document proves the grammar
ran; only comparing what came out proves it read. Twelve of the corpus's cases disagreed with this
reader on the first run, and every one was a real defect — a lone carriage return taken as a line
ending, `+0xff` accepted, `0_0` read as zero, a dotted key adding to a table that had been written as
a value. All twelve are pinned as tests in `sh/sysl/toml/tests.sysl` under the names `toml-test` gives
them.

## What it needs

`heap = true`, and nothing else — no `os` and no `posix`. A `Source` is made over text the caller
already has, so where the document came from is the program's business. A document's size is not
known until it has been read, which is what the heap is for.

## Building it

```
sysl test .
```
