---
name: openspec-ff-change
description: Fast-forward through OpenSpec artifact creation. Use when the user wants to quickly create all artifacts needed for implementation without stepping through each one individually.
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires openspec CLI. Requires an OSOW planning store (Odysseus OSOW planner prerequisites).
metadata:
  author: openspec
  version: "2.0"
  generatedBy: "1.13.0"
---

**OSOW planner shape (MANDATORY in this store)**: Changes in this store are planned by Odysseus as OSOWs from `.openspec.yaml` alone (`POST /api/orchestrator/stores/{store_id}/osows`). Every change MUST carry `depends_on:` as a YAML list (`[]` when none — an absent key is a `400` refusal), exactly one `repository: owner/name`, an optional single `base_branch:`, a `tasks.md`, a `## Depends On` heading in `proposal.md` mirroring `depends_on`, and an id matching `^[a-z0-9]+(-[a-z0-9]+)*$` (≤ 80 chars). `openspec new change` writes none of the planner fields; add them by hand right after scaffolding. The full shape, the procedure for choosing the **correct** `depends_on` value, and a self-check script live in `../openspec-propose/references/odysseus-planner-shape.md` — read it before creating a change and run the self-check before reporting the change ready.


Fast-forward through artifact creation - generate everything needed to start implementation in one go.

**Store selection:** If the user names a store (a store is a standalone OpenSpec repo registered on this machine) or the work lives in one, run `openspec store list --json` to discover registered store ids, then pass `--store <id>` on the commands that read or write specs and changes (`new change`, `status`, `instructions`, `list`, `show`, `validate`, `archive`, `doctor`, `context`, `schemas`, `view`). Once selected, treat `--store <id>` as sticky for the rest of the workflow. Every unscoped example of those commands below is shorthand: before running it, append the flag. For example, run `openspec status --change "<name>" --json --store "<id>"`, not the unscoped form shown below. Other commands do not take the flag. Hints printed by commands already carry the flag; keep it on follow-ups. Without a store, commands act on the nearest local `openspec/` root.

**Input**: The user's request should include a change name (kebab-case) OR a description of what they want to build.

**Steps**

1. **If no clear input provided, ask what they want to build**

   Ask the user (open-ended, no preset options):
   > "What change do you want to work on? Describe what you want to build or fix."

   From their description, derive a kebab-case name (e.g., "add user authentication" → `add-user-auth`) matching `^[a-z0-9]+(-[a-z0-9]+)*$`, at most 80 characters (P5).

   Also establish the target **repository** (`owner/name`, exactly one — split cross-repository work into separate changes, P9), the **base branch** (optional), and the **direct prerequisite changes** in this store (run `ls openspec/changes openspec/changes/archive`; apply the decision procedure in `../openspec-propose/references/odysseus-planner-shape.md`). Warn the user if a prerequisite is active, unplanned and not shipping in the same pull request (P7).

   **IMPORTANT**: Do NOT proceed without understanding what the user wants to build.

2. **Create the change directory**
   ```bash
   openspec new change "<name>"
   ```
   This creates a scaffolded change in the planning home resolved by the CLI.

   **Then add the planner metadata to `<changeRoot>/.openspec.yaml`** (the CLI writes only `schema:` and `created:`):

   ```yaml
   schema: spec-driven
   created: <as written>
   depends_on: []            # or `- <change-id>` per verified direct prerequisite (P2)
   repository: owner/name    # exactly one (P3)
   base_branch: <your store's convention>          # optional (P4)
   ```

   Verify it parses and that every `depends_on` id is an existing change directory before continuing.

3. **Get the artifact build order**
   ```bash
   openspec status --change "<name>" --json
   ```
   Parse the JSON to get:
   - `applyRequires`: array of artifact IDs needed before implementation (e.g., `["tasks"]`)
   - `artifacts`: list of all artifacts, each with its `status` and its `requires` edges (the artifact IDs it directly depends on)
   - `planningHome`, `changeRoot`, `artifactPaths`, and `actionContext`: path and scope context. Use these instead of assuming repo-local paths.

4. **Create every artifact in the required set**

   Use a todo list to track progress through the artifacts.

   Loop through artifacts in dependency order (artifacts with no pending dependencies first):

   a. **For each artifact that is `ready` (dependencies satisfied)**:
      - Get instructions:
        ```bash
        openspec instructions <artifact-id> --change "<name>" --json
        ```
      - The instructions JSON includes:
        - `context`: Project background (constraints for you - do NOT include in output)
        - `rules`: Artifact-specific rules (constraints for you - do NOT include in output)
        - `template`: The structure to use for your output file
        - `instruction`: Schema-specific guidance for this artifact type
        - `skipped`/`warning`: present when the change declares skip_specs and this artifact must NOT be created - stop and pick another artifact
        - `resolvedOutputPath`: Resolved path or pattern to write the artifact
        - `dependencies`: Completed artifacts to read for context
      - Read any completed dependency files for context - always re-read them from disk, even if you saw them earlier in the conversation (the user may have edited them)
      - **Inspect the relevant project before drafting**: Read `context` and `rules` first, then inspect relevant implementation, nearby tests, configuration, and documentation outside `openspec/`. Keep inspection read-only and proportional to the change; reuse findings for later artifacts and inspect more only as needed.
        - Identify the target project from the request and project context; the planning home may be separate from the code. If the target is unclear, ask. For greenfield or non-code changes, inspect the available structure and relevant documents. If source is unavailable, state the limitation and ask when it materially affects the plan.
        - Ground scope, approach, and tasks in what you find. Distinguish observed behavior from assumptions and proposed additions; surface conflicts with existing specs instead of silently deciding which is correct.
        - Do this discovery now, rather than leaving generic "explore the codebase" or "make a plan" tasks for implementation. Keep any necessary follow-up investigation specific to an unresolved question.
      - If the `instruction` field delegates creation to a specific skill or command, invoke it to produce the artifact instead of writing the file yourself, then verify the artifact file exists at `resolvedOutputPath`
      - Otherwise create the artifact file using `template` as the structure and write it to `resolvedOutputPath`. If `resolvedOutputPath` is a glob, follow `instruction` to choose the concrete file path
      - **For `proposal.md` (this store)**: include a `## Depends On` heading listing exactly the ids in `.openspec.yaml` `depends_on`, or `None` when it is `[]`. Do not use the deprecated `**Repositories:**` header. **For `tasks.md`**: it MUST exist (P6) and every task MUST be doable inside the one `repository`
      - Apply `context` and `rules` as constraints - but do NOT copy them into the file
      - Show brief progress: "✓ Created <artifact-id>"

   b. **Continue until every artifact in the required set exists (not just `apply.requires`)**
      - After creating each artifact, re-run `openspec status --change "<name>" --json`
      - The required set is `applyRequires` plus every artifact reachable from those by following the `requires` edges in `status --json` - walk them transitively (spec-driven closes over proposal, specs, design, tasks). Leave artifacts outside that set alone
      - `status` is file-existence only, so an `applyRequires` artifact reading `done` does NOT mean its dependencies exist - writing `tasks.md` early marks `tasks` done while `specs` was never written. Use each artifact's `requires` edges, not its `status`, to build the required set: a `done` artifact still lists what it depends on
      - An artifact already reading `status: "skipped"` is satisfied: the change declares `skip_specs` in `.openspec.yaml`, so its files must NOT exist. Never try to create one
      - Create every artifact in the required set that is missing, then re-check - creating one can unblock others
      - Skip one only when `status` already reports it `skipped`, or when its own `instruction` says it is conditional: run `openspec instructions <artifact-id> --change "<name>" --json` and skip only if its `instruction` field marks it optional (e.g. "create only if..."). Spec-driven's `design.md` qualifies; `specs` qualifies only via the `skipped` status above, never by your own judgment. Tell the user, and do not reconsider it
      - Dependencies are enablers, not gates: if a required artifact is still `blocked` only because you skipped a conditional dependency, write it anyway
      - Stop when every artifact in the required set is `done`, `skipped`, or was deliberately skipped

   c. **If an artifact requires user input** (unclear context):
      - Ask the user to clarify
      - Then continue with creation

5. **Show final status**
   ```bash
   openspec status --change "<name>"
   ```

6. **Verify the OSOW planner shape (do not skip)**

   Run the self-check in `../openspec-propose/references/odysseus-planner-shape.md` § "Self-check". All lines must pass (`depends_on` list present, one `repository`, `tasks.md` exists, `## Depends On` mirrors `depends_on`, every dependency id resolves, id is branch-safe). Fix and re-run before reporting ready.

**Output**

After completing all artifacts, summarize:
- Change name and location
- List of artifacts created with brief descriptions, plus any conditional artifact you skipped and why
- Planner metadata verbatim from `.openspec.yaml` (`depends_on`, `repository`, `base_branch`) and the self-check result
- Reminder: commit on an `odysseus/`-prefixed branch and open a pull request to `master` (P1)
- What's ready: "All artifacts needed for implementation are ready."
- Prompt: "Run `/openspec-apply-change` or ask me to implement to start working on the tasks."

**Artifact Creation Guidelines**

- Follow the `instruction` field from `openspec instructions` for each artifact type - it is the authoritative guidance, even for familiar artifact names
- If the `instruction` field directs you to use a specific skill or command to create the artifact, invoke it instead of writing the artifact directly
- The schema defines what each artifact should contain - follow it
- Read dependency artifacts for context before creating new ones
- Use `template` as the structure for your output file - fill in its sections
- **IMPORTANT**: `context` and `rules` are constraints for YOU, not content for the file
  - Do NOT copy `<context>`, `<rules>`, `<project_context>` blocks into the artifact
  - These guide what you write, but should never appear in the output

**Guardrails**
- Never finish with `.openspec.yaml` lacking a `depends_on:` list or a single `repository:`; never write an unverified dependency id; never introduce a cycle; `## Depends On` and `depends_on` MUST agree
- Create every artifact the apply phase transitively depends on, not just the ids listed in `apply.requires`
- Always read dependency artifacts before creating a new one - re-read from disk, not from conversation memory (files may have changed since you last saw them)
- If context is critically unclear, ask the user - but prefer making reasonable decisions to keep momentum
- If a change with that name already exists, suggest continuing that change instead
- Verify each artifact file exists after writing before proceeding to next
