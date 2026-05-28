# CI governance for conveo repos

Mandatory CI gates for repos under the `conveo` org. This file is the source of
truth; the reusable workflows in `.github/workflows/` are the implementation.

## Why this exists

The May 2026 audit of GTM apps (`~/conveo/gtm/AUDIT-SUMMARY.md`) found live API
keys committed in three of five repos: Anthropic, Gemini, ElevenLabs, Logo.dev,
Firecrawl, Notion, Voyage, a Google OAuth client secret, and a Supabase
service-role JWT (exp 2036). None of those repos ran any secret scanner. The
single highest-leverage fix was "add gitleaks to every repo." This repo is that
fix, centralized so we configure the rule set once.

## Required gates

| Gate     | Status   | Workflow                                                |
|----------|----------|---------------------------------------------------------|
| gitleaks | required | `conveo/.github/.github/workflows/gitleaks.yml@<ref>`   |

The gitleaks workflow is dual-triggered: it accepts `workflow_call` (explicit
opt-in via `uses:`) and `pull_request` / `merge_group` (auto-enforced via
org-level branch ruleset, no per-repo file needed).

## Scope

Required for every repo that holds production code, infrastructure, or anything
that runs in a Conveo environment. At minimum:

- `agentic-marketing-engine`
- `calls-mcp`
- `cockpit`
- `gmail-proxy`
- `gtm-engineered-seredipity-engine`
- `openclaws`

New repos under `conveo` adopt the gate on creation. Exceptions (e.g. a public
docs-only repo) need an explicit note in this file.

## Onboarding a repo

### Option A — Org-wide enforcement via ruleset (preferred, zero-touch)

Targeted repos get the gitleaks gate automatically. No files added to them.

One-time org setup:

1. **Org Settings → Custom properties** → add property `governance` with allowed
   values `mandatory`, `exempt`.
2. **Org Settings → Repository → Custom properties** → set `governance =
   mandatory` on the six audit-listed repos. New repos get tagged at creation.
3. **Org Settings → Repository → Repository rules → New ruleset → New branch
   ruleset**. Configure:
   - Name: `gitleaks-required`
   - Enforcement: `Evaluate` for one week, then flip to `Active`.
   - Target repositories: dynamic list where `governance = mandatory`.
   - Target branches: "Default branch".
   - Rules: enable **Require workflows to pass before merging** → add
     workflow:
     - Source: `conveo/.github`
     - Path: `.github/workflows/gitleaks.yml`
     - Ref: `main` (pin to a tag once we cut releases)

Caveat — Actions cross-repo access: the workflow checks out `conveo/.github` at
runtime to pull the baseline `.gitleaks.toml`. For this to work on private
target repos, either keep `conveo/.github` public (current state) or set
**Org Settings → Actions → General → Allow specified actions and reusable
workflows → Allow actions and reusable workflows from this organization**.

### Option B — Per-repo opt-in (for repos outside the ruleset, e.g. exempted ones)

1. Copy `examples/caller-gitleaks.yml` into the repo at
   `.github/workflows/gitleaks.yml`.
2. Open a PR. The first run scans full history — expect findings in
   pre-existing commits.
3. For each finding: **rotate the credential first**, then either purge from
   history or accept the finding and add a narrow allowlist entry in a
   repo-local `.gitleaks.toml`.
4. Mark the gitleaks check as required in branch protection for `main`.

Do not add `continue-on-error: true` to bypass a finding. The audit specifically
called this pattern out in `gtm-engineered-seredipity-engine`'s `audit-ci` step.

## Complementary: GitHub native secret scanning

Enable **secret scanning + push protection** at the org level (Org Settings →
Code security → Global settings). This blocks `git push` at the GitHub side
when a known-provider key pattern is detected — catches new leaks before they
hit history. It does not replace gitleaks (which does full-history scans and
custom rules), but it shortens the gap between "developer pastes a key" and
"key is in the repo" from minutes to zero.

## Handling a finding

Treat a positive hit as a credential leak until proven otherwise. The steps,
in order:

1. **Rotate** the credential at the provider. Assume it is compromised — the
   commit is in the reflog, in CI logs, in any clone, and in any fork.
2. **Confirm rotation propagated** to every consuming environment.
3. Only then, decide whether to:
   - leave the secret in history (acceptable since it's rotated) and proceed, or
   - rewrite history with `git filter-repo` (rarely worth the disruption).
4. If the finding was a false positive, add a narrow allowlist to a repo-local
   `.gitleaks.toml` — narrow by path or by regex, never by rule-id globally.

## Customizing per repo

Three ways to override the baseline, in precedence order:

1. Pass `config-path: <path>` to the reusable workflow.
2. Commit a `.gitleaks.toml` at the repo root.
3. Fall through to the baseline at `.gitleaks.toml` in this repo.

Repo-local configs should `extend.useDefault = true` and add narrow rules — do
not redefine the full rule set per repo.

## Bumping gitleaks

The CLI version is pinned in `.github/workflows/gitleaks.yml` via the
`gitleaks-version` input default. To bump:

1. Update the default in `.github/workflows/gitleaks.yml`.
2. Open a PR. CI runs against this repo as a smoke test.
3. After merge, callers pick up the new version on next run (no caller changes
   needed unless a caller pins `gitleaks-version` explicitly).
