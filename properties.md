# Properties of the LSP-Integrated Expander

These are correctness properties of the implemented system (span-keyed LSP tables, fault-tolerant expansion, cursor-insertion autocomplete). They are ordered roughly by how compelling they would be as formal claims in a paper.

## 1. Autocomplete Soundness

If `autocomplete` at position `p` returns name `n`, then replacing the syntax at `p` with `n` and re-expanding produces a program where `n` resolves to a binding (not `stx-error`).

This connects the scope graph traversal (`scope->names`) to the actual expansion behavior. For the cursor-insertion path, it additionally claims that inserting a cursor and re-expanding produces the same scope environment as the original expansion — the cursor is "transparent."

Edge cases that make this non-trivial:
- Macros that duplicate syntax: the intersection semantics must be right.
- The scope snapshot must faithfully capture the scope at resolution time, not after later mutations.
- Cursor insertion must not perturb macro pattern matching.

## 2. Goto-Definition / Find-References Duality

If `goto-definition` at span `r` returns span `d`, then `find-references` at span `d` returns a set containing `r`. Conversely, if `find-references` at span `d` returns a set containing `r`, then `goto-definition` at `r` returns a set containing `d`.

This is a consistency property between two tables (`resolutions` and `references`) that are populated independently during expansion. A bug where one table is updated but not the other would violate it. Its proof would force verification that `record-resolution!` maintains both tables in sync.

## 3. Fault-Tolerance Preservation

Let `P` be a program and `P'` be `P` with some subexpression replaced by a well-formed expression. For any surface identifier `id` in `P` whose span is unchanged and whose binding is unaffected by the edit, the resolution recorded for `id` in `expand(P')` equals the resolution recorded in `expand(P)`.

This says errors in one part of the program don't corrupt LSP results in unrelated parts. "Unaffected by the edit" means `id`'s binding site is in a scope that doesn't contain the error. The proof connects to the scope graph structure: errors are local to their scope subtree.

## 4. Expansion-LSP Agreement

For every surface identifier `id` with span `s` that the expander resolves via `scope-resolve`, the resolutions table contains an entry at `s`, and the binding recorded in that entry is the same binding that `scope-resolve` returned.

This is about completeness of instrumentation: no surface resolution is missed. `scope-resolve` is called in many contexts — expression expansion, definition pass 1, definition pass 2, macro application, pattern variable resolution — and the claim is that all of these paths record correctly. A missed `scope-resolve-internal` call (which doesn't record) where `scope-resolve` was intended would violate this.

## 5. Surface-Only Output

Every span returned by `goto-definition`, `find-references`, or `autocomplete` is a span that appears in the original surface syntax (i.e., is a key in `span->stx`).

LSP never points to a macro template location. This falls out of a design choice (keying tables on source spans) rather than explicit filtering. The proof would need to verify that macro template syntax never acquires a span that collides with a surface span.

## 6. Scope Snapshot Faithfulness

The names returned by `autocomplete` at a surface identifier `id` are exactly the names that were resolvable in the scope at the time `id` was resolved, not the names resolvable in that scope after subsequent `scope-bind!` calls.

This is about the `scope-snapshot` mechanism (issue #45). Without snapshots, autocomplete at a binding site in a `block` would incorrectly include later-defined names that weren't in scope during resolution. The proof would need to show that `scope-snapshot` produces a deep copy unaffected by future mutations.
