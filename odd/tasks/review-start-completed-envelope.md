# review-start-completed-envelope

A durable review START whose reviewers run inside the provider (not host relay) can finish the
whole review before the envelope is rendered; the façade still attaches the rctx2 repository
context, the v4 status continuation and the artifact subjects, and `Validate` refuses all three
because the authority is no longer `reviewing`.
Branch: `fix/review-start-completed-authority`, based on `origin/main` = `5ec0bb79` (worktree `/tmp/ga-fix`).
Candidate: `internal/cli/review_start_contract.go`.

## Specs

- **S1 — Apply the solution.** The observed defect envelope must stop happening: a START that
  creates a durable authority and lets the provider run every selected lens to a verdict must
  return a valid envelope instead of
  `candidate_context_unavailable` / "negotiated START repository context does not match the active
  reviewing authority". The corrected envelope must publish the authority's real terminal state
  (`correction_required` / `approved`) and no reviewing-only fields.
- **S2 — Verify it works.** Proof must exist at two levels: a failing-then-passing test that
  reproduces the completed-inside-START shape, and the real production envelope that motivated it.
  The binary-level side-by-side is NOT reachable and must not be forced: a build of `origin/main`
  stops at `managed_assets_outdated` (its embedded managed-asset version is newer than the installed
  `v4.5.0` assets), and a `v4.5.0` build stops at the legitimate `consent_required` gate before any
  authority is assembled. Granting consent by hand to obtain the comparison would fabricate review
  authority, so it is out of scope.
- **S3 — Then report and contribute, with evidence.** Only after S1 and S2 hold: a definitive
  open+closed issue lookup on `Gentleman-Programming/gentle-ai`, then the report and the
  contribution (patch on the fork branch) quoting the sanitized evidence collected in S2.

## Tasks

- **T1** (S1, inline, -) RED: reproduce the envelope refusal for a provider-run START that
  completes the review, asserting the exact failure string.
- **T2** (S1, inline, -) GREEN: guard the reviewing-only fields (`repository_context`,
  `next_transition`, `artifact_subjects`) on the authority state, leaving the validator untouched.
- **T3** (S2, inline, -) Focused package tests plus the surrounding contract/replay suites.
- **T4** (S2, inline, -) DROPPED after two honest attempts: `managed_assets_outdated` on a
  `origin/main` build, `consent_required` on a `v4.5.0` build. Both runs left the authority byte
  identical (state hash unchanged). The production failure envelope stands as the real-world
  "before"; the package test is the reproduction.
- **T5** (S3, inline, -) Definitive issue lookup, then report with the S2 evidence.
- **T6** (S3, inline, -) Work-unit commit; the fork branch is the contribution.

## Log

- **L1** (verbatim, user): «vamos a retomar, que queda pendiente?»
- **L2** (verbatim, user): «apliquemos la solucion, verifiquemos que funcione y luego procedamos
  hacer el reporte con la evidencia y el aporte con la evidencia»
- **L3** (evidence, native state): lineage `review-da231e123802e424` persisted at revision
  `sha256:a3065c11…` with `state = correction_required`, `capture_phase_revision = sha256:fac6dbf5…`,
  and three `admitted_role_results` (lens risk, lens reliability, refuter `corroborated` on
  `R3-001`) — the review completed inside the START that the consumer saw fail.
- **L4** (evidence, code): `review_start_contract.go:242-251` requires the repository context iff
  `State == StateReviewing && action in {created,resumed,replayed}`, and the same predicate gates
  `next_transition` (`:252-255`) and the artifact subjects (`:288-292`). `review_facade.go:2305-2330`
  attaches all three unconditionally for a created/resumed START.
- **L5** (note): the upstream repository's `rdd-defect-workflow` skill asks for an approved issue
  before implementation. The user's order is S1 → S2 → S3; this patch is developed on the user's
  fork branch as the evidence-backed contribution, and the issue is filed in T5.
- **L6** (decision, user-requested freshness check): the first patch was written on
  `fix/rctx2-host-resolution-on-main` at `1ef2fc64` dated 2026-09-29, which is **1266 commits behind**
  `origin/main` (`5ec0bb79`, 2026-11-20). The defect is NOT fixed upstream: `origin/main` still carries
  the exact `needsRepositoryContext` predicate, `review_facade.go` still attaches the context and the
  continuation unconditionally (there at `:2739-2754`), and the released tag `v4.6.0` has no state
  guard either. The user's own commit `1ef2fc64` is not an ancestor of `origin/main`.
  Therefore the patch is rebased onto `origin/main` in a fresh worktree `/tmp/ga-fix` on branch
  `fix/review-start-completed-authority`; the 149-line diff applied cleanly and the helper the new
  test needs (`completeNegotiatedStartReview`) already exists there at
  `internal/cli/review_start_context_test.go:393`.
- **L7** (evidence, rebased tree): RED reproduced on `origin/main` itself —
  `TestNegotiatedReviewStartCompletedAuthorityPublishesOnlyItsTerminalState/completed_authority_publishes_only_its_terminal_state`
  fails with `validate negotiated START response: negotiated START repository context does not match
  the active reviewing authority`; the same test passes with the 12-line guard restored.
- **L8** (evidence, released code): the installed `v4.5.0` release that produced the production
  failure carries the same shape — the predicate at `internal/cli/review_start_contract.go:243-250`
  and the unconditional attachment at `internal/cli/review_facade.go:2739-2754`. So the defect is not
  a version skew between the fork and the release; the same lines are live in `v4.5.0`, `v4.6.0` and
  today's `origin/main`.
- **L9** (verification note): binaries built from `origin/main` refuse at preflight with
  `managed_assets_outdated` (a source build embeds a newer managed-asset version than the installed
  `v4.5.0` assets), and nothing mutates the authority (state hash unchanged). The binary-level A/B is
  therefore built from the `v4.5.0` tag, where the asset version matches the installed one, while the
  contribution itself stays on `origin/main`.
- **L10** (result of L9's A/B attempt): both `v4.5.0` binaries return `consent_required` for the same
  invocation, because a fresh START on that lineage asks for consent before assembling any authority
  envelope — so the A/B cannot reach the defect without granting consent by hand, which would
  fabricate review authority. The comparison is dropped, not faked. Evidence for the report is: the
  production envelope (`candidate_context_unavailable`, cause naming the `a3065c11` authority), the
  RED on `origin/main`, the GREEN with the guard, and the full `internal/cli` suite (195.092s, green)
  with `gofmt` and `go vet` clean.
