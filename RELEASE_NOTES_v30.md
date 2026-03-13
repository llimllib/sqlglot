# SQLGlot v30 Release Notes

SQLGlot v30 is a major release focused on performance and compilation. Some of the core components of the library are now fully compilable by [mypyc](https://mypyc.readthedocs.io/), delivering significant speedups when installed with the `[c]` extra. This required restructuring several internal modules, which introduces breaking changes for users who depend on internal APIs, subclass parsers, or import from internal paths.

> **If you only use the public API** (`sqlglot.parse`, `sqlglot.parse_one`, `sqlglot.transpile`, `sqlglot.exp.*`, `sqlglot.optimizer.*`), most code will work without changes.

---

## Migration Guide

### 1. Rust tokenizer removed — use `[c]` instead of `[rs]`

The Rust-based tokenizer (`sqlglotrs`) has been removed (since v29) and replaced with a mypyc-compiled C extension (`sqlglotc`).

```bash
# Before
pip install "sqlglot[rs]"

# After
pip install "sqlglot[c]"
```

The `[rs]` extra still installs but is now a deprecated no-op stub. The following APIs are removed:
- `use_rs_tokenizer` parameter and attribute on `Tokenizer`
- `RsTokenizer`, `RsTokenizerSettings`, `RsTokenTypeSettings` imports
- `USE_RS_TOKENIZER` constant from `tokens.py`

### 2. `expressions.py` split into a package

The monolithic `sqlglot/expressions.py` has been split into `sqlglot/expressions/` with submodules:

| Module | Contents |
|---|---|
| `core.py` | `Expr`, `Expression`, `Condition`, `Func`, `AggFunc`, `Column`, `Literal`, etc. |
| `datatypes.py` | `DataType`, `DType`, `DataTypeParam`, `Interval` |
| `query.py` | `Select`, `Query`, `SetOperation`, `UDTF`, `Subquery` |
| `ddl.py` | `Create`, `Alter`, `Drop`, DDL statements |
| `dml.py` | `Insert`, `Update`, `Delete`, `Merge` |
| `properties.py` | All `*Property` classes, `PropertiesLocation` |
| `constraints.py` | All `*ColumnConstraint` classes |
| `math.py` | Arithmetic operators (`Add`, `Sub`, `Mul`, `Div`, etc.) |
| `string.py` | String functions (`Concat`, `Length`, `Upper`, etc.) |
| `temporal.py` | Date/time functions (`DateAdd`, `DateDiff`, etc.) |
| `aggregate.py` | Aggregate functions (`Count`, `Sum`, `Avg`, etc.) |
| `array.py` | Array functions (`ArrayAgg`, `Explode`, etc.) |
| `json.py` | JSON functions (`JSONExtract`, etc.) |
| `functions.py` | Other functions (`Coalesce`, `If`, `Case`, `Cast`, etc.) |
| `builders.py` | Builder helpers (`select()`, `from_()`, `condition()`, etc.) |

**Backwards-compatibility:** `from sqlglot.expressions import *` and `from sqlglot import expressions as exp` still work, because everything is re-exported from `expressions/__init__.py`. However, if you were importing from `sqlglot.expressions` by relying on it being a single file (e.g., inspecting `__file__`), that will break.

### 3. `Parser.expression()` no longer accepts `**kwargs`

This affects anyone subclassing `Parser` or calling `self.expression()` in custom parse methods.

```python
# Before
self.expression(exp.Select, distinct=True, expressions=cols)

# After
self.expression(exp.Select(distinct=True, expressions=cols))
```

The expression instance is now constructed by the caller and passed directly. This eliminates `**kwargs` dict allocation overhead.

### 4. Scope traversal: `bfs` parameter removed

The `bfs` parameter has been removed from all scope traversal functions. Traversal is now always depth-first (DFS).

```python
# Before
scope.walk(bfs=True)
scope.find(exp.Column, bfs=False)
walk_in_scope(expr, bfs=True)

# After
scope.walk()
scope.find(exp.Column)
walk_in_scope(expr)
```

**Behavioral change:** The old default was `bfs=True`. Now traversal is always DFS. Code that depended on BFS ordering from these functions will get results in a different order.

Affected functions: `Scope.walk()`, `Scope.find()`, `Scope.find_all()`, `walk_in_scope()`, `find_in_scope()`, `find_all_in_scope()`.

### 5. Dialect metaclass no longer mutates Parser token sets

Previously, the `_Dialect` metaclass dynamically modified parser token sets (`ID_VAR_TOKENS`, `TABLE_ALIAS_TOKENS`, `NO_PAREN_FUNCTIONS`) at class creation time based on dialect flags like `SUPPORTS_SEMI_ANTI_JOIN`. These mutations are removed — each parser now declares its token sets statically.

- `Dialect.SUPPORTS_SEMI_ANTI_JOIN` has been removed.
- `SHOW_TRIE` / `SET_TRIE` are no longer auto-computed from `SHOW_PARSERS` / `SET_PARSERS`.

### 6. Use `Expr` instead of `Expression` for generic `isinstance` checks

Base classes like `Func`, `Condition`, `Binary`, and other traits now inherit from `Expr` directly, not from `Expression`. This means `isinstance(node, exp.Expression)` will **not** match these trait classes. If your code uses `isinstance` to check for "any AST node", switch to `exp.Expr`:

```python
# Before
isinstance(node, exp.Expression)

# After
isinstance(node, exp.Expr)
```

### 7. Compiled classes cannot be subclassed (when using `[c]`)

When `sqlglot[c]` is installed, many core classes are compiled via mypyc. **Compiled classes cannot be subclassed at runtime** — class definition succeeds, but instantiation raises `TypeError: interpreted classes cannot inherit from compiled`.

**Affected classes (compiled):**

| Class | Subclassable? |
|---|---|
| All parsers (`BigQueryParser`, `SnowflakeParser`, etc.) | No |
| `Parser` (base) | No |
| `Expression`, `Expr`, and all AST nodes (`Select`, `Column`, `Func`, etc.) | No |
| `MappingSchema`, `AbstractMappingSchema` | No |
| `Scope` | No |
| Optimizer rules (`scope.py`, `qualify.py`, `qualify_columns.py`, etc.) | No |

**Not compiled (still subclassable):**

| Class | Subclassable? |
|---|---|
| `Generator` and all dialect generators | Yes |
| `Tokenizer` and all dialect tokenizers | Yes |
| `Dialect` and all dialect classes | Yes |

If you need to subclass compiled classes (parsers, expressions, schema, etc.), install the pure Python version instead:

```bash
pip install sqlglot        # pure Python — full subclassing support
pip install "sqlglot[c]"   # compiled — faster, but no subclassing
```
