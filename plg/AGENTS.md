# Product-Led Growth (PLG)

**Version 1.0.0**
Growth Engineering
January 2026

> **Note:**
> This document is for agents and LLMs to follow when designing, building,
> or reviewing product experiences focused on self-serve growth. Humans
> may also find it useful, but guidance here is optimized for automation
> and consistency by AI-assisted workflows.

---

## Abstract

Product-Led Growth patterns for building products where the product itself
drives acquisition, conversion, and expansion. PLG flips the traditional
funnel: users experience value BEFORE you ask for anything. The core
principle: **deliver value first, capture value second.**

---

## Table of Contents

1. [Activation](#1-activation) — **CRITICAL**
   - 1.1 [Identify the Aha Moment](#11-identify-the-aha-moment)
   - 1.2 [Minimize Time to Value](#12-minimize-time-to-value)
   - 1.3 [Remove Friction Ruthlessly](#13-remove-friction-ruthlessly)
2. [Onboarding](#2-onboarding) — **HIGH**
   - 2.1 [Deliver Progressive Value](#21-deliver-progressive-value)
   - 2.2 [Skip Everything Skippable](#22-skip-everything-skippable)
   - 2.3 [Teach by Doing](#23-teach-by-doing)
3. [Conversion](#3-conversion) — **HIGH**
   - 3.1 [PQLs Over MQLs](#31-pqls-over-mqls)
   - 3.2 [Self-Serve Upgrade Paths](#32-self-serve-upgrade-paths)
   - 3.3 [In-Context Upgrade Prompts](#33-in-context-upgrade-prompts)
4. [Virality](#4-virality) — **MEDIUM**
   - 4.1 [Inherent Sharing Mechanics](#41-inherent-sharing-mechanics)
   - 4.2 [Incentivize After Activation](#42-incentivize-after-activation)
   - 4.3 [Both Sides Win](#43-both-sides-win)
5. [Expansion](#5-expansion) — **MEDIUM**
   - 5.1 [Usage-Based Pricing](#51-usage-based-pricing)
   - 5.2 [Proactive Tier Suggestions](#52-proactive-tier-suggestions)
   - 5.3 [Seat-Based Growth](#53-seat-based-growth)
6. [Retention](#6-retention) — **MEDIUM**
   - 6.1 [Habit Loops](#61-habit-loops)
   - 6.2 [Variable Rewards](#62-variable-rewards)
   - 6.3 [Notification Value](#63-notification-value)
7. [Metrics Framework](#7-metrics-framework)
8. [Anti-Pattern Reference](#8-anti-pattern-reference)

---

## The PLG Mental Model

```
Traditional Sales-Led:  Marketing → Sales → Value → Payment
Product-Led:            Value → Aha Moment → Habit → Payment → Expansion

PLG flips the funnel: users experience value BEFORE you ask for anything.
```

### Why PLG Works

| Factor            | Sales-Led          | Product-Led           |
| ----------------- | ------------------ | --------------------- |
| CAC               | High (sales team)  | Low (self-serve)      |
| Scalability       | Linear (headcount) | Exponential (product) |
| User alignment    | Company goals      | User goals            |
| Conversion signal | Demo completed     | Value received        |

### The PLG Growth Loops

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

## 1. Activation

**Impact: CRITICAL**

Activation is the moment a user first experiences your product's core value.
Get this wrong and nothing else matters.

### 1.1 Identify the Aha Moment

**Impact: CRITICAL (foundation for all growth)**

The aha moment is the single action where users first experience core value.
Everything in your product should drive toward this moment.

**Examples of aha moments:**

- Slack: Send first message in a channel
- Dropbox: First file synced across devices
- Figma: First shape drawn with a collaborator watching
- Calendly: First meeting booked through a shared link

**Incorrect: vague or multi-step aha moment**

```
"Users understand the value when they've explored the dashboard,
set up their profile, and invited team members."
```

**Correct: specific, measurable action**

```
"Users experience value when they send their first message
and receive a reply within the same session."
```

**Implementation:**

```typescript
// Track the specific aha moment action
const trackAhaMoment = async (userId: string, action: AhaAction) => {
  await analytics.track("aha_moment_reached", {
    userId,
    action,
    timeFromSignup: Date.now() - user.createdAt,
  });

  // Trigger celebration and next step
  await showCelebration(userId);
  await suggestNextValueAction(userId);
};
```

### 1.2 Minimize Time to Value

**Impact: CRITICAL (determines activation rate)**

Time to Value (TTV) is the duration between signup and aha moment. Measure it
in minutes, not days. Every step you add between signup and value is a leak
in your funnel.

**Incorrect: long setup before value**

```
Signup → Email verification → Profile setup → Team creation →
Workspace config → Integration setup → Tutorial → First action
```

**Correct: value first, setup later**

```
Signup → First action (value!) → Optional profile → Optional setup
```

**Implementation checklist:**

- [ ] Can a user get value in under 60 seconds?
- [ ] Is signup friction minimized (SSO, magic links)?
- [ ] Does onboarding guide toward aha moment, not feature tour?
- [ ] Are optional fields actually optional?
- [ ] Can users skip email verification initially?

### 1.3 Remove Friction Ruthlessly

**Impact: HIGH (prevents drop-off)**

Every field, every click, every decision point is friction. Defer everything
that doesn't lead directly to the aha moment.

**Incorrect: friction before value**

```typescript
// Requiring info before any value
const SignupFlow = () => {
  return (
    <Form>
      <Input name="email" required />
      <Input name="password" required />
      <Input name="company" required />
      <Input name="role" required />
      <Input name="teamSize" required />
      <Input name="useCase" required />
      <Checkbox name="terms" required />
      <Checkbox name="marketing" />
      <Button>Create Account</Button>
    </Form>
  );
};
```

**Correct: minimal signup, progressive profiling**

```typescript
// Get to value fast, collect info later
const SignupFlow = () => {
  return (
    <Form>
      <Input name="email" required />
      <Button>Get Started</Button>
      <Divider>or</Divider>
      <GoogleSignIn />
    </Form>
  );
};

// Ask for more info AFTER they've experienced value
const PostActivationProfile = () => {
  const { hasReachedAhaMoment } = useUser();

  if (!hasReachedAhaMoment) return null;

  return (
    <ContextualPrompt>
      <Text>Help us personalize your experience</Text>
      <Input name="role" />
      <Input name="teamSize" />
      <SkipButton />
    </ContextualPrompt>
  );
};
```

---

## 2. Onboarding

**Impact: HIGH**

Onboarding bridges signup to activation. Each step must deliver immediate
value, not just collect data or show features.

### 2.1 Deliver Progressive Value

**Impact: HIGH (keeps users moving forward)**

Every onboarding step must answer: "What value does the user get NOW?" If a
step doesn't deliver value, remove it or defer it.

**Incorrect: data collection disguised as onboarding**

```typescript
const OnboardingSteps = [
  { step: "Tell us about yourself", value: null }, // No value
  { step: "Invite your team", value: null }, // No value yet
  { step: "Connect integrations", value: null }, // No value yet
  { step: "Watch tutorial", value: null }, // No value
  { step: "Start using product", value: "finally!" },
];
```

**Correct: value at each step**

```typescript
const OnboardingSteps = [
  {
    step: "Create your first project",
    value: "See your idea take shape",
  },
  {
    step: "Add a task",
    value: "Experience the workflow",
  },
  {
    step: "Invite a teammate (optional)",
    value: "See real-time collaboration",
  },
];
```

**Implementation pattern:**

```typescript
type OnboardingStep = {
  id: string;
  isRequired: boolean;
  isComplete: (user: User) => boolean;
  valueDelivered: string; // What user gains from this step
  skipPenalty: "none" | "degraded" | "blocking";
};

// Every step must have a clear valueDelivered
const steps: OnboardingStep[] = [
  {
    id: "first-project",
    isRequired: true,
    isComplete: (user) => user.projects.length > 0,
    valueDelivered: "Your first project is ready to use",
    skipPenalty: "blocking", // Can't skip - it IS the value
  },
  {
    id: "invite-team",
    isRequired: false,
    isComplete: (user) => user.invitesSent > 0,
    valueDelivered: "Collaborate in real-time",
    skipPenalty: "degraded", // Product works, but less valuable
  },
];
```

### 2.2 Skip Everything Skippable

**Impact: MEDIUM (reduces drop-off)**

If a step isn't required for the user to get value, make it skippable. Better
yet, defer it until after activation.

**Onboarding types by context:**

| Context        | Pattern                    | Example                          |
| -------------- | -------------------------- | -------------------------------- |
| B2C Simple     | Empty state → First action | Twitter: "Follow 5 people"       |
| B2C Complex    | Progressive disclosure     | Duolingo: 1 lesson, then profile |
| B2B Individual | Template/example data      | Notion: Pre-made templates       |
| B2B Team       | Invite + collaborate       | Figma: Share link, edit together |

### 2.3 Teach by Doing

**Impact: MEDIUM (improves retention of learning)**

Guide users through real actions, not passive feature tours. The best
onboarding is invisible—users learn by accomplishing their actual goals.

**Incorrect: passive feature tour**

```typescript
const FeatureTour = () => {
  return (
    <Modal>
      <Carousel>
        <Slide>Here's the dashboard...</Slide>
        <Slide>Click here to create...</Slide>
        <Slide>This button does...</Slide>
        <Slide>You can also...</Slide>
      </Carousel>
      <Button>Got it!</Button>
    </Modal>
  );
};
```

**Correct: guided action**

```typescript
const GuidedOnboarding = () => {
  const { currentStep } = useOnboarding();

  return (
    <Tooltip target={currentStep.target} position="bottom">
      <Text>{currentStep.instruction}</Text>
      {/* User completes real action, tooltip advances */}
    </Tooltip>
  );
};

// Steps are real actions, not passive viewing
const onboardingSteps = [
  { target: "#new-project-btn", instruction: "Create your first project" },
  { target: "#task-input", instruction: "Add your first task" },
  { target: "#share-btn", instruction: "Share with a teammate" },
];
```

---

## 3. Conversion

**Impact: HIGH**

Conversion in PLG is self-serve: users upgrade themselves at the moment of
realized need, not when sales calls them.

### 3.1 PQLs Over MQLs

**Impact: HIGH (5-10x better conversion rates)**

Product-Qualified Leads (PQLs) convert far better than Marketing-Qualified
Leads (MQLs) because they've already experienced value.

```
Traditional: MQL (filled form) → Sales call → Maybe value
PLG:         Usage signals → PQL (experienced value) → Self-serve upgrade

PQLs convert 5-10x better than MQLs because they've already received value.
```

**PQL signals to track:**

```typescript
const pqlSignals = {
  // Usage-based signals
  usageThreshold: {
    signal: "api_calls >= 800", // 80% of free limit
    score: 30,
  },
  featureAdoption: {
    signal: "used >= 3 premium features",
    score: 25,
  },

  // Team signals
  teamGrowth: {
    signal: "invited >= 3 teammates",
    score: 25,
  },
  collaboration: {
    signal: "shared >= 5 resources",
    score: 15,
  },

  // Engagement signals
  retention: {
    signal: "active 4+ days in past week",
    score: 20,
  },
  depth: {
    signal: "completed core workflow >= 5 times",
    score: 20,
  },
};

// PQL when score >= 60
const isPQL = (user: User) => calculatePQLScore(user) >= 60;
```

### 3.2 Self-Serve Upgrade Paths

**Impact: HIGH (removes sales bottleneck)**

The user should upgrade themselves at the moment of realized need. No sales
call required for obvious use cases.

**Conversion triggers:**

| Trigger Type        | Signal                  | Response                  |
| ------------------- | ----------------------- | ------------------------- |
| **Usage limit**     | Hit free tier cap       | In-context upgrade prompt |
| **Feature gate**    | Tried premium feature   | "Unlock this" CTA         |
| **Team growth**     | Invited 3+ people       | Team plan suggestion      |
| **Time-based**      | Active for 14 days      | "You've done X, Y, Z"     |
| **Value milestone** | Completed core workflow | "Get more with Pro"       |

**Implementation:**

```typescript
const conversionTriggers = {
  usageLimit: {
    condition: (user) => user.apiCalls >= FREE_TIER_LIMIT,
    message: "You've made 1000 API calls this month. Upgrade for unlimited.",
    placement: "in-context", // Show where they hit the limit
    urgency: "soft", // Don't block, just inform
  },

  featureGate: {
    condition: (action) => PREMIUM_FEATURES.includes(action),
    message: "Advanced analytics is a Pro feature. See what you'd get:",
    placement: "inline-preview", // Show grayed-out version
    urgency: "none", // Let them discover value, not feel blocked
  },

  teamGrowth: {
    condition: (user) => user.invitesSent >= 3,
    message: "Collaborating with your team? Team plans include...",
    placement: "contextual-banner",
    urgency: "soft",
  },
};
```

### 3.3 In-Context Upgrade Prompts

**Impact: MEDIUM (higher conversion than generic prompts)**

Show upgrade CTAs where users hit limits, not in random banners or emails.
Context is everything.

**Incorrect: generic upgrade banner**

```typescript
// Random banner shown everywhere
const UpgradeBanner = () => (
  <Banner>
    <Text>Upgrade to Pro for more features!</Text>
    <Button>Learn More</Button>
  </Banner>
);
```

**Correct: contextual upgrade at point of friction**

```typescript
// Shown exactly when user hits the limit
const UsageLimitPrompt = ({ current, limit, feature }) => {
  if (current < limit * 0.8) return null;

  return (
    <InlinePrompt>
      <Text>
        You've used {current} of {limit} {feature} this month.
      </Text>
      <Text>Upgrade for unlimited {feature}.</Text>
      <Button>See Plans</Button>
      <DismissButton />
    </InlinePrompt>
  );
};

// Show premium feature preview, not just a lock icon
const PremiumFeatureGate = ({ feature, children }) => {
  const { isPremium } = useUser();

  if (isPremium) return children;

  return (
    <BlurredPreview>
      {children}
      <Overlay>
        <Text>This is a Pro feature</Text>
        <Text>See what you'd get:</Text>
        <FeaturePreview feature={feature} />
        <Button>Unlock with Pro</Button>
      </Overlay>
    </BlurredPreview>
  );
};
```

**Pricing page best practices:**

- Lead with value, not features
- Social proof (logos, testimonials) above the fold
- Clear "most popular" or "recommended" tier
- Annual discount prominently shown
- Free tier should feel generous, not crippled

---

## 4. Virality

**Impact: MEDIUM**

Virality means features that naturally cause users to bring in other users.
Build it into the product, don't bolt it on.

### 4.1 Inherent Sharing Mechanics

**Impact: HIGH (compounds growth)**

Build sharing into the core product action, not as a separate feature.

**Virality types:**

| Type              | Mechanism               | Example                             |
| ----------------- | ----------------------- | ----------------------------------- |
| **Inherent**      | Product requires others | Slack (messaging), Zoom (meetings)  |
| **Collaborative** | Better with others      | Figma (design), Notion (docs)       |
| **Word-of-mouth** | So good users talk      | Superhuman (email), Linear (issues) |
| **Incentivized**  | Rewards for referrals   | Dropbox (free space)                |

**Viral coefficient formula:**

```
K = (invites per user) × (conversion rate of invites)

K > 1 = Exponential growth
K = 0.5 = Each user brings half a user (still valuable!)
```

**Implementation:**

```typescript
// Inherent virality: Sharing IS the product
const shareDocument = async (doc: Doc, email: string) => {
  await sendCollaborationInvite(doc, email);
  trackVirality("inherent_share", { docId: doc.id });
};

// The share action is a core product action, not a marketing feature
```

### 4.2 Incentivize After Activation

**Impact: MEDIUM (incentives can't fix churn)**

Only ask for referrals after users love the product. Incentives amplify
existing satisfaction—they can't create it.

**Incorrect: asking too early**

```typescript
// Immediately after signup
const WelcomeScreen = () => (
  <Modal>
    <Text>Welcome! Invite friends for free credits!</Text>
    <ReferralLink />
  </Modal>
);
```

**Correct: asking after activation**

```typescript
const ReferralPrompt = () => {
  const { hasReachedAhaMoment, successfulActions } = useUser();

  // Only show after user has experienced value
  if (!hasReachedAhaMoment || successfulActions < 5) return null;

  return (
    <ContextualPrompt trigger="after_success">
      <Text>Enjoying the product?</Text>
      <Text>Give a friend 1 month free (you get one too)</Text>
      <ReferralLink />
    </ContextualPrompt>
  );
};
```

### 4.3 Both Sides Win

**Impact: MEDIUM (increases invite acceptance)**

Reward both referrer and referee equally. One-sided incentives feel
extractive.

**Incorrect: one-sided referral**

```typescript
const referralProgram = {
  referrer: "$50 credit",
  referee: "nothing", // They just signed up normally
};
```

**Correct: mutual benefit**

```typescript
const referralProgram = {
  trigger: "user_activated", // Only ask after they love it
  offer: {
    referrer: "1 month free",
    referee: "1 month free", // Both sides win
  },
  friction: "one_click", // Pre-filled message, easy share
};
```

---

## 5. Expansion

**Impact: MEDIUM**

Expansion means revenue grows as user value grows. Align your success with
theirs.

### 5.1 Usage-Based Pricing

**Impact: HIGH (enables land-and-expand)**

Revenue should correlate with value delivered. As users get more value, they
pay more—and they're happy to because they're getting more.

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

### 5.2 Proactive Tier Suggestions

**Impact: MEDIUM (prevents hard limit frustration)**

Surface upgrade suggestions before users hit hard limits. Proactive is better
than reactive.

**Implementation:**

```typescript
const expansionSignals = {
  usageGrowth: {
    signal: "approaching_tier_limit",
    threshold: 0.8, // 80% of limit
    action: "proactive_upgrade_offer",
    message: "You're at 80% of your plan limit. Upgrade to avoid interruption.",
  },

  featureAdoption: {
    signal: "premium_feature_attempted",
    threshold: 3, // Tried 3 premium features
    action: "personalized_upgrade_pitch",
    message: "You've been exploring Pro features. Here's what you'd unlock.",
  },
};
```

### 5.3 Seat-Based Growth

**Impact: MEDIUM (expands within accounts)**

Make adding team members frictionless. The easier it is to add seats, the
more seats get added.

**Implementation:**

```typescript
const teamGrowthSignals = {
  teamGrowth: {
    signal: "team_member_invited",
    threshold: 5,
    action: "suggest_team_plan",
    message: "Your team is growing! Team plans include admin controls and...",
  },
};
```

---

## 6. Retention

**Impact: MEDIUM**

Retention requires recurring value. One-time value = churn. Design habit
loops that bring users back.

### 6.1 Habit Loops

**Impact: HIGH (drives organic return)**

Design cue → routine → reward cycles that create cravings.

**The habit loop:**

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│   CUE   │ ──→ │ ROUTINE │ ──→ │ REWARD  │ ──→ │ CRAVING │
│(trigger)│     │ (action)│     │ (value) │     │ (repeat)│
└─────────┘     └─────────┘     └─────────┘     └─────────┘
     ↑                                               │
     └───────────────────────────────────────────────┘
```

**Implementation:**

```typescript
const habitLoop = {
  cue: {
    type: "notification",
    trigger: "daily_digest",
    message: "3 teammates updated projects you follow",
    timing: "user_active_hours", // Learn when they're receptive
  },

  routine: {
    action: "view_updates",
    friction: "one_click", // Deep link directly to content
  },

  reward: {
    type: "variable", // Variable rewards are more engaging
    content: "personalized_updates", // Different every time
    social: "see_team_activity", // Human connection
  },

  metrics: {
    cue_to_routine: "notification_click_rate",
    routine_to_reward: "session_depth",
    reward_to_craving: "organic_return_rate",
  },
};
```

### 6.2 Variable Rewards

**Impact: MEDIUM (maintains engagement)**

Vary the reward to maintain engagement. Predictable rewards get boring;
variable rewards create anticipation.

**Retention mechanics:**

| Mechanic          | How It Works                  | Example              |
| ----------------- | ----------------------------- | -------------------- |
| **Streaks**       | Loss aversion drives return   | Duolingo, GitHub     |
| **Notifications** | External trigger to return    | Slack, Email digests |
| **Social**        | Others' activity creates FOMO | Twitter, LinkedIn    |
| **Data lock-in**  | Your data becomes valuable    | Notion, Airtable     |
| **Workflow**      | Part of daily process         | Linear, Superhuman   |

### 6.3 Notification Value

**Impact: MEDIUM (prevents notification fatigue)**

Every notification must deliver value, not just ask for attention.
Notifications that don't deliver value train users to ignore you.

**Incorrect: attention-seeking notifications**

```typescript
const badNotifications = [
  "You haven't logged in for 3 days!", // Guilt trip
  "Check out our new feature!", // Marketing, not value
  "Your friends are using the app!", // Vague, no value
];
```

**Correct: value-delivering notifications**

```typescript
const goodNotifications = [
  "Sarah commented on your project: 'Love this approach!'", // Specific value
  "Your report is ready: Revenue up 12% this month", // Delivers insight
  "3 tasks completed by your team today", // Actionable summary
];
```

---

## 7. Metrics Framework

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

| Metric                    | Definition                      | Good Benchmark           |
| ------------------------- | ------------------------------- | ------------------------ |
| **Time to Value (TTV)**   | Signup → Aha moment             | < 5 minutes              |
| **Activation Rate**       | % reaching aha moment           | > 40%                    |
| **Free-to-Paid**          | % converting to paid            | 2-5% (B2C), 10-25% (B2B) |
| **PQL Rate**              | % becoming product-qualified    | > 20% of active users    |
| **Expansion Revenue**     | Revenue from existing customers | > 30% of new revenue     |
| **Net Revenue Retention** | Revenue retained + expanded     | > 100%                   |
| **Viral Coefficient**     | New users per existing user     | > 0.3                    |

---

## 8. Anti-Pattern Reference

| Anti-Pattern                    | Why It Fails                  | PLG Alternative            |
| ------------------------------- | ----------------------------- | -------------------------- |
| Gating value behind signup      | No trust established          | Let them try first         |
| Requiring credit card for trial | Friction kills conversion     | Free tier or no-card trial |
| Complex pricing                 | Causes analysis paralysis     | Simple tiers, clear value  |
| Feature dumping                 | Overwhelms new users          | Progressive disclosure     |
| Forced tutorials                | Delays value, annoys users    | Contextual guidance        |
| Sales-required upgrade          | Kills self-serve momentum     | In-product upgrade flow    |
| Vanity metrics                  | False positive on health      | Focus on value metrics     |
| Dark patterns                   | Erodes trust, increases churn | Honest value exchange      |
| Notifications without value     | Trains users to ignore you    | Every ping delivers value  |
| Incentives before activation    | Can't fix bad product         | Amplify existing love      |

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

**The PLG Manifesto:**

1. **Value before capture** — Give before you ask
2. **Self-serve by default** — Remove human bottlenecks
3. **Friction is the enemy** — Measure and minimize ruthlessly
4. **Data over opinions** — Instrument everything, test everything
5. **Align incentives** — Your growth = user success
6. **Loops over funnels** — Build compounding, not linear
7. **Patience on monetization** — Retention first, revenue follows

**Remember:** PLG is not about tricks or hacks. It's about building a product
so good that it sells itself. The best PLG products win because they genuinely
deliver more value than they capture.

---

## References

1. [Product-Led Growth Foundations](https://www.productled.org/foundations/what-is-product-led-growth)
2. [OpenView PLG Resources](https://openviewpartners.com/product-led-growth/)
3. [Reforge Growth Loops](https://www.reforge.com/blog/growth-loops)
