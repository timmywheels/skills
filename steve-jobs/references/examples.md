# Jobs-Style Design Examples

Concrete before/after transformations demonstrating the methodology in action.

## Contents

1. [Feature Request → Simplification](#feature-request--simplification)
2. [Onboarding Flow → First Impression](#onboarding-flow--first-impression)
3. [Settings Page → Invisible Defaults](#settings-page--invisible-defaults)
4. [Error Handling → Graceful Recovery](#error-handling--graceful-recovery)
5. [Dashboard → Focused Action](#dashboard--focused-action)

---

## Feature Request → Simplification

**Scenario:** Users request an "advanced search" with filters for date range, category, author, status, and file type.

### Before (Feature-Driven)

```
Search: [_______________] [Advanced ▼]

Advanced Search:
  Date: [From: ___] [To: ___]
  Category: [Dropdown ▼]
  Author: [Dropdown ▼]
  Status: [Dropdown ▼]
  File Type: [Dropdown ▼]
  [Apply Filters] [Reset]
```

### Jobs Analysis

- Why are users searching? To find something specific.
- Why do they want filters? Because basic search returns too much noise.
- Root problem: Search results aren't relevant enough.

### After (Problem-Driven)

```
Search: [_______________]
```

With intelligent search that:

- Understands natural language ("my documents from last week")
- Prioritizes recent and frequently accessed items
- Shows smart suggestions as you type
- Learns from what you actually click

**Principle applied:** Eliminate the need for the feature rather than building the feature.

---

## Onboarding Flow → First Impression

**Scenario:** New user signs up for a project management tool.

### Before (Information-Driven)

```
Welcome! Let's set up your workspace.

Step 1 of 6: Personal Information
  Name: [_______________]
  Role: [Dropdown ▼]
  Company size: [Dropdown ▼]
  Industry: [Dropdown ▼]

Step 2 of 6: Workspace Settings
  [Continue →]
```

### Jobs Analysis

- What does the user want? To manage a project.
- What are we asking? Everything except that.
- First impression: "This tool is about filling out forms."

### After (Action-Driven)

```
What's the first thing you want to get done?

[Create a project] [Invite your team] [Explore on my own]
```

User clicks "Create a project" → immediately in the product, doing the thing they came to do. Gather other information later, contextually, or infer it.

**Principle applied:** The best onboarding is no onboarding. Get users to value immediately.

---

## Settings Page → Invisible Defaults

**Scenario:** Email notification preferences.

### Before (Complete Control)

```
Notification Settings

Email Notifications:
☑ New messages
☑ Mentions
☑ Task assignments
☑ Task completions
☑ Comment replies
☑ Project updates
☑ Weekly digest
☐ Daily digest
☑ Team announcements
☐ Marketing updates

Frequency: [Immediate ▼]
Quiet hours: [None ▼]

[Save Changes]
```

### Jobs Analysis

- Why do users visit settings? Because defaults failed them.
- What do most users actually want? Relevant notifications, not noise.
- Each checkbox is an admission of failure.

### After (Smart Defaults)

```
Notifications

[Essential only]  ●
Only things that need your attention right now.

[Keep me informed] ○
Activity on projects you're involved in.

[Everything]      ○
All activity across your workspace.
```

One choice. Clear trade-offs. Sensible default pre-selected.

**Principle applied:** Settings are a tax on users. Minimize the tax through smart defaults and simple choices.

---

## Error Handling → Graceful Recovery

**Scenario:** User's document fails to save.

### Before (Technical Accuracy)

```
⚠️ Error

Unable to save document.

Error code: CONN_TIMEOUT_503
The server connection timed out after 30000ms.
Please check your network connection and try again.

[OK]
```

### Jobs Analysis

- What does the user care about? Not losing their work.
- What do they need to know? That their work is safe.
- What do they need to do? Probably nothing—system should handle it.

### After (User-Centered)

```
✓ Your changes are safe

We're having trouble connecting right now,
but everything you've written is saved locally.

We'll sync automatically when connection returns.

[Got it]
```

Even better: No message at all. Just a subtle icon showing "syncing..." that resolves silently when connection returns.

**Principle applied:** Handle problems; don't report them. Users want outcomes, not status updates.

---

## Dashboard → Focused Action

**Scenario:** Homepage for a fitness tracking app.

### Before (Data Display)

```
Dashboard                    [+ Add] [⚙️]

Today's Stats
┌─────────────────────────────────────┐
│ Steps: 4,521    Calories: 1,847     │
│ Active mins: 23  Heart rate: 72     │
│ Sleep: 6h 43m   Water: 3 glasses    │
└─────────────────────────────────────┘

Weekly Progress     Monthly Trends
[Chart]             [Chart]

Recent Activities   Achievements
[List of 10 items]  [Grid of badges]

Challenges          Community
[3 active]          [Feed]
```

### Jobs Analysis

- Why does someone open a fitness app? To do something healthy.
- What does this dashboard enable? Looking at numbers.
- The ONE thing: Help the user take their next healthy action.

### After (Action-Oriented)

```
Good morning, Sarah.

You're 2,000 steps from your goal.
A 15-minute walk would do it.

        [Start a walk →]

──────────────────────────────

Yesterday you walked 8,500 steps.
Your best week this month.
```

Stats exist, but buried. The primary UI answers: "What should I do right now?"

**Principle applied:** Products should drive action, not display information. Data serves decisions.

---

## Applying These Patterns

When facing a design challenge:

1. **Identify the conventional approach** — What would most products do?
2. **Question the premise** — Why does this convention exist?
3. **Find the user's actual goal** — What are they trying to accomplish?
4. **Design for that goal** — Remove everything that doesn't serve it
5. **Validate the emotion** — How does the result _feel_ to use?
