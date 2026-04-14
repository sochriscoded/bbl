## You appear to be advocating a new:

**Functional-featured:** closures, map/filter/reduce over business collections (line items, order sets), maybe pipes. But forcing monadic IO on a language targeting SAP-adjacent developers is a non-starter.

**Imperative + OOP:** this mirrors ABAP's procedural roots while modernizing toward the object model ABAP added awkwardly in the late 90s. Business logic is naturally stateful and sequential, so imperative fits.
 
**Eager:** lazy evaluation introduces unpredictable performance characteristics that are genuinely hostile in ERP-style workloads where you need to reason about when a DB query fires.

**Statically-typed:** this is the right call for business software. ABAP's runtime type errors in production have cost companies real money. Static typing catches domain mistakes (assigning a quantity to a currency field) at compile time.


## Unfortunately, your language has:

**Concurrency:** ABAP's lack of real concurrency primitives is a known pain point. Background jobs and parallel processing are core to batch business workloads. BBL needs at minimum async/await and ideally structured concurrency.

**REPL:** essential for adoption. Developers experimenting with business logic transformations need fast feedback.

**Debugger support:**non-negotiable for enterprise. This needs to be a first-class tooling concern, not an afterthought.

**IDE support:** LSP implementation should be planned from day one. ABAP developers are accustomed to the ABAP Workbench and SE80; if BBL's tooling is worse, adoption dies.

**Interop with other languages:** BBL will need FFI or at minimum clean interfaces to existing systems (RFC, BAPI, REST). Nobody is rewriting their entire ERP stack in BBL on day one.

**Semicolons:** already decided, C-style. Correct call. Whitespace sensitivity (ABAP's period-as-terminator) is a genuine source of bugs and editor frustration.
Closures: necessary for any higher-order collection operations over business data.

**Exceptions:** yes, but design them carefully. Checked vs unchecked is a real debate. Business errors (insufficient stock) vs programming errors (null deref) should probably be distinct.

**Type inference:** reduces boilerplate significantly without sacrificing static safety. Letting the compiler infer the type of a local variable is not the same as dynamic typing.

**Algebraic datatypes / tagged unions:** extremely useful for modeling business states. An order that is either Draft | Submitted | Approved | Cancelled is cleaner as an ADT than as an integer status field.

**Multi-line strings:** essential for embedded queries, report templates, message text.

**Tail recursion:** include it, even if most BBL developers won't use it directly. It enables clean recursive data processing without stack overflow risk.
Regexes: business data is full of pattern matching (VAT numbers, ISBNs, postal codes). First-class regex support matters.

**Recursive types:** A recursive type is simply a type that references itself in its own definition — the canonical example being a linked list or tree node. For BBL this matters because business data is frequently hierarchical: a bill of materials where a component can itself have sub-components, an org chart, a category tree in a product catalog. Without recursive types you end up modeling these with workarounds like parallel arrays or string-encoded paths, which is exactly the kind of thing ABAP developers currently suffer through. The implementation cost is low if your type system is designed with it in mind from the start.
Parametric polymorphism (generics) — definitely yes. A List<T> or Result<T, E> that works across any type is essential. Without this, your standard library becomes a combinatorial explosion of type-specific functions.

**Dependent types:** Exclude from v1, keep the door open. Dependent types let the type system encode values — the classic example being a vector type that carries its length in the type itself, so Vec<T, 3> and Vec<T, 4> are genuinely different types and you can't accidentally add them. For business software this is tantalizing: a Money<Currency.USD> and Money<Currency.EUR> being distinct types would catch entire categories of financial bugs at compile time. The pragmatic middle ground is phantom types — a lightweight technique that achieves the currency/unit tagging goal without full dependent types. Money<USD> vs Money<EUR> as distinct types can be done with phantom type parameters, which are parametric polymorphism you already have. That's the design to reach for.

**Infix operators:** Include, with discipline. Infix operators are just operators written between their operands — a + b rather than add(a, b). The question is really whether BBL allows user-defined infix operators, not whether + and > exist (obviously they do).

**Call-by-value** as the default. This means a function receives a copy of its argument. It's the most predictable model — callers don't have to worry that passing something to a function will mutate it. This is especially important in business logic where you're frequently passing financial documents, order objects, or configuration records around and the last thing you want is action at a distance.

**Call-by-reference** as an explicit opt-in. There are legitimate performance cases — passing a large internal table to a processing function without copying it — where reference semantics matter. ABAP actually handles this reasonably with its CHANGING and EXPORTING parameter keywords making mutation explicit at the call site. BBL should do the same: a reference parameter should be syntactically visible at the call site, not hidden inside the function signature. Something like a ref keyword on the argument.

## The following philosophical objections apply:
[X] Programmers should not need to understand category theory to write "Hello, World!"
[X] Programmers should not develop RSI from writing "Hello, World!"
[X] The most significant program written in your language is its own compiler
[X] Your compiler errors are completely understandable!
[X] You don't seem to understand basic optimization techniques
[X] You don't seem to understand basic systems programming
[X] You don't seem to understand pointers
[X] You don't seem to understand functions

## Additionally, your marketing has the following problems:
[X] Rejection of orthodox programming-language theory without justification

## Taking the wider ecosystem into account, I would like to note that:
[X] We already have an unsafe imperative language
[X] We already have a safe imperative OO language
[X] You have reinvented Java but worse
[X] You have reinvented C++ but worse
[X] You have reinvented PHP better, but that's still no justification

## In conclusion, this is what I think of you:
[X] This is a bad language, and you should feel bad for inventing it.
[X] Programming in this language is an adequate punishment for inventing it.