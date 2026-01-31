---
title: Usage-Based Pricing
impact: HIGH
impactDescription: enables land-and-expand revenue model
tags: plg, expansion, pricing, revenue
---

## Usage-Based Pricing

Revenue should correlate with value delivered. As users get more value, they
pay more—and they're happy to because they're getting more. This alignment
is the foundation of PLG expansion.

**Expansion levers:**

| Lever        | Mechanism                       | Example            |
| ------------ | ------------------------------- | ------------------ |
| **Seats**    | More users = more revenue       | Slack, Notion      |
| **Usage**    | More consumption = more revenue | Twilio, AWS        |
| **Features** | Deeper usage = premium features | Zoom, GitHub       |
| **Tiers**    | Success triggers tier upgrade   | HubSpot, Mailchimp |

**Net Revenue Retention (NRR):**

```
NRR = (Starting MRR + Expansion - Contraction - Churn) / Starting MRR

NRR > 100% = You grow even without new customers
Best PLG companies: 120-150% NRR
```

**Incorrect (flat pricing, no expansion path):**

```typescript
const pricingPlans = [
  { name: "Pro", price: 20, features: "unlimited everything" },
];

// Problem: No expansion revenue
// User pays $20 forever, regardless of how much value they get
// Your most successful customers subsidize everyone else
```

**Correct (usage-aligned pricing):**

```typescript
const pricingPlans = [
  {
    name: "Free",
    price: 0,
    limits: { seats: 3, storage: "1GB", apiCalls: 1000 },
  },
  {
    name: "Pro",
    price: 10, // per seat
    limits: { seats: "unlimited", storage: "100GB", apiCalls: 10000 },
  },
  {
    name: "Team",
    price: 15, // per seat
    limits: { seats: "unlimited", storage: "unlimited", apiCalls: "unlimited" },
    features: ["SSO", "Admin controls", "Audit logs"],
  },
];

// Expansion happens naturally as teams grow
// More seats = more revenue
// More usage = hits limits = upgrades
```

**Implementation:**

```typescript
const expansionTriggers = {
  // Seat-based expansion
  teamGrowth: {
    signal: "team_member_invited",
    threshold: (user) => user.seats >= user.plan.seatLimit * 0.8,
    action: "suggest_seat_upgrade",
    message: "Your team is growing! Add more seats to keep collaborating.",
  },

  // Usage-based expansion
  usageGrowth: {
    signal: "approaching_usage_limit",
    threshold: (user) => user.usage >= user.plan.usageLimit * 0.8,
    action: "proactive_upgrade_offer",
    message: "You're at 80% of your monthly limit. Upgrade to avoid interruption.",
  },

  // Feature-based expansion
  featureAdoption: {
    signal: "premium_feature_attempted",
    threshold: (user) => user.premiumFeaturesAttempted >= 3,
    action: "personalized_upgrade_pitch",
    message: "You've been exploring advanced features. Here's what Pro unlocks.",
  },
};
```

**Pricing design principles:**

- Free tier should be genuinely useful (not crippled)
- Upgrade triggers should feel helpful, not punitive
- Pricing should scale with value delivered
- Enterprise tier for high-touch, custom needs
- Transparent pricing (no "contact sales" for obvious tiers)
