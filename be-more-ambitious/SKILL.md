---
name: be-more-ambitious
description: >-
  Turn a narrow ask into a more ambitious one before doing it: give the model
  the goal beyond the goal (altitude), the methods of a field-leading expert
  (depth), and the methods of other fields (range). Use when the user says "be
  more ambitious", "think bigger", "go deeper", "what's the goal beyond the
  goal", or asks to rewrite a prompt; when the same kind of task has been asked
  for more than twice (fix this crash, update these numbers, write this page);
  when a task would end in a meeting or a decision; when a result is about to
  be checked only by the agent that made it; or when planning a report,
  analysis, page, or program of work. From Felix Rieseberg, "How to be more
  ambitious: a tutorial for humans with models" (felixrieseberg.com).
---

# Be more ambitious

The ask you were given is someone's best guess at how to get an outcome.
Before doing it, ask what they want to happen next. Then pick one or more of
the three kinds of ambition below. Speed (the same work, faster) is the
default and needs no skill.

| Kind | Question | Use when |
| --- | --- | --- |
| Altitude | What's the goal beyond the goal? | The work leads to a decision or a meeting, or the same task keeps coming back |
| Depth | How would the best person in the field do this? | The task should be done really well, not just done |
| Range | Which other fields would see this differently? | The output has to persuade, look great, or survive an adversary |

## Altitude: own the outcome, not the task

1. **Name the outcome above the task.** "Fix this crash" becomes "keep the app
   crash-free". "Fix this failing build" becomes "keep the main branch green".
2. **Take all the data, not the template.** Don't just update the numbers. Ask
   what changed, why, and what to do about it.
3. **End in decisions.** Find the few decisions that matter most and argue for
   and against each.
4. **Question why the artifact exists.** A quarterly review exists so problems
   get caught early. Design a better way to do that, prove from past data that
   it would have caught things sooner, and run it beside the old one.
5. **Build the plumbing first.** An outcome can only be owned if the agent can
   see it, reproduce it and verify the fix on its own: logs, crash reports, a
   way to reproduce, a test. If the plumbing is missing, building it is the
   first task. Say so and propose it rather than asking for the next task.

Altitude is weaker for creative work. It's not a license to widen scope
silently: state the higher outcome and get a yes before working at it.

## Depth: the expert's methods, not the expert's vocabulary

1. **Use the expert's tools.** Acting like a pro usually changes the words, not
   the method. Reach for real data and real instruments: a database, the
   analytics, statistics, a profiler, a benchmark, the real API. Don't settle
   for a markdown table of guesses.
2. **Set an unreasonable target and treat it as reasonable.** 10s to 5s? Aim for
   200ms. If it looks impossible, keep going before reporting that it is.
3. **Check what we believe against raw data.** "We think churn is onboarding.
   Reproduce that from the raw events." Report it when the data disagrees.
4. **Produce something new.** Once deep in the data, look for the signal nobody
   has named yet, and show it holds on data it hasn't seen.
5. **Use an outside judge.** Never grade your own work. Use something you can
   run: a test suite, CI, a type checker, a benchmark, a real-API probe, a
   screenshot, a held-out dataset, or a separate reviewer given the work and
   not your conclusion.

Before starting a depth task, answer these four in a few lines:

1. What would a top professional do here that wasn't asked for?
2. What is the rawest material they'd start from, instead of what's already
   been summarized?
3. Which methods and tools would they use?
4. What, other than me or the user, will show the result is right?

Then do that version.

## Range: borrow other fields' methods

1. **Turn the noun into verbs.** A quarterly review isn't "a deck". It's
   "getting busy people to believe a story about cause and effect and decide,
   in 30 minutes". A landing page isn't "a page". It's "getting a stranger to
   trust us enough to sign up". The verbs show which fields help.
2. **Assemble the crew.** If the stakes were huge, who would be on the team?
   Analyst, journalist, editor, designer. Then a red team: a skeptical
   customer, a competitor, a short seller, a forensic accountant.
3. **Use their methods, not their voice.** Don't write like a short seller.
   Research the way one does: look for the weakest claim and try to break it.
   Role play makes the transcript weirder without making the work better.

Subagents are a good fit for range: one per field, each given the material and
not your conclusion.

## Output

Before the work, tell the user in plain words:

- **Asked:** the task as given, in one line.
- **Goal beyond it:** the outcome it serves, in one line.
- **Ambitious version:** what you'll do instead, and which kind (altitude,
  depth, range).
- **Judge:** what will show it's right, other than you.
- **Missing plumbing:** anything that has to exist first, if any.

Then ask "Do you want me to do the ambitious version?" and wait. If the user
only asked for a prompt rewrite, the rewritten prompt is the deliverable.

## Why this works

People do more when they believe a thing is possible (self-efficacy).
Specific hard goals beat "do your best". Watching someone else do it helps
(after the first sub-two-hour marathon, the runner-up also broke two hours in
the same race). So does permission to fail (Linux was "just a hobby"). Give
the model the same: a specific, hard target, stated as reachable.
