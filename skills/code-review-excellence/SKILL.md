---
name: code-review-excellence
description: Give or receive high-quality code reviews — use when someone asks how to review a PR, write review comments, respond to reviewer feedback, structure a PR, or improve code review culture.
---

# Code Review Excellence — Giving and Receiving

## The Purpose of Code Review

Code review is **not** a bug-finding mechanism — that is what tests, linters, and type checkers are for. Code review is:

- **Knowledge sharing**: the reviewer learns what changed; the author learns patterns they missed
- **Consistency**: the team's collective taste applied before code merges
- **Design feedback**: a second opinion on approach before it becomes load-bearing
- **Mentoring**: junior engineers grow through specific, reasoned feedback

What code review is NOT:
- A gatekeeping ritual where senior engineers block junior engineers
- A place to re-argue architecture decisions that were already made
- A style guide enforcement mechanism (automate that with linters)
- The reviewer's opportunity to rewrite the PR in their preferred style

The reviewer is not the author's boss. It is a **collaborative dialogue** between peers with shared ownership of the codebase.

---

## What to Review — Priority Order

### 1. Design (Highest Priority)
- Does this solve the right problem?
- Is the approach sound, or does it introduce hidden complexity?
- Does it fit the existing architecture, or does it introduce inconsistency?
- Are there simpler alternatives that achieve the same goal?
- Does it create tight coupling that will be painful to change later?

Questions to ask yourself as a reviewer:
- "Would I be comfortable maintaining this in a year?"
- "Is there an existing abstraction this should reuse?"
- "Will this approach scale if usage doubles?"

### 2. Functionality
- Does the code do what the PR description says it does?
- Are there edge cases the author hasn't handled?
- Does it handle null/empty/zero/negative inputs?
- Are error cases handled and surfaced appropriately?

### 3. Complexity
- Is it more complex than it needs to be?
- Could a new team member understand this in 5 minutes?
- Are there deeply nested conditionals that could be flattened?
- Is there logic that would be clearer as a named function?

**Cyclomatic complexity > 10 in a single function is a warning sign.** Not a hard rule, but worth examining.

### 4. Tests
- Are there tests? If not, why not?
- Do tests cover the important cases — not just the happy path?
- Are tests testing **behavior** or **implementation**?
  - Good: `assertThat(order.total()).isEqualTo(99.99)` (behavior)
  - Bad: `verify(priceCalculator).calculate(items)` (implementation detail)
- Will the tests catch regressions, or will they pass even if the code is broken?

### 5. Naming
- Are variable, method, and class names accurate and clear?
- Does the name reveal intent, or just describe type? (`userList` vs `activeAdmins`)
- Are abbreviations avoided unless universally understood?

### 6. Comments
- Are comments explaining **why**, not **what**? (The code already says what.)
- Is there commented-out code? (Delete it; git remembers it.)
- Are there TODO comments without a ticket reference? (Link to an issue.)
- Is there a comment explaining a non-obvious decision or workaround?

Good comment:
```java
// Using INSERT ... ON CONFLICT instead of a read-then-write here to avoid a race
// condition under concurrent requests. See issue #4521.
```

Bad comment:
```java
// loop through items
for (Item item : items) {
```

### 7. Style (Lowest Priority — Automate It)
- If your team has a linter, formatter, or style guide, configure it in CI and stop reviewing style manually
- Do NOT block a PR because of formatting; it wastes both parties' time
- Reserve "style" comments for things linters can't catch: file organization, package structure

---

## Giving Reviews — The Craft

### Be Specific

Vague: "This is too complex."
Specific: "This method is doing two things — parsing the JSON payload and validating business rules. Consider splitting it into `parseRequest()` and `validateOrder()` so each has a single responsibility and can be tested independently."

### Critique the Code, Not the Author

Avoid: "You named this variable poorly."
Better: "The name `data` doesn't convey what this contains — `pendingOrders` or `orderQueue` would make the intent clear at a glance."

### Offer Alternatives

Don't just point out a problem — suggest a path forward:

"Rather than catching `Exception` here and logging it, consider letting the exception propagate to the caller and handling it at the controller layer where you can return the appropriate HTTP status code. That way the service layer stays clean."

### Distinguish Blocking From Non-Blocking

Use a prefix so the author knows what must be addressed before merge:

- `nit:` — non-blocking style/preference suggestion; fix if you agree, ignore if you disagree
- `optional:` — something worth considering but not required for this PR
- `blocking:` — must be resolved before merge (or no prefix, which implicitly means blocking)
- `question:` — genuine question; author should answer but may not need to change code

Examples:
```
nit: s/userList/activeAdmins/ — more descriptive

question: why are we using a List here instead of a Set? Is order significant?

blocking: this endpoint does not validate the JWT token before reading the user's data —
          any unauthenticated request can access this resource.
```

### Ask Questions Before Making Statements

If something looks wrong but you're not sure you understand the intent:

Less effective: "This approach is wrong."
Better: "I'm not sure I follow the logic in lines 42-48 — specifically, why does the loop decrement `i` inside the condition? Could you add a comment or walk me through the intent?"

### Approve With Comments

Don't block a PR for nits. Use "Approve + comment" for non-blocking feedback. The author can decide whether to incorporate it now or in a follow-up.

Blocking a PR on style preferences erodes trust and slows delivery with no quality benefit.

### Respond Within 24 Business Hours

Stale PRs accumulate merge conflicts, lose context, and sit in the back of the author's mind as cognitive overhead. If you can't review fully within 24h, send a brief acknowledgment:

"I'll look at this by end of day tomorrow — the DB migration section will need careful review."

---

## Receiving Reviews — The Craft

### Don't Take It Personally

The reviewer is critiquing code written by a past version of you, under constraints you may not have explained. Separate your identity from the code.

### Respond to Every Comment

Even a brief response closes the loop:
- "Done — moved to the service layer."
- "Good catch, fixed in latest commit."
- "I disagree with this one — see below."

Do NOT silently close comments by pushing a new commit. The reviewer doesn't know if you addressed it, forgot it, or rejected it.

### Document What Changed

When you push a follow-up commit, reference which comments it addresses:

"Pushed a new commit addressing the feedback on exception handling (#comment-42) and the naming suggestions (#comment-51). The refactoring suggestion (#comment-38) I'll defer to a follow-up since it's a larger change."

### Push Back Respectfully

Disagreement is healthy. State your position with reasoning:

"I prefer keeping the parsing and validation together here because they share the same error handling context — splitting them would require passing the same `ParsedRequest` object through multiple layers. Happy to discuss if you see a cleaner way to structure this."

The reviewer may agree, disagree, or suggest a third approach — all of those are valuable outcomes.

### Learn From Patterns

If the same type of comment appears across multiple reviews, update your mental model. Repeated feedback on the same issue usually signals a gap in your understanding of the team's standards — ask the reviewer to explain the principle, not just the fix.

---

## PR Size and Structure

### Size Sweet Spot: Under 400 Lines Changed

Research consistently shows review quality degrades above 400 lines. Reviewers stop reading carefully, miss subtle bugs, and spend more time on trivial comments.

```
< 200 lines:  ideal; reviewer can hold the entire change in working memory
200-400 lines: acceptable; may need 2 review sessions
400-800 lines: review quality degrades significantly; split if possible
> 800 lines:  reviewers will rubber-stamp; nearly always should be split
```

### Single Responsibility

One PR = one change. Never mix:
- Refactoring + new feature (reviewer can't tell which lines are intentional changes)
- Bug fix + cleanup (if the bug fix is reverted, cleanup goes with it)
- Multiple unrelated features (delays everything if one needs rework)

Strategy for large features: feature flags let you merge incomplete features in small PRs without exposing them to users.

### PR Description Quality

A good PR description answers:
1. **What changed** — the `git log` can say what, the description says it in plain English
2. **Why it changed** — what problem does this solve? what would happen without this change?
3. **How to test** — how can the reviewer verify it works?
4. **Screenshots** (if UI change) — before and after

Template:
```markdown
## What
Brief description of the change.

## Why
The problem this solves, or the requirement driving it. Link to issue/ticket.

## How to Test
Step-by-step instructions for verifying the change works as expected.

## Notes
Any non-obvious decisions, known trade-offs, or follow-up items.
```

### Self-Review First

Read your own diff before requesting review. You will catch:
- Debug logging left in
- Commented-out code you forgot to delete
- Obvious naming issues
- Missing tests for cases you implemented

A self-review takes 10 minutes and saves the reviewer from wasting time on obvious issues.

---

## Reviewer Checklist

### Design
- [ ] Does the PR description explain the why?
- [ ] Does the approach fit the existing architecture?
- [ ] Is there a simpler solution that solves the same problem?
- [ ] Does it introduce new coupling or dependencies that could be avoided?

### Correctness
- [ ] Are there security implications? (input validation, auth checks, secrets in code)
- [ ] Are there performance implications? (N+1 queries, missing pagination, large in-memory collections)
- [ ] Does error handling surface failures appropriately to callers?
- [ ] Does it break the API contract for existing consumers?

### Quality
- [ ] Are tests present and meaningful (behavior, not implementation)?
- [ ] Does it introduce tech debt? Is it intentional and tracked with a ticket?
- [ ] Does it update relevant documentation (API docs, READMEs, runbooks)?

---

## Code Review Anti-Patterns

**Rubber Stamping (LGTM Without Reading)**
- Approving without reading is worse than not approving; it creates false confidence
- If you don't have time to review, say so rather than fake-approving

**Nit-Picking Only**
- Spending the entire review on naming and formatting while ignoring a missing auth check
- Prioritize: design > correctness > complexity > tests > naming > style
- Automate formatting; don't waste review cycles on it

**Blocking on Style Opinions**
- "I would have structured this differently" is not a blocking comment unless the structure causes a concrete problem
- Personal style preferences that aren't captured in team standards should be `nit:` or optional

**Drive-By Reviewing**
- Leaving 15 comments on day 1 and never responding when the author addresses them
- If you review, you own the thread until it is resolved or explicitly handed off

**Too Many Reviewers**
- Assigning 5 reviewers: each assumes the others will catch issues; nobody feels responsible
- Designate 1-2 required reviewers; add others as optional/FYI

**Reviewing After Merge**
- "I noticed in production that..." — useful for post-mortems, not for changing the merged code
- If you see something critical post-merge, open a new issue immediately

**Asking Questions That Should Be Requirements**
- "Should this handle the case where the user is deleted mid-request?" — this is a requirement question; escalate it, don't block the PR for it
- If the behavior is undefined, define it in a ticket and let the author proceed with a reasonable default

---

## Building a Review Culture

**Make it safe to disagree**: reviewers should feel safe saying "I think this is wrong" and authors should feel safe saying "I disagree."

**Celebrate good reviews**: acknowledge reviewers who catch real issues or explain why something is a problem; this signals that reviewing is valued work, not a tax.

**Define team standards explicitly**: "we use immutable DTOs" is much easier to enforce in reviews when it is written down somewhere. Convert repeated review comments into documented standards.

**Pair programming as a complement**: for complex changes, 30 minutes of pairing before submitting the PR is worth more than 2 hours of async review.

**Review your own team's stats quarterly**:
- Average time-to-first-review
- Average PR cycle time (open → merge)
- PR size distribution
- Number of PR iterations before merge

Large average PR size, long cycle time, or many iterations are signals of review culture problems — not individual failures.
