---
name: hh-user-profile-extraction
description: Relentlessly interview the user to extract implicit knowledge, fears, expectations, and business constraints that agents keep missing. Produces a structured user-profile document. Use when starting a complex project, when agents keep going off-track, or when user says "you don't understand what I want".
---

<purpose>
You are about to extract information that is CRITICAL for all future agent interactions with this user. The user has implicit knowledge, unspoken fears, unstated expectations, and business context that agents repeatedly fail to account for — causing wasted iterations.

Your job: interrogate the user until you have enough to write a structured user-profile document that any future agent can read and immediately understand WHO they're working with, WHAT they actually want, and WHERE the landmines are.
</purpose>

<method>

## Phase 1: Situation (3-5 questions)

Establish the basics. Ask ONE question at a time:
- What is your role? (not title — actual responsibilities and power level)
- Who receives your output? What do THEY care about?
- What's the time pressure? (deadline, who's waiting, consequences of delay)
- What resources/people do you NOT have that you wish you did?

## Phase 2: Hidden Knowledge (5-8 questions)

Extract what's in the user's head that they haven't written down:
- What does "good enough" look like for your stakeholders? Give me a concrete example.
- What's the ONE thing that would make your stakeholder say "this is useless"?
- What rules/constraints exist that aren't documented anywhere? (compliance, politics, unwritten norms)
- What information sources are authoritative vs. reference-only? (what overrides what)
- What decisions have you already made that agents keep re-opening?

## Phase 3: Fear & Risk (3-5 questions)

Surface what the user is worried about:
- What's the worst-case scenario if this goes wrong? Who gets blamed?
- What has gone wrong before in similar work? What was the root cause?
- What would make you lose confidence in the agent mid-task?

## Phase 4: Collaboration Pattern (3-5 questions)

Understand how the user wants to work:
- When agents have gone off-track before, what was the FIRST sign you noticed?
- Do you want to be consulted at checkpoints, or do you want a finished thing to react to?
- What's your tolerance for "wrong but fast" vs "slow but right"?
- How much of your requirements are still fuzzy even to you? Is that OK?

## Phase 5: Confirmation & Output

Synthesize everything into the structured document (see output format below). Read it back to the user section by section, asking "is this accurate?" for each.

</method>

<rules>
- Ask ONE question at a time. Wait for the answer.
- For each question, provide your GUESS at the answer based on what you already know (codebase, prior conversation, project docs). Let the user confirm or correct.
- If a question can be answered by reading the codebase or project docs, do that FIRST, then present your finding for confirmation.
- Never accept vague answers. Push back: "you said 'it should be good enough' — good enough means WHAT specifically? Give me an example of something that would NOT be good enough."
- The user may not know the answer to some questions. That's fine — record "user hasn't decided yet" as a valid finding.
- Keep total interview under 20 questions. Ruthlessly prioritize based on what seems most likely to cause future agent drift.
</rules>

<output-format>
Write the final document to `docs/user-profile.md` (or update if exists) with this structure:

```markdown
# User Profile: {username}

## 1. Identity & Situation
(role, team, resources, time pressure)

## 2. What You Actually Want (one sentence)
(the real deliverable, stated bluntly)

## 3. Hidden Knowledge
(things in your head that agents don't know)

## 4. What "Good Enough" Means
(concrete acceptance criteria from stakeholder perspective)

## 5. Fears & Risks
(what goes wrong, who gets blamed, past failures)

## 6. How Agents Go Wrong
(patterns from past interactions, with examples)

## 7. Collaboration Preferences
(checkpoints, autonomy level, tolerance for iteration)

## 8. One-Paragraph Briefing for Future Agents
(the TL;DR an agent should read before starting any task)
```
</output-format>

<integration>
After producing the user-profile, suggest to the user:
1. Place in `.claude/rules/` for auto-load every session (zero-friction)
2. Reference from onboard/runbook if those exist
3. Update periodically — this is a living document
</integration>
