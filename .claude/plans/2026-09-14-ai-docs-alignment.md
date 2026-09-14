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

## The problem

The AI documentation is not loaded by Claude Code, and where agents do reach it, it states facts that are no longer true — including a safety net that was deleted. Agents act on those facts: wrong package names in consumer docs, `git add -A` trusted to a hook that does not exist, an auditor that can edit what it audits. Nothing in the repo notices when a document drifts from the code.

### Findings

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

Facts either option must account for (verified 2026-09-14):

- **The hook layer is broken today in a second way.** `prepare` runs `husky install`.
  Husky `9.1.7` prints `install command is DEPRECATED` but still sets
  `core.hooksPath=.husky/_`. Once that is set, git ignores `.git/hooks/`, so the
  `post-merge` hint that `scripts/git/setup-merge-drivers.sh` writes into
  `$(git rev-parse --git-common-dir)/hooks` never runs — for anyone who ran `yarn install`
  with scripts enabled. No `.husky/` directory is tracked at all; `.husky/_` (git-ignored shims)
  exists only on machines where `prepare` ran, and with no `.husky/pre-commit` to call, those
  shims run nothing. So today neither the pre-commit net nor the post-merge hint exists.
- **The deleted hook was heavier than the docs say.** `git show 0d23e93^:.husky/pre-commit`:
  besides unstaging generated files it ran `yarn typecheck && yarn lint && yarn test` on
  every commit, and it deliberately did NOT unstage `src/components.d.ts` (consumers depend
  on it), while `AGENTS.md` lists that file as never-stage. It also sources `_/husky.sh`,
  the Husky 8 form that Husky 9 deprecates.
- **Option A therefore means:** `prepare` → `husky && node scripts/git/setup-merge-drivers.mjs`;
  move the post-merge hint to a tracked `.husky/post-merge` (the `.git/hooks` copy can never
  run under `core.hooksPath`); a Husky 9 `.husky/pre-commit` that only unstages generated
  paths (the team decides separately whether commits also run typecheck/lint/test); one
  `components.d.ts` policy, stated identically in the hook and in the docs.
- **Option B therefore means:** remove Husky from `prepare` as well, otherwise
  `core.hooksPath` keeps disabling the post-merge hint; contributors who already ran
  `yarn install` need `git config --unset core.hooksPath` once (put it in the PR description).

### D2 — Vendored skills `systematic-debugging`, `verification-before-completion` (F7)

| Option | Cost | Effect |
| --- | --- | --- |
| **A. Declare the upstream plugin at project level** — add the `superpowers` plugin to `.claude/settings.json` (`enabledPlugins`, marketplace entry if required), delete the copies, update 12 + 5 references to the namespaced skill names | Settings change + reference sweep. Claude Code settings docs confirm `extraKnownMarketplaces` in the shared project file applies once a teammate trusts the folder; confirm the `enabledPlugins` key and its value shape in the settings reference before writing it | One maintained copy; teammates get it on trust |
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

## Acceptance bar

The plan is done when all of these hold on `docs/ai-docs-alignment` before the PR.

Zero-tolerance:

- Instructions load — graded by `/context` in a fresh Claude Code session at the root: it lists
  `CLAUDE.md` and `AGENTS.md` under Memory files, and after reading a file under `tokens/` it
  also lists `tokens/CLAUDE.md`.
- Guard described = guard running (`sh .husky/pre-commit` probe, Phase 2.1) — under D1 = A all
  four probe expectations hold; under D1 = B the Phase 0.2 husky count prints `0`.
- No dangling reference (`node scripts/docs/check-ai-docs.mjs; echo $?` prints `0`) in the four
  rule classes the checker covers: relative links / `_agents/` paths, Node major, `@egovmd/`,
  absolute paths in settings. This bar does **not** cover
  `yarn <script>` mentions or test-runner prose; those are handled once by hand in 5.11 and
  carry no ratchet (stated in Out of scope).
- No read-only agent can write (`rg -n '^tools:.*(Write|Edit)'` over the five read-only agents)
  — full command: `rg -n '^tools:.*(Write|Edit)' .claude/agents/{a11y-verifier,integration-checker,pixel-perfect-verifier,token-validator,audit-production}.md`
  printing nothing.
- Scope held (`git diff --name-only main...HEAD`) — full command:
  `git diff --name-only main...HEAD -- src tokens .storybook | rg -v '^src/legacy/file-upload-helper\.ts$'`
  printing nothing (D1 hook files live under `.husky/` and `scripts/git/`, outside this filter).
- Nothing regressed — graded by `yarn lint && yarn test` exiting 0.

Numeric (each baseline below was run on 2026-09-14; the target is the same command's output
after the plan):

```derived
$ rg -c --hidden -g '!node_modules' -g '!dist' -g '!storybook-static' -g '!CHANGELOG.md' -g '!.claude/plans' '@egovmd/' . | awk -F: '{s+=$2} END{print "egovmd:", s}'
egovmd: 78
```

- `@egovmd/` outside `CHANGELOG.md` and `.claude/plans/**`: 78 → 0, graded by the command above
  (106 including historical plans, the figure the report quotes).

```derived
$ rg -n --hidden -g '!node_modules' -g '!.claude/plans' '\.husky/pre-commit' . | wc -l
9
```

- Files naming the deleted hook (`\.husky/pre-commit` count above): 9 → 0 under D1 = B; under D1 = A
  the count may stay 9 and the Phase 2.1 probe grades that each describes the hook as written.

```derived
$ rg -n '/Users/' .claude/settings.json | wc -l
8
```

- Foreign absolute paths in shared settings (`rg -n '/Users/'` count above): 8 → 0.

```derived
$ for f in AGENTS.md src/components/AGENTS.md tokens/AGENTS.md; do d=$(dirname "$f"); rg -o '`(\.\./)*_agents/[^`*]+\.md`' "$f" | tr -d '`' | sort -u | while read -r p; do [ -f "$d/$p" ] || echo "MISSING in $f: $p"; done; done | wc -l
6
```

- Unresolvable `_agents/` index paths: 6 → 0, graded by the command above.

```derived
$ rg -n 'Node >=22|>= ?22|Style Dictionary 4|Jest 30' AGENTS.md _agents .claude/agents .specs STACK.md | wc -l
4
```

- Stale version claims (`rg -n 'Node >=22|…'` count above): 4 → 0.
- `AGENTS.md` ≤ 152 lines and `tokens/AGENTS.md` ≤ 136 lines, graded by
  `wc -l AGENTS.md tokens/AGENTS.md` (derivation of both targets in Phase 5.10).

---

## Self-refute log

Four questions asked of this plan before any review, with what was scanned.

1. **Does the fix reuse the defect's own mechanism?** The defect is prose that nobody checks
   drifting away from the code. Phases 2–5 are more prose edited by hand, so on their own they
   repeat it. Instance found and fixed: the checker was originally the last phase, where it
   could not measure the edits before it. It is now Phase 0.4, so every later phase is graded
   by an instrument that is not the person making the edit, and Phase 6 only wires it in.
2. **Can the letter be met with the intent violated?**
   - *Delete the link instead of fixing it* — the link check goes to 0 while agents lose the
     index row to a file that exists. Guard: 2.4 rewrites paths and never removes rows; Phase
     4's deletion rule only removes references to files the plan itself deletes.
   - *Widen the exclusions* — the `@egovmd/` count reaches 0 by skipping more directories.
     Guard: the exclusions are fixed in the bar (`CHANGELOG.md`, `.claude/plans/**`) and in
     checker rule 3.
   - *Hook present but inert* — `.husky/pre-commit` exists but `core.hooksPath` is unset, or the
     probe uses `git commit --dry-run`, which skips hooks. Guard: the 2.1 probe runs the hook
     directly and asserts `core.hooksPath`.
3. **Does every number have a denominator and an instrument outside what it grades?** Every
   numeric bar row has a `derived` fence with the command and its 2026-09-14 output. Instance
   found and fixed while writing them: the first link check fell back to the repo root and
   passed exactly the five broken `src/components/AGENTS.md` paths. Its real baseline is 6,
   not 0, and the fallback is gone. The line-count targets are derived from section line
   ranges (5.10), not chosen.
4. **Do two of the plan's own rules interact into an unintended pass?**
   - *One home per fact vs. keeping the Figma STOP in `AGENTS.md` (3.9)* — two homes by design.
     This is resolved by keeping only the imperative in `AGENTS.md` and the full rule in
     `_agents/workflow-rules.md`, with a link, so the values cannot diverge.
   - *Archive closed plans (4.8) vs. the checker excluding `.claude/plans/**` (rule 3)* —
     archived plans keep `@egovmd/` legitimately. The exclusion covers them and does not reach
     live docs.
   - *D1 = A restores a hook that runs `yarn test` on every commit vs. "one concern per
     commit"* — a slow hook pushes contributors toward `--no-verify`, which bypasses the
     unstaging too. D1 = A therefore asks the team to decide the test run separately from the
     unstaging.

---

## Execution matrix

Dispatch verdict: **mixed.** Phases 0, 3, 4 and 5 can each go to a separate worker as a written
brief, with a diff and verify output coming back. Phases 1, 2, 6 and 7 stay in the main session:
they either carry a team decision (D1, tooling approval), need a fresh Claude Code session to
verify, or are a handful of lines where handing off costs more than doing.

| Phase | Model | Effort | Wave | Mode | Depends on / notes |
| --- | --- | --- | --- | --- | --- |
| 0 Baseline + checker | Sonnet 5 | medium | A | subagent | none; the checker and its spec are a self-contained brief with its own RED→GREEN test |
| 1 Load the docs | Opus 5 | low | B | inline | 0 (the checker gets `CLAUDE.md` into its scope); verify needs a new session |
| 2 Unsafe / false claims | Opus 5 | high | C (gate: team — D1 decided) | inline | 1; D1 touches `prepare`, hooks and merge scripts |
| 3 One home per fact | Sonnet 5 | medium | D | subagent | 2 — shares `AGENTS.md` and `.claude/agents/audit-production.md` with Phase 2, so not parallel; the 3.4 package-name sweep may run as its own leg on Haiku 4.5 · low, since it is a literal substitution with a count check |
| 4 Remove what is dead | Sonnet 5 | medium | E (gate: team — D2, D3, D4 decided) | subagent | 3; every deletion and the fix to its inbound references land in the same commit |
| 5 Fill the gaps | Sonnet 5 | high | F | subagent | 4 — writes `_agents/planning.md` and moves sections whose targets Phase 4 may have archived |
| 6 Keep it true | Opus 5 | medium | G (gate: team — tooling change approved) | inline | 5 — the checker must already pass before it becomes a gate |
| 7 Final verification + PR | Opus 5 | high | H | inline | 6; 7.3 runs in a fresh Claude Code session |

Every wave holds one phase, so no two phases write in parallel. Each phase is one commit (or one
commit per numbered task where the phase says so), made by the main session after the phase's
verify step passes.

Routing rationale: no phase needs more than Opus 5 at high — the decisions are already written
into this plan, and the two phases priced high are the ones that change hooks (2) and the one that
signs the result off (7). Writing new guidance (5) is priced high on Sonnet because its output is
read by every later session. Escalation: if a phase fails its verify step twice, restart that
phase one tier up with a fresh context instead of iterating in place. Run `yarn lint && yarn test`
once per phase, not once per task.

---

## Phase 0 — Baseline

**Executor**: Sonnet 5 · medium · Wave A · subagent

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
  **Expected now** (run 2026-09-14 on `fe6d651` + this plan's commit):
  ```text
  egovmd: 106
  9
  2
  8
  ```
  Nine files state the deleted pre-commit hook: `AGENTS.md:157`, `_agents/anti-patterns.md:24`,
  `scripts/audit/CLEANUP.md:84`, `.claude/skills/parallel-aux-tasks/SKILL.md:179`,
  `.claude/commands/migrate-component.md:135`, `.claude/agents/custom-component.md:187`,
  `.claude/agents/new-component.md:242`, `.claude/agents/redesign-component.md:304`,
  `.claude/agents/refactor-component.md:210` — plus `CONTRIBUTING.md:221,237`, which names
  Husky without the path.
- [ ] **0.3** Record the index link check (the same loop Phase 2.8 re-runs). It resolves each
  path **only** relative to the file that contains it — no second lookup from the repo root,
  which would silently pass exactly the broken paths this check exists to find — and skips
  glob mentions such as `_agents/*.md`.
  ```bash
  for f in AGENTS.md src/components/AGENTS.md tokens/AGENTS.md; do
    d=$(dirname "$f")
    rg -o '`(\.\./)*_agents/[^`*]+\.md`' "$f" | tr -d '`' | sort -u | while read -r p; do
      [ -f "$d/$p" ] || echo "MISSING in $f: $p"
    done
  done
  ```
  **Expected now:**
  ```text
  MISSING in src/components/AGENTS.md: _agents/anti-patterns.md
  MISSING in src/components/AGENTS.md: _agents/pixel-perfect-qa.md
  MISSING in src/components/AGENTS.md: _agents/pre-implementation.md
  MISSING in src/components/AGENTS.md: _agents/shadow-dom-patterns.md
  MISSING in src/components/AGENTS.md: _agents/typescript-strict.md
  MISSING in tokens/AGENTS.md: _agents/pre-implementation.md
  ```
- [ ] **0.4** Build the instrument before the fixes it measures:
  `scripts/docs/check-ai-docs.mjs` (Node, no dependencies) with a spec
  `scripts/__tests__/check-ai-docs.spec.mjs` (picked up by the existing `test:scripts` glob).
  Run it with `node scripts/docs/check-ai-docs.mjs`; no `package.json` or CI change yet
  (that is Phase 6, gated). It reports `file:line` per hit and exits 1 on any:
  1. a relative Markdown link, or a backticked `_agents/…md` path, in `CLAUDE.md`,
     `AGENTS.md` ×3, `_agents/**`, `src/components/_agents/**`, `tokens/_agents/**`,
     `.claude/{agents,commands,skills}/**` that does not resolve **relative to the file
     containing it** (glob mentions like `_agents/*.md` are skipped);
  2. a Node version claim (`Node >=NN`, `>= NN`, `node -v (>= NN)`) whose major is outside
     `package.json` `engines.node`;
  3. `@egovmd/` anywhere outside `CHANGELOG.md` and `.claude/plans/**`;
  4. an absolute `/Users/` or `C:\Users\` path in `.claude/settings.json`.

  Out of scope for the checker: `yarn <script>` mentions (too many legitimate forms in prose to
  parse reliably — the audit's dead-command list is triaged by hand in Phase 5).
  **Spec:** one fixture per rule that must fail and one clean fixture that must pass; the spec
  must fail before the checker exists (`Cannot find module`).
  **Verify:** `yarn test:scripts` passes; `node scripts/docs/check-ai-docs.mjs; echo $?` prints
  the Phase 0.2/0.3 defects and `1`. Paste the hit count into the PR description as the
  baseline.

---

## Phase 1 — Make the documentation load (F1)

**Executor**: Opus 5 · low · Wave B · inline

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

**Executor**: Opus 5 · high · Wave C (gate: team — D1 decided) · inline

**Why:** these are the claims that cause wrong actions (`git add -A`, an auditor editing,
loading a file that does not exist), not just wrong reading.

- [ ] **2.1 (D1)** Apply the chosen option.
  - **If A:** write `.husky/pre-commit` (Husky 9 form: no `_/husky.sh` line) from the
    unstaging block of `git show 0d23e93^:.husky/pre-commit`, with `GENERATED_PATTERNS`
    re-checked against today's generated paths and the `components.d.ts` policy from D1;
    restore `.husky/commit-msg` (`npx --no -- commitlint --edit "$1"`); add a tracked
    `.husky/post-merge` carrying the hint `setup-merge-drivers.sh` writes today, and drop
    the `.git/hooks` write from `setup-merge-drivers.{sh,ps1}`; `prepare` →
    `husky && node scripts/git/setup-merge-drivers.mjs`. Then reconcile the nine files from
    Phase 0.2 with the hook actually written (they describe it almost correctly already).
    Propose the CI step upstream (Phase 6.3).
  - **If B:** rewrite the nine files from Phase 0.2 and `CONTRIBUTING.md:221,237` to state the
    real guard (explicit staging + `yarn build` after merges touching generated files, see
    5.5); remove `husky install` from `prepare`; add the one-time
    `git config --unset core.hooksPath` note to the PR description.
  - **Verify (A)** — `git commit --dry-run` does not run hooks, so exercise the hook itself:
    ```bash
    printf '\n' >> .storybook/custom-elements.json && git add .storybook/custom-elements.json
    sh .husky/pre-commit; git diff --cached --name-only   # expect: custom-elements.json absent
    git checkout -- .storybook/custom-elements.json
    printf 'bad message\n' | npx --no -- commitlint       # expect: non-zero exit
    git config --get core.hooksPath                       # expect: .husky/_
    ```
  - **Verify (B):** the Phase 0.2 husky count prints `0`, and
    `node -p 'require("./package.json").scripts.prepare'` contains no `husky`.
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
  and label the table "Root `_agents/` (paths relative to this file)". Same for
  `tokens/AGENTS.md`'s `_agents/pre-implementation.md` → `../_agents/pre-implementation.md`.
- [ ] **2.5 (F8)** `tokens/AGENTS.md` — remove the reference to the missing
  `analizeaza-structura-la-fisierul-breezy-tower.md`; keep the instruction it supported
  ("author `tokens/core/effects.tokens.json` before `--apply`") as a plain rule.
- [ ] **2.6 (F8)** `.claude/skills/audit-component/SKILL.md:698,875` → `../../../_agents/anti-patterns.md`;
  `.claude/skills/mud-design/SKILL.md:149` → fix or remove `reference/`.
- [ ] **2.7 (F12)** `.claude/settings.json` — delete the 8 allow entries containing
  `/Users/vitalie/`. **Verify:** `rg -n '/Users/' .claude/settings.json` → 0;
  `node -e 'require("./.claude/settings.json")'` parses.
- [ ] **2.8 Verify phase:** re-run the Phase 0.3 loop, and `yarn docs:check` from Phase 0.4
  if it already exists. **Expected:** the loop prints no `MISSING` lines.

---

## Phase 3 — One home per fact (F3, F4, F10)

**Executor**: Sonnet 5 · medium · Wave D · subagent (3.4 sweep: Haiku 4.5 · low)

**Why:** F2 and F4 are both "one fact changed, the copies did not". Fixing values without
removing copies re-creates the defect at the next upgrade.

- [ ] **3.1** `STACK.md` becomes the only place versions are written. Add at its top:
  "Versions below are copied from `package.json`; when they disagree, `package.json` wins
  and this table is fixed in the same PR." `STACK.md` is not fully consistent itself: its table
  says Style Dictionary `^5.4.1` but the rationale heading at `STACK.md:32` reads
  "Style Dictionary 4.x for tokens" — fix the heading to carry no version.
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
  **Verify:** the `@egovmd/` command in the Acceptance bar (which excludes `CHANGELOG.md` and
  `.claude/plans/`) prints `egovmd: 0`.
- [ ] **3.5 (F3)** `SECURITY.md` supported-versions table → `@egov-moldova/mud`,
  `@egov-moldova/mud-web-components`, `@egov-moldova/mud-react` (state its real publish
  status: check `npm view @egov-moldova/mud-react version`).
- [ ] **3.6 (F4)** `.specs/PROJECT-SPECIFICATION.md:23` — Jest 30.x → Vitest + `@stencil/vitest`
  (version: see `STACK.md`); `:25` Node `>= 22` → see `STACK.md`. `.claude/kanban/README.md:48` — Jest → Vitest (or archived via D3).
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
  **Expected now (before Phase 3):** 4 hits — `AGENTS.md:5`, `STACK.md:32`,
  `.specs/PROJECT-SPECIFICATION.md:25`, `_agents/cross-platform-guide.md:11`.
  **Expected after Phase 3:** exactly 1 — `_agents/cross-platform-guide.md:11`, which Phase 4.3
  deletes. (`.specs/PROJECT-SPECIFICATION.md:23` says "Jest (with Stencil test runner) | 30.x"
  and is not matched by this pattern; 3.6 fixes it by reading, and the checker does not cover
  Jest claims.)

---

## Phase 4 — Remove what is dead (F7, F9, F11, D2–D4)

**Executor**: Sonnet 5 · medium · Wave E (gate: team — D2, D3, D4 decided) · subagent

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
    repeated wrong assumption; fix `component-structure.md:180` and
    `composition-interactive.md:76` ("`cor` prefix" → "`mud` prefix", e.g.
    `@Event() mudButtonClick`); update the two index descriptions in
    `src/components/AGENTS.md:19,25` and `component-structure.md:5` that still say
    "stenciljs skill corrections".
  - `tokens/AGENTS.md:174` — delete (duplicate of `tokens/_agents/token-structure.md:151`);
    rework the latter the same way.
  - `_agents/mcp-tools.md:349` ("Skill File Tool Name Corrections") — **keep for now**: live
    files still use the old tool spellings it corrects (`rg -c 'figma_get_|playwright_|ctx7_'`
    → `.claude/commands/audit-accessibility.md` 18, `pre-pr-check.md` 3, `modify-component.md` 3,
    `README.md` 1, `.claude/agents/custom-component.md` 2). Replace those spellings with the
    `mcp__…` names first, then delete the table in the same commit.
  **Expected now:** `rg -n 'Skill Says|stencil-atomic|stenciljs skill|`cor` prefix' src tokens _agents`
  → 10 hits. **After:** 0 (the `mcp-tools.md` row goes with its sub-task).
- [ ] **4.7 (D3)** If Kanban is retired: `git mv .claude/kanban .claude/plans/_archive/kanban`;
  update `.gitattributes`, `.claude/settings.json`, `.claude/agents/redesign-component.md`,
  `AGENTS.md` references found by `rg -n 'kanban' -g '!.claude/plans'`.
- [ ] **4.8** Archive closed plans: `git mv` every `.claude/plans/*.prompt.md` and
  `claude-code-automation-improvements-implementation-plan,prompt.md` into
  `.claude/plans/_archive/`, after confirming each is merged or abandoned (`git log --oneline`
  for the feature it describes). Dated plans (`2026-*`) stay. Only one inbound reference exists
  outside `.claude/plans/`: `scripts/audit/CLEANUP.md:73` links
  `claude-code-automation-improvements-implementation-plan,prompt.md` — update that link in the
  same commit.
- [ ] **4.9 Verify phase:** re-run the Phase 0.3 link loop and
  `rg -n '_agents/cross-platform-guide|\.impeccable|skill-creator' -g '!.claude/plans' .` → 0.

---

## Phase 5 — Fill the gaps (F6, F11, F14)

**Executor**: Sonnet 5 · high · Wave F · subagent

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
- [ ] **5.3 (F14)** Move `AGENTS.md` "Merge driver for auto-generated files" (lines 147–185)
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
- [ ] **5.10 Verify phase:** `wc -l AGENTS.md tokens/AGENTS.md`. Targets are derived from the
  sections moved, not picked: `AGENTS.md` is 185 lines and "Merge driver" spans lines 147–185
  (39 lines) → ≤ 152 with a 3-line summary and the Phase 5.2 index rows (+5) counted in;
  `tokens/AGENTS.md` is 178 lines and "Tokenhaus Sync Workflow" spans 115–161 (47 lines) →
  ≤ 136 with a 5-line summary. Phase 0.3 loop and `node scripts/docs/check-ai-docs.mjs` clean
  for the files touched so far.
- [ ] **5.11 (F8)** Dead `yarn` commands, one pass by hand: list every `yarn <name>` in
  `AGENTS.md` ×3, `_agents/**`, `src/components/_agents/**`, `tokens/_agents/**`,
  `.claude/{agents,commands,skills}/**`, and compare against `package.json` `scripts`:
  ```bash
  node -e 'const s=Object.keys(require("./package.json").scripts);process.stdout.write(s.join("\n"))' | sort > /tmp/scripts.txt
  rg -o --no-filename 'yarn ([a-z][a-z0-9:._-]*)' -r '$1' AGENTS.md src/components/AGENTS.md tokens/AGENTS.md _agents src/components/_agents tokens/_agents .claude/agents .claude/commands .claude/skills | sort -u | comm -23 - /tmp/scripts.txt
  ```
  The output also contains yarn built-ins (`install`, `add`, `dlx`, `workspace` …) — drop those by
  reading, fix or remove the rest. No ratchet follows (see Out of scope).

---

## Phase 6 — Keep it true (F13) — tooling, gated

**Executor**: Opus 5 · medium · Wave G (gate: team — tooling change approved) · inline

**Why:** every defect in the report was a doc that stayed the same while the code moved. Only
a check that runs without anyone remembering it prevents the next round.

- [ ] **6.1** The checker already exists (Phase 0.4) and passes (Phases 2–5). This phase only
  makes it run without anyone remembering to.
- [ ] **6.2** Wire it: `package.json` `"docs:check": "wireit"` with a wireit block
  (`command: node scripts/docs/check-ai-docs.mjs`, `files` = the doc globs it reads) and the
  same wireit form for `test:scripts`, then add both to `check.verify.dependencies`
  (`package.json:398`, today `["typecheck","lint","test"]`). Every current dependency there is
  a wireit script; keep the new ones in that form rather than relying on plain scripts being
  accepted as dependencies.
  **Verify:** `yarn check` output shows both running; add a broken `_agents/` path to
  `AGENTS.md` locally → `yarn check` exits non-zero naming `AGENTS.md:<line>`; revert.
- [ ] **6.3** Upstream CI (`egov-moldova/design-system`, `.github/workflows/ci.yml`): propose a
  PR adding `yarn docs:check` and `yarn test:scripts` to the Test job, and — if D1 = A — the
  "Verify no stale generated files" step after Build. This is a change to the shared pipeline:
  open it as its own PR, not inside the docs PR.
- [ ] **6.4 Verify phase:** `yarn docs:check` exits 0 on the branch; the Phase 0.2 counts are
  0 (or only the documented exceptions).

---

## Phase 7 — Final verification and PR

**Executor**: Opus 5 · high · Wave H · inline

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

- A ratchet for `yarn <script>` mentions and test-runner prose. Both appear in too many
  legitimate prose forms to parse reliably; 5.11 and 3.6 fix today's instances once, and a
  later audit is the only thing that catches new ones.

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
