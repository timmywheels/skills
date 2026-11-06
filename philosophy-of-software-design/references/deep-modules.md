# Deep Modules and Information Hiding

Detail for sections 1–6 of `SKILL.md`, covering chapters 4–9 of the book, with TypeScript translations. The book's examples are kept where they teach best; the TypeScript examples are new.

## Table of Contents

- [Measuring Depth](#measuring-depth)
  - [Abstractions Fail Two Ways](#abstractions-fail-two-ways)
- [Shallow Modules in TypeScript](#shallow-modules-in-typescript)
  - [Pass-Through Services](#pass-through-services)
  - [Wrappers That Hide Nothing](#wrappers-that-hide-nothing)
  - [Components and Hooks](#components-and-hooks)
  - [Classitis](#classitis)
- [Make the Common Case Simple](#make-the-common-case-simple)
- [Information Hiding](#information-hiding)
  - [What to Hide](#what-to-hide)
  - [Leakage You Can See, and Leakage You Can't](#leakage-you-can-see-and-leakage-you-cant)
  - [Temporal Decomposition](#temporal-decomposition)
- [Somewhat General-Purpose Interfaces](#somewhat-general-purpose-interfaces)
  - [Specialization Up and Down](#specialization-up-and-down)
  - [Special Cases](#special-cases)
- [Layers and Pass-Throughs](#layers-and-pass-throughs)
  - [Pass-Through Variables](#pass-through-variables)
- [Pull Complexity Downward](#pull-complexity-downward)
- [Together or Apart](#together-or-apart)
  - [Splitting and Joining Methods](#splitting-and-joining-methods)

---

## Measuring Depth

Depth compares what a caller gets with what a caller must know. To judge it, list the interface's full cost, informal parts included:

| Cost | Ask |
|---|---|
| Entry points | How many functions, methods, props, or endpoints? |
| Parameters | How many, how many does a typical caller set, and do any interact? |
| Types | How many new types must a caller learn to make one call? |
| Errors | What can fail, and what should a caller do about each failure? |
| Ordering | Must calls happen in an order? Is there an `init` or a `dispose`? |
| Side effects | What changes that the signature doesn't mention: rows, caches, events, global state? |
| Performance | Must the caller know it's slow, batch calls, or avoid calling it in a loop? |
| Configuration | What must be set up before it works? |

Then list the benefit: which decisions does the module make that callers would otherwise make, and what would callers have to write without it?

When the two lists are about the same length, the module is shallow. When the benefit list dwarfs the cost list, it's deep.

### Abstractions Fail Two Ways

An abstraction is "a simplified view of an entity which omits unimportant details." It can go wrong in two directions:

- **It includes details that don't matter.** The interface is bigger than it needs to be, and every caller pays the cognitive load.
- **It omits details that do matter.** This is a *false abstraction*: the interface looks simple, but callers can't use it correctly without reading the implementation. It creates unknown unknowns, and it's the worse failure.

This is why "fewer parameters" is not the same as "simpler." A function that silently depends on a global, or that only works if called after another function, has a hidden parameter.

---

## Shallow Modules in TypeScript

### Pass-Through Services

```typescript
export async function getMember(id: string) {
  return memberRepository.findById(id);
}

export async function updateMember(id: string, patch: Partial<Member>) {
  return memberRepository.update(id, patch);
}
```

Fixes, in order of preference:

1. **Delete the layer.** Callers use `memberRepository` directly.
2. **Give the layer a decision to own.** If the service exists to enforce access rules or business invariants, enforce them here and nowhere else. Then it isn't a pass-through.
3. **Merge the layers.** If the repository exists only to serve this service, make one module.

A layer is justified by the decisions it owns, not by the architecture diagram. "Every feature has a controller, a service, and a repository" produces pass-throughs whenever a feature is simple.

### Wrappers That Hide Nothing

```typescript
export function formatDate(date: Date, pattern: string) {
  return format(date, pattern);
}
```

This adds a name to learn and a file to open, and hides nothing. Compare a wrapper that owns a decision every caller would otherwise make:

```typescript
export function formatForGym(date: Date, gym: { timezone: string }) {
  return formatInTimeZone(date, gym.timezone, "MMM d, yyyy");
}
```

The rule "gym-facing dates render in the gym's timezone, in this format" now lives in one place. Same size, very different depth.

### Components and Hooks

A component's props are its interface. When every caller has to wire up the same internal state, the component is shallow:

```tsx
const [open, setOpen] = useState(false);
const [query, setQuery] = useState("");
const [selectedId, setSelectedId] = useState<string | null>(null);

<MemberPicker
  open={open}
  onOpenChange={setOpen}
  query={query}
  onQueryChange={setQuery}
  members={members.filter((m) => matches(m, query))}
  selectedId={selectedId}
  onSelect={setSelectedId}
/>
```

The deep version keeps the open state, the search query, and the filtering inside, and exposes what callers care about:

```tsx
<MemberPicker value={memberId} onChange={setMemberId} />
```

A caller that really needs to control the open state can get an optional controlled prop. The common case shouldn't pay for it.

Hooks follow the same rule. A hook that returns ten setters for its internal state is a shallow module; one that returns the operations callers need (`submit`, `reset`) and the state they render is deep.

Two frontend leaks deserve names, because they're common and the book's vocabulary covers them only indirectly:

- **Hidden channels.** A component that reads its configuration from a global store, which each caller must write before rendering it, has a hidden parameter. "Write the store, then render" is part of its interface, but its props don't say so: a false abstraction. Take the value as a prop, and if other code needs the store, have the component own the write.
- **Two sources of truth.** When a form's render conditions decide which fields appear and a separate list decides which fields get validated and submitted, one decision lives in two places. A field shown but left out of the list is silently never validated. Derive both from one function: `const shown = shownFields(config)`, then render and validate from `shown`.

### Classitis

Classitis is the belief that more, smaller units are automatically better. In TypeScript codebases it looks like:

- A `utils/` folder of one-line functions, each a name to learn and none hiding a decision.
- An interface, a class, a factory, and a DI token for something with one implementation that will never have a second.
- One file per tiny function, so reading one feature means opening ten files.
- A new class per variation instead of one class that handles the variations internally.

Each unit adds an interface. Ask what decision each one hides; if the answer is "none," fold it into its caller or callee.

---

## Make the Common Case Simple

"Interfaces should be designed to make the most common usage as simple as possible."

- **Defaults for everything rare.** An options object where every field is optional and the defaults are what most callers want.
- **Don't make common callers opt into the common behavior.** Java's I/O makes you stack `FileInputStream`, `BufferedInputStream`, and `ObjectInputStream` to get buffering that almost everyone wants. Unix makes the common case (sequential reads and writes) need no extra calls; random access costs one more call, `lseek`.
- **A separate entry point for the rare case** is often better than a parameter on the common one. `charge(invoice)` and `chargeWithoutRetry(invoice)` beat `charge(invoice, { retry: false })` only when the rare case really is rare and different. When it isn't, a defaulted option is fine.

From the book's HTTP server project:

- **Parameter handling.** Students did two things well: they merged parameters from the URL and the body, since handlers don't care where a parameter came from, and they decoded URL encoding so callers got readable values. The weak choice was a method that returned the internal parameter map, which exposed the representation, made every lookup two steps, and let callers modify internal state. Better: `getParameter(name)` and `getIntParameter(name)`, which hide storage and conversion.
- **Defaults.** Students made callers specify the HTTP version of each response, though the library can take it from the request. The `Date` header deserves a default too. "The best features are the ones you get without even knowing they exist."

---

## Information Hiding

Information hiding comes from David Parnas's 1972 paper "On the Criteria to Be Used in Decomposing Systems into Modules." It is the main tool for making modules deep: the more a module hides, the less its callers need to know.

### What to Hide

- **Data structures and formats:** the shape of a stored JSON column, a file format, a cache key, a wire format.
- **Algorithms:** how a price is prorated, how a schedule is generated, how matches are ranked.
- **Protocols:** the order of calls to a provider, pagination, retries, rate limits.
- **Policies:** which state transitions are legal, which failures retry, who may see what.
- **Lower-level mechanisms:** which queue, which database, which SDK, which version.

### Leakage You Can See, and Leakage You Can't

**Overt leakage** shows up in an interface:

- A function returns raw database rows, so the table schema is now everyone's API.
- A function takes a provider SDK type as a parameter, so every caller depends on the SDK.
- An exported constant or enum that callers must pass back in exactly.

**Back-door leakage** is worse because nothing points to it:

- Two modules build the same cache key or queue name from a format string.
- A frontend and a backend each keep their own status-to-label map.
- Two services each parse the same JSON column.
- Code that relies on the order of an array or the numeric values of an enum that another module produces.

Fixes:

1. **Merge the modules** that share the knowledge, if they are really one concern.
2. **Extract a module that owns the decision,** and have both use it: `profileCache`, `describeLineItem`, `parseMemberSettings`. This only helps if the new module's interface is simple and hides most of the details; a new module that exposes the format just moves the leak.
3. **When the leak crosses a boundary you can't merge** (frontend and backend, two services), make the shared knowledge explicit in one contract both sides import or generate from: a schema package, OpenAPI types, a shared enum.

### Temporal Decomposition

A member-import feature organized by execution order:

```text
import/
  read-file.ts       knows the encoding and delimiter
  parse-rows.ts      knows the column order, header names, date format
  validate-rows.ts   knows the column order, date format, required columns
  save-rows.ts       knows how columns map to member fields
```

Every step knows the file format, so changing a column means changing four files. Organized by knowledge instead:

```text
import/
  member-import-file.ts   owns the format: encoding, headers, columns, dates → MemberImportRow
  member-import.ts        owns the policy: validation rules, duplicates, what gets saved
```

The order of operations still exists; it lives inside a module instead of defining the boundaries between modules.

The book's version: students building an HTTP server put reading a request and parsing it in separate classes. Reading a request requires parsing much of it, because the `Content-Length` header says how much body to read. So both classes understood HTTP, the parsing code was duplicated, and callers had to call the two in the right order. One class that owns the request format gives callers a single, simpler method.

Order of operations can still show up in a program's behavior. It should drive module boundaries only when the phases really use different information.

**Information hiding within a module** matters too. Minimize the number of places each field or variable is used, so less of the module's own code depends on each detail.

**Taking it too far:** hide only what callers don't need. If callers need something to use the module well, such as a setting that affects performance in a way only they can judge, it belongs in the interface.

---

## Somewhat General-Purpose Interfaces

The sweet spot is a module whose *functionality* covers today's needs and whose *interface* is general enough for several uses. General-purpose interfaces tend to be simpler, deeper, and better at hiding information, even when used for one purpose.

The book's text-editor example, translated:

```typescript
interface TextDocument {
  backspace(cursor: Cursor): void;
  delete(cursor: Cursor): void;
  deleteSelection(selection: Selection): void;
}
```

Each method mirrors one UI action, each is used once, and the text class has to know about cursors and selections. The general version:

```typescript
interface TextDocument {
  insert(at: Position, text: string): void;
  delete(start: Position, end: Position): void;
  changePosition(position: Position, characters: number): Position;
}
```

Backspace moves into the UI, as one line:

```typescript
doc.delete(doc.changePosition(cursor, -1), cursor);
```

The UI code is slightly longer and much more obvious, and the text class can now serve a search-and-replace feature it was never designed for.

The three questions to ask:

1. **What is the simplest interface that will cover all my current needs?** Reducing the number of methods helps only if each stays simple. Merging methods by adding parameters doesn't simplify anything.
2. **In how many situations will this method be used?** If one, it's probably too specialized. Try replacing several special-purpose methods with one general one.
3. **Is this API easy to use for my current needs?** If callers need loops or glue code, it's too low-level. A text class with only single-character operations is general but makes every caller loop.

### Specialization Up and Down

Some specialization is unavoidable. Keep it out of the general mechanism by pushing it:

- **Up,** into the top-level code that knows the use case: the UI, the route handler, the feature module.
- **Down,** into adapters that implement a general interface: device drivers implement block reads and writes so the operating system's core knows nothing about specific devices.

Ousterhout's undo design has three parts with three owners: a general `History` that records actions and groups them with "fences", the specific actions (`UndoableInsert`, `UndoableSelection`), each of which knows how to undo and redo itself, and the grouping policy, which lives in UI code. Mixing them into the text class leaks UI concerns into text storage.

A TypeScript example: notifications built as `sendWelcomeEmail(member)`, `sendPaymentFailedEmail(member, invoice)`, `sendClassReminderSms(member, booking)`, each with its own copy of the opt-out check, quiet hours, and transport. The general mechanism, `notify(recipient, message)`, owns opt-out, quiet hours, and delivery. The specialization goes up (which message, when, with what content) and down (email and SMS adapters).

### Special Cases

Special cases add `if` statements, and each `if` is a branch to understand, test, and get wrong. Design the normal path so the special case is just a normal input:

| Special case | Designed out |
|---|---|
| `selection: Selection \| null`, checked everywhere | An empty selection whose start equals its end |
| `items?: Item[]`, with `items ?? []` at every use | `items: Item[]` that defaults to `[]` at the boundary |
| `logger?: Logger`, with `logger?.info(...)` everywhere | A no-op logger by default |
| First iteration handled separately in a loop | A sentinel or an initial value that makes the first iteration normal |

Don't erase a distinction callers need. If `null` means "unknown" and `0` means "none," they are different facts, and folding one into the other is a false abstraction.

---

## Layers and Pass-Throughs

Recognizing the problem: adjacent layers whose methods have the same names and the same arguments. A controller calls `service.updateMember(id, patch)`, which calls `repository.updateMember(id, patch)`. The book's example is a student text editor whose `TextDocument` class forwarded 13 of its 15 public methods to `TextArea`, a sign that nobody had decided which class was responsible for what.

Fixes for a pass-through method:

1. **Expose the lower layer directly** to the callers who need it.
2. **Redistribute functionality** so each layer does something distinct.
3. **Merge the layers.**

**When duplicate signatures are fine:**

- **Dispatchers.** A method with the same arguments that picks which implementation to run: a webhook handler that routes `event.type` to per-type handlers, a router.
- **Multiple implementations of one interface.** Payment-provider adapters, storage drivers, strategies. The shared signature is the point.

**Decorators** (wrappers that add one feature to an existing object) tend to be shallow and multiply. Before writing one, consider:

- adding the feature directly to the underlying module, if it's general-purpose;
- merging it into the one use case that needs it, if it's specific;
- merging it with an existing decorator;
- implementing it as a separate standalone module, if it isn't really tied to the wrapped one.

A wrapper is most justified when it adapts an external module whose interface you can't change, and even that is less common than it looks.

**The interface should differ from the implementation.** The book's example is a text class that stores lines internally; its interface should still offer character ranges if that's what callers need, rather than making them split and join lines. In TypeScript the common version is an API module that mirrors its table: one endpoint per column update, a `status` field any caller can write, a response shaped like a row.

### Pass-Through Variables

A variable threaded through a chain of signatures where only the deepest function uses it: a logger, a request ID, a tenant ID, a feature-flag client, a transaction. Every function in the chain now has a parameter it doesn't use, and adding a new one means editing all of them.

The book considers three fixes: a shared object that every layer already has, a global (rejected, because it prevents two instances in one process, such as in tests), and a **context object** that holds the system's cross-cutting state and is stored in long-lived objects rather than passed on every call.

TypeScript equivalents:

- **Server:** a request-scoped context: `AsyncLocalStorage`, or the framework's request object with decorated properties.
- **React:** context providers for values many components need.
- **Long-lived objects:** inject dependencies once at construction instead of passing them per call.

Context objects are not free. They make dependencies implicit, so a reader can't see from a signature what a function relies on. Keep them small, make their values immutable where possible, and don't let them become a bag of everything.

---

## Pull Complexity Downward

"It is more important for a module to have a simple interface than a simple implementation."

Two common ways to push complexity up instead:

- **Throwing errors** for conditions the module could handle itself. See [`errors-and-special-cases.md`](errors-and-special-cases.md).
- **Configuration parameters** for decisions the module hasn't made. A parameter is a decision exported to every caller, many of whom know less than the module's author about the right value.

Before exporting one, ask "will users (or higher-level modules) be able to determine a better value than we can determine here?" The book's example: a network protocol can measure response times and compute a sensible retry interval, which beats making every user guess one.

```typescript
await withRetry(() => provider.charge(input), {
  retries: 3,
  backoffMs: 200,
  retryOn: isTransientProviderError,
});
```

Repeated at every call site, each caller re-decides retry policy and must know which provider errors are transient. Pulled down, `provider.charge(input)` retries transient failures itself, and callers see only the failures they can act on.

**Taking it too far.** Pull complexity down when all three are true:

1. It's closely related to what the module already does.
2. It simplifies code elsewhere in the system.
3. It simplifies the module's interface.

Moving every UI behavior into the text class would satisfy none of them.

---

## Together or Apart

Subdividing has costs:

- **More components,** each with an interface to learn.
- **More code to manage the components.**
- **More separation:** related code ends up in different places, and readers miss dependencies they can't see.
- **Duplication,** when the pieces each need the same logic.

Signs that two pieces belong together:

- **They share information,** such as both understanding the same format.
- **They are used together,** in both directions. A disk cache uses a hash table, but hash tables are used everywhere, so they stay separate.
- **They overlap conceptually** under one simple higher-level category, as substring search and case conversion are both string manipulation.
- **One can't be understood without the other.**

And specific reasons to combine: it **simplifies the interface** (buffering inside an I/O module, so callers don't coordinate it) or it **removes duplication.** The main reason to separate: **general-purpose and special-purpose code.**

The book's two examples cut in opposite directions:

- **Insertion cursor and selection: better apart.** One team combined them into one object holding two positions plus flags for which end held the cursor and whether a selection existed. Higher-level code still treated them as two things, and the object had to derive the cursor from the selection bounds. Separating them was simpler. The redesign needed no cursor or selection classes at all, just a general `Position` class: a cursor is one `Position`, a selection is two, and `Position` proved useful elsewhere.
- **A separate class for logging: better together.** Students moved error logging into a `NetworkErrorLogger` class with methods like `logRpcOpenError` and `logRpcSendError`. Each was mostly one line, called from one place, and so dependent on its caller that readers had to flip between the two. The fix was to delete the class and log where each error is detected.

Relatedness is a judgment call each time. The test is whether combining makes the code that uses it simpler.

### Splitting and Joining Methods

"Length by itself is rarely a good reason for splitting up a method." Each method should do one thing and do it completely.

Split a method when:

- **A subtask is cleanly separable:** someone can read the child without the parent, and the parent without the child's implementation. The child is often general-purpose.
- **The method's interface is doing two jobs.** Then split it into two methods callers choose between, but only if most callers need just one of them. If every caller must call both, the split made things worse.

Join methods when they are **conjoined:** reading one requires reading the other, as when one sets up state the other silently depends on.

```typescript
class CheckoutFlow {
  private pending?: PendingCharge;

  prepare(cart: Cart) {
    this.pending = buildPendingCharge(cart);
  }

  async submit() {
    if (!this.pending) throw new Error("prepare() must be called first");
    return this.chargePending();
  }
}
```

Neither method makes sense alone, and callers must know the order. `submit(cart)` that does both is one thing done completely.

"Depth is more important than length": make functions deep first, then short enough to read easily, and don't trade depth for length. For the full debate with *Clean Code*, see [`contrasts.md`](contrasts.md).
