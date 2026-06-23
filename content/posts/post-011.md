---
title: "Annotating ClickHouse-connect layer by layer"
slug: "clickhouse-connect-dbapi-annotations"
date: 2026-06-23
draft: false
tags: ["python", "mypy", "clickhouse", "open-source", "type-annotations"]
summary: "Five PRs to annotate clickhouse-connect for issue #692: what bugs the type annotation surfaced, what Copilot caught on review, and the design of the SQLAlchemy layer."
---

[clickhouse-connect](https://github.com/ClickHouse/clickhouse-connect) is the official Python client for ClickHouse. A while back, the project opened [issue #692](https://github.com/ClickHouse/clickhouse-connect/issues/692) to track the work of adding mypy annotations across the entire codebase, module by module. I contributed to that effort by annotating several layers — `dbapi`, driver utilities, driver clients, and the SQLAlchemy integration — across five PRs. This post covers what was interesting along the way: where mypy failed to meet expectations, what Copilot caught on review, and the dependency challenges that come with annotating a layered codebase in sequence. The story starts with [PR #800](https://github.com/ClickHouse/clickhouse-connect/pull/800), the `dbapi` layer: the `connect()` entrypoint, the `Connection` class, and the `Cursor` class.

The annotation work itself was quite mechanical.What turned out to be interesting to me was what happened *after* push: a maintainer triggered a Copilot review, and the bot left five comments. Some flagged pre-existing bugs that the annotation work had exposed. Others were false alarms. Telling them apart required understanding the codebase more in depth.

## What mypy caught and what it didn't

Getting modules to pass mypy was mostly about adding annotations that reflected what the code was already doing. The only tricky part was `_summary` on the `Cursor` class. I had annotated it as `list[dict[str, str]]`, which looked reasonable to me (the summary is a dictionary of strings from the server response), but `QueryResult.summary` in the driver layer is `dict[str, Any]` and thus Mypy didn't flag the mismatch because the assignment from `query_result.summary` (typed `dict[str, Any]`) to `list[dict[str, str]]` happens via `.append()`, and mypy is lenient there. Copilot caught it.

The other issue mypy missed entirely had more to do with behaviour: after a successful bulk insert in `_try_bulk_insert`, the code set `self.data = []` but left `_rowcount` and `_ix` unchanged. This meant `cursor.rowcount` would reflect the *previous* query after a bulk `executemany`. PEP 249 requires `rowcount` to reflect the affected rows. Mypy has no way to catch this, it's a runtime semantic issue, not a type error.

## Sentinel leakage

The `create_client` function in the driver uses `port: int = 0` and `database: str = "__default__"` as internal sentinels for "not specified". These conventions are fine inside the driver, but they were leaking into the public-facing `Connection.__init__` and `connect()` signatures, which I had annotated with that definition in mind.

Copilot flagged this, and I think it was right in spirit. A caller of `connect()` who wants to express "I'm not specifying a port" has no natural way to do so: `None` would be rejected by the type checker, and `0` is an internal convention a library user shouldn't need to know about. The fix was to keep `port: int | None = None` in the public `connect()` and normalize to `0` before passing to `Connection`.

The `database` case was a false alarm. Copilot worried that passing `"__default__"` instead of `None` would interfere with DSN-based database selection. But `_parse_connection_params` already specifies `database == "__default__"` in a special case, on line 61:

```python
if parsed.path and (not database or database == "__default__"):
    database = unquote(parsed.path[1:].split("/")[0])
```

The driver treats `"__default__"` as equivalent to **"not set"** for DSN purposes. Knowing this required reading the driver internals; something Copilot couldn't do from the diff alone. I resolved the conversation by pointing at that line.

## The generator bug

The bulk insert path accessed `data[0]` to peek at the first row before deciding whether to take the fast path. PEP 249's `executemany` accepts any iterable as `seq_of_parameters`, including generators. A generator doesn't support indexing and `data[0]` raises `TypeError`.

The fix was a guard: `if not isinstance(data, Sequence) or len(data) == 0: return False`. Non-indexable iterables fall through to the row-by-row path, which consumes them correctly via a `for` loop. This is another bug that's almost invisible in a "normal" usage (most callers pass lists) but then breaks the moment someone writes `cursor.executemany(sql, (row for row in source))`.

## Collaborating on an open-source PR

The maintainer who reviewed the PR, Joe Spadola, asked me to add unit tests for the bulk insert behaviour changes before merging. His reasoning was clear: the existing tests used a bare `Mock()` client, so `insert_summary.written_rows` and `insert_summary.summary` were never exercised. The new tests cover `rowcount` after bulk insert, `summary` population, and the generator fallback.

He also asked me to highlight the `rowcount` fix in the changelog rather than the annotations, his view being that the annotations work should get a changelog entry when `py.typed` is restored at the end of the series, not per PR. That's a sensible way to manage a multi-PR annotation project.

Working with Copilot as a first reviewer was interesting. It's catches real things, but it seems it didn't know the codebase enough :-) For example, it can see that `"__default__"` is hardcoded, but it can't see that the driver already treats it as a well-defined sentinel. Resolving its comments means explaining the context inline rather than just dismissing, I hope this can be useful for future readers of the thread.

Globally, the dbapi layer was where the review dynamic was most interesting: Copilot as a first pass, a maintainer asking for tests, a few real bugs among the false alarms. The layers that followed were quieter on the review side but more architecturally focused: the SQLAlchemy integration in particular made me more aware of the design decisions of this nice project. I hadn't had the chance to dig into them, until this typing exercise pushed me into it.

## Managing dependencies across a multi-PR annotation series

The annotation work for issue #692 spans five PRs, each covering a different layer of the codebase. When you annotate in layers, you quickly hit a problem: the callers in an upper layer break the moment the lower layer's types change.

The driver-utilities layer ([PR #805](https://github.com/ClickHouse/clickhouse-connect/pull/805)) widened several `QueryContext` fields from `str` to `str | None`, correctly reflecting that the driver handles absent values without internal sentinels. The driver-clients layer (#806) and the SQLAlchemy layer (#807) call those same functions. Without #805 merged first, both PRs would have `[arg-type]` errors at those call sites.

The naive option — one monolithic PR — would have been too large to review meaningfully.

The approach that worked: add `# type: ignore[arg-type]  # widen after #805 merges` at each affected call site. The suppression is temporary and self-documenting. When #805 merges, rebasing #806 and #807 becomes mechanical: grep for the comment, delete the lines, verify mypy passes.

The ordering constraint surfaces naturally in the PR description: *this PR should be merged after #805 so the rebase removes these suppressions cleanly*. The reviewer knows what to expect before they open the diff.

## Into the SQLAlchemy layer

The SQLAlchemy integration (`cc_sqlalchemy`) was the largest and architecturally most interesting layer to annotate.

**Declared versus structural inheritance.** `TableEngine`, the base class for all ClickHouse table engine descriptors, inherited from `SchemaEventTarget` and `Visitable`: these are two SQLAlchemy base classes. `SchemaItem` is their named combination: SQLAlchemy defines it as `SchemaEventTarget + Visitable`. So `TableEngine` *was* already a `SchemaItem`: it responded to `_set_parent`, participated in schema events, and worked at runtime when passed to `Table()`. But it didn't *declare* it explicitly. Every call like `Table("...", MergeTree(order_by="version_num"), ...)` made mypy emit `[arg-type]`. The fix was one line: `class TableEngine(SchemaItem)` instead of the two separate bases, with no behaviour change.

**`__new__` as a factory.** `Nullable` and `LowCardinality` override `__new__` to return an instance of a completely different type. `Nullable(some_type)` doesn't give you back a `Nullable`, it gives you a wrapped `ChSqlaType`. Python allows this at runtime; mypy flags it with `[misc]` because `__new__` is supposed to return a subtype of its class. The suppression is unavoidable. After some research, this kind of constructor is called "pseudo-constructor" and it hides behind a class name.

**Monkey-patching as the last suppression.** The ClickHouse dialect adds `final()`, `sample()`, `prewhere()` and `limit_by()` methods to SQLAlchemy's `Select` class. There is no typed way to declare additional attributes on a class you don't own without maintaining stubs or subclassing. The six `# type: ignore[attr-defined]` on those assignments are irreducible. They are the annotation's acknowledgment that static typing has a limit here: you can fully annotate what you own, but not what you borrow and modify.

## My takeaways

It's striking how much a typing exercise like this taught me about the codebase. The types reveal design decisions that aren't accidents: base classes that carry specific semantics, constructors repurposed as factories, external classes extended to fit a dialect. A type annotation is an explicit claim about intent and you can discover where the design is subtler than it first appears.

Moreover, every type error asks a question: is the annotation wrong, or is the code? And the answer is sometimes surprising.

Maybe type annotation work is as much archaeology as engineering :) You're not inventing types, it's more you're uncovering assumptions that were always there, unnamed. It made me think of the contrast with Rust: there, you never uncover assumptions (or at least not in the same way) as the compiler made you state them upfront. Two different worlds, both full of things to learn and build with.


[clickhouse-connect](https://github.com/ClickHouse/clickhouse-connect) GitHub repo.
