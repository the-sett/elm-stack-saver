# elm-stack-saver

A Python tool to find recursive cycles and closure-capture bugs (which can lead to stack overflow) in compiled Elm JavaScript files.

## Why?

Elm compiles to JavaScript, and this tool helps identify two classes of problems:

1. **Recursive cycles** that might cause stack overflows if not tail-call optimized:
   - Self-recursive functions
   - Mutual recursion cycles (A calls B, B calls C, C calls A)

2. **Closure-capture bugs** in tail-call-optimized code ([elm/compiler#2268](https://github.com/elm/compiler/issues/2268)):
   - When a closure inside a TCO `while(true)` loop captures a `var` variable
   - All loop iterations share the same variable reference due to JavaScript's function-scoped `var`
   - This causes all closures to see the final value instead of their iteration's value

## How it works

**For recursion detection:**
1. Parses the JavaScript file to extract all function definitions
2. Builds a complete function call graph
3. Uses Tarjan's algorithm to find strongly connected components (cycles)
4. Converts JavaScript names back to readable Elm-style names

**For closure-capture detection:**

The tool finds closures that match ALL of these conditions:

| Condition | JavaScript Pattern |
|-----------|-------------------|
| TCO Loop | `label: while (true) {` with `continue label;` inside |
| Loop-level var | `var x = ...;` not inside a nested function |
| Closure captures var | `function () { ...x... }` references the var |
| Not an IIFE | No `}()` immediately after the function |
| Not in return | Closure is not part of a `return` statement |
| Continue reachable | `continue label;` is in the same branch as the closure |

**Safe patterns (not flagged):**
- Closures that are returned (not accumulated across iterations)
- IIFEs (immediately invoked function expressions)
- Closures where `continue` is in a different branch

## Installation

Requires Python 3.9+. No external dependencies.

```bash
git clone https://github.com/the-sett/elm-stack-saver.git
cd elm-stack-saver
```

## Usage

```bash
python ess.py <path-to-js-file>
```

By default, the tool runs both closure-capture detection and recursion analysis.

### Example

```bash
python ess.py elm.js
```

Output includes both analyses:

```
Analyzing: elm.js

Checking for closure-capture bugs in TCO loops (strict mode)...

================================================================================
CLOSURE-CAPTURE BUGS IN TCO LOOPS
================================================================================

These are locations where a closure inside a while(true) loop captures
a `var` variable. Due to JavaScript's function-scoped `var`, all iterations
share the same variable, causing incorrect behavior.

See: https://github.com/elm/compiler/issues/2268

No closure-capture bugs found! ✓

Building call graph...
Found 1234 functions
Call graph has 5678 edges

Finding cycles (strongly connected components)...
================================================================================
COMPLETE LIST OF RECURSIVE CYCLES
================================================================================

## SELF-RECURSIVE FUNCTIONS (42 total)

  elm/core List.foldl (line 1234)
  elm/core Dict.removeMin (line 2345)
  ...

## MUTUAL RECURSION CYCLES (3 total)

  Cycle 1 (2 functions):
    - elm/json Decode.decodeValue (line 3456)
    - elm/json Decode.runHelp (line 3478)

  ...

================================================================================
SUMMARY: 42 self-recursive + 3 mutual recursion cycles
         (48 total functions involved)
================================================================================
```

### Options

| Option | Description |
|--------|-------------|
| `--jsnames` | Display original JavaScript function names instead of Elm names |
| `--closures-only` | Only check for closure-capture bugs (skip recursion analysis) |
| `--no-closures` | Skip closure-capture bug detection |
| `--lenient` | Use lenient detection (more false positives, includes pattern-matching loops) |
| `--csv FILE` | Write closure-capture bugs to a CSV file |

```bash
# Just check for closure bugs
python ess.py elm.js --closures-only

# Just check for recursion (original behavior)
python ess.py elm.js --no-closures

# Export findings to CSV
python ess.py elm.js --closures-only --csv bugs.csv
```

## Name conversion

The tool automatically converts JavaScript names to readable Elm names:

| JavaScript | Elm |
|------------|-----|
| `$elm$core$Dict$removeMin` | `elm/core Dict.removeMin` |
| `$author$project$Main$view` | `Main.view` |
| `_Utils_cmp` | `_Utils_cmp` (runtime) |
| `loop` | `loop` (local helper) |

## License

BSD-3-Clause
