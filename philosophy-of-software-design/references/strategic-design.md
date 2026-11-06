# Strategic Design

Detail for the process side of `SKILL.md`: complexity (chapter 2), strategic programming (chapter 3), design it twice (chapter 11), modifying existing code (chapter 16), consistency (chapter 17), performance (chapter 20), and deciding what matters (chapter 21).

## Table of Contents

- [Complexity in Depth](#complexity-in-depth)
  - [Symptoms](#symptoms)
  - [Causes](#causes)
- [Tactical Versus Strategic](#tactical-versus-strategic)
- [Design It Twice](#design-it-twice)
  - [A Template for Plans](#a-template-for-plans)
- [Modifying Existing Code](#modifying-existing-code)
- [Consistency](#consistency)
- [Decide What Matters](#decide-what-matters)
- [Designing for Performance](#designing-for-performance)

---

## Complexity in Depth

> Complexity is anything related to the structure of a software system that makes it hard to understand and modify the system.

- **It's about structure, not size.** A large system with clean structure can be simple to work in; a small one can be miserable.
- **It's weighted by attention.** C = Σ c<sub>p</sub>·t<sub>p</sub>: the complexity of each part times the fraction of time developers spend in it. Isolating complexity where nobody has to look is nearly as good as eliminating it.
- **The reader decides.** Complexity is more apparent to readers than writers. If others find your code complex, it is.
- **It's easier to recognize than to avoid.** "It is easier to tell whether a design is simple than it is to create a simple design." That's why review works and why design it twice works.
- **It's incremental.** It doesn't come from one catastrophic mistake but accumulates in small pieces, each of which seems too minor to matter. Once it has accumulated, it's very hard to remove, so the only defense is a zero-tolerance attitude toward the small pieces.

### Symptoms

- **Change amplification.** The book's example: a website where every page specifies the banner's background color. Changing the color means editing every page. A shared variable fixes it.
- **Cognitive load.** How much a developer must know to complete a task. A C API that allocates memory and requires the caller to free it adds cognitive load to every caller; forgetting causes leaks. Fewer lines can mean more cognitive load, if the shorter version requires more knowledge.
- **Unknown unknowns.** Extending the website example: some pages use a darker shade derived from the banner color, hardcoded. The developer who changes the shared variable has no way to know those pages need updating. The worst symptom, because nothing tells you it's there.

### Causes

- **Dependencies.** A dependency exists when code can't be understood or modified in isolation. Some are unavoidable (every caller depends on a function's signature); the goal is fewer, simpler, and more obvious ones.
- **Obscurity.** Important information isn't obvious: a variable named `time` with no units, one name used for two things, a dependency nobody documented. Obscurity often signals a design problem: a clean, obvious design needs less documentation.

Dependencies cause change amplification and cognitive load; obscurity causes unknown unknowns and cognitive load.

---

## Tactical Versus Strategic

**Tactical programming** focuses on getting the next feature or fix working as quickly as possible. Each shortcut adds a little complexity; patches on patches add more. The approach is hard to reverse once a team commits to it. Its extreme is the *tactical tornado*, an engineer who ships faster than anyone, is often praised by management, and leaves messes for colleagues whose careful work then looks slow by comparison.

**Strategic programming** makes the primary goal a great design that also works. "Working code isn't enough." It takes two kinds of investment:

- **Proactive:** exploring alternative designs, anticipating likely changes, writing documentation.
- **Reactive:** fixing a design problem properly when you find it, instead of patching around it.

How much: about 10–20% of total development time. Strategic programming starts slower, but the gains compound; Ousterhout estimates the break-even point at 6 to 18 months, after which the team is faster. Tactical shortcuts are technical debt, and unlike financial debt, most of it is never repaid.

On startups: the pressure to ship tempts teams into tactical work, and the resulting codebase is hard to fix later and drives away strong engineers. Facebook's "move fast and break things" produced a codebase it later had to stabilize, and the motto became "move fast with stable infrastructure." Google and VMware grew at the same time with a strong design culture. Companies can succeed either way, but a codebase that values design is faster and more rewarding to work in.

The investment has to be continuous. Deferring cleanup "until after the crunch" fails, because another crunch follows, and the delay becomes permanent.

---

## Design It Twice

Before committing to a design, consider at least two **radically different** alternatives. Variations on one idea don't count; the point is to see the design space.

The book's example is the text class for an editor. One design is line-oriented (insert and delete lines), another character-oriented (insert and delete characters), a third range-oriented (insert a string at a position, delete a range of positions). Line-oriented makes callers split and join lines for ordinary edits; character-oriented makes callers loop; range-oriented fits what callers do.

How to compare:

- **Ease of use for the callers** is the most important criterion.
- **Which interface is simpler?**
- **Which is more general-purpose?**
- **Which allows the more efficient implementation?**

Often the best design takes features from several alternatives. And if none is attractive, the comparison tells you what's wrong, which is a good input to a third design.

It applies at every level: interfaces first, then implementations, then whole systems and features. The cost is small (an hour or two for a class) against the cost of living with a weak design. Ousterhout notes it's hardest for smart people to accept: they're used to their first idea being good enough, and on hard problems it isn't.

### A Template for Plans

Beyond the book: when an agent plans a design, it should write the alternatives out where the human can see them.

```text
Abstraction: <one sentence: what this module is and what it hides>

Option A — <name>
  Interface: <signatures>
  Common-case caller:
    <3–5 lines of calling code>
  Hides: <decisions callers no longer make>
  Callers must know: <everything else, including errors and ordering>

Option B — <a radically different name>
  (same fields)

Comparison: simpler interface? more general? easier common case? efficient?
Choice: <option, or a combination>, because <reason>
```

When two options are close, present both and let the human choose.

---

## Modifying Existing Code

Small tactical changes to existing code add special cases and dependencies; over many changes, the design decays. The goal for every change: when you're done, the system should have "the structure it would have had if you had designed it from the start with that change in mind."

- **If you're not making the design better, you're probably making it worse.** Resist the minimal patch when the design no longer fits the change; refactor so it does.
- **Fix nearby flaws** you notice, even when the change itself doesn't require refactoring.
- **When the proper fix is too expensive** (a deadline, a compatibility constraint), look for an approach that is nearly as clean but takes days instead of months. Or ask for time to clean up afterward. Organizations should budget for this.

**Beyond the book,** for agents: "refactor so it fits" is bounded by the task the human gave you. Make the improvement that fits inside the change. When the clean fix is larger (it touches other teams' code, changes a public interface, or would triple the diff), describe it and ask rather than expanding scope on your own. A stack of small improvements the human can review is better than one large one they can't.

Keeping documentation accurate while modifying code is covered in [`comments-and-names.md`](comments-and-names.md#keeping-comments-accurate).

---

## Consistency

Similar things done in similar ways, and different things done differently. Once a reader learns one instance, they understand the others, and they can safely assume that code that looks familiar behaves as expected.

Where consistency matters:

- **Names** (see [`comments-and-names.md`](comments-and-names.md#consistent)).
- **Coding style,** beyond what the compiler enforces.
- **Interfaces:** several implementations of one interface are easy to learn once you know one.
- **Design patterns:** known solutions read quickly.
- **Invariants:** "every line ends with a newline" removes special cases everywhere.

How to keep it:

- **Document the conventions** somewhere prominent, and local invariants in the code.
- **Enforce them with tools** that block violations. Ousterhout's example: a commit check that rejected carriage returns ended a recurring Unix/Windows line-ending problem. Code review teaches the rest.
- **When in Rome, do as the Romans do.** Before writing new code, study how the existing code is organized and follow it. Before making a design decision, look for a similar decision elsewhere and reuse it.
- **Don't change existing conventions lightly.** "Having a 'better idea' is not a sufficient excuse to introduce inconsistencies." Before changing one, ask whether you have new information that wasn't available when it was set, and whether the new way is good enough to justify updating all the old code. If not, keep the old way.

**Taking it too far:** consistency also means different things look different. Reusing a name for an unrelated purpose, or forcing a pattern onto a problem it doesn't fit, breaks the promise that "if it looks like x, it is x."

---

## Decide What Matters

"Separate what matters from what doesn't matter and emphasize the things that matter." The book's last principle, added in the second edition, is close to a summary of all the others.

**How to decide.** External constraints (performance targets, regulations) sometimes decide for you. Otherwise:

- **Look for leverage:** one idea that solves many problems, or one piece of knowledge that explains many things. A general `delete(start, end)` solves many problems; `backspace` solves one. Knowing a variable's invariant lets you predict its behavior in many places.
- **Compare options,** as in design it twice: when naming, list candidate words and keep the few that carry the most information.
- **Make a hypothesis** when unsure ("I think this is what matters most"), build on it, and learn from whether it holds. Especially useful early in a career.

**Minimize what matters.** Fewer constructor arguments, with defaults for the common case. Errors handled at a low level, so they don't matter higher up. Settings computed from the system's behavior rather than exposed for someone to tune. Every piece of hidden information is one less thing that matters outside its module.

**Emphasize what matters** through:

- **Prominence:** interface documentation, names, the parameters of heavily used functions.
- **Repetition:** key ideas recur throughout the design.
- **Centrality:** important ideas sit at the core and shape the structure around them, like an operating system's device-driver interface, which thousands of drivers depend on.

Conversely, things that don't matter should be hidden, rarely encountered, and kept from shaping the structure.

**The two mistakes:**

1. **Treating too many things as important.** Unimportant details clutter the design: arguments most callers don't care about, Java's forced choice between buffered and unbuffered I/O. Shallow classes often come from this.
2. **Missing what is important.** Important information gets hidden, or needed functionality is missing so developers keep recreating it. This produces unknown unknowns.

Ousterhout calls the ability to tell the difference **good taste**, and names it as the mark of a good designer. It applies beyond code: a document should introduce a few key ideas early and hang the details from them, and a life should spend most of its energy on the few things that matter most.

---

## Designing for Performance

Don't optimize every statement; that slows development and adds complexity. Don't ignore performance either; a system designed without regard for it can end up several times slower than necessary, with no single place to fix. Choose designs that are naturally efficient *and* clean. Simple code tends to be fast, because it does less unnecessary work.

**Know what's expensive** (the book's rough figures):

| Operation | Cost |
|---|---|
| Network round trip | 10–50 µs within a datacenter; 10–100 ms across a wide-area network |
| Storage I/O | 5–10 ms for disk; 10–100 µs for flash |
| Dynamic memory allocation | Significant, including freeing and garbage collection |
| Cache misses | Often as large a factor as the computation itself |

**Beyond the book,** the TypeScript-service equivalents: N+1 queries, `await` inside a loop where the calls could run concurrently or be batched, re-fetching data the request already loaded, and serializing large objects across process boundaries.

When two designs are equally simple, pick the more efficient one: a hash map over an ordered map when order doesn't matter. When a faster design is more complex, start simple unless there's clear evidence performance will matter, and prefer complexity that stays hidden inside a module over complexity that leaks into interfaces. Deep modules tend to be faster than shallow ones, because each layer crossing costs something.

**Measure before modifying.** Intuition about performance is unreliable, even among experienced developers. Measure to find where time actually goes and to get a baseline. If a change doesn't produce a measurable improvement, revert it, unless it also made the code simpler.

**Design around the critical path:**

1. Look for a fundamental fix first: a cache, a better algorithm, a different data structure.
2. Failing that, ask what the minimum code for the common case would be, ignoring the existing structure and the special cases. That's the ideal.
3. Design a clean structure as close to the ideal as possible, adding abstraction only where it's needed.
4. Take special cases off the critical path. Ideally one test at the start detects every special case, and the common path runs with no further checks. Special cases can be handled elsewhere, with simpler and slower code.

**The RAMCloud `Buffer` example.** A `Buffer` holds data as a list of chunks. The most common operation is appending a little data to the last chunk. The original code passed through three shallow methods with the same signature and checked about six conditions, some twice. The redesign added one field, the free space after the last chunk (zero whenever a direct append isn't possible), so one test covers every special case and the fast path is a single method. The class shrank by about 20% (1,886 to 1,476 lines), appending one byte went from 8.8 ns to 4.75 ns, and creating, appending to, and destroying a small buffer went from 24 ns to 12 ns. The faster code was also the simpler code.
