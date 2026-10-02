# osow-openspec-skills

A shared OpenSpec authoring skillset for every repository that plans or
implements work through the Odysseus OSOW orchestrator. Odysseus plans a merged
pull request as one OSOW built solely from each change's `.openspec.yaml`, so
the skills that create, continue, update or verify a change enforce the planner
shape:

- `depends_on:` — a YAML list, always present (`[]` when none), holding the
  verified ids of this change's direct prerequisites;
- `repository:` — exactly one `owner/name`;
- `base_branch:` — optional, one branch name;
- `tasks.md` present, a `## Depends On` heading in `proposal.md` mirroring
  `depends_on`, and a branch-safe change id.

The canonical description of that shape, the procedure for choosing the correct
`depends_on` value, and a self-check are in
`openspec-propose/references/odysseus-planner-shape.md`.

## Skills

| Skill                                                     | Planner-shape duty                                                 |
| --------------------------------------------------------- | ------------------------------------------------------------------ |
| `openspec-propose`                                        | Mandates and writes the full shape; runs the self-check            |
| `openspec-new-change`, `openspec-ff-change`               | Write the metadata right after scaffolding; ff runs the self-check |
| `openspec-continue-change`, `openspec-update-change`      | Keep `.openspec.yaml` and `## Depends On` coherent                 |
| `openspec-verify-change`                                  | Reports shape failures as CRITICAL                                 |
| `openspec-archive-change`, `openspec-bulk-archive-change` | Archived changes count as satisfied dependencies                   |
| others                                                    | Reference the shape for awareness                                  |

## Using it as a submodule

Add it to a consumer repository at `.agents/skills`:

```bash
git submodule add https://github.com/chriszhang08/osow-openspec-skills .agents/skills
```

Update it by bumping the submodule commit, then re-run `openspec validate`
locally:

```bash
git -C .agents/skills pull origin main
git add .agents/skills
```

## Store-specific customization

This repository holds only generic content. A store that needs its own rules
(for example fork-ownership labels) keeps that content locally, layered on top
of the submodule, rather than editing the shared copy.
