# Evaluation: Treason as PL Research

## What This Is

Treason is a small Lisp (supporting `let`, `if`, `define`, `block`, `let-syntax`, `define-syntax`, and `syntax-rules`) with a hygienic macro expander that simultaneously populates data structures consumed by a Language Server Protocol (LSP) implementation. The expander uses a scope-graph model for hygiene and is designed to be *fault-tolerant*: expansion continues past errors, producing `stx-error` sentinel nodes so that LSP features (goto-definition, find-references, autocomplete, semantic tokens) remain functional even in incomplete or erroneous programs.

## Core Design Ideas

### 1. Expansion as LSP Instrumentation

The central design idea is that the macro expander itself is the source of truth for IDE services. During expansion, every name resolution records a `Resolution` (binding, reference syntax, scope snapshot) into mutable tables keyed by source span. Every `scope-bind!` records the binding site. These tables are the *only* data structures the LSP queries read — there is no separate "analysis pass."

This is in contrast to systems like DrRacket, where the expander and the IDE annotation mechanism (`syntax-property`, `'disappeared-use`, `'disappeared-binding`) are separate systems that must be coordinated, often by individual macro authors.

### 2. Surface vs. Macro-Introduced Distinction

The expander carefully distinguishes surface syntax (from the source file, with source spans) from macro-introduced syntax (from templates, which may retain template spans but are not navigable). LSP tables are keyed by source spans, so macro-introduced identifiers are naturally excluded from IDE results. Pattern variable substitutions preserve use-site spans, maintaining the connection between the user's source and macro-expanded code.

### 3. Fault-Tolerant Expansion for LSP

The expander catches `stx-error` exceptions and embeds them as values in the expanded AST rather than aborting. This means:
- Partial expansion results are available for LSP queries.
- Multiple errors are reported simultaneously.
- Goto-definition and autocomplete work on code with errors (e.g., `(let ([x unbound]) x)` — goto-def on `x` still works).

### 4. Autocomplete via Cursor Insertion and Re-expansion

When the cursor is not on an existing identifier, treason inserts a synthetic cursor identifier at the cursor position and re-expands the entire program. The cursor goes through normal expansion (acquires marks, enters scopes) so autocomplete respects hygiene and macro-introduced scoping.

### 5. Scope Snapshots for Correctness

When recording resolutions for autocomplete, the expander snapshots the scope graph (deep-copies mutable binding tables). This prevents later `scope-bind!` mutations from corrupting autocomplete results — a subtle correctness issue (tracked as issue #45) that arises because expansion is imperative and scope graphs are mutable.

### 6. Pattern Variable LSP Support

Pattern variables in `syntax-rules` patterns get their own binding/resolution tracking. Goto-definition on a template reference to a pattern variable navigates to the pattern, and find-references from a pattern variable finds all template uses. This works even for macros that are never invoked, because pvar resolutions are recorded eagerly at `let-syntax`/`define-syntax` time.

### 7. Multi-Valued Resolution for Duplicated Syntax

When a macro duplicates use-site syntax (e.g., `(syntax-rules () [(_ x) (let ([a x]) x)])`), the same source span may be resolved multiple times under different scopes. The resolution tables are multi-valued (lists of Resolutions per span). For autocomplete, the intersection of names-in-scope across all resolutions is returned — a principled answer to the question "what can I type here if this syntax appears in multiple expansion contexts?"

## Honest Assessment of Novelty

### Not novel: Fault tolerance

All production systems achieve the same basic property. rust-analyzer continues analysis past macro failures (proc macros that fail produce `compile_error!`; the surrounding code still gets full analysis). Lean 4 retains `PartialTermInfo` and `ChoiceInfo` from failed elaborators so the language server provides interactivity even when elaboration fails. In all three systems — treason, rust-analyzer, Lean 4 — macros are all-or-nothing (they succeed or fail), and the surrounding system recovers structurally at the enclosing binding form. Treason does nothing here that the others don't.

### Not novel: Expander produces IDE metadata

Lean 4's elaborator produces an `InfoTree` during elaboration — the same architecture as treason's approach. The elaborator *is* the IDE analysis; there is no separate pass. `InfoTree` nodes record types, goals, local contexts, macro expansion steps, and completion info. The `InfoTree` even supports metavariable-like holes for incremental elaboration. This is more sophisticated than treason's approach, not less.

### Not novel as a technique, but interesting tradeoff: Cursor insertion for autocomplete

rust-analyzer uses the same fake-identifier trick (the "IntelliJ Trick") and `expand_speculative` for re-expanding the enclosing macro. The technique itself is not novel.

Lean 4 takes a different approach: it never re-elaborates for completion. `CompletionInfo` nodes are emitted during elaboration and read directly. For non-identifier positions, a synthetic completion fallback inspects the syntax tree. This is more efficient for use-site completions.

However, Lean 4's single-pass architecture means quotation template bodies are never elaborated, so **completions inside macro templates are zero**. The same design that makes use-site completion cheap makes template-site completion impossible. rust-analyzer similarly provides no completions inside `macro_rules!` template bodies.

Treason's cursor-insertion approach is more expensive (full re-expansion) but works **everywhere**, including inside macro templates. A cursor inserted into a template position goes through the same expansion paths as any other template identifier — it gets pvar resolution eagerly and expansion-informed scope when the macro has invocations. See `autocomplete-design.md` for the detailed comparison.

### Novel: Expansion-informed IDE services in macro templates

Treason's span-keyed LSP tables cause expansion-time semantic information to flow back to macro template positions automatically. When a macro is invoked, template-introduced identifiers are resolved during expansion, and those resolutions are recorded at the template's source spans. This gives macro authors:

- **Completions** at template positions reflecting what's in scope during expansion.
- **Goto-definition** on template-introduced bindings/references.
- **Semantic highlighting** based on expansion-time binding types.
- **Multi-invocation accumulation**: each invocation adds resolutions; autocomplete returns the intersection.

Neither Lean 4 nor rust-analyzer have this property. In Lean 4, quotation template bodies are not elaborated and nothing flows back from expansion. In rust-analyzer, `macro_rules!` template bodies have minimal IDE support and the `SpanMap` does not accumulate semantic information at template positions.

This requires no special mechanism — it falls out of two design choices: (1) template-introduced identifiers retain their template spans during `expand-template`, and (2) `scope-resolve` records resolutions keyed by span. These are individually straightforward, but their combination produces a novel capability.

### Potentially novel: Pattern variable IDE support

rust-analyzer has **no support** for navigating within `macro_rules!` definitions. You cannot goto-definition from a `$x` reference in the transcriber (RHS) to its declaration in the matcher (LHS). This is an open feature request (issue #7890).

Lean 4 **does** have pattern variable IDE support, but through a different mechanism: pattern variables in `macro_rules` are regular Lean-level `let` bindings of type `TSyntax k`. The `$x` splice in a quotation template is a regular Lean expression referencing that binding, so goto-def "just works" with no special machinery. This is arguably a stronger design — Lean gets pattern variable IDE support for free by making pattern variables first-class language bindings.

Treason provides full IDE support for pattern variables via special-purpose machinery: `pattern-variable-binding` types and `record-all-pvar-resolutions-for-macrot!` for eager resolution at definition time. This works even for never-invoked macros. But it achieves the same result as Lean with more effort.

The only system treason clearly outperforms here is rust-analyzer, and rust-analyzer's limitation is a known engineering gap (open issue #7890), not a fundamental architectural issue.

### Potentially novel: Multi-valued resolution with intersection semantics

When a macro duplicates use-site syntax, the same source span is resolved multiple times under different scopes. The intersection semantics for autocomplete gives a principled answer. rust-analyzer and Lean 4 don't face this specific problem because their macro systems don't duplicate use-site syntax in the same way. But again, this is a narrow case.

### Not novel but different from DrRacket: No macro-author cooperation needed

DrRacket's check-syntax requires macro authors to attach `syntax-property` annotations (`'disappeared-use`, `'disappeared-binding`) for correct binding arrows. Treason's approach is automatic for `syntax-rules` macros. However, this advantage disappears if treason were extended to procedural macros — the same cooperation problem would arise. And Lean 4 and rust-analyzer also don't require macro-author cooperation for basic IDE features within expanded code.

## Closest Related Work

### DrRacket Check Syntax
DrRacket's check-syntax draws binding arrows using `syntax-property` annotations. Requires macro-author cooperation. Runs after expansion as a separate traversal. DrRacket does not attempt fault-tolerant expansion — a macro error halts analysis. Treason's approach is more automatic but limited to `syntax-rules`.

### Lean 4 InfoTree
The elaborator produces an `InfoTree` during elaboration — same "expansion as analysis" architecture. `PartialTermInfo` and `ChoiceInfo` retain partial results from failed elaborators. The `InfoTree` supports metavariable-like holes for incremental elaboration. Completion uses `CompletionInfo` nodes recorded during elaboration, reading local context and expected type directly from the tree (not via re-elaboration with a synthetic identifier). Lean 4 also has a "canonical synthetic `SourceInfo`" flag that lets macro-generated identifiers be treated "as if" the user wrote them for IDE purposes — a more sophisticated version of treason's surface-vs-macro-introduced distinction. Lean does not need DrRacket's `disappeared-use`/`disappeared-binding` because binding information is preserved through canonical source info, pre-resolved identifiers, and macro scopes. Pattern variables in `macro_rules` are regular Lean bindings and get IDE features for free. More sophisticated than treason in every dimension.

### rust-analyzer
Uses fake-identifier insertion + `expand_speculative` for completions inside macros — the same basic approach as treason's cursor insertion. Continues analysis past macro failures. Has no support for navigating within `macro_rules!` definitions (open issue #7890). Faces harder problems than treason due to proc macros (opaque, potentially non-deterministic, can crash). Navigation for items *produced* by macro expansion works, but navigation *within* macro definitions is limited.

### Spoofax / Statix
Uses scope graphs to declaratively specify name binding and derive IDE editor services. Most architecturally similar: scope graphs are the shared representation for semantic analysis and IDE features. Does not handle macro expansion. Recent work on "Language-Parametric Static Semantic Code Completion" (OOPSLA 2022) derives code completion from Statix specs.

### syntax-spec (Ballantyne et al., ICFP 2024)
Metalanguage for creating hosted DSLs in Racket. Provides grammar and binding rule declarations. Generates a macro expander that checks binding. Relies on DrRacket for IDE support. Does not itself provide LSP features or fault-tolerant expansion.

## What Would Make This More Interesting

### 1. Procedural Macros
The current system only supports `syntax-rules`. Extending to procedural macros (arbitrary Racket functions producing syntax) is the real test. The key question: can the expander still automatically provide IDE features when macro transformers are opaque functions? This is where DrRacket needs `disappeared-use`/`disappeared-binding`, and where rust-analyzer's `expand_speculative` breaks down for attribute macros.

### 2. Formal Properties and Proofs
See `properties.md`. The most interesting property is autocomplete soundness: any name returned by autocomplete, if inserted, would not produce an unbound error. Proving this connects the scope graph traversal (`scope->names`) to actual expansion behavior and requires showing that cursor insertion is "transparent" (doesn't perturb the rest of expansion). No existing system has proven such a property.

### 3. Incremental Re-expansion
Currently, every edit triggers full re-expansion (and autocomplete at non-identifier positions triggers a *second* full re-expansion). Lean 4's snapshot-based incremental architecture is far more sophisticated. Implementing even partial incrementality would address scalability.

### 4. Grammar and Binding Rule Declarations
Declaring the grammar and binding structure of macros (as syntax-spec does) would enable richer completions, better error recovery, and analysis without full expansion. This connects to both the syntax-spec line of work and the Spoofax/Statix approach.

### 5. Evaluation Against DrRacket
A concrete comparison with DrRacket's check-syntax on equivalent programs — especially programs with macros, errors, and incomplete code — would provide evidence of advantages. DrRacket's inability to provide IDE features after macro errors is a real limitation that treason addresses.

## Summary

After careful comparison with Lean 4, rust-analyzer, and DrRacket:

- **Fault tolerance**: Same level as Lean 4 and rust-analyzer. Not a contribution.
- **"Expander as IDE analysis" architecture**: Same as Lean 4's InfoTree. Not a contribution on its own.
- **Cursor insertion for autocomplete**: Same technique as rust-analyzer. Not novel as a technique.
- **No macro-author cooperation**: True vs. DrRacket, but same as Lean 4 and rust-analyzer.

- **Expansion-informed IDE services in macro templates**: Genuinely novel. Neither Lean 4 (quotation templates aren't elaborated) nor rust-analyzer (`macro_rules!` templates have minimal IDE support) provide IDE features inside macro templates that reflect actual expansion behavior. Treason's span-keyed tables cause this to happen automatically. See `autocomplete-design.md`.
- **Pattern variable IDE support**: Treason and Lean 4 both support this (Lean via first-class bindings, treason via special machinery). rust-analyzer lacks it (open issue #7890).
- **Multi-valued resolution / intersection semantics**: Falls out of the template-span design. Principled but narrow.

The strongest potential contribution is the expansion-informed template IDE story: the observation that keying LSP tables by span + preserving template spans through expansion causes semantic information to flow from use sites back to the macro definition automatically. This is a design insight that could inform production systems. The most promising directions for a paper are (a) developing this insight with formal properties, (b) extending to procedural macros, or (c) integrating with grammar/binding-rule declarations to provide template IDE services without requiring invocations.
