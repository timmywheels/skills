# Where Ousterhout Disagrees with Popular Advice

Chapter 19 of the book tests popular practices against one question: do they reduce complexity? The book also argues directly with Robert Martin's *Clean Code*, and the two authors debated their differences in writing from late 2024 to early 2025 (`github.com/johnousterhout/aposd-vs-clean-code`). This file summarizes both, and how to act when the advice conflicts with a codebase's rules.

## Table of Contents

- [Function Length](#function-length)
- [Comments](#comments)
- [Names](#names)
- [Test-Driven Development](#test-driven-development)
- [Unit Tests](#unit-tests)
- [Agile Development](#agile-development)
- [Inheritance](#inheritance)
- [Design Patterns](#design-patterns)
- [Getters and Setters](#getters-and-setters)
- [When Advice Conflicts with the Codebase](#when-advice-conflicts-with-the-codebase)

---

## Function Length

**Clean Code:** "The first rule of functions is that they should be small." Blocks inside `if` and `while` statements should be one line, nesting one or two levels deep, and each function should do one thing.

**Ousterhout:** shorter functions are generally easier to read, but once a function is a few dozen lines, further shortening does little. Every split adds an interface to learn. Over-splitting produces shallow, *conjoined* functions that have to be read together. "Depth is more important than length": make functions deep first, then short enough to read, and "don't sacrifice depth for length."

**In the debate,** they agreed that modular design is good, that over-decomposition is possible, and that the first edition of *Clean Code* gives little guidance on recognizing it (Martin says the second edition is more balanced). They agreed the `PrimeGenerator` example in *Clean Code* was decomposed too far. Martin's position is that he'd "rather err on the side of decomposition," since a split can always be inlined; Ousterhout's is that the "One Thing" rule lacks guardrails and "is simply wrong in many cases." Martin summarized the remaining gap as different weightings of the same two values.

The rewrites are instructive. Ousterhout rewrote `PrimeGenerator` as one well-commented function; Martin rewrote it as four functions with few comments. Martin's first version split a loop in two and ran 3–4 times slower; merging the loops fixed it, and his final version ended up faster than Ousterhout's. Splitting code can change behavior as well as readability, and both authors ended up preferring fewer functions than the original.

**In practice:** split a function when a piece can be understood alone, not because it's long. When unsure, keep it together; a function that is long but reads top to bottom costs less than five that must be read together.

---

## Comments

**Clean Code:** comments are, at best, a necessary evil and a sign of failure to express yourself in code. Prefer extracting well-named functions.

**Ousterhout:** good design reduces the need for comments, especially inside function bodies, but comments aren't failures. They carry information code can't: rationale, units, contracts, the abstraction itself. He worries that calling comments failures discourages people from writing the ones that matter.

**In the debate,** they agreed that some comments are necessary, particularly for public APIs; that English works well for explaining *why*; and that non-obvious code needs implementation comments. They disagreed on how much to trust comments (Martin reads every comment as potentially misleading) and on how many to write (Ousterhout would write several times more). In the `PrimeGenerator` exchange, a comment of Ousterhout's caught a real bug.

---

## Names

**Clean Code:** long, descriptive, sentence-like names, scaled to their scope, can replace comments.

**Ousterhout:** precise names matter enormously, but very long names are hard to read, carry less information than a good comment, and are re-read at every use. He objects to names like `isLeastRelevantMultipleOfNextLargerPrimeFactor`. He agrees that the farther a name is used from its declaration, the longer it should be.

---

## Test-Driven Development

**The book:** Ousterhout strongly supports unit tests but not TDD. He argues TDD focuses attention on getting specific features working rather than finding the best design, which is tactical programming. The one place he recommends writing the test first is fixing a bug: write a test that fails because of the bug, then fix it. Fixing first risks a test that never actually triggered the bug.

**In the debate,** Ousterhout acknowledged that the book mischaracterized TDD and said he'd correct it. He proposed "bundling" instead: write a few functions or a class, then its tests. Martin described TDD's red-green-refactor cycle and its benefits: less debugging, tests as low-level documentation, less coupling, and fearless refactoring. They agreed that unit tests are indispensable and that both approaches give fearless refactoring. They disagreed on whether TDD encourages good design, whether tests document code well, and whether mocking reduces coupling.

**In practice:**

1. **Design the interface first,** whichever way the tests get written. Design it twice; write the interface comment or its equivalent.
2. **Then follow the codebase's testing convention:** test-first if the team uses TDD, or in bundles if not. Both produce the refactoring safety net.
3. **For a bug fix, write the failing test first.** Both authors agree, and many teams require it.
4. **Don't let a test drive the interface into a shape that's easy to test and hard to use.** If a test is awkward, check whether the interface is the problem before changing the interface to suit the test.

---

## Unit Tests

Ousterhout is a strong advocate. Developers should write unit tests for their own code, and coverage tools help confirm every line runs. Tests are what make refactoring possible: without them, developers avoid structural improvements and complexity accumulates. His example: when Tcl's bytecode engine was rewritten, the existing unit tests caught nearly all the bugs in the new engine.

---

## Agile Development

Incremental development is good. The risk is that agile encourages putting off design to ship working features quickly, which is tactical programming. "The increments of software development should be abstractions, not features." When a feature reveals the need for an abstraction, take the time to design it properly, general enough to serve more than that one feature, rather than growing it a piece at a time.

---

## Inheritance

Object-oriented mechanisms such as private members help with information hiding but don't guarantee it; shallow classes with exposed state are complex in any language.

- **Interface inheritance** is valuable: one interface with many implementations is deep, and knowledge of one implementation transfers to the others.
- **Implementation inheritance** reduces repetition but creates dependencies between the parent and each subclass, and leaks the parent's internals across the hierarchy. Use it cautiously; prefer composition with small helper classes.

---

## Design Patterns

Patterns help because they're well-understood solutions to common problems. The biggest risk is over-application: not every problem fits a pattern, and forcing one in makes code worse. Using a pattern doesn't make a design good.

---

## Getters and Setters

Getters and setters are shallow (usually one line each), so they clutter an interface without adding functionality, and they expose the implementation. They're better than public fields when exposing a field is truly necessary, since behavior can be added later. But it's best to avoid exposing implementation data at all: offer operations (`membership.cancel()`) rather than access to state (`membership.setStatus("canceled")`).

---

## When Advice Conflicts with the Codebase

The skill will sometimes disagree with a team's conventions or with another skill: a lint rule that bans comments, a required layer structure, a mandated TDD workflow, a style guide demanding very short functions.

1. **The codebase's rules win.** Consistency is one of Ousterhout's principles, and a convention followed everywhere beats a better idea followed in one file.
2. **Serve the underlying goal inside the rule.** No comments: put the information in names, types, tests, and docs (see [`comments-and-names.md`](comments-and-names.md#codebases-that-forbid-comments)). A required layer: give it a real decision to own so it isn't a pass-through. TDD: design the interface before the first test.
3. **If the rule is causing real complexity,** say so to the human, with the red flag and the evidence. Don't quietly break the rule.
