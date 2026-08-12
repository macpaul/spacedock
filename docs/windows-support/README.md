---
commissioned-by: spacedock@0.27.0
entity-type: task
entity-label: task
entity-label-plural: tasks
id-style: sd-b32
state: .spacedock-state
stages:
  # Stage names must match ^[a-z0-9][a-z0-9-]*[a-z0-9]$ (kebab-case lowercase, no underscores or spaces); `status --validate` rejects others.
  defaults:
    worktree: false
    concurrency: 2
  states:
    - name: backlog
      initial: true
      gate: true
    - name: ideation
      gate: true
    - name: implementation
      worktree: true
      context-sections:
        - Review-finding disposition
    - name: validation
      worktree: true
      fresh: true
      feedback-to: implementation
      gate: true
      context-sections:
        - Review-finding disposition
    - name: done
      terminal: true
---

# Windows x86-32 and x86-64 Cross-Compilation Support

This workflow adds first-class Windows support to the spacedock binary — both 32-bit (GOARCH=386) and 64-bit (GOARCH=amd64) — using Go's native cross-compilation capability and the MinGW-w64 toolchain for CGO boundary verification. Because a Windows OS environment is not available on the development machine, each task is verified by cross-compiling from Linux using MinGW-w64 (`x86_64-w64-mingw32-gcc` and `i686-w64-mingw32-gcc`) and inspecting the resulting PE binaries. Final end-to-end smoke testing is deferred to a physical Windows machine after all cross-compilation tasks reach `done`.

Tasks flow from a backlog holding stage through ideation (scope the change, write acceptance criteria), implementation (write the code on a dedicated worktree branch, cross-compile locally to verify), and an independent validation pass (a fresh agent reproduces the cross-compilation and binary-inspection proof), then land via PR review.

## File Naming

Each task lives as either:

- a flat markdown file `{slug}.md` (default), or
- a folder `{slug}/` containing `index.md` when the task produces sibling artifacts (build logs, MinGW diagnostic output, comparison tables) that belong with the tracker.

Slugs are lowercase, hyphens, no spaces. Example: `win32-cross-compilation.md`.

## Schema

Every task file has YAML frontmatter. Fields are documented below; see **Task Template** for a copy-paste starter.

### Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique SD-B32 identifier (24-char, human-safe alphabet) |
| `title` | string | Human-readable task name |
| `status` | enum | One of: backlog, ideation, implementation, validation, done |
| `source` | string | Where this task came from (issue, captain note, retrospective) |
| `started` | ISO 8601 | When active work began |
| `completed` | ISO 8601 | When the task reached terminal status |
| `verdict` | enum | PASSED or REJECTED — set at validation |
| `score` | number | Priority score, 0.0–1.0 (optional) |
| `worktree` | string | Worktree path while a dispatched agent is active, empty otherwise |
| `issue` | string | GitHub issue reference (e.g., `#42`) — optional cross-reference |
| `pr` | string | GitHub PR reference (e.g., `#57`) — set when a PR is opened for this task's branch |
| `mod-block` | string | Pending mod-declared blocking action, format `{lifecycle_point}:{mod_name}` |

### ID Style

`id-style: sd-b32` — IDs are 24-character lowercase SD-B32 values derived from SHA-256 digest material using Spacedock's human-safe alphabet `0123456789abcdefghjkmnpqrstvwxyz`. The status viewer shows shortest unique prefixes with `MIN_PREFIX: 2`.

## Stages

### `backlog`

A task enters backlog when it is first proposed: a description of the Windows target to support, no design work yet. The captain decides which tasks advance to ideation based on dependency order and risk.

- **Inputs:** Task description naming the Windows target (Win32, Win64, or a build pipeline change), including what the expected deliverable looks like and any known constraints (e.g., CGO disabled, pure Go only).
- **Outputs:** A seed entity file in `docs/windows-support/` with a clear title, description, and a rough scope note.
- **Good:** The task is scoped to a single Windows target or a single pipeline concern. The title unambiguously names what Windows variant is being added.
- **Bad:** A task that conflates Win32 and Win64 into a single entity, or mixes cross-compilation with release pipeline changes without a deliberate reason.
- **Gate content:** Show the seed description, the proposed Windows target (arch/OS pair), included and excluded scope, and the proof needed to decide whether design should start. Include any known MinGW toolchain prerequisites.

### `ideation`

The captain greenlights a task for design: determine how Go's cross-compilation flags (`GOOS=windows`, `GOARCH=386` or `amd64`) apply to this codebase, identify any CGO surface that needs a MinGW cross-compiler, write acceptance criteria as end-state properties with `Verified by:` clauses reproducible on Linux, and define the test plan.

- Split each acceptance criterion by how it is verified: **offline** (a `file` command, `objdump` output, or cross-compile exit code a fresh agent reproduces) or **interactive** (requires a physical Windows machine to execute the binary). Declare the split at ideation — interactive ACs are validated by the captain on a real Windows box, not by new Linux-side automation.
- **Inputs:** The seed description, Go module layout (`go.mod`, `cmd/`, `internal/`), the existing `.goreleaser.yaml` to understand current build targets, and the MinGW package names available (`mingw-w64`, `gcc-mingw-w64`).
- **Outputs:** Updated task body with: proposed build flags and cross-compiler invocations, a list of AC items each with an explicit `Verified by:` clause naming a command or exit code, a test plan distinguishing offline checks from Windows-only smoke tests, and a clear out-of-scope section (e.g., no Windows installer, no CI loop in this task).
- **Good:** ACs are falsifiable — each names the concrete change that would flip it. The test plan lists the exact cross-compile commands a fresh agent can run. The offline/interactive split is explicit.
- **Bad:** ACs that say "the binary works on Windows" without naming a Linux-reproducible proof step. Vague scope that lumps both 32-bit and 64-bit into one AC. Missing out-of-scope section.
- **Gate content:** Show the selected approach (Go flags + MinGW invocations), risk evidence (any CGO usage, any platform-specific build tags), the offline `Verified by:` proof for each AC, and the interactive (Windows-only) smoke test list.

### `implementation`

The design is approved and the change is built in a dedicated worktree on a feature branch. The worker modifies source, build scripts, and/or the GoReleaser config to add the Windows target, then cross-compiles on Linux to produce a `.exe` and verifies it with `file` and `objdump`.

- When a finding arrives, follow `## Review-finding disposition`: investigate read-only, preserve its evidence, propose materiality/ownership/disposition, and obtain FO authorization before any candidate edit, commit, or reviewer rerun.
- **Inputs:** The ideation AC list, the accepted approach (Go flags, cross-compiler invocations), the current `cmd/spacedock/main.go` and `internal/` layout, and the existing `.goreleaser.yaml`.
- **Outputs:** Code changes that add the Windows target (build tags if needed, goreleaser stanzas), a successful cross-compilation run (`GOOS=windows GOARCH=... CGO_ENABLED=0 go build ./cmd/spacedock/`), a `file` output confirming PE format and correct bitness, and a stage report recording the commands run and their outputs.
- **Good:** Cross-compile succeeds with zero errors. The `file` command confirms "PE32 executable" for 386 or "PE32+ executable" for amd64. No new platform ifdefs leak into shared code. Changes are minimal — only what is necessary to satisfy the AC.
- **Bad:** Cross-compilation passes but leaves unreachable dead code behind. Using `// +build` instead of the modern `//go:build` constraint syntax. Silently skipping CGO surface without documenting why `CGO_ENABLED=0` is acceptable. A stage report that pastes tool output without naming which AC each output satisfies.

### `validation`

A fresh agent independently verifies the implementation's deliverable against the ideation ACs, reproducing each `Verified by:` clause from scratch on the worktree branch without reading the implementation's stage report. The validator checks what was produced; it does not produce it. Either gate-approve to `done` or reject back to `implementation` with concrete fixes.

- **Small-change fast path.** If the diff is a goreleaser stanza addition only (no source changes), scale validation proportionally — reproduce the cross-compile, check the PE output, done. Match rigor to blast radius.
- **Inputs:** The worktree branch (checked out fresh), the ideation AC list, the implementation's stage report (read after independent reproduction to compare, not before).
- **Outputs:** Non-empty Stage Report recording: each AC, the exact command run to verify it, the observed output, and PASSED/FAILED per AC. Any review findings under workflow labels (Material / Deferred risk / Polish / Needs decision). A gate recommendation: approve to `done` or reject to `implementation` with the specific failing ACs and concrete required fixes.
- **Good:** Every offline AC is reproduced independently with the exact commands from the ideation test plan. Findings that are Polish or Deferred risk are named and recorded rather than silently dropped. The gate recommendation cites specific evidence.
- **Bad:** Trusting the implementation's self-report without re-running the cross-compile commands. Passing validation when a `file` output shows unexpected architecture. Treating a missing interactive (Windows-only) AC as a blocker — those are deferred and documented, not a gate fail.
- **Gate content:** Show non-empty Stage Report results, the cross-compile commands run and their outputs, the `file`/`objdump` evidence for each AC, any reviewer findings with disposition, and whether delivery can proceed. Interactive (Windows-only) ACs must be listed as "deferred to captain's Windows smoke test" rather than omitted.

### `done`

Terminal state: the task's PR is merged (tracked via the `pr` field and the `pr-merge` mod), `completed` set, `verdict: PASSED`, entity archived. Reached via real merge, not a manual flag flip.

## Review-finding disposition

Every finding enters this checkpoint when it arrives during implementation, validation, a detached audit, consequential FO quick work, or a correction routed from a rejected gate.

1. The reviewer owns observation, not task ownership or authorization.
2. The worker preserves the finding, investigates without candidate mutation, records the four evidence fields, and proposes materiality, task ownership, and disposition separately. Its `actor:ensign` round Resolution is advisory.
3. The FO sends a distinct `fix`, `decline`, `hold`, or `route for decision` authorization through the runtime's addressable-worker boundary.
4. The validator recommends `PASSED` or `REJECTED`; a new finding re-enters step 1.
5. Only the captain changes approved scope, accepted value, thresholds, tolerance, or acceptance criteria.
6. After revise is selected, rejection routing transports the evidence, workflow classifications, authorized dispositions, and concrete assignment unchanged; it never re-triages.

Before FO authorization, candidate bytes and Git HEAD stay unchanged, no candidate commit is made, and no reviewer rerun starts. Read-only file/history inspection, non-mutating reproductions, existing tests, and adversarial work in a throwaway checkout are allowed. After authorization, perform only that disposition; `hold` and `route for decision` forbid mutation and rerun. Changed evidence re-enters the checkpoint, and an unobservable runtime authorization means hold and re-consult.

The four evidence fields are: released user and normal workflow; observable harm; affected value AC or non-negotiable boundary; and trigger evidence.

- **Material:** all four fields establish supported-workflow harm to a value AC or protected boundary.
- **Deferred risk:** the trigger is hypothetical, unsupported, unobserved, or outside current promises; record its promote-to-material condition.
- **Polish:** no current user-visible loss or protected boundary is at risk.
- **Needs decision:** the task cannot own the required scope, product, or compatibility decision.

## Workflow-specific rules

The FO/ensign operating contract governs generic stage semantics and proof discipline. Tasks in this workflow inherit those rules. The rules below add Windows cross-compilation specifics.

- **Repo-mutation worktree layer.** `implementation` and `validation` run in a dedicated worktree; `validation` is `fresh` for an independent check. PR state lives on the `pr` field, managed by the `pr-merge` mod — there is no `pr_open` or `awaiting_merge` stage.
- **Cross-compile is the offline proof gate.** A cross-compilation success (`go build ./cmd/spacedock/`) plus `file` output confirming PE format and correct bitness is the minimum offline AC evidence. Any AC whose only proof is "the binary works" without a reproducible Linux command is invalid.
- **CGO surface must be declared at ideation.** If any dependency or internal package uses CGO, the ideation AC must name the MinGW cross-compiler (`i686-w64-mingw32-gcc` or `x86_64-w64-mingw32-gcc`) and the `CC=` override. If the codebase is pure Go (`CGO_ENABLED=0`), state that explicitly.
- **Interactive ACs are deferred, not skipped.** ACs that require executing the `.exe` on a real Windows machine are marked `deferred: captain-windows-smoke` at ideation and excluded from the validation gate. The captain runs these manually after the PR is merged.
- **No prose-grep over instruction files.** A string match over this README or the FO/ensign contract never proves a behavioral claim — it only proves the file contains what was written. Offline proof must come from a reproducible command whose output can fail.
- **Evidence must be able to fail.** Each AC's cited evidence names the concrete change that would flip it. An author who cannot name what would make the evidence fail has not shown it can fail.

## Workflow State

View the workflow overview:

```bash
spacedock status --workflow-dir docs/windows-support
```

Find dispatchable tasks ready for their next stage:

```bash
spacedock status --workflow-dir docs/windows-support --next
```

## Task Template

```yaml
---
id:
title: Task title here
status: backlog
source:
started:
completed:
verdict:
score:
worktree:
issue:
pr:
mod-block:
---

## Problem

{What Windows target is missing and why it matters — e.g., "spacedock cannot be cross-compiled to Windows 32-bit; users on 32-bit Windows cannot install the binary."}

## Proposed approach

{The Go cross-compilation flags and MinGW toolchain invocations needed. Concrete enough that a worker can run the commands directly.}

## Acceptance criteria

Each AC names a property of the finished task (not a stage action) and how it is verified.

**AC-1 — {End-state property.}**
Verified by: {exact command + expected output or exit code — something a fresh agent can reproduce on Linux that can fail if the property is absent.}

## Test plan

{Offline checks (cross-compile + file/objdump) and interactive checks (Windows machine execution), estimated cost, which are deferred.}

## Out of scope

{What this task deliberately does not address — e.g., "No Windows installer. No CI loop for this PR. Cygwin not tested."}
```

## Commit Discipline

- Commit status changes at dispatch and merge boundaries
- Commit task body updates when substantive
- Implementation commits land on the worktree branch; merge to main happens via the `pr-merge` mod after PR review
