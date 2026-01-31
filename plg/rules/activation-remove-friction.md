---
title: Remove Friction Ruthlessly
impact: HIGH
impactDescription: prevents drop-off at every step
tags: plg, activation, friction, signup
---

## Remove Friction Ruthlessly

Every field, every click, every decision point is friction. Defer everything
that doesn't lead directly to the aha moment. If it's not required for value,
don't require it.

**Incorrect (friction before value):**

```typescript
const SignupForm = () => (
  <Form>
    <Input name="email" required />
    <Input name="password" required />
    <Input name="confirmPassword" required />
    <Input name="firstName" required />
    <Input name="lastName" required />
    <Input name="company" required />
    <Input name="role" required />
    <Select name="teamSize" required />
    <Select name="useCase" required />
    <Checkbox name="terms" required />
    <Checkbox name="marketing" />
    <Button>Create Account</Button>
  </Form>
);

// 10 fields before any value. Most users won't finish.
```

**Correct (minimal signup, progressive profiling):**

```typescript
// Get to value fast
const SignupForm = () => (
  <Form>
    <Input name="email" required placeholder="Enter your email" />
    <Button>Get Started</Button>
    <Divider>or</Divider>
    <GoogleSignIn />
    <GitHubSignIn />
  </Form>
);

// Ask for more AFTER they've experienced value
const PostActivationProfile = () => {
  const { hasReachedAhaMoment } = useUser();

  // Don't interrupt until they love the product
  if (!hasReachedAhaMoment) return null;

  return (
    <ContextualPrompt trigger="natural_pause">
      <Text>Help us personalize your experience</Text>
      <Input name="role" placeholder="Your role (optional)" />
      <Select name="teamSize" placeholder="Team size (optional)" />
      <Row>
        <SkipButton>Maybe later</SkipButton>
        <SubmitButton>Save</SubmitButton>
      </Row>
    </ContextualPrompt>
  );
};
```

**Friction audit questions:**

| Question                                   | If No, Fix It                 |
| ------------------------------------------ | ----------------------------- |
| Can users skip email verification?         | Defer until needed            |
| Can users skip profile completion?         | Make fields optional          |
| Can users skip tutorials?                  | Let them learn by doing       |
| Can users skip team invites?               | Single-player mode first      |
| Is SSO/magic link available?               | Add passwordless options      |
| Can users try before creating an account?  | Add anonymous/guest mode      |
