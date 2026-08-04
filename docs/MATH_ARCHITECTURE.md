# math_rep Architecture

`math_rep` is the core mathematical IR of *formalite*.  It defines a typed expression tree for representing mathematical and logical formulas, which the rest of the system manipulates (rewrites, skolemizes, pattern-matches) and ultimately lowers to a language-specific code representation via `codegen`.

---

## Glossary

**Expression**
Any instance of `FormalContent` — the single base class for everything the system can represent.  The word covers both value-producing nodes (`Term`) and boolean-valued nodes (`Condition`), as well as structural nodes such as `FunctionDefinitionExpr`, `ClassDefinitionExpr`, and `MathModule`.  Expressions form an immutable, interned tree; structural traversal goes through `arguments()` and structural replacement through `with_argument()`/`with_arguments()`.

**Type**
An instance of `MType` that describes the kind of value an expression produces.  Types are carried on `FormalContent.type` and on every `QualifiedName`.  They drive codegen (e.g., choosing a Java boxed type), guide the subtype checker (`is_subtype`), and may constrain rewrite rules.  The special values `M_ANY` and `M_UNKNOWN` are used when no tighter type is known; `M_BOTTOM` denotes an empty type.

**Qualified Name**
A `QualifiedName` is a name together with a *lexical path* — a tuple of scope identifiers (innermost first) that places the name in a specific scope.  Two qualified names are equal only if both their `name` and `lexical_path` match, so `+` in the `*math*` scope is a distinct entity from a Java or Python `+`, and may have a different type and different semantics.  Qualified names act as the identity tokens for variables, operators, functions, and class members throughout the IR.

**Frame**
A named lexical scope, identified by a lexical path, and representing lexical inclusion in programming languages such as C and Python.  Every `QualifiedName` belongs to a frame via its `lexical_path`.  Special frames represent fixed concepts, and by convention have a lexical path whose first component is marked with asterisks at the beginning and the end.  The `*math*` frame (defined in `math_frame.py`) owns all built-in arithmetic, relational, and logical operators; the `*var*` frame owns generated variable names.  User-defined names carry the dotted module path of the source file as their frame.  The frame concept allows the same surface string to name different things in different scopes without collision.

**Dataflow Analysis**
In this codebase, *dataflow analysis* refers to the propagation-based system in `dataflow.py` that tracks how information (such as types) flows through the expression graph.  A `DataFlow` object encodes a directed dependency from a *source* expression to a *target* expression or domain-info object; when the source's domain changes, the propagator's `propagate()` method is called to derive updated information for the target.  The analysis runs until no propagator produces new information (a fixed point).

**Constraint Propagation**
The specific algorithm realised by the dataflow system: starting from known bounds on some expressions, repeatedly fire `DataFlow` propagators to narrow the feasible domain of related expressions.  Each propagator encodes one inference rule (e.g., "if $x ≤ 5$ and $y = x + 1$ then $y ≤ 6$").  The process terminates when the agenda is empty (fixed point reached) or a domain becomes empty (infeasibility detected).  This is used in optimization and constraint-solving pipelines that consume the `math_rep` IR.

**Domain Table**
The runtime data structure managed by `AbstractDFlowTable`.  It maps each live `FormalContent` node to an *application-specific info* object (`appl_info`) that holds the current application-specific information for that expression.  The table also maintains an *agenda* — an ordered set of expressions some of whose information has changed and whose outgoing propagators must be re-fired.  `AbstractDFlowTable.build(term)` seeds the table by visiting an expression tree; `add_to_agenda(expr)` schedules re-propagation.  Concrete subclasses supply the domain representation and propagator logic.

---

## Module map

| File | Responsibility |
|---|---|
| `expression_types.py` | Type system (`MType` hierarchy) and `QualifiedName` |
| `constants.py` | Unicode symbol constants (∃ ∀ ∧ ∨ …) |
| `math_frame.py` | Lexical-scope frame markers for built-in operators |
| `math_symbols.py` | Pre-built `QualifiedName` instances for every built-in operator |
| `expr.py` | All expression node classes (`FormalContent` hierarchy) |
| `dataflow.py` | Abstract constraint-propagation / domain-table infrastructure |

---

## Type system (`expression_types.py`)

All types are subclasses of the abstract `MType`.

```
MType
├── MAtomicType          – Boolean, Number, String, Integer, int16/32/64, float, …
├── MClassType           – user-defined class (wraps a QualifiedName)
├── WithMembers (abstract)
│   ├── MSetType         – Set[T]
│   ├── MStreamType      – Stream[T]  (lazy sequence)
│   ├── MCollectionType  – Collection[T]
│   └── MArray           – T[dim₀][dim₁]…
├── MFunctionType        – (A₀, A₁, …) → R  (arity: fixed | '?' | '*')
├── MUnionType           – Union[T₀, T₁, …]
├── MTupleType           – Tuple[T₀, T₁, …]
├── MRange               – integer range [start, stop)
├── MMappingType         – Mapping[K → V]  (total or partial)
├── MRecordType          – Record[(name₀, T₀), …]
└── DeferredType         – placeholder for an as-yet-unknown dimension
```

**Pre-defined singleton types** (module-level constants):
`M_BOOLEAN`, `M_NUMBER`, `M_INT`, `M_INT16/32/64`, `M_FLOAT`, `M_DOUBLE`,
`M_STRING`, `M_NONE`, `M_ANY`, `M_UNKNOWN`, `M_BOTTOM`, `M_VOID`,
`M_FIXED_POINT`, `M_TYPE`.

**Subtype relation** — `is_subtype(a, b)` handles:
- identity and `M_ANY`/`M_BOTTOM` boundary cases
- numeric widening chain: `M_BOOLEAN ⊆ M_INT16 ⊆ … ⊆ M_NUMBER`
- covariant container types, contra/covariant function types, range inclusion

**Type utilities**:
- `type_intersection(*types)` — greatest lower bound
- `type_of(value)` — infer `MType` from a Python literal

### QualifiedName

`QualifiedName` is a frozen dataclass representing a named entity (variable, operator, function, …):

```python
@dataclass(frozen=True)
class QualifiedName:
    name: str | tuple[str, ...]   # identifier or multi-word name
    type: MType                   # declared type (default M_ANY)
    lexical_path: tuple[str, ...] # scope chain (innermost first)
```

Two `QualifiedName` objects are equal iff both `name` and `lexical_path` are equal (type is excluded from equality/hash).  Helper methods: `with_path`, `with_type`, `with_extended_path`, `with_extended_name`, `describe`.

---

## Expression hierarchy (`expr.py`)

All nodes inherit from **`FormalContent`**, which is the abstract base for everything the system can represent.

```
FormalContent (Acceptor, metaclass=MetaPatternable)
├── Condition (abstract)  – evaluates to Boolean
│   ├── Negation
│   ├── LogicalOperator          – AND / OR / IMPLIES (n-ary)
│   ├── LogicalOperatorAsExpression  – logical op lifted to Term context
│   ├── Comparison               – lhs OP rhs  (=, ≠, <, ≤, >, ≥)
│   ├── SetMembership            – expr ∈ container
│   ├── TemporalOrder            – temporal Allen-relation
│   ├── PredicateAppl            – pred(arg₀, arg₁, …)
│   └── DefinedBy                – dependency annotation (reduces to True)
│
├── Term (abstract)              – evaluates to a value
│   ├── Atom                     – named constant (words-tuple + lexical path)
│   │   └── Predicate            – predicate name (may be negated)
│   ├── MathVariable             – single named variable  ($name)
│   ├── MathVariableArray        – array of variables  ($$name[dims…])
│   ├── IndexedMathVariable      – element of a MathVariableArray
│   ├── Quantity                 – literal value (bool/int/float/str/None)
│   ├── StringTerm               – string literal
│   ├── Attribute                – attr(container)
│   ├── Subscripted              – obj[sub₀, sub₁, …]
│   ├── FunctionApplication      – f(arg₀, arg₁, …)  [or method target.f(…)]
│   ├── LogicalCombination       – logical combo to be lifted (Term level)
│   ├── Aggregate                – Σ / π / SET  term  FOR vars IN container
│   ├── Quantifier               – ∀/∃ vars ∈ container. formula
│   ├── IFTE                     – cond ? pos : neg
│   ├── LambdaExpression         – (vars) -> body
│   ├── Cast                     – CAST(type; term)
│   ├── Stream                   – STREAM(term  FOR … IN …)
│   ├── Between                  – lower ≤ value ≤ upper
│   ├── GeneralSet               – { e₀, e₁, … }
│   ├── GeneralSequence          – [ e₀, e₁, … ]
│   ├── Percentage               – pct% OF whole
│   ├── RangeExpr                – Range(start, stop)
│   ├── RealSpace                – ℝⁿ
│   ├── DomainDim                – deferred dimension (resolves on type info)
│   ├── Epsilon                  – infinitesimal ε
│   ├── LetExpr                  – LET var = value IN body
│   └── DummyTerm                – debugging placeholder, must not reach codegen
│
├── ComprehensionElement (abstract)
│   ├── ComprehensionContainer   – FOR vars IN container [rest]
│   └── ComprehensionCondition   – S.T. condition [rest]
│
├── FunctionDefinitionExpr       – DEFINE name(params): return_type = body
├── ClassDefinitionExpr          – CLASS name(superclasses): fields + defs
├── BodyExpr                     – body container (doc + defs + optional value)
├── MathTypeDeclaration          – var: type  (field/parameter declaration)
├── InitializedVariable          – var = init
├── NamedArgument                – name=expr
├── MathModule                   – top-level compilation unit
├── Package                      – hierarchical package/module name
├── TypeAlias                    – ALIAS name AS type
└── TypeExpr                     – encapsulates an MType as a FormalContent
```

### FormalContent API

Every node implements:

| Method | Purpose |
|---|---|
| `describe(parent_binding)` | Produce a human-readable string; `parent_binding` adds parentheses when needed |
| `arguments()` | Return child nodes (enables generic tree traversal and rewriting) |
| `with_argument(index, arg)` | Return a copy with one child replaced (structural immutability) |
| `with_arguments(args)` | Bulk replacement (default delegates to `with_argument`) |
| `substitute(substitutions)` | Recursively replace `QualifiedName → FormalContent` |
| `skolemize(universals, factory)` | Eliminate existential quantifiers via Skolem functions |
| `to_code_rep()` | Lower to a `codegen.abstract_rep` node for code generation |
| `operator()` | Return the distinguishing operator/key of this node (used by pattern matching) |
| `is_eq(other)` | Deep structural equality (separate from `__eq__` which builds a `Comparison`) |

`Term` overloads Python arithmetic and comparison operators so that ordinary
Python expressions build `math_rep` nodes rather than evaluating numerically:

| Python syntax | Produces |
|---|---|
| `a + b`, `a - b`, `a * b`, `a / b` | `FunctionApplication(PLUS/MINUS/TIMES/DIV_QN, …)` |
| `a @ b` | `FunctionApplication(MATMUL_QN, …)` |
| `-a` | `FunctionApplication(MINUS_QN, (a,))` |
| `a < b`, `a <= b`, `a > b`, `a >= b` | `Comparison(a, LT/LE/GT/GE_QN, b)` |
| `a == b`, `a != b` | `Comparison(a, EQ/NEQ_QN, b)` — note: does *not* return `bool` |

`Condition` overloads `&`, `|`, `~` for AND / OR / NOT.

This means the same operator symbols serve two roles: as Python syntax for
constructing the IR, and as `QualifiedName` tokens inside the IR.  There is
no conflict because the Python-level operators are defined on `Term`/`Condition`
objects, not on raw Python numbers or booleans.

---

## Key design patterns

### Interning

Most expression types are decorated with `@intern_expr` (backed by `codegen.utils.Intern`, a `WeakValueDictionary` keyed by a structural hash of constructor arguments).  Structurally identical expressions share a single object in memory, so `a is b` is a reliable fast-path equality check.

### Smart construction via `__new__`

Several classes override `__new__` to perform algebraic simplification at construction time before `__init__` is ever called:

- `LogicalOperator` — absorbs identity elements (True in AND, False in OR), flattens single-element conjunctions/disjunctions
- `FunctionApplication` — cancels zero-terms in `+`, identity-terms in `*`, simplifies constant folding for multi-`Quantity` sums
- `MUnionType` — collapses to a single type when all components are subtypes of one another
- `Quantifier(∀, True, …)` — short-circuits to `True`

### `@disown` decorator

Marks constructor parameters as "owned" key fields.  When combined with the intern cache, this prevents re-initialization of already-interned objects; `__init__` guards against double-initialization via the `already_initialized` sentinel.

### Visitor dispatch (`Acceptor`)

`FormalContent` mixes in `Acceptor`.  When a subclass is defined, `Acceptor.__init_subclass__` automatically generates an `accept(visitor)` method that calls `visitor.visit_<snake_case_classname>(self)`.  External visitors (e.g., the Java code generator) implement one `visit_*` method per node type.

### Pattern matching (`MetaPatternable`)

`FormalContent`'s metaclass is `MetaPatternable`, which implements `__class_getitem__` so that class-subscription syntax constructs a `CompoundPattern`:

```python
# Match any FunctionApplication with operator '+' and two arguments
FunctionApplication['+', 'lhs', 'rhs']
```

`CompoundPattern`, `PatternVariable`, `ClassPattern`, `Let`, `OneOf`, `OrPattern` are defined in `rewriting/patterns.py`.  Patterns match against `operator()` and `arguments()`.

### `QualifiedName.__call__` injection

This is a deliberate special case: after `expr.py` finishes defining all
expression classes, it monkey-patches `QualifiedName.__call__ = qn_call`.
This cannot be done inside `expression_types.py` (where `QualifiedName` is
defined) because `FunctionApplication` does not exist there yet — the two
modules have a mutual dependency that is broken by deferring the patch.

The effect is that any `QualifiedName` can be called like a function:

```python
PLUS_QN(a, b)          # → FunctionApplication(PLUS_QN, (a, b))
ZIP_QN(x, y, z, …)        # → FunctionApplication(ZIP_QN, (x, y, z, …))
```

This complements the Python operator overloads: operators with dedicated
syntax use `__add__` etc., while named functions and user-defined operators
use the call syntax directly on their `QualifiedName`.

---

## Comprehensions and quantifiers

Comprehensions are represented as a linked-list chain of `ComprehensionElement` nodes:

```
ComprehensionContainer(vars, container, rest?)
    → ComprehensionCondition(condition, rest?)
        → ComprehensionContainer(…)  # for multi-range comprehensions
```

A `ComprehensionContainer.substitute_bound_vars()` call renames bound variables to avoid capture, using `FormalContent.fresh_name` to generate unique names.

`Aggregate` and `Stream` own a `ComprehensionContainer`; `Quantifier` does the same and additionally calls `substitute_bound_vars` in its constructor to enforce hygienic scoping.

---

## Dataflow / constraint propagation (`dataflow.py`)

`dataflow.py` provides the abstract skeleton for domain-propagation solvers that operate over the expression graph:

- **`AbstractDFlowTable`** — manages an agenda of `FormalContent` nodes whose application-specific information need re-processing; resets the intern caches on construction so the same expression objects can carry mutable `appl_info`.
- **`AbstractDFlowBuilder`** — visits expressions to initialize propagators.
- **`DataFlow`** — a directed propagator from a `source` Term to a `target` (Term or `ApplicationSpecificInfo`).  Registers itself on `source.appl_info` and queues the source on the agenda.  Subclasses implement `propagate(table)`.
- **`DomainInfoWithDataFlow`** — mixin for per-expression domain objects that tracks active propagators, pending additions, and deactivated propagators.

---

## Integration with `codegen`

Every `FormalContent` node implements `to_code_rep()`, which returns an object from `codegen.abstract_rep` (e.g., `ComparisonExpr`, `FunctionApplExpr`, `LogicalExpr`).  `MathModule.to_code_rep()` produces a `CompilationUnit`, which is the entry point for language-specific generators (e.g., `codegen/java/java_generator.py`).

The pipeline is:

```
Python source / DSL
       │
       ▼
math_rep expression tree
       │  rewriting/rules.py (term rewriting)
       ▼
math_rep expression tree (normalized)
       │  FormalContent.to_code_rep()
       ▼
codegen.abstract_rep tree
       │
       ├─── codegen/java/java_generator.py  →  Java source
       │
       └─── (other backends)
                Python source       [see the optimistic project]
                IBM CPLEX OPL DSL   [see the optimistic project]
                …
```

The `codegen.abstract_rep` layer is intentionally backend-agnostic: every
node knows only its abstract structure (e.g., `FunctionApplExpr`,
`ComparisonExpr`), not the target language.  This repository ships the Java
backend; Python and IBM CPLEX OPL generation live in the separate
[optimistic](https://github.com/IBM/optimistic) project, which consumes the
same `abstract_rep` tree.

---

## Built-in operator catalog

Defined in `math_symbols.py` as `QualifiedName` constants in the `*math*` lexical scope:

| Category | Symbols |
|---|---|
| Arithmetic | `+` `-` `*` `÷` `/` `%` `^` `@` (matmul) |
| Comparison | `=` `≠` `<` `≤` `>` `≥` |
| Logical | `∧` `∨` `⇒` `¬` |
| Aggregate | `Σ` (sum) `π` (product) |
| Quantifier | `∀` `∃` |
| Set/Stream | `∈` `∉` `∩` `∪` `zip` `range` `make-stream` `stream-concatenate` `make-array` |
| Math functions | `min` `max` `abs` `ceil` |
