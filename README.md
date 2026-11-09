# dev-gates

**The development environment for the [Dynamic Agents](https://github.com/dynamicagents)
gates side.**

The gatekeeper and the wire contract it speaks, pinned together at a known-good
combination, plus the agent skills they share — so a laptop, a cloud session, or a machine
that has never seen this project all reach the same place from one clone.

```bash
git clone --recurse-submodules https://github.com/dynamicagents/dev-gates.git
cd dev-gates
npm install && npm run bootstrap
```

Then launch your agent from this directory.

| repo | |
| ---- | --- |
| [`g2a-protocol`](https://github.com/dynamicagents/g2a-protocol) | the gatekeeper↔agent wire contract |
| [`slack-gatekeeper`](https://github.com/dynamicagents/slack-gatekeeper) | the Slack-anchored gatekeeper |

`slack-gatekeeper` hosts its built-in agents on `@dynamicagents/core` and calls them
across the contract like any remote agent. The agent side is a separate workspace,
[`dev-agents`](https://github.com/dynamicagents/dev-agents), which pins `g2a-protocol`
too.

[`AGENTS.md`](AGENTS.md) is the working guide. It is symlinked to `CLAUDE.md`, because
Claude Code reads that name.

---

## Skills

Skills live in `.agents/skills/`, with a link to each in `.claude/skills/`, which is where
Claude Code reads them. Both are committed, so a clone carries its skills, and
`skills-lock.json` records where each came from. The workspace's own skills are authored in
[`dev-skills`](https://github.com/dynamicagents/dev-skills): `superproject`, which is how this
workspace works and the scripts `npm run` calls, and `pull-requests`. The platform skills come
from Cloudflare's and Vercel's own repositories.

## Commands

| | |
| --- | --- |
| `npm run bootstrap` | submodules on a branch, `node_modules`, and skill links. Safe to re-run. |
| `npm run check` | skill links and submodule structure are intact |
| `npm run skills` | re-link `.claude/skills/` after adding or removing a skill |
| `npm run sync` | put every submodule on its branch and fast-forward it |
| `node .agents/skills/superproject/scripts/submodules.mjs --pushed` | every pinned commit is on a remote. Needs the network, so it is not in `check` |

## The submodule pointers

A pointer is a known-good combination, not a mirror, and `bootstrap` and `sync` leave every
submodule on its branch rather than a detached HEAD. The
[`superproject`](.agents/skills/superproject/SKILL.md) skill has both rules and why.

## License

Apache 2.0. Each submodule carries its own copy.
