# Rowshape Fixture Spec

**A portable, committable description of what a production database looks like —
structure plus statistical shape, containing no production row values.**

A fixture exists so a migration can be tested against realistic data by someone
with no access to the real data. That someone is increasingly not a person.

Two operations define the format:

- **Profile:** production database → `rowshape.yaml`
- **Hydrate:** `rowshape.yaml` + seed → a local database with realistic rows

The spec is **[RFC-0001](RFC-0001.md)**. Format version `1`. MIT.

```yaml
rowshape_fixture: "1"
meta:
  engine: { name: postgres, version: "16" }
  privacy: standard
tables:
  public.users:
    rows: { value: 5000000, confidence: exact }
    columns:
      email:
        type: text
        nullable: true
        null_fraction: { value: 0.03, confidence: estimated }
        unique: { value: true, confidence: exact, via: constraint }
```

Every fact carries a **confidence**, and that is the point. A fixture never
asserts more than it measured, so a tool reading one can decline to certify what
it cannot prove rather than guessing.

## Why this is a separate repo

The format is deliberately not owned by any one tool. Anyone can emit a fixture,
and a fixture emitted by anyone should hydrate anywhere. Publishing the spec, the
JSON Schema and the conformance suite apart from the CLI is what makes that a
position rather than an aspiration.

## What is here

| Path | |
|---|---|
| [`RFC-0001.md`](RFC-0001.md) | The specification |
| [`schema/rowshape.schema.json`](schema/rowshape.schema.json) | Machine-readable JSON Schema |
| [`conformance/`](conformance/) | Valid and invalid fixtures an implementation must agree about |

The conformance **fixtures** are language-neutral, which is the half an
independent implementation needs. The Go runner that executes them lives in the
[CLI repo](https://github.com/rowshape/rowshape) alongside the implementation it
tests.

## Compatibility rules (RFC §12)

These are the load-bearing part of the format, and they are short on purpose:

- **Unknown fields MUST be ignored by readers, not rejected.** A newer emitter
  must not break an older reader.
- **Vendor extensions use an `x_` prefix** and MUST NOT change hydration
  semantics. An extension that alters what gets hydrated is a fork, not an
  extension.
- **`rowshape_fixture: "1"` is the major version.** Additive fields ship without
  a bump. Removing or reinterpreting a field requires `"2"`.
- **A reader encountering an unknown major version MUST refuse** rather than
  best-effort.

That last rule is the one worth arguing about, so here is the argument: silent
partial understanding is how you get a PASS that means nothing. A tool that reads
half of a version-2 fixture and reports success has not been lenient, it has been
wrong — and the whole value of the format is that its answer can be trusted by
something that cannot check the work.

## Two constraints that are not stylistic

- **`range` is never emitted for a text or bytea column.** Its min and max would
  be real values from the database, and a fixture contains none.
- **`unique` is exact or absent, never sampled.** A sample cannot establish
  uniqueness, and treating it as if it could is how a unique index gets certified
  and then fails to build.

## Implementations

- **[rowshape](https://github.com/rowshape/rowshape)** — the reference emitter,
  hydrator and validator. `rowshape pull` writes a fixture; `rowshape hydrate`
  reconstructs a database from one.
- **[MCPFold](https://github.com/dj-pearson/MCPFold)** — one canonical MCP config
  folded out to every client, loading only the tools each agent needs. Built to
  the same discipline of a small, checkable contract between tools that do not
  share a runtime.

Emitting fixtures from another database, language or toolchain is the point.
Run the conformance suite against your implementation and open an issue if the
spec is ambiguous — an ambiguity here is a defect, because two readers
disagreeing about the same file defeats the purpose.

## License

MIT. The spec and the CLI are both MIT so neither can be a gate on the other.
