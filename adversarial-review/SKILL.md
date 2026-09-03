---
name: adversarial-review
description: Hand finished work to a read-only agent whose job is to break it — verify every fact and every nuance, decide which of them are false positives and argue why, and report what actually fails. You then triage the report and apply the fixes yourself. Trigger when the person asks for it — "try to break it", "check what you built", "run a second pair of eyes", "let an agent review it" — and only then.
---

# Adversarial review

The goal is not "a second opinion" but **"try to break this, and prove which of the breaks are real."**

Two things make the run worth its cost:

- The agent **tries to break the work**, rather than reading it approvingly.
- The agent **filters its own false positives, with arguments** — a report that separates
  what it proved from what it merely suspects, and says why it dismissed the rest.

**The agent MUST NOT touch files.** A reviewer that reports keeps one author on the work,
and every fix stays a decision you can defend.

## Run it only when asked

There is no other precondition. The person asked; that is the trigger.

## Stage 1 — Assemble the brief

Give the agent everything it should reason from. Anything you leave out, it will guess at,
and a guess comes back as a confident finding you then have to argue with.

1. **What is under review** — absolute paths to every artifact, each labelled.
2. **Anything worth relying on.** If material exists that the work should be judged
   against — a working equivalent, a specification, an example the work must match, the
   data it will really receive — put it on disk beside the artifact and name it in the
   prompt. Hand over the real material, not your summary of it: a comparison against your
   own retelling proves nothing.
3. **Ground truth in code** — paths to whatever produces and consumes the thing, so claims
   get traced rather than reasoned about.
4. **Established facts — take as given, do not re-litigate.** Everything already proved in
   this session and every decision the person has already made. **Skipping this is the most
   expensive mistake in this workflow**: you get a well-argued report against a choice made
   an hour ago, and you spend a round trip explaining it. Include the failures you already
   hit — they tell the agent which class of bug is live here.
5. **What to hunt** — the failure modes that matter for this artifact. Always include the
   silent ones: what produces nothing without erroring, what is inconsistent between the
   files, what no producer will ever populate.

## Stage 2 — Launch

Use a **read-only** agent type — one whose tools exclude Edit and Write — in the
background, and repeat the restriction in the prompt so the tooling and the words agree.

```
You are an adversarial reviewer. Try to break this. Be hostile and specific.
Do NOT modify, create, or fix any file — findings only.

## Under review
<paths, each labelled>

## Material to judge it against
<paths>. Where the work deviates from what this material establishes, say so and judge
the risk — including a construct with no precedent here, which is high-risk even when
nothing proves it broken.

## Ground truth
<paths to the producer, the consumer, the schema, the config>. Trace claims there.

## Established — take as given, do not re-litigate
<what is already proved; what the person has already decided>

## Hunt for
<artifact-specific failure modes>

## Output
Verify every claim you are about to make, then rank them worst-first. For each: file,
line, what breaks, the input that triggers it, and CONFIRMED (traced in the code or plain
in the artifact) versus SUSPECTED (plausible, unverified — say what would settle it).

Separately, list what you considered and dismissed as a false positive, with the argument
for dismissing it. That list is as valuable as the findings: it tells me what you ruled
out rather than missed.

Close with what you checked and found correct, so my fixes do not churn working parts.
No fixes, no edits.
```

While it runs, **do not redo its work on the same files.** Wait, or do something disjoint.

## Stage 3 — Triage

Yours, not the agent's. Every finding lands in exactly one bucket:

| Bucket | Action |
|---|---|
| **Real defect** | Fix it. State the failure in one sentence — what would have happened, to whom. |
| **Unjustified deviation from the material it was judged against** | Bring it back in line. Parity with something known to work beats a local improvement you cannot test. |
| **Pre-existing, outside this change** | Leave it. Note it once; fixing it here is scope creep. |
| **Contradicts a decision the person made** | Reject it, say so plainly, do not reopen the decision. They can overrule. |

Severity, for what you fix and how you report it:

- **CRITICAL** — silently produces nothing, or the wrong thing, in the common path.
- **MAJOR** — wrong behaviour under realistic input; a contract mismatch with the consumer.
- **MINOR** — real but bounded: a cosmetic break in one client, a stale comment.
- **NIT** — optional polish. Keep these few.

**Rules.** Act on CONFIRMED; settle a SUSPECTED yourself before touching code, and say
which way it went. A long report is not a mandate — some findings are wrong, and an agent
that read the ticket but not this session's decisions will be confidently wrong. Read the
agent's own dismissals too: one of them may be dismissed on a bad argument. MUST NOT
rewrite working code on a guess.

## Stage 4 — Fix, then re-run the real checks

Apply the fixes yourself, then run the project's own gates — types, lint, tests — and say
which passed. **An agent's approval is not a build.**

## Report

```markdown
### Fixed
- <what was broken, in one sentence each — the failure, not the diff>

### Declined
- <finding> — <why: pre-existing / already decided / the argument does not hold>

### Still unverifiable
- <what only the real environment can confirm, and what it would take>
```

Reporting the declines as clearly as the fixes is what shows the review was read rather
than obeyed.

## Anti-patterns

- Letting the agent edit. Two authors in one file, no coherent story.
- Withholding material the agent needed, then arguing with what it guessed.
- Omitting the established-facts section, then spending a round trip on a settled question.
- Accepting every finding, or accepting none.
- Treating the agent's silence on an area as coverage of it.
