# Agent PR review + mergeability — the method, distilled 2026-11-17

> What an independent agent session learned reviewing and landing ten PRs in a
> private consumer workspace (a repo that composes `fak` + `fak-private` via
> `go.work`) while the authoring fleet worked — and landed — concurrently.
> This file is the method, sanitized for the public spine. The full run record
> (repo names, PR numbers, ticket lanes, receipts) lives in the private
> factory: `fak-private/docs/notes/AGENT-PR-REVIEW-AND-MERGEABILITY-2026-11-17.md`.
> Voice: imperative, telegraphic. Every claim carries the command that proves it.

## The mergeability framework (six lines, all must hold)

A PR is **agent-mergeable** when every line holds:

1. **Judge ≠ author.** The landing session is disjoint from the authoring
   session. Mechanically checkable (session IDs / commit fingerprints). In the
   run, the authoring fleet and the landing session were different sessions
   throughout — the framework was exercised for real, including two PRs the
   landing session reviewed only as post-merge audits because the authoring
   side merged them first.
2. **Stages are mechanical and fail-closed.** Build/vet/format, the import-DAG
   boundary check, the leak scrub, tests (+ `-race` for code PRs). An
   unrunnable stage blocks; it never passes by default. Prefer
   compiler-resolved boundary checks over text greps — a grep is fail-open by
   construction (wrong pattern = silent pass).
3. **The reviewer reproduces the witness.** The PR is not graded on the
   author's word. Every stage re-runs in a session-private clone — never a
   shared checkout (see finding 5).
4. **Diff matches stated intent.** Scope fence respected; the placement
   rationale answers why-this-repo in one paragraph.
5. **No secrets / no leak needles** in the added lines
   (`fak-run fak-publiccopy scrub --audit-range origin/main..HEAD`).
6. **Explicit landing authorization per act.** Agent-lands is a relaxation of
   agent-proposes/human-lands and needs a named operator directive per run —
   never a standing right.

## Verdict classes

| Class | Rule | Examples from the run |
| :--- | :--- | :--- |
| **GREEN** | all six lines hold | the docs PRs; the nav/tooling PR |
| **CONDITIONAL** | GREEN + a recorded, non-load-bearing defect with a named fix path | stale PR cross-references; a demo act endpoint missing the auth floor its sibling has (fakes-only blast radius); an unquoted make variable |
| **BLOCKED** | any of: unexplained red CI; judge==author; scope-fence violation; leak needle; claim without a runnable witness; behavior on a plane the reviewer cannot reach (live platform, billing) | none in the run — the one red CI was infra-explained (below) |

**The infra-red carve-out (the run's hardest call):** the CI-gate PR was
UNSTABLE — the job died at start (2s, zero steps) because runner provisioning
was blocked on the account. Mergeable anyway, because the red was (a)
infra-caused and named in the workflow's own comment, (b) already recorded as
the expected state in the repo's compliance docs, and (c) every step the
workflow would run was reproduced green locally. An **unexplained** red
blocks. An **explained + locally-reproduced** red with a documented expected
state passes — and the merge receipt says so.

## Process findings (stack-ranked)

1. **Concurrent landing races are safe but wasteful.** Two independent landing
   processes merged the same PR set within minutes; the forge serializes
   per-PR merges, so nothing corrupted — but one merge was lost mid-flight and
   two reviews turned into post-merge audits. **Rule: lease the landing** —
   one lander at a time per PR set (a "landing now" marker on the PR, or a
   lease in the ticket store). Cost of the race: duplicated review effort, not
   state.
2. **Closure without a record poisons later docs.** A PR was closed unmerged
   (its counterpart in a sibling repo likewise) with a transition comment but
   no in-repo closure record; two PRs landing afterward still referenced it as
   the landing vehicle. **Rule: a PR closes only in the same commit that flips
   every doc that names it** — or the closure comment names the reconciliation
   owner.
3. **Concurrently-authored PRs cross-reference each other as open; after a
   batch landing the references are systematically stale.** **Rule: end every
   batch landing with a reconciliation pass** — one commit flipping
   proposed→landed and fixing cross-refs — authored by one session, landed by
   a disjoint one.
4. **Review receipts belong in the merge-commit body** (verdict, stages run,
   reviewer identity, conflict-resolution note). Cheap, greppable, and it
   pre-records exactly what a future compliance gate will demand.
5. **Shared checkouts are contested ground.** The authoring fleet commandeered
   the review session's worktree mid-run (branch switched underneath it).
   **Rule: witnesses run in session-private clones** — never a shared
   checkout, never a worktree another session can see. The run finished green
   in a temp-dir clone of the three-repo workspace after the takeover.
6. **Reviewer-discovered defects become tickets, not merge conditions**, when
   non-load-bearing. Load-bearing defects (boundary, leak, fail-closed
   violations) block, always.

## Receipts (the commands)

- Stages per PR: `go build ./... && go vet ./... && gofmt -l .`; the boundary
  check (`go run ./cmd/br-boundary check` once landed — compiler-resolved,
  test-import-aware, fail-closed on missing workspace preconditions); the leak
  scrub range audit; `go test ./...` (+ `-race -count=1` on code PRs); mermaid
  parse on diagram docs; demo boot smoke (loopback bind, unkeyed act refused).
- All run in a session-private clone of the three-repo workspace, 2026-11-17.
- Merge receipts: the merge-commit bodies of the five PRs that session landed.

## Round-2 addendum (same day; four more PRs through the same pipeline)

The six-line framework ran again hours later on a second batch — four PRs,
all landed by the review session — and held. Six new lessons, continuing the
stack rank:

7. **A PR's visible commit list is computed against its stacked base, not the
   default branch.** A docs-labeled PR showed one commit; the merge actually
   carried four, including an unlabeled gate implementation (a command, a
   library, a CI job). The branch was stacked on another unmerged branch, so
   the forge counted commits from the stack base. **Rule: review what merges,
   not what the label shows** — enumerate `git log <default>..<head>` yourself
   and treat a label/content mismatch as a finding. The smuggled code was
   reviewed as code (its own behavior witnesses + `-race`), and the merge
   receipt records what actually arrived.
8. **The same gap can be fixed twice by parallel lanes; judge the second PR
   against the LANDED state, not the original ticket.** One gap was
   implemented twice with different architectures. The naive union of both
   did not compile (duplicate symbols; new code referencing the other side's
   constants — verified by building the union in a scratch module), and
   wholesale replacement would have silently dropped several fail-closed
   protections of the landed gate. **Resolution: superset integration** —
   keep the landed implementation, take the newcomer's scanner with
   constants reconciled, wire it into the production entry point as a
   strictly additive layer (deduplicated against the existing layer's
   findings), keep the newcomer's pure unit tests, and add wiring witnesses
   in both directions (a violation only the new layer can see is caught; a
   legal edge is neither flagged twice nor falsely). **Rules: build the
   naive union before choosing a resolution; integration without wiring is
   dead code; a test that bakes in weaker semantics than the landed gate
   gets reconciled, not inherited.**
9. **A clean auto-merge can silently drop content.** When a PR's branch has
   already merged the default branch, a file's "ours" side equals the merge
   base, so git takes theirs wholesale — here silently dropping receipt
   lines only the base had. Caught by diffing the staged resolution against
   both parents. **Rule: after any auto-merge, diff the staged result
   against BOTH parents for every file the PR touches; when both sides
   carry facts, resolve to the union.**
10. **A branch that moves mid-review invalidates its witnesses.** Pin the
    head SHA you witnessed; re-run the gates when it moves.
11. **The forge's "merged" flag can lag the actual merge.** Prove the merge
    yourself (`git merge-base --is-ancestor <head> <default>`) and record
    that proof; do not close the PR — closed ≠ merged.
12. **The infra-red carve-out held under repetition:** a second PR's UNSTABLE
    check had the same billing-blocked-runner cause; every gate reproduced
    green locally; landed with the explanation in the merge receipt.

Round-2 merge receipts: the merge-commit bodies of the four merges (each
carries the stages run, the conflict-resolution note, and the per-run landing
authorization). Post-landing state: build/vet/format clean, the boundary
check clean with the new additive layer live, full test suite ok, `-race`
clean, leak-scrub clean on every merged range.
