---
name: openspec-new-change
description: Start a new OpenSpec change using the experimental artifact workflow. Use when the user wants to create a new feature, fix, or modification with a structured step-by-step approach.
allowed-tools: Bash(openspec:*)
license: MIT
compatibility: Requires openspec CLI. Requires an OSOW planning store (Odysseus OSOW planner prerequisites).
metadata:
  author: openspec
  version: "2.0"
  generatedBy: "1.13.0"
---

**OSOW planner shape (MANDATORY in this store)**: Changes in this store are planned by Odysseus as OSOWs from `.openspec.yaml` alone (`POST /api/orchestrator/stores/{store_id}/osows`). Every change MUST carry `depends_on:` as a YAML list (`[]` when none — an absent key is a `400` refusal), exactly one `repository: owner/name`, an optional single `base_branch:`, a `tasks.md`, a `## Depends On` heading in `proposal.md` mirroring `depends_on`, and an id matching `^[a-z0-9]+(-[a-z0-9]+)*$` (≤ 80 chars). `openspec new change` writes none of the planner fields; add them by hand right after scaffolding. The full shape, the procedure for choosing the **correct** `depends_on` value, and a self-check script live in `../openspec-propose/references/odysseus-planner-shape.md` — read it before creating a change and run the self-check before reporting the change ready.


Start a new change using the experimental artifact-driven approach.

**Store selection:** If the user names a store (a store is a standalone OpenSpec repo registered on this machine) or the work lives in one, run `openspec store list --json` to discover registered store ids, then pass `--store <id>` on the commands that read or write specs and changes (`new change`, `status`, `instructions`, `list`, `show`, `validate`, `archive`, `doctor`, `context`, `schemas`, `view`). Once selected, treat `--store <id>` as sticky for the rest of the workflow. Every unscoped example of those commands below is shorthand: before running it, append the flag. For example, run `openspec status --change "<name>" --json --store "<id>"`, not the unscoped form shown below. Other commands do not take the flag. Hints printed by commands already carry the flag; keep it on follow-ups. Without a store, commands act on the nearest local `openspec/` root.

**Input**: The user's request should include a change name (kebab-case) OR a description of what they want to build.

**Steps**

1. **If no clear input provided, ask what they want to build**

   Ask the user (open-ended, no preset options):
   > "What change do you want to work on? Describe what you want to build or fix."

   From their description, derive a kebab-case name (e.g., "add user authentication" → `add-user-auth`).

   **IMPORTANT**: Do NOT proceed without understanding what the user wants to build.

2. **Determine the workflow schema**

   Use the default schema (omit `--schema`) unless the user explicitly requests a different workflow.

   **Use a different schema only if the user mentions:**
   - A specific schema name → use `--schema <name>`
   - "show workflows" or "what workflows" → run `openspec schemas --json` and let them choose

   **Otherwise**: Omit `--schema` to use the default.

3. **Create the change directory**
   ```bash
   openspec new change "<name>"
   ```
   Add `--schema <name>` only if the user requested a specific workflow.
   This creates a scaffolded change in the planning home resolved by the CLI.

   **Then add the planner metadata to `<changeRoot>/.openspec.yaml`** (the CLI writes only `schema:` and `created:`). Ask the user for the target repository and base branch if not already known, and decide the direct prerequisites per `../openspec-propose/references/odysseus-planner-shape.md` § "Choosing the correct `depends_on` value" (check `ls openspec/changes openspec/changes/archive`). The file MUST end up as:

   ```yaml
   schema: spec-driven
   created: <as written>
   depends_on: []            # or `- <change-id>` per verified direct prerequisite (P2)
   repository: owner/name    # exactly one (P3)
   base_branch: <your store's convention>          # optional (P4)
   ```

   This is metadata, not an artifact — writing it here does not violate the "no artifacts yet" guardrail. Tell the user in the output what you set, and that `proposal.md` will need a matching `## Depends On` heading.

4. **Show the artifact status**
   ```bash
   openspec status --change "<name>" --json
   ```
   Use the returned `planningHome`, `changeRoot`, `artifactPaths`, and `nextSteps` instead of assuming repo-local paths.

5. **Get instructions for the first artifact**
   The first artifact depends on the schema (e.g., `proposal` for spec-driven).
   Check the status output to find the first artifact with status "ready".
   ```bash
   openspec instructions <first-artifact-id> --change "<name>"
   ```
   This outputs the template and context for creating the first artifact.

6. **STOP and wait for user direction**

**Output**

After completing the steps, summarize:
- Change name and location
- Schema/workflow being used and its artifact sequence
- Current status (0/N artifacts complete)
- Planner metadata written to `.openspec.yaml`: `depends_on`, `repository`, `base_branch`
- The template for the first artifact
- Prompt: "Ready to create the first artifact? Just describe what this change is about and I'll draft it, or ask me to continue."

**Guardrails**
- Do NOT create any artifacts yet - just show the instructions
- Do NOT advance beyond showing the first artifact template
- If the name is invalid (not `^[a-z0-9]+(-[a-z0-9]+)*$`, or longer than 80 characters), ask for a valid name — Odysseus derives the branch `odysseus/pr<N>-<name>` from it (P5)
- Never leave `.openspec.yaml` without a `depends_on:` list and a single `repository:`; never write a dependency id you have not verified exists
- If a change with that name already exists, suggest continuing that change instead
- Pass --schema if using a non-default workflow
