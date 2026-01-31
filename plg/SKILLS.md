# Product-Led Growth (PLG) Skill

<skill-context>
This skill provides expert guidance on implementing Product-Led Growth patterns. Use this when designing features, user flows, onboarding experiences, pricing models, or growth mechanics. PLG is a go-to-market strategy where the product itself drives acquisition, conversion, and expansion. The core principle: **deliver value first, capture value second.**
</skill-context>

## The PLG Mental Model

```
Traditional Sales-Led:  Marketing → Sales → Value → Payment
Product-Led:            Value → Aha Moment → Habit → Payment → Expansion

PLG flips the funnel: users experience value BEFORE you ask for anything.
```

### Why PLG Works

| Factor | Sales-Led | Product-Led |
|--------|-----------|-------------|
| CAC | High (sales team) | Low (self-serve) |
| Scalability | Linear (headcount) | Exponential (product) |
| User alignment | Company goals | User goals |
| Conversion signal | Demo completed | Value received |

---

## The PLG Growth Loop

Every PLG feature should connect to one of these loops:

```
┌─────────────────────────────────────────────────────┐
│                  ACQUISITION LOOP                   │
│   User gets value → Shares/invites → New users     │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│                  ENGAGEMENT LOOP                    │
│   User acts → Gets reward → Wants to act again     │
└─────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│                  MONETIZATION LOOP                  │
│   User hits limit → Sees value → Upgrades → More   │
│   value → Hits new limit → Upgrades again          │
└─────────────────────────────────────────────────────┘
```

---

## The 6 PLG Pillars

### 1. Activation: The Aha Moment

**Definition**: The moment a user first experiences the core value of your product.

**Implementation Pattern**:
```
1. Identify your aha moment (one specific action)
2. Remove EVERY step between signup and that moment
3. Measure time-to-value (TTV) obsessively
4. Optimize for hours/minutes, not days
```

**Examples**:
- Slack: Send first message in a channel
- Dropbox: First file synced across devices
- Figma: First shape drawn with a collaborator watching

**Anti-patterns to Avoid**:
- Requiring email verification before any value
- Mandatory profile completion
- Feature tours that delay actual usage
- Asking for payment info upfront

**Implementation Checklist**:
- [ ] Can a user get value in under 60 seconds?
- [ ] Is signup friction minimized (SSO, magic links)?
- [ ] Does onboarding guide toward aha moment, not feature tour?
- [ ] Are optional fields actually optional?

---

### 2. Onboarding: Progressive Value Delivery

**Principle**: Teach by doing, not by telling. Each step should deliver immediate value.

**The Onboarding Framework**:
```
┌──────────────────────────────────────────────────────┐
│  SETUP (required)     │  ACTION (guided)    │ HABIT  │
│  - Essential config   │  - First success    │  Loop  │
│  - Minimal friction   │  - Celebrate win    │   ↺    │
│  - Skip everything    │  - Show next value  │        │
│    skippable          │                     │        │
└──────────────────────────────────────────────────────┘
```

**Onboarding Types by Context**:

| Context | Pattern | Example |
|---------|---------|---------|
| B2C Simple | Empty state → First action | Twitter: "Follow 5 people" |
| B2C Complex | Progressive disclosure | Duolingo: 1 lesson, then profile |
| B2B Individual | Template/example data | Notion: Pre-made templates |
| B2B Team | Invite + collaborate | Figma: Share link, edit together |

**Implementation Pattern**:
```typescript
// Onboarding state machine pattern
type OnboardingStep = {
  id: string;
  isRequired: boolean;
  isComplete: (user: User) => boolean;
  valueDelivered: string;  // What user gains from this step
  skipPenalty: 'none' | 'degraded' | 'blocking';
};

// Every step must answer: "What value does the user get NOW?"
```

**Anti-patterns**:
- Long setup wizards before any product usage
- Feature checklists that don't deliver value per step
- Forced tutorials that can't be skipped
- Collecting data you don't immediately use

---

### 3. Virality: Built-In Growth Mechanics

**Definition**: Features that naturally cause users to bring in other users.

**Virality Types**:

| Type | Mechanism | Example |
|------|-----------|---------|
| **Inherent** | Product requires others | Slack (messaging), Zoom (meetings) |
| **Collaborative** | Better with others | Figma (design), Notion (docs) |
| **Word-of-mouth** | So good users talk | Superhuman (email), Linear (issues) |
| **Incentivized** | Rewards for referrals | Dropbox (free space) |

**Viral Coefficient Formula**:
```
K = (invites per user) × (conversion rate of invites)

K > 1 = Exponential growth
K = 0.5 = Each user brings half a user (still valuable!)
```

**Implementation Patterns**:

```typescript
// Inherent virality: Make sharing the core action
const shareDocument = async (doc: Doc, email: string) => {
  // Sharing IS the product, not a bolt-on
  await sendCollaborationInvite(doc, email);
  trackVirality('inherent_share', { docId: doc.id });
};

// Incentivized virality: Clear value exchange
const referralProgram = {
  trigger: 'user_activated',  // Only ask after they love it
  offer: {
    referrer: '1 month free',
    referee: '1 month free',  // Both sides win
  },
  friction: 'one_click',  // Pre-filled message, easy share
};
```

**When to Invest in Each Type**:
- **Inherent**: Build from day 1 if your product is collaborative
- **Collaborative**: Add sharing features early if value compounds with users
- **Word-of-mouth**: Focus on product quality; you can't manufacture this
- **Incentivized**: Add after strong retention; incentives can't fix churn

---

### 4. Conversion: Self-Serve Upgrade Paths

**Principle**: The user should upgrade themselves at the moment of realized need.

**The PQL (Product-Qualified Lead) Model**:
```
Traditional: MQL (filled form) → Sales call → Maybe value
PLG:         Usage signals → PQL (experienced value) → Self-serve upgrade

PQLs convert 5-10x better than MQLs because they've already received value.
```

**Conversion Triggers**:

| Trigger Type | Signal | Response |
|--------------|--------|----------|
| **Usage limit** | Hit free tier cap | In-context upgrade prompt |
| **Feature gate** | Tried premium feature | "Unlock this" CTA |
| **Team growth** | Invited 3+ people | Team plan suggestion |
| **Time-based** | Active for 14 days | "You've done X, Y, Z" |
| **Value milestone** | Completed core workflow | "Get more with Pro" |

**Implementation Pattern**:
```typescript
// Conversion trigger system
const conversionTriggers = {
  usageLimit: {
    condition: (user) => user.apiCalls >= FREE_TIER_LIMIT,
    message: "You've made 1000 API calls this month. Upgrade for unlimited.",
    placement: 'in-context',  // Show where they hit the limit
    urgency: 'soft',  // Don't block, just inform
  },

  featureGate: {
    condition: (action) => PREMIUM_FEATURES.includes(action),
    message: "Advanced analytics is a Pro feature. See what you'd get:",
    placement: 'inline-preview',  // Show grayed-out version
    urgency: 'none',  // Let them discover value, not feel blocked
  },

  teamGrowth: {
    condition: (user) => user.invitesSent >= 3,
    message: "Collaborating with your team? Team plans include...",
    placement: 'contextual-banner',
    urgency: 'soft',
  },
};
```

**Pricing Page Best Practices**:
- Lead with value, not features
- Social proof (logos, testimonials) above the fold
- Clear "most popular" or "recommended" tier
- Annual discount prominently shown
- Free tier should feel generous, not crippled

**Anti-patterns**:
- Paywalls before value demonstration
- Complex pricing requiring sales call
- Hiding pricing (kills trust)
- Feature matrices with 50 rows
- "Contact us" for obvious self-serve use cases

---

### 5. Expansion: Usage-Based Growth

**Principle**: Revenue grows as user value grows. Align your success with theirs.

**Expansion Levers**:

| Lever | Mechanism | Example |
|-------|-----------|---------|
| **Seats** | More users = more revenue | Slack, Notion |
| **Usage** | More consumption = more revenue | Twilio, AWS |
| **Features** | Deeper usage = premium features | Zoom, GitHub |
| **Tiers** | Success triggers tier upgrade | HubSpot, Mailchimp |

**Net Revenue Retention (NRR)**:
```
NRR = (Starting MRR + Expansion - Contraction - Churn) / Starting MRR

NRR > 100% = You grow even without new customers
Best PLG companies: 120-150% NRR
```

**Expansion Triggers**:
```typescript
const expansionSignals = {
  // Seat-based expansion
  teamGrowth: {
    signal: 'team_member_invited',
    threshold: 5,
    action: 'suggest_team_plan',
  },

  // Usage-based expansion
  usageGrowth: {
    signal: 'approaching_tier_limit',
    threshold: 0.8,  // 80% of limit
    action: 'proactive_upgrade_offer',
  },

  // Feature-based expansion
  featureAdoption: {
    signal: 'premium_feature_attempted',
    threshold: 3,  // Tried 3 premium features
    action: 'personalized_upgrade_pitch',
  },

  // Success-based expansion
  valueRealized: {
    signal: 'core_workflow_completed',
    threshold: 10,  // Did it 10 times
    action: 'power_user_features_unlock',
  },
};
```

---

### 6. Retention: Habit Loops

**Principle**: Value must be recurring to create retention. One-time value = churn.

**The Habit Loop**:
```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│   CUE   │ ──→ │ ROUTINE │ ──→ │ REWARD  │ ──→ │ CRAVING │
│(trigger)│     │ (action)│     │ (value) │     │ (repeat)│
└─────────┘     └─────────┘     └─────────┘     └─────────┘
     ↑                                               │
     └───────────────────────────────────────────────┘
```

**Retention Mechanics**:

| Mechanic | How It Works | Example |
|----------|--------------|---------|
| **Streaks** | Loss aversion drives return | Duolingo, GitHub |
| **Notifications** | External trigger to return | Slack, Email digests |
| **Social** | Others' activity creates FOMO | Twitter, LinkedIn |
| **Data lock-in** | Your data becomes valuable | Notion, Airtable |
| **Workflow** | Part of daily process | Linear, Superhuman |

**Implementation Pattern**:
```typescript
// Habit loop implementation
const habitLoop = {
  cue: {
    type: 'notification',
    trigger: 'daily_digest',
    message: "3 teammates updated projects you follow",
    timing: 'user_active_hours',  // Learn when they're receptive
  },

  routine: {
    action: 'view_updates',
    friction: 'one_click',  // Deep link directly to content
  },

  reward: {
    type: 'variable',  // Variable rewards are more engaging
    content: 'personalized_updates',  // Different every time
    social: 'see_team_activity',  // Human connection
  },

  // Measure the loop
  metrics: {
    cue_to_routine: 'notification_click_rate',
    routine_to_reward: 'session_depth',
    reward_to_craving: 'organic_return_rate',
  },
};
```

**Anti-patterns**:
- Notifications without value (annoying, not engaging)
- Streaks for non-daily products (feels forced)
- Gamification without core value (manipulation)
- Dark patterns that erode trust

---

## PLG Metrics Framework

### The Pirate Metrics (AARRR) for PLG

```
┌────────────────────────────────────────────────────────────┐
│  ACQUISITION  │ How do users find you?                    │
│               │ Metrics: Signups, Source attribution       │
├───────────────┼────────────────────────────────────────────┤
│  ACTIVATION   │ Do they get value quickly?                │
│               │ Metrics: Time to value, Aha moment rate   │
├───────────────┼────────────────────────────────────────────┤
│  RETENTION    │ Do they come back?                        │
│               │ Metrics: D1/D7/D30 retention, DAU/MAU     │
├───────────────┼────────────────────────────────────────────┤
│  REVENUE      │ Do they pay?                              │
│               │ Metrics: Conversion rate, ARPU, LTV       │
├───────────────┼────────────────────────────────────────────┤
│  REFERRAL     │ Do they bring others?                     │
│               │ Metrics: Viral coefficient, NPS           │
└───────────────┴────────────────────────────────────────────┘
```

### Key PLG-Specific Metrics

| Metric | Definition | Good Benchmark |
|--------|------------|----------------|
| **Time to Value (TTV)** | Signup → Aha moment | < 5 minutes |
| **Activation Rate** | % reaching aha moment | > 40% |
| **Free-to-Paid** | % converting to paid | 2-5% (B2C), 10-25% (B2B) |
| **PQL Rate** | % becoming product-qualified | > 20% of active users |
| **Expansion Revenue** | Revenue from existing customers | > 30% of new revenue |
| **Net Revenue Retention** | Revenue retained + expanded | > 100% |
| **Viral Coefficient** | New users per existing user | > 0.3 |

---

## Implementation Checklist

### Before Building Any Feature

Ask these questions:

1. **Acquisition**: Does this feature give users a reason to share?
2. **Activation**: Does this get users to value faster?
3. **Retention**: Does this create a reason to return?
4. **Revenue**: Does this create a natural upgrade path?
5. **Referral**: Does this make users look good when they share?

### PLG Feature Audit Template

```markdown
## Feature: [Name]

### Value Proposition
- What value does the user get?
- How quickly do they get it?

### Growth Loop Connection
- [ ] Acquisition: Users will share because...
- [ ] Engagement: Users return because...
- [ ] Monetization: Users upgrade because...

### Friction Analysis
- Steps to value: [count]
- Required fields: [count]
- Time to value: [estimate]
- Can this be reduced? [yes/no + how]

### Metrics
- Primary: [what success looks like]
- Secondary: [supporting metrics]
- Anti-metric: [what to watch for negative effects]
```

---

## PLG Anti-Patterns

| Anti-Pattern | Why It Fails | PLG Alternative |
|--------------|--------------|-----------------|
| Gating value behind signup | No trust established | Let them try first |
| Requiring credit card for trial | Friction kills conversion | Free tier or no-card trial |
| Complex pricing | Causes analysis paralysis | Simple tiers, clear value |
| Feature dumping | Overwhelms new users | Progressive disclosure |
| Forced tutorials | Delays value, annoys users | Contextual guidance |
| Sales-required upgrade | Kills self-serve momentum | In-product upgrade flow |
| Vanity metrics | False positive on health | Focus on value metrics |
| Dark patterns | Erodes trust, increases churn | Honest value exchange |

---

## Quick Decision Framework

When designing a feature, use this flow:

```
1. WHAT value does the user get?
   └─→ If unclear: Don't build it

2. HOW FAST do they get it?
   └─→ If > 5 min: Reduce steps

3. WHY would they return?
   └─→ If once-and-done: Add recurring value

4. WHY would they share?
   └─→ If no reason: Add social/viral element

5. WHEN would they upgrade?
   └─→ If unclear: Define the trigger

6. HOW do we measure success?
   └─→ If no metric: Define before building
```

---

## Summary

**The PLG Manifesto**:

1. **Value before capture** — Give before you ask
2. **Self-serve by default** — Remove human bottlenecks
3. **Friction is the enemy** — Measure and minimize ruthlessly
4. **Data over opinions** — Instrument everything, test everything
5. **Align incentives** — Your growth = user success
6. **Loops over funnels** — Build compounding, not linear
7. **Patience on monetization** — Retention first, revenue follows

**Remember**: PLG is not about tricks or hacks. It's about building a product so good that it sells itself. The best PLG products win because they genuinely deliver more value than they capture.
