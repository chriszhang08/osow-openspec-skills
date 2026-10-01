# OSOW planner shape for a change in this store

This is the shape every active change under `openspec/changes/<id>/` in this
store MUST have before it is merged to `master`. Odysseus plans a merged
planning pull request as one **OSOW** (`POST /api/orchestrator/stores/{store_id}/osows`)
by reading each touched change's `.openspec.yaml` at the merge commit. No model
is consulted, nothing is inferred from prose, and one change that fails a
prerequisite refuses the **whole** OSOW for that pull request.

Authoritative sources (read them, do not paraphrase from memory):

- `openspec/specs/osow-planning/spec.md`, requirement *A change that does not
  meet the planner prerequisites refuses the whole OSOW* (until
  `replace-change-batching-with-osow-dag` is archived, read the delta at
  `openspec/changes/replace-change-batching-with-osow-dag/specs/osow-planning/spec.md`).
- `openspec/specs/openspec-master-store/spec.md`, requirement *A change's
  planner metadata is read from its own metadata file* (same caveat: read the
  delta under the active change until it is archived).
- `openspec/config.yaml` → `rules.proposal` (the `## Depends On` rule).
- `docs/openspec-planner-prerequisites.md` in the `chriszhang08/odysseus`
  repository (author-facing P1–P11 list).

## `.openspec.yaml` — the machine-read contract

`openspec new change` writes only `schema:` and `created:`. Every planner field
below MUST be added by hand, in this file, in the change directory root.

```yaml
schema: spec-driven
created: 2026-09-26
# Planner prerequisites — read by Odysseus at POST /osows. Never inferred from prose.
depends_on: []            # REQUIRED. A YAML list of change ids. `[]` when none. Absent key = refused (P2).
repository: owner/name    # REQUIRED. Exactly one `owner/name` string, never a list (P3).
base_branch: <your store's convention>          # OPTIONAL. One legal git branch name. Omit only if a human will set it in the panel (P4).
```

| Field | Correct | Wrong |
|---|---|---|
| `depends_on` | Always present. `[]` **or** a list of ids of other changes in this store. | Missing key; `depends_on: none`; a string instead of a list; ids that are not real change directories. |
| `repository` | One `owner/name` (e.g. `chriszhang08/odysseus`). | A list; a URL; absent; naming the repo only in `proposal.md` (prose is never parsed). |
| `base_branch` | One branch name (e.g. `uat`, `master`). Need not exist yet. | A list; empty string; illegal ref characters. |

### Choosing the correct `depends_on` value

`depends_on` is the set of **direct** prerequisites: other changes in this
store that must be landed (or accepted by a human) before this change's run may
be released. Decide it deliberately, per id:

1. **Does this change assume another change's work is already live?** (It
   extends a route/table/module that another active change introduces, or its
   tasks would fail to compile/run without it.) → list that change id.
2. **Is the prerequisite already archived** under `openspec/changes/archive/`?
   → it is already satisfied. You MAY still list it (it is recorded as
   satisfied, no edge); do not list it merely because it is "historically
   related".
3. **Is the prerequisite an active change that is neither in this pull request
   nor already planned in an earlier OSOW?** → the OSOW will be **refused
   (P7)**. Either include that change's directory in the same pull request, or
   drop the dependency and explain the sequencing in the proposal.
4. **Would this id close a cycle** (directly, or through an earlier OSOW)? →
   refused (P8). Remove one edge; the newer change depends on the older, never
   both ways.
5. **Nothing above applies** → `depends_on: []`. This is a decision, not a
   default; say `None` under `## Depends On` too.

Transitive dependencies are **not** walked: if C directly relies on both A and
B, C lists both, even if B already lists A.

Each id MUST be the exact directory name under `openspec/changes/` (or the
`<id>` part of `openspec/changes/archive/<date>-<id>/`, without the date
prefix). Verify with `ls openspec/changes openspec/changes/archive` before
writing it.

## `proposal.md` — the human-read mirror

`openspec/config.yaml` `rules.proposal` requires a `## Depends On` heading
listing the same ids as `depends_on`, or the literal word `None`. This heading
is for a human reader only; Odysseus never parses it. The two MUST agree.

```markdown
## Depends On

- add-export-endpoint
```

or

```markdown
## Depends On

None
```

Do **not** use the deprecated `**Repositories:**` prose header. The repository
lives in `.openspec.yaml` `repository:` only.

## Prerequisites checklist (P1–P11)

| # | What the author must do | Enforced |
|---|---|---|
| P1 | Land the change on `master` through a pull request on an `odysseus/`-prefixed branch. Never push a change to `master` directly. Do not archive a change in the pull request that plans it. | workflow / server |
| P2 | `.openspec.yaml` has `depends_on:` as a list (`[]` is fine, absent is not). | server `400` |
| P3 | `.openspec.yaml` has exactly one `repository: owner/name`. | server `400` |
| P4 | `base_branch:`, if present, is one legal branch name. | server `400` / approval |
| P5 | Change id matches `^[a-z0-9]+(-[a-z0-9]+)*$`, ≤ 80 chars (branch becomes `odysseus/pr<N>-<id>`). | server `400` |
| P6 | `tasks.md` exists. | server `400` |
| P7 | Every `depends_on` id is: in the same pull request, already planned in an earlier OSOW, or archived. | server `400` |
| P8 | Store-wide graph stays acyclic. | server `400` |
| P9 | The change is implementable in its one `repository` as one pull request. Split cross-repo work into separate changes linked by `depends_on`. | author |
| P10 | A change belongs to one OSOW; a later PR amends it in place while unreleased. | server |
| P11 | A PR that touches no active change directory records nothing. | workflow |

## Self-check before finishing any planning skill

Run from the store root and confirm every line passes for the change:

```bash
id=<change-id>
f=openspec/changes/$id/.openspec.yaml
test -f "$f" && test -f "openspec/changes/$id/tasks.md"                  # P6
grep -qE '^## Depends On' "openspec/changes/$id/proposal.md"              # config rule
echo "$id" | grep -qE '^[a-z0-9]+(-[a-z0-9]+)*$' && test ${#id} -le 80    # P5
python3 - "$f" <<'PY'
import sys, yaml
d = yaml.safe_load(open(sys.argv[1]))
assert isinstance(d, dict), "metadata is not a mapping"
assert isinstance(d.get("depends_on"), list), "depends_on missing or not a list (P2)"
r = d.get("repository"); assert isinstance(r, str) and r.count("/") == 1, "repository must be one owner/name (P3)"
b = d.get("base_branch"); assert b is None or (isinstance(b, str) and b.strip()), "base_branch must be one non-empty string (P4)"
print("planner metadata OK:", d["depends_on"], r, b)
PY
```

Then, for each id in `depends_on`, confirm `openspec/changes/<id>/` exists or
`openspec/changes/archive/*-<id>/` exists (P7), and that the ids under
`## Depends On` in `proposal.md` are the same set.
