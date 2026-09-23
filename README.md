# regexp

`ecosystem::regexp::syntax` parses a bounded regular expression language, exposes its syntax tree, simplifies structural wrappers, and compiles it to an inspectable Thompson NFA. `ecosystem::regexp` searches UTF-8 text with that NFA, captures groups, replaces matches, and splits text. Neither package uses the host Go regexp engine or Logos token selection rules.

```toml
[dependencies]
"ecosystem::regexp" = "0.1.0"
```

```gom
use ecosystem::regexp::syntax;

fn main() -> () {
    let parsed = syntax::parse("^(go|goml)[0-9]+$").unwrap();
    println(parsed.capture_count.to_string());
    let program = syntax::compile("^(go|goml)[0-9]+$").unwrap();
    println(program.state_count().to_string());
}
```

`parse` and `parse_with` return `Parsed { expression, capture_count }`. `simplify` flattens nested concatenations and alternations, removes empty concatenation members, and reduces exact zero and one repetitions. It preserves capture numbers and assertions. `compile` and `compile_with` parse, simplify, then return `Program { states, start, capture_count }`; states and class terms use immutable vectors. `State` distinguishes ordered epsilon splits, character consumption, assertions, capture start/end tags, and acceptance. Split order preserves the pattern's alternative order and greedy repetition branch first.

## Matching

```gom
use ecosystem::regexp;

fn main() -> () {
    let regex = regexp::Regex::compile("(go|goml)[0-9]+").unwrap();
    let found = regex.find("try goml42").unwrap().unwrap();
    println(found.text("try goml42"));
    println(regex.replace_all("go1 goml42", "[$1]").unwrap());
}
```

`Regex::compile` and `compile_with` return syntax diagnostics. `find`, `find_from_with`, and `find_all` return matches; `replace_all` expands `$0` for the entire match, `$1` through `$9`, `${n}` for larger capture numbers, and `$$` for a literal dollar sign. An unmatched optional capture expands to empty text. Invalid references and template syntax return `InvalidReplacement`. `split` returns the segments between nonoverlapping matches. Every operation has a `*_with` variant that accepts `regexp::Limits`.

Search selects the earliest start and then the longest end at that start. Equal-span capture histories keep the first ordered NFA path. This is explicit leftmost-longest behavior, not Go's default leftmost-first submatch policy. `Match.span` and every capture span use half-open UTF-8 byte offsets. Capture zero is the whole match; an unmatched group is `None`. `Match::text` and `capture` take the original input string for extraction. `find_from_with` rejects offsets inside a UTF-8 scalar.

The NFA keeps at most one thread per state at each input position and remembers capture tags. Epsilon cycles are visited once per position. `find_all` uses nonoverlapping matches. An empty match advances by one full Unicode scalar; an empty match immediately adjacent to the end of a preceding nonempty match is omitted. Replacement includes valid empty matches at the beginning and end. Split ignores empty delimiters at those two edges. `^`, `$`, `\A`, and `\z` assert absolute text boundaries; `\b` and `\B` compare ASCII word characters on each side.

`regexp::Limits::new()` allows 1 MiB input, 10 million charged work units, 100,000 matches, and 16 MiB output. A replacement template is also limited to the input byte cap. Work includes NFA state scans, class terms, capture copies, and per-position state bookkeeping. All matches in `find_all`, replacement, and split share one work budget. That budget is additionally capped by a multiple of input bytes, program states, capture slots, and class terms. Thus an operation that would revisit long suffixes returns `WorkLimit` within a linear bound on input length for a fixed compiled program. The API does not claim every valid pattern and input pair completes under the default limits. `RuntimeError` distinguishes invalid limits and offsets from input, work, match, output, and replacement failures.

The supported grammar includes UTF-8 literals, concatenation, alternation, capturing groups numbered in opening order, `(?:...)`, `.`, positive and negated character classes, ranges, `?`, `*`, `+`, `{n}`, `{n,}`, and `{n,m}`. Dot excludes LF. `^`, `$`, `\A`, and `\z` mean absolute text boundaries; `\b` and `\B` are word-boundary assertions for the future matcher. The shorthand classes `\d`, `\w`, and `\s` and their uppercase complements use ASCII definitions: `[0-9]`, `[A-Za-z0-9_]`, and ASCII whitespace. Recognized Unicode properties are `L`/`Letter`, `N`/`Number`, `White_Space`, `Lowercase`, `Uppercase`, and `ASCII`, with `\p{...}` and `\P{...}` forms. Escapes include `\n`, `\r`, `\t`, `\f`, `\v`, `\a`, `\0`, `\xHH`, `\uHHHH`, `\u{H...}`, and escaped punctuation.

Lookarounds, backreferences, lazy or stacked quantifiers, inline flags, named captures, class set operations, nested classes, and other properties return `Error { offset, message }`. Offsets are UTF-8 byte offsets at the point where invalid syntax was detected. Invalid Unicode scalars, reversed ranges, and malformed repetitions also fail. The compiler reports state and expansion failures at offset zero because they apply to the whole expression. A syntax tree can be inspected by callers; compilation always reparses the original pattern, so callers cannot inject unchecked syntax nodes into a program.

`syntax::Limits::new()` allows 65,536 pattern bytes, 64 group levels, repeat counts up to 1,024, 1,024 captures, 16,384 parsed nodes, and 16,384 NFA states. `parse_with` and `compile_with` accept smaller or larger explicit limits within validated absolute ceilings. Compilation also bounds expansion work to eight times the state limit. These bounds prevent repeated expressions and deep nesting from exhausting the compiler.

From the repository root, run `just ecosystem-test regexp` to check formatting, library tests, the independent consumer, and a cached build.
