# AGENTS.md — the Dynamic Agents gates workspace

The gates half of the system: the service agents talk *through*, and the wire
contract that keeps it and the agent runtime from drifting apart.

| repo | what it is |
| ---- | ---------- |
| [`g2a-protocol`](g2a-protocol/AGENTS.md) | the gatekeeper↔agent wire contract: constants and pure functions, no dependencies |
| [`slack-gatekeeper`](slack-gatekeeper/AGENTS.md) | the Slack-anchored gatekeeper — routing, registration, the A2A crossing, and its built-in agents |

`slack-gatekeeper` hosts its built-in agents — admin and onboarding — on
`@dynamicagents/core`, as core tenants in its own Worker. It still **calls them
through core's A2A edge exactly as it calls a remote agent**: the same gatekeeper
token, the same push callback to `/a2a/notifications`, only handed over in-process.
So `@dynamicagents/g2a-protocol` stays the one coupling on the wire, built-in or
remote, and every rule in it — no dependencies above all — holds for the same reason
as before: both sides of every crossing depend on it.

Hosting core ties the gatekeeper to core's release train. Its `agents` and
`@cloudflare/think` versions are bounded by core's peer ranges, so a dependency round
here follows core's rather than running ahead of it, and it depends on core from the
registry — a git ref is temporary, as it is everywhere in the train.

The agent side lives in a separate workspace (`dev-agents`: `core`, `plugins`,
`starter`). `g2a-protocol` is a submodule of both, pinned independently — each records
what was green for its own side, so the two pins may legitimately differ.

---

## Where a change goes

| You are changing… | It goes in |
| ----------------- | ---------- |
| a value both sides must spell identically | `g2a-protocol` |
| anything that signs, verifies, fetches, or reads config | the consumer, never the protocol |
| Slack routing, registration, the A2A crossing | `slack-gatekeeper` |
| the agent runtime's half of verification | `core`, in the other workspace |
| a developer skill, or a script both workspaces run | [`dev-skills`](https://github.com/dynamicagents/dev-skills), then a skills update here |

The test for `g2a-protocol` is two questions, and both must point that way: **must two
repos spell it identically**, and **can they both import it from somewhere that already
owns it?** If the second is yes, it belongs there and not here.

The card and endpoint checks stay in the gatekeeper; the verification chain stays in
core. They are not the same check and must not be made to look like one.

---

## A change to `g2a-protocol` is a change to the wire

The two sides do not interoperate across it in either direction — a mismatched claim
name is a total outage, not a degraded mode. So:

- **Bump the minor**, with `npm version minor`. On 0.x, npm reads `^0.1.0` as `0.1.x`,
  so a minor is a hard break nobody picks up by accident: someone has to type the new
  range and notice why.
- **Ship both consumers in the same release.** There is no ordering where one goes first
  safely.
- **Update `src/claims.spec.ts` by hand.** Those assertions are literals, not references,
  deliberately — importing a constant and asserting it equals itself tests nothing.
  Typing the string twice is the point; the second time is when the cost registers.

A version bump reaching `main` is what ships it: on the first green Test run for a commit
carrying that version, `release.yml` publishes over OIDC and only then cuts the tag.

---

## Working here

```bash
npm run bootstrap    # submodules on a branch, node_modules, skill links. Run this first.
npm run check        # skill links and submodule structure are intact
npm run skills       # re-link .claude/skills after adding or removing a skill
npm run sync         # put every submodule on its branch and fast-forward it
```

**How this workspace works is the `superproject` skill**, and so are the scripts these
commands run (`.agents/skills/superproject/scripts/`). Load it before cloning, syncing,
editing `.gitmodules` or committing a gitlink. In short: `bootstrap` and `sync` leave every
submodule on the branch `.gitmodules` declares, `main` for each, and never on a detached
HEAD; every submodule url is HTTPS, and SSH is a global `insteadOf` of your own; a
submodule pointer is a known-good combination, and a PR whose only change is a gitlink is
never opened unprompted. The skill has the why of each.

The `git+ssh://` line for `slackify-markdown` in slack-gatekeeper's `package-lock.json` is
the lockfile case the skill's section on HTTPS remotes describes: changing it makes things
worse.

**Verify with `npm run check` in the repo you touched, not `npm test`.** Vitest
transpiles specs without typechecking them, so a type error passes a green suite. Where
`wrangler.jsonc` bindings or compat settings moved, run `npm run types` first and commit
the regenerated `worker-configuration.d.ts` — it is generated but committed, and
`check` fails when it goes stale.

**`npm run cf`** in `slack-gatekeeper` is a thin Cloudflare API proxy for inspecting the
deployed Worker — logs, Workflow instances, AI Gateway calls. It reads credentials from a
gitignored `.cf.env` and redacts them from all output, so the token never lands in shell
history or in an agent's context. Prefer it to pasting a token anywhere.

---

## Pull requests

**Before opening, answering or merging a PR in any of these repos, load the
`pull-requests` skill.** It holds the rules and the scripts: one Copilot review per PR,
requested when it opens ready for review, answered in one pass with every thread replied
to and resolved; never another review asked for, and never one waited for; and a merge
only once the review has landed, because merging does not cancel one in progress.

---

## Skills

Skills live in `.agents/skills/`, and `.claude/skills/` holds a relative link to each,
which is where Claude Code looks. Both are committed, so a clone carries its skills, and
`skills-lock.json` records where each installed skill came from:

- **`superproject` and `pull-requests`** are authored in
  [`dynamicagents/dev-skills`](https://github.com/dynamicagents/dev-skills), which
  `dev-agents` installs from too. Change one there, never here: the next update overwrites
  this copy.
- **The platform skills** come from Cloudflare's and Vercel's own repositories.

**The Cloudflare skills are the reference for the platform: use them instead of recalling
it.** `cloudflare`, `wrangler`, `workers-best-practices`, `durable-objects` and `agents-sdk`
retrieve current documentation rather than relying on training data, which for Workers APIs
and limits goes stale fast. Reach for them before answering from memory about bindings,
limits, compatibility dates, or Durable Object and Workflow rules. Neither this file nor a
repo's own `AGENTS.md` should grow a third copy of what they already say.

Adding, removing and updating a skill, and why the install names two agents, are the
`superproject` skill's, under its section on the skills a workspace installs. A dependency
round updates them: `npx skills check`, then `npx skills update -p -y`, with the updated
copies and `skills-lock.json` in the round's PR. `npm run check` fails on a skill with no
link, because Claude Code does not announce a skill it never found.

---

## Running Claude here

Launch from this directory. Skill discovery walks parent directories only as far as the
repository root, and each submodule is its own repository root — so a session started
inside `slack-gatekeeper/` may not see the workspace skills, while one started here sees
both these and, on demand, each repo's own `AGENTS.md` as you work in it.
