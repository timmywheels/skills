---
title: Self-Serve Upgrade Paths
impact: HIGH
impactDescription: removes sales bottleneck
tags: plg, conversion, pricing, self-serve
---

## Self-Serve Upgrade Paths

The user should upgrade themselves at the moment of realized need. No sales
call required for obvious use cases. Design upgrade flows that feel helpful,
not pushy.

**Conversion triggers:**

| Trigger Type        | Signal                  | Response                  |
| ------------------- | ----------------------- | ------------------------- |
| **Usage limit**     | Hit free tier cap       | In-context upgrade prompt |
| **Feature gate**    | Tried premium feature   | "Unlock this" preview     |
| **Team growth**     | Invited 3+ people       | Team plan suggestion      |
| **Time-based**      | Active for 14 days      | Value summary + upgrade   |
| **Value milestone** | Completed core workflow | "Get more with Pro"       |

**Incorrect (sales-gated upgrade):**

```typescript
const UpgradeButton = () => (
  <Button onClick={() => openModal(<ContactSalesForm />)}>
    Contact Sales to Upgrade
  </Button>
);

// Problem: Friction kills self-serve momentum
// Users who want to pay you can't
```

**Correct (self-serve with optional sales):**

```typescript
const UpgradePath = ({ user }: { user: User }) => {
  const showSalesOption = user.teamSize > 50; // Enterprise threshold

  return (
    <UpgradeFlow>
      {/* Primary: Self-serve */}
      <PricingTable
        plans={plans}
        currentPlan={user.plan}
        onSelectPlan={handleSelfServeUpgrade}
      />

      {/* Secondary: Sales for complex needs */}
      {showSalesOption && (
        <Text>
          Need custom terms or volume pricing?{" "}
          <Link onClick={openSalesChat}>Talk to sales</Link>
        </Text>
      )}
    </UpgradeFlow>
  );
};

const handleSelfServeUpgrade = async (plan: Plan) => {
  // Instant upgrade, no friction
  const checkout = await createCheckoutSession(plan);
  redirect(checkout.url);
};
```

**Implementation pattern:**

```typescript
const conversionTriggers = {
  usageLimit: {
    condition: (user) => user.usage >= user.plan.limit,
    message: "You've reached your plan limit. Upgrade for unlimited access.",
    placement: "in-context", // Where they hit the limit
    urgency: "soft", // Inform, don't block
    cta: "See Plans",
  },

  featureGate: {
    condition: (action) => PREMIUM_FEATURES.includes(action),
    message: "This is a Pro feature. See what you'd unlock:",
    placement: "inline-preview", // Show blurred/grayed version
    urgency: "none", // Discovery, not pressure
    cta: "Unlock with Pro",
  },

  teamGrowth: {
    condition: (user) => user.invites >= 3 && user.plan === "individual",
    message: "Growing team? Team plans include admin controls, SSO, and more.",
    placement: "contextual-banner",
    urgency: "soft",
    cta: "Explore Team Plans",
  },
};
```

**Pricing page best practices:**

- Lead with value, not feature lists
- Social proof above the fold
- Clear "recommended" tier
- Annual discount visible (but don't hide monthly)
- Free tier feels generous, not crippled
- FAQ answers common objections
- No "Contact Sales" for obvious self-serve tiers
