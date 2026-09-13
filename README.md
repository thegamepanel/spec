# The Game Panel Spec

The design record and current specification for The Game Panel, managed with
[ArchDoc](https://github.com/ollieread/archdoc).

- **`rfc/`** proposes how something should work, and has a page in the spec once built.
- **`adr/`** records a decision that constrains designs across the spec.
- **`spec/`** describes how the project works right now, and is always current.
- **`ref/`** holds research and background material, never normative.

[`INDEX.md`](INDEX.md) lists every document and how they relate; it is generated,
so do not edit it by hand.

Read [`PROCESS.md`](PROCESS.md) before adding anything.

## Working on this repository

Install [ArchDoc](https://github.com/ollieread/archdoc), then enable the hooks once
per clone:

```
git config core.hooksPath .githooks
```

`pre-commit` runs `archdoc lint`. `pre-push` runs it again and checks `INDEX.md` is
current, because pushing is what freezes a document that has reached a terminal
status: after that, nothing may edit it to correct a mistake.

If `INDEX.md` is reported out of date, run `archdoc index` and commit the result.

CI runs the same two checks, so the hooks are early warning rather than the
authority. `git push --no-verify` skips them when you need it.
