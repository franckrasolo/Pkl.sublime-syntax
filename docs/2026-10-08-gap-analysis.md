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
