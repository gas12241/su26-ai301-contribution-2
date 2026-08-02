# Contribution 2: debuginfo: Add integration tests for debuginfo.Store.Exists and debuginfo.Store.Upload using the debuginfo.Client

**Contribution Number:** 2  
**Student:** George Alvarado-Salinas
**Issue:** https://github.com/parca-dev/parca/issues/1160
**Status:** Phase II Complete!

---

## Why I Chose This Issue

[1-2 paragraphs explaining why this issue interests you, how it matches your skills/learning goals, what you hope to learn]

I chose this issue for a few reasons. From a personal perspective, this issue has to do with writing tests, which is somewhat similar to what I'm doing in AI201 (which is where my interest comes from). This issue is also very active and has not had any comments on it (which has been hard to find). The problem is pretty understandable, and I think a big part of that comes from the description specifically stating what it wants from the contributor (me).

---

## Understanding the Issue

### Problem Description

From my understanding, there is a test that tests debuginfo.Store.Exists and debuginfo.Store.Upload, but it doesn't do so in a way that is good for testing. Both are testing in a giant test, where they are not tested extensively either. Considering we would want to test both separately, inside the test file would have to be a "test helper" that deals with the setup for the tests, as well as the integration tests for both debuginfo.Store.Exists and debuginfo.Store.Upload. this isn't something that is  broken, per se, but it is missing.

### Expected Behavior

There should be a focused integration test (or tests) that spin up a real `debuginfo.Store` gRPC server and drive it through the real `debuginfo.Client` (`GrpcUploadClient` in `pkg/debuginfo/client.go`), verifying the existence-check and upload RPCs work correctly over an actual gRPC connection — not just at the Go-struct level with fakes standing in for the network boundary.

### Current Behavior

There's already a real-gRPC test in `pkg/debuginfo/store_test.go` (`TestStore`, lines 74–260) that does dial a real client against a real server — so this isn't a from-scratch gap. But it has two shortcomings relative to what the issue asks for:

1. It's one large, monolithic test covering the entire upload lifecycle (initiate → upload → mark finished → existence/quality checks → debuginfod fallback) rather than focused tests specifically for `Exists`-style checks and `Upload`.
2. It opens a real TCP socket (`net.Listen("tcp", "127.0.0.1:0")`, line 98) instead of using an in-process `bufconn` listener, which is unnecessary overhead/flakiness risk for a unit-level integration test.

Also worth flagging: there's no method literally named `Store.Exists` in `pkg/debuginfo/store.go` — existence checks are exposed through `ShouldInitiateUpload`'s reason codes (`ReasonDebuginfoAlreadyExists`, `ReasonDebuginfoInDebuginfod`, etc.). The issue title's "Store.Exists" likely refers to that existence-check behavior conceptually, not a literal method — worth clarifying with whoever filed it.

### Affected Components

- `pkg/debuginfo/store.go` — `Store.ShouldInitiateUpload`, `Store.Upload`, `Store.MarkUploadFinished` (the RPCs under test)
- `pkg/debuginfo/client.go` — `GrpcUploadClient`, `NewGrpcUploadClient`, `GrpcDebuginfoUploadServiceClient` (the real client being exercised)
- `pkg/debuginfo/store_test.go` — existing `TestStore`, to be refactored/extended
- `gen/proto/go/parca/debuginfo/v1alpha1` — generated `DebuginfoServiceClient`/`DebuginfoServiceServer` used by both sides

---

## Reproduction Process

*(Not applicable in the traditional sense — this is a test-coverage gap, not a bug, so there's no failing behavior to reproduce. Reframed below as "how I set up the environment" and "how I confirmed the gap.")*

### Environment Setup

Followed `CONTRIBUTING.md`'s Prerequisites/Getting Started on macOS (arm64, macOS 26.3.1). Hit and fixed several environment-specific issues along the way (documented in full in a personal setup log, happy to share/upstream separately):

- Go/Node PATH not picked up until terminal restart after install
- `pnpm` required for `make build`'s UI step but not listed in CONTRIBUTING.md
- `env-local-test.sh` (part of `make dev/setup`) hardcodes a Linux minikube binary regardless of host OS — worked around via `brew install minikube`
- minikube's default `qemu2` driver on macOS arm64 has broken in-VM DNS resolution — switched to `--driver=docker`
- `Dockerfile.dev`'s UI build stages pinned a stale `node:16.20.2-alpine` (inconsistent with `.nvmrc`'s 22.22.2), and were missing `ui/pnpm-workspace.yaml` in the `COPY` list — both fixed locally
- `deploy/Makefile`'s `AGENT_VERSION` resolution used `grep -oP`, a GNU-only flag unsupported by macOS's BSD grep — fixed with `sed -E`

After these fixes, `make dev/setup`, `make dev/up`, and `make go/test` all complete successfully (`go/test`: 141 passed, 0 failed).

### Steps to Reproduce

1. Ran `go test -tags assert -v ./pkg/debuginfo/...` — `TestStore` passes today, confirming the *existing* real-client/real-server test path already works.
2. Read `pkg/debuginfo/store_test.go` end-to-end and confirmed it exercises the client/server boundary correctly, but as one combined lifecycle test rather than scoped `Exists`/`Upload` integration tests.
3. Searched the repo for `bufconn` usage (`grep -rn "bufconn"`) — zero matches, confirming no existing precedent for the lighter-weight in-process gRPC test pattern this issue implicitly wants.

### Reproduction Evidence

- **Commit showing reproduction:** N/A — no code change was needed to observe the gap; it's evident directly from reading `store_test.go`. That being said, here is a link to my branch that I made for this issue: https://github.com/gas12241/parca/tree/fix-debuginfo-store-1160
- **Screenshots/logs:** `go test -tags assert -v ./pkg/debuginfo/...` output showing `TestStore` passing (available on request).
- **My findings:** The building blocks for the requested integration tests already exist in `TestStore` (real `grpc.NewServer()` + real `debuginfopb.NewDebuginfoServiceClient` + `NewGrpcUploadClient`) — this is additive/refactoring work, not net-new infrastructure.

---

## Solution Approach

### Analysis

The "root cause" here is scope, not breakage: `TestStore` already proves the client/server integration path works, but it's not structured as the kind of focused, reusable integration tests the issue asks for, and it uses real sockets instead of `bufconn`, which is slightly heavier/flakier than necessary for CI.

### Proposed Solution

Extract a shared test helper that builds a real `*Store` + real in-process `bufconn`-backed gRPC server/client pair, then write focused tests: one exercising existence-check behavior (`ShouldInitiateUpload`'s reason codes) through the client, and one exercising `Upload` (chunked upload → `MarkUploadFinished` → verify bytes landed in the bucket) through the client — independent of the big combined lifecycle test.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Add integration tests that drive `debuginfo.Store`'s existence-check and upload behavior through the real `debuginfo.Client`, rather than only unit-testing internals with fakes.

**Match:** `pkg/debuginfo/store_test.go` lines 74–260 (`TestStore`) already shows the exact pattern to reuse: `objstore.NewInMemBucket()` + `NewObjectStoreMetadata` + `NewStore(...)`, a real `grpc.NewServer()` registered with `debuginfopb.RegisterDebuginfoServiceServer`, dialed via `grpc.NewClient(...)`, wrapped in `debuginfopb.NewDebuginfoServiceClient` and `NewGrpcUploadClient`. `google.golang.org/grpc` is already a direct dependency (`go.mod`), so `google.golang.org/grpc/test/bufconn` needs no new dependency.

**Plan:**
1. Add a `newTestStoreAndClient(t *testing.T) (*Store, debuginfopb.DebuginfoServiceClient, *GrpcUploadClient)` helper in `pkg/debuginfo/store_test.go` (or a new `integration_test.go`) using `bufconn.Listen(...)` instead of `net.Listen("tcp", ...)`.
2. Add `TestStoreExistsIntegration` — covers `ShouldInitiateUpload` reason codes (`ReasonFirstTimeSeen`, `ReasonDebuginfoAlreadyExists`, `ReasonDebuginfoInDebuginfod`, etc.) driven through the real client.
3. Add `TestStoreUploadIntegration` — covers the chunked `Upload` → `MarkUploadFinished` path through `GrpcUploadClient`, asserting the resulting bytes in the bucket, independent of the existence-check assertions.
4. Leave the existing `TestStore` in place (or slim it down) to avoid regressing its existing coverage while the new focused tests land.
5. Update tests to use the shared helper; run `make go/test` to confirm no regressions.

**Implement:** [Link to your branch/commits as you work]

**Review:** Confirm changes follow `CONTRIBUTING.md` (commit message format: subject ≤70 chars/body wrapped at 80 with `Fixes #1160`; run `make go/lint`; add/update tests per "Making a PR" section).

**Evaluate:** `make go/test` passes locally (141/141 today; new tests should raise that count), and CI runs green on the PR.


---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
