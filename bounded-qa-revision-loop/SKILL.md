---
name: bounded-qa-revision-loop
description: Run a deliverable (marketing copy, a document, a code snippet, a product listing) through independent reviewer passes chosen by deliverable type, then revise and re-review in a loop that cannot run forever. A rejection only counts if it comes with a concrete, actionable fix (the actionable-rejection gate); rejections without a fix are downgraded to notes. The loop stops at a pass, after 5 rounds, when a round finds nothing new, or when the same finding comes back twice, and then hands the open items to a human with a ready-made escalation message. Use when "review my text" prompts give you vague criticism and endless rewrites.
trigger: The user hands over a finished or near-finished deliverable and asks for QA, a review before publishing, a pre-launch check, or "keep reviewing and fixing until it is good", or pastes a draft and wants it checked from several angles and improved.
version: 1.0.0
compatible_agents: [claude-code, codex, openclaw]
---

# Bounded QA Revision Loop — rejections must carry a fix, and the loop must end

A generic "review my text" prompt has two failure modes. It returns vague objections ("feels
weak", "could be clearer") that nobody can act on, and when you ask it to keep improving, it never
says stop. This Skill fixes both with two hard rules: **a rejection without a concrete fix does
not count**, and **the loop ends after at most 5 rounds** with a hand-off to a human.

It reviews and revises text, code snippets and listings you paste. It does not publish anything,
call outside services, or ask for secrets or credentials. It is **not legal, medical or financial
advice**, and a pass from this Skill is **not a substitute for human sign-off**: for anything with
legal, health, money or safety consequences, a qualified person must approve it before release.

## Rules that apply throughout

- **Actionable-rejection gate.** A finding blocks the deliverable only if it has (a) a quote from
  the deliverable as evidence, and (b) a concrete fix: replacement text, a specific deletion, or a
  precise instruction a stranger could carry out without asking questions. Otherwise it is a note.
- **Independent passes.** Each lens reviews the same version of the deliverable without seeing the
  other lenses' verdicts. If you can spawn separate sub-agents, do; if not, run each lens as a
  separate pass and write its verdict before starting the next one, without revisiting earlier ones.
- **Evidence over taste.** Quote the exact words under criticism. "Tone is off" with no quote is
  not a finding. A claim you cannot check from the text itself is marked "unverified", never "false".
- **No invented facts.** Fixes may rewrite, cut or hedge a claim. They may not add statistics,
  features, prices, guarantees or sources the owner did not supply. If a fix needs a fact only the
  owner has, the fix is "owner supplies X", and it is flagged for the human.
- **Hard cap: 5 rounds.** Count rounds from 1. Round 6 never happens; escalate instead.
- **Only the author decides taste.** Style preferences with no risk attached are notes.
- **Never ask for or print a secret.** If the deliverable contains a key, password or personal
  data, report it as a blocker with the fix "remove and rotate", and do not repeat the value.

## Phase 1 — Intake

Write down, with "unknown" for anything missing (do not guess):

1. **Deliverable type** (see the table in Phase 2) and where it will be shown.
2. **Audience** and the one action you want them to take.
3. **Hard constraints** supplied by the owner (word limit, required disclosures, banned words, platform rules).
4. **Facts the owner confirms are true** (features, prices, policies). Everything else is unverified.
5. **The exact version under review**, labelled `v1`.

If there is no deliverable text yet, stop and ask for it.

## Phase 2 — Choose reviewer lenses

Pick 3 to 5 lenses; do not run all of them every time. Choose by the risk that matters most.

| Deliverable type | Default lenses |
|---|---|
| Product page / listing | Claims and honesty, Buyer clarity, Policy completeness, Skeptical competitor |
| Marketing email / ad copy | Claims and honesty, Audience fit, Policy completeness |
| Documentation / README | New-user walkthrough, Accuracy, Completeness |
| Code snippet / script | Correctness (trace by hand), Failure paths, Security and secrets, Readability |
| Report / proposal | Logic and evidence, Audience fit, Risk and omissions |
| Pricing or offer text | Claims and honesty, Buyer clarity, Policy completeness |
| Type not listed | Choose the 3 lenses covering the biggest risks and state why |

Lens definitions: **Claims and honesty** flags overclaims, absolutes and unverifiable superlatives.
**Buyer clarity** asks what a first-time reader cannot tell. **Policy completeness** checks that
refunds, limits, support, licensing and similar statements a buyer needs are present. **Skeptical
competitor** looks for reasons not to choose this. **New-user walkthrough** follows the steps as a
beginner would. Define any other lens in one sentence before using it.

## Phase 3 — Review pass: write rejection records

Each lens returns a verdict (PASS or REJECT) and zero or more records in this exact format:

```
ID: F-<n>            Lens: <name>
Finding:   <one sentence, what is wrong>
Evidence:  "<exact quote from the deliverable>"
Fix:       <replacement text or precise instruction; or NONE>
Severity:  BLOCKER | MAJOR | MINOR
```

Severity guide: **BLOCKER** is misleading, unsafe, broken, or missing something the audience needs
to decide. **MAJOR** clearly weakens the deliverable. **MINOR** is polish.

## Phase 4 — Apply the gate and downgrade rule

For every record, check in order:

1. Is the Evidence an exact quote that appears in this version? No: discard it.
2. Is the Fix concrete (replacement text or a step someone can perform)? "Make it better", "be more
   specific", "consider rewording" are **not** fixes.
3. If the Fix is NONE or vague: **downgrade to a note.** It no longer blocks and is not revised
   this round. Notes are listed at the end for the human.
4. Exception: a BLOCKER with no fix stays open but is marked **NEEDS-OWNER** (a fact or decision
   only the owner can supply). It is escalated rather than guessed at.

Lens verdict after the gate: REJECT only if at least one record survives as BLOCKER or MAJOR.
Records that survive as MINOR are applied if cheap, otherwise listed as notes.

## Phase 5 — Revise

Apply every surviving fix to produce the next version (`v2`, `v3`, ...). Change only what a fix
calls for, and list each change as `F-<n> -> what you changed`. If two fixes conflict, apply the
higher severity and note the conflict. Never "improve" text no record asked you to touch.

## Phase 6 — Round log and stop check

Maintain this table, one row per round, and show it in the output:

| Round | Version | Lenses run | Valid rejections (B/M/m) | Downgraded to notes | New findings | Result |
|---|---|---|---|---|---|---|

Stop when the **first** of these is true, and say which one:

1. **PASS**: every lens passes after the gate, with no open BLOCKER or MAJOR.
2. **ROUND CAP**: round 5 has finished and items are still open.
3. **NO NEW FINDINGS**: a round produced no valid finding that was not already seen. Further rounds
   would only reshuffle wording.
4. **REPEAT FINDING**: the same finding (same quote or same issue) appears in two rounds, even after
   a fix was applied. That means the fix is wrong or the problem needs a human decision.

Otherwise run the next round, re-reviewing only the lenses that rejected plus any lens that could be
affected by the changes. A PASS is a PASS only on the final version; re-run a full pass on it if
many edits were made.

## Phase 7 — Escalation message (stop conditions 2, 3 and 4, or any NEEDS-OWNER item)

Fill this template, with no extra prose:

```
QA loop stopped: <PASS | round cap | no new findings | repeat finding> after <n> round(s).
Current version: <vN, attached below>

Open items (need your decision):
1. <ID> [<severity>] "<quote>" -- <what is wrong>. Tried: <fix applied or NONE>. Needed from you: <fact or decision>.
2. ...

Notes (downgraded, no fix offered): <list, one line each>
Not checked: <claims marked unverified, anything outside the lenses>

This review does not replace human sign-off. Anything with legal, medical,
financial or safety impact needs review by a qualified person before release.
```

## Output format

Use these headings in order: `## Intake`, `## Lenses chosen`, `## Round log` (the table, updated each
round, with findings listed under each row), `## Final version`, `## Escalation` (only if the loop
did not end in a clean PASS), `## Notes`. Keep rejection records in the exact Phase 3 format. If the
user's deliverable is long, quote only the lines at issue. End with one line stating the stop
condition reached and the version number that is ready, or not ready, for human sign-off.
