# Design and Review Checklist

Use this for a full design pass on a new module or a review of a diff. Unlike the questions in `SKILL.md`, these have yes-or-no answers. If you can't tick a box, write down why. Skip sections that don't apply; scale the effort to how long the code will live and how many people will use it.

---

## 1. Before Writing

- [ ] I can say in one sentence what this module is and what it hides.
- [ ] The name came easily. If it didn't, I treated that as a design problem.
- [ ] I sketched at least two radically different interfaces, with caller code for the common case, and chose deliberately.
- [ ] I looked for an existing module that already owns this decision before creating a new one.

## 2. Interface and Depth

- [ ] The interface is much simpler than the implementation.
- [ ] The common case needs the fewest arguments and the least knowledge; rare features don't burden it.
- [ ] Every parameter is one most callers need. The rest have defaults or live elsewhere.
- [ ] There are no hidden ordering requirements, such as "call `init` first," or they're unavoidable and documented.
- [ ] Side effects are either obvious from the name or documented.
- [ ] No setters or getters expose internal representation; callers get operations.
- [ ] No new pass-through methods, decorators, or wrappers that hide nothing.

## 3. Information Hiding

- [ ] Each design decision (format, key, protocol, policy, rule) lives in exactly one module.
- [ ] No two files build the same string format, parse the same structure, or keep the same mapping.
- [ ] No raw database rows or third-party SDK types cross the interface unless that's the module's job.
- [ ] Module boundaries follow knowledge, not the order operations run in.

## 4. Generality and Special Cases

- [ ] The interface is somewhat general-purpose: it would serve the obvious next use without being built for imagined ones.
- [ ] No shared module gained a parameter or branch for a single caller.
- [ ] Special-purpose code sits above (in the caller) or below (in an adapter), not inside the general mechanism.
- [ ] Edge cases are handled by the normal path where possible (empty collections, no-op defaults) rather than by `if` checks spread around.

## 5. Errors

- [ ] Each error the module can raise is one a caller can act on.
- [ ] Errors that could be defined away ("ensure X" instead of "do X") have been.
- [ ] Transient failures are masked inside the module, with a log or metric so operators can still see them.
- [ ] Errors that propagate are handled in one place, not at every call site.
- [ ] Nothing swallows a failure callers need to know about: no empty `catch`, no "failed" turned into "empty," no "unknown" turned into zero.
- [ ] Expected business outcomes are in the return type; rare unrecoverable failures throw.

## 6. Structure

- [ ] Code that shares information, is always used together, or can't be understood apart is together.
- [ ] Each function does one thing completely; none was split only for length.
- [ ] No two functions must be read together to understand either one.
- [ ] Adjacent layers provide different abstractions.
- [ ] No variable is threaded through functions that don't use it.

## 7. Names

- [ ] Every name would let a reader seeing it alone guess what it is.
- [ ] Numbers carry their unit in the name or the type.
- [ ] Booleans read as predicates.
- [ ] Each concept uses the codebase's existing word for it, and that word means only that concept.
- [ ] No filler words: `Object`, `Data`, `Info`, `Manager`, or the class name repeated in a field.

## 8. Comments or Their Equivalents

- [ ] What callers need that the signature can't say (units, boundaries, errors, side effects, invariants) is documented in the place the codebase allows: comments, types, tests, or docs.
- [ ] No comment restates the code.
- [ ] Interface documentation describes behavior, not implementation.
- [ ] The reason for anything surprising is recorded where the next reader will find it, not only in a commit message.
- [ ] Documentation next to changed code was updated in the same diff.

## 9. Obviousness and Consistency

- [ ] A reader's first guess about each function's behavior would be right.
- [ ] No tuples, `Record<string, unknown>` bags, positional booleans, or casts where a named type would do.
- [ ] Nothing behaves differently from what its name and shape suggest.
- [ ] The change follows the codebase's existing conventions for structure, naming, and error handling.

## 10. Performance

- [ ] No network or database calls in loops where they could be batched or run concurrently.
- [ ] Any optimization is backed by a measurement, and was reverted if it didn't help and didn't simplify.
- [ ] On hot paths, special cases are checked once, up front.

## 11. The Change Itself

- [ ] The system now looks as if it had been designed with this change in mind.
- [ ] The fix is in the module that owns the problem, not in the caller that noticed it.
- [ ] The design is at least as good as before: no new red flags in the diff.
- [ ] Larger improvements I noticed but didn't make are written down for the human, not silently skipped or silently done.

## 12. When Reviewing Someone Else's Diff

- [ ] Every finding was checked against the callers and the code it claims is duplicated.
- [ ] Every finding was checked against its red flag's false-alarm note and against Taking It Too Far.
- [ ] Every finding names a red flag, a cost, and a fix, and is labeled new, extended, or pre-existing.
- [ ] Pre-existing findings are kept apart from the ranked list.
- [ ] Correctness bugs found along the way are reported separately, not dropped.
