---
name: philosophy-of-software-design
description: Design, change, and review code the way John Ousterhout teaches in "A Philosophy of Software Design" — treat complexity as the enemy, build deep modules with simple interfaces, hide information, make interfaces somewhat general-purpose, pull complexity downward, define errors out of existence, design it twice, and program strategically instead of tactically. Use when designing or refactoring a module, class, service, hook, component, function, or internal API; deciding whether to split or merge code; choosing how code reports errors; naming things; writing or judging comments; planning a change that touches many files; or reviewing a diff for design quality. Triggers on "deep module", "shallow module", "information leakage", "pass-through", "tactical vs strategic", "design it twice", "define errors out of existence", "Ousterhout", "APOSD", "red flags", or "how should I structure this".
metadata:
  author: timmywheels
  version: "1.0"
---

# A Philosophy of Software Design

John Ousterhout's *A Philosophy of Software Design* (2nd edition, 2021) starts from one observation: "The greatest limitation in writing software is our ability to understand the systems we are creating." The job of design is to keep complexity down so that understanding stays possible. This skill turns the book's sixteen principles and fourteen red flags into working rules for designing code, changing it, and reviewing it.

The ideas apply to anything with an interface and an implementation: a function, a class, a React component or hook, a file, a package, an HTTP endpoint, a service. Where this skill says **module**, read any of those.

Sections marked **Beyond the book** are extensions, mostly about how coding agents go wrong. Everything else is Ousterhout's, with TypeScript translations.

---

## The Core Philosophy

> **Complexity is anything related to the structure of a software system that makes it hard to understand and modify the system.**

Every rule below follows from that definition:

1. **The reader decides.** "If you write a piece of code and it seems simple to you, but other people think it is complex, then it is complex."
2. **Complexity is weighted by attention.** Ousterhout's formula, C = Σ c<sub>p</sub>·t<sub>p</sub>, weights each part's complexity by the time developers spend in it. A mess hidden in a module nobody opens costs little. A mess in the code everyone touches costs a lot.
3. **Complexity is incremental.** No single shortcut makes a system complex. Hundreds of reasonable-looking ones do. You have to sweat the small stuff.
4. **There are two ways to fight it.** Eliminate it by making code simpler and more obvious, or encapsulate it in modules so most developers never have to see it.

### Symptoms

| Symptom | What it looks like |
|---|---|
| **Change amplification** | A simple change needs edits in many places |
| **Cognitive load** | A developer must know a lot to finish a task: more APIs, more state, more special cases, more "remember to also…" |
| **Unknown unknowns** | It isn't obvious which code must change, or what you need to know, to change it safely. The worst of the three, because you can't fix what you can't see |

Fewer lines is not the same as less complexity. A longer approach that needs less knowledge to use is simpler.

### Causes

- **Dependencies:** code that can't be understood or changed in isolation. They can't be eliminated; make them few, simple, and obvious.
- **Obscurity:** important information isn't obvious. A generic name, an undocumented unit, an inconsistency, a dependency nothing points to.

---

## Strategic, Not Tactical

**Tactical programming** aims to get the next feature or fix working as fast as possible. Each shortcut looks harmless; together they are how systems become complicated. The extreme case is the *tactical tornado*: prolific, praised for speed, and leaving wreckage for everyone else to clean up.

**Strategic programming** aims for "a great design, which also happens to work." Working code isn't enough. Strategic programmers make continual small investments: a better name, a deeper interface, a special case removed, a comment written while the knowledge is fresh. Ousterhout suggests spending about 10–20% of development time on these and estimates they pay for themselves within 6 to 18 months, after which development is faster than it would have been.

### Beyond the Book: Agents Are Tactical by Default

A coding agent is rewarded for the task passing now, sees only part of the system, and won't maintain what it writes. Those are the conditions that produce tactical programming. The common agent moves, the red flag each raises, and the strategic alternative:

| Tactical move | Red flag | Strategic move |
|---|---|---|
| Add a boolean or option parameter so an existing function handles one caller's case | Special-General Mixture, Overexposure | Push the special case up to the caller, or make the general mechanism cover it |
| Add a service, wrapper, or hook that forwards to something with the same signature | Pass-Through Method, Shallow Module | Call the lower layer directly, or give the new layer a decision to own |
| Split a function into single-use helpers to keep each one short | Conjoined Methods, Shallow Module | Keep it whole until a piece can be understood without the others |
| Throw for each edge case and let callers deal with it | Overexposure | Define the error away, mask it, or aggregate it |
| Add a config option instead of deciding | Complexity pushed upward | Pick the value or compute it; expose a knob only if callers know better |
| Fix the bug at the one call site that hit it | Information Leakage, Repetition | Fix the module so no caller can hit it |
| Copy a similar block and tweak it | Repetition | Find the abstraction both copies share |
| One file per pipeline step: fetch, transform, save | Temporal Decomposition | One module per piece of knowledge: the format, the protocol, the policy |
| Return tuples, `Record<string, unknown>`, or `{ data, error, meta }` grab bags | Nonobvious Code | Name the type and its fields |
| A getter and a setter for every field | Shallow Module, Information Leakage | Expose operations, not representation |
| A comment that restates the line below it | Comment Repeats Code | Delete it; document only what the code can't say |
| A new word for a concept the codebase already names | Vague Name, inconsistency | Use the existing name |

These are tendencies to check, not verdicts. A new parameter is fine when it's a real variant several callers could use, such as a layout option. A small new file is fine when it owns a decision, such as an eligibility gate. Each red flag's false alarms are listed in [`references/red-flags.md`](references/red-flags.md).

The target after every change: "the system will have the structure it would have had if you had designed it from the start with that change in mind."

Strategic doesn't mean unbounded. The user's request sets the scope. Improve the design inside the code you are already changing; when the right fix is bigger than the task, say so and propose it instead of doing it unasked.

---

## The Sixteen Principles

The book's own summary, with what each means in practice.

| # | Principle | In practice |
|---|---|---|
| 1 | Complexity is incremental: you have to sweat the small stuff | No "just this once" shortcuts; small messes compound |
| 2 | Working code isn't enough | Passing tests is the floor, not the finish line |
| 3 | Make continual small investments to improve system design | Leave every module you touch a little better |
| 4 | Modules should be deep | Lots of functionality behind a small interface |
| 5 | Interfaces should be designed to make the most common usage as simple as possible | Defaults for the common case; rare features don't tax common callers |
| 6 | It's more important for a module to have a simple interface than a simple implementation | The implementer absorbs the pain, not every caller |
| 7 | General-purpose modules are deeper | Design the interface for a family of uses; implement today's |
| 8 | Separate general-purpose and special-purpose code | Mechanisms below; policies and special cases above or in adapters |
| 9 | Different layers should have different abstractions | Adjacent layers with the same signature are a smell |
| 10 | Pull complexity downward | Handle it inside the module instead of exporting it to callers |
| 11 | Define errors out of existence | Shape the semantics so the error case can't occur |
| 12 | Design it twice | Compare at least two radically different designs before committing |
| 13 | Comments should describe things that are not obvious from the code | Why, units, invariants, contracts; never a restatement |
| 14 | Software should be designed for ease of reading, not ease of writing | Optimize for the next reader's first guess |
| 15 | The increments of software development should be abstractions, not features | When a feature needs an abstraction, design the abstraction properly |
| 16 | Separate what matters from what doesn't matter and emphasize the things that matter | Make the important visible and the unimportant invisible |

---

## The Fourteen Red Flags

Definitions are the book's. A red flag doesn't prove a problem, but it should make you look. The full catalog, with signals in a diff, fixes, and false positives, is in [`references/red-flags.md`](references/red-flags.md).

| Red flag | Definition | Usual fix |
|---|---|---|
| **Shallow Module** | The interface for a class or method isn't much simpler than its implementation | Merge it into its caller or callee, or give it more to hide |
| **Information Leakage** | A design decision is reflected in multiple modules | Put the decision in one module |
| **Temporal Decomposition** | The code structure is based on the order in which operations are executed, not on information hiding | Organize around knowledge, not sequence |
| **Overexposure** | An API forces callers to be aware of rarely used features in order to use commonly used features | Defaults; separate entry points for rare features |
| **Pass-Through Method** | A method does almost nothing except pass its arguments to another method with a similar signature | Remove the layer, or redistribute the work |
| **Repetition** | A nontrivial piece of code is repeated over and over | Extract the abstraction it repeats |
| **Special-General Mixture** | Special-purpose code is not cleanly separated from general-purpose code | Push the specialization up or down |
| **Conjoined Methods** | Two methods have so many dependencies that it's hard to understand the implementation of one without understanding the implementation of the other | Merge them, or redraw the boundary |
| **Comment Repeats Code** | All of the information in a comment is immediately obvious from the code next to the comment | Delete it, or replace it with what isn't obvious |
| **Implementation Documentation Contaminates Interface** | An interface comment describes implementation details not needed by users of the thing being documented | Describe the abstraction; move the details inside |
| **Vague Name** | The name of a variable or method is so imprecise that it doesn't convey much useful information | Name what it is, specifically |
| **Hard to Pick Name** | It is difficult to come up with a precise and intuitive name for an entity | Treat it as a design problem: the entity may do too much, or too little |
| **Hard to Describe** | In order to be complete, the documentation for a variable or method must be long | Simplify the interface until the description is short |
| **Nonobvious Code** | The behavior or meaning of a piece of code cannot be understood easily | Precise names, specific types, fewer surprises |

---

## 1. Modules Should Be Deep

"The most important issue in designing classes and other modules is to make them deep."

- **Interface is cost; implementation is benefit.** The interface is everything a caller must know: the signature (the formal part) plus the informal parts, such as side effects, ordering constraints ("call `init` first"), error behavior, performance characteristics, and which arguments interact. Count the informal parts; they are usually the expensive ones.
- **A deep module hides a lot behind a little.** Unix file I/O is five calls (`open`, `read`, `write`, `lseek`, `close`) over disk layout, caching, permissions, scheduling, and concurrent access. A garbage collector has almost no interface at all.
- **A shallow module costs about as much to learn as it would cost to read.** A one-line method that forwards to another, or a class that wraps a library without hiding any decision, adds an interface and removes nothing.
- **Classitis** is the belief that more, smaller classes (or functions, or files) are automatically better. Each one adds an interface. Small is not the goal; deep is.
- **Make the common case simple.** Java makes you write `new ObjectInputStream(new BufferedInputStream(new FileInputStream(name)))` to get the buffering almost everyone wants. Buffering should be the default; the rare caller who doesn't want it can opt out.

```typescript
class MembershipService {
  constructor(private readonly repo: MembershipRepository) {}

  findById(id: string) {
    return this.repo.findById(id);
  }

  setStatus(id: string, status: MembershipStatus) {
    return this.repo.setStatus(id, status);
  }
}
```

Callers learn two classes and get nothing for it, and `setStatus` lets any caller put a membership in any state. A deep version owns the decisions:

```typescript
interface Memberships {
  cancel(membershipId: string, options?: { atPeriodEnd?: boolean }): Promise<Membership>;
}
```

Behind `cancel`: which transitions are legal, proration, the billing provider call, the audit record, idempotency, and the notification. The caller knows an ID and one optional choice.

More in [`references/deep-modules.md`](references/deep-modules.md).

---

## 2. Hide Information, and Watch for Leaks

- **Each module owns a few design decisions:** a data format, an algorithm, a protocol, a policy, the details of a lower-level mechanism. Nothing outside depends on them, so they can change without touching callers.
- **Information leakage** is one decision showing up in more than one module. Sometimes it's visible in an interface. The worse kind is *back-door* leakage, where two modules share knowledge no interface mentions: both build the same cache key, both know the JSON shape of a column, both assume a list is sorted.
- **Temporal decomposition is the usual cause.** Modules that mirror the order of execution (read, then parse, then validate, then save) each end up knowing the same format. "When designing modules, focus on the knowledge that's needed to perform each task, not the order in which tasks occur." Ousterhout's students split an HTTP server into a class that read requests and a class that parsed them, and both had to understand HTTP.
- **`private` is not information hiding.** A private field with a public getter and setter is known to every caller.

```typescript
await redis.del(`profile:${gymId}:${memberId}`);
```

```typescript
const cached = await redis.get(`profile:${gymId}:${memberId}`);
```

Two files, one key format, and nothing tells you about the second when you change the first. Give the cache an owner: `profileCache.get(gymId, memberId)` and `profileCache.invalidate(gymId, memberId)`.

**Taking it too far:** don't hide what callers need. If most callers must control something, it belongs in the interface.

---

## 3. Make Modules Somewhat General-Purpose

Over-specialization is one of the largest sources of complexity. The sweet spot is *somewhat* general-purpose: the functionality reflects today's needs, but the interface is general enough to serve several.

Ousterhout's text-editor example: students gave the text class methods named after UI actions, such as `backspace(cursor)`, `delete(cursor)`, and `deleteSelection(selection)`. Each was used once, and every UI change meant changing the text class. The general version, `insert(position, text)` and `delete(start, end)`, is smaller, leaks no UI concepts, and implements backspace in one line of UI code.

Ask:

1. **What is the simplest interface that will cover all my current needs?** Fewer, more general methods, as long as each stays simple. Adding parameters to merge methods doesn't count.
2. **In how many situations will this method be used?** One is a warning sign.
3. **Is this API easy to use for my current needs?** If callers must write loops or glue to use it, it's too low-level.

**Push specialization up, or down.** Special-purpose code belongs at the top (the feature, the UI, the route handler) or at the bottom (an adapter or driver implementing a general interface), not mixed into the general mechanism. Ousterhout's undo system separates into a general `History` that records and groups actions, the specific actions that each know how to undo themselves, and the grouping policy, which lives in UI code.

**Eliminate special cases.** Design the normal path so edge cases fall out of it. Represent "no selection" as an empty selection whose start equals its end; then copy and delete work with no `if (selection)` checks. In TypeScript: an empty array instead of `undefined`, a no-op implementation instead of an optional dependency.

---

## 4. Different Layer, Different Abstraction

Each layer should provide a different abstraction from the layers above and below it. Adjacent layers with similar abstractions are "a red flag that suggests a problem with the class decomposition."

- **Pass-through methods** forward their arguments to a method with the same signature. Fix by letting callers use the lower layer directly, redistributing functionality so each layer does real work, or merging the layers.
- **Duplicate signatures are fine when the method does something:** a dispatcher that picks an implementation based on its arguments, or several implementations of one interface (drivers, adapters, strategies).
- **Decorators are usually shallow.** Before wrapping, consider adding the feature to the underlying class, merging it into the one place that needs it, or merging it with an existing wrapper.
- **The interface should differ from the representation.** A class that stores text as lines shouldn't make callers think in lines. A table with a `status` column shouldn't mean every caller writes `status`.
- **Pass-through variables** are threaded through many signatures by functions that never use them: a logger, a request ID, a tenant ID, feature flags. Ousterhout's fix is a context object. In TypeScript that's a request-scoped context such as `AsyncLocalStorage`, a framework request object, or React context. It isn't free: it hides a dependency, so keep it small, immutable where possible, and not a dumping ground.

---

## 5. Pull Complexity Downward

"It is more important for a module to have a simple interface than a simple implementation." Most modules have more users than developers, so it's better for the developer to suffer than the users.

- **Exceptions and configuration parameters are the classic ways to push complexity up.** Throwing an error you don't know how to handle, or exposing a setting you haven't decided, makes every caller deal with it.
- **Before exporting a configuration parameter, ask:** "will users (or higher-level modules) be able to determine a better value than we can determine here?" Usually they can't. Pick a sensible value or compute one (a retry interval from observed latency, a batch size from the payload). If the knob must exist, give it a default.
- **Taking it too far:** pull complexity down only when it's closely related to the module's existing job, when it simplifies callers, and when it simplifies the interface. Pulling one caller's policy into a shared module is leakage in the other direction.

---

## 6. Together or Apart?

Splitting has costs: every new piece is another interface, more code to manage the pieces, more distance between related code, and sometimes duplication. Bring code together when the pieces:

- **share information**, such as the same format,
- **are used together**, in both directions,
- **overlap conceptually** under a simple higher-level category,
- **can't be understood apart**,
- or when merging **simplifies the interface** or **removes duplication**.

Keep **general-purpose and special-purpose code** apart.

For methods: "Length by itself is rarely a good reason for splitting up a method." Each method should do one thing and do it completely. Split when a subtask is cleanly separable, meaning someone can read the child without the parent and the parent without the child's implementation. Also split when one method's interface was secretly doing two jobs; then make two methods that callers choose between. Join methods that can't be read apart. "Depth is more important than length."

The test for any structure: "Pick the structure that results in the best information hiding, the fewest dependencies, and the deepest interfaces."

---

## 7. Define Errors Out of Existence

"Exception handling is one of the worst sources of complexity in software systems." Every error a module can raise is part of its interface, and every place that must handle one is special-case code that rarely runs and is rarely tested. Reduce the number of places errors must be handled, in this order:

1. **Define the error out of existence.** Change the semantics so the case is normal. Tcl's `unset` threw if the variable didn't exist; it should have meant "make sure this variable doesn't exist," which is already true. Unix lets you delete an open file, and the data goes away when the last handle closes; Windows refuses and makes the user hunt for the process holding it. Java's `substring` throws on out-of-range indexes; JavaScript's `slice` clamps them.
2. **Mask it.** Handle it at a low level so callers never see it. TCP retransmits lost packets; a client library retries a transient failure.
3. **Aggregate it.** Replace many handlers with one. A web server lets every missing-parameter error propagate to one top-level handler that returns an error response. RAMCloud sends rare corruption cases through the same recovery path as a server crash.
4. **Just crash.** For errors that are rare and not worth handling (out of memory, a broken invariant), fail loudly instead of writing recovery code nobody will test.

```typescript
function removeTag(tags: readonly string[], tag: string): string[] {
  if (!tags.includes(tag)) throw new TagNotFoundError(tag);
  return tags.filter((t) => t !== tag);
}
```

Every caller now needs a `try`. Defined as "ensure `tag` is absent," the error disappears and the body is one line: `return tags.filter((t) => t !== tag)`.

**Taking it too far:** this is not swallowing errors. If callers need to know (a declined payment, a write that didn't happen, a failure the user must fix), the error is part of the interface and must be exposed clearly. Returning `null` from a `catch` is acceptable only when `null` has a meaning callers can act on. Never discard data or hide a failure to make an interface look simpler.

More in [`references/errors-and-special-cases.md`](references/errors-and-special-cases.md).

---

## 8. Design It Twice

Your first idea is unlikely to be the best. Before committing to the design of anything that matters (a module, an interface, a schema, a tricky algorithm), sketch at least two **radically different** approaches, not variations of one. Compare them: Which has the simpler interface? Which is more general-purpose? Which allows an efficient implementation? What does the caller's code look like in each? The winner often combines features of several.

It costs an hour or two for a class, less for a function, against years of living with a mediocre interface. Ousterhout notes it is hardest for smart people to accept, because they are used to their first idea being good enough.

For agents: put the alternatives in the plan, each with a few lines of *caller* code, before writing the implementation. When two are close, let the human choose.

---

## 9. Comments, Names, and Obvious Code

**Comments** describe things that aren't obvious from the code. They work in two directions:

- **Lower-level comments add precision:** units, inclusive or exclusive bounds, what `null` means, who owns a resource, invariants.
- **Higher-level comments add intuition:** what a block is for and why it is done this way.

Interface comments describe the abstraction: what a caller needs, not how it works. "If users must read the code of a method in order to use it, then there is no abstraction." Implementation comments say what and why, not how. Write comments first, as a design tool: if the interface comment is long or hard to write, the interface is wrong.

**Names** create an image in the reader's mind. Make them precise (`cursorVisible`, not `blinkStatus`; `numActiveIndexlets`, not `count`; `charIndex` and `lineIndex`, not `x` and `y`), consistent (one name per concept, and one concept per name), and free of filler words. A name that is hard to pick is a design smell. Ousterhout once spent months chasing a bug caused by one name, `block`, used for both file block numbers and disk block numbers.

**Obvious code** is code where "someone can read the code quickly, without much thought, and their first guesses about the behavior or meaning of the code will be correct." If a reviewer says it isn't obvious, it isn't. Common culprits: event handlers whose invocation is invisible, generic containers (tuples, `Pair`, `Record<string, unknown>`), a declared type that differs from the real one, and code that violates the reader's expectations.

### Codebases That Forbid Comments

Some codebases ban comments through lint rules or hooks. Follow the house rule; consistency is one of the principles too. The information still matters, so route it somewhere a reader will find it:

| Information | Where it goes instead |
|---|---|
| Units, ranges, what empty or `null` means | The name (`timeoutMs`, `endExclusive`) and the type (branded types, explicit unions) |
| Invariants | Types that make illegal states unrepresentable; a constructor or parser that enforces them |
| The contract callers rely on | The signature, precise names, and tests named for the behavior |
| Why this approach, and what was rejected | The PR description; an ADR or design doc for decisions that outlive the PR |
| Decisions that span modules | A design doc or ADR, findable from the names in the code |

Still write the interface comment first, as a draft in the plan, to test whether the interface is simple. Then move each sentence to its comment-free home. If a sentence has no home, tell the human rather than dropping it.

Full guidance in [`references/comments-and-names.md`](references/comments-and-names.md).

---

## 10. Consistency, What Matters, and Performance

**Consistency** lets knowledge transfer: once you learn how one thing works, you know how the similar things work. Be consistent in names, style, interfaces, design patterns, and invariants. When in Rome, do as the Romans do: "having a 'better idea' is not a sufficient excuse to introduce inconsistencies." Change all of it or none of it. Don't overdo it: forcing dissimilar things into the same shape is its own obscurity.

**Decide what matters.** Structure systems around the few things that matter and hide the rest. Look for leverage, where one idea solves many problems. Minimize what matters with defaults, computed values, and errors handled low. Emphasize what does matter through prominence (names, interface docs, the parameters of heavily used methods), repetition, and centrality. The two mistakes: treating too many things as important, which clutters interfaces, and missing what is important, which creates unknown unknowns. Ousterhout calls telling the two apart good taste.

**Performance.** Simple and fast usually agree, because deep modules do more per call. Know what is expensive: network round trips, disk and database I/O, memory allocation, cache misses, and in TypeScript services, N+1 queries and `await` inside loops. Measure before modifying. For hot code, design around the critical path: imagine the least code the common case needs, then shape the design so that path stays clean and special cases are checked once, up front.

More in [`references/strategic-design.md`](references/strategic-design.md).

---

## Taking It Too Far

Ousterhout warns that every principle can be overapplied. Agents overapply rules readily, so check each fix against its guardrail.

| Principle | Overapplied | Guardrail |
|---|---|---|
| Deep modules | A god object with a small interface over unrelated jobs | Depth hides *related* complexity; unrelated jobs still separate |
| Information hiding | Hiding what callers need to control | If most callers need it, it's interface |
| General-purpose | A framework built for one use | *Somewhat* general: today's functionality, a general-enough interface |
| Pull complexity down | One caller's policy inside a shared module | Only complexity related to the module's own job |
| Define errors out | Swallowing failures callers must react to | Errors that change what a caller should do stay visible |
| Fewer, bigger methods | One method with modes and flags | Each method does one thing completely |
| Comments | Commentary on every line | Only what the code can't say |
| Consistency | Dissimilar things forced into one shape | Similar things alike, different things different |
| Design it twice | Days of options for a throwaway script | Scale the effort to how long the design lives and how many people use it |
| Strategic investment | Rewriting the neighborhood during a bug fix | Improve what you touch; propose the rest |

---

## How to Apply

### Designing Something New

1. **Name the abstraction in one sentence.** If you can't, or the name is hard to pick, rethink the boundary.
2. **Write the interface first:** signatures plus everything a caller must know. Draft it as an interface comment, or in the plan if the codebase bans comments. If the description is long, the interface is too complex.
3. **Design it twice.** At least two radically different interfaces, each with caller code for the common case.
4. **Choose** the one that needs the least caller knowledge for the common case, hides the most decisions, and is general enough for the obvious next use.
5. **Plan the errors.** For each failure: define it away, mask it, aggregate it, or crash. Expose only what callers must act on.
6. **Implement** with special cases off the main path, then reread it as a stranger. Would their first guess be right?

### Changing Existing Code

1. **Find the owner.** Locate the module that owns the decision you're changing, and change it there rather than in the caller that noticed.
2. **Check the fit.** If the change doesn't fit the current design, adjust the design so it does, within the scope of the task. Propose bigger restructuring rather than doing it unasked.
3. **Match the existing conventions** for names, structure, and error handling.
4. **Review your own diff** before finishing: new parameters, new pass-throughs, new special cases, repeated code, vague names.
5. **Update the documentation** that lives next to the code you changed, in the same diff.

### Reviewing Code

1. **Collect candidates** by reading the diff against the red-flag table and [`references/checklist.md`](references/checklist.md).
2. **Verify each candidate against the code around it.** Read the callers, the other implementations, and anything the candidate claims is duplicated. Then check it against that red flag's false-alarm note in [`references/red-flags.md`](references/red-flags.md) and against [Taking It Too Far](#taking-it-too-far). Most candidates should die here.
3. **Keep only what you can name.** A finding names the red flag and the complexity it adds. "Feels messy" is not a finding.
4. **Label where it came from:** *new* (the diff creates it), *extended* (the diff adds to an existing one), or *pre-existing* (the diff only sits near it). Rank new and extended findings. Mention pre-existing ones separately, and only when the diff makes them harder to fix later.
5. **Rank by C = Σ c<sub>p</sub>·t<sub>p</sub>.** Estimate attention from what the repo shows: how many callers and features depend on the code, how often it changes (`git log`), and whether it's about to be deleted. Duplication of code that a flag-gated rewrite is replacing ranks low if there's a removal plan; without one, it's permanent.
6. **Write each finding** with the red flag and its label, every location (`file:line`), the cost (change amplification, cognitive load, or unknown unknowns), a concrete consequence when you can show one, and the fix. If the right fix is bigger than the PR should absorb, give the smallest fix that removes the red flag and name the larger one as a follow-up.
7. **Report bugs separately.** Design reviews find correctness bugs in passing. List them apart from the design findings rather than dropping them or dressing them up as red flags.

```text
Information Leakage (new): billing/invoice.ts:42, billing/receipt.ts:88-95
Both build the line-item label, with different rules: the receipt drops the quantity when it's 1,
so one purchase reads differently on the two documents. Cost: change amplification, and a
mismatch customers will notice.
Fix: one describeLineItem(item) in the line-item module, used by both, with a test named for the
quantity rule.
```

A shallow interface on code every feature touches outranks a messy private helper nobody reads.

---

## Where Ousterhout Disagrees with Popular Advice

| Topic | Popular advice | Ousterhout |
|---|---|---|
| Function size | Functions should be tiny and do one thing (*Clean Code*) | Depth over length; over-splitting creates shallow, conjoined methods |
| Comments | Comments are failures; code should explain itself | Code can't express why, units, or contracts; comments are part of the design |
| TDD | Write the test first, always | Unit tests yes; test-first can crowd out design. Design the interface first. Test-first fits bug fixes well |
| Agile | Ship features in small increments | Increments yes, but they should be abstractions, not features |
| Inheritance | Reuse through class hierarchies | Interface inheritance gives depth; implementation inheritance leaks parent internals into children. Prefer composition |
| Getters and setters | Encapsulate fields behind accessors | They are shallow and expose representation. Expose operations instead |
| Design patterns | Apply known patterns | Good when they fit; the risk is over-application |

Ousterhout and Robert Martin debated the first three in writing in 2024–25 (`github.com/johnousterhout/aposd-vs-clean-code`). Where they landed, and the practical middle ground, is in [`references/contrasts.md`](references/contrasts.md).

---

## Reference Documents

- **[`references/deep-modules.md`](references/deep-modules.md)**: measuring depth, information hiding and leakage, general-purpose interfaces, layers, pulling complexity down, together or apart
- **[`references/errors-and-special-cases.md`](references/errors-and-special-cases.md)**: the four techniques, special cases, choosing between throwing and returning, where it goes wrong
- **[`references/comments-and-names.md`](references/comments-and-names.md)**: what comments are for, writing them first, names, obvious code, codebases that ban comments
- **[`references/strategic-design.md`](references/strategic-design.md)**: complexity in depth, tactical vs. strategic, design it twice, modifying existing code, consistency, deciding what matters, performance
- **[`references/red-flags.md`](references/red-flags.md)**: all fourteen red flags with signals in a diff, fixes, and false positives
- **[`references/contrasts.md`](references/contrasts.md)**: *Clean Code*, TDD, agile, inheritance, patterns, accessors
- **[`references/checklist.md`](references/checklist.md)**: design and review checklist

---

## Summary

1. **Complexity is the enemy.** The reader measures it, and it is weighted by how often code is touched.
2. **Be strategic.** Working code isn't enough; invest a little in the design with every change.
3. **Make modules deep:** lots of functionality behind a small interface, with the common case simplest.
4. **Hide each design decision in one place.** Organize by knowledge, not by execution order.
5. **Make interfaces somewhat general-purpose,** and push special cases up or down.
6. **Give each layer a different abstraction.** Remove pass-throughs.
7. **Pull complexity downward.** The implementer suffers so callers don't.
8. **Define errors out of existence,** then mask, aggregate, or crash, but never hide what callers must know.
9. **Design it twice.**
10. **Write for the reader:** precise names, comments (or their equivalents) for what code can't say, and obvious code.
11. **Be consistent, and decide what matters.**
