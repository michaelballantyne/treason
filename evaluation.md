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

When the cursor is not on an existing identifier, treason inserts a synthetic cursor identifier at the cursor position and re-expands the entire program. The cursor goes through normal expansion (acquires marks, enters scopes) so autocomplete respects hygiene and macro-introduced scoping. This is a clean solution to the problem of "what names are available at this position" that naturally handles macros.

### 5. Scope Snapshots for Correctness

When recording resolutions for autocomplete, the expander snapshots the scope graph (deep-copies mutable binding tables). This prevents later `scope-bind!` mutations from corrupting autocomplete results — a subtle correctness issue (tracked as issue #45) that arises because expansion is imperative and scope graphs are mutable.

### 6. Pattern Variable LSP Support

Pattern variables in `syntax-rules` patterns get their own binding/resolution tracking. Goto-definition on a template reference to a pattern variable navigates to the pattern, and find-references from a pattern variable finds all template uses. This works even for macros that are never invoked, because pvar resolutions are recorded eagerly at `let-syntax`/`define-syntax` time.

### 7. Multi-Valued Resolution for Duplicated Syntax

When a macro duplicates use-site syntax (e.g., `(syntax-rules () [(_ x) (let ([a x]) x)])`), the same source span may be resolved multiple times under different scopes. The resolution tables are multi-valued (lists of Resolutions per span). For autocomplete, the intersection of names-in-scope across all resolutions is returned — a principled answer to the question "what can I type here if this syntax appears in multiple expansion contexts?"

## Assessment of Novelty

### What's new here

**The integration of fault-tolerant expansion, LSP table instrumentation, and hygienic macro expansion into a single unified mechanism** is the main contribution. No prior system that I'm aware of provides all of the following simultaneously:

1. **The expander *is* the IDE analysis** — resolution recording is woven into `scope-resolve`, not bolted on after the fact.
2. **Fault tolerance** — expansion continues past errors, and LSP features degrade gracefully.
3. **Hygiene-aware autocomplete** via cursor insertion and re-expansion.
4. **Multi-valued resolution** for syntax duplicated by macros, with principled intersection semantics.
5. **Pattern variable IDE support** with eager resolution recording.

Each of these ideas is individually modest, but their combination in a clean, small system is what makes the contribution interesting.

### What's not new

- Hygienic macro expansion via scope graphs (prior work by the author, Flatt's "Binding as Sets of Scopes").
- LSP implementations for macro-extensible languages exist (DrRacket/check-syntax, racket-langserver).
- Fault-tolerant parsing and type-checking for IDEs is well-established (tree-sitter, rust-analyzer, etc.).
- Scope graphs for name resolution (Néron, Tolmach, Visser — ESOP 2015).

## Closest Related Work

### DrRacket Check Syntax
DrRacket's check-syntax draws binding arrows between identifiers using `syntax-property` annotations (`'disappeared-use`, `'disappeared-binding`). However, this system requires macro authors to explicitly cooperate by attaching properties. It also runs *after* expansion as a separate traversal, rather than being integrated into the expander. DrRacket does not attempt fault-tolerant expansion — a macro error halts analysis. Treason's approach is more automatic (no macro-author cooperation needed for basic LSP features) but limited to `syntax-rules` macros (no procedural macros).

### Lean 4 InfoTree
Lean 4's elaborator produces an `InfoTree` during elaboration that records types, goals, macro expansion steps, and other metadata used by the language server. The `InfoTree` supports metavariable-like holes for incremental elaboration. This is architecturally similar to treason's approach — elaboration/expansion produces metadata consumed by IDE features — but operates in a much more complex setting (dependent types, tactics, universe polymorphism). Lean does not specifically address the problem of fault-tolerant elaboration for IDE use, though its incremental snapshot-based architecture provides partial results.

### rust-analyzer
rust-analyzer faces the challenge that name resolution and macro expansion are deeply intertwined in Rust. It handles proc macros by running them in a separate process and caching results. It has significant challenges with fault tolerance (proc macros that receive syntactically invalid input tend to produce `compile_error!`). The rust-analyzer blog post "IDEs and Macros" (2021) articulates many of the same problems treason addresses, but in a more complex and less clean setting. rust-analyzer does not have a principled story for autocomplete inside macro-generated code.

### Spoofax / Statix
The Spoofax language workbench uses scope graphs (via the Statix metalanguage) to declaratively specify name binding, and derives IDE editor services (completion, renaming) from these specifications. This is the most architecturally similar approach: scope graphs are the shared representation for both semantic analysis and IDE features. However, Spoofax targets language *workbench* users (language designers), not language *implementation* (a hand-written expander). Spoofax does not handle macro expansion — its scope graphs describe the binding structure of the surface language. Recent work on "Language-Parametric Static Semantic Code Completion" (OOPSLA 2022) derives code completion from Statix specs, but without macro expansion.

### syntax-spec (Ballantyne et al., ICFP 2024)
syntax-spec is a metalanguage for creating hosted DSLs in Racket. It provides grammar and binding rule declarations, and generates a macro expander that checks binding and expands DSL macros. This is closely related: it also integrates binding analysis with macro expansion. However, syntax-spec targets DSL *creation* within Racket's ecosystem and relies on DrRacket for IDE support. It does not itself provide LSP features or fault-tolerant expansion. Treason could be seen as exploring a complementary dimension: what if the expander itself were designed from the ground up to produce IDE metadata?

### Macros for Domain-Specific Languages (Ballantyne, King, Felleisen — OOPSLA 2020)
The `ee-lib` API provides scoping and binding primitives for DSL macro expanders. The paper discusses how DSL expanders need to manage scopes, bindings, and macro application. This is the intellectual ancestor of treason's approach, but `ee-lib` is an API for building expanders within Racket, not a standalone system with integrated LSP support and fault tolerance.

## Potential Paper Pitch

### Title
"Expansion as Analysis: Integrating IDE Services into a Hygienic Macro Expander"

### Thesis
IDE features for macro-extensible languages can be provided *automatically* — without cooperation from macro authors — by instrumenting the macro expander to record resolution metadata during expansion. Combined with fault-tolerant expansion and cursor-insertion-based autocomplete, this yields a complete LSP implementation from a single expansion pass.

### Venue
- **OOPSLA** (Experience Reports or main track): The work is systems-oriented with a clear design contribution. OOPSLA has published related work (Ballantyne et al. 2020, Spoofax papers).
- **ICFP** (Functional Pearl): If the presentation emphasizes the elegance of the single-pass design and the cursor-insertion trick.
- **SLE** (Software Language Engineering): Natural fit for language tooling and IDE support research.
- **<Programming>**: Good venue for language design and implementation papers with a practical bent.

### Key Claims
1. A macro expander can serve as the sole source of IDE metadata, eliminating the need for separate analysis passes or macro-author cooperation.
2. Fault-tolerant expansion (via error sentinel values) enables LSP features on incomplete programs.
3. Cursor insertion with re-expansion provides hygiene-aware autocomplete that correctly handles macro-introduced scoping.
4. Multi-valued resolution tables with intersection semantics give principled answers for syntax duplicated by macros.

## What Would Make This More Interesting

### 1. Procedural Macros
The current system only supports `syntax-rules`. Extending to procedural macros (arbitrary Racket functions producing syntax) would dramatically increase the scope and challenge. The key question: can the expander still automatically provide IDE features when macro transformers are opaque functions? This is the problem that makes DrRacket's `disappeared-use`/`disappeared-binding` mechanism necessary.

### 2. Formal Properties and Proofs
The design document lists 13 correctness properties (ID uniqueness, ID preservation through expansion, hygiene preservation in LSP, autocomplete soundness, etc.). Formally stating and proving these — even for the restricted `syntax-rules` setting — would significantly strengthen the contribution. Property 13 (autocomplete soundness: "any name returned by autocomplete, if inserted, would not produce an unbound error") is particularly interesting and non-trivial.

### 3. Incremental Re-expansion
Currently, every edit triggers full re-expansion. The notes.md file contains extensive thinking about incremental/reactive re-expansion. Implementing this — even partially — would address scalability and connect to the Lean 4 snapshot-based approach and Spoofax's incremental constraint solving.

### 4. Evaluation on Real Programs / User Study
A comparison with DrRacket's check-syntax on equivalent programs (especially programs with macros, errors, and incomplete code) would provide concrete evidence of the advantages. A user study comparing IDE responsiveness and correctness across treason, DrRacket, and a baseline (no macro awareness) would be compelling.

### 5. Grammar and Binding Rule Declarations
The TODO mentions grammar-informed early subexpression expansion for incomplete macro uses. Declaring the grammar and binding structure of macros (as syntax-spec does) would enable richer completions, better error recovery, and analysis without full expansion. This connects directly to the syntax-spec line of work and the Spoofax/Statix approach.

### 6. Multi-File / Module Support
The current system is single-file. Adding module support would introduce cross-file resolution, import/export tracking, and the need for incremental analysis — all significant IDE challenges.

### 7. Blame and Error Reporting for Macros
The notes.md discusses the subtle problem of error blame in macro-expanded code (should errors point to the use site or the template?). A principled treatment of error blame that leverages the expansion metadata would be a standalone contribution.

## Summary

Treason represents a clean, focused exploration of an underexplored design point: **what happens when you design a macro expander from the ground up to produce IDE metadata?** The resulting system is simple enough to understand completely, yet addresses real problems (fault tolerance, hygiene-aware autocomplete, macro-duplicated syntax) that production systems struggle with. The main limitation is scope — `syntax-rules` only, single-file, no incremental re-expansion. The most promising directions for strengthening the contribution are (a) formal properties with proofs, (b) extension to procedural macros, and (c) an evaluation comparing with DrRacket's check-syntax on programs with macros and errors.
