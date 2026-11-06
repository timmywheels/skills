# Errors and Special Cases

Detail for section 7 of `SKILL.md`, covering chapter 10 of the book plus the special-case material from chapters 6 and 20. Book examples are kept; TypeScript translations and the section on throwing versus returning are new.

## Table of Contents

- [Why Errors Add Complexity](#why-errors-add-complexity)
- [Too Many Errors](#too-many-errors)
- [1. Define Errors Out of Existence](#1-define-errors-out-of-existence)
- [2. Mask Errors](#2-mask-errors)
- [3. Aggregate Errors](#3-aggregate-errors)
- [4. Just Crash](#4-just-crash)
- [Special Cases](#special-cases)
- [Taking It Too Far](#taking-it-too-far)
- [Beyond the Book: Throwing Versus Returning in TypeScript](#beyond-the-book-throwing-versus-returning-in-typescript)
- [Questions for Every New Error](#questions-for-every-new-error)

---

## Why Errors Add Complexity

In the book, an *exception* is any uncommon condition that changes the normal flow of control. That includes thrown errors and also special return values: `null`, `-1`, `{ ok: false }`. They come from:

- bad arguments or configuration from callers,
- operations that fail (I/O errors, unavailable resources),
- distributed-system problems (lost or delayed messages, servers that don't respond),
- bugs or inconsistencies the code detects in itself.

Handling them is expensive:

- **The handling itself is hard.** Code must either push forward (retry, fall back to a replica) or abort and restore a consistent state. Both are tricky.
- **Handlers create new exceptions.** Retries produce duplicates. Recovery hits secondary failures.
- **The syntax is noisy.** `try`/`catch` boilerplate can outweigh the normal logic, and many small `try` blocks fragment the code.
- **Handlers rarely run, so they are rarely tested,** and they fail when they are finally needed. The book cites a study finding that more than 90% of catastrophic failures in distributed data-intensive systems came from incorrect error handling.

---

## Too Many Errors

Programmers define unnecessary errors because more error detection feels safer. The result is an overly defensive style, and every error is a branch some caller must handle.

Ousterhout's own example: Tcl's `unset` command threw an error if the variable didn't exist. Most callers used `unset` to clean up state that might or might not exist, so they wrapped every call in a `catch` and ignored the error. He calls it one of his biggest mistakes in Tcl's design.

Throwing is easy and handling is hard, so throwing often just passes a problem to callers who know even less about what to do. The errors a module can raise are part of its interface, so a module with many of them has a complex interface and is shallow. The goal is to **reduce the number of places that must handle errors.**

```typescript
export async function archiveMember(memberId: string) {
  const member = await members.findById(memberId);
  if (!member) throw new MemberNotFoundError(memberId);
  if (member.archivedAt) throw new MemberAlreadyArchivedError(memberId);
  return members.update(memberId, { archivedAt: new Date() });
}
```

A bulk-archive job, a retry, or a double-click all have to catch `MemberAlreadyArchivedError` and ignore it. Defined as "make sure this member is archived," the second case is success: return the member as it is. Whether `MemberNotFoundError` stays depends on whether callers can act on it. An admin screen can show "member not found"; a cleanup job can't do anything useful with it.

---

## 1. Define Errors Out of Existence

Change the definition of the operation so that the error case is a normal case.

The book's examples:

- **Tcl `unset`.** Redefined from "delete a variable" to "make sure a variable no longer exists," calling it on a missing variable has already achieved its goal.
- **File deletion.** Windows refuses to delete a file that a process has open, so users hunt for the process or reboot. Unix removes the name from the directory immediately and reports success; processes that already have the file open keep using it, and the data is freed when the last one closes it. Neither the deleter nor the other processes see an error.
- **Java `substring`.** It throws `IndexOutOfBoundsException` when an index is out of range, so getting "the characters in this range, if any" takes 5–10 lines of clamping. Defined as "the characters whose index is at least `begin` and less than `end`," it never needs to throw and is deeper. Python's slices already work this way, and so does JavaScript's `slice`.

To the objection that exceptions catch bugs, Ousterhout's answer is that exceptions add complexity, complexity causes bugs, and simpler APIs need less code. Fewer defects come from simpler software.

TypeScript translations:

| Throws | Defined away |
|---|---|
| `removeTag` throws if the tag is absent | "Ensure the tag is absent" |
| `cancel` throws if already canceled | "Ensure it's canceled"; return the current state |
| `fs.unlink` throws `ENOENT` | `fs.rm(path, { force: true })` |
| `create` throws on a duplicate | Upsert, or return the existing row behind a unique key |
| `getPage(n)` throws past the last page | Return an empty page |
| `subscribe(listener)` throws if already subscribed | Subscribing twice is the same as subscribing once |

---

## 2. Mask Errors

Handle the error at a low level so higher levels never see it. Masking is pulling complexity downward.

- **TCP** retransmits lost packets. Applications never learn a packet was lost.
- **NFS** is the controversial example. When a file server stops responding, clients retry indefinitely; applications hang, and the console eventually shows a message. Ousterhout defends it: applications could do little with an error, retrying once inside NFS is simpler than in every application, and aborting would crash user work that would then have to be restarted. Users who tire of waiting can abort.

TypeScript translations:

- A provider client retries transient failures (timeouts, 429s, 5xx) itself, with backoff.
- A transaction helper retries deadlocks.
- A cache read that fails falls through to the source of truth.

**Beyond the book:** masked errors should still leave a trace for operators, such as a log line or a metric. Masking hides an error from callers, not from the people running the system.

---

## 3. Aggregate Errors

Handle many errors with one piece of code.

- **The web server.** Students wrapped every `getParameter` call in its own handler for `NoSuchParameter`, each producing the same error response. Instead, let the exceptions propagate to the top-level dispatcher, where one handler builds the error response from the message in the exception. Other errors (wrong type, missing permission) work the same way, and new handlers plug in without changes if their errors share a base class.
- **The request loop.** More generally, a loop that processes requests can define an error that means "abort this request," caught once near the top of the loop, which cleans up and moves on.
- **RAMCloud.** Many minor errors, such as finding a corrupted object, are escalated into a server crash. The system already has to recover from crashes using replicas, so one well-tested recovery path covers many rare cases. Escalation is costly, so it suits rare errors, not frequent ones like lost packets.

Masking suits errors from low-level libraries; aggregation suits errors that propagate through several levels.

TypeScript translations:

```typescript
class HttpError extends Error {
  constructor(
    readonly statusCode: number,
    message: string,
  ) {
    super(message);
  }
}

app.setErrorHandler((error, request, reply) => {
  if (error instanceof HttpError) {
    return reply.status(error.statusCode).send({ message: error.message });
  }
  request.log.error(error);
  return reply.status(500).send({ message: "Something went wrong" });
});
```

Handlers anywhere in the request throw `HttpError` (or a subclass) and never format a response themselves. Other examples: a React error boundary per screen, and a job runner with one `catch` per job that records the failure and schedules a retry.

---

## 4. Just Crash

For errors that are rare and hard or impossible to handle, print diagnostics and abort.

- **Out of memory.** C's `malloc` returns `NULL` on failure; checking every call adds complexity, and a forgotten check causes a confusing crash later. Ousterhout's `ckalloc` wrapper checks once and aborts with a message. In languages where allocation throws, catching it is pointless because the handler needs memory too.
- **Unrecoverable I/O errors, sockets that can't be opened, internal inconsistencies:** crash with a clear message.

Not always appropriate: a replicated storage system must recover lost data from replicas when it hits an I/O error, because that recovery is the reason the system exists.

**Beyond the book:** in a server, "crash" usually means "fail this request or job loudly" (an error response, an error log, an alert) rather than killing the process. In TypeScript, the typical case is an exhaustive `switch` whose `default` branch receives a value of type `never`: throw there, and don't write recovery code for states the types say can't happen.

---

## Special Cases

Special cases are the same problem in miniature: each one is a branch to understand, test, and get wrong. The best fix is a design where the normal path handles them.

- **The editor selection.** Most students tracked "no selection" as a separate state and checked for it everywhere. Represented as an empty selection whose start equals its end, copy and delete need no checks: copying zero characters and deleting an empty range already work.
- **The critical path.** In RAMCloud's `Buffer` class, appending data passed through about six checks for special cases. The redesign added one field, the space available after the last chunk (zero when a direct append isn't possible), so a single test covers every special case and the common path does no further checking. See [`strategic-design.md`](strategic-design.md#designing-for-performance).

TypeScript patterns:

| Special case | Designed out |
|---|---|
| `T[] \| undefined` checked at every use | `T[]`, defaulting to `[]` at the boundary |
| Optional dependency checked at every call | A no-op implementation as the default |
| First or last loop iteration handled separately | An initial value or sentinel that makes it normal |
| Several flags checked on the hot path | One derived value tested once up front |

---

## Taking It Too Far

Defining errors away or masking them is right only when callers don't need the information. Ousterhout's example of overdoing it: a student networking module that masked every network error, so applications couldn't tell a lost message from a failed server and couldn't be made robust. When callers need to know, the error must be exposed.

**Beyond the book:** agents produce a recognizable set of fake simplifications. Each makes an interface look simpler by hiding something a caller needed:

| Fake simplification | What it hides |
|---|---|
| `catch {}` | That the operation failed |
| `catch { return []; }` | The difference between "failed" and "empty" |
| `value ?? 0` on a lookup | The difference between "unknown" and "zero" |
| A default for a required setting | That the environment is misconfigured |
| Retry forever, with no log or limit | That a dependency is down |
| Averaging an empty list to `0` | That there was nothing to average |

The rule: an error stays visible when a caller would do something different if it knew. A declined payment, a validation error the user must fix, a partial failure in a batch, a write that didn't happen: all part of the interface.

---

## Beyond the Book: Throwing Versus Returning in TypeScript

The book is language-neutral about mechanism. In TypeScript, the choice of mechanism decides where the handling lands:

- **Return a discriminated union for outcomes callers must handle as normal flow:** a card declined, a slot already full, a validation failure. The outcome is in the signature, and the compiler makes callers deal with it.

  ```typescript
  type BookingResult =
    | { status: "booked"; booking: Booking }
    | { status: "full"; waitlistPosition: number }
    | { status: "not_eligible"; reason: EligibilityReason };
  ```

- **Throw for rare failures that nobody nearby can fix:** the database is down, an invariant broke. Aggregate them at one boundary.
- **Never both for the same condition.**
- **Don't wrap everything in a `Result` type.** Making every function return `Result` spreads error handling to every caller, which is the opposite of aggregation.

---

## Questions for Every New Error

Before adding a `throw`, an error return, or a new error class:

1. **Can the definition change so this isn't an error?** "Ensure X," not "do X."
2. **Can this module handle it?** Retry, fall back, or use a default that is actually correct.
3. **Can it propagate to a handler that already exists?**
4. **Is it rare and unrecoverable?** Then fail loudly, without recovery code.
5. **If callers must know:** is it in the signature or interface documentation, along with what callers should do about it?
