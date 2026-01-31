---
title: Notification Value
impact: MEDIUM
impactDescription: prevents notification fatigue and unsubscribes
tags: plg, retention, notifications, engagement
---

## Notification Value

Every notification must deliver value, not just ask for attention. Notifications
that don't deliver value train users to ignore you—or worse, unsubscribe.
Treat notification inventory as precious and finite.

**Incorrect (attention-seeking notifications):**

```typescript
const badNotifications = [
  // Guilt trips
  "You haven't logged in for 3 days!",
  "We miss you!",

  // Marketing, not value
  "Check out our new feature!",
  "We just launched something cool!",

  // Vague, no value
  "Your friends are using the app!",
  "Something happened!",

  // Engagement bait
  "You have unread notifications",
  "Don't miss out!",
];
```

**Correct (value-delivering notifications):**

```typescript
const goodNotifications = [
  // Specific, actionable value
  "Sarah commented on your project: 'Love this approach!'",
  "Your report is ready: Revenue up 12% this month",
  "3 tasks completed by your team today",

  // Time-sensitive, relevant
  "Meeting with Alex starts in 10 minutes",
  "Your file finished processing - ready to download",

  // Social proof with context
  "Your post reached 100 views - here's who's engaging",
];
```

**Implementation:**

```typescript
interface Notification {
  value: string; // REQUIRED: What does user gain from this?
  action: string; // What can they do with this info?
  urgency: "immediate" | "timely" | "informational";
  deepLink: string; // One tap to the value
}

const notificationRules = {
  // Only send if there's real value
  shouldSend: (notification: Notification) => {
    return (
      notification.value !== undefined &&
      notification.action !== undefined
    );
  },

  // Batch low-urgency notifications
  batchingRules: {
    immediate: { maxDelay: 0 }, // Send now
    timely: { maxDelay: 3600 }, // Within an hour
    informational: { batch: "daily_digest" }, // Once per day
  },

  // Respect user preferences
  preferences: {
    quietHours: true,
    channelPreferences: true, // Email vs push vs in-app
    frequencyCaps: true, // Max N per day
  },
};

// Example: Good notification implementation
const sendTaskCompletedNotification = async (task: Task, user: User) => {
  const notification: Notification = {
    value: `${task.assignee.name} completed "${task.title}"`,
    action: "View task and next steps",
    urgency: "timely",
    deepLink: `/tasks/${task.id}`,
  };

  // Check if this passes our value bar
  if (!notificationRules.shouldSend(notification)) {
    return; // Don't spam
  }

  await sendNotification(user, notification);
};
```

**Notification audit checklist:**

| Question                                  | If No                          |
| ----------------------------------------- | ------------------------------ |
| Does this notification deliver value?     | Don't send it                  |
| Can the user take action on it?           | Add a clear CTA or don't send  |
| Is it personalized to this user?          | Make it specific or batch it   |
| Does it respect their preferences?        | Check quiet hours and settings |
| Would I want to receive this?             | If not, neither do they        |
