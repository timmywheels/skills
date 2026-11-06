# The Fourteen Red Flags

The book's red flags, each with its definition (quoted from the book), what it tends to look like in a TypeScript diff, why it costs, the usual fix, and when it's a false alarm. The signals and false alarms are extensions; the definitions are Ousterhout's.

A red flag is a reason to look, not a verdict. Report one only when you can say which complexity symptom it causes: change amplification, cognitive load, or unknown unknowns.

## Table of Contents

- [Shallow Module](#shallow-module)
- [Information Leakage](#information-leakage)
- [Temporal Decomposition](#temporal-decomposition)
- [Overexposure](#overexposure)
- [Pass-Through Method](#pass-through-method)
- [Repetition](#repetition)
- [Special-General Mixture](#special-general-mixture)
- [Conjoined Methods](#conjoined-methods)
- [Comment Repeats Code](#comment-repeats-code)
- [Implementation Documentation Contaminates Interface](#implementation-documentation-contaminates-interface)
- [Vague Name](#vague-name)
- [Hard to Pick Name](#hard-to-pick-name)
- [Hard to Describe](#hard-to-describe)
- [Nonobvious Code](#nonobvious-code)

---

## Shallow Module

> The interface for a class or method isn't much simpler than its implementation.

**Signals:**

- A function whose body is one call to another function.
- A class whose methods each forward to one collaborator.
- A new file whose one function forwards to another, or does nothing its single caller couldn't do inline.
- A component whose props mirror its internal state, or a hook that returns a setter for every field.
- An interface, class, factory, and DI token for one implementation.

**Cost:** cognitive load. Another name to learn and another file to open, with no decision hidden behind it.

**Fix:** inline it into its caller or its callee, or give it a decision to own so it hides something.

**False alarm when:** it adapts an external interface you can't change; it's one of several implementations of an interface; or it owns a decision, even a small one used once. `isBillable(member)` hides a business rule, and a component that decides which version of a page a visitor gets hides an eligibility rule. That's depth, at any size.

---

## Information Leakage

> A design decision is reflected in multiple modules.

**Signals:**

- The same format string, regular expression, key layout, or magic value in two or more files.
- Two modules that parse or build the same JSON, CSV, or URL shape.
- A frontend and a backend each keeping their own mapping of the same enum.
- A function that returns raw database rows or provider SDK objects to callers.
- A change in one file that requires a matching change in another, with nothing (compiler, test, import) to say so.
- One feature deciding the same thing in two places: which form fields render versus which get validated; whether a visitor sees the new version in a gate and again in a success page.
- One value reaching a module two ways, such as a prop and a global store.
- A module that only works if its caller first writes a global store, context, or module variable. That ordering is a hidden parameter: part of the interface, but invisible in the signature.
- Hand-written types for a shape that generated types already describe (an API error, a response body).

**Cost:** change amplification, and unknown unknowns when the leak is back-door.

**Fix:** merge the modules, or extract one module that owns the decision behind a simple interface. Across a boundary you can't merge, define the shared knowledge once and import or generate it on both sides.

**False alarm when:** both sides already import a single definition, which is the fix. Duplication with code being replaced behind a flag is a lower priority if there's a plan to delete the old copy; without one, treat it as permanent.

---

## Temporal Decomposition

> The code structure is based on the order in which operations are executed, not on information hiding.

**Signals:**

- Files or functions named for steps: `fetchX`, `transformX`, `saveX`, `preprocess`, `postprocess`, `step1`.
- Modules that only work when called in a fixed sequence.
- Each stage re-deriving or re-validating the same structure.
- A format change that touches every stage.

**Cost:** information leakage between the stages, and shallow modules.

**Fix:** one module per piece of knowledge (the file format, the protocol, the policy). Keep the sequence inside a module.

**False alarm when:** the phases really use different information, such as authenticating a request and then applying business rules.

---

## Overexposure

> An API forces callers to be aware of rarely used features in order to use commonly used features.

**Signals:**

- Common calls that pass many options, most of them the same everywhere.
- Required parameters that nearly every caller sets to the same value.
- Options that only work in combination.
- A component whose typical use passes eight props.
- Callers that must handle errors they can do nothing about.

**Cost:** cognitive load on every caller, for features few of them use.

**Fix:** defaults for the common case; separate entry points for rare features; define away or mask errors callers can't act on.

**False alarm when:** most callers really do need the option. Then it's interface, not overexposure.

---

## Pass-Through Method

> A method does almost nothing except pass its arguments to another method with a similar signature.

**Signals:**

- `return this.repository.updateMember(id, patch)` inside `updateMember(id, patch)`.
- Adjacent layers (controller, service, repository) with matching method names and arguments.
- A wrapper that exists because "every feature has a service."

**Cost:** an extra interface and an extra dependency, no added functionality.

**Fix:** let callers use the lower layer directly, move real responsibility into one of the layers, or merge them.

**False alarm when:** it's a dispatcher choosing among implementations, one of several implementations of an interface, or an adapter over an interface you can't change.

---

## Repetition

> A nontrivial piece of code is repeated over and over.

**Signals:**

- Near-identical blocks that differ by a value or two.
- The same `try`/`catch`, retry, or logging wrapper at many call sites.
- The same validation in several handlers.
- A copied function with a tweak, which agents produce constantly.

**Cost:** change amplification, and bugs when one copy is fixed and the others aren't.

**Fix:** find the abstraction the copies share and give it one home. Make sure the result is deep: a function with six parameters to cover each copy's variation isn't an abstraction. Repeated error handling usually means aggregate it (see [`errors-and-special-cases.md`](errors-and-special-cases.md#3-aggregate-errors)).

**False alarm when:** the similarity is coincidental and the copies will change for different reasons; or a few repeated lines in tests keep each test readable on its own.

---

## Special-General Mixture

> Special-purpose code is not cleanly separated from general purpose code.

**Signals:**

- A shared utility gaining a parameter for one caller: `formatPrice(amount, { forInvoicePdf: true })`.
- `if (source === "kiosk")` inside a general mechanism.
- A shared component with a prop only one screen uses.
- A general module importing from a feature module.

**Cost:** the general module becomes harder to understand and to change, and every special case leaks into it.

**Fix:** push the specialization up into the caller, or down into an adapter that implements a general interface.

**False alarm when:** the "special" case is really part of the general concept, such as a layout or display variant several callers could use. The test: does the parameter describe a variant of the general thing (`centered`, `compact`), or one caller's situation (`forInvoicePdf`, `fromKiosk`)? Check the callers before reporting.

---

## Conjoined Methods

> Two methods have so many dependencies that it's hard to understand the implementation of one without understanding the implementation of the other.

**Signals:**

- A method that only works after another has been called (`prepare()` then `submit()`).
- Helpers that communicate through shared mutable fields.
- Half-built state passed between helpers.
- Reading the code means jumping back and forth between two functions.
- Helper names like `handlePart2` or `finishProcessing`.

**Cost:** cognitive load; the split looks like modularity but behaves like one function scattered across the file.

**Fix:** merge them, or redraw the boundary so each piece can be read alone.

**False alarm when:** the dependency is only through a clear interface. A parent that calls a well-named child without knowing how the child works is fine.

---

## Comment Repeats Code

> All of the information in a comment is immediately obvious from the code next to the comment.

**Signals:**

- The comment uses the same words as the code: `// get the member` above `getMember()`.
- Doc comments that restate parameter names and types.
- Comments narrating each line.

**Cost:** noise that trains readers to skip comments, including the useful ones.

**Fix:** delete it, or replace it with what isn't obvious: units, boundaries, why.

**False alarm when:** never, but check whether the codebase allows comments at all before writing the replacement. See [`comments-and-names.md`](comments-and-names.md#codebases-that-forbid-comments).

---

## Implementation Documentation Contaminates Interface

> An interface comment describes implementation details not needed by users of the thing being documented.

**Signals:**

- Interface documentation that names private fields, internal helpers, SDK calls, or algorithm steps.
- A README for a module that describes its internal file layout instead of how to use it.
- Docs that must change whenever the implementation is refactored.

**Cost:** cognitive load for callers, and documentation that goes stale with every refactor. Often a sign the module is shallow.

**Fix:** describe observable behavior, inputs, outputs, side effects, and errors. Move the details into implementation comments.

**False alarm when:** the detail affects callers, such as a performance characteristic they must plan around.

---

## Vague Name

> The name of a variable or method is so imprecise that it doesn't convey much useful information.

**Signals:** `data`, `info`, `item`, `obj`, `value`, `temp`, `flag`, `status` for a boolean, `count` of something unstated, `handle`, `process`, `manager`, `helper`, `util`, `result` for something that isn't the return value, a number with no unit.

**Cost:** obscurity; readers must trace the code to learn what the name means, and may guess wrong (the Sprite `block` bug).

**Fix:** name what it is specifically, with the unit when the type doesn't carry it.

**False alarm when:** the scope is a few lines and the meaning is visible (`i` in a short loop, `result` for the value being returned).

---

## Hard to Pick Name

> It is difficult to come up with a precise and intuitive name for an entity.

**Signals:**

- Names joined by "And" or "Or": `validateAndSave`, `loadOrCreate`.
- `Manager`, `Handler`, `Processor`, `Helper` because nothing more specific fits.
- Long names that list responsibilities.
- Visible struggle over naming in a plan or review.

**Cost:** a hint that the entity has no clean definition: it does two things, half a thing, or something nobody has defined.

**Fix:** treat it as a design problem. Split it if it does two things, merge it if it does half of one, or define its purpose before naming it.

**False alarm when:** the concept is genuinely new to the codebase and the team needs to coin a term. Coin it once and use it everywhere.

---

## Hard to Describe

> In order to be complete, the documentation for a variable or method must be long.

**Signals:**

- Interface documentation full of "unless," "except when," and "must be called after."
- A long list of parameter interactions.
- A PR description that needs a paragraph to explain how to call the new function.

**Cost:** a complex interface. The description is long because callers really do need to know all of it.

**Fix:** simplify the interface: define errors away, add defaults, split modes into separate functions, pull complexity down.

**False alarm when:** the module is deep and the description is long because it does a lot. That's rare; most long descriptions are describing interface complexity.

---

## Nonobvious Code

> The behavior or meaning of a piece of code cannot be understood easily.

**Signals:**

- Tuples and generic containers: `[boolean, string]`, `Record<string, unknown>`.
- Positional booleans at call sites: `send(member, true, false)`.
- `any`, `as` casts, `as unknown as T`.
- Side effects in getters or in functions named like queries.
- Fire-and-forget promises; handlers whose invocation is invisible.
- Magic numbers; dense one-liners; nested ternaries.

**Cost:** readers guess wrong, which causes bugs and slows every change.

**Fix:** named types with named fields, options objects instead of positional flags, precise names, removing the surprise or documenting it where it's defined and where it's used.

**False alarm when:** the reader is the problem, not the code: an idiom that's standard in this codebase is obvious to its readers, even if it's unfamiliar to you.
