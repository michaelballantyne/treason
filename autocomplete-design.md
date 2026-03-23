# Autocomplete Design Comparison: Treason vs. Lean 4 vs. rust-analyzer

## The Key Question

When writing a macro definition, what IDE services are available *inside the template*? And do those services reflect what happens when the macro is actually used?

## Treason

### At macro definition time (eager, before any invocation)

When `let-syntax` or `define-syntax` is processed, `record-all-pvar-resolutions-for-macrot!` eagerly walks every clause's template:

1. Builds a `pvar-only-scp`: a scope containing `pattern-variable-binding` entries for each pattern variable, with `initial-scope` (keywords only) as the parent.
2. Calls `record-pvar-resolutions!`, which resolves every template identifier against this scope and records the result.

Effect: even for never-invoked macros, template positions get:
- **Autocomplete**: pattern variables + keywords.
- **Goto-definition**: template pvar references navigate to the pattern.
- **Find-references**: pattern pvars find all template uses.
- **Semantic highlighting**: pvars are highlighted as "macro" type.

### When the macro is invoked (expansion-informed)

`expand-template` instantiates the template: pattern variables are substituted with use-site syntax, and template-introduced identifiers are marked with `def-mark`. The result is then expanded by `expand-expr` in a `disjoin` scope. During this expansion, `scope-resolve` is called on every identifier — including template-introduced ones — and records resolutions keyed by **template spans** (the positions in the macro definition source).

This means expansion-time semantic information flows back into the template automatically:

- A template like `(let ([a $x]) a)` — when the macro is invoked, `a` is bound by `let` during expansion. The resolution is recorded at `a`'s template span. Clicking on `a` in the macro definition now gives you goto-def to the template's binding site, and autocomplete at that position shows what's in scope during expansion.
- **Multiple invocations** record additional resolutions at the same template spans. Autocomplete returns the intersection of names-in-scope across all invocations — a principled answer to "what names would be valid here regardless of how the macro is called?"
- **Semantic highlighting** reflects expansion-time binding types: a template identifier that resolves to a variable binding during expansion is highlighted as a variable, not as an unresolved name.

### Hygiene and autocomplete filtering

Treason's scope graph traversal (`scope->names`) naturally respects hygiene:

- At a **use site**: the `disjoin` scope has a use-site edge (no mark) and a def-site edge (with `def-mark`). An identifier without marks traverses the use-site edge, seeing only use-site bindings. Macro-introduced bindings (on the def-site edge) are invisible. Autocomplete at a use-site position shows only names the user could actually reference.
- At a **template position**: a template-introduced identifier has `def-mark`, so it traverses the def-site edge during expansion. Autocomplete at that position shows names reachable from the def-site scope — which includes macro-introduced bindings but not use-site-only bindings.
- **Scope snapshots** (issue #45): resolutions record a deep copy of the scope at resolution time, preventing later `scope-bind!` mutations from corrupting autocomplete results.

### Cursor insertion fallback

When the cursor is not on an existing identifier (whitespace, after an open paren), treason inserts a synthetic cursor identifier at the cursor position and re-expands the entire program. The cursor goes through normal expansion — acquiring marks, entering scopes — so autocomplete respects hygiene. The cursor's symbol is removed from the result set.

This is the expensive path: full re-expansion of the program. But it handles arbitrary positions, including positions inside macro templates and inside macro-expanded code.

## Lean 4

### At macro definition time

Quotation bodies (`` `(...) ``) are **parsed but not elaborated**. The Lean 4 reference manual states: "Quoted code is parsed, but not elaborated — while it must be syntactically correct, it need not make sense."

Consequence: **no `CompletionInfo` nodes are generated in the `InfoTree` for positions inside quotation templates.** The elaborator does not analyze the semantic content of the template; it treats the quotation as syntax data. This means:

- **No completions** for identifiers inside the template. The IDE does not know what names would be valid in eventual expansion contexts.
- **No goto-definition** for identifiers inside the template (except that identifiers are pre-resolved for hygiene via `Syntax.Preresolved`, but this does not produce `InfoTree` entries).
- **No hover information** — terms inside the quotation are never elaborated into `Expr` values.
- **No semantic highlighting** based on binding structure — only syntax-level highlighting from the parser.

Pattern variable splices (`$x`) are antiquotations that reference Lean-level bindings (of type `TSyntax k`). These are elaborated as Lean expressions *outside* the quotation, so `$x` gets IDE features (hover shows its type, goto-def works). But this is about the splice mechanism, not about the content of the template.

### At macro use sites

When a macro is invoked, the expanded syntax is elaborated normally. The `InfoTree` is populated with full `TermInfo`, `CompletionInfo`, etc. IDE features work for the expanded code. Source locations are mapped back to the call site via `MonadRef`/`getRef`.

Critically: **expansion-time information does NOT flow back to the template positions.** The `InfoTree` entries for the expanded code reference call-site positions (or synthetic positions), not template positions. If you go back to the macro definition and click on an identifier in the template, there is still no information.

### Hygiene in completions at use sites

At use sites, Lean 4's completion system explicitly handles hygiene in `CompletionCollectors.lean`:

- If an identifier has `SourceInfo.synthetic` and has macro scopes, the completion system returns **no completions** for it. Macro-generated identifiers are not offered as completion targets.
- If an identifier has `SourceInfo.original` (user-written), macro scopes are erased via `id.eraseMacroScopes` before matching against completion candidates. The user sees clean names.

This prevents macro-introduced names from leaking into use-site completions. But it is a binary filter: macro-introduced identifiers get nothing; user-written identifiers get everything in the local context (with scopes erased). There is no scope-graph-aware traversal that respects the fine-grained mark/scope structure.

### No re-elaboration for completion

Lean 4 never re-elaborates for completion. `CompletionInfo` nodes are emitted during the single elaboration pass. For positions with no `CompletionInfo` (whitespace, empty blocks), a "synthetic completion" fallback inspects the syntax tree. This is efficient but means completions are limited to what the elaborator chose to record.

## rust-analyzer

### At macro definition time

Inside `macro_rules!` definitions, IDE support is **severely limited**:

- **No `$x` pattern variable navigation** — you cannot goto-definition from a `$x` reference in the transcriber (RHS) to its declaration in the matcher (LHS). This is an open feature request (issue #7890).
- **No semantic analysis of template content** — identifiers in the macro body are not resolved. The blog post notes you "can't click on `_print`, for example" inside `println!`'s definition.
- **No completions** inside `macro_rules!` template bodies.
- **Minimal semantic highlighting** inside macro definitions.

### At macro use sites

When a `macro_rules!` macro is invoked, rust-analyzer expands it with its built-in declarative macro expander and semantically analyzes the result. The `SpanMap` maps tokens in the expanded code back to source positions (either call-site or macro-definition positions).

For identifiers that came from the call-site arguments (via captured `$x`), the span points back to the call site — goto-definition works. For identifiers from the macro body (fixed tokens), the span points into the macro definition — but semantic analysis at those positions is limited because the macro body is not independently analyzed.

### Completions inside macro invocations

rust-analyzer uses the "IntelliJ Trick":

1. Insert a fake identifier (`complete_me`) at the cursor position.
2. Re-parse the modified file.
3. For macro contexts, use `expand_speculative` to re-expand the enclosing macro with the fake identifier in its arguments.
4. Map the fake token into the expansion and analyze the semantic context.

This is more targeted than treason's full re-expansion (only re-expands the relevant macro, not the whole program) but does not produce information that flows back to the macro definition template.

### Hygiene

Rust's macro hygiene is based on `SyntaxContextId` (edition-based hygiene, not Scheme-style marks). rust-analyzer tracks `SyntaxContextId` through expansion via the `SpanMap`. Name resolution respects hygiene contexts. But completion does not have special hygiene-aware filtering comparable to Lean 4's `SourceInfo` check or treason's scope-graph traversal.

## Summary Comparison

| Feature | Treason | Lean 4 | rust-analyzer |
|---|---|---|---|
| **Completions inside template (definition time)** | Pvars + keywords (eager) | None (not elaborated) | None |
| **Completions inside template (after invocation)** | Full scope from expansion, intersection across invocations | None (doesn't flow back) | None (doesn't flow back) |
| **Goto-def on template pvar** | Yes (to pattern) | Yes (Lean-level binding, not template content) | No (open issue #7890) |
| **Semantic highlighting in template** | Yes (pvar/keyword/variable from expansion) | Syntax-level only | Minimal |
| **Completions at use site** | From resolution tables or cursor re-expansion | From CompletionInfo in InfoTree | Fake ident + expand_speculative |
| **Hygiene in use-site completions** | Scope graph traversal with marks | Binary filter: synthetic+scoped → nothing, original → erase scopes | SyntaxContextId in name resolution |
| **Re-expansion cost** | Full program re-expansion (cursor fallback) | None (single-pass) | Single macro re-expansion |
| **Expansion info flows to template?** | Yes (via span-keyed tables) | No | No |

## The Design Insight

Treason's key design choice — keying LSP tables by source span, and preserving template spans through expansion — causes expansion-time semantic information to automatically accumulate at template positions. This requires no special mechanism: it's a natural consequence of:

1. Template-introduced identifiers retain their template spans during `expand-template`.
2. `scope-resolve` records resolutions keyed by span.
3. The same span can accumulate multiple resolutions from multiple invocations.

Neither Lean 4 nor rust-analyzer have this property. In Lean 4, the InfoTree records information at expanded-code positions (call site or synthetic), not template positions. In rust-analyzer, the SpanMap maps expanded tokens back to definition positions, but the definition body is not semantically analyzed — the spans are used for navigation targets, not for accumulating semantic information at template positions.

The result is that treason is the only system where writing a macro and then using it progressively enriches the IDE experience at the macro definition site. A macro author editing a template sees completions, highlighting, and navigation that reflect actual usage — without any explicit annotation or cooperation.

## Limitations

- **Cost**: treason re-expands the entire program for cursor-insertion autocomplete. Lean 4's single-pass approach is more efficient.
- **Scope**: treason only supports `syntax-rules` macros. The template-span-preservation trick relies on the template being visible syntax, not an opaque procedure.
- **Intersection semantics**: when a macro is used in different contexts, autocomplete shows the intersection of names from all contexts. This is principled but conservative — some valid names may be excluded if they're not available in all invocation contexts.
- **Stale information**: if a macro invocation is removed, its resolutions remain in the tables until the next full re-expansion. There is no garbage collection of stale resolutions.
