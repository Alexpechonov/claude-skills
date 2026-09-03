---
name: adversarial-review
description: Spawn a read-only agent whose job is to break the work you just produced, then triage its findings and apply the fixes yourself. Trigger when the user asks to "check it", "try to break it", "review what you built", "run a second pair of eyes", or after finishing an implementation whose failure mode is silent — a template, a config, a generated payload, a migration, anything that renders or runs somewhere you cannot observe. Also use when the work must match an existing, already-working reference.
---

# Adversarial review

A reviewer that reports is worth more than a reviewer that edits: you keep one coherent
author, and every fix is a decision you can defend. So the agent never touches files.

## Run it

Use the read-only agent type available in this session (one whose tool list excludes
Edit/Write), in the background, and say "report only, no fixes, no edits" in the prompt
itself — the tool restriction and the instruction should agree.

While it runs, do not redo its work on the same files. Wait.

## The prompt

Fill these six sections. The middle two are what separate a useful run from a noisy one.

**1. Stance.** "You are an adversarial reviewer. Find every way this breaks. Be hostile
and specific. Do NOT modify, create or fix any file — findings only."

**2. The deliverable.** Absolute paths to every file under review. Say which is which.

**3. The reference — the strongest section.** If a working equivalent exists, hand it
over as a file and name it the source of truth: "if the new one deviates from a pattern
the shipped one uses, the deviation is a defect unless justified". A reviewer with a
reference finds real problems; a reviewer with only a specification finds opinions.
Copy the reference into a file next to the deliverable rather than describing it — a
comparison against your own summary proves nothing.

**4. Ground truth in code.** Paths to whatever actually produces or consumes the thing:
the sender, the schema, the consumer, the config. Ask for claims to be traced there.

**5. Established facts — take as given, do not re-litigate.** Everything you already
proved this session, and every decision the user has already made. Without this section
you will get a confident report arguing against a choice the user made an hour ago, and
you will waste a round trip explaining that. Include the failures you already hit: they
tell the agent what class of bug is live here.

**6. What to hunt.** Be concrete about the failure modes that matter for this artifact,
and always include: what silently produces nothing, what renders empty, what is
inconsistent between the files, what no producer will ever populate.

## Ask for this output shape

- Ranked worst-first, each with file, line, what breaks, the input that triggers it.
- **CONFIRMED** (traced in code or plain in the markup) vs **SUSPECTED** (plausible,
  unverified). You act on these differently.
- A closing list of what was checked and found correct — that list is what stops you
  from churning working code on the next pass.
- When the work has a reference: for each construct, whether the reference uses that
  exact form, a different one, or none. **A construct with no precedent in the reference
  is the highest-risk item in any report**, even when nothing proves it broken. That is
  where silent failures live.

## Triage — yours, not the agent's

Sort every finding into one of four:

- **Real bug** → fix it, and say what the failure was in one sentence.
- **Deviation from the reference with no justification** → revert to the reference.
  Parity with something known to work beats a local improvement you cannot test.
- **Inherited from the reference** → usually leave it. Fixing the reference's own
  problems inside your change is scope creep; note it and move on.
- **Contradicts a decision the user already made** → reject it, say so plainly, and do
  not reopen the decision. Mention it once so the user can overrule if they want.

Report the rejects as clearly as the fixes. "I declined three findings and here is why"
is the part that tells the user the review was read rather than obeyed.

## After

Re-run the project's own checks — types, lint, tests — and say which passed. An agent's
approval is not a build.

## Anti-patterns

- Letting the agent edit. Two authors, one file, no coherent story.
- Running it without a reference when a working equivalent exists.
- Omitting the established-facts section, then arguing with the report.
- Accepting every finding. A long report is not a mandate; some findings are wrong.
- Treating SUSPECTED as CONFIRMED and rewriting working code on a guess.
