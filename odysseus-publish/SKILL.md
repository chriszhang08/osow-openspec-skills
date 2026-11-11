---
name: odysseus-publish
description: How this repository wants an Odysseus change run to publish its work. Use when an Odysseus run has implemented its OpenSpec change and needs to commit, push and open a pull request. The branch and base come from the run's instructions and must be used exactly.
---

# Publishing an Odysseus change

**Copy this into `.claude/skills/odysseus-publish/SKILL.md` in your repository and
edit it.** It is a starting point, not a standard: the whole reason this lives
here rather than in Odysseus is that publishing is repository-specific.
Odysseus supplies the _variables_ — which change, which branch, which base —
and this file supplies the _procedure_. Odysseus supplies no credentials; the
agent-canvas backend server provides them in your environment.

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

## The credentials you use, and where they come from

Odysseus does not supply credentials. It injects nothing into a run and keeps
no credential vault of its own. Every secret this skill uses is provided by the
**agent-canvas backend server**, which puts it in your run's environment.
Read each one from your own environment by the exact name below:

| Variable | Used for |
| --- | --- |
| `GITHUB_TOKEN` | Fetching, pushing the branch and opening the pull request |
| `ODYSSEUS_URL` | The manual publication-report fallback (below) |
| `ODYSSEUS_PUBLICATION_CREDENTIAL` | The bearer token for that fallback |

**Check before you start.** Confirm each variable is set and non-empty (for
example `[ -n "$GITHUB_TOKEN" ]`). Never print, echo or log its value.

**If one is missing, flag it in the conversation.** agent-canvas should
always provide these, so a missing variable is a misconfiguration of the
backend server, not something for you to fix. State which variable is
missing and which step it blocks. Do not ask Odysseus for a credential,
borrow another one (a personal token, Infisical credentials, a
credential helper, a different secret), mint one, or otherwise work around
the gap. Finish whatever work does not need the missing secret, then stop
and report it.

## How Odysseus learns the pull request opened

Opening the pull request is not enough on its own. A workflow in the target
repository, `.github/workflows/odysseus-publication-report.yml`, reports it:
on `pull_request` `opened`/`reopened` for branches starting with `odysseus/`,
it POSTs three fields — `repository`, `branch`, `pull_request_number` — to
`$ODYSSEUS_URL/api/orchestrator/publication-report`. It sends no code; Odysseus
reads the pull request from GitHub itself.

**The publication credential is sent as a bearer token:**

```
Authorization: Bearer $ODYSSEUS_PUBLICATION_CREDENTIAL
```

The old `X-Odysseus-Publication-Credential` header is no longer accepted and
gets a 401. The repository secrets are `ODYSSEUS_URL` and
`ODYSSEUS_PUBLICATION_CREDENTIAL`. If the workflow is missing or still uses the
old header, the batch fails as `unreported` once its run finishes.

Do not substitute `GITHUB_TOKEN`, Infisical credentials or any other credential
for the publication credential, and do not retry a refused report in a loop —
Odysseus gives the same refusal for every failure. A report is a manual
fallback only: if the workflow did not run, post the same three fields once to
`$ODYSSEUS_URL/api/orchestrator/publication-report` with the bearer header,
using `ODYSSEUS_URL` and `ODYSSEUS_PUBLICATION_CREDENTIAL` from your
environment, and tell the user the result. If either is missing, flag it as
described above instead of posting.

## Steps

0. Check that the credentials above are in your environment. Flag any that
   are missing.
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
