---
name: chong-ready
description: Get my relevant open PRs merge-ready. For each relevant pending PR (across the workspace repos), rebase onto origin/main, resolve conflicts, update the remote PR, then run a fixed quality checklist (Pydantic-first, function length, feature grouping + config-from-config, no per-vertical leak in generic code, typed signatures, migration FK/reference integrity, data-model alignment) and FIX what it finds. Triggers on "/chong-ready", "chong ready", "make my PRs ready", "get my PRs ready to merge", "rebase and clean up my PRs".
metadata:
  version: "1.0"
---

# chong-ready — make my open PRs merge-ready

Take the user's relevant open PRs from wherever they are now to **rebased, conflict-free,
quality-checked, and pushed**. This skill **fixes** what it finds — it is not a read-only
review. It composes the existing review skills; it does not duplicate them.

**Announce at start:** "Running chong-ready: discovering relevant PRs, then rebase → checks → push for each."

## Arguments
`/chong-ready [<pr-ref> ...] [<feature-keyword>]`
- No args → discover relevant open PRs authored by the user across the workspace repos.
- `<pr-ref>` → restrict to specific PRs, e.g. `poneglyph#1420 poneglyph-webapp#462`.
- `<feature-keyword>` → restrict to PRs whose branch/title matches (e.g. `staff`).

## Workspace repos
| Repo | Role |
|---|---|
| `poneglyph` | Backend (Python/FastAPI/Temporal) |
| `poneglyph-webapp` | Frontend (Next.js/TS) |
| `logpose` | Infra (Terraform/Helm/ArgoCD) |

**Never hardcode filesystem paths.** Resolve each repo's clone dynamically: start from the repo
you're invoked in (`git rev-parse --show-toplevel`), and find sibling clones by matching the
GitHub remote (`gh repo list`, or look in the parent directory of the current clone) — ask if a
clone can't be located. Work in the per-PR **worktree** (relative to the repo root, by convention
`.claude/worktrees/<date>-<branch>`) whose branch matches the PR head; if none exists, create one
off `origin/<branch>`.

---

## Phase 0 — Discover the relevant PRs

1. For each repo with a local clone, list the user's open PRs:
   `gh pr list --repo <owner>/<repo> --author @me --state open --json number,title,headRefName,isDraft,url`
2. **Relevance** = open + authored by the user + matches the args (PR refs / keyword) if given.
   When no args: treat all open PRs by the user that have a local worktree as relevant; if PRs
   from different repos share a feature (same branch name / linked feature), treat them as one
   group and process the **dependency order** (backend/proto before the webapp that pins it).
3. Print the resolved PR list (repo, number, branch, draft?) and the order. Then process each.

If zero relevant PRs are found, say so and stop.

---

## Phase 1 — Rebase + update the remote PR (per PR, in dependency order)

In the PR's worktree:

1. **Clean tree first.** `git status --porcelain` — if dirty, commit or stash deliberately (never rebase over uncommitted work blindly).
2. **Fetch + measure.** `git fetch origin main` then `git log --oneline HEAD..origin/main | wc -l`. If 0 behind, skip to Phase 2.
3. **Rebase.** `git rebase origin/main`. Resolve each conflict by **understanding both sides**, not picking blindly:
   - **Additive enum/list** (both sides add a value) → keep **both**.
   - **Same function rewritten on both sides** → reconcile to one definition that satisfies both
     features; verify the call sites match the kept signature. Read the surrounding code.
   - **Duplicate symbol introduced by the merge** (e.g. both sides added `get_x`) → keep the
     more complete/correct one (privileged-scope-aware, typed), delete the duplicate (F811).
   - **`buf.gen.yaml` proto pin** (webapp) → set the ref to the **current backend PR head SHA**
     (push the backend first so the SHA is fetchable); a branch is a superset of main + its commits,
     so a single pin covers everything. After resolving, `pnpm buf:generate` and confirm `lib/gen`
     is unchanged (committed bindings already match) or commit the regen.
   - **Alembic migrations** — main almost always added migrations past your base. If your migration
     revision numbers now **collide** (`alembic heads` warns "present more than once"), **re-chain**:
     find main's true head (`alembic heads` should be ONE), renumber your migrations to sit on top
     (rename files + `revision`/`down_revision` + docstring `Revision ID:`/`Revises:`), confirm a
     single linear head. Use `review-data-model` for the integrity checks.
4. **Validate before pushing** (the repo's pre-push gate — see AGENTS.md §9 for backend):
   - Backend: `ruff check src/ tests/` · `ruff format --check src/ tests/` · `mypy <touched>` · `pytest -m unit` · `alembic heads` = single head.
   - Webapp: `npx tsc --noEmit` (clear stale `.next` first if it references removed routes) · `eslint <changed>` · `pnpm build`.
5. **Push** `git push --force-with-lease <https-remote> <branch>` (force-with-lease because the rebase rewrote history). This updates the remote PR.
6. If the local test DB drifted from the re-chained migrations, `alembic upgrade head` (staff/idempotent migrations no-op cleanly); never wipe a seeded local DB to "fix" alembic state.

---

## Phase 2 — Quality checklist (run on the PR diff; FIX, don't just report)

Compute the diff once: `git diff --name-only origin/main...HEAD`. Read each changed file in full.
For each item below, **find violations introduced by this PR and fix them**, then re-validate.
Invoke the named sub-skill for the deep version of the check.

| # | Check | What to do | Deep skill / ref |
|---|---|---|---|
| **2.1** | **Pydantic-first** | Any data crossing a function/service/RPC boundary as a loose `dict`/tuple/bare-primitive bag, or `dict[str, Any]` return, becomes a Pydantic model (frozen view for reads). Don't model-ize a single-row read or a logic-free helper. | AGENTS.md §3, `python-backend-design` |
| **2.2** | **Function length** | Any function > **120 lines** (or with 3+ nesting levels / two unrelated jobs) → extract small, named, single-purpose helpers; keep a thin orchestrator that reads top-to-bottom. Don't fragment a clear linear sequence of well-named calls. | AGENTS.md §5 |
| **2.3** | **Feature grouping + config from config** | Functions introduced by this PR that belong to one feature live together (one module / a deps-holding class), not scattered by layer. **Constants/tunables come from the config layer** (`pydantic-settings` / `system_settings` / `domain_config`), never hardcoded inline. | AGENTS.md §2, `python-backend-design` |
| **2.4** | **No per-vertical leak in generic code** | Generic/core code (search, retrieval, RAG, graph, ranking, prompts, tool/field names, docstrings, fallbacks) carries **zero** business-domain nouns. Vertical strings are read from `domain_config`; graph labels/edge types are caller-supplied params, not literals. Grep the diff for domain nouns; route each hit through config or delete it. | `keeping-core-domain-neutral` |
| **2.5** | **Typed signatures** | Every public signature has explicit typed inputs and a typed output — Pydantic models where a shape exists, `str \| None` not `Optional`, no `Any` creep, no `dep: T \| None = None` lie for an always-injected dep. Validate at the edge; trust the interior. | AGENTS.md §3 & §6 |
| **2.6** | **Migration integrity** | If the diff touches `alembic/versions/`: every `*_id` column has a `REFERENCES` clause with explicit `ON DELETE`; FKs reference **existing entities by id (UUID/PK), never by name/string**; FK type matches the referenced PK type; `DateTime(timezone=True)`; no duplicate table/column that already exists elsewhere; single linear alembic head. **Verify against the live schema (`information_schema`), not assumptions.** | `review-data-model` |
| **2.7** | **Data-model alignment** | New ORM models / columns match existing codebase conventions (naming, nullability style, vertical_id NOT NULL + index for domain tables keyed on account, `users` is identity-only — features key on `account`/`node`). No column derivable from a JOIN unless justified. | `review-data-model` |

After fixing, re-run the Phase 1.4 validation gate for the affected repo.

---

## Phase 3 — Push fixes + watch CI (non-blocking)

1. Commit each logical fix with a clear message; push (`force-with-lease` only if you rebased again, else a normal push).
2. **Do not block** polling CI in the main session. Dispatch a **background subagent** (`Agent` with `run_in_background: true`) per PR to poll `gh pr checks <n>`, root-cause + fix + push any red check until green (skipped draft jobs are fine), and notify on completion.
3. Surface every PR URL.

---

## Output — validation report (mandatory)

Per PR: a table of `rebase / conflicts / 2.1–2.7 / lint / format / types / unit / build` → PASS/FAIL/FIXED/SKIPPED with a one-line reason, the commits pushed, and the PR URL. Free-form "done" is not acceptable.

## Guardrails
- **One stack, current branch** for any local run; kill stale servers before spinning up; only `localhost`/`*.orb.local`, never GKE.
- **Keep PRs focused** — fixes must trace to the PR's purpose or be a quality-checklist fix on code the PR already touches; don't bundle unrelated refactors.
- **Root-cause only** — no "quick vs proper" menus; ship the proper fix.
- **Pre-launch** — full refactor in one PR; no shims/feature-flags/`_old` columns.
