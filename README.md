# GlowHalo free agent skills

Two free `SKILL.md` files for AI coding agents (Claude Code, Codex CLI, OpenClaw). They run entirely inside your agent: no network calls, no API keys, no extra tools.

| Skill | What it does |
|---|---|
| [`bounded-qa-revision-loop`](bounded-qa-revision-loop/SKILL.md) | Reviews a deliverable (copy, doc, code snippet, listing) with independent reviewer passes, accepts only rejections that come with a concrete fix, and always stops: after a pass, 5 rounds, no new findings, or a repeated finding. Ends with a ready-made escalation message for a human. |
| [`link-and-session-liveness-checker`](link-and-session-liveness-checker/SKILL.md) | Tells dead links and expired sessions apart from bot blocks (403/429) and login walls, with a verdict table, evidence and next step per item. Never asks for passwords, tokens or cookie values. |

## Install

Copy a skill folder into your agent's skills directory, for example:

```bash
# Claude Code (personal skills)
cp -r bounded-qa-revision-loop ~/.claude/skills/
```

For Codex CLI or OpenClaw, put the folder wherever that agent loads skills from. Then ask your agent to use the skill, for example "run the bounded QA revision loop on this draft".

## More skills

About 50 more skills (review boards, QA checklists, planners) with real example outputs are listed at **https://skills.glowhalo.org**. They are paid; these two are free.

## License

MIT. Skills give structured review help, not legal, medical or financial advice, and results are not guaranteed.
