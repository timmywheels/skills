---
title: Deliver Progressive Value
impact: HIGH
impactDescription: keeps users moving through the funnel
tags: plg, onboarding, activation
---

## Deliver Progressive Value

Every onboarding step must answer: "What value does the user get NOW?" If a
step doesn't deliver value, remove it or defer it. Onboarding is not data
collection—it's value delivery.

**Incorrect (data collection disguised as onboarding):**

```typescript
const OnboardingFlow = () => {
  const steps = [
    { name: "Tell us about yourself", value: null },
    { name: "Invite your team", value: null },
    { name: "Connect integrations", value: null },
    { name: "Watch a 5-minute tutorial", value: null },
    { name: "Set up your workspace", value: null },
    { name: "Finally use the product", value: "Here's the value!" },
  ];

  // 5 steps of friction before any value
  // Most users drop off before step 6
};
```

**Correct (value at each step):**

```typescript
type OnboardingStep = {
  id: string;
  action: string;
  valueDelivered: string; // REQUIRED: what user gains
  isRequired: boolean;
  skipPenalty: "none" | "degraded" | "blocking";
};

const onboardingSteps: OnboardingStep[] = [
  {
    id: "first-project",
    action: "Create your first project",
    valueDelivered: "See your idea take shape immediately",
    isRequired: true,
    skipPenalty: "blocking", // This IS the value
  },
  {
    id: "first-task",
    action: "Add your first task",
    valueDelivered: "Experience the workflow you'll use daily",
    isRequired: true,
    skipPenalty: "blocking",
  },
  {
    id: "invite-teammate",
    action: "Invite a teammate",
    valueDelivered: "See real-time collaboration in action",
    isRequired: false,
    skipPenalty: "degraded", // Works alone, better together
  },
  {
    id: "connect-calendar",
    action: "Connect your calendar",
    valueDelivered: "Automatic deadline syncing",
    isRequired: false,
    skipPenalty: "none", // Nice to have
  },
];
```

**Onboarding patterns by context:**

| Context        | Pattern                    | Example                          |
| -------------- | -------------------------- | -------------------------------- |
| B2C Simple     | Empty state → First action | Twitter: "Follow 5 people"       |
| B2C Complex    | Progressive disclosure     | Duolingo: 1 lesson, then profile |
| B2B Individual | Template/example data      | Notion: Pre-made templates       |
| B2B Team       | Invite + collaborate       | Figma: Share link, edit together |

**Implementation:**

```typescript
const OnboardingStep = ({ step }: { step: OnboardingStep }) => {
  const { markComplete, skip } = useOnboarding();

  return (
    <Card>
      <Heading>{step.action}</Heading>
      <Text color="muted">{step.valueDelivered}</Text>

      <StepContent step={step} onComplete={() => markComplete(step.id)} />

      {!step.isRequired && (
        <SkipButton onClick={() => skip(step.id)}>
          Skip for now
        </SkipButton>
      )}
    </Card>
  );
};
```
