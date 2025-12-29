# elm-stack-saver

A Python tool to find all recursive cycles in compiled Elm JavaScript files.

## Why?

Elm compiles to JavaScript, and recursive functions can cause stack overflow errors if they aren't properly tail-call optimized. This tool helps you identify:

1. **Self-recursive functions** - functions that call themselves directly
2. **Mutual recursion cycles** - groups of functions that call each other in a cycle (A calls B, B calls C, C calls A)

By finding these patterns in your compiled Elm output, you can identify which functions might need refactoring to avoid stack overflows.

## How it works

The tool:
1. Parses the JavaScript file to extract all function definitions
2. Builds a complete function call graph
3. Uses Tarjan's algorithm to find strongly connected components (cycles)
4. Converts JavaScript names back to readable Elm-style names

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

### Example

Analyze your compiled Elm application:

```bash
python ess.py elm.js
```

Output:

```
Analyzing: elm.js
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

- `--jsnames` - Display original JavaScript function names instead of Elm names

```bash
python ess.py elm.js --jsnames
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
