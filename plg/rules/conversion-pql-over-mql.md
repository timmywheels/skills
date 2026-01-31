---
title: PQLs Over MQLs
impact: HIGH
impactDescription: 5-10x better conversion rates
tags: plg, conversion, leads, sales
---

## PQLs Over MQLs

Product-Qualified Leads (PQLs) convert far better than Marketing-Qualified
Leads (MQLs) because they've already experienced value. Don't qualify leads
by form fills—qualify them by product usage.

```
Traditional: MQL (filled form) → Sales call → Maybe value
PLG:         Usage signals → PQL (experienced value) → Self-serve upgrade

PQLs convert 5-10x better because they've already received value.
```

**Incorrect (MQL-based qualification):**

```typescript
const isQualifiedLead = (lead: Lead) => {
  return (
    lead.downloadedWhitepaper &&
    lead.attendedWebinar &&
    lead.companySize > 100 &&
    lead.filledContactForm
  );
};

// Problem: None of this means they've used or valued the product
```

**Correct (PQL-based qualification):**

```typescript
const pqlSignals = {
  // Usage signals (strongest)
  usageThreshold: {
    signal: (user) => user.apiCalls >= user.plan.limit * 0.8,
    score: 30,
  },
  featureDepth: {
    signal: (user) => user.premiumFeaturesUsed >= 3,
    score: 25,
  },

  // Team signals (expansion indicator)
  teamGrowth: {
    signal: (user) => user.invitesSent >= 3,
    score: 25,
  },
  collaboration: {
    signal: (user) => user.sharedResources >= 5,
    score: 15,
  },

  // Engagement signals (retention indicator)
  retention: {
    signal: (user) => user.activeDaysLastWeek >= 4,
    score: 20,
  },
  workflowCompletion: {
    signal: (user) => user.coreWorkflowsCompleted >= 5,
    score: 20,
  },
};

const calculatePQLScore = (user: User): number => {
  return Object.values(pqlSignals).reduce((score, { signal, score: points }) => {
    return signal(user) ? score + points : score;
  }, 0);
};

// PQL threshold: score >= 60
const isPQL = (user: User) => calculatePQLScore(user) >= 60;

// Route PQLs to appropriate conversion path
const handlePQL = async (user: User) => {
  if (user.plan === "free" && isPQL(user)) {
    // Self-serve: show contextual upgrade prompt
    await showUpgradePrompt(user, "usage_based");

    // High-value accounts: optional sales touch
    if (user.teamSize > 20) {
      await notifySales(user, "pql_enterprise");
    }
  }
};
```

**PQL vs MQL comparison:**

| Metric           | MQL          | PQL              |
| ---------------- | ------------ | ---------------- |
| Conversion rate  | 1-2%         | 10-25%           |
| Sales cycle      | Weeks/months | Days/self-serve  |
| Trust level      | Low          | High (used it)   |
| Churn risk       | High         | Low (proven fit) |
