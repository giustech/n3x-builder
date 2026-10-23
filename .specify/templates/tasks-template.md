---

description: "Task list template for feature implementation"
---

# Tasks: [FEATURE NAME]

**Input**: Design documents from `/specs/[###-feature-name]/`

**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: MANDATORY. The constitution ("Testes") requires TDD from failure scenarios and counts as an explicit TDD request: test tasks come before implementation tasks and must FAIL first. Every `N3X-Bxxx` code in the spec's Failure Scenarios gets at least one test that checks code, rule, exit code and that `/out` (or the workspace, for `add`) is untouched. Use `testify` with the reporter in strict mode.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **CLI**: `cmd/n3x-plugin/`; **pipeline**: `internal/<area>/` (core as pure functions, ports as Go interfaces, adapters separate)
- **Tests**: `*_test.go` next to the code; fixtures, test plugins, malicious corpus and golden files in `testdata/`
- **Other areas**: `targets/`, `images/` (and `images/tools/`), `ci/n3xci/`, `compat/` (append-only), `examples/`
- Adjust to the Structure Decision in plan.md

<!--
  ============================================================================
  IMPORTANT: The tasks below are SAMPLE TASKS for illustration purposes only.

  The /speckit-tasks command MUST replace these with actual tasks based on:
  - User stories from spec.md (with their priorities P1, P2, P3...)
  - Feature requirements from plan.md
  - Entities from data-model.md
  - Endpoints from contracts/

  Tasks MUST be organized by user story so each story can be:
  - Implemented independently
  - Tested independently
  - Delivered as an MVP increment

  DO NOT keep these sample tasks in the generated tasks.md file.
  ============================================================================
-->

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create project structure per implementation plan
- [ ] T002 Initialize [language] project with [framework] dependencies
- [ ] T003 [P] Configure linting and formatting tools

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

Examples of foundational tasks (adjust based on your project):

- [ ] T004 Define ports (Go interfaces) for the external resources this feature uses in internal/[area]/ports.go
- [ ] T005 [P] Write fakes for those ports in internal/[area]/fake_test.go (also fakes for SDK rules not yet released)
- [ ] T006 [P] Register new `N3X-Bxxx` codes, rules and warning kinds in the reporter catalog (internal/report)
- [ ] T007 Regenerate docs/erros.md from the catalog
- [ ] T008 Wire the step into the pipeline order without changing other steps' contracts

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - [Title] (Priority: P1) 🎯 MVP

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Tests for User Story 1 (MANDATORY - TDD) ⚠️

> **NOTE: Write these tests FIRST, ensure they FAIL before implementation**

- [ ] T010 [P] [US1] Failure-scenario test for N3X-B[xxx] (code, rule, exit, `/out` untouched) in internal/[area]/[name]_test.go
- [ ] T011 [P] [US1] Golden-file test for [generated output] in internal/[area]/[name]_test.go with testdata/[area]/
- [ ] T012 [P] [US1] Malicious-corpus case for [bypass] in testdata/corpus/[case]/ (if the story touches analysis)

### Implementation for User Story 1

- [ ] T013 [P] [US1] Implement [rule] as a pure function in internal/[area]/[file].go
- [ ] T014 [P] [US1] Implement [adapter] behind its port in internal/[area]/[adapter].go
- [ ] T015 [US1] Wire [step/command] in cmd/n3x-plugin/[command].go (depends on T013, T014)
- [ ] T016 [US1] Report failures and warnings only through the reporter (internal/report)

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - [Title] (Priority: P2)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Tests for User Story 2 (MANDATORY - TDD) ⚠️

- [ ] T018 [P] [US2] Failure-scenario test for N3X-B[xxx] in internal/[area]/[name]_test.go
- [ ] T019 [P] [US2] Golden-file test for [output] in internal/[area]/[name]_test.go

### Implementation for User Story 2

- [ ] T020 [P] [US2] Implement [rule] as a pure function in internal/[area]/[file].go
- [ ] T021 [US2] Implement [adapter] behind its port in internal/[area]/[adapter].go
- [ ] T022 [US2] Wire [step/command] in cmd/n3x-plugin/[command].go
- [ ] T023 [US2] Integrate with User Story 1 components (if needed)

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - [Title] (Priority: P3)

**Goal**: [Brief description of what this story delivers]

**Independent Test**: [How to verify this story works on its own]

### Tests for User Story 3 (MANDATORY - TDD) ⚠️

- [ ] T024 [P] [US3] Failure-scenario test for N3X-B[xxx] in internal/[area]/[name]_test.go
- [ ] T025 [P] [US3] Golden-file test for [output] in internal/[area]/[name]_test.go

### Implementation for User Story 3

- [ ] T026 [P] [US3] Implement [rule] as a pure function in internal/[area]/[file].go
- [ ] T027 [US3] Implement [adapter] behind its port in internal/[area]/[adapter].go
- [ ] T028 [US3] Wire [step/command] in cmd/n3x-plugin/[command].go

**Checkpoint**: All user stories should now be independently functional

---

[Add more user story phases as needed, following the same pattern]

---

## Phase N: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [ ] TXXX [P] Regenerate docs/erros.md and update contract docs for every added code, rule, warning kind or manifest field
- [ ] TXXX [P] Add fuzz targets for new readers of external input (run nightly)
- [ ] TXXX Check coverage: ≥ 85% overall, ≥ 90% in internal/symbols and artifact
- [ ] TXXX Check reproducibility: double compilation gives the same `SHF_ALLOC` hash
- [ ] TXXX Check the log never shows credentials in URLs or absolute module-cache paths
- [ ] TXXX Run `go vet`, `gofmt -l`, `go test -race ./...`, `gosec`, `govulncheck` and `actionlint`
- [ ] TXXX Measure the affected "Metas não funcionais" as stated in plan.md
- [ ] TXXX Run quickstart.md validation

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed in parallel (if staffed)
  - Or sequentially in priority order (P1 → P2 → P3)
- **Polish (Final Phase)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P2)**: Can start after Foundational (Phase 2) - May integrate with US1 but should be independently testable
- **User Story 3 (P3)**: Can start after Foundational (Phase 2) - May integrate with US1/US2 but should be independently testable

### Within Each User Story

- Tests MUST be written and FAIL before implementation
- Ports before core rules and adapters
- Core rules (pure functions) before adapters
- Adapters before CLI wiring
- Story complete before moving to next priority

### Parallel Opportunities

- All Setup tasks marked [P] can run in parallel
- All Foundational tasks marked [P] can run in parallel (within Phase 2)
- Once Foundational phase completes, all user stories can start in parallel (if team capacity allows)
- All tests for a user story marked [P] can run in parallel
- Models within a story marked [P] can run in parallel
- Different user stories can be worked on in parallel by different team members

---

## Parallel Example: User Story 1

```bash
# Launch all tests for User Story 1 together:
Task: "Failure-scenario test for N3X-B[xxx] in internal/[area]/[name]_test.go"
Task: "Golden-file test for [generated output] in internal/[area]/[name]_test.go"

# Launch the core rule and the adapter for User Story 1 together:
Task: "Implement [rule] as a pure function in internal/[area]/[file].go"
Task: "Implement [adapter] behind its port in internal/[area]/[adapter].go"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL - blocks all stories)
3. Complete Phase 3: User Story 1
4. **STOP and VALIDATE**: Test User Story 1 independently
5. Deploy/demo if ready

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently → Deploy/Demo (MVP!)
3. Add User Story 2 → Test independently → Deploy/Demo
4. Add User Story 3 → Test independently → Deploy/Demo
5. Each story adds value without breaking previous stories

### Parallel Team Strategy

With multiple developers:

1. Team completes Setup + Foundational together
2. Once Foundational is done:
   - Developer A: User Story 1
   - Developer B: User Story 2
   - Developer C: User Story 3
3. Stories complete and integrate independently

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- Verify tests fail before implementing
- Commit after each task or logical group
- Stop at any checkpoint to validate story independently
- Avoid: vague tasks, same file conflicts, cross-story dependencies that break independence
