---
name: plg
description:
  Build products that sell themselves. Use when designing onboarding, pricing,
  conversion flows, viral mechanics, or retention strategies. Triggers on tasks
  involving user activation, freemium models, self-serve upgrades, or growth
  loops.
license: MIT
metadata:
  author: timmywheels
  version: "1.0.0"
---

# Product-Led Growth (PLG)

Patterns for building products where the product itself drives acquisition,
conversion, and expansion. PLG flips the traditional funnel: users experience
value BEFORE you ask for anything. These patterns help teams design
self-serve experiences that compound growth without scaling sales headcount.

## When to Apply

Reference these guidelines when:

- Designing user onboarding or activation flows
- Building freemium or free trial experiences
- Creating self-serve upgrade paths
- Adding viral or referral mechanics
- Optimizing conversion funnels
- Defining product-qualified leads (PQLs)
- Setting up retention and engagement loops

## Rule Categories by Priority

| Priority | Category   | Impact   | Prefix        |
| -------- | ---------- | -------- | ------------- |
| 1        | Activation | CRITICAL | `activation-` |
| 2        | Onboarding | HIGH     | `onboarding-` |
| 3        | Conversion | HIGH     | `conversion-` |
| 4        | Virality   | MEDIUM   | `virality-`   |
| 5        | Expansion  | MEDIUM   | `expansion-`  |
| 6        | Retention  | MEDIUM   | `retention-`  |

## Quick Reference

### 1. Activation (CRITICAL)

- `activation-aha-moment` - Identify and optimize the single moment users first experience core value
- `activation-time-to-value` - Minimize steps between signup and value; measure in minutes, not days
- `activation-remove-friction` - Defer everything that doesn't lead directly to the aha moment

### 2. Onboarding (HIGH)

- `onboarding-progressive-value` - Each step must deliver immediate value, not just collect data
- `onboarding-skip-everything-skippable` - Only require what's essential; make the rest optional
- `onboarding-teach-by-doing` - Guide users through real actions, not feature tours

### 3. Conversion (HIGH)

- `conversion-pql-over-mql` - Trigger upgrades from usage signals, not form fills
- `conversion-self-serve-upgrade` - Users upgrade themselves at moment of realized need
- `conversion-in-context-prompts` - Show upgrade CTAs where users hit limits, not random banners

### 4. Virality (MEDIUM)

- `virality-inherent-sharing` - Build sharing into the core product action
- `virality-incentivize-after-activation` - Only ask for referrals after users love the product
- `virality-both-sides-win` - Reward both referrer and referee equally

### 5. Expansion (MEDIUM)

- `expansion-usage-based-pricing` - Revenue grows as user value grows
- `expansion-proactive-tier-suggestions` - Surface upgrades before users hit hard limits
- `expansion-seat-based-growth` - Make adding team members frictionless

### 6. Retention (MEDIUM)

- `retention-habit-loops` - Design cue → routine → reward cycles that create cravings
- `retention-variable-rewards` - Vary the reward to maintain engagement
- `retention-notification-value` - Every notification must deliver value, not just ask for attention

## How to Use

Read individual rule files for detailed explanations and code examples:

```
rules/activation-aha-moment.md
rules/conversion-self-serve-upgrade.md
```

Each rule file contains:

- Brief explanation of why it matters
- Anti-patterns to avoid
- Correct implementation pattern
- Code examples where applicable

## Full Compiled Document

For the complete guide with all rules expanded: `AGENTS.md`
