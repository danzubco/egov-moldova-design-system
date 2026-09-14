# AI Documentation Alignment — Implementation Plan

**Goal:** make the repo's AI documentation load, tell the truth about the code, and stay
true — fix what is wrong, remove what is dead, document what is missing, and add the checks
that stop the same drift from coming back.

**Findings and evidence:** [`2026-09-14-ai-docs-audit-report.md`](2026-09-14-ai-docs-audit-report.md)
(F1–F14). Every task below cites the finding it resolves. Read the report first — this plan
does not repeat its evidence.

**Scope:** documentation and AI configuration only — `CLAUDE.md` (new), `AGENTS.md` ×3,
`_agents/` ×3, `.claude/**`, `PRINCIPLES.md`, `PRODUCT.md`, `SECURITY.md`, `STACK.md`,
`DESIGN.md`, `INTEGRATION.md`, `CONTRIBUTING.md`, `README.md`, `.specs/`, `.impeccable/`.
Two tasks touch tooling (Phase 6 and Decision D1) and are gated on an explicit decision.
No component source, tokens or generated files change, except one doc comment
(`src/legacy/file-upload-helper.ts:8`).

**Tech stack touched:** Markdown, `.claude/settings.json`, `package.json` scripts (Phase 6
only), GitHub Actions (Phase 6 only, upstream repo).

---

## Original request

> Analyse the project's AI documentation — `AGENTS.md` at several levels, `.claude/`,
> `_agents/` at several levels, `PRINCIPLES`, `PRODUCT`, `SECURITY`, `STACK`, `.impeccable/`.
> Find what gets in the way of development, what conflicts with reality, what can be removed,
> what conflicts internally or is weaker than it should be. Report on the quality of the AI
> documentation: how far it contributes to a quality result and how far it hinders, and what
> can be corrected or adjusted. Look at it as a senior developer who takes this project over
> for issue fixing and further development: what would you do.
>
> Then: write a well-thought-out plan in `.claude/plans/` to align the docs with the project —
> clean up what is not needed, correct what is no longer true, cover the gaps and the defects
> found, and everything recommended — including this request, the findings, the report and
> the justifications.

---

## Summary of what was found

| # | Finding | Impact |
| --- | --- | --- |
| F1 | No `CLAUDE.md` → Claude Code never loads `AGENTS.md` (root or scoped) | Blocker — every rule is inactive by default |
| F2 | Docs promise a Husky pre-commit hook deleted in `0d23e93`; upstream CI dropped the stale-generated-files gate; `merge=ours` still on | Blocker — stale generated files merge silently |
| F3 | Package name: docs `@egovmd/mud` (106×), `SECURITY.md` `@egov-moldova/design-system`; real `@egov-moldova/mud` | Consumer-facing errors |
| F4 | Node 22 vs 24, Style Dictionary 4 vs 5, Context7 "configured" but absent, Jest in `.specs` | Agents act on wrong facts |
| F5 | `audit-production` auditor has `Write, Edit` | Audit can alter what it audits |
| F6 | Plan-driven workflow undocumented; Cline Kanban workflow documented but retired | Docs describe the wrong process |
| F7 | Vendored `superpowers` skills drifted; `skill-creator` unrelated | Duplicate triggers, dead links |
| F8 | Broken/ambiguous paths (`src/components/AGENTS.md:43-47`, missing plan, skill links) | Agents load nothing or the wrong file |
| F9 | "Skill corrections" tables target skills that no longer exist; `cor` prefix leftover | Noise and wrong advice |
| F10 | Same fact in 4–5 homes with different values | Root cause of F2/F4 |
| F11 | Orphans: `cross-platform-guide.md`, `.impeccable/`, `.specs/_archive/`; unindexed PRINCIPLES/PRODUCT/STACK/DESIGN | Dead weight / unreachable good content |
| F12 | `.claude/settings.json` holds another machine's paths | Leak + clutter |
| F13 | `test:scripts` not in `check.verify`; nothing checks the docs | Drift returns |
| F14 | Minor: tier note, banners, severity inflation, oversized scoped index, PR-title rule | Quality |

**What is good and must survive the cleanup:** the Stencil/slot/CSS/token/a11y knowledge
in `src/components/_agents/`, `tokens/_agents/`, `stencil-compliance`,
`accessibility-compliance`, `token-creation`, `mud-design`; the executable audit scripts
`scripts/audit/01–14`; the hooks `pre-edit-guard.mjs` and `post-edit-format.mjs`;
`_agents/workflow-rules.md` stop/auto-proceed taxonomy; `STACK.md`.

---

## Decisions required before execution

Each decision changes tooling, a shared workflow, or something other contributors may rely
on, so it is taken by the team, not inside a task.

### D1 — Generated-files safety net (F2)

| Option | Cost | Effect |
| --- | --- | --- |
| **A. Restore the net** — re-add `.husky/pre-commit` (unstage generated paths) + `commit-msg` (commitlint), and re-add the "Verify no stale generated files" step to upstream CI | Two small hook files + one CI step; touches tooling config | Docs become true again; `merge=ours` is safe again |
| B. Remove the promise — drop the hook claims, drop `husky install` from `prepare`, keep `merge=ours`, document "run `yarn build` after every merge touching generated files" | Docs-only | Safety depends on discipline; stale snapshots still possible |
| C. Remove `merge=ours` too, regenerate on conflict | `.gitattributes` change + docs | Conflict markers return on parallel branches |

**Recommendation: A.** The merge strategy was designed with both nets; B keeps the risky half
of the design without its guard. A is the only option where the existing docs become
correct rather than rewritten.

### D2 — Vendored skills `systematic-debugging`, `verification-before-completion` (F7)

| Option | Cost | Effect |
| --- | --- | --- |
| **A. Declare the upstream plugin at project level** — add the `superpowers` plugin to `.claude/settings.json` (`enabledPlugins`, marketplace entry if required), delete the copies, update 12 + 5 references to the namespaced skill names | Settings change + reference sweep; verify the settings keys against current Claude Code docs first | One maintained copy; teammates get it on trust |
| B. Keep the copies, re-sync from upstream, add a provenance header (upstream, version, date) | Re-copy + header | Still two same-named skills when the plugin is installed |
| C. Delete and rewrite the references to point at project docs only | Reference sweep | Loses the debugging/verification discipline for teammates |

**Recommendation: A**, falling back to B if project-level plugin declaration is not
supported by the Claude Code version the team runs.

### D3 — Cline Kanban and parallel-worktree material (F6)

Confirm with the team whether `.claude/kanban/` is still used.
**Recommendation:** if unused since 2026-06-12, move it to `.claude/plans/_archive/kanban/`
and reduce the `AGENTS.md` "Merge driver" narrative to what is true without it.

### D4 — `.impeccable/design.json` (F11)

**Recommendation: delete.** Tokens are the source of truth; a generated colour snapshot with
no consumer only drifts. Regenerate on demand if the tool that produced it is used again.

---

## Global constraints

- **Change scope** (`AGENTS.md` rule 11): one concern per commit; no repo-wide `yarn format`
  mixed in. Run `git diff --stat main...HEAD` before the PR.
- **Branch**: all phases land on `docs/ai-docs-alignment`, which already exists — created
  from `main` at `fe6d651` (identical to `egov-moldova/design-system` `main` on 2026-09-14)
  with this plan and the report as its first commit. Rebase it on the current `main` before
  starting and before opening the PR. Never work on it from an issue branch.
- **Commits**: Conventional Commits, `docs(agents): …`, `chore(claude): …`, `ci: …`.
- **Never** hand-edit generated files (`src/components.d.ts`, `*/readme.md`,
  `.storybook/custom-elements.json`, `tokens/generated/**`); the pre-edit hook blocks it.
- **One home per fact**: versions live in `STACK.md`; package name lives in `package.json`
  and is quoted, never paraphrased; a rule lives in one file and others link to it.
- **Deletion rule**: before deleting or moving any file, `rg` for inbound references and
  update or remove every hit in the same commit. A deletion that leaves a dangling reference
  is a failed task.
- **Every task ends on its verify command.** A task is done when that command prints the
  expected result, not when the edit is made.

---

## Sequencing

| Phase | Depends on | Can run in parallel with | Risk |
| --- | --- | --- | --- |
| 0 Baseline | — | — | none |
| 1 Load the docs | 0 | — | low |
| 2 Unsafe / false claims | 1, D1 | 3 | medium (D1 touches tooling) |
| 3 Single home for facts | 1 | 2 | low |
| 4 Cleanup | 2, 3, D2–D4 | — | medium (reference sweeps) |
| 5 Fill the gaps | 4 | — | low |
| 6 Keep it true (ratchets) | 5 | — | medium (tooling config) |
| 7 Final verification + PR | 6 | — | low |

Phases 2 and 3 edit overlapping files (`AGENTS.md`); if run in parallel, split by file, not
by finding.

---

## Phase 0 — Baseline

- [ ] **0.1** Bring the branch up to date with `main`.
  ```bash
  # `upstream` = https://github.com/egov-moldova/design-system (add it if missing)
  git fetch upstream && git switch main && git merge --ff-only upstream/main
  git switch docs/ai-docs-alignment && git rebase main
  ```
- [ ] **0.2** Record the baseline counts the later phases must drive down. Save the output
  into the PR description.
  ```bash
  rg -c --hidden -g '!node_modules' -g '!dist' -g '!storybook-static' -g '!CHANGELOG.md' '@egovmd/' . | awk -F: '{s+=$2} END{print "egovmd:", s}'
  rg -n --hidden -g '!node_modules' -g '!.claude/plans' '\.husky/pre-commit' . | wc -l
  rg -n 'Node >=22|>= 22|Style Dictionary 4' AGENTS.md _agents .claude/agents | wc -l
  rg -n '/Users/' .claude/settings.json | wc -l
  ```
  **Expected now:** egovmd 106 · husky 4+ · version claims 3+ · foreign paths 8.

---

## Phase 1 — Make the documentation load (F1)

**Why first:** until this lands, every later fix improves a file Claude Code never reads.

- [ ] **1.1** Create `CLAUDE.md` at the root:
  ```markdown
  @AGENTS.md
  ```
  Import, not symlink: symlinks need Developer Mode on Windows, and the repo ships
  Windows tooling (`.claude/kanban/worktree-init.ps1`).
- [ ] **1.2** Create `src/components/CLAUDE.md` and `tokens/CLAUDE.md`, each containing
  `@AGENTS.md`. Nested `CLAUDE.md` files load on demand when files in that directory are read.
- [ ] **1.3** Update `AGENTS.md:9` ("This file is the single source of truth") to add one
  line: "Claude Code loads it through `CLAUDE.md`; other agents read it directly."
- [ ] **1.4 Verify:** start a new Claude Code session at the root, run `/context`, confirm
  `CLAUDE.md` and `AGENTS.md` are listed under **Memory files**. Read a file in `tokens/`,
  run `/context` again, confirm `tokens/CLAUDE.md` appears.

---

## Phase 2 — Remove unsafe and false claims (F2, F5, F8, F12)

**Why:** these are the claims that cause wrong actions (`git add -A`, an auditor editing,
loading a file that does not exist), not just wrong reading.

- [ ] **2.1 (D1)** Apply the chosen option.
  - **If A:** restore `.husky/pre-commit` and `.husky/commit-msg` from `0d23e93^`
    (`git show 0d23e93^:.husky/pre-commit`), review the restored `GENERATED_PATTERNS` against
    today's generated paths, `chmod +x`; replace `husky install` in `prepare` with `husky`
    (Husky 9 syntax) after confirming the installed major; propose the CI step upstream
    (Phase 6.3). Docs stay as they are.
  - **If B:** edit the five locations to state the real guard:
    `AGENTS.md:157` (table row), `_agents/anti-patterns.md:24`,
    `.claude/commands/migrate-component.md:135`,
    `.claude/skills/parallel-aux-tasks/SKILL.md:179`, `CONTRIBUTING.md:221,237`; drop
    `husky install` from `prepare`.
  - **Verify (A):** `git add src/components.d.ts && git commit --dry-run` shows it unstaged;
    a commit message `bad message` is rejected.
  - **Verify (B):** `rg -n '\.husky' AGENTS.md _agents .claude CONTRIBUTING.md` → 0 hits.
- [ ] **2.2 (F5)** `.claude/agents/audit-production.md:4` — remove `Write, Edit` from `tools:`.
  **Verify:** `rg -n '^tools:' .claude/agents/audit-production.md` shows no `Write`/`Edit`.
- [ ] **2.3 (F4)** Context7: either add it to `.mcp.json` (team decision — it is an external
  service) or remove the claims in `AGENTS.md:7` and `_agents/mcp-tools.md:17` and the
  `mcp__context7__*` entries from `new-component.md`, `custom-component.md`,
  `redesign-component.md` (and `.claude/agents/README.md`, `.claude/commands/README.md`).
  **Recommendation:** remove — the `mcp__context7__*` names do not match how Context7 is
  commonly installed (as a plugin, with a different tool prefix), so the grant silently
  resolves to nothing in both cases.
  **Verify:** `rg -n 'context7' AGENTS.md _agents .claude/agents .mcp.json` matches the choice.
- [ ] **2.4 (F8)** `src/components/AGENTS.md:43-47` — rewrite paths to `../../_agents/<file>.md`
  and label the table "Root `_agents/` (paths relative to this file)".
- [ ] **2.5 (F8)** `tokens/AGENTS.md` — remove the reference to the missing
  `analizeaza-structura-la-fisierul-breezy-tower.md`; keep the instruction it supported
  ("author `tokens/core/effects.tokens.json` before `--apply`") as a plain rule.
- [ ] **2.6 (F8)** `.claude/skills/audit-component/SKILL.md:698,875` → `../../../_agents/anti-patterns.md`;
  `.claude/skills/mud-design/SKILL.md:149` → fix or remove `reference/`.
- [ ] **2.7 (F12)** `.claude/settings.json` — delete the 8 allow entries containing
  `/Users/vitalie/`. **Verify:** `rg -n '/Users/' .claude/settings.json` → 0;
  `node -e 'require("./.claude/settings.json")'` parses.
- [ ] **2.8 Verify phase:** every relative Markdown link in `AGENTS.md`, `src/components/AGENTS.md`,
  `tokens/AGENTS.md` resolves (manual until Phase 6 adds the script):
  ```bash
  for f in AGENTS.md src/components/AGENTS.md tokens/AGENTS.md; do
    d=$(dirname "$f"); rg -o '`(\.\./)*_agents/[^`]+\.md`' "$f" | tr -d '`' | while read p; do
      [ -f "$d/$p" ] || [ -f "$p" ] || echo "MISSING in $f: $p"; done; done
  ```
  **Expected:** no `MISSING` lines.

---

## Phase 3 — One home per fact (F3, F4, F10)

**Why:** F2 and F4 are both "one fact changed, the copies did not". Fixing values without
removing copies re-creates the defect at the next upgrade.

- [ ] **3.1** `STACK.md` becomes the only place versions are written. Add at its top:
  "Versions below are copied from `package.json`; when they disagree, `package.json` wins
  and this table is fixed in the same PR."
- [ ] **3.2** `AGENTS.md:5` — replace the version list with the stack *names* and
  "versions: see `STACK.md`". Keep the port (6007) and the test command, which are
  operational, not versions.
- [ ] **3.3** `.claude/agents/audit-production.md:65` — Node claim → "per `package.json`
  `engines.node`".
- [ ] **3.4 (F3)** Package name sweep in documentation: `@egovmd/mud` → `@egov-moldova/mud`,
  `@egovmd/mud-web-components` → `@egov-moldova/mud-web-components`. Files: the list from
  Phase 0.2 minus `CHANGELOG.md` (history stays as written) and minus `.claude/plans/`
  (historical plans stay as written). Includes the 40+ demo HTML `<title>` strings and the
  doc comment in `src/legacy/file-upload-helper.ts:8`. Use a scripted substitution only with a
  per-file before/after count check; spot-read `INTEGRATION.md` afterwards.
  **Verify:** Phase 0.2 `egovmd` count → only `CHANGELOG.md` and `.claude/plans/` remain.
- [ ] **3.5 (F3)** `SECURITY.md` supported-versions table → `@egov-moldova/mud`,
  `@egov-moldova/mud-web-components`, `@egov-moldova/mud-react` (state its real publish
  status: check `npm view @egov-moldova/mud-react version`).
- [ ] **3.6 (F4)** `.specs/PROJECT-SPECIFICATION.md:23` — Jest 30.x → Vitest + `@stencil/vitest`
  (version: see `STACK.md`). `.claude/kanban/README.md:48` — Jest → Vitest (or archived via D3).
- [ ] **3.7 (F4)** `INTEGRATION.md:3`, `README.md:111`, `web-components/README.md:17` — read
  each Webpack/Rollup/esbuild mention; keep it if it is consumer-bundler guidance, fix it if it
  claims this repo builds with it.
- [ ] **3.8 (F10)** TypeScript strict rules: keep the full text only in
  `_agents/typescript-strict.md`; in `AGENTS.md` rule 6, `_agents/anti-patterns.md`,
  `src/components/_agents/component-structure.md` and `e2e-testing.md` leave the one-line
  imperative plus a link.
- [ ] **3.9 (F10)** Figma-first: full rule only in `_agents/workflow-rules.md`; `AGENTS.md`
  keeps the 3-line STOP (it is the rule most worth having always loaded) and rule 1 links to
  it; `PRINCIPLES.md` and `.claude/commands/README.md` link instead of restating.
- [ ] **3.10 Verify phase:**
  ```bash
  rg -n 'Node >=22|>= ?22|Style Dictionary 4|Jest 30' AGENTS.md _agents .claude/agents .specs STACK.md
  ```
  **Expected:** 0 hits.

---

## Phase 4 — Remove what is dead (F7, F9, F11, D2–D4)

**Why:** dead files are not free — they rank in search, get linked by agents, and produced a
third of the stale-claim hits in the audit.

- [ ] **4.1 (F7)** Delete `.claude/skills/skill-creator/`; remove its line from
  `.claude/skills/LOCAL-SETUP.md`. **Verify:** `rg -n 'skill-creator' . -g '!.claude/plans'` → 0.
- [ ] **4.2 (D2)** Apply the chosen option for `systematic-debugging` and
  `verification-before-completion`. For option A, update every inbound reference:
  ```bash
  rg -l 'systematic-debugging|verification-before-completion' -g '!.claude/plans' -g '!.claude/skills/systematic-debugging' -g '!.claude/skills/verification-before-completion' .
  ```
  (17 files at audit time: `AGENTS.md`, `_agents/skills-and-workflows.md`,
  `.claude/skills/LOCAL-SETUP.md`, `.claude/commands/{fix-visual-bug,pre-pr-check}.md`,
  `.claude/agents/{README,custom-component,new-component,redesign-component,refactor-component}.md`,
  `.claude/skills/parallel-aux-tasks/SKILL.md`, `.specs/COMPONENT-DEVELOPMENT-GUIDE.md`.)
  **Verify:** the command above prints only references to the chosen canonical names.
- [ ] **4.3 (F11)** Delete `_agents/cross-platform-guide.md` (zero inbound references). If any
  Windows-specific command in it is still needed, move that single row to
  `_agents/environment-commands.md` first.
- [ ] **4.4 (D4)** Delete `.impeccable/`; remove the mention in
  `docs/screenshots/inputs-build-progress.md`.
- [ ] **4.5 (F11)** Delete `.specs/_archive/`. **Verify:** `rg -n '_archive' .specs` → 0.
- [ ] **4.6 (F9)** Corrections tables:
  - `src/components/_agents/component-structure.md:172` and `css-architecture.md:98` — rename the
    column "Skill Says" → "Model assumes"; delete rows that no longer describe a real,
    repeated wrong assumption; fix `:178` ("Use `cor` prefix" → "Use the `mud` prefix:
    `@Event() mudButtonClick`").
  - `tokens/AGENTS.md:172` — delete (duplicate of `tokens/_agents/token-structure.md:149`);
    rework the latter the same way.
  **Verify:** `rg -n 'Skill Says|stencil-atomic|stenciljs skill|`cor` prefix' src tokens _agents` → 0.
- [ ] **4.7 (D3)** If Kanban is retired: `git mv .claude/kanban .claude/plans/_archive/kanban`;
  update `.gitattributes`, `.claude/settings.json`, `.claude/agents/redesign-component.md`,
  `AGENTS.md` references found by `rg -n 'kanban' -g '!.claude/plans'`.
- [ ] **4.8** Archive closed plans: `git mv` every `.claude/plans/*.prompt.md` and
  `claude-code-automation-improvements-implementation-plan,prompt.md` into
  `.claude/plans/_archive/`, after confirming each is merged or abandoned (`git log --oneline`
  for the feature it describes). Dated plans (`2026-*`) stay.
- [ ] **4.9 Verify phase:** re-run the Phase 2.8 link check and
  `rg -n '_agents/cross-platform-guide|\.impeccable|skill-creator' -g '!.claude/plans' .` → 0.

---

## Phase 5 — Fill the gaps (F6, F11, F14)

**Why:** the plan-driven workflow is how issues actually get fixed here; good policy documents
exist but agents never reach them.

- [ ] **5.1 (F6)** New `_agents/planning.md` (≤80 lines), scope banner first. Content:
  - When a plan is required: any change touching a public contract (props, events, slots,
    parts, token names), more than one component, or with more than one viable approach.
    Not required: single-file fixes with one obvious approach.
  - Location and name: `.claude/plans/YYYY-MM-DD-<slug>.md`; closed plans move to
    `.claude/plans/_archive/`.
  - Required sections: Goal · Spec/issue link · Options (table) · Decision · Global
    constraints · Tasks with checkboxes and a verify command each · Not verified.
  - Plans are written in English and committed with the change they drive.
  - Keep it tool-agnostic: describe the artefact, not any particular assistant's workflow.
- [ ] **5.2 (F6, F11)** `AGENTS.md` Subfile Index — add rows:
  `_agents/planning.md` (before non-trivial work), `PRINCIPLES.md` (code-shape decisions),
  `PRODUCT.md` (users, tone, UX trade-offs), `STACK.md` (versions and rejected choices),
  `DESIGN.md` (visual language).
- [ ] **5.3 (F14)** Move `AGENTS.md` "Merge driver for auto-generated files" (lines ~151-185)
  into `_agents/generated-files.md`; leave a 3-line summary + link in `AGENTS.md`.
  Rewrite the "parallel agent worktrees (Cline Kanban …)" framing per D3.
- [ ] **5.4 (F14)** Move the Tokenhaus sync runbook from `tokens/AGENTS.md` into
  `tokens/_agents/tokenhaus-sync.md`; leave the two commands and the "preview with
  `--apply --dry-run` first" rule in the index.
- [ ] **5.5 (F2)** `_agents/verification-git.md` — add a short "Staging" rule that holds under
  either D1 option: stage paths explicitly (`git add <paths>`), never `git add -A` / `.`;
  generated files are regenerated with `yarn build`, never staged by hand.
- [ ] **5.6 (F11)** `_agents/continuous-improvement.md` — reduce to what is practised: when the
  same component needs a third fix on the same behaviour, add a regression test for the
  contract (as done for issues #10 and #17) and record the missing rule in the relevant
  `_agents/` file in the same PR. Drop the unused "Workflow Gap Analysis" template.
- [ ] **5.7 (F14)** `_agents/verification-git.md:193` — state that `ISSUE TYPE :: KEY ::` applies to
  PR titles only, or remove it if the team no longer uses it.
- [ ] **5.8 (F14)** `.claude/agents/custom-component.md` — one line on why it runs on a lower
  tier than `new-component` (no Figma extraction or token diff).
- [ ] **5.9 (F14)** Add a one-line scope banner ("Governs … Load when …") to detail files that
  lack one; start with the files indexed from the three `AGENTS.md` files.
- [ ] **5.10 Verify phase:** `wc -l AGENTS.md src/components/AGENTS.md tokens/AGENTS.md` —
  each ≤ 150 lines after the moves; Phase 2.8 link check passes.

---

## Phase 6 — Keep it true (F13) — tooling, gated

**Why:** every defect in the report was a doc that stayed the same while the code moved. Only
a check that runs without anyone remembering it prevents the next round.

- [ ] **6.1** `scripts/docs/check-ai-docs.mjs` (Node, no dependencies), with a spec in
  `scripts/__tests__/check-ai-docs.spec.mjs`. It fails on:
  1. a relative Markdown link or backticked `_agents/…md` path in `AGENTS.md`, `CLAUDE.md`,
     `_agents/**`, `.claude/{agents,commands,skills}/**` that does not resolve;
  2. a Node version claim that disagrees with `package.json` `engines.node`;
  3. a package name matching `@egovmd/` outside `CHANGELOG.md` and `.claude/plans/_archive/`;
  4. a `yarn <script>` mentioned in those docs that is not in `package.json` `scripts`;
  5. an absolute `/Users/` or `C:\\Users\\` path in `.claude/settings.json`.

  Report `file:line` per hit; exit 1 on any.
- [ ] **6.2** Wire it: `package.json` `"docs:check": "node scripts/docs/check-ai-docs.mjs"`; add
  `docs:check` and `test:scripts` to `check.verify` dependencies (`package.json:398`).
  **Verify:** `yarn check` runs both; introduce a broken link locally → `yarn check` fails.
- [ ] **6.3** Upstream CI (`egov-moldova/design-system`, `.github/workflows/ci.yml`): propose a
  PR adding `yarn docs:check` and `yarn test:scripts` to the Test job, and — if D1 = A — the
  "Verify no stale generated files" step after Build. This is a change to the shared pipeline:
  open it as its own PR, not inside the docs PR.
- [ ] **6.4 Verify phase:** `yarn docs:check` exits 0 on the branch; the Phase 0.2 counts are
  0 (or only the documented exceptions).

---

## Phase 7 — Final verification and PR

- [ ] **7.1** `yarn lint` and `yarn test` pass (docs-only changes must not break them;
  Prettier runs on Markdown).
- [ ] **7.2** `git diff --stat main...HEAD` — only files listed in this plan.
- [ ] **7.3** New Claude Code session: `/context` shows `CLAUDE.md` + `AGENTS.md`; ask it
  "What Node version and which package name does this repo use?" — the answer must be
  `>=24 <25` and `@egov-moldova/mud`, sourced from `STACK.md` / `package.json`.
- [ ] **7.4** PR description: link this plan and the report; paste Phase 0.2 before/after
  counts; list the decisions taken (D1–D4) and who took them.
- [ ] **7.5** Move this plan and the report to `.claude/plans/_archive/` in the PR that closes
  the last phase.

---

## Out of scope

- Component source, token values, generated files.
- Rewriting the domain skills (`stencil-compliance`, `accessibility-compliance`,
  `audit-component`) beyond link fixes — they are the valuable part; size reduction of
  `audit-component/SKILL.md` (875 lines) is a separate follow-up.
- Changing the design-to-code pipeline itself (Figma-first, pixel-perfect QA).

## Not verified (carried from the report)

- ~30 WARN-level dead paths and ~25 dead `yarn` commands — Phase 6.1 enumerates them.
- `DESIGN.md` vs tokens palette parity.
- PR titles against the `ISSUE TYPE :: KEY ::` rule.
- Whether anyone still uses `.claude/kanban/` or relies on the vendored skills (D2, D3).
- Project-level plugin declaration keys in `.claude/settings.json` (D2 option A) — check
  current Claude Code settings documentation before implementing.
