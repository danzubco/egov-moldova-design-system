# AI Documentation Audit — `@egov-moldova/mud` — 2026-09-14

**Scope**: every document an AI coding agent reads in this repo — `AGENTS.md` (root,
`src/components/`, `tokens/`), the three `_agents/` folders, `.claude/` (agents, commands,
skills, hooks, kanban, plans, settings), `PRINCIPLES.md`, `PRODUCT.md`, `SECURITY.md`,
`STACK.md`, `DESIGN.md`, `.impeccable/`, `.specs/`.

**Companion plan**: [`2026-09-14-ai-docs-alignment.md`](2026-09-14-ai-docs-alignment.md) —
every finding below maps to a task there.

**Method**: automated scans (dead relative links, index entries pointing at missing files,
version and tool claims compared against `package.json`, foreign absolute paths in
settings), then manual review of each flagged item against the code, git history, the npm
registry and the upstream `egov-moldova/design-system` repository. Scanner hits that turned
out to be false positives are listed as such and are not counted as defects.

**Baseline** (automated scan, before triage): 46 FAIL · 171 WARN · 67 INFO across 6 checks.

---

## 1. Verdict

The domain knowledge is the strongest part of this documentation set and is worth keeping:
the Stencil, slot, CSS, token and accessibility guidance in `src/components/_agents/`,
`tokens/_agents/` and the `stencil-compliance` / `accessibility-compliance` / `token-creation`
skills. The component-audit tooling (`scripts/audit/01–14`) is real executable verification
that the docs call, not prose.

The docs are hurting development in four places:

1. **Claude Code does not load any of it automatically** — there is no `CLAUDE.md`.
2. **They state a safety net that no longer exists** (the Husky pre-commit hook and, upstream,
   the stale-generated-files CI gate), and the merge strategy that depends on it is still on.
3. **Package identity and versions contradict reality** in the most-read places.
4. **The docs describe a workflow the team no longer uses** (Cline Kanban, parallel worktrees)
   and do not describe the one it does use (dated plans in `.claude/plans/`).

On top of that sits cleanup debt: vendored third-party skills that have drifted from their
upstream, orphaned files, "correction" tables aimed at skills that no longer exist, and
closed plans that are responsible for a third of the stale-claim hits.

---

## 2. Findings — ranked by impact

Each finding: **symptom** · **located cause** · **evidence** · **why it matters**.

### F1 — Project instructions are never loaded by Claude Code (BLOCKER)

- **Cause**: no `CLAUDE.md` at the root, in `src/components/` or in `tokens/`.
- **Evidence**: Claude Code documentation, *How Claude remembers your project → AGENTS.md*:
  "Claude Code reads `CLAUDE.md`, not `AGENTS.md`. If your repository already uses
  `AGENTS.md` for other coding agents, create a `CLAUDE.md` that imports it." Nested
  `CLAUDE.md` files load on demand when files in that directory are read — the same holds
  for the scoped `AGENTS.md` files only through such an import.
- **Why it matters**: `AGENTS.md:9` declares itself "the single source of truth. It overrides
  all skill files." In practice it is invisible unless a command/agent/skill links to it, so
  every rule marked "Always Active" is only active by accident. Every other finding in this
  report is amplified by this one.

### F2 — Documented pre-commit safety net was deleted; the merge strategy relying on it is still active (BLOCKER)

- **Cause**: `.husky/pre-commit` and `.husky/commit-msg` were removed in `0d23e93`
  (2026-06-15). `package.json` `prepare` still runs `husky install`.
- **Stale claims** ("the pre-commit hook auto-unstages generated files, so `git add -A` is
  harmless") in nine files:
  - `AGENTS.md:157`
  - `_agents/anti-patterns.md:24`
  - `scripts/audit/CLEANUP.md:84`
  - `.claude/commands/migrate-component.md:135`
  - `.claude/skills/parallel-aux-tasks/SKILL.md:179`
  - `.claude/agents/custom-component.md:187`, `new-component.md:242`,
    `redesign-component.md:304`, `refactor-component.md:210`
  - plus `CONTRIBUTING.md:221,237` (commitlint via Husky `commit-msg`)
- **The post-merge hint is dead too.** `prepare` runs `husky install`; Husky `9.1.7` marks it
  deprecated but still sets `core.hooksPath=.husky/_`. With `core.hooksPath` set, git ignores
  `.git/hooks/`, which is exactly where `scripts/git/setup-merge-drivers.sh` writes the
  `post-merge` hint `AGENTS.md` describes. Every contributor who ran `yarn install` with scripts
  has neither the pre-commit net nor the post-merge hint.
- **The deleted hook differed from the docs.** It also ran `yarn typecheck && yarn lint &&
  yarn test` on every commit, and it deliberately did not unstage `src/components.d.ts`, which
  `AGENTS.md` lists as never-stage.
- **Second net, also gone upstream**: the local `.github/workflows/ci.yml` still has the
  `Validate (PR)` → "Verify no stale generated files" step. `egov-moldova/design-system`
  `main` has a rewritten CI (Lint / Typecheck / Test / Dependencies audit / Build) with no
  `git diff --exit-code` step, no `test:scripts`, and no component audit.
- **Why it matters**: `.gitattributes` keeps `merge=ours` on `src/components.d.ts`,
  `.storybook/custom-elements.json`, per-component `readme.md` and `tokens/generated/**`.
  With no hook and no CI gate, a merge silently keeps the current branch's generated files
  and nothing detects a stale snapshot. Agents that believe the docs will run `git add -A`.
  Commit-message conventions are also unenforced while `CONTRIBUTING.md` says they are.

### F3 — Package identity contradicts the published package

- **Reality**: `package.json` → `@egov-moldova/mud`; npm registry → `@egov-moldova/mud@1.1.9`.
  `@egovmd/mud` and `@egov-moldova/design-system` do not exist on npm.
- **Docs say** `@egovmd/mud` / `@egovmd/mud-web-components` — 106 occurrences outside
  `CHANGELOG.md`, including `AGENTS.md` (4), `INTEGRATION.md` / `INTEGRATION.html` (6 each),
  `PRODUCT.md`, `PRINCIPLES.md`, `STACK.md`, `TESTING.md`, `.specs/*`,
  `_agents/environment-commands.md`, `_agents/verification-git.md`, 40+ demo HTML pages and
  one source doc comment (`src/legacy/file-upload-helper.ts:8`).
- **`SECURITY.md`** lists a third, different set: `@egov-moldova/design-system`,
  `-web-components`, `-react`.
- **Why it matters**: `INTEGRATION.md` and `SECURITY.md` face consumers and security
  reporters; an agent writing install or import instructions copies the wrong name.

### F4 — Versions and tool claims contradict `package.json`

| Claim | Where | Reality |
| --- | --- | --- |
| Node `>=22` | `AGENTS.md:5`, `_agents/cross-platform-guide.md` | `engines.node` `>=24.0.0 <25.0.0` |
| Style Dictionary `4.x` | `AGENTS.md:5`, `STACK.md:32` (rationale heading) | `style-dictionary ^5.4.1` |
| Node `>= 22` | `.specs/PROJECT-SPECIFICATION.md:25` | `>=24 <25` |
| Context7 "configured in `.mcp.json`" | `AGENTS.md:7`, `_agents/mcp-tools.md:17` | not in `.mcp.json` |
| Jest 30.x for unit + E2E | `.specs/PROJECT-SPECIFICATION.md:23` | Vitest 4 + `@stencil/vitest` |
| Jest in the CPU budget | `.claude/kanban/README.md:48` | Vitest |
| Node `>=22` | `.claude/agents/audit-production.md:65` | `>=24 <25` |
| Webpack / Rollup / esbuild | `INTEGRATION.md:3`, `README.md:111`, `web-components/README.md:17` | needs manual check — may be legitimate consumer-bundler guidance |

`STACK.md`'s version table matches `package.json`; its own rationale heading at `:32` does not.

**False positives (not defects)**: the scanner's "Jest" hits in
`.claude/agents/test-writer.md:12`, `src/components/_agents/testing.md:5`,
`src/components/_agents/e2e-testing.md:11`, `src/components/_agents/a11y-testing.md:11`,
`.claude/skills/audit-component/SKILL.md:425` and
`.claude/skills/stencil-compliance/references/functional-api.md:180` are migration notes that
correctly say Jest was retired.

**Why it matters**: when the always-loaded header (once F1 is fixed) disagrees with
`STACK.md`, an agent picks one arbitrarily. Agents granted `mcp__context7__*` tools
(`new-component`, `custom-component`, `redesign-component`) get nothing when the server is
not configured, and the failure is silent.

### F5 — A read-only auditor agent is granted write tools

- **Cause**: `.claude/agents/audit-production.md:4` grants `Write, Edit`; its own description
  (`:3`) promises a report and `:639` says "Do NOT auto-fix".
- **Evidence**: none of its 11 phases writes a file; the four verifier agents
  (`a11y-verifier`, `integration-checker`, `pixel-perfect-verifier`, `token-validator`)
  correctly restrict themselves to `Read, Glob, Grep, Bash`.
- **Why it matters**: an auditor that can edit what it audits can "fix" a finding instead of
  reporting it, and the report stops being evidence.

### F6 — The workflow actually used is undocumented; a retired one is documented

- **Used**: 36 commits touching `.claude/plans/`; issue branches such as
  `fix/issue-17-accordion-item-slotted-disabled` are driven end to end by a dated plan with
  an Options table, a Decision section, Global Constraints and a review log.
- **Not documented**: `AGENTS.md`, `_agents/workflow-rules.md`,
  `_agents/skills-and-workflows.md` and `.claude/commands/*` never mention plans, their
  naming, required sections, or when a plan is required.
- **Documented but stale**: `AGENTS.md:151-185` presents "parallel agent worktrees (Cline
  Kanban + `.claude` orchestrators) where 3–5 components are built simultaneously" as how the
  repo runs; `.claude/kanban/` was last touched 2026-06-12.
- **Good news**: the Figma-first stop does not block bug fixes. `fix-visual-bug` and
  `modify-component` treat the Figma URL as optional, and `fix-visual-bug` covers state and
  interaction bugs.

### F7 — Vendored third-party skills have drifted from their upstream

- `.claude/skills/systematic-debugging/` and `.claude/skills/verification-before-completion/`
  are copies of the `superpowers` plugin's skills of the same name; both differ from the
  current upstream (6.3.0). When the plugin is also installed, two skills with the same name
  and different text compete for the same trigger.
- `verification-before-completion` is referenced from 12 files (agents, commands, `AGENTS.md`,
  `.specs/COMPONENT-DEVELOPMENT-GUIDE.md`), `systematic-debugging` from 5 — they cannot be
  deleted without a replacement.
- `.claude/skills/skill-creator/` is Anthropic's generic skill-authoring skill (Python
  scripts, a license file, six dead links to `FORMS.md`, `DOCX-JS.md`, `OOXML.md`, …).
  Referenced only from `.claude/skills/LOCAL-SETUP.md`. It has nothing to do with the design
  system.

### F8 — Broken or ambiguous references

- `src/components/AGENTS.md:43-47` lists root-level files as `_agents/typescript-strict.md`
  etc. Relative to that file they do not exist; the real paths are `../../_agents/…`.
  `tokens/AGENTS.md` does the same with `_agents/pre-implementation.md`. Six unresolvable
  index paths in total.
- `tokens/AGENTS.md` cites `.claude/plans/analizeaza-structura-la-fisierul-breezy-tower.md`
  (PR C) — the file does not exist.
- `.claude/skills/audit-component/SKILL.md:698,875` → `src/components/_agents/anti-patterns.md`
  (it lives at root `_agents/`).
- `.claude/skills/mud-design/SKILL.md:149` → `reference/` (missing).
- Plus ~30 WARN-level dead paths and ~25 dead `yarn` commands, not individually triaged
  (plan Phase 6 adds a check that lists them).

### F9 — "Skill corrections" tables correct skills that no longer exist

- `src/components/_agents/component-structure.md:172`, `src/components/_agents/css-architecture.md:98`,
  `tokens/AGENTS.md:172`, `tokens/_agents/token-structure.md:149` correct external skills from
  the pre-Claude-Code tooling era (`stenciljs`, `stencil-atomic`), which are not in the repo.
- They carry rename leftovers: `component-structure.md:180` — "Use `cor` prefix:
  `@Event() mudButtonClick`"; `composition-interactive.md:76` — "`@Event()` with `cor` prefix".
- The exception is `_agents/mcp-tools.md:349` ("Skill File Tool Name Corrections"): live files
  still use the old tool spellings it corrects (`figma_get_…`, `playwright_…`) —
  `.claude/commands/audit-accessibility.md` 18 times, `pre-pr-check.md` 3,
  `modify-component.md` 3, `.claude/agents/custom-component.md` 2. That table is still load-bearing
  until those spellings are replaced.
- `tokens/AGENTS.md:172` and `tokens/_agents/token-structure.md:149` are the same table twice.
- **Why it matters**: the table form is valuable when it corrects what a model *assumes*;
  aimed at a vanished skill it is noise, and the leftover prefix is wrong advice.

### F10 — Same fact, several homes, several values

| Fact | Homes |
| --- | --- |
| Node / tool versions | `AGENTS.md:5`, `STACK.md`, `_agents/cross-platform-guide.md`, `.claude/agents/audit-production.md:65` |
| TypeScript strict rules | `AGENTS.md` rule 6, `_agents/typescript-strict.md`, `_agents/anti-patterns.md`, `src/components/_agents/component-structure.md`, `src/components/_agents/e2e-testing.md` |
| Figma-first rule | `AGENTS.md:13-17` and rule 1, `_agents/workflow-rules.md`, `PRINCIPLES.md`, `.claude/commands/README.md` |
| Generated-files safety net | five files (F2) |

**Why it matters**: F2 and F4 are exactly this failure — one fact changed, the copies did not.

### F11 — Orphaned and dead files

| File | Status |
| --- | --- |
| `_agents/cross-platform-guide.md` | zero inbound references; stale Node 22 |
| `.impeccable/design.json` | generated 2026-05-22, never updated, only referenced from `docs/screenshots/inputs-build-progress.md`; 45 hard-coded hex colours that duplicate the tokens and will drift from them |
| `.specs/_archive/AI-ORCHESTRATION-GUIDE.md` | 1,448 lines, zero references |
| `PRINCIPLES.md`, `PRODUCT.md`, `STACK.md`, `DESIGN.md` | good content, but not indexed by `AGENTS.md`; agents never reach them |
| `_agents/continuous-improvement.md` | defines a "Workflow Gap Analysis" triggered by "3+ fix rounds"; untouched since 2026-05-29, never produced — issue #17 had five fix commits on one target without triggering it |

### F12 — Settings carry another machine's paths

- `.claude/settings.json:24,28,31,33,35,37,40,90` allow commands under
  `/Users/vitalie/.nvm/…` and `/Users/vitalie/corlab-projects/age-design/…` (a different
  project). They grant nothing useful here and leak a local layout into a shared file.

### F13 — Verification ratchets are partial

- `test:scripts` (a real regression test tied to issue #10) is not a dependency of
  `check.verify` (`package.json:398`), so `yarn check` does not run it; upstream CI no longer
  runs it either.
- Nothing checks the AI docs themselves (links, version claims, package name), which is why
  F2–F4 and F8 accumulated unnoticed.
- Local `ci.yml` "Unit tests" runs `stencil test --spec` with no Jest installed — superseded
  upstream by `yarn test`; resolve by syncing from upstream, not by editing locally.

### F14 — Minor

- `.claude/agents/custom-component.md` (sonnet) and `new-component.md` (opus) run the same
  build sequence at different model tiers with no stated reason.
- 53 detail files open without a one-line scope/when-to-load banner; 15 skills have no
  failure-modes section; 101 `MUST`/`NEVER`/`⚠️`/`STOP` markers, few stating the cost of
  breaking the rule.
- `tokens/AGENTS.md` (178 lines) carries the full Tokenhaus sync runbook — detail that belongs
  in `tokens/_agents/`, not in a scoped index.
- `_agents/verification-git.md:193` prescribes PR titles `ISSUE TYPE :: KEY :: DESCRIPTION`;
  0 of the last 40 commit subjects follow it (PR titles themselves not checked) — either
  the rule is stale or it applies to PR titles only and should say so.

---

## 3. Per-document verdict

| Document | Keep? | Action |
| --- | --- | --- |
| `AGENTS.md` (root, 185 lines) | Keep | Make it load (F1); fix header facts to cite `STACK.md` (F4, F10); remove Husky claims (F2); move "Merge driver" section (~35 lines) to `_agents/generated-files.md`; index PRINCIPLES/PRODUCT/STACK/DESIGN and plans (F6, F11) |
| `src/components/AGENTS.md` | Keep | Fix cross-reference paths (F8); make it load (F1) |
| `tokens/AGENTS.md` | Keep | Remove missing-plan reference (F8); move sync runbook to `tokens/_agents/` (F14); drop duplicate corrections table (F9); make it load (F1) |
| `_agents/` (root) | Keep most | Delete `cross-platform-guide.md` (F11); fix `anti-patterns.md:24` (F2); fix `mcp-tools.md` Context7 (F4); rewrite or trim `continuous-improvement.md` (F11) |
| `src/components/_agents/`, `tokens/_agents/` | Keep | Rework corrections tables (F9); banners (F14) |
| `.claude/agents/` | Keep | `audit-production` tools (F5); Context7 tool names (F4); tier note (F14) |
| `.claude/commands/`, `.claude/hooks/` | Keep | `migrate-component.md:135` (F2). Hooks are correct as written |
| `.claude/skills/` | Keep domain skills | Decide vendored skills (F7); delete `skill-creator` (F7); fix dead links (F8) |
| `.claude/kanban/` | Decision | Archive if Cline Kanban is retired (F6) |
| `.claude/plans/` | Keep | Archive closed `*.prompt.md` plans to `.claude/plans/_archive/` (8 of the stale-claim FAILs come from them) |
| `.claude/settings.json` | Keep | Remove foreign paths (F12) |
| `PRINCIPLES.md` | Keep | Index it; fix package name |
| `PRODUCT.md` | Keep | Index it; fix package name |
| `STACK.md` | Keep — canonical | Becomes the single home for versions |
| `SECURITY.md` | Keep | Fix package names (F3) |
| `DESIGN.md` | Keep | Index it |
| `.impeccable/design.json` | Delete | Tokens are the source of truth (F11) |
| `.specs/` | Keep live files | Fix Jest claim (F4); delete `_archive/` (F11) |

---

## 4. Not verified

- The ~30 WARN dead paths and ~25 dead `yarn` commands were not triaged one by one.
- `DESIGN.md` palette vs `.impeccable/design.json` vs `tokens/` parity.
- PR titles (only commit subjects) against the `ISSUE TYPE :: KEY ::` rule.
- Whether Webpack/Rollup/esbuild mentions in `INTEGRATION.md` / `README.md` are legitimate
  consumer guidance rather than stale build claims.
- Git behaviour of `parallel-aux-tasks` in `parallel-write` mode.
- Whether any team member still relies on `.claude/kanban/` or the vendored skills.
- This checkout is 17 commits ahead of / 9 behind `egov-moldova/design-system` `main`; the
  upstream diff touches only CI files (`.github/workflows/ci.yml`,
  `.github/workflows/webhook-deploy.yml`, `.github/actions/setup-node-deps/action.yml`), so
  the doc findings hold for both, and CI findings are stated against upstream.
