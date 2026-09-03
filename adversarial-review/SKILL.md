---
name: adversarial-review
description: Hand finished work to a read-only agent whose only job is to break it, then triage its findings and apply the fixes yourself. Trigger on "try to break it", "check what you built", "run a second pair of eyes", "review it against the working one" — and ALSO unprompted when you have just finished something whose failure mode is silent — a template, a generated payload, a config, a query, a migration, anything that renders or runs where you cannot watch it. Especially when a working equivalent already exists to compare against. Do NOT trigger for a change you can fully verify by running it, and do not use it as a substitute for the project's own tests and gates.
---

# Adversarial review

The goal is not "a second opinion" but **"find what will fail where I cannot see it, before it ships."**

A reviewer that reports is worth more than a reviewer that edits: you keep one author, and every fix stays a decision you can defend. **The agent MUST NOT touch files.**

## Where this sits

| Step | What runs |
|---|---|
| While writing | the project's per-edit loop — types, lint, the touched tests |
| When the implementation is written | your own read of the diff |
| When it is finished and you cannot observe it running | **this skill** |
| When the findings are addressed | the project's release gates |

Running this on half-finished work reviews a draft. Running it instead of the tests finds
nothing a test would have caught.

## Stage 0 — Preconditions

The work is complete, your own checks pass, and you can name the thing you cannot verify:
the template renders inside a vendor product, the payload is consumed by a service you
cannot call, the config takes effect only on deploy. That gap is the reason to run this.

State the gap in one line before you start. If there is no gap, run the tests instead.

## Stage 1 — Assemble the brief

**Put the reference on disk.** If a working equivalent exists — the shipped template, the
service that already does this, the endpoint's live consumer — copy it into a file beside
the deliverable and name it the source of truth. A comparison against your own summary of
the reference proves nothing; the agent must read the real thing.

```bash
# whatever form it takes: a file the user pasted, a sibling implementation, a live export
cp <reference> <workdir>/reference-<name>.<ext>
```

Then write down, before you spawn anything:

1. **Deliverable** — absolute path to every file under review, each labelled.
2. **Reference** — the file above, with the rule: *a deviation from a pattern the working
   one uses is a defect unless justified.*
3. **Ground truth in code** — paths to what produces and what consumes the thing, so
   claims get traced rather than reasoned about.
4. **Established facts** — everything you already proved this session, and every decision
   the user has already made, marked *take as given, do not re-litigate*. **Skipping this
   is the most expensive mistake in this workflow**: you get a confident report arguing
   against a choice made an hour ago, and you spend a round trip explaining it. Include
   the failures you already hit — they tell the agent which class of bug is live here.
5. **What to hunt** — the failure modes specific to this artifact, always including: what
   silently produces nothing, what renders empty, what is inconsistent between the files,
   what no producer will ever populate.

## Stage 2 — Launch

Use a **read-only** agent type — one whose tools exclude Edit and Write — in the
background, and repeat the restriction in the prompt so the tooling and the words agree.

```
You are an adversarial reviewer. Find every way this breaks. Be hostile and specific.
Do NOT modify, create, or fix any file — findings only.

## Under review
<paths, each labelled>

## Reference — the source of truth
<path>. It is live and working. Where the new work deviates from a pattern it uses,
say so and judge the risk. A construct with NO precedent in the reference is the
highest-risk item in your report, even when nothing proves it broken.

## Ground truth
<paths to the producer, the consumer, the schema, the config>. Trace claims there.

## Established — take as given, do not re-litigate
<what you proved this session; what the user has already decided>

## Hunt for
<artifact-specific failure modes>

## Output
Ranked worst-first. For each: file, line, what breaks, the input that triggers it, and
CONFIRMED (traced in code or plain in the markup) vs SUSPECTED (plausible, unverified).
Then a construct-provenance table: for each construct I used, does the reference use
that exact form, a different one, or none. Then a short list of what you checked and
found correct, so my fixes do not churn working parts. No fixes, no edits.
```

While it runs, **do not redo its work on the same files.** Wait, or do something disjoint.

## Stage 3 — Triage

Yours, not the agent's. Every finding lands in exactly one bucket:

| Bucket | Action |
|---|---|
| **Real defect** | Fix it. State the failure in one sentence — what would have happened, to whom. |
| **Unjustified deviation from the reference** | Revert to the reference. Parity with something known to work beats a local improvement you cannot test. |
| **Inherited from the reference** | Leave it. Fixing the reference's own problems inside your change is scope creep — note it once. |
| **Contradicts a decision the user made** | Reject it, say so plainly, do not reopen the decision. The user can overrule. |

Severity, for what you fix and how you report it:

- **CRITICAL** — silently produces nothing, or the wrong thing, in the common path.
- **MAJOR** — wrong behaviour under realistic input; a contract mismatch with the consumer.
- **MINOR** — real but bounded: a cosmetic break in one client, a stale comment.
- **NIT** — optional polish. Keep these few.

**Rules.** Act on CONFIRMED; verify a SUSPECTED before touching code, and say which way it
went. A long report is not a mandate — some findings are wrong, and an agent that read the
ticket but not this session's decisions will be confidently wrong. MUST NOT rewrite working
code on a guess.

## Stage 4 — Fix, then re-run the real checks

Apply the fixes yourself, then run the project's own gates — types, lint, tests — and say
which passed. **An agent's approval is not a build.**

## Report

```markdown
### Fixed
- <what was broken, in one sentence each — the failure, not the diff>

### Declined
- <finding> — <why: inherited / already decided / wrong>

### Still unverifiable
- <what only the real environment can confirm, and what it would take>
```

Reporting the declines as clearly as the fixes is what shows the review was read rather
than obeyed.

## Anti-patterns

- Letting the agent edit. Two authors in one file, no coherent story.
- Running it with no reference when a working equivalent exists.
- Omitting the established-facts section, then arguing with the report.
- Accepting every finding, or accepting none.
- Treating the agent's silence on an area as coverage of it.
