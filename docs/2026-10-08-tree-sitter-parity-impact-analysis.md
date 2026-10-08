# Tree-sitter scope-parity pass

## Plan & Impact Analysis

Status: scoped and ready to implement (visible items only).

## Decisions (agreed)

1. **Stack scopes.** Keep the existing `meta.*` scopes and add tree-sitter
   compatible scopes alongside them; do not replace.
2. **Keep our finer categories.** Aim for "compatible where it matters",
   not byte-for-byte parity with tree-sitter. Where tree-sitter lumps
   (`@keyword` for modifiers, `@type` guessed from `^[A-Z]`), we keep the
   more precise category unless it blocks theming.
3. **Scope limited to the visible items.** Only identifier references and
   interpolation contents are in scope. Everything else is deferred.

## Goal

The grammar already tokenizes Pkl correctly; the difference from Neovim is
that some tokens carry scopes tree-sitter-based themes do not style. The
most visible case is identifiers, scoped `meta.name.pkl` (a meta scope),
which falls back to the theme's default foreground. This pass adds the
tree-sitter capture scopes on top so those themes colour Pkl like Neovim,
without regressing Sublime or `bat`.

## Sources

- `apple/tree-sitter-pkl/queries/highlights.scm` — capture names.
- `syntaxes/Pkl.sublime-syntax` — current scopes.
- Prior decision `ed8fe4d` (`variable.*.pkl` -> `meta.*.pkl`) — the reason
  is not recorded; stacking preserves it.

## Capture mapping (full table, with scope decision)

| tree-sitter capture | current Sublime scope | in this pass |
|---|---|---|
| `(identifier) @variable` | `meta.name.pkl` | **yes** |
| interpolation contents (`@embedded` -> normal scopes) | `meta.name.pkl` (via `main`) | **yes** (follows identifiers) |
| `(classProperty)/(objectProperty) @property` | `meta.property.pkl`, `meta.object.pkl` | no |
| `(parameterList)/(objectBodyParameters) @variable.parameter` | `variable.parameter.pkl` | already aligned |
| `@type` (clazz, typeAlias, `^[A-Z]` idents) | `entity.name.type.pkl`, `support.type.pkl`, `entity.name.class.pkl` | already compatible |
| `@function.method` | `entity.name.function.pkl` | already aligned |
| `(thisExpr)/(outerExpr)/(super) @variable.builtin` | `support.function.built.pkl` | no |
| `@function.method.builtin` (`import`/`read`/`throw`/`trace`) | `keyword.import.pkl`, `keyword.pkl` | no |
| `(moduleExpr) "module" @type.builtin` | `keyword.pkl` / `storage.type.pkl` | no |
| `@operator` (`?? @ = < > ! == ...`) | `keyword.operator.*`; `@` = annotation punctuation | no |
| `@punctuation.delimiter` (`, : . ?.`) | `punctuation.separator.pkl`, `punctuation.accessor.dot.pkl` | already compatible |
| `@punctuation.bracket` | `punctuation.section.*`, `punctuation.definition.generic.*` | already compatible |
| `@escape` | `constant.character.escape.pkl` | already aligned |
| `@string` | `string.quoted.*.pkl` | already aligned |
| `@number` | `constant.numeric.*.pkl` | already aligned |
| `@comment` | `comment.line*`, `comment.block.pkl` | already aligned |
| `@constant.builtin` (`true`/`false`/`null`) | `constant.language.*.pkl` | compatible |
| `@keyword` incl. modifiers | `storage.modifier.pkl` for those | no (keep finer) |

## Proposed pass

### Change 1 — identifier references (the core change)

In the `identifiers` context, the rule that scopes a plain identifier
currently emits `meta.name.pkl`:

```yaml
    - match: '{{base_ident}}'
      scope: meta.name.pkl
      push: after-expression
```

Stack the tree-sitter-compatible scope on top of it, mirroring the
existing quoted-identifier rule (`meta.quoted.pkl variable.other.pkl`):

```yaml
    - match: '{{base_ident}}'
      scope: meta.name.pkl variable.other.pkl
      push: after-expression
```

Order is outermost-to-innermost; `meta.name` stays available to anything
that selects it, and `variable.other` matches tree-sitter themes.

### Change 2 — interpolation contents (verification, likely no edit)

Interpolation bodies are parsed with `include: main` after
`clear_scopes: 1` and `meta_scope: meta.interpolation.pkl`. Because they
include `main`, the identifier rule above applies inside them, so
`\(bird)` will colour `bird` as `variable.other` automatically. This step
is a verification, not a new rule: confirm the inner expression keeps the
identifier scope, and only adjust `clear_scopes`/`meta.interpolation` if
the added scope is stripped.

## Impact analysis

### Rendering
- Identifier references (properties in expressions, module names, lambda
  bodies, `bird`, `general_kenobi`, ...) become themed by tree-sitter
  themes. This is the intended, **visually significant** change.
- Verified motivation: with `bat --theme='Catppuccin Mocha'`, identifiers
  currently render with the default foreground; adding `variable.other`
  gives them Catppuccin's variable colour, matching Neovim.
- No change for themes that style `meta.name` only: that scope is retained.

### Backward compatibility / stacking
- The change is **purely additive**. Every previously emitted scope is
  still emitted, so existing selectors, snippets and themes keep working.
- Precedent: the quoted-identifier rule already stacks
  `meta.quoted.pkl variable.other.pkl`; this change follows the same
  pattern, so the ordering is consistent with the file's conventions.

### Snippets and selectors
- `Snippets/*.sublime-snippet` exclude on `meta.function.parameters`,
  `meta.function-call`, `meta.statement`, `meta.mapping`, `meta.sequence`,
  `meta.set`. Adding a scope does not affect these; several of those meta
  scopes already do not exist, but that is pre-existing and untouched.
- `preferences/Comments.tmPreferences` and `Pkl.sublime-completions` are
  scope-agnostic.

### Other consumers
- LSP/semantic highlighting (`pkl-lsp`) is independent; no interaction.
- `example/pkl.tmLanguage.pkl` is a historical TextMate grammar in Pkl and
  is not active.

### Effort and risk
- Effort: a one-line scope change plus verification of interpolation
  contents. Low.
- Risk: low. The only realistic issue is a theme that styles both
  `meta.name` and `variable.other` with conflicting colours; stacking makes
  the more specific inner scope (`variable.other`) win, which is the
  desired behaviour.
- No automated tests exist; verification is manual (probe theme + a
  tree-sitter-style theme + a plain theme) as used throughout remediation.

## Verification

For each change:
1. Scope dump with a probe theme (map `variable.other` to a distinct
   colour) to confirm identifiers and interpolation contents carry it.
2. Render `example/syntax_test.pkl` with `Catppuccin Mocha` to confirm
   identifiers colour like Neovim.
3. Render with a non-tree-sitter theme (for example `Solarized (dark)`) to
   confirm no regression.

## Deferred (out of scope for this pass)

- Property-declaration scopes (`@property`): `meta.property.pkl` /
  `meta.object.pkl` -> add `variable.other.property.pkl`.
- Builtins: `this`/`outer`/`super` (`@variable.builtin`);
  `import`/`read`/`throw`/`trace` (`@function.method.builtin`).
- Operator/punctuation normalisation, including whether `@` should be an
  operator or annotation punctuation.
- Exact keyword parity (`@keyword` vs our `storage.modifier` split) and
  tree-sitter's `^[A-Z]` -> `@type` heuristic. Kept as-is: our categories
  are more precise and "compatible where it matters" is the agreed bar.
