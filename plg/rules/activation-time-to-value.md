---
title: Minimize Time to Value
impact: CRITICAL
impactDescription: determines activation rate
tags: plg, activation, friction
---

## Minimize Time to Value

Time to Value (TTV) is the duration between signup and aha moment. Measure it
in minutes, not days. Every step you add between signup and value is a leak
in your funnel.

**Incorrect (long setup before value):**

```
Signup → Email verification → Profile setup → Team creation →
Workspace config → Integration setup → Tutorial → First action

Time to value: 15-30 minutes (if they don't drop off)
```

**Correct (value first, setup later):**

```
Signup → First action (value!) → Optional profile → Optional setup

Time to value: 60 seconds
```

**Implementation checklist:**

```typescript
const activationAudit = {
  // Critical questions
  canGetValueIn60Seconds: boolean,
  isSignupFrictionMinimized: boolean, // SSO, magic links
  doesOnboardingGuideToAha: boolean, // Not feature tour
  areOptionalFieldsActualOptional: boolean,
  canSkipEmailVerification: boolean, // Verify later

  // Measure these
  metrics: {
    signupToAhaTime: "< 5 minutes",
    dropOffByStep: "identify biggest leak",
    completionRate: "> 40% reach aha moment",
  },
};
```

**Incorrect (blocking verification):**

```typescript
const SignupFlow = () => {
  const [verified, setVerified] = useState(false);

  if (!verified) {
    return <VerifyEmailScreen />; // User stuck here, no value
  }

  return <App />;
};
```

**Correct (defer verification):**

```typescript
const SignupFlow = () => {
  const { emailVerified } = useUser();

  return (
    <App>
      {/* User gets full value immediately */}
      {!emailVerified && (
        <Banner>
          Verify your email to unlock sharing features
          <VerifyButton />
        </Banner>
      )}
    </App>
  );
};
```
