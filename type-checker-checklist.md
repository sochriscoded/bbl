# BBL Type System — Completeness Checklist

> **How to use this document**
> Items marked **Decide:** are design questions — you must resolve them before writing the rule that depends on them.
> Items marked with a plain description are implementation tasks — write the formal rule and add it to `BBL_TypeSystem.tex`.
> The final two sections are audit passes over work already done, not new rules to write.

-----

## §1 — Primitive Types

*The atoms — every type is built from these.*

- [ ] Integer type(s) — how many widths? signed/unsigned?
- [ ] Floating point type(s)
- [ ] Decimal / fixed-point type (critical for business)
- [ ] Boolean type
- [ ] String / character type(s)
- [ ] Void / Unit type (type of expressions with no meaningful value)
- [ ] Bottom type ⊥ (type of expressions that never return — errors, divergence)
- [ ] Typing rules for all literal forms
- [ ] **Decide:** are numeric literals polymorphic or monomorphic?

-----

## §2 — Coercions & Conversions

*What happens when types don’t match.*

- [ ] Define all implicit (automatic) coercions, if any
- [ ] Define all explicit (cast) conversions
- [ ] Widening conversions — which are safe? (Int → Float?)
- [ ] Narrowing conversions — are they allowed? How?
- [ ] Numeric tower — is Int a subtype of Float, or are they unrelated?
- [ ] Typing rule for each implicit coercion
- [ ] Typing rule for explicit cast expressions
- [ ] **Decide:** what happens on narrowing failure at runtime?

-----

## §3 — Operators & Expressions

*Rules for every operator in the language.*

- [ ] Arithmetic: +, -, *, / for each numeric type
- [ ] Integer division semantics — truncating or flooring?
- [ ] Modulo / remainder semantics
- [ ] Comparison operators: ==, !=, <, >, <=, >= — what types support them?
- [ ] Logical operators: &&, ||, ! — Bool only or short-circuit semantics?
- [ ] Bitwise operators, if any
- [ ] String concatenation — operator or method?
- [ ] Operator overloading policy — allowed? On which types?
- [ ] **Decide:** mixed-type arithmetic (Int + Float) — error or coercion?
- [ ] Typing rule for each operator form

-----

## §4 — Variables & Bindings

*How names are introduced and resolved.*

- [ ] Variable declaration rule (let / var / const)
- [ ] Mutability — is it tracked in the type system or just enforced syntactically?
- [ ] Variable lookup rule (T-Var)
- [ ] Shadowing — allowed? Any type-level restrictions?
- [ ] Scope rules — block scope, function scope, module scope
- [ ] Typed vs. inferred declarations — when can the annotation be omitted?
- [ ] Assignment rule — is assignment an expression or a statement?
- [ ] Typing rule for assignment (must check LHS type matches RHS type)

-----

## §5 — Control Flow

*Every branching and looping construct.*

- [ ] if/else — condition must be Bool; both branches must have same type
- [ ] **Decide:** is `if` an expression (returns value) or a statement (returns Void)?
- [ ] `if` without `else` — what is the type? (Void or Option?)
- [ ] `while` loop — type is Void; condition must be Bool
- [ ] `for` loop — what does the iterator yield? Type of loop variable?
- [ ] for-each / range loops — typing rule for the range/iterator type
- [ ] `break` / `continue` — type is ⊥ (Bottom)
- [ ] `return` — type is ⊥ (Bottom); must check return type matches function signature
- [ ] `match` / `switch` — exhaustiveness, all branches same type
- [ ] **Decide:** are there labeled breaks/continues? Type implications?

-----

## §6 — Functions

*The core of any type system.*

- [ ] Function type syntax — how is τ₁ → τ₂ written in BBL?
- [ ] Function declaration rule (T-Abs)
- [ ] Function application rule (T-App)
- [ ] Multi-parameter functions — tuple of args or curried?
- [ ] Return type annotation — required or inferred?
- [ ] Void functions — explicit return type or implicit?
- [ ] Named vs. anonymous functions — same rules or different?
- [ ] Closures — do they capture by value or reference? Type implications?
- [ ] Higher-order functions — functions as arguments and return values
- [ ] **Decide:** call-by-value vs. call-by-reference — how is reference passing typed?
- [ ] Recursive functions — any special typing rule needed?
- [ ] Mutually recursive functions — how are they typed?
- [ ] Optional / default parameters — how are they typed?
- [ ] Variadic parameters — how are they typed?

-----

## §7 — Product Types (Structs / Records)

*AND composition — values that have multiple fields.*

- [ ] Record/struct type definition rule
- [ ] Record introduction rule (T-Record) — typing a struct literal
- [ ] Record field access rule (T-Field)
- [ ] Record update / mutation rule, if fields are mutable
- [ ] **Decide:** structural or nominal typing for records?
- [ ] Width subtyping for records — a record with more fields is a subtype?
- [ ] Depth subtyping for records — field types can be subtypes?
- [ ] Nested records — do the rules compose correctly?
- [ ] Recursive structs — how is self-reference handled?
- [ ] **Decide:** are all fields required on construction, or can any be optional?

-----

## §8 — Sum Types (Enums / Variants)

*OR composition — values that are one of several alternatives.*

- [ ] Enum / variant type definition rule
- [ ] Variant introduction rules — one rule per variant constructor
- [ ] Variant elimination rule (T-Match / T-Case)
- [ ] Exhaustiveness checking — all variants must be covered in a match
- [ ] Binding in match arms — payload bound to variable in scope for that arm
- [ ] Nested pattern matching — do the rules compose?
- [ ] Wildcard / catch-all pattern — type implications?
- [ ] Guard clauses in match — how are they typed?
- [ ] Variants with no payload — type is Unit/Void or omitted?
- [ ] Variants with multiple fields — typed as a tuple or record?

-----

## §9 — Absence & Optionality

*The billion-dollar question — how is null / missing modeled?*

- [ ] **Decide:** null, Option<T>, nullable types, or something else?
- [ ] Option<T> introduction rules — Some(e) and None
- [ ] Option<T> elimination rule — must unwrap before use
- [ ] Typing rule for None — what is its type? (Option<α> with fresh α?)
- [ ] **Decide:** is there a null-coalescing operator? How is it typed?
- [ ] **Decide:** is optional chaining (e?.field) supported? How is it typed?
- [ ] Ensure non-optional types can never be null at compile time

-----

## §10 — Error Handling

*How failure is represented in the type system.*

- [ ] **Decide:** exceptions, Result<T,E>, error codes, or combination?
- [ ] Result<T,E> introduction rules — Ok(e) and Err(e)
- [ ] Result<T,E> elimination rule — must handle both cases
- [ ] Typing rule for the ? / early-return operator, if any
- [ ] Panic / fatal error — type is ⊥ (Bottom)
- [ ] **Decide:** are exceptions tracked in the type system (checked exceptions)?
- [ ] Error type hierarchy — is there a base Error type?
- [ ] Interop between Result and Option — how do they compose?

-----

## §11 — Generics (Parametric Polymorphism)

*Types parameterized over other types.*

- [ ] Type variable syntax — T, α, or both?
- [ ] Generic type definition — how are type parameters declared on structs/enums?
- [ ] Generic function definition — how are type parameters declared on functions?
- [ ] Type application rule (T-TApp) — instantiating a generic type
- [ ] Type abstraction rule (T-TAbs) — introducing a polymorphic type
- [ ] **Decide:** explicit type arguments, inferred, or both?
- [ ] Substitution — define τ[σ/α] formally for all type forms
- [ ] Freshness condition — type variable must not appear free in context
- [ ] **Decide:** should type inference be full HM, local, or none?
- [ ] Higher-kinded types — are type constructors allowed as type arguments?

-----

## §12 — Trait Bounds & Constraints

*Constraining what type variables can be.*

- [ ] Trait / interface definition — what does a trait declare?
- [ ] Trait implementation rule — what does it mean for a type to implement a trait?
- [ ] Bounded quantification — ∀α <: Trait. τ
- [ ] Typing rule for trait method dispatch
- [ ] Multiple bounds — T: Trait1 + Trait2
- [ ] **Decide:** trait coherence rules — can the same type implement a trait twice?
- [ ] **Decide:** blanket implementations — can a trait be auto-implemented?
- [ ] Trait objects / dynamic dispatch — is runtime dispatch supported? How typed?
- [ ] Associated types in traits, if any
- [ ] **Decide:** default method implementations — typing implications?

-----

## §13 — Subtyping

*When one type can stand in for another.*

- [ ] Reflexivity rule — every type is a subtype of itself
- [ ] Transitivity rule — subtyping chains
- [ ] Subsumption rule (T-Sub) — using a subtype where a supertype is expected
- [ ] **Decide:** nominal or structural subtyping, or both?
- [ ] Record width subtyping rule
- [ ] Record depth subtyping rule
- [ ] Function subtyping rule — contravariant input, covariant output
- [ ] Subtyping for sum types
- [ ] Top type ⊤ — is there a type every type is a subtype of?
- [ ] Bottom type ⊥ subtyping — ⊥ is a subtype of every type
- [ ] Variance for generic types — covariant, contravariant, or invariant per parameter?
- [ ] **Decide:** is variance declared, inferred, or fixed as invariant?

-----

## §14 — Collections & Built-in Generics

*The standard parameterized types every language needs.*

- [ ] List / Array — introduction rule, indexing rule, mutation rule
- [ ] Map / Dictionary — key type constraints, lookup rule
- [ ] Set — element type constraints
- [ ] Tuple types — introduction and projection rules
- [ ] **Decide:** are collections homogeneous (typed) or heterogeneous?
- [ ] Iterator protocol — what type does iteration yield?
- [ ] Range types, if any
- [ ] **Decide:** fixed-size vs. dynamic collections — type-level distinction?

-----

## §15 — Recursive Types

*Types that refer to themselves.*

- [ ] **Decide:** how is self-reference in struct/enum definitions handled?
- [ ] Ensure recursive types are always guarded by a type constructor (not naked)
- [ ] Typing rule for recursive data construction
- [ ] Typing rule for recursive data consumption (fold/match)
- [ ] Mutual recursion between type definitions
- [ ] Verify: recursive types don’t cause infinite expansion during type-checking

-----

## §16 — Modules & Namespaces

*How types interact with the module system.*

- [ ] Module-level type context — how are top-level names typed?
- [ ] Import rule — bringing names from another module into scope
- [ ] Export rule — what types are visible outside a module?
- [ ] **Decide:** opaque types — can a module hide the definition of a type?
- [ ] Typing rule for qualified name access (Module.Type, Module.fn)
- [ ] Circular module dependencies — are they allowed? Type implications?
- [ ] **Decide:** how are generic types from imported modules instantiated?

-----

## §17 — Concurrency

*If the type system needs to track thread safety.*

- [ ] **Decide:** does the type system track sendability across threads?
- [ ] **Decide:** does the type system track shared mutable state?
- [ ] Async/await — what is the type of an async expression? (Future<T>?)
- [ ] Typing rule for await expressions
- [ ] Typing rule for spawn / task creation
- [ ] Channel types — how are typed channels declared?
- [ ] **Decide:** are there marker traits for Send / Sync equivalents?

-----

## §18 — Interoperability & FFI

*Where the type system meets the outside world.*

- [ ] Foreign function declarations — how are external functions typed?
- [ ] Unsafe / unchecked escape hatch — is there one? How is it typed?
- [ ] **Decide:** how are untyped external values brought into the type system?
- [ ] Serialization / deserialization — type implications of parsing external data
- [ ] **Decide:** are there platform-specific types that need special rules?

-----

## §19 — Type Inference

*What the compiler figures out so the programmer doesn’t have to.*

- [ ] **Decide:** scope of inference — local only, full HM, or somewhere between?
- [ ] Define unification algorithm for your type forms
- [ ] Define substitution formally for all type constructors in the language
- [ ] Occurs check — prevent infinite types during unification
- [ ] **Decide:** how are inference failures reported to the programmer?
- [ ] **Decide:** can inference cross function boundaries?
- [ ] **Decide:** are there any constructs where annotation is always required?
- [ ] Monomorphization vs. boxing for generics — type inference implications

-----

## §20 — Soundness & Completeness

*Audit pass — verify the rules you’ve written hold together.*

- [ ] Progress — every well-typed program either is a value or can take a step
- [ ] Preservation — type is preserved across evaluation steps
- [ ] Verify: no rule allows a well-typed program to produce a runtime type error
- [ ] Verify: every syntactic form has at least one typing rule
- [ ] Verify: introduction/elimination pairs exist for every type
- [ ] Verify: substitution lemma holds — substituting types preserves typing
- [ ] Verify: weakening holds — adding unused bindings to context preserves typing
- [ ] Identify any known unsound corners (and decide whether to accept them)
- [ ] Document all deliberate unsoundness (like type casts) explicitly

-----

## §21 — Error Reporting

*The human side of the type system — audit pass over completed rules.*

- [ ] For each rule, define what a failed premise means in plain language
- [ ] Unification failure messages — expected X, found Y
- [ ] Scope error messages — unbound variable, undefined type
- [ ] Exhaustiveness failure messages — missing match cases
- [ ] Arity mismatch messages — wrong number of arguments
- [ ] **Decide:** should error messages suggest fixes, not just report problems?
- [ ] Ensure error messages reference source locations, not internal types

-----

*Completeness means every syntactic construct has a typing rule, every type has both introduction and elimination rules, and progress + preservation hold. Not every item here requires a formal proof — but each requires a deliberate decision.*