---
title: Design Habit Loops
impact: HIGH
impactDescription: drives organic return without marketing spend
tags: plg, retention, engagement, habits
---

## Design Habit Loops

Retention requires recurring value. One-time value = churn. Design
cue → routine → reward cycles that create cravings and drive organic return.

**The habit loop:**

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│   CUE   │ ──→ │ ROUTINE │ ──→ │ REWARD  │ ──→ │ CRAVING │
│(trigger)│     │ (action)│     │ (value) │     │ (repeat)│
└─────────┘     └─────────┘     └─────────┘     └─────────┘
     ↑                                               │
     └───────────────────────────────────────────────┘
```

**Incorrect (no habit loop):**

```typescript
// One-time value, no reason to return
const App = () => {
  return (
    <div>
      <ReportGenerator /> {/* Generate report, leave, never return */}
    </div>
  );
};
```

**Correct (habit loop designed in):**

```typescript
const habitLoop = {
  // CUE: External trigger to return
  cue: {
    type: "notification",
    trigger: "daily_digest",
    message: "3 teammates updated projects you follow",
    timing: "user_active_hours", // Learn when they're receptive

    // Make the cue impossible to ignore (in a good way)
    personalization: true, // "Sarah commented" not "Someone commented"
    urgency: "social", // Others are active, you might miss out
  },

  // ROUTINE: Low-friction action
  routine: {
    action: "view_updates",
    friction: "one_click", // Deep link directly to content
    // Don't make them log in, navigate, or search
  },

  // REWARD: Variable, valuable, social
  reward: {
    type: "variable", // Different every time maintains interest
    content: "personalized_updates", // Relevant to them
    social: "see_team_activity", // Human connection
    // Immediate value, not delayed gratification
  },
};

// Implementation
const DailyDigestNotification = ({ user }: { user: User }) => {
  const updates = useTeamUpdates(user);

  if (updates.length === 0) return null; // Don't notify without value

  return (
    <PushNotification
      title={`${updates.length} updates from your team`}
      body={updates[0].preview}
      deepLink={`/activity?highlight=${updates[0].id}`}
      timing={user.preferredNotificationTime}
    />
  );
};
```

**Retention mechanics by type:**

| Mechanic          | How It Works                  | Example              |
| ----------------- | ----------------------------- | -------------------- |
| **Streaks**       | Loss aversion drives return   | Duolingo, GitHub     |
| **Notifications** | External trigger to return    | Slack, Email digests |
| **Social**        | Others' activity creates FOMO | Twitter, LinkedIn    |
| **Data lock-in**  | Your data becomes valuable    | Notion, Airtable     |
| **Workflow**      | Part of daily process         | Linear, Superhuman   |

**Measure the loop:**

```typescript
const habitMetrics = {
  cue_to_routine: "notification_click_rate", // Are cues working?
  routine_to_reward: "session_depth", // Are they getting value?
  reward_to_craving: "organic_return_rate", // Are they coming back unprompted?

  // Target benchmarks
  targets: {
    notification_click_rate: "> 10%",
    session_depth: "> 3 actions",
    organic_return_rate: "> 40% weekly",
  },
};
```
