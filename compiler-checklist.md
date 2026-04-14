# BBL Compiler Checklist

> You are building a compiler for BBL (Best Business Language) — a statically-typed,
> imperative + OOP language with functional features, designed for business logic by
> someone who was driven to the edge by ABAP. The compiler is written in C and emits
> bytecode that runs on a custom virtual machine (the BBLVM), modeled after the JVM
> architecture. This document is the step-by-step build order.

---

## Folder Map

```
bbl/
├── compiler/               # The BBL compiler (C)
│   ├── src/
│   │   ├── lexer/          # Phase 1: Tokenizer
│   │   ├── parser/         # Phase 2: Parser → AST
│   │   ├── ast/            # Phase 3: AST node definitions
│   │   ├── sema/           # Phase 4: Semantic analysis & type checker
│   │   ├── codegen/        # Phase 6: Bytecode emitter
│   │   └── util/           # Shared: arena allocator, string interning, hash map
│   ├── include/            # Public headers for all compiler modules
│   └── tests/              # Unit tests for compiler phases
│
├── vm/                     # The BBLVM virtual machine (C)
│   ├── src/
│   │   ├── bytecode/       # Phase 5: Bytecode format definition, loader, verifier
│   │   ├── core/           # Phase 7: Execution loop, instruction dispatch
│   │   ├── memory/         # Phase 8: Garbage collector, heap management
│   │   ├── runtime/        # Runtime object model, value representation
│   │   └── stdlib/         # Phase 9: Native implementations of stdlib functions
│   ├── include/
│   └── tests/
│
├── stdlib/                 # BBL standard library (written in BBL once it runs)
│   ├── core/               # Option, Result, basic types
│   ├── collections/        # List, Map, Set, business tables
│   ├── io/                 # File, network, database I/O
│   ├── text/               # String processing, regex, formatting
│   └── time/               # Date/time, durations, timezones
│
├── cli/                    # Phase 10: `bbl` CLI entry point
│   └── src/
│
├── tools/                  # Dev tooling (post-compiler)
│   ├── fmt/                # bbl fmt — formatter
│   ├── lint/               # bbl lint — linter
│   ├── lsp/                # Language Server Protocol implementation
│   └── repl/               # Interactive REPL
│
├── spec/                   # Language spec docs, grammar (EBNF), type rules
├── examples/               # BBL example programs (hello world → batch jobs)
├── tests/
│   ├── programs/           # Integration test: .bbl source files
│   └── expected/           # Expected output / bytecode for each test
└── docs/                   # End-user documentation
```

---

## Phase 0 — Project Infrastructure

> `compiler/`, root `CMakeLists.txt` or `Makefile`

These are the boring things you build once and never want to think about again.

- [ ] **0.1** Choose a build system — CMake is the pragmatic call for a multi-target C
      project with a compiler and a VM as separate binaries. Create root
      `CMakeLists.txt` with subdirectory targets for `compiler/` and `vm/`.
- [ ] **0.2** Set up a C standard — target **C11** minimum. You need `_Generic`,
      anonymous structs, and `stdint.h` throughout.
- [ ] **0.3** Implement an **arena allocator** in `compiler/src/util/arena.c`.
      The compiler will allocate thousands of AST nodes — individual `malloc`/`free`
      calls are both slow and painful to manage. An arena lets you free an entire
      compilation unit in one shot. This is not optional.
- [ ] **0.4** Implement a **string intern table** in `compiler/src/util/intern.c`.
      Identifiers and keywords will appear thousands of times. Interning means every
      occurrence of `"balance"` points to the same char*, so identifier comparison is
      a pointer compare, not `strcmp`.
- [ ] **0.5** Implement a **hash map** in `compiler/src/util/map.c` — open addressing,
      string keys. You will use this for symbol tables, intern tables, and the type
      registry. Do not reach for a third-party library; a basic Robin Hood hash map
      in ~200 lines of C is sufficient.
- [ ] **0.6** Set up a **test harness** — a minimal test runner that compiles and
      runs `.bbl` files in `tests/programs/` and diffs against `tests/expected/`.
      This is your regression net for every phase that follows.

---

## Phase 1 — Lexer (Tokenizer)

> `compiler/src/lexer/`

The lexer reads raw source text and produces a flat stream of tokens. It knows
nothing about grammar — that is the parser's problem.

- [ ] **1.1** Define the **token type enum** in `compiler/include/token.h`.
      Every token kind gets a named constant: `TOK_IDENT`, `TOK_INT_LIT`,
      `TOK_STRING_LIT`, `TOK_PLUS`, `TOK_SEMICOLON`, `TOK_KEYWORD_IF`, etc.
      Include a `TOK_EOF` sentinel. Keywords get their own token types — do not
      leave them as bare identifiers for the parser to sort out.

- [ ] **1.2** Define the **Token struct**:
      ```c
      typedef struct {
          TokenType type;
          const char *start;   // pointer into source buffer (interned for identifiers)
          int        length;
          int        line;
          int        col;
      } Token;
      ```

- [ ] **1.3** Implement the **Lexer struct** and `lexer_init(Lexer*, const char* src)`.
      Store current position, line/col tracking, and a pointer to the intern table.

- [ ] **1.4** Implement `lexer_next_token(Lexer*)` — the core scan loop:
      - Skip whitespace and comments (`//` line comments, `/* */` block comments)
      - Handle single-char tokens: `{`, `}`, `(`, `)`, `[`, `]`, `;`, `,`, `.`
      - Handle multi-char operators: `==`, `!=`, `<=`, `>=`, `->`, `=>`, `::`, `..`
      - Handle integer literals (decimal, `0x` hex, `0b` binary)
      - Handle float/decimal literals — pay attention to BBL's `decimal` type for
        currency values; a literal like `199.99d` should produce `TOK_DECIMAL_LIT`
      - Handle string literals with escape sequences (`\n`, `\t`, `\\`, `\"`)
      - Handle multi-line strings (triple-quote `"""..."""` or backtick style)
      - Handle identifiers: scan alphanumeric + `_`, then check intern table
        against the keyword list to emit the right `TOK_KEYWORD_*` type

- [ ] **1.5** Implement **keyword recognition** — maintain a static table of
      BBL keywords mapped to their token types. Checked after every identifier scan.

- [ ] **1.6** Implement **lexer error reporting** — unknown characters, unterminated
      strings, and malformed number literals should emit a structured diagnostic
      (file, line, col, message) rather than calling `exit()`. The error type should
      be returnable so the lexer can attempt recovery and continue scanning.

- [ ] **1.7** Write **lexer unit tests** in `compiler/tests/` — feed known source
      strings and assert the exact token stream produced.

---

## Phase 2 — Parser

> `compiler/src/parser/`

The parser consumes the token stream and produces an Abstract Syntax Tree. Use a
**hand-written recursive descent parser** — do not use yacc/bison. It gives you
better error messages and full control, which matters for a language designed to
be pleasant to use.

- [ ] **2.1** Implement a **token lookahead buffer** — the parser needs at minimum
      1-token lookahead (peek), but 2 tokens makes some grammar decisions cleaner.
      Implement `parser_peek(Parser*)` and `parser_advance(Parser*)`.

- [ ] **2.2** Implement `parser_expect(Parser*, TokenType)` — advances and returns
      the next token, or emits a diagnostic if the type doesn't match. This is your
      most-called function.

- [ ] **2.3** Parse **top-level declarations** first:
      - `func` declarations
      - `class` declarations
      - `interface` declarations
      - `import` statements
      - `const` / module-level `let` bindings

- [ ] **2.4** Parse **statements**:
      - Variable declarations (`let x: Int = 5;`, `let x = 5;` with inferred type)
      - Assignment (`x = expr;`, `x += expr;`)
      - `if` / `else if` / `else` — all blocks use `{}`
      - `while` loop, `for` loop, `foreach` loop
      - `return`, `break`, `continue`
      - Expression statements (function calls as statements)
      - `try` / `catch` / `finally` blocks

- [ ] **2.5** Parse **expressions** with correct operator precedence using a
      **Pratt parser** (top-down operator precedence) inside the recursive descent
      framework. Assign binding powers to:
      - Assignment (`=`, `+=`, `-=`, etc.) — lowest precedence, right-associative
      - Logical OR (`||`), AND (`&&`)
      - Equality (`==`, `!=`)
      - Comparison (`<`, `>`, `<=`, `>=`)
      - Addition/subtraction (`+`, `-`)
      - Multiplication/division/modulo (`*`, `/`, `%`)
      - Unary (`!`, `-`, `ref`)
      - Postfix: function call `f(args)`, index `a[i]`, member access `a.b`

- [ ] **2.6** Parse **type expressions** — used in variable annotations, function
      signatures, and generic parameters:
      - Primitive types: `Int`, `Float`, `Decimal`, `String`, `Bool`, `Date`
      - Generic types: `List<T>`, `Result<T, E>`, `Map<K, V>`
      - Nullable types: `Int?`
      - Function types: `(Int, Int) -> Bool`

- [ ] **2.7** Parse **BBL query syntax** — the LINQ/OpenSQL-style expressions:
      ```
      SELECT * FROM orders WHERE status == "OPEN" ORDER BY createdAt;
      ```
      These are first-class expression forms in BBL. Parse them into their own
      AST node type, not as raw function calls.

- [ ] **2.8** Implement **synchronization on error** — when the parser hits an
      unexpected token, it should emit a diagnostic and then skip forward to a
      safe synchronization point (usually the next `;` or `}`) to continue parsing
      and find more errors. A compiler that stops at the first error is annoying.

- [ ] **2.9** Write **parser unit tests** — parse snippets and assert the shape of
      the resulting AST. Test error recovery paths explicitly.

---

## Phase 3 — AST Node Definitions

> `compiler/src/ast/` and `compiler/include/ast.h`

The AST is the data structure that the parser produces and every subsequent phase
consumes. Get this right before building semantic analysis.

- [ ] **3.1** Define the **node kind enum** — one entry per node type:
      `NODE_FUNC_DECL`, `NODE_CLASS_DECL`, `NODE_IF_STMT`, `NODE_BINARY_EXPR`,
      `NODE_CALL_EXPR`, `NODE_IDENT`, `NODE_INT_LIT`, etc.

- [ ] **3.2** Define the **AST node union** — a tagged union is the idiomatic C
      approach:
      ```c
      typedef struct AstNode {
          NodeKind kind;
          int line, col;
          union {
              FuncDeclNode   func_decl;
              BinaryExprNode binary_expr;
              CallExprNode   call_expr;
              // ...
          };
      } AstNode;
      ```
      All nodes are allocated from the arena (Phase 0.3).

- [ ] **3.3** Define individual **node structs** — each AST node type gets its own
      struct capturing exactly the information it needs:
      - `FuncDeclNode`: name, param list, return type, body block
      - `ClassDeclNode`: name, superclass, interfaces, members
      - `BinaryExprNode`: left, operator, right
      - `CallExprNode`: callee expression, argument list
      - `IfStmtNode`: condition, then-block, optional else block
      - `VarDeclNode`: name, type annotation (nullable), initializer
      - `QueryExprNode`: select list, from source, where clause, order-by, group-by
      - etc.

- [ ] **3.4** Implement an **AST printer** in `compiler/src/ast/print.c` — walks the
      tree and prints a human-readable s-expression representation. Essential for
      debugging the parser. You will use this constantly.

- [ ] **3.5** Implement the **visitor pattern** (or equivalent walk function) —
      a generic `ast_walk(AstNode*, Visitor*)` that traverses the tree. Semantic
      analysis, type checking, and code generation all consume the AST via walks.

---

## Phase 4 — Semantic Analysis & Type Checking

> `compiler/src/sema/`

This phase transforms a syntactically valid AST into a semantically valid one by:
resolving names, checking types, enforcing language rules. Most of the complexity
of a statically-typed language lives here.

- [ ] **4.1** Implement the **symbol table** (scope chain) — a stack of hash maps.
      Each block `{}` pushes a new scope; leaving it pops. The `lookup(name)`
      function walks the chain outward. Use the interned string table (Phase 0.4)
      for keys.

- [ ] **4.2** Implement **name resolution** — walk the AST, bind every identifier
      reference to its declaration, or emit "undefined variable" / "undefined type"
      diagnostics. Resolve:
      - Local variables
      - Function names (including overloads)
      - Class names and member accesses
      - Import references

- [ ] **4.3** Define the **type representation** in `compiler/include/types.h`:
      ```c
      typedef struct Type {
          TypeKind kind;   // TYPE_INT, TYPE_STRING, TYPE_CLASS, TYPE_GENERIC, ...
          union {
              ClassType   class_type;
              GenericType generic_type;
              FuncType    func_type;
          };
      } Type;
      ```

- [ ] **4.4** Implement **type inference** — for `let x = expr;` without a type
      annotation, infer the type from the expression. Implement Hindley-Milner
      style unification for generics, or a simpler bidirectional type checking
      approach if full HM is too ambitious for v1.

- [ ] **4.5** Implement **type checking** — walk every expression and:
      - Verify operand types match operator expectations
      - Verify function argument types match parameter types
      - Verify assignment types are compatible
      - Check return types match declared return type
      - Verify index expressions use integer types
      - Enforce null safety: `Int?` is not assignable to `Int` without unwrap

- [ ] **4.6** Implement **class and interface checking**:
      - Verify all interface methods are implemented
      - Verify method override signatures are compatible
      - Verify visibility rules (private members not accessible outside class)
      - Verify super constructor is called if required

- [ ] **4.7** Implement **BBL-specific business type checks**:
      - `Decimal` vs `Float` — prevent implicit mixing in arithmetic
      - `Money<Currency>` phantom type enforcement
      - Business table type compatibility in queries

- [ ] **4.8** Annotate the **AST with resolved types** — after type checking, every
      expression node should carry its resolved `Type*`. Code generation depends on
      this. Store it directly in `AstNode`.

- [ ] **4.9** Implement **control flow analysis**:
      - All code paths in a non-void function must return
      - Variables must be assigned before use
      - `break`/`continue` only valid inside loops
      - Unreachable code after `return` should warn, not error

---

## Phase 5 — Bytecode Format Definition

> `vm/src/bytecode/` and `vm/include/bytecode.h`

Before writing the code generator, define exactly what bytecode you are
generating. This is your instruction set architecture — change it later and you
break everything.

- [ ] **5.1** Define the **instruction set** — a flat enum of opcodes:
      ```c
      typedef enum {
          // Stack operations
          OP_PUSH_INT,    // push integer constant
          OP_PUSH_FLOAT,
          OP_PUSH_STR,
          OP_PUSH_BOOL,
          OP_PUSH_NULL,
          OP_POP,
          OP_DUP,

          // Local variables
          OP_LOAD_LOCAL,   // push value of local[index] onto stack
          OP_STORE_LOCAL,  // pop and store into local[index]

          // Arithmetic
          OP_ADD, OP_SUB, OP_MUL, OP_DIV, OP_MOD,
          OP_NEG,

          // Comparison
          OP_EQ, OP_NEQ, OP_LT, OP_LTE, OP_GT, OP_GTE,

          // Logical
          OP_AND, OP_OR, OP_NOT,

          // Control flow
          OP_JUMP,           // unconditional jump to offset
          OP_JUMP_IF_FALSE,  // pop, jump if falsy
          OP_JUMP_IF_TRUE,   // pop, jump if truthy

          // Functions
          OP_CALL,           // call function with N args
          OP_CALL_NATIVE,    // call native (C) function
          OP_RETURN,
          OP_RETURN_VOID,

          // Objects & fields
          OP_NEW,            // allocate object of class K
          OP_GET_FIELD,      // pop object, push field
          OP_SET_FIELD,      // pop value, pop object, set field
          OP_INVOKE,         // virtual method dispatch

          // Collections
          OP_NEW_LIST,
          OP_LIST_PUSH,
          OP_LIST_GET,
          OP_LIST_SET,
          OP_LIST_LEN,

          // Error handling
          OP_THROW,
          OP_ENTER_TRY,
          OP_EXIT_TRY,

          OP_HALT,
      } Opcode;
      ```

- [ ] **5.2** Define the **constant pool** — each compiled class/function has a
      constant pool of values it references: integer literals, string literals,
      field name strings, class references, function references. The bytecode
      references constants by index. Define `ConstPool` and `Constant` structs.

- [ ] **5.3** Define the **.bbc (BBL Bytecode) file format**:
      ```
      [4 bytes]  magic: 0x42424C43  ("BBLC")
      [2 bytes]  version major
      [2 bytes]  version minor
      [4 bytes]  constant pool count
      [variable] constant pool entries
      [4 bytes]  class count
      [variable] class definitions
        [4 bytes]  method count
        [variable] method definitions
          [4 bytes]  bytecode length
          [variable] bytecode instructions
          [4 bytes]  local variable count
          [4 bytes]  max stack depth
      ```

- [ ] **5.4** Implement a **bytecode disassembler** in `vm/src/bytecode/disasm.c` —
      reads a `.bbc` file and prints human-readable output of every instruction.
      You will stare at this output for hours debugging the codegen. It is not
      optional.

- [ ] **5.5** Implement a **bytecode verifier** in `vm/src/bytecode/verify.c` —
      before executing, validate that bytecode is structurally sound: stack depth
      never goes negative, jump targets are within bounds, method signatures match
      calls. Saves you from hours of VM crashes on malformed bytecode.

---

## Phase 6 — Code Generation

> `compiler/src/codegen/`

Takes the type-annotated AST (output of Phase 4) and emits bytecode instructions
into the format defined in Phase 5.

- [ ] **6.1** Implement the **code generation context** — tracks the current function
      being compiled, the instruction buffer, the constant pool being built, and a
      local variable slot allocator.

- [ ] **6.2** Implement **expression code generation** — emit instructions for every
      expression node type:
      - Integer/float/string/bool literals → `OP_PUSH_*`
      - Identifiers → `OP_LOAD_LOCAL` (or `OP_GET_FIELD` for `self.x`)
      - Binary expressions → recurse left, recurse right, emit operator opcode
      - Unary expressions → recurse operand, emit `OP_NEG` / `OP_NOT`
      - Function calls → emit args, emit `OP_CALL` or `OP_INVOKE`
      - Object construction → emit args, emit `OP_NEW`
      - Field access → emit object, emit `OP_GET_FIELD`

- [ ] **6.3** Implement **statement code generation**:
      - Variable declaration → allocate local slot, emit initializer, `OP_STORE_LOCAL`
      - Assignment → emit value, `OP_STORE_LOCAL` (or `OP_SET_FIELD`)
      - `if`/`else` → emit condition, `OP_JUMP_IF_FALSE` with backpatch, emit branches
      - `while` → emit loop body with `OP_JUMP` back, `OP_JUMP_IF_FALSE` to exit
      - `for` → desugar to `while` with counter logic
      - `foreach` → emit iterator protocol: `OP_CALL` `iter()`, loop on `OP_CALL` `next()`
      - `return` → emit return value, `OP_RETURN`
      - `try`/`catch` → emit `OP_ENTER_TRY` with handler offset, body, `OP_EXIT_TRY`

- [ ] **6.4** Implement **backpatching** — `OP_JUMP_IF_FALSE` and `OP_JUMP` need to
      know their target offset at emit time, but the target hasn't been emitted yet.
      Standard solution: emit a placeholder offset, save the instruction's position,
      then fill it in once the target is known. Implement `emit_jump()` and
      `patch_jump(pos, target)`.

- [ ] **6.5** Implement **local variable slot allocation** — each function's locals
      get integer slot indices. Variables declared in nested scopes can reuse slots
      after their scope ends (slot recycling). Track `max_locals` for the method
      descriptor.

- [ ] **6.6** Implement **constant pool construction** — as the codegen walks the AST,
      add literals and references to the constant pool. Deduplicate: the same string
      literal appearing 10 times should have one pool entry.

- [ ] **6.7** Implement **class and method compilation** — for each class declaration,
      emit a class descriptor with field layout and method table. Compile each method
      into its own bytecode buffer.

- [ ] **6.8** Implement **bytecode serialization** — write the complete in-memory
      bytecode structures to a `.bbc` file using the format defined in Phase 5.

---

## Phase 7 — Virtual Machine (Execution Engine)

> `vm/src/core/`

The VM reads `.bbc` files and executes them. It is a stack-based interpreter,
like the JVM.

- [ ] **7.1** Define the **value representation** in `vm/src/runtime/value.h`.
      Use a tagged union (or NaN-boxing if you want to be fancy):
      ```c
      typedef struct {
          ValueTag tag;  // VAL_INT, VAL_FLOAT, VAL_DECIMAL, VAL_BOOL,
                         // VAL_NULL, VAL_OBJ
          union {
              int64_t   int_val;
              double    float_val;
              BBLDecimal decimal_val;
              bool      bool_val;
              ObjHeader *obj;     // pointer to heap-allocated object
          };
      } Value;
      ```

- [ ] **7.2** Define the **object model** in `vm/src/runtime/object.h`:
      - `ObjHeader`: GC metadata, class pointer, object size
      - `ObjString`: interned character data
      - `ObjList`: dynamic array of `Value`
      - `ObjInstance`: class pointer + field array
      - `ObjClosure`: function pointer + captured variable array

- [ ] **7.3** Implement the **call stack** — a `CallFrame` per active function call,
      containing: instruction pointer, pointer into the value stack for locals,
      pointer to the current function's bytecode.

- [ ] **7.4** Implement the **main execution loop** in `vm/src/core/vm.c` —
      a `switch` on the current opcode, advancing the instruction pointer.
      This is the hottest code in the entire project. Every instruction is a `case`:
      ```c
      case OP_ADD: {
          Value b = stack_pop(vm);
          Value a = stack_pop(vm);
          stack_push(vm, value_add(a, b));
          break;
      }
      ```

- [ ] **7.5** Implement **virtual dispatch** (`OP_INVOKE`) — look up the method in
      the class's vtable, push a new call frame.

- [ ] **7.6** Implement **native function binding** (`OP_CALL_NATIVE`) — a mechanism
      for BBL code to call C functions. Each native is registered by name and maps
      to a C function pointer with signature `Value native_fn(VM*, int argc, Value* argv)`.

- [ ] **7.7** Implement the **exception mechanism** — `OP_THROW` walks the call stack
      looking for the nearest `OP_ENTER_TRY` handler. If none found, print the
      unhandled exception and halt.

- [ ] **7.8** Implement **bytecode loading** — read a `.bbc` file, deserialize the
      constant pool and class definitions into VM-internal structures, ready to execute.

---

## Phase 8 — Garbage Collector

> `vm/src/memory/`

The VM allocates heap objects (strings, lists, class instances). You need a GC or
you have a memory leak. Start with **mark-and-sweep** — it is not optimal but it
is correct and implementable in a weekend.

- [ ] **8.1** Implement a **heap allocator** — all VM heap objects go through
      `vm_alloc(vm, size)`, which calls `malloc` and registers the object in the
      GC's tracked object list.

- [ ] **8.2** Implement **GC roots identification** — the set of `Value`s that are
      always live: the value stack, all call frame locals, global variables,
      interned strings.

- [ ] **8.3** Implement the **mark phase** — starting from roots, recursively mark
      every reachable object. Each `ObjHeader` has a `marked` bit.

- [ ] **8.4** Implement the **sweep phase** — walk the full list of tracked objects,
      free any that are not marked, unmark the rest.

- [ ] **8.5** Implement **GC triggering** — run the GC when total heap allocation
      since the last collection exceeds a threshold (e.g., 1 MB). Tune the threshold
      later. Avoid collecting on every allocation.

- [ ] **8.6** (Optional v1 enhancement) Implement a **string intern table** in the
      VM — identical string values share one `ObjString` allocation. This is a big
      win for business applications that process the same field name strings millions
      of times.

---

## Phase 9 — Standard Library (Native Bindings)

> `vm/src/stdlib/` (native C side), `stdlib/` (BBL side once runnable)

- [ ] **9.1** Implement **core I/O natives**:
      - `print(val)` — write to stdout
      - `println(val)` — write with newline
      - `readLine() -> String` — read from stdin

- [ ] **9.2** Implement **string natives**:
      - `String.length()`, `String.slice(start, end)`, `String.contains(sub)`
      - `String.toUpper()`, `String.toLower()`
      - `String.split(delim) -> List<String>`
      - `String.format(args...)` — interpolation

- [ ] **9.3** Implement **list/collection natives**:
      - `List.push(val)`, `List.pop()`, `List.get(i)`, `List.set(i, val)`
      - `List.length()`, `List.slice(start, end)`
      - `List.map(fn)`, `List.filter(fn)`, `List.reduce(fn, init)`
      - `List.sort(comparator)`

- [ ] **9.4** Implement **Decimal arithmetic natives** — `Decimal` is not a float.
      It is fixed-point arithmetic for money. Implement `decimal_add`,
      `decimal_mul`, `decimal_round(places)` correctly. This is the type that
      ABAP gets right and most languages get wrong. Do not use `double` here.

- [ ] **9.5** Implement **date/time natives**:
      - `Date.today()`, `Date.parse(str, format)`
      - `Date.addDays(n)`, `Date.diffDays(other)`
      - `DateTime.now()`, `DateTime.toTimezone(tz)`

- [ ] **9.6** Implement **Result and Option types** as BBL-level classes with native
      support. `Result<T, E>` carries either a value or an error. `Option<T>` is
      either `Some(val)` or `None`. These are the null-safety backbone.

- [ ] **9.7** Implement **file I/O natives**:
      - `File.read(path) -> Result<String, IOError>`
      - `File.write(path, content) -> Result<Unit, IOError>`
      - `File.readLines(path) -> Result<List<String>, IOError>`

---

## Phase 10 — CLI Entry Point

> `cli/src/`

- [ ] **10.1** Implement `bbl run <file.bbl>` — compiles the file in memory and
      immediately passes the bytecode to the VM. No `.bbc` file written to disk.

- [ ] **10.2** Implement `bbl build <file.bbl> -o <out.bbc>` — compiles and writes
      the `.bbc` bytecode file. Supports `--release` flag for future optimization passes.

- [ ] **10.3** Implement `bbl exec <file.bbc>` — runs a pre-compiled bytecode file
      directly through the VM. This is your production deployment path.

- [ ] **10.4** Implement `bbl check <file.bbl>` — runs lexer + parser + sema only,
      no code generation. Fast feedback for editor integrations and CI.

- [ ] **10.5** Implement `bbl init <project-name>` — scaffolds a new project directory
      with a `bbl.toml` manifest and a `src/main.bbl` hello-world entry point.

- [ ] **10.6** Implement `bbl fmt <file.bbl>` — pipe source through a formatting pass
      and write back in-place (or print to stdout with `--check`).

---

## Phase 11 — Error Reporting

> Thread through all phases

This deserves its own checklist because bad error messages are a primary reason
people abandon new languages — and you know this personally from ABAP's cryptic
dumps.

- [ ] **11.1** Define a **Diagnostic struct**:
      ```c
      typedef struct {
          DiagLevel level;    // ERROR, WARNING, NOTE
          const char *file;
          int line, col;
          const char *message;
          const char *source_line;  // the actual line of source text
          int caret_col;            // where the ^ points
      } Diagnostic;
      ```

- [ ] **11.2** Implement **source-location tracking** throughout all phases — every
      AST node, every type, every symbol carries a source location. Never emit an
      error without a location.

- [ ] **11.3** Implement **caret-style error output**:
      ```
      error[E042]: type mismatch — expected Decimal, got Float
        --> src/billing.bbl:47:23
         |
      47 |   let subtotal: Decimal = price * 0.85;
         |                                   ^^^^ Float literal
         = note: use 0.85d for a Decimal literal
      ```
      Steal this format shamelessly from Rust. It is good.

- [ ] **11.4** Implement **error recovery** in both the lexer and parser so a single
      source file produces all its errors in one pass, not just the first.

- [ ] **11.5** Define **error codes** — `E001` through `E999` by category. Makes it
      easy to add `--explain E042` help text later.

---

## Recommended Build Order Summary

| Step | What You Build           | Where It Lives             |
|------|--------------------------|----------------------------|
| 0    | Infrastructure & utils   | `compiler/src/util/`       |
| 1    | Lexer                    | `compiler/src/lexer/`      |
| 2    | Parser                   | `compiler/src/parser/`     |
| 3    | AST nodes                | `compiler/src/ast/`        |
| 4    | Semantic analysis        | `compiler/src/sema/`       |
| 5    | Bytecode format          | `vm/src/bytecode/`         |
| 6    | Code generator           | `compiler/src/codegen/`    |
| 7    | VM execution engine      | `vm/src/core/`             |
| 8    | Garbage collector        | `vm/src/memory/`           |
| 9    | Standard library         | `vm/src/stdlib/` + `stdlib/` |
| 10   | CLI                      | `cli/src/`                 |
| 11   | Error reporting          | All phases, threaded through |

**First working milestone:** Phases 0–7 give you a compiler that can turn a BBL
source file into bytecode and run it. At that point you have a real programming
language, even if it has no standard library and terrible error messages.

**Do not skip Phase 5 (bytecode format) before Phase 6 (codegen).** Designing the
instruction set before writing the emitter saves you from rewriting the codegen
when you realize `OP_JUMP` needs to carry a different kind of offset than you
assumed.
