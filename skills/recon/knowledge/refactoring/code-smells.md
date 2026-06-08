# Code Smells — Refactoring-Auditor Knowledge Base

This file is the reference catalogue for the refactoring-auditor subagent. Each
entry describes a recognisable symptom pattern — a "code smell" — that signals
a structural weakness in a codebase. The taxonomy follows established
object-oriented design methodology (as systematised by Martin Fowler and Kent
Beck) restated here in original prose for a TypeScript / NestJS / Apollo
GraphQL / Mongoose / React + Vite stack. Every "Fix with:" line cross-references
a `### ` heading in `refactoring-techniques.md`; any technique named there must
appear verbatim as a heading in that companion file.

---

## Bloaters

Bloaters are constructs that have grown so large they are hard to work with.
They accumulate over time as features are added without concurrent refactoring.

### Long Method
**Category:** Bloaters
**Symptoms:** A function or method body that scrolls off the screen; deeply
nested loops or conditionals; multiple comment blocks labelling logical
"phases" within one body; local variables that only live for a single phase.
**Why it hurts:** Long methods are hard to name, test, and reason about in
isolation. Each added feature increases cognitive load non-linearly.
**Fix with:** Extract Method, Extract Variable, Replace Temp with Query, Replace Method with Method Object, Decompose Conditional
**TS signal:** A NestJS service method, resolver, or React hook body exceeding
roughly 60 lines; a `useEffect` callback longer than 20 lines.

### Large Class
**Category:** Bloaters
**Symptoms:** A class with dozens of fields and methods; a module file that
exports a single class but exceeds 400–500 lines; multiple distinct clusters of
fields that are only used together in specific method subsets.
**Why it hurts:** A large class is doing too many things, making it impossible
to change one responsibility without risking breakage in unrelated
functionality.
**Fix with:** Extract Class, Extract Superclass, Extract Interface
**TS signal:** A NestJS service or Mongoose model file exceeding ~300 lines;
a React component file whose default export is a class or function exceeding
~200 lines.

### Primitive Obsession
**Category:** Bloaters
**Symptoms:** Domain concepts like money, email addresses, phone numbers,
coordinates, or date ranges represented as raw `string` or `number` values
rather than typed objects; repeated validation logic scattered wherever the
primitive is consumed.
**Why it hurts:** Each consumer must re-implement the same guard logic,
producing inconsistency and duplication. A single invalid value can propagate
far before being caught.
**Fix with:** Replace Primitive with Object, Introduce Parameter Object,
Replace Magic Number with Symbolic Constant
**TS signal:** DTO fields typed `string` for email/phone/UUID with no branded
type or value-object wrapper; `number` used for currency amounts without a
`Money` or `Decimal` abstraction.

### Long Parameter List
**Category:** Bloaters
**Symptoms:** Functions or constructors with four or more parameters; callers
passing several related arguments that always travel together; booleans as
parameters whose purpose is unclear at the call site.
**Why it hurts:** Long parameter lists are hard to remember and easy to
mis-order, especially when several have the same type. They also signal that
the function may be doing too much.
**Fix with:** Introduce Parameter Object, Preserve Whole Object, Replace Parameter with Method Call
**TS signal:** A NestJS controller method or GraphQL resolver with more than
four positional arguments; a utility function signature with three or more
`boolean` flags.

### Data Clumps
**Category:** Bloaters
**Symptoms:** The same two or three fields appear together in multiple class
definitions, function signatures, and destructuring assignments; removing one
from the group makes the others meaningless.
**Why it hurts:** The implicit grouping is never made explicit, so it cannot
be named, validated, or evolved in one place. Changes ripple across every
site where the clump appears.
**Fix with:** Introduce Parameter Object, Extract Class
**TS signal:** `startDate` + `endDate` + `timezone` passed as three separate
params in more than two places; `lat` + `lng` (or `x` + `y`) scattered across
DTOs and service calls.

---

## OO Abusers

OO Abusers misapply or ignore object-oriented principles, producing code that
fights the language's design rather than working with it.

### Switch Statements
**Category:** OO Abusers
**Symptoms:** A `switch` or long `if-else` chain that discriminates on a type
tag or enum value and dispatches to distinct behaviour blocks; the same
`switch` replicated in multiple places to handle the same set of cases.
**Why it hurts:** Every new variant requires editing every switch site, a
classic Open/Closed violation. Missed cases produce silent `undefined` returns.
**Fix with:** Replace Conditional with Polymorphism
**TS signal:** A `switch` on a discriminated-union `type` field in a NestJS
handler or GraphQL resolver, where each case calls a different service method.

### Temporary Field
**Category:** OO Abusers
**Symptoms:** A class field that is only populated in a particular algorithm's
execution path and is `undefined` or `null` at all other times; fields set
in one method and consumed only in one other, with no meaningful lifecycle
outside that pair.
**Why it hurts:** Consumers of the object cannot tell when the field is safe
to read, leading to defensive null checks everywhere or subtle bugs when the
field is read outside its intended context.
**Fix with:** Extract Class, Introduce Null Object
**TS signal:** A NestJS provider property decorated `@Optional()` or typed
`SomeType | undefined` that is only assigned inside a single private method.

### Refused Bequest
**Category:** OO Abusers
**Symptoms:** A subclass overrides a parent's method only to throw
`new Error('not supported')` or to return a stub; a child class ignores or
suppresses most of the contract it inherited.
**Why it hurts:** Violates the Liskov Substitution Principle: code that
accepts the parent cannot safely use the child, making the hierarchy
deceptive.
**Fix with:** Extract Interface, Replace Inheritance with Delegation
**TS signal:** An NestJS guard or interceptor subclass that overrides the base
implementation but deliberately no-ops most of it; a Mongoose model subclass
that suppresses schema hooks defined in the parent.

### Alternative Classes with Different Interfaces
**Category:** OO Abusers
**Symptoms:** Two or more classes that do the same job but expose incompatible
method names or signatures; adapters written purely to normalise their
differences; callers must know which concrete type they hold to call the right
method.
**Why it hurts:** Duplication and inconsistency compound over time. The two
parallel implementations diverge and must be kept in sync manually.
**Fix with:** Extract Interface, Pull Up Method
**TS signal:** Two NestJS services that both "send a notification" but one has
`send(payload)` and the other has `dispatch(message, recipient)`; two React
data-fetching hooks that return `{ data, loading }` vs. `{ result, isLoading }`.

---

## Change Preventers

Change Preventers make a single logical change expensive by forcing edits in
many unrelated locations.

### Divergent Change
**Category:** Change Preventers
**Symptoms:** A single class is modified for multiple unrelated reasons; the
class has distinct method clusters that evolve independently — e.g. a NestJS
service updated when a new API field is added but also when a database schema
changes.
**Why it hurts:** Any given change touches a class that also contains code for
a different concern, creating merge conflicts and raising the blast radius of
each edit.
**Fix with:** Extract Class, Move Method, Move Field
**TS signal:** A service class whose git log shows commits driven by both
"feature A" and "feature B" with no shared logic between them.

### Shotgun Surgery
**Category:** Change Preventers
**Symptoms:** One logical change requires touching many small, scattered files;
a new field on a domain entity requires updating DTOs, resolvers, service
methods, validators, and tests spread across the whole codebase.
**Why it hurts:** It is easy to miss one of the many sites, leaving the system
in an inconsistent state. High cognitive and coordination cost for each change.
**Fix with:** Move Method, Move Field, Inline Class
**TS signal:** Adding a field to a Mongoose schema requiring parallel edits in
five or more separate files; a GraphQL type change that propagates across
multiple unrelated resolver files.

### Parallel Inheritance Hierarchies
**Category:** Change Preventers
**Symptoms:** Every time a subclass is added to one hierarchy, a paired
subclass must also be added to another; the two trees mirror each other with a
predictable naming convention (e.g. `FooRepository` / `FooService`,
`BarRepository` / `BarService`).
**Why it hurts:** The coupling between the two hierarchies is invisible — there
is no explicit constraint that enforces the pairing, so the trees can silently
drift out of sync.
**Fix with:** Extract Superclass, Move Method, Move Field
**TS signal:** Classes that extend mirrored generic base classes — e.g. `FooRepository extends AbstractRepository<Foo>` paired with `FooService extends AbstractService<Foo>` — such that adding a new subclass to one hierarchy forces adding a corresponding subclass to the other. This does NOT fire merely because every module has an entity, repository, service, and resolver as independent classes with no shared inheritance tree.

---

## Dispensables

Dispensables are things that add noise without adding value — code or
commentary that should not exist.

### Comments
**Category:** Dispensables
**Symptoms:** Block comments that translate the code into prose without adding
domain insight; commented-out code blocks left as "just in case"; TODO comments
that have aged beyond a sprint.
**Why it hurts:** Misleading comments erode trust. Commented-out code
confuses readers and creates maintenance debt without the safety net of version
control history.
**Fix with:** Extract Method, Extract Variable, Rename Method
**TS signal:** A TypeScript file with more comment lines than code lines; stale
`// TODO(2023):` markers; large `/* ... */` blocks of commented-out logic.

### Duplicate Code
**Category:** Dispensables
**Symptoms:** The same or near-identical block appearing in two or more places;
copy-pasted validation logic across DTOs or resolvers; nearly identical React
component trees differing only in a string label.
**Why it hurts:** Every bug fix or behaviour change must be applied to all
copies. Missed copies produce divergent behaviour.
**Fix with:** Extract Method, Pull Up Method, Extract Class
**TS signal:** Two NestJS pipes that share a `validate()` body differing only
in the target class; two React components whose JSX differs only in `title`
and `icon` props.

### Lazy Class
**Category:** Dispensables
**Symptoms:** A class, module, or file that contains so little logic that its
existence is more overhead than value — it delegates everything immediately to
another class without adding meaningful abstraction.
**Why it hurts:** Each indirection layer adds cognitive overhead. Unnecessary
wrappers slow down navigation and obscure where logic actually lives.
**Fix with:** Inline Class, Remove Middle Man
**TS signal:** A NestJS service that is a one-line wrapper around a repository
with no business logic; a React context provider file whose value object is
just a re-export of a Zustand store.

### Data Class
**Category:** Dispensables
**Symptoms:** A class (or interface) that exists only as a property bag — all
fields, a constructor, and possibly some auto-generated getters/setters, with
no behaviour of its own; business logic that belongs on the class lives in its
callers instead.
**Why it hurts:** Behaviour is scattered across consumers rather than
co-located with the data it operates on, making it hard to enforce invariants
and find related logic.
**Fix with:** Extract Method, Move Method
**TS signal:** A Mongoose document interface or NestJS DTO with ten fields and
zero methods, while the corresponding service contains all the logic that
should arguably live on the entity.

### Dead Code
**Category:** Dispensables
**Symptoms:** Unreachable branches (conditions that can never be true given the
type system); exported functions or types with zero import references; entire
files with no live callers; `@deprecated` symbols still present in production
code with no migration plan.
**Why it hurts:** Dead code misleads readers into believing it is relevant. It
increases bundle size and test maintenance burden for zero benefit.
**Fix with:** Remove Dead Code, Inline Class
**TS signal:** TypeScript `never`-typed branches; `ts-unused-exports` or ESLint
`no-unused-vars` warnings; Vite tree-shaking reports showing large unreferenced
modules.

### Speculative Generality
**Category:** Dispensables
**Symptoms:** Abstract base classes with a single concrete subclass; hooks or
parameters added "for future flexibility" with no current caller; generic type
parameters constrained so tightly they only ever resolve to one concrete type.
**Why it hurts:** Unused abstraction is overhead: it must be understood,
maintained, and navigated around without delivering present value.
**Fix with:** Inline Class, Remove Parameter
**TS signal:** A NestJS module designed with a plugin interface used in exactly
one place; a React hook that accepts a `strategy` callback parameter that is
always passed the same function literal at every call site.

---

## Couplers

Couplers create excessive entanglement between classes or modules, making
independent change and testing difficult.

### Feature Envy
**Category:** Couplers
**Symptoms:** A method that spends most of its time accessing fields and
calling methods of another class rather than its own; a function that imports
and reaches deeply into an object from a different domain.
**Why it hurts:** The method clearly belongs elsewhere. As the foreign class
evolves its API, the envious method breaks even though it "lives" in a
different file.
**Fix with:** Move Method, Extract Method
**TS signal:** A NestJS `AuthService` method that primarily reads and mutates
fields on a `UserService` entity model; a React component that destructures
six fields from a context object belonging to another feature domain.

### Inappropriate Intimacy
**Category:** Couplers
**Symptoms:** Two classes that access each other's private or internal state;
circular imports between modules; a resolver that directly queries a
repository that belongs to a different service's domain.
**Why it hurts:** Tight coupling makes independent deployment and testing
impossible. Any change to one class's internals can silently break the other.
**Fix with:** Move Method, Move Field, Hide Delegate, Extract Interface
**TS signal:** Circular `import` warnings from the TypeScript compiler or
NestJS module resolver; a service importing another module's repository
directly rather than going through the owning service.

### Message Chains
**Category:** Couplers
**Symptoms:** A series of chained member accesses: `a.b().c().d().getValue()`;
callers forced to know the internal structure of an object graph just to
reach a leaf value; the Law of Demeter violated repeatedly.
**Why it hurts:** The caller is coupled to every intermediate step. If any
link in the chain changes its shape, every chain that passes through it breaks.
**Fix with:** Hide Delegate, Extract Method
**TS signal:** Single-object navigation chains on one receiver, such as
`ctx.req.user.profile.merchant.id` or `context.req.user.profile.settings.locale`
in an Apollo resolver. This does NOT fire for React prop drilling (passing a
prop through intermediate components), which is a distinct smell unrelated to
object-graph traversal chains.

### Middle Man
**Category:** Couplers
**Symptoms:** A class or service whose methods are almost entirely delegating
calls to another class, adding no logic, transformation, or abstraction of
their own; a facade that has become a transparent passthrough.
**Why it hurts:** Pure delegation layers add indirection overhead without value.
Callers would be better served calling the delegate directly or having the
façade merged into it.
**Fix with:** Remove Middle Man, Inline Class
**TS signal:** A NestJS controller that has more than three methods each
consisting solely of `return this.service.sameMethodName(...args)` with no
transformation or guard logic.
