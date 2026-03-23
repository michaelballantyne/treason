# IDE Services for Macro-Extensible Languages: Gaps and Directions

## Services × Contexts Matrix

For each combination: what works, what doesn't, and what could.

### Contexts

- **A. Ordinary code** — no macros involved
- **B. Use-site code through macros** — user-written syntax that passes through macro expansion
- **C. Macro-introduced code** — template-originated syntax (not from user input)
- **D. Macro definitions** — macro names, pattern variables, template structure
- **E. Erroneous code** — programs with syntax or semantic errors
- **F. Incomplete macro uses** — partially typed macro invocations where pattern matching hasn't succeeded
- **G. DSL syntax** — macros defining new syntactic forms with custom grammars

### Gap Analysis

| Service | A (ordinary) | B (use-site) | C (template) | D (macro def) | E (errors) | F (incomplete) | G (DSL) |
|---|---|---|---|---|---|---|---|
| Goto-def | ✓ all | ✓ all | treason only | partial | ✓ all | ✗ | ✗ |
| Find-refs | ✓ all | ✓ all | treason only | partial | ✓ all | ✗ | ✗ |
| Autocomplete | ✓ all | ✓ all | treason only | partial | ✓ all | ✗ | ✗ |
| Semantic HL | ✓ all | ✓ all | treason only | partial | ✓ all | ✗ | ✗ |
| Diagnostics | ✓ all | ✓ all | treason only | ✗ | ✓ all | ✗ | ✗ |
| Hover/type | Lean/RA | Lean/RA | ✗ | ✗ | partial | ✗ | ✗ |
| Rename | Lean/RA | limited | ✗ | ✗ | ✗ | ✗ | ✗ |
| Sig help | Lean/RA | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |

Legend: ✓ all = treason + Lean 4 + rust-analyzer; partial = some systems; ✗ = no system

### The Big Gaps

**1. Incomplete macro uses (column F)**

This is the most impactful gap for day-to-day macro use. When you're typing `(let ([x `, the `let` pattern hasn't matched — no binding has occurred, no expansion has happened. Current systems provide nothing: no completions for what should come next, no indication of the expected form, no partial binding of already-typed identifiers.

With grammar declarations, the system could know that the next expected token is an expression (the RHS of the binding). It could:
- Show completions appropriate for expression position
- Already bind `x` as a variable in scope for the body (even though the body isn't typed yet)
- Show signature help: "let expects ([var expr]) body"
- Highlight `x` as a binding site

This requires knowing the macro's grammar *before* the pattern fully matches. syntax-rules patterns already declare this structure implicitly — each pattern IS a grammar. The gap is that current expanders wait for a complete match before doing anything.

**2. DSL syntax (column G)**

When a macro defines a new syntactic form (a DSL), the IDE needs to understand:
- What syntax is valid (grammar)
- What names are bound and where (binding rules)
- How DSL constructs relate to host-language constructs

Currently, DSL syntax is opaque to all IDE systems until the macro expands it to host-language code. If expansion fails (because the code is incomplete), the IDE provides nothing.

With grammar + binding rule declarations (syntax-spec), the IDE could provide services for DSL syntax directly:
- Grammar-aware completions (only valid DSL forms)
- DSL-level goto-definition (navigating within the DSL, not the expansion)
- DSL-specific diagnostics before expansion

This is where syntax-spec's binding rules become essential: they declare the binding structure of DSL forms, enabling IDE services without expanding to host code.

**3. Rename/refactoring across macro boundaries (column "Rename")**

Renaming a variable that's used across macro boundaries is hard:
- Rename a use-site variable that a macro pattern-matches and re-binds: need to rename in the use site only
- Rename a variable that appears in both use-site and template code: need to understand which occurrences are "the same"
- Rename a pattern variable: need to rename in pattern AND all template uses
- Rename a macro name: need to rename definition and all use sites

Treason's span-keyed tables contain enough information for some of these (pvar rename, macro name rename). But renaming variables that cross the use-site/template boundary requires understanding the macro's binding structure.

**4. Diagnostics in macro templates (column D/C)**

Can we detect errors in a macro template at definition time, before any invocation?

- "This template identifier is never bound in any expansion context" — could flag dead template code
- "This template references a name that's only sometimes in scope" — the intersection semantics could power warnings
- "This pattern variable is bound but never used in the template" — straightforward with pvar tracking
- Grammar violations in the template output — if the template produces syntax that doesn't match the expected grammar

The first two require expansion-informed analysis (treason's current approach). The last requires grammar declarations.

**5. Hover/type information (column "Hover")**

Treason has no type system, so hover is limited to "this identifier resolves to this binding." But even without types, hover could show:
- For a macro name: the macro's pattern(s) — signature help
- For a pattern variable: its position in the pattern, what syntax category it matches
- For a template identifier: what it resolves to during expansion
- For a use-site identifier: the binding site, and whether it passes through macro expansion

## Directions

### Direction 1: Grammar-Informed Services for Incomplete Macro Uses

**The problem**: When a macro use is partially typed, the pattern doesn't match, so no expansion occurs. The IDE is dark.

**The insight**: A `syntax-rules` pattern IS a grammar declaration. The pattern `(_ ([var expr]) body)` declares that the macro expects a binding pair followed by a body. Before the pattern fully matches, the system can:

1. **Recognize partial matches**: The pattern has matched up to a certain point. The next expected element is known.
2. **Pre-bind identifiers**: If the pattern has matched `(_ ([x `, we know `x` is in binding position. Tentatively bind it.
3. **Offer grammar-aware completions**: At the cursor, show what the grammar expects (an expression, an identifier, etc.).
4. **Expand subexpressions early**: The already-matched subexpressions can be expanded in the current scope, providing IDE services for the parts that are complete.

This is the "grammar-informed early subexpression expansion" mentioned in TODO.md (treason-idb).

**What's needed**: A partial pattern matching algorithm that returns the set of possible continuations at the match frontier, plus the bindings discovered so far.

**Novelty**: No existing system does this. Lean 4's macros fail completely on partial syntax. rust-analyzer's macro expansion is all-or-nothing. Spoofax has grammar-aware completion for surface syntax but not for macro invocations.

### Direction 2: Binding Rules for Template Analysis

**The problem**: IDE services in macro templates currently require invocations (treason) or provide nothing (Lean 4, rust-analyzer).

**The insight**: If binding rules are declared for macro output, the template can be analyzed statically:

- The binding rules say "in `(let ([var expr]) body)`, `var` is bound in `body`"
- Applied to a template like `(let ([a $x]) a)`, this tells us: `a` is a binder, `a` in the body is a reference to it, `$x` is an expression in the RHS
- No expansion needed — the binding structure is determined by the grammar

**What's needed**: syntax-spec already has these declarations. The gap is connecting them to IDE service generation.

**Novelty**: Spoofax/Statix derives IDE services from binding specs for surface languages. But doing this for macro *templates* — where the binding specs describe the output grammar, not the input grammar — is new. The template is written in the output grammar, so the output's binding rules apply directly.

### Direction 3: Compositional IDE for Macro-Defining Macros

**The problem**: When macros define other macros (as in syntax-spec DSLs), there are multiple levels of template. IDE services need to work at each level.

**The insight**: If each macro level has grammar + binding rule declarations, the IDE can provide services at each level independently:

- Level 0: The host language (let, define, etc.)
- Level 1: A macro written in the host language
- Level 2: A macro written using a macro from level 1

At each level, the binding rules describe the scope structure, enabling template IDE services.

**What's needed**: A compositional framework where binding rules at one level can reference binding rules at another level. syntax-spec's "nonterminal" system is a starting point.

**Novelty**: No existing system provides IDE services for multi-level macro definitions.

### Direction 4: IDE-Aware Macro API for Procedural Macros

**The problem**: Procedural macros are opaque functions. The template IDE property doesn't apply because there's no visible template.

**The insight**: If procedural macros use a structured API for constructing output (rather than raw syntax manipulation), the API can track:

- Which output tokens came from input (span preservation)
- Which output tokens are macro-introduced (template-like)
- What binding structure the output has (scope introduction)

This is essentially what syntax-spec's `bind!` and `scope-tagger` already do, but viewed through an IDE lens.

**What's needed**: Design an API where the "easy way" to write a procedural macro automatically provides IDE metadata. Make it easier to write the macro correctly (using the API) than incorrectly (raw syntax manipulation).

**Novelty**: DrRacket's `disappeared-use`/`disappeared-binding` is the current approach, but it's opt-in annotation after the fact. An API where IDE metadata is a natural byproduct of correct macro construction would be a significant improvement.

## Priorities

**Highest impact, most feasible**: Direction 1 (grammar-informed incomplete macro uses). This addresses the most common pain point (IDE going dark while typing), builds naturally on treason's existing syntax-rules patterns, and is self-contained.

**Highest novelty**: Direction 2 + 3 (binding rules for templates, compositional). This tells the strongest research story and connects to syntax-spec.

**Highest practical value**: Direction 4 (procedural macro API). This is what production systems need. But it's also the hardest and most open-ended.

**Recommended path**: Start with Direction 1 (incomplete macro uses — it's immediately useful and feasible), then extend to Direction 2 (binding rules for templates — strengthens the research story), and frame the paper around the combination: "grammar and binding rule declarations enable comprehensive IDE services for macro-extensible languages, including during editing (incomplete code) and at macro definition sites (templates)."
