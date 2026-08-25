# json

JSON, read and written, for [sysl](https://github.com/sysl-lang/sysl).

A document reads into an ordinary sysl value that owns itself — an array is a `Buf[Json]` and an
object a `Buf[Member]`, with no reference anywhere — so every walk over one is a plain `match` and
nothing has to be dereferenced first. It writes as well as reads, because a program that changes one
member of a configuration file and writes it back needs both halves and neither alone.

The reading is built on [`parsing`](https://github.com/sysl-lang/parsing), which is where the cursor,
the spans, the escape decoding and the diagnostics come from. What is written here is the grammar and
the three places JSON is *narrower* than the readers underneath it.

```
sh/sysl/json/
    json.sysl       the value, its accessors, and what makes two of them equal
    read.sysl       a document into a value, or the first thing wrong with it
    write.sysl      a value back out, compactly or laid out for a person
    build.sysl      an array and an object assembled a piece at a time
    tests.sysl      what all of it claims, run by `sysl test .`
package.hocon       who this package is, and what it needs of the machine
```

The module is **`sh.sysl.json`**, and the three directories are that name: a dotted module name
mirrors its path from the library root. The prefix is the reverse-DNS of `sysl.sh`, so that a package
claims a name nobody else will mint rather than the top-level word `json`.

## Using it

Name it in your project's `package.hocon` and `sysl build` fetches it:

```hocon
dependencies {
  json { git = "github.com/sysl-lang/json", version = "0.1.0" }
}
```

Naming this one is enough to reach `sh.sysl.parsing` as well, since imports are transitive — which a
caller needs, because `parse` takes a `Source` and answers with a `Diagnostic`.

It needs sysl 0.0.79 or newer, for the reason `package.hocon` gives.

## Reading

```sysl
import sh.sysl.json.*
import sh.sysl.parsing.source_of

main()
    val src = source_of("config.json", "{\"port\": 8080, \"tls\": true, \"tags\": [\"a\", \"b\"]}")

    parse(src) match
        Err(d) -> print(d.render(src))
        Ok(doc) ->
            print(doc.get("port").unwrap().as_int())       // Some(8080)
            print(doc.get("tags").unwrap().len())          // 2
            print(doc.get("missing"))                      // None
```

The caller builds the `Source` and keeps it, which is why `parse` takes one rather than a string: a
diagnostic names a place by byte offset, and turning that into `config.json:3:8` with the line quoted
underneath needs the input and its line table.

```
error: expected a value
 --> config.json:3:8
  |
3 |   "b": @
  |        ^ found `@`
  |
  = note: a value is a number, a string, `true`, `false`, `null`, an array or an object
```

## The value

```sysl
enum Json
    Null
    Bool(b: bool)
    Int(n: long)
    Real(x: real)
    Str(s: string)
    Arr(items: Buf[Json])
    Obj(members: Buf[Member])
```

**Numbers are two variants rather than one.** JSON's grammar has a single numeric production and says
nothing about storage, so a reader has to choose and either choice alone is wrong for somebody:
carrying everything as a `real` loses the exact value of an integer past 2⁵³, which is where an
identifier or a millisecond timestamp lives, and carrying everything as a `long` cannot represent
`0.5`. A number with no fraction and no exponent is an `Int`, everything else is a `Real`, and a
caller that does not care asks `as_real`.

**A document keeps the order it was written in.** Members are a sequence rather than a map, so
`{"b":1,"a":2}` renders back the way it arrived, `get` is a linear scan, and a repeated name keeps
both copies with the first winning. That is the only representation that can round-trip, which is
what a reader and a writer in one package are for.

`is_null`, `is_bool`, `is_number`, `is_str`, `is_array` and `is_object` ask what a value is;
`as_bool`, `as_int`, `as_real` and `as_str` take it out. The `as_*` family never converts: a string
reading `"true"` is a string, and `Str("1").as_int()` is `None`. `get(name)`, `at(index)`,
`member(index)` and `len()` reach inside, and each answers `None` — or zero — for a shape that has
none, so an accessor chain is safe to write against a document nobody has checked yet.

## Writing

The compact rendering is the `Display`, so `str(j)`, `print(j)` and `s"$j"` all produce JSON:

```sysl
print(doc)                    // {"port":8080,"tls":true,"tags":["a","b"]}
print(pretty(doc, 2))
```

```json
{
  "port": 8080,
  "tls": true,
  "tags": [
    "a",
    "b"
  ]
}
```

**A float is written as the shortest text that reads back as the same value.** `str(x)` is `%g` at
six significant digits, which would write a third as `0.333333` — a different number, in a format
whose whole job is carrying the value somewhere else. A float with no fraction keeps a `.0`, so that
`Real(1.0)` does not read back as `Int(1)`; a NaN or an infinity is written `null`, which is what
JavaScript's own `JSON.stringify` does and the only thing JSON's grammar leaves available.

## Building

The variants are already constructors — `Str("sysl")`, `Int(3)`, `Null` — so nothing is needed for a
scalar. The two shapes that grow get builders:

```sysl
var o = object_builder()

o.put_str("name", "sysl")
o.put_int("port", 8080)
o.put_bool("tls", true)
o.put("tags", array_of_str(["systems", "small"]))

print(o.finish())
```

`put`/`push` take a `Json`; the `put_*`/`push_*` families take the value and leave the representation
to the builder, which is the whole ergonomic argument — `o.put_int("port", port)` says what a line is
about where `o.put("port", Int(long(port)))` says how it is stored.

## Where JSON is narrower than the readers underneath

The toolkit's readers know the C family's rules, and JSON has a subset of them. Every narrowing is
done at the call rather than by a flag on the reader, so that a second grammar can disagree:

* **Escapes.** `read_escape` knows fifteen and JSON has eight, so the byte after the backslash is
  checked first. `\u{1f600}` is sysl's spelling of a code point and JSON writes it `😀`,
  which is checked all four digits at a time so the message is about the escape.
* **Numbers.** `read_number` reads underscores, radix prefixes and leading zeros, and JSON has none
  of them, so the text it consumed is examined afterwards.
* **Whitespace.** JSON names four bytes and C's `isspace` takes six — a form feed and a vertical tab
  are not whitespace here.

## What is deliberately not here

**No streaming reader.** The whole input has to be one value and trailing text is refused, which is
what makes this a reader for documents rather than for a stream of them.

**No map.** Lookup is a linear scan over the members in the order they were written. A document with
a thousand members in it wants a different structure, and building one from what this hands back is
five lines that this package should not choose for you.

**No schema, no derivation, no reflection.** A document becomes a `Json` and what it means is the
program's business.

## Nesting

A recursive-descent reader has the machine's stack for a depth limit, and hostile input knows it:
`[[[[[…` is one byte per level. So the limit is the reader's, it is a diagnostic like any other, and
`max_depth` is 128 — far past anything a document written by a person or a program has.

## Licence

ISC.
