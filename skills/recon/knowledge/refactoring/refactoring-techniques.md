# Refactoring Techniques — Refactoring-Auditor Knowledge Base

This file catalogues the behaviour-preserving transformation techniques that
the refactoring-auditor subagent proposes as fixes. Technique names appear
verbatim as `### ` headings and are cross-referenced by `Fix with:` lines in
`code-smells.md` — every name used there must match a heading here exactly.
The taxonomy draws on the established object-oriented refactoring body of work
(Fowler, Beck, and others); all descriptions are original prose.

Each entry states the group the technique belongs to, the situation that
triggers it, a concise numbered set of behaviour-preserving steps, and the
single most common way the transformation breaks behaviour when done hastily.

This file is a reference library; it may include techniques not currently
cross-referenced by any smell entry — such entries are not errors, but
additional tools available for use when appropriate.

---

## Composing Methods

### Extract Method
**Group:** Composing Methods
**When:** A contiguous block inside a longer method forms a coherent sub-task
that can be named; there are comments labelling a phase; or the block is
duplicated elsewhere.
**Mechanics:**
1. Identify the contiguous block and decide on a descriptive name for the new method.
2. Determine all local variables in the block that are read but not declared inside it — these become parameters.
3. Determine whether the block sets a local variable used after it — if exactly one, that becomes the return value.
4. Create the new method with the chosen name, parameters, and return type.
5. Replace the block with a call to the new method, passing the required arguments.
6. Run tests; inline the method and retry if the name reveals a poor abstraction.
**Watch out:** Modifying a variable that is used in multiple subsequent places — returning one value is safe; returning multiple signals the block is not yet a single coherent concept.

### Inline Method
**Group:** Composing Methods
**When:** A method's body is as clear as its name, or the method is an
unnecessary relay that only calls one other method and adds no clarity.
**Mechanics:**
1. Confirm the method is not polymorphic (not overridden in any subclass).
2. Find every call site for the method.
3. At each call site, replace the call with the method's body, substituting parameters for arguments.
4. Delete the now-unused method.
5. Run tests.
**Watch out:** Inlining a method that is called from inside a loop — duplicate the body only if the call was already inside the loop; otherwise beware of unintended duplication of side effects.

### Extract Variable
**Group:** Composing Methods
**When:** A complex expression is hard to parse at a glance; the same
sub-expression appears more than once in nearby code; or an intermediate
result should be given a meaningful domain name.
**Mechanics:**
1. Identify the expression to name.
2. Declare a `const` variable with a descriptive name and assign the expression to it.
3. Replace all occurrences of the exact expression in the same scope with the variable.
4. Run tests.
**Watch out:** The expression having side effects (e.g. incrementing a counter) — extracting it to a variable changes the number of times the side effect runs if there were multiple occurrences.

### Replace Temp with Query
**Group:** Composing Methods
**When:** A temporary local variable holds the result of an expression that
could be computed on demand; the variable is assigned once and only read, not
modified.
**Mechanics:**
1. Verify the temporary is assigned exactly once before each use and the expression is pure (or idempotent in context).
2. Extract the right-hand expression into a private method (or getter in TypeScript) with a clear name.
3. Replace every reference to the temp with a call to the new method/getter.
4. Remove the temporary variable declaration.
5. Run tests.
**Watch out:** Expensive expressions — replacing a temp with a query may cause repeated computation; verify performance is acceptable or memoize as needed.

### Replace Method with Method Object
**Group:** Composing Methods
**When:** A long method that cannot be cleanly decomposed with Extract Method
because it uses many local variables that would become an unwieldy parameter
list if extracted separately.
**Mechanics:**
1. Create a new class named for what the method does.
2. Add a field for each local variable and for the object the original method lived on (if needed).
3. Add a constructor that takes the original method's arguments and initialises the fields.
4. Create a single `compute()` (or semantically named) method containing the original body, using `this.fieldName` instead of locals.
5. Replace the original method's body with construction of the new class and a call to `compute()`.
6. Run tests, then apply Extract Method freely within the new class.
**Watch out:** Passing mutable state via the constructor — the method object captures state at construction time; mutations after construction are not reflected.

### Substitute Algorithm
**Group:** Composing Methods
**When:** A working but convoluted algorithm can be replaced wholesale with a
cleaner implementation that produces identical results; a library function
already exists that covers the same ground.
**Mechanics:**
1. Document the expected input/output contract and write characterisation tests.
2. Write the replacement algorithm in a separate function.
3. Run both algorithms against the same test suite to confirm identical outputs.
4. Swap the implementation: replace the old body with the new one.
5. Remove the old implementation and run full tests.
**Watch out:** Edge cases the original algorithm handled implicitly — the characterisation tests must cover boundary inputs (empty collections, null values, numeric extremes).

### Remove Dead Code
**Group:** Composing Methods
**When:** Code is unreachable (a branch whose condition can never be true given
the type system) or has zero live references (exported symbols with no
importers, private methods never called, entire files with no callers).
**Mechanics:**
1. Confirm there are no references via grep and the TypeScript type-checker; also check for dynamic, reflective, or dependency-injection access that static search would miss.
2. Delete the dead code (branch, function, file, or type declaration).
3. Remove any imports that are now unused as a result.
4. Run the build and the full test suite to verify nothing is broken.
**Watch out:** Reflection, dependency-injection registration (e.g. NestJS `@Injectable()` providers wired up by string token), or string-keyed dynamic access — static reference search alone does not detect these usages.

---

## Moving Features

### Move Method
**Group:** Moving Features
**When:** A method uses data from another class more than from its own class;
the logic belongs closer to the data it operates on.
**Mechanics:**
1. Inspect whether the method uses fields or methods of another class more than its own.
2. Create the method on the target class, accepting the source object as a parameter if needed to maintain access.
3. Update the source method to delegate to the new location (to avoid breaking callers).
4. Once all callers use the new location, remove the source method or make it an alias.
5. Run tests.
**Watch out:** The method referring to `this` of the source class — ensure all necessary source-class data is passed explicitly or the source object is passed as a parameter.

### Move Field
**Group:** Moving Features
**When:** A field is used more often by another class than by the class that
owns it; it logically belongs with a different set of data.
**Mechanics:**
1. Create the field on the target class with an appropriate access modifier.
2. In the source class, replace every direct access to the field with an accessor that reads from the target.
3. Once all accesses go through the target, remove the field from the source class.
4. Clean up any now-unnecessary accessors.
5. Run tests.
**Watch out:** The field being part of a serialisation contract (e.g. a Mongoose schema or a GraphQL type) — moving it changes the wire/storage format and may require a migration.

### Extract Class
**Group:** Moving Features
**When:** A class is doing the work of two; a subset of fields and methods
form a coherent cluster that could stand alone as a named concept.
**Mechanics:**
1. Decide what to move and choose a name for the new class.
2. Create the new class and establish a reference from the original.
3. Move each field and method from the original to the new class, compiling and testing after each move.
4. Expose only what is necessary from the new class (keep internals private).
5. Update all external references to go through the new class where appropriate.
**Watch out:** Moving a field that is directly serialised — update schema definitions, migrations, and GraphQL types simultaneously.

### Inline Class
**Group:** Moving Features
**When:** A class is no longer carrying its weight — it has been stripped of
most responsibility and is now just a thin relay to another class.
**Mechanics:**
1. Identify all public features of the class to be inlined and recreate them on the target class.
2. Change all references from the slim class to point directly to the target.
3. Remove the now-empty class.
4. Run tests.
**Watch out:** Removing a class that is registered as a NestJS provider or Mongoose discriminator — dependency injection references must be updated in module metadata.

### Hide Delegate
**Group:** Moving Features
**When:** A client accesses an object through a chain of navigations; the
intermediate object should shield the client from the implementation details
of the downstream object.
**Mechanics:**
1. For each method on the delegate that the client calls via the intermediary, add a delegating method on the intermediary.
2. Update each client to call the new delegating method instead of chaining.
3. Verify no client accesses the delegate object directly any more.
4. Run tests.
**Watch out:** Adding too many delegating methods that expose the entire delegate interface — this defeats the encapsulation goal; expose only what clients actually need.

### Remove Middle Man
**Group:** Moving Features
**When:** A class is so full of delegating methods that it no longer provides
any real abstraction; clients would be better off calling the delegate directly.
**Mechanics:**
1. Create an accessor on the intermediary for the delegate object.
2. For each delegating method, update callers to call the delegate via the accessor, then remove the delegating method.
3. Once all delegating methods are gone, remove the accessor if no longer needed.
4. Run tests.
**Watch out:** The intermediary performing access control or logging before delegation — removing the middle man also removes those cross-cutting concerns; confirm they are no longer required or move them to the call sites.

---

## Organizing Data

### Replace Primitive with Object
**Group:** Organizing Data
**When:** A raw primitive value (string, number) carries domain meaning that
has associated behaviour or validation — such as email addresses, monetary
amounts, or identifiers.
**Mechanics:**
1. Create a small value-object class (or branded type in TypeScript) that wraps the primitive.
2. Add a constructor that validates the value; expose an accessor for the raw form.
3. Replace each field/parameter typed as the primitive with the new type.
4. Migrate call sites to construct and use the value object.
5. Run tests.
**Watch out:** Serialisation/deserialisation boundaries (JSON, Mongoose) — ensure the value object serialises to the expected primitive representation and Mongoose schema definitions match.

### Introduce Parameter Object
**Group:** Organizing Data
**When:** Several parameters always appear together across multiple function
signatures, signalling that they form a concept worth naming.
**Mechanics:**
1. Create an interface or class representing the grouped parameters.
2. Add the new parameter object to each function signature (temporarily alongside the originals).
3. Update callers to construct and pass the new object.
4. Remove the individual parameters from the signatures once all callers pass the object.
5. Run tests.
**Watch out:** Partially overlapping groups — if not all callers pass all members of the group, the abstraction may not be universal; check before committing to the grouping.

### Replace Magic Number with Symbolic Constant
**Group:** Organizing Data
**When:** A literal number (or string) appears in code whose meaning is not
self-evident from context; the value is used in multiple places.
**Mechanics:**
1. Declare a `const` (or `enum` member) with a descriptive name and assign the literal value.
2. Replace each occurrence of the literal with the named constant.
3. Run tests.
**Watch out:** Magic numbers that are part of a protocol or serialisation format — changing the value of the constant must be intentional; naming it makes accidental changes more visible, which is the goal.

### Encapsulate Field
**Group:** Organizing Data
**When:** A public field is accessed directly by external code, bypassing any
possibility of validation or change notification.
**Mechanics:**
1. Create getter and setter methods (or a TypeScript `get`/`set` accessor pair) for the field.
2. Change the field's access modifier to `private`.
3. Update all external accesses to use the new accessors.
4. Run tests, then add validation or side-effects to the accessor if needed.
**Watch out:** TypeScript's structural typing means a field and a getter are sometimes interchangeable at the type level, but not always — verify that decorators (e.g. `@Expose()` from class-transformer) still apply correctly.

### Encapsulate Collection
**Group:** Organizing Data
**When:** A class exposes a mutable collection field directly, allowing clients
to add or remove items without the class knowing; invariants on the collection
can then be violated externally.
**Mechanics:**
1. Replace direct access to the collection field with `add`, `remove`, and read-only access methods.
2. Return an immutable view (or a copy) from the getter rather than the raw collection reference.
3. Update all callers to use the new methods.
4. Run tests.
**Watch out:** Performance-sensitive code iterating large collections — returning a copy on every read may be costly; consider returning a read-only proxy or an iterator instead.

---

## Simplifying Conditionals

### Decompose Conditional
**Group:** Simplifying Conditionals
**When:** A conditional branch (or its body) is complex enough to obscure
intent; the condition itself and the then/else branches would be clearer as
named methods.
**Mechanics:**
1. Extract the condition expression into a method whose name states what is being checked (e.g. `isEligibleForDiscount()`).
2. Extract the then-branch body into a method named for what it does.
3. Extract the else-branch body similarly.
4. Replace the original conditional with calls to the extracted methods.
5. Run tests.
**Watch out:** Conditions with side effects — extracting them to a method changes the point at which the side effect fires relative to surrounding code.

### Consolidate Conditional Expression
**Group:** Simplifying Conditionals
**When:** Multiple separate `if` checks all lead to the same result action and
could be expressed as one combined condition.
**Mechanics:**
1. Confirm each check is independent and the combined result is the same action.
2. Combine the checks using `&&` or `||` as appropriate, using De Morgan's law where needed.
3. Extract the combined condition into a named method if still complex (see Decompose Conditional).
4. Run tests.
**Watch out:** Side-effecting conditions — short-circuit evaluation (`||` / `&&`) changes which side effects execute; only consolidate purely boolean sub-expressions.

### Replace Nested Conditional with Guard Clauses
**Group:** Simplifying Conditionals
**When:** A method has a deeply nested conditional structure that makes the
normal (happy) path hard to see; abnormal paths (null checks, precondition
failures) are interleaved with the main logic.
**Mechanics:**
1. Identify every special-case or precondition check at the top of the conditional.
2. Convert each into an early-return guard clause at the top of the method.
3. Remove the corresponding `else` branch.
4. Repeat until the nesting depth is reduced to a single level for the main path.
5. Run tests.
**Watch out:** Guard clauses that share a resource cleanup responsibility with code at the end of the method — ensure the cleanup runs in all exit paths (consider `try/finally` or `defer`-style patterns).

### Replace Conditional with Polymorphism
**Group:** Simplifying Conditionals
**When:** A `switch` or long `if-else` chain dispatches on a type tag to
perform type-specific behaviour; the same tag check recurs in multiple places.
**Mechanics:**
1. Define an interface (or abstract class) with the shared method contract.
2. Create a concrete class (or subclass) for each variant, implementing the method.
3. Replace each branch of the conditional with a call to the polymorphic method.
4. Ensure all consumers depend on the interface rather than the concrete type.
5. Run tests and remove the conditional.
**Watch out:** Variants that need to share state from the host object — the new subclasses must receive that state via constructor injection or a method parameter, not by reaching back into the original class.

### Introduce Null Object
**Group:** Simplifying Conditionals
**When:** Code is littered with null checks before calling a method on an
optional object; the null case always produces a default (often no-op)
behaviour.
**Mechanics:**
1. Create a `NullFoo` class (or a `None`-typed singleton) implementing the same interface as the real object.
2. Implement each method as a safe no-op or default return value.
3. Replace every site that creates or returns `null` / `undefined` with the null object.
4. Remove the null guards from calling code.
5. Run tests.
**Watch out:** Code that genuinely needs to distinguish the null case from the real case — the null object pattern hides the distinction; do not apply it where the caller must act differently when nothing is present.

---

## Simplifying Method Calls

### Rename Method
**Group:** Simplifying Method Calls
**When:** A method's name does not communicate what it does; the name reflects
the implementation rather than the intent; or a name that was adequate has
been outgrown by evolving usage.
**Mechanics:**
1. Decide on a better name that expresses intent.
2. Create a new method with the new name containing the original body (or, in TypeScript, use IDE rename-symbol).
3. Make the old method delegate to the new one.
4. Update all call sites to use the new name.
5. Remove the old method.
6. Run tests.
**Watch out:** Method names that form part of a public API or a GraphQL schema — renaming them is a breaking change for clients; version the API or add a deprecated alias.

### Add Parameter
**Group:** Simplifying Method Calls
**When:** A method needs information from its caller that it currently lacks,
and that information cannot be obtained from the object's own state.
**Mechanics:**
1. Add the new parameter to the method signature with a sensible type.
2. Update all call sites to supply the argument (provide a default where backward compatibility is required).
3. Use the new parameter within the method body.
4. Run tests.
**Watch out:** Adding a required parameter to a widely-used method — prefer adding it with a default value or overload signature first to allow incremental migration.

### Remove Parameter
**Group:** Simplifying Method Calls
**When:** A parameter is no longer used within the method body; it never was
used and was added speculatively.
**Mechanics:**
1. Confirm the parameter is not used anywhere inside the method.
2. Remove it from the signature.
3. Update all call sites to stop passing the argument.
4. Run tests.
**Watch out:** Removing a parameter from an overriding method in a class hierarchy — all overrides and the interface definition must be updated simultaneously.

### Preserve Whole Object
**Group:** Simplifying Method Calls
**When:** Code extracts several values from an object and passes them
individually to a method, when the method could simply receive the object.
**Mechanics:**
1. Check that the method is permitted to have a dependency on the object's type (consider coupling).
2. Replace the individual parameters with the whole object parameter.
3. Update the method body to read values from the object directly.
4. Remove the now-unused individual parameters.
5. Run tests.
**Watch out:** Introducing an inappropriate coupling — only pass the whole object if the method belongs in the same layer or module as the object's type; passing an HTTP request object deep into a domain service is an anti-pattern.

### Replace Parameter with Method Call
**Group:** Simplifying Method Calls
**When:** A method receives a parameter whose value the method could compute
itself by calling another method; the parameter is always derived from the
same calculation at every call site.
**Mechanics:**
1. Extract the calculation into a separate method if it is not already one.
2. Have the target method call the calculation method internally.
3. Remove the parameter from the target method signature.
4. Update call sites to stop passing the now-computed value.
5. Run tests.
**Watch out:** The calculation being expensive or having side effects — moving it inside the method changes when and how often it runs.

---

## Dealing with Generalization

### Pull Up Method
**Group:** Dealing with Generalization
**When:** Two or more subclasses implement the same method with identical (or
near-identical) bodies; the method belongs in the superclass.
**Mechanics:**
1. Confirm the methods are genuinely identical (or can be made so with small adjustments).
2. Move one copy to the superclass.
3. Remove the duplicate from each subclass.
4. Run tests against all subclass instances.
**Watch out:** Methods that reference fields present only in the subclass — pull up the fields first (Pull Up Field), or pass them as parameters to the pulled-up method.

### Pull Up Field
**Group:** Dealing with Generalization
**When:** Two or more subclasses declare the same field; it logically belongs
in the shared superclass.
**Mechanics:**
1. Inspect all usages across the subclasses to confirm the field means the same thing in each.
2. Declare the field in the superclass.
3. Remove the duplicate declarations from each subclass.
4. Run tests.
**Watch out:** Subclasses with different initial values for the field — ensure the superclass constructor or default satisfies all subclasses, or introduce an overridable factory method.

### Push Down Method
**Group:** Dealing with Generalization
**When:** A method in a superclass is only relevant to one subclass and is
never called (or never should be called) in the context of the others.
**Mechanics:**
1. Copy the method to the relevant subclass.
2. Remove it from the superclass.
3. Remove any now-abstract declaration from the superclass interface/type if no other subclass uses it.
4. Run tests.
**Watch out:** Removing the method from the superclass when it is still referenced via the superclass type elsewhere — ensure no callers depend on the superclass having this method before removing it.

### Extract Superclass
**Group:** Dealing with Generalization
**When:** Two classes share similar behaviour; a common abstraction would
eliminate duplication and make the relationship explicit.
**Mechanics:**
1. Create a new superclass.
2. Move shared fields and methods to it (Pull Up Field, Pull Up Method).
3. Make both classes extend the superclass.
4. Run tests.
**Watch out:** TypeScript does not support multiple inheritance — if either class already extends another, use Extract Interface or Replace Inheritance with Delegation instead.

### Extract Interface
**Group:** Dealing with Generalization
**When:** Clients only use a subset of a class's interface; or several classes
implement the same contract without a formal shared type.
**Mechanics:**
1. Create an interface declaring only the methods/properties the clients need.
2. Have the class (and any others that implement the same contract) declare that it implements the interface.
3. Update client code to depend on the interface type, not the concrete class.
4. Run tests.
**Watch out:** Adding too many members to the interface (the Interface Segregation Principle) — a broad interface is little better than the concrete type; keep interfaces focused on a single client's needs.

### Replace Inheritance with Delegation
**Group:** Dealing with Generalization
**When:** A subclass uses only part of the superclass interface and overrides
or suppresses the rest (Refused Bequest); or composition would model the
relationship more accurately than inheritance.
**Mechanics:**
1. Create a field on the subclass that holds an instance of the former superclass.
2. Adjust each method that used to rely on inherited behaviour to delegate to the field.
3. Remove the `extends` declaration.
4. Run tests.
**Watch out:** The subclass being used polymorphically via the superclass type — removing `extends` breaks this; ensure callers depend on a shared interface rather than the concrete superclass before making the change.
