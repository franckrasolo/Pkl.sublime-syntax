# Pkl syntax highlighting gap analysis

**Next action:** review the findings below

I checked `syntaxes/Pkl.sublime-syntax` against the [0.32.1 language reference](https://pkl-lang.org/main/current/language-reference/index.html), the `apple/pkl` `main` lexer/parser (`pkl-parser/.../Lexer.java`, `ParserImpl.java`), and the release notes. I also verified sample snippets with the installed `pkl 0.31.0`. Two engine-level regex bugs were reproduced with Ruby/Oniguruma, which is the same regex engine Sublime uses.

---

## Findings (ranked)

### Tier 1 — confirmed bugs in current rules

| # | Gap | Evidence | Location |
|---|-----|----------|----------|
| 1 | Pipe `\|>` and union `\|` operators never match. The regex literally contains apostrophes: `'\|>` … `\|'`. | `Parser.Operator.PIPE`; docs §Anonymous Functions; Ruby test: `"a \|> b"` → no match | `Pkl.sublime-syntax:222` |
| 2 | Leading-dot floats (`.23`) never match because the rule starts with `\b` before `.`. | Reference §Floats: `num1 = .23`; Ruby test: `.23` → no match | `Pkl.sublime-syntax:194` |
| 3 | Keyword set is wrong: `match` is highlighted but is **not** a Pkl keyword (it's a valid identifier), while reserved `switch` is missing. | `Lexer.getKeywordOrIdentifier`; docs §Reserved keywords; `pkl eval` rejects `switch`, accepts `match` | `Pkl.sublime-syntax:247` |
| 4 | `unknown` type keyword is not highlighted; `module`/`this` in type position aren't either. | Reference §Unknown Type, §Module Types; `ParserImpl.parseTypeAtom` | `type` context :350-379 |
| 5 | `pklbase_classes_properties` (line 51) and `pklbase_module_properties` (line 49) are **defined but never referenced** (grep count 0), so property access after `.` (`.value`, `.unit`, `.keys`, …) is unscoped. | grep of the syntax file | `after-expression:534-551` |

### Tier 2 — documented language features not covered

| # | Gap | Evidence |
|---|-----|----------|
| 6 | Nullable type marker `?` (`Bird?`, `Bird(nullable)?`) is unscoped; the `type` context has no `?` rule. | Reference §Nullable Types; `parseTypeEnd` |
| 7 | Function types `(A, B) -> C` are mis-scoped as a *type constraint*. | `ParserImpl.parseTypeAtom` FUNCTION_TYPE; 0.30 release note example |
| 8 | Doc comments `///` are just `comment.line.pkl`; no documentation scope. | Reference §Doc Comments |
| 9 | Shebang `#!…` (allowed only at file start since 0.29) is unscoped: `#` unmatched, `!` becomes a logical operator. | 0.29 release note; `Lexer.lexShebang` |
| 10 | Typealias type parameters (`typealias StringMap<Value> = …`) are unscoped. | Reference §Type Aliases; `pkl eval` accepts it |
| 11 | `new` with type arguments (`new Mapping<String, Base> {}`) leaves `<String, Base>` as comparison operators. | Reference §Defining Listings/Mappings |
| 12 | Qualified type names (`com.example.Foo`, `bird.SomeType`) break after the first dot. `baseType` only allows a single ident. | `ParserImpl.parseQualifiedIdentifier` used for DeclaredType |

### Tier 3 — polish / maintenance

| # | Gap | Notes |
|---|-----|-------|
| 13 | Custom string delimiters only support n = 0, 1, 2 pounds; Pkl allows any n ≥ 1. | `strings` contexts :137-191; `Lexer.lexStringStartPounds` loops over `#` |
| 14 | Stdlib symbol lists (`pklbase_classes_methods`, `pklbase_classes_properties`, …) are stale vs current stdlib. | Comments in file show the reflection snippet used to generate them |
| 15 | Commas in parameter/argument lists are unscoped (no `punctuation.separator`). Trailing commas (0.30) then highlight inconsistently. | 0.30 release note |
| 16 | Annotation names `@Foo` are scoped wholesale as `keyword.pkl`; could be `punctuation.definition.annotation` + type. | `super` context :268-270 |
| 17 | Dead code: unused `type`/`baseType` variables (:20-21) and empty `\|\|` alternatives in `pklbase_types` (:41) and `instantiation` (:343). | grep |

Not a gap (verified and **excluded**): block-comment nesting was *removed* in 0.29, so the current non-nesting behavior is correct despite the reference page still saying "nestable". Unparenthesized lambdas (`_ -> x`) are **not** valid Pkl, so no gap there.

---

## Resolution status

All Tier 1 and Tier 2 items are fixed. Tier 3 is fixed except the string
delimiter cap.

| # | Status | Commit(s) |
|---|--------|-----------|
| 1 | Fixed | `8663304` |
| 2 | Fixed | `09e90a5` |
| 3 | Fixed | `9cfb0a2` |
| 4 | Fixed | `aa9b026` |
| 5 | Fixed (stdlib properties after `.`) | `396f106` |
| 6 | Fixed | `fae468c` |
| 7 | Fixed | `c910345`, `93b5772` |
| 8 | Fixed | `4d22ce6` |
| 9 | Fixed | `e12c9e8` |
| 10 | Fixed | `e25df63` |
| 11 | Fixed | `08694c4` |
| 12 | Fixed | `fc17a2c` |
| 13 | Partial: supports n = 0..4; n >= 5 open | `9f7c305` |
| 14 | Fixed (regenerated from `pkl 0.31.0`) | `38fdceb` |
| 15 | Fixed | `e51aea7` |
| 16 | Fixed | `262f80d` |
| 17 | Fixed | `3ac3d4d` |

Changes made during remediation that were not in the original list:

| Change | Commit |
|--------|--------|
| Scope function/lambda parameters as `variable.parameter.pkl` | `1fda13a` |
| Scope type tests with a non-identifier left side (`42 is Int`) | `5d7cfee` |

### Open / deferred

- **String delimiters n >= 5.** Truly arbitrary counts are not expressible
  in TextMate. The Sublime-native parent-capture backreference approach
  panics syntect (`invalid backref number/name`), which would crash `bat`,
  so the cap is four pounds.
- **Tree-sitter scope parity.** Identifiers are scoped `meta.name.pkl`
  rather than `variable`, and interpolation contents likewise, so themes
  built for tree-sitter captures (`@variable`, `@escape`, ...) colour them
  differently. This is a scope-naming pass, not a tokenization gap.

Note on escapes/interpolation: verified against `Lexer.java` and
`apple/tree-sitter-pkl/grammar.js` — a custom-delimiter escape or
interpolation marker is `\` followed by exactly n pounds (`\#` for `#"`,
`\##` for `##"`, ...). For n = 0..4 this now matches the official
behaviour, including leaving wrong-pound-count sequences as literal
content.

