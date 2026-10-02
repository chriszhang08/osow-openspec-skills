---
name: odysseus-publish
description: How this repository wants an Odysseus change run to publish its work. Use when an Odysseus run has implemented its OpenSpec change and needs to commit, push and open a pull request. The branch and base come from the run's instructions and must be used exactly.
---

# Publishing an Odysseus change

**Copy this into `.claude/skills/odysseus-publish/SKILL.md` in your repository and
edit it.** It is a starting point, not a standard: the whole reason this lives
here rather than in Odysseus is that publishing is repository-specific.
Odysseus supplies the _variables_ — which change, which branch, which base —
and this file supplies the _procedure_.

## What Odysseus gives you, and what you must not change

Your run instructions name a **planning store**, a **source commit** and a
**change directory**, and a **branch to publish** and a **base branch**.

Read the change from your checkout of the planning store at the given commit
— never from the store's current branch, which may have moved on — and
implement what its `tasks.md` and spec deltas say. Nothing about the change is
repeated in the instructions; the store is the source of truth.

You are given a branch and a base. Use both exactly as given. They are not
suggestions. Later
changes in the same OSOW may be stacked on your branch, and the pull request
opening is how Odysseus learns this change finished. A pull request from a
different branch, or onto a different base, is refused — the change stays
unpublished and its release halts.

## The credential you push with

Odysseus no longer injects credentials into a run per dispatch. They are
configured once, ambiently, on the OpenHands agent server itself (see
`docs/openhands-agent-server.md`) and persist there at rest across every run —
Odysseus keeps no credential vault and no baseline set of its own. A batch
only sees the secrets a human ticked for it in the orchestrator panel; nothing
is attached by default.

**The one you push with is `GITHUB_TOKEN`.** It must have been ticked for this
batch. If it was not — or if a step here needs a credential your run does not
hold — say so rather than working around it: the grant is a decision somebody
made, and routing around it is worse than failing.

## Steps

1. Fetch, and create the base branch from the repository's default branch if
   it does not exist yet. A base branch is chosen by a human in the Odysseus
   panel and may not have been created; Odysseus has no push credential into
   this repository and leaves creating it to you. Fetch first: another run may
   have just created it, and a second push of an identical ref is a no-op.
2. Branch off the base you were given, onto the branch you were given.
3. Commit your work on that branch.
4. Push it to the repository you were given.
5. Open a pull request from that branch into the base you were given.

## Edit these for your repository

Delete what does not apply and add what does. The items below are the ones that
most often differ, and getting them wrong is what a reviewer will send back:

- **Commit messages** — conventional commits? A ticket reference? A sign-off
  (`git commit -s`)? Signed commits (`-S`)?
- **Pull request body** — is there a template in `.github/`? Which sections are
  required? Should the change identifier and the planning pull request appear,
  and where?
- **Pre-push checks** — hooks, formatters, a generated-file step that must run
  before committing.
- **Anything a reviewer always asks for.** If you find yourself repeating it in
  review, it belongs here instead.

## Things worth telling the agent

- **The change's own tasks and spec scenarios are its acceptance.** Run what
  they say before publishing. If something does not pass, publish anyway and
  say so plainly in the pull request body — a silent failure is worse than a
  flagged one, and a human reviews every pull request before it lands.
- **Stay inside the change.** Work that the change does not describe belongs
  in another change. Say so in the pull request description if you had to
  reach outside it to finish.
