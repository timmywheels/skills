---
title: Inherent Sharing Mechanics
impact: HIGH
impactDescription: compounds growth without marketing spend
tags: plg, virality, growth, sharing
---

## Inherent Sharing Mechanics

Build sharing into the core product action, not as a separate "invite friends"
feature. The best viral products require sharing to work—or become dramatically
better with others.

**Virality types:**

| Type              | Mechanism               | Example                             |
| ----------------- | ----------------------- | ----------------------------------- |
| **Inherent**      | Product requires others | Slack (messaging), Zoom (meetings)  |
| **Collaborative** | Better with others      | Figma (design), Notion (docs)       |
| **Word-of-mouth** | So good users talk      | Superhuman (email), Linear (issues) |
| **Incentivized**  | Rewards for referrals   | Dropbox (free space)                |

**Viral coefficient:**

```
K = (invites per user) × (conversion rate of invites)

K > 1  = Exponential growth (rare, powerful)
K = 0.5 = Each user brings half a user (still valuable!)
K = 0.3 = Healthy organic growth
```

**Incorrect (sharing bolted on):**

```typescript
// Sharing is an afterthought, not core to the product
const App = () => {
  return (
    <div>
      <MainProduct />
      <Footer>
        <ShareButton>Share with friends</ShareButton> {/* Ignored */}
      </Footer>
    </div>
  );
};
```

**Correct (sharing IS the product):**

```typescript
// Sharing is the core action, not a feature
const DocumentEditor = () => {
  const { doc } = useDocument();

  return (
    <Editor>
      <Toolbar>
        {/* Share is prominent because collaboration IS the value */}
        <ShareButton
          variant="primary"
          onClick={() => openShareModal(doc)}
        />
      </Toolbar>
      <Canvas doc={doc} />
      <CollaboratorAvatars /> {/* Social proof of sharing */}
    </Editor>
  );
};

// Sharing creates value for both parties
const shareDocument = async (doc: Doc, email: string) => {
  // This isn't marketing—it's the product working as designed
  await sendCollaborationInvite(doc, email);

  // Track virality (but don't optimize for vanity metrics)
  trackVirality("inherent_share", {
    docId: doc.id,
    isNewUser: await isNewUser(email),
  });
};
```

**When to invest in each type:**

| Type          | When to Build                          | When to Skip                |
| ------------- | -------------------------------------- | --------------------------- |
| Inherent      | Product is collaborative by nature     | Single-player value prop    |
| Collaborative | Value compounds with more users        | Value is individual         |
| Word-of-mouth | Product is exceptional                 | Can't be manufactured       |
| Incentivized  | After strong retention is established  | Before product-market fit   |

**Implementation for collaborative products:**

```typescript
// Make the share action frictionless
const ShareModal = ({ doc }: { doc: Doc }) => {
  return (
    <Modal>
      {/* One-click sharing */}
      <CopyLinkButton link={doc.shareableLink} />

      {/* Email invite with pre-filled message */}
      <EmailInvite
        defaultMessage={`Check out this ${doc.type} I'm working on`}
        onSend={(email) => shareDocument(doc, email)}
      />

      {/* Show who already has access (social proof) */}
      <CollaboratorList collaborators={doc.collaborators} />
    </Modal>
  );
};
```
