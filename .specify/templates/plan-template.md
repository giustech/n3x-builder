# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]

**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: [Go 1.26 with `GOTOOLCHAIN=local` (constitution, "Stack"); note any deviation or NEEDS CLARIFICATION]

**Primary Dependencies**: [only what this feature uses from the constitution "Stack" (e.g., `cobra`, `n3x-sdk`, `golang.org/x/tools`); anything else goes to Complexity Tracking, or NEEDS CLARIFICATION]

**Storage**: [N/A: the builder has no state; list the files read or written, e.g., workspace `/n3x/work`, `/n3x/build`, `/out`, `/etc/n3x/*.yaml`]

**Testing**: [`go test -race` with `testify`, golden files, malicious corpus, fuzzing; coverage ≥ 85% (≥ 90% in `internal/symbols` and `artifact`)]

**Target Platform**: [Linux amd64 and arm64 inside BuildKit, in `sidecar-builder` and/or `controller-builder`]

**Project Type**: [CLI and build images (`n3x-plugin`, `n3xci`, `images/`)]

**Performance Goals**: [goals from the constitution "Metas não funcionais" this feature affects and how they are measured, or N/A]

**Constraints**: [e.g., no module network after step 2, no credentials, identical `SHF_ALLOC` hash across hosts, `/out` untouched on failure]

**Scale/Scope**: [e.g., number of plugins per build, targets × architectures covered]

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Source: `.specify/memory/constitution.md` (v1.0.0). Principles and MUST/NEVER (DEVE/NUNCA) rules
admit no exception in a plan; changing one requires a constitution amendment. Mark each gate as
passed, or N/A with the reason.

- [ ] **I. Falhar no build, nunca em produção**: every defect the feature can detect before
  runtime stops `add` or `build` with an `N3X-Bxxx` code; the failing step is the last one to run.
- [ ] **II. Build reproduzível**: generated code and compiled output keep identical `SHF_ALLOC`
  sections across hosts and architectures; generated lists have a fixed order; nothing outside the
  build image and the resolved modules enters the build.
- [ ] **III. Determinismo, sem fallback silencioso**: no version swap, `latest` or `replace`; a
  failed `build` leaves `/out` untouched and a failed `add` leaves the workspace untouched; every
  skipped step and every fallback is visible (`skipped` status, warning).
- [ ] **IV. Zero credencial**: no credential by file, env var, argument, `ARG`, secret mount or
  `.netrc`; only allowlisted `ARG`s reach `go`; no import of the SDK `secret/`; URLs with
  credentials and absolute cache paths never reach the log or the manifest.
- [ ] **V. Contratos só crescem por adição**: codes, exit codes, rule names, warning kinds,
  manifest fields, JSON events and `io.n3x.*` labels are only added; each addition updates
  `docs/erros.md`, golden files and the contract docs in the same PR.
- [ ] **VI. Erros acionáveis**: every failure carries code, rule, reason, "como resolver" and the
  `docs/erros.md` link; each step lists all its failures (up to 100); warnings never change the
  exit code.
- [ ] **VII. Imagens imutáveis**: no tag overwrite or deletion, no floating tag; base images and
  actions pinned by digest or SHA.
- [ ] **VIII. Regras do SDK reusadas, nunca reimplementadas**: SDK rules (ID, `n3x-plugin.yaml`,
  `sdk` range, namespace, `schemagen`, `artifact`) are called, never copied; a missing SDK rule is
  requested upstream and faked in tests.
- [ ] **IX. Segurança visível**: `govulncheck` only warns; insecure operator settings raise
  `sumdb_off` or `insecure_fetch`; checksum mismatch fails with `N3X-B015`; every new bypass becomes
  a corpus case.
- [ ] **Architecture**: pipeline rules in the core as pure functions; external resources only
  through ports (Go interfaces) and adapters; module network only in the resolution adapter at
  step 2; output only through the reporter (`internal/report`).
- [ ] **Stack and scope**: Go 1.26, `cobra`, `testify` and the listed dependencies only (others go
  to Complexity Tracking); nothing from the v1 out-of-scope list.
- [ ] **Tests and CI**: TDD from failure scenarios; at least one test per `N3X-Bxxx` code (code,
  rule, exit code, `/out` untouched); golden files; corpus; coverage ≥ 85% (≥ 90% in
  `internal/symbols` and `artifact`); double compilation with hash comparison stays green.

## Project Structure

### Documentation (this feature)

```text
specs/[###-feature]/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)
<!--
  ACTION REQUIRED: Keep only the paths this feature touches and expand them with
  real files. The layout comes from the constitution ("Arquitetura"); a new
  top-level area or internal/ package must be justified in Complexity Tracking.
  Go tests live next to the code (*_test.go); fixtures go in testdata/.
-->

```text
cmd/n3x-plugin/          # CLI (cobra): add, build, version, targets
internal/
├── catalog/             # target catalog
├── image/               # image identity
├── workspace/           # /n3x/work
├── resolve/             # step 2: the only adapter that uses the module network
├── check/               # step 3: compatibility (pure)
├── analysis/            # step 4: static analysis and govulncheck
├── generate/            # step 5: main.go, registry, schemas
├── compile/             # step 6
├── symbols/             # step 7: symbols and load test (loadtest/)
├── manifest/            # step 8
└── report/              # reporter: the only output path
targets/                 # <target>.yaml catalog
images/                  # build image Dockerfiles and images/tools
testdata/                # test plugins, malicious corpus, fixtures, golden files
ci/n3xci/                # CI tool (cobra)
compat/                  # published images and tested pairs (append-only)
examples/                # operator Dockerfiles, generated at release
```

**Structure Decision**: [List the packages and directories this feature creates or
changes, and which are ports, core (pure functions) or adapters]

## Complexity Tracking

> **Fill ONLY for choices the constitution leaves open** (e.g., a dependency outside the
> "Stack", a new code area). Principles and MUST/NEVER rules cannot be waived here: they
> need a constitution amendment.

| Choice | Why Needed | Simpler Alternative Rejected Because |
|--------|------------|-------------------------------------|
| [e.g., new Go dependency] | [current need] | [why the stdlib or a listed dependency is not enough] |
| [e.g., new internal/ package] | [specific problem] | [why an existing area does not fit] |
