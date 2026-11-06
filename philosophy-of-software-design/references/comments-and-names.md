# Comments, Names, and Obvious Code

Detail for section 9 of `SKILL.md`, covering chapters 12–16 and 18 of the book. The TypeScript examples and the section on codebases that forbid comments are new.

## Table of Contents

- [Why Comments](#why-comments)
- [Four Kinds of Comments](#four-kinds-of-comments)
- [Don't Repeat the Code](#dont-repeat-the-code)
- [Lower-Level Comments Add Precision](#lower-level-comments-add-precision)
- [Higher-Level Comments Add Intuition](#higher-level-comments-add-intuition)
- [Interface Comments](#interface-comments)
- [Implementation Comments](#implementation-comments)
- [Decisions That Span Modules](#decisions-that-span-modules)
- [Write the Comments First](#write-the-comments-first)
- [Keeping Comments Accurate](#keeping-comments-accurate)
- [Choosing Names](#choosing-names)
  - [Precise](#precise)
  - [Consistent](#consistent)
  - [No Extra Words](#no-extra-words)
  - [How Long](#how-long)
- [Obvious Code](#obvious-code)
- [Codebases That Forbid Comments](#codebases-that-forbid-comments)

---

## Why Comments

A comment captures what the designer knew but the code can't represent: why a design was chosen, what a caller may rely on, what a value's units are, which hardware quirk forced the odd code below. Abstraction depends on it. A signature can't tell you whether the end index of a range is inclusive.

Ousterhout answers four excuses for not commenting:

1. **"Good code is self-documenting."** Names help, but code can't express the informal parts of an interface, the reasons for decisions, or preconditions. And if callers must read the code to use it, it isn't an abstraction.
2. **"I don't have time."** Coding is a fraction of development time, and comments cost a fraction of coding. Good comments add at most about 10% and pay it back quickly.
3. **"Comments get out of date."** Less often than claimed. Large documentation changes only come with large code changes, and keeping comments next to the code (and checking them in review) keeps them current.
4. **"All the comments I've seen are worthless."** The most valid excuse. The fix is to write better ones.

Comments attack all three symptoms of complexity: they reduce cognitive load by telling readers only what they need, they expose structure so unknown unknowns become known, and they clarify dependencies and ambiguity.

---

## Four Kinds of Comments

Pick a convention (TSDoc, JSDoc, or the project's own) and follow it. The book's four kinds:

1. **Interface comments** on classes, functions, and methods. The most important kind.
2. **Data comments** on fields and important variables.
3. **Implementation comments** inside function bodies. Often unnecessary.
4. **Cross-module comments** about decisions that span modules. Rare, sometimes essential.

Ousterhout's position is that every class, field, and method gets a comment. Many teams disagree; see [When the Codebase Forbids Comments](#codebases-that-forbid-comments).

---

## Don't Repeat the Code

The test: could someone who has never seen this code write the comment just by reading the lines next to it? If so, it adds nothing.

```typescript
// Set the member's status to active
member.status = "active";
```

A common variant reuses the words in the name: "the gym ID" on `gymId`. Use different words that add meaning. If something here isn't obvious, say that instead:

```typescript
// Reactivation clears any scheduled cancellation. Billing resumes at the next
// period boundary, not immediately.
member.status = "active";
```

---

## Lower-Level Comments Add Precision

Most useful on declarations. Say what the code can't:

- **Units:** milliseconds or seconds, cents or dollars, bytes or characters.
- **Boundaries:** inclusive or exclusive.
- **Empty values:** what `null`, `undefined`, or an empty string means.
- **Ownership:** who closes or releases a resource.
- **Invariants:** "always has at least one entry."
- **For maps:** what the keys and values are, and what a missing entry means.

Describe what a variable *represents*, not how the code manipulates it. For a flag, say what true means, not when the code sets it.

```typescript
// The offset
offset: number;
```

```typescript
// Byte offset into the upload of the first byte the server hasn't acknowledged.
offset: number;
```

---

## Higher-Level Comments Add Intuition

These say what a block is trying to do, so each detail inside it makes sense. They're harder to write because you have to find the simplest framing of the code's purpose.

```typescript
// If a batch exists for this gym, isn't sent, and has room, add to it
```

```typescript
// Try to add this message to an unsent batch for the same gym, so a burst of
// notifications goes out as one provider call.
```

The first restates the conditions. The second explains them. Higher-level comments can also say why a path exists or when a function gets called, which matters most for unusual cases.

---

## Interface Comments

Comments are how an abstraction is defined; the code is too detailed to serve as one.

- **A class comment** gives the abstraction at a high level: what an instance represents and its limits, such as not being safe for concurrent use.
- **A function comment** starts with the observable behavior, then each parameter and the return value (including constraints and interactions between parameters), side effects, errors, and preconditions.

```typescript
/**
 * Charges the member's default payment method for every invoice that is due.
 *
 * Declines don't throw: a declined invoice stays open and is scheduled for a
 * retry. Returns one result per invoice attempted, oldest due date first.
 * Safe to call repeatedly; invoices already paid are skipped.
 *
 * @throws NoPaymentMethodError when the member has no default payment method.
 *   Nothing is charged.
 */
```

Two warning signs:

- **Implementation details contaminate the interface.** "Loops over open invoices and creates a PaymentIntent with `off_session: true`" tells callers nothing they need and goes stale with every refactor. If an interface comment must describe the implementation to make sense, the function is probably shallow.
- **Obvious statements.** "Pass the member ID" on a `memberId` parameter.

The book's exercise on documenting an index-lookup class is a useful filter for what belongs: message formats sent to servers (no), the comparison used for range checks (yes, callers depend on it), server-side data structures (no), whether requests run concurrently against several servers (perhaps, as high-level performance information), how crashes are handled (no, recovery is automatic and invisible to callers).

---

## Implementation Comments

What and why, not how. Use them for:

- **Long functions:** a short comment before each major phase.
- **Long or complex loops:** what each iteration accomplishes.
- **Non-obvious code, including bug fixes:** why it's needed. For a fix, reference the ticket rather than copying it.
- **Important variables in long functions:** what they represent.

```typescript
// The webhook can arrive before the checkout request commits, so a missing
// order is expected here. Retry instead of failing.
```

---

## Decisions That Span Modules

Some decisions live in several modules at once, and the hard part is finding a place to write them down.

- **If there's a natural central place, use it.** Adding a value to a status enum requires changes in several files; the enum's declaration is where someone adding a value will look, so the checklist goes there.
- **If there isn't,** the book uses a central `designNotes` file organized by topic, with short pointers from the code ("See 'Zombies' in designNotes"). It keeps one copy that's easy to find, at the cost of distance from the code. In most repositories today, this is an ADR or a design doc.

---

## Write the Comments First

Comments written after the code tend to be skipped or written hastily, and they repeat the code. Ousterhout's procedure:

1. Write the class or module interface comment.
2. Write signatures and interface comments for the most important public functions, with empty bodies. Iterate until the structure feels right.
3. Declare and comment the important fields.
4. Fill in the bodies, adding implementation comments as needed.

New functions and fields get their comments before their bodies, so the documentation is done when the code is.

**Comments are a design tool,** and a canary. Writing the comment forces you to find the essence of the abstraction while it's cheap to change. If a function needs a long comment to describe, its abstraction is weak. If the interface comment is short and complete, the interface is probably simple. Comparing the interface comment with the implementation shows whether the function is deep.

The cost is small: comments are under 10% of development time, and better abstractions found early often save more than that.

---

## Keeping Comments Accurate

From chapter 16:

- **Keep comments next to the code they describe.** Interface comments go on the implementation, not in a distant header or a separate doc.
- **Put implementation comments where they apply,** above each phase, rather than one block at the top. The farther a comment is from the code, the more abstract it should be.
- **Comments belong in the code, not the commit log.** If the subtle reason for a change lives only in a commit message, someone will revert it and reintroduce the bug.
- **Avoid duplication.** Record each decision once, in the most obvious place. Don't document one module's behavior in another module, and link external specs instead of copying them.
- **Check the diff before committing,** for comments the change made wrong, along with leftover debug code and unfinished TODOs.
- **Higher-level comments are easier to maintain,** because small code changes don't invalidate them.

---

## Choosing Names

Names are a form of abstraction: a good name highlights what matters about an entity and hides the rest.

Ousterhout's cautionary bug: in the Sprite operating system, files occasionally had blocks zeroed for no reason. It took six months to find. The variable `block` held physical disk block numbers in some places and logical file block numbers in others, and one call passed the wrong kind. Several people had read the code and missed it, because they assumed `block` always meant the same thing. `fileBlock` and `diskBlock`, or distinct types, would have prevented it. Don't settle for names that are "reasonably close."

**Create an image.** Ask: if someone saw this name alone, without its declaration or documentation, how close would their guess be? A name should say what something is and what it isn't, in two or three words.

### Precise

The book's examples:

| Vague | Precise | Why |
|---|---|---|
| `getCount()` | `numActiveIndexlets` | Count of what? |
| `x`, `y` for a character's position | `charIndex`, `lineIndex` | Easily read as pixels |
| `blinkStatus` | `cursorVisible` | "Status" doesn't say what true means; booleans should be predicates |
| `VOTED_FOR_SENTINEL_VALUE` | `NOT_YET_VOTED` | Say what the special value means |
| `result` in a function that returns nothing | `mergedLine`, `totalChars` | Misleading; `result` is fine for the value a function returns |
| `struct socket` and `struct sock` | `struct sock_base` and `struct inet_sock` | Too similar to tell apart |

A name can also be too specific: `delete(selection: Range)` implies the range came from the UI, so `range` is better. `i` and `j` are fine for short loops whose meaning is visible at a glance.

If a precise name is hard to find, the entity may have an unclear definition or may be several things at once. Treat it as a design problem before a vocabulary problem.

**Beyond the book,** for TypeScript:

- Put the unit in the name when the type doesn't carry it: `timeoutMs`, `amountCents`, `durationSeconds`.
- Booleans read as predicates: `isOverdue`, `hasPaymentMethod`, `cursorVisible`.
- A function named `get…` doesn't write; a function named `validate…` doesn't mutate.

### Consistent

Three rules:

1. Always use the common name for a given purpose.
2. Never use that name for anything else.
3. Keep the purpose narrow enough that every variable with the name behaves the same. The Sprite bug broke this rule.

When several variables refer to the same kind of thing, add a distinguishing prefix: `srcFileBlock`, `dstFileBlock`. Use `i` for outer loops and `j` for nested ones, every time.

In a domain codebase, this means one word per concept. If `member`, `customer`, `client`, and `user` all mean the person who buys a membership, readers can't tell whether they are four things or one.

### No Extra Words

Every word should add information:

- `fileObject` → `file`.
- No Hungarian notation encoding types into names; editors show types.
- Don't repeat the class name in a field: inside `File`, `fileBlock` is just `block`, unless the class holds several kinds of block.

### How Long

Ousterhout agrees with Andrew Gerrand of the Go team that "the greater the distance between a name's declaration and its uses, the longer the name should be." He disagrees that very short names are generally more readable, and objects to reusing short names for different things (`ch` for both characters and channels). The reader decides what's readable.

---

## Obvious Code

Code is obvious when "someone can read the code quickly, without much thought, and their first guesses about the behavior or meaning of the code will be correct." The reader judges, not the author: if a reviewer says it isn't obvious, it isn't. Don't argue; fix the code or the comment.

There are three ways to make code obvious, best first:

1. **Reduce the information needed,** through abstraction and by eliminating special cases.
2. **Use what readers already know,** by following conventions and meeting expectations.
3. **Supply what's missing,** with good names and well-placed comments.

What helps: precise names, consistency, whitespace that separates the phases of a function, and comments where the code is unavoidably hard.

What hurts, with TypeScript equivalents:

| The book's case | TypeScript equivalents | Remedy |
|---|---|---|
| **Event-driven programming:** handlers invoked indirectly, so control flow is invisible | Event emitters, queue and job handlers, webhooks, `useEffect` chains | Document or name when each handler runs; prefer direct calls when there's one listener |
| **Generic containers** like `Pair`, whose `getKey()`/`getValue()` say nothing | Tuples (`[boolean, string]`), `Record<string, unknown>`, `Map<string, [number, number]>` | A named type with named fields |
| **Declared type differs from the real one,** like `List` holding an `ArrayList` | `as` casts that lie, `as unknown as T`, `any` | Let the declared type be the true type |
| **Code that violates expectations,** like a `main` that returns while a thread it started keeps the program alive | Fire-and-forget promises, a `get…` that writes, a constructor that starts work | Remove the surprise, or document it at the definition and the call site |

---

## Codebases That Forbid Comments

Some codebases ban comments, through a lint rule or a hook, usually because comments in that codebase had drifted into restating code or going stale. Follow the house rule. Consistency is one of Ousterhout's principles too, and an agent that fights a team's convention is creating inconsistency.

The information Ousterhout wants in comments still matters. Route each kind somewhere a reader will find it:

| Information | Where it goes instead |
|---|---|
| Units, ranges, what empty or `null` means | The name (`timeoutMs`, `endExclusive`) and the type (branded types, explicit unions instead of `null`) |
| Invariants | Types that make illegal states unrepresentable; a constructor or parser that enforces them |
| What a caller can rely on | The signature, precise names, and tests named for the behavior (`it("skips invoices that are already paid")`) |
| Errors and edge cases | Return types that list the outcomes; tests for each one |
| Why this approach, and what was rejected | The PR description for this change; an ADR or design doc if it outlives the PR |
| Decisions spanning modules | An ADR or design doc, named so a search from the code finds it |
| A non-obvious line, such as a workaround | Extract it into a function whose name says why (`waitForCheckoutCommit`) |

If doc tags are allowed (some no-comment rules permit bare `@param` or `@throws` tags), use them for the parts of the contract a tag can express.

The process still starts with comments:

1. **Draft the interface comment anyway,** in the plan or a scratch file, as a design tool. If it's long or hard to write, fix the design.
2. **Move each sentence** to its comment-free home from the table above.
3. **List any sentence with no home** in the PR description or your summary to the human. Losing it silently creates the unknown unknowns that comments were meant to prevent.
