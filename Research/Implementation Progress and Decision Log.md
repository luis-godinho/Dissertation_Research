---
status: evolving
tags: [research, implementation, decisions, progress]
updated: 2026-11-01
---

# Implementation Progress and Decision Log

## Purpose and related notes

Record what has been implemented, the decisions taken and their rationale, the evidence available, and what is deliberately deferred. This is a dated progress record, not a replacement for the target architecture.

- [[Home]]
- [[Architecture/Architecture Decisions|Architecture Decisions]]
- [[Architecture/Dicoogle Integration|Dicoogle Integration]]
- [[Architecture/Caching and Prefetching|Caching and Prefetching]]
- [[Specification/Project Requirements|Project Requirements]]
- [[Specification/Key Management|Key Management]]
- [[Specification/Threat Model|Threat Model]]
- [[Research/Evaluation Protocol|Evaluation Protocol]]
- [[Research/Performance and Usability|Performance and Usability]]
- [[Research/Security Tests|Security Tests]]
- [[diagrams/README|Diagrams]]

## Evidence and status conventions

- **Decided:** selected scope or design constraint, not proof of implementation.
- **Implemented / focused tests reported passing:** recorded implementation results with the verification scope stated.
- **Resolved within tested scope:** a specific correction and executed regression result have been reported; broader coverage is not implied.
- **Runtime not verified:** no live service/browser demonstration establishes the behaviour yet.
- **Planned:** intended next work, not completed work.
- **Deferred:** excluded from the current milestone, not abandoned.

This snapshot is based on implementation reports and the subsequent correction report supplied by the project owner on 2026-10-24. Tests were not independently rerun for this research note. A documentation commit does not publish the implementation code or establish deployment/runtime success.

The 2026-10-24 snapshot and next-checkpoint statement are retained as dated history. The continuation dated 2026-10-28 records subsequent implementation, verification and superseding decisions. Earlier results keep their original scope; later work is not retroactively attributed to them.

## Current snapshot — 2026-10-24

| Area | Recorded state | Evidence / limit |
|---|---|---|
| Implementation vehicle | Incremental work from `dicoogle-next` dev on `feat/dissertation-frame-baseline` | Initial base `75427c8a75d6ca7cb9cc65cb536473c1c67d4813` |
| Implementation revision | Local commit `b6476c0`, `feat(backend): add native-frame retrieval endpoint` | Reported committed non-Markdown changes; not pushed; detailed Markdown reports remain local/untracked |
| Synthetic DICOM fixture | Implemented; nine focused tests initially reported passing | Included again in backend regressions; structural validation, not external IOD certification |
| Native-frame endpoint | Implemented; latest focused run reports 44 tests passing | 18 protocol + 26 application; zero failures/errors/skips; full backend suite not run |
| Encapsulated/fragmented PixelData rejection | Corrected and service-level regression reported passing | Fragments returns 415; missing/invalid supported native PixelData remains 422; fragment-specific HTTP multipart-body behaviour not verified |
| Isolated runtime setup and API routing | Planned | No completion/runtime evidence supplied |
| Selected-study series/instance enumeration | Planned for the demonstration path | Reuse existing QIDO APIs; no study-search semantics rewrite |
| Native-frame browser transport, decode and viewport | Planned | No browser/Cornerstone integration result recorded for the new endpoint |
| P0/P1 demonstration and timing | Not yet verified | No live latency, eight-viewport, or passing 500 ms result |
| Bounded native RAM, encrypted OPFS and shared-key access | Deferred | Final architectural target remains unchanged |

The base revision identifies the starting code. The latest local implementation commit is recorded separately. Remote availability of that implementation commit has not been established; no remote link is asserted. Future evidence should record the tested commit and any additional uncommitted diff/file snapshot.

## Decisions and rationale

### ID-01 — Incremental development from the selected dev baseline

Decision: continue on `feat/dissertation-frame-baseline`, created from the selected dev revision. `feat/frontend-frame-retrieval` is a read-only implementation reference, not the development vehicle or an automatic bulk import.

Reason: preserve the chosen current codebase and introduce small changes with controlled dependencies and verification. Existing reference work is not treated as nonexistent, but is not imported wholesale.

Consequence: retain current dev fixes, existing full-instance viewing and unrelated user changes. A branch merge, rebase, broad dependency import or deployment replacement is not implied.

### ID-02 — Demonstrate retrieval before reusable cache and encryption layers

Decision: first establish a limited native-frame P1-like path in one Cornerstone viewport, retaining the full-instance viewer as local baseline P0.

Reason: fix frame identity, representation, transport, errors and measurement boundaries before adding cache reuse and cryptographic overhead.

Consequence: this milestone does not demonstrate P2, P4a, P4b or clinical suitability. The final proposed design remains shared encrypted OPFS plus bounded plaintext native-frame RAM and separately managed decoded/worker/GPU state.

### ID-03 — Restrict the initial supported pixel profile

Decision: Explicit VR Little Endian (`1.2.840.10008.1.2.1`), native uncompressed PixelData, SamplesPerPixel=1, MONOCHROME2, BitsAllocated=16, BitsStored=16, HighBit=15 and unsigned PixelRepresentation=0, for the selected Secondary Capture SOP classes.

Reason: make correctness deterministic without simultaneously implementing compressed-frame parsing, transcoding or broad modality support.

Consequence: unsupported layouts, bit depths, signed pixels, colour, SOP classes and transfer syntaxes are explicitly rejected. Syntax must not be inferred merely from payload size. Encapsulated support remains a future, named-transfer-syntax extension.

### ID-04 — Keep representations distinct

| Representation | Role |
|---|---|
| Complete Part-10 instance | Existing P0 retrieval baseline |
| `native-frame-v1` | Selected native frame payload; intended future cache unit |
| `wrapped-single-frame-v1` | Possible transient decode envelope; not the network payload or persistent cache unit |
| Decoded pixels | Decoder/Cornerstone memory, separate from native-byte RAM |
| Rendered PNG preview | Existing preview path; excluded from native-frame claims |

Decision: no `/dic2png`, silent frame substitution or legacy fallback to manufacture apparent success. A future transient wrapper must preserve the requested frame's metadata and representation association.

### ID-05 — Use explicit frame-list and resource identity semantics

```text
GET /api/dicom-web/studies/{studyInstanceUID}/series/{seriesInstanceUID}/instances/{sopInstanceUID}/frames/{frameList}
```

The application context supplies `/api`; the controller is mapped under `/dicom-web`.

- API frames are one-based; convert internal zero-based indices explicitly.
- Reject malformed, duplicate, nonpositive and out-of-range values; no silent deduplication or frame-zero/frame-one substitution.
- Preserve valid unique request order, including `8,1,3`.
- Limit the demonstration to eight unique requested frames. This is an engineering bound, not a universal protocol limit.
- Match the complete study/series/SOP tuple against the stored and parsed instance. Unknown/mismatched tuples use the same not-found outcome; no SOP-only fallback.

### ID-06 — Return binary multipart frames and an explicit cache policy

Decision: `multipart/related` with `application/octet-stream` parts, per-part frame association, provenance and `Cache-Control: no-store`. Invalid requests do not produce successful partial multipart output.

Reason: preserve selected-frame identity and make the demand-only baseline observable.

Consequence: this limited experimental profile is not comprehensive DICOMweb conformance. No-store expresses HTTP cache policy, not proof of absence of process/browser/GPU residue.

### ID-07 — Bound complete-instance reads; do not claim selective disk I/O

Decision: read/parse the complete stored instance within an enforced bound, then extract the selected native frames.

| Engineering limit | Value |
|---|---:|
| Stored-instance input | 64 MiB |
| Native frame | 16 MiB |
| Sum of selected response frame payloads | 64 MiB |
| Unique requested frames | 8 |

The input cap is enforced during reading rather than after an unbounded read. Dimensions/counts/sizes/offsets use checked arithmetic before allocation/extraction. Multipart framing is additional to the payload cap and bounded separately by the request/part limits.

Consequence: `frameSelectiveResponse=true`, `frameSelectiveBackendIo=false`. Backend instance-read bytes/counts and selected response payload bytes are separate measurements. Parsing and response construction can create additional copies; the input cap is not a peak-memory claim. Ranged reads and large-instance frame indexing remain deferred.

### ID-08 — Preserve a genuinely demand-only P1 path

Decision: no reusable native-byte cache or adjacent prefetch in the new demonstration path; revisits perform fresh frame-body retrieval. Do not invoke existing load-all prefetch behaviour for this path.

Reason: obtain a measurable baseline before P2/persistent-cache comparisons.

Consequence: distinguish actual HTTP requests, cache policy, native hits and decoded-cache reuse. Cornerstone transient memory is not native-byte caching. A bypass flag alone does not prove requests occurred.

### ID-09 — Reject stale display work during frontend integration

Decision: use request identity/generation checks and cancellation where supported so an earlier completion cannot overwrite the current frame or update a closed viewport.

Reason: asynchronous frame-selection correctness is required before broader cache/session lifecycle work.

Consequence: examine internal loader/render/cache side effects; a generation check after stale publication is insufficient. Full ownership cleanup and key lifecycle remain separate milestones.

### ID-10 — Keep the first runtime demonstration isolated and synthetic

Decision: synthetic-only storage/index roots; no real patient files, broad home/root mounts, production data or deployment claim.

Reason: frame-endpoint authentication does not resolve study-policy or broader deployment concerns.

Consequence: startup, fixture registration/indexing and browser routing require explicit configuration review and observation. Isolation is a prerequisite, not proof of production security or user separation.

### ID-11 — Keep timing claims narrower than the project target

Decision: distinguish request/headers/body completion, decode, display-call, animation-frame and supported render-event boundaries.

Reason: post-fetch display timing, playback FPS and clinical interpretation duration are not interchangeable with end-to-end viewer delay.

Consequence: render-call return, render events and rAF are operational proxies, not physical presentation proof. Eight frames in one object or sequentially viewed in one viewport are not eight simultaneous displays. The eight-image/500 ms target still requires a defined workload/event/conditions and an actual measured result.

## Dated implementation and verification records

### 2026-10-24 — Synthetic fixture completed

- One synthetic study, two series, four instances.
- Series A: eight distinct 64x64 unsigned 16-bit MONOCHROME2 frames in a Multi-frame Grayscale Word Secondary Capture object (`1.2.840.10008.5.1.4.1.1.7.3`).
- Series B: three single-frame Secondary Capture companions (`1.2.840.10008.5.1.4.1.1.7`).
- Explicit VR Little Endian; 8192 native bytes/frame, 65536 total native pixel bytes in A. Full Part-10 file sizes include metadata and are larger.
- Deterministic patterns, fixed synthetic identifiers/dates and frame/instance hashes in a manifest. The manifest generation timestamp is not a stable fixture-content identifier.
- Test-scope generator/tests: `backend/dicoogle-next/src/test/java/org/dicoogle/app/testing/fixtures/SyntheticDicomFixtureGenerator.java` and `SyntheticDicomFixtureGeneratorTest.java`.
- Ignored generated output: `backend/dicoogle-next/target/synthetic-dicom-fixtures/synthetic-study-v1/`.

Evidence supplied: nine focused tests passing with zero failures/errors/skips. Checks included fresh re-read, frame values/distinctness, manifest consistency, deterministic regeneration and controlled reset/overwrite refusal. These tests were included again in the backend regression selections.

Limits: structural/module-attribute validation, not external IOD certification, clinical validity, runtime ingestion or performance evidence.

### 2026-10-24 — Native-frame endpoint implemented; initial focused run

Recorded components:

- `NativeFrameService`, `NativeFrameLimits`, `NativeFrameResponse` in the DICOMweb protocol module.
- Frame route in `DicomwebController`.
- Existing locator/storage abstractions, authentication and synthetic fixture reused.
- Service, controller and controller-limit tests; existing full-instance and fixture regressions retained.
- Selected-frame binary multipart output, request-order preservation, bounds/profile/error checks and no preview/legacy fallback.
- Global `HttpMediaTypeNotAcceptableException` mapping in `ApiExceptionHandler`, correcting unacceptable media negotiation from generic 500 to 406. This is a cross-cutting change and was treated as such in review.
- Metrics for requests/success/failure, instance-read bytes, selected-frame response bytes and backend reads; not browser timing evidence.

Initial reported focused run: 17 protocol + 26 application = 43 tests, zero failures/errors/skips, build success. This result preceded the fragment-specific review finding and is retained as historical evidence, not the latest count.

### 2026-10-24 — Fragment-specific review finding

The supported-profile validator classified every non-byte-array PixelData value as structural inconsistency (422). The fragment-specific `Fragments` representation needed the contract's unsupported-organisation outcome (415). Existing profile tests did not exercise that branch.

Decision: make the smallest explicit distinction rather than adding compressed support. Missing/invalid supported native PixelData keeps 422. Encapsulated/fragmented data is rejected with 415. This is the current experimental contract, not a statement that encapsulated DICOM is inherently corrupt.

### 2026-10-24 — Correction completed and focused regressions rerun

Owner-supplied correction record:

- `NativeFrameService.java:19,252-258`: explicit `Fragments` rejection with 415; missing/invalid PixelData retains 422.
- `NativeFrameServiceTest.java:191-201,415-456`: new regression using Explicit VR Little Endian and valid preceding metadata so it reaches the fragment-specific branch, rather than succeeding only through earlier syntax rejection.
- Local implementation commit: `b6476c0`, `feat(backend): add native-frame retrieval endpoint`; not pushed.
- Detailed implementation report: `docs/dissertation/implementation-reports/01b-native-frame-endpoint.md`, section 10 appended and read back.
- Markdown documentation and local context files remained excluded/untracked according to the supplied record.

Reported executed verification, from the implementation backend directory:

```bash
./mvnw -Dspotless.apply.skip=true -Dspotless.check.skip=true -pl dicoogle-next-protocol-dicomweb,dicoogle-next -am -Dtest=NativeFrameServiceTest,DicomwebRetrieveServiceTest,SyntheticDicomFixtureGeneratorTest,NativeFrameControllerTest,NativeFrameControllerLimitTest,DicomwebControllerTest -Dsurefire.failIfNoSpecifiedTests=false test
```

| Latest focused run | Tests | Failures | Errors | Skipped |
|---|---:|---:|---:|---:|
| Protocol | 18 | 0 | 0 | 0 |
| Application | 26 | 0 | 0 | 0 |
| Total | 44 | 0 | 0 | 0 |

Result supplied: BUILD SUCCESS, 29.290 seconds. The initial 43-test run and latest 44-test run are repeated verification selections, not 87 distinct tests.

Status: the identified fragment-status blocker is resolved within the reported service-level test scope.

Still not verified: the full backend suite, frontend/browser/live runtime, fragment-specific HTTP multipart-body behaviour, external IOD conformance, or performance. A service exception assertion does not independently establish HTTP body behaviour for that same input. No native-cache, encryption, study-policy or shared-user success is implied.

## Deferred features and re-entry conditions

| Item | Current boundary / rationale | Re-entry condition |
|---|---|---|
| Bounded native-byte RAM (P2) | First establish correct P1 transport/display | Verified baseline, explicit budget and ownership/eviction tests |
| Full cache/session cleanup | Minimal stale-display handling first | Reusable caches with delayed-work/logout/user-switch tests |
| AES-GCM envelope | Outside retrieval milestone | Approved context/key-use rules and isolated round-trip/negative tests |
| Ciphertext OPFS | Storage should not hide representation defects | Tested crypto, bounded namespace/metadata/publication design |
| Session-only P4a | Optional intermediate/comparison mode | Explicit scope approval and tested same-session key lifecycle |
| Study-level authorisation | Authentication alone is insufficient | Approved server-enforced policy and allow/deny/bypass tests |
| Backend KEK/DEK custody | Required for final P4b, not initial P1 | Approved provisioning, durable protection, scope/version/retirement and release tests |
| Authorised User B reuse (P4b) | Depends on real policy and recoverable scoped keys | Fresh permission/key access, ciphertext reuse and denied-user/cleanup evidence |
| Compressed/encapsulated formats | Current fixed-size extraction is uncompressed only | Relevant dataset and named syntax; correct fragment/frame reconstruction, response type and decoder tests; never assume one fragment equals one frame |
| Directional prefetch | Keep initial demand-only behaviour observable | Stable baseline, bounded scheduling and measured off/on comparison |
| Advanced prediction/adaptation | Not necessary for the first working parts | Specific hypothesis and measurable incremental benefit |
| Eight simultaneous images / 500 ms | One-viewport milestone; target not operationally fixed | Approved workload/event definition, relevant dataset and repeatable complete measurements |
| Multi-tab coordination | Concurrent publication and session/key ownership need design | Stable single-context operation, locking/recovery and concurrent tests |
| Selective backend disk I/O | Complete bounded instance is still read | Evidence justifies indexing/ranged reads and measured storage comparison |
| Broader modalities/clinical datasets | Narrow synthetic OT/SC fixture first | Dataset provenance, required syntax support and validated security/deployment prerequisites |

Deferred does not mean abandoned. The final target remains shared encrypted OPFS plus bounded RAM. No offline-access, cross-device sharing or new key hierarchy is selected by this milestone.

## Unresolved security and deployment concerns

Earlier inspections identified study-policy gaps, anonymous preview/filesystem routes, broad roots/mounts, token handling and lifecycle/header/logging concerns in the inspected baseline. Keep these tracked until specific corrections and evidence exist.

- Static findings are not automatically demonstrated runtime exposures.
- The new authenticated frame route does not secure every sensitive retrieval path or implement study-level policy.
- Synthetic data does not make a publicly exposed or broadly mounted deployment safe.
- No real-data, production-security or meaningful multi-user-isolation claim follows from the current endpoint.
- HTTP cache policy and ordinary cleanup are not proof of forensic sanitisation.

## Next checkpoint

With the fragment-specific correction and focused rerun now recorded, the next bounded work is isolated local configuration and API routing, followed by selected-study enumeration, binary-safe transport, one-viewport native-frame navigation, minimal stale-result handling and reproducible P0/P1 evidence.

No completion evidence for that live/frontend work has been supplied. Do not start reusable caching, encryption or shared-key work automatically. Freeze a stable partial demonstration according to the actual presentation deadline and observed progress.

## Maintenance

For each future entry record:

```text
Date:
Decision or implementation change:
Reason / alternatives not selected:
Implementation branch and tested commit or diff snapshot:
Files/components affected:
Verification actually executed and observed result:
Checks not run / limitations:
Current status:
Deferred work / next dependency:
```

Preserve dated history and explain supersession. Keep design decisions, implementation presence, focused test success, runtime results and performance claims separate. Detailed evidence remains in the implementation checkout's `docs/dissertation/implementation-reports/`; this note summarises it. Omit machine-specific absolute paths, credentials and patient data.

## Change history

- 2026-10-24: recorded the dev-based implementation vehicle, synthetic fixture and limited native-frame decisions; preserved the initial 43-test run and review finding, then recorded its correction, local unpushed commit `b6476c0`, and latest 44-test focused verification. Cache/security/performance extensions remain explicitly deferred.
- 2026-10-28: added native-frame browser verification, the revised visual-cache demonstration scope, native RAM core and sliding-window retention results, and the next viewer-integration decisions. Earlier dated entries remain historical.
- 2026-11-01: recorded the RAM-only browser evidence, the decision to add a plaintext OPFS + RAM control ahead of the encrypted tier, the storage adapter and its test results, the adapter review findings, and the next sequencing decisions. Earlier dated entries remain historical.
- 2026-11-01 (later): recorded the move-only adapter decision and fix, the dual-window policy, the P3 session, viewer and residency strip, the snapshot-freshness decision, and the real-browser evidence for the plaintext OPFS control. Earlier dated entries remain historical.

## Continuation — 2026-10-28

### Current snapshot

| Area | State | Verification / limit |
|---|---|---|
| Implementation vehicle | Same `feat/dissertation-frame-baseline` branch | Latest inspected revision `fe2bf244a229725ed4efd74ee6b02a84bac45661`, with additional local changes tracked separately |
| Instance selection | Existing selected-instance path retained | The enumeration replacement was reverted; no QIDO-first restoration is required |
| Native-frame transport and wrapper | Implemented and tested | Bounded binary multipart validation; transient Part-10 decode container; no persistent wrapper storage |
| Native-frame viewport | Implemented and exercised through nginx and Chromium | One visible viewport, frame navigation and stale-selection handling |
| P0/P1 controls | Live baseline established | Full-instance and selected-frame hashes match the synthetic fixture; nine frame responses for 1–8 plus revisit |
| Native RAM core | Implemented; focused tests passing | 58-test frontend suite; no viewer integration at that stage |
| Sliding-window retention | Implemented; focused tests passing | 70-test frontend suite; pure policy, no neighbour fetching |
| P2 viewer integration | Selected next work | No completed integration or live P2 hit result yet |
| Residency visualisation | Planned | Actual cache state must drive colours and counters |
| OPFS, encryption and shared-key access | Deferred | OPFS/BOTH remain disabled in the RAM demonstration; no plaintext OPFS stage selected |

The 58- and 70-test results are successive suite runs, not independent totals to add together. A passing cache-core or policy test is not evidence that the browser already uses the cache.

### ID-12 — Expand the demonstration while preserving comparison controls

Decision: extend the first presentation to include a separately labelled P2 native-byte RAM tier, sliding-window retention, live residency colours, counters and lifecycle checks. Preserve P0 full-instance viewing and P1 demand-only retrieval.

Reason: frame display establishes the retrieval path, but does not demonstrate what is retained, reused or evicted. Those transitions need to be visible and independently measurable.

Consequence: this supersedes the earlier first-presentation exclusion of P2, not the definition of P1. Implement cache core, window policy, viewer integration, visualisation and lifecycle changes separately, then rehearse the combined demonstration. No cache hit, performance gain or secure persistence is claimed before it is exercised.

### ID-13 — Obtain live pixel context without replacing enumeration

Decision: use the existing selected-instance locator and load pixel attributes through instance metadata once per open. Expose the same source file's File Meta Information TransferSyntaxUID through `X-Source-Transfer-Syntax`, keeping group `0002` out of the JSON dataset.

Reason: current search/index fields do not supply the full pixel context, and dataset metadata alone does not carry the stored file's transfer syntax. Hardcoding that syntax or bootstrapping validation from an initial frame response was not selected.

Consequence: validate identity and the supported profile before frame retrieval; reject missing or invalid source syntax. Count metadata separately from frame-body requests. The header describes source encoding, not negotiated JSON encoding, integrity, freshness or study permission. The earlier enumeration replacement stays reverted.

### ID-14 — Bound native payload reuse and make ownership explicit

Decision: use one RAM cache per active viewer scope, keyed by principal/session/study/series/SOP/frame, native representation, transfer syntax and source version. Copy incoming payloads and return copies on hits; do not retain mutable caller keys or buffers.

Reason: native bytes are the reusable experimental unit. Wrappers, decoded Cornerstone images and worker/GPU memory are separate representations with separate ownership.

Consequence: configurable demonstration defaults are 32 KiB retained bytes, 8 KiB per entry, five entries and 64 history events. Hard byte limits take priority. Returned-copy and bypass byte totals are cumulative, not current occupancy or process-memory measurements. Monotonic generations reject late insertion; close clears owned entries. Public events use safe identities and explicitly signal history overflow so the visual consumer can resynchronise. These settings are not clinical sizing or transport limits.

### ID-15 — Retain a sliding window without silently prefetching

Decision: retain a configurable one-frame-behind/one-frame-ahead window by default, superseding the earlier radius-two planning value. Keep window epoch distinct from session/request generation.

Reason: retention of visited frames can be tested without simultaneously introducing speculative requests or a larger scheduler.

Consequence: desired neighbours are not automatically resident or fetched. Evict outside-window entries first, then apply deterministic distance/recency pressure decisions. Prefer the current frame only within hard limits; oversized frames may remain transiently displayable without reusable residency. Admission accounts for incoming bytes and same-key replacement. Wrong-context, stale-epoch, outside-window and oversized completions do not trigger pressure eviction. Prefetch remains a separately controlled later comparison.

### ID-16 — Visualise actual residency rather than intended storage

Decision: add a compact numbered frame strip with absent, native RAM, committed OPFS and both-tier states; show requested/displayed frames, desired window, pending operations and errors separately. Use labels or shapes as well as colours.

Reason: displayed content is not automatically a native cache entry, a desired window can contain missing frames, and an eventual frame may exist in both tiers.

Consequence: only absent/RAM states become active in the first RAM demonstration. OPFS/BOTH stay explicitly disabled until real storage and viewer integration exist; queued writes must not appear as completed persistence. Show actual frame calls, hits/misses, eviction, native bytes and budget. No unencrypted OPFS mode has been selected. The final target remains shared encrypted OPFS plus bounded native RAM, with session-only P4a distinct from backend-recoverable P4b.

### ID-17 — Add P2 through the existing session and presenter

Decision: retain P1 as default and add P2 to the existing native-frame session. P2 uses native RAM on a hit and the same authenticated validated transport on a miss. Both paths create fresh wrappers and request-unique presentation identities.

Reason: avoid a second viewer/cache and prevent an old decoded image from being mistaken for the selected native-byte source.

Consequence: keep a small immutable entry-ID/key mapping, not another payload store; recheck generation, window and occupancy before mutation. Mode switches invalidate pending presentation as well as fetch, reset owned reusable cache state and keep the last displayed source truthful. Clear RAM does not globally purge unrelated viewers. Restrict the fixed `synthetic-study-v1` version to recognised fixture instances; disable P2 when valid source-version scope is unavailable. Implementation and integration verification remain pending.

### 2026-10-25 to 2026-10-27 — Native-frame frontend and browser baseline

Implemented authenticated one-frame transport, bounded binary multipart parsing, the transient single-frame wrapper, bounded metadata loading and a dedicated native-frame viewport. Missing/blank-token requests were corrected to reject before fetching and covered by zero-fetch assertions. Hidden staging and unique image identities prevent superseded work from being deliberately published as the new selection.

Live verification through nginx and a temporary Chromium context established:

- P0 opens and retrieves the expected 66,876-byte full instance, with a matching fixture hash.
- P1 displays frames 1, 4 and 8. Native payloads are 8,192 bytes; multipart bodies are 8,337 bytes. Selected payload hashes, including revisit 1, match the fixture manifest.
- Sequential selections 1–8 and revisit 1 produce nine frame responses and one metadata response. Auth/search/preview traffic is counted separately. Surrounding PNG thumbnails are not native payloads or fallback.
- Delayed frame 2 is aborted after frame 3 is selected; frame 3 remains visible. The observed close scenario produces no additional frame response.
- No native payload persistence is observed in the inspected application-origin storage APIs. Existing token/plugin state is not frame content; this does not establish absence from process, worker or OS memory.

The existing running environment and indexed synthetic fixture were reused without rebuild or reindex. The fixture was served through a pre-existing broad bind mount and network-published development services. Matching hashes establish the expected synthetic bytes, not compliance with the earlier isolation objective or production security. That distinction remains unresolved.

Limits: no live wrapper-PixelData hash, complete delayed-decode/close-race coverage, backend disk/TTFB, worker/WASM, GPU or physical-paint measurement. Selection-to-response/commit timings are application boundaries, not a passing eight-image/500 ms result.

### 2026-10-27 to 2026-10-28 — Native RAM core and window policy

Implemented `nativeRamCache.ts` with copied ownership, immutable keys, hard limits, deterministic LRU, failed-replacement atomicity, monotonic generation checks, close rejection and bounded safe events/snapshots. Twelve new tests brought the focused frontend suite to 58 passing, zero failures. Type-check, lint and build passed. Viewer/session/backend paths were unchanged.

Implemented `nativeRamWindowPolicy.ts` with clamped bounds, desired-versus-resident state, outside-window and pressure eviction, current-frame preference, stale/context admission rejection, oversized bypass and incoming/replacement accounting. Twelve additional tests brought the suite to 70 passing, zero failures. Type-check, lint and build passed. Tests applied decisions through the real cache's invalidation API; the policy itself does not mutate or fetch.

The latest inspected revision is `fe2bf244a229725ed4efd74ee6b02a84bac45661`, with local additions identified separately. These results establish the core and policy, not browser P2 cache hits, a completed residency strip or a performance benefit. Remote publication of the underlying changes is not asserted here.

### Next checkpoint — 2026-10-28

Integrate the existing native cache and window policy into the viewer without replacing P0/P1. Verify direct selections `1,2,1`: three frame requests in P1, two in retained P2. Also test eviction/revisit misses, fresh wrapper input on hits, mode switches, clear RAM, close and pending-presentation invalidation. Distinguish unit/integration tests from live browser behaviour.

Then add the truthful residency strip and lifecycle handling, and capture the P0/P1/P2 comparison with matching cache snapshots, counters and network observations. OPFS, encryption, study policy, shared keys, prefetch and the eight-image/500 ms target remain separate work. No plaintext persistence, cross-user security or completed P2 demonstration is implied.

## Continuation — 2026-11-01

### Current snapshot

| Area | State | Verification / limit |
|---|---|---|
| Implementation vehicle | Same `feat/dissertation-frame-baseline` branch | RAM-only live evidence taken at local revision `cdfbff7f601d663453b9b7b77d6d3dc70af1b822` (`feat(webapp): add bounded P2 residency and lifecycle integration`); plaintext OPFS work since then is uncommitted; remote publication is not asserted |
| P0/P1/P2 RAM-only demonstration | Verified in two browser rehearsals | Headless Chromium through nginx at 1440×900; 119 focused frontend tests passing; OPFS disabled during these runs |
| Plaintext OPFS + RAM control (P3) | Scope decided; storage adapter implemented | 145 focused tests passing against an in-memory OPFS substitute; no real-browser OPFS run yet |
| Adapter review | Completed | Three High and four Medium findings, concerning peak usage, residue and accounting rather than false residency |
| Real-browser adapter check | Planned | Temporary browser profile and loopback origin; not run |
| Dual-window policy | Approved; not yet implemented | Pure decision function, no storage or network access |
| Viewer integration, OPFS/BOTH visuals, lifecycle handling, optional dual-tier prefetch | Planned | None started |
| Encryption, key custody, shared reuse | Deferred | Unchanged |

The 119- and 145-test results are successive runs of the same growing focused suite, not independent totals to add together. Tests that pass against an in-memory OPFS substitute establish the adapter's logic, not real browser storage behaviour.

### ID-18 — Keep the RAM-only run as the comparison baseline

Decision: retain the RAM-only P0/P1/P2 browser evidence unchanged as the baseline against which the dual-tier mode is compared. The executed prefetch comparison (trace 1, 5, 6, 7, 8 at 700 ms pacing with explicit termination) is authoritative over the shorter trace used in earlier planning.

Reason: the final demonstration includes a persistent tier, so each set of evidence must remain attributable to the configuration that produced it.

Consequence: dual-tier evidence is collected separately and does not overwrite the RAM-only results, which were taken with the OPFS lanes disabled.

### ID-19 — Add a plaintext OPFS + RAM control (P3) ahead of the encrypted tier

Decision: add P3, an opt-in, synthetic-only control combining the bounded RAM tier with a plaintext OPFS tier. The viewer still starts in P1; P0, P1 and P2 keep their meanings. Limits: RAM 1 behind / 1 ahead, 32 KiB, 8 KiB per entry, five entries; OPFS 3 behind / 3 ahead, 64 KiB committed, 8 KiB per entry, eight entries, and a 128 KiB root-wide cap on all application-owned files (committed, temporary, replacement and lease files). One active write and one queued write per viewer, with pending payload bounded at 16 KiB; a full queue is a visible error, not an unbounded retry.

Reason: the persistence tier's mechanics, window behaviour and visuals can be measured and shown before cryptography is added, using synthetic data only. A 3/3 window needs at most seven frames (57,344 bytes), which fits the 64 KiB cap with one entry of headroom. A frame held in both tiers is counted once in each; storage accounting is not deduplicated.

Consequence: this supersedes, for the plaintext control only, the earlier statements that no unencrypted OPFS mode is selected and that OPFS follows the cryptographic work. It is not encryption, access control, freshness, shared-user reuse or secure erasure, and it must never be presented as the final secure design. Its storage namespace stays separate from any later encrypted tier. The final target remains shared encrypted OPFS plus bounded native RAM.

### ID-20 — Store only verified native bytes and verify them on every read

Decision: store exactly the validated `native-frame-v1` bytes — never the transient wrapper, a whole instance, a PNG or decoded pixels — as one file per frame with a fixed 80-byte header (magic, format version, payload length, a digest over the full cache key, and a payload SHA-256). Reads require a caller-supplied fixed fixture digest taken from an immutable table of the 11 synthetic frames (eight multiframe, three single-frame). Size, header, context, self-stored checksum and fixture digest are all checked before any byte is returned or promoted to RAM. If hashing is unavailable, the adapter reports `digest-unavailable` and nothing is treated as verified.

Reason: the expected identity must come from the request, never from what a stored file claims, and a corrupted or mismatched candidate must stop before decode.

Consequence: the checksum and digest detect accidental corruption and fixture mismatch. They are not authentication against someone who can modify the files, and the fixed fixture version is not a freshness mechanism. No file name or header carries a UID, token, principal or session value.

### ID-21 — Bound ownership, concurrency and recovery of plaintext storage

Decision: each viewer uses a random opaque namespace under a dedicated versioned root, with one active writer per namespace and no reuse across reload, session, user or device. Publication writes a temporary file, closes it, verifies its size and only then publishes the final entry (rename when `move()` exists; copy, verify and delete otherwise); residency is declared only after publication. A namespace is treated as abandoned, and removed, only when its Web Lock can be acquired and its five-minute lease (30-second heartbeat) has expired. Without lock support, an existing lease or existing files block admission instead of being guessed at. Recovery and deletion stay inside the dedicated root. Clear RAM keeps the disk tier. Mode exit, close and observed logout invalidate reads immediately and then drain and delete the owned namespace. Window movement deletes entries outside the OPFS window and never triggers fetching. Unsupported storage or a quota failure yields labelled degraded RAM-only display with a visible storage error and no false OPFS residency. Manual Clear OPFS and Clear Both controls act on the owned namespace only.

Reason: stale plaintext must not accumulate silently, and one viewer must not delete another live viewer's files.

Consequence: a crash can leave plaintext behind until a later bounded recovery pass, and deletion is best-effort application cleanup, not media sanitisation. The root cap is shared across live viewers, so an earlier viewer can leave a later one little room; this is accepted and must surface as an explicit capacity error. No multi-tab transaction or crash-atomicity claim is made.

### ID-22 — Paint P3 residency only from committed storage

Decision: grey means absent from the observed tiers; blue, committed native RAM; purple, committed and size-verified plaintext OPFS; a split tile, both tiers. Pending write (`S`), pending delete (`X`), storage errors, requested, displayed and the two desired windows (`Wr` for RAM, `Wo` for OPFS) are separate markers. A queued write never turns a tile purple, and a delete in progress keeps the entry present but ineligible until deletion is confirmed. The source of the last completed selection (`network`, `ram` or `opfs`) is recorded separately from any promotion to RAM. A permanent banner states that the mode is plaintext, synthetic and not encrypted.

Reason: window membership, a queued write and committed storage are different facts, and showing them as one would misstate the experiment.

Consequence: visuals follow the adapter's committed, deleting and error events and its bounded snapshots, not an expected animation or filesystem polling. Eight frames shown in one viewport remain navigation targets, not simultaneous images.

### ID-23 — Fix the adapter's High findings before integrating it

Decision: after the pure dual-window policy and a trimmed real-browser check, correct the adapter's High findings, together with the inexpensive Medium ones, in a bounded task before it is wired into the viewer. Whether the publication fallback stays is left open until the browser check records whether the target browser supports `move()`. The alternatives are to require `move()` and report unsupported browsers as degraded, or to reserve and clean up the fallback completely.

Reason: integration will rely on the adapter's byte accounting and cleanup, which the review found incomplete on the fallback and late-write paths.

Consequence: the order is policy, browser check, adapter correction, integration, visuals, lifecycle handling and optional dual-tier prefetch, then live evidence. None of these is started yet.

### 2026-11-01 — RAM-only live evidence

Two equivalent rehearsals ran from fresh headless-Chromium contexts through the existing nginx route at 1440×900, after a frontend-only rebuild so the served bundle contained the P2, residency and prefetch interface. The backend, volumes, fixture and index were untouched. Both runs produced identical phase-level counts and no console or page errors.

| Check | Observed |
|---|---|
| P0 full instance | 66,876 bytes; hash matches the fixture manifest |
| P1, frames 1–8 then revisit 1 | Nine frame-body responses, none prefetch; frame-1 hash matches |
| P2, prefetch off, 1, 2, 1 | Two frame requests; the revisit is a RAM hit; 8,192 bytes per entry; two entries / 16,384 bytes |
| P2, move to frame 8, revisit 1 | Earlier entries evicted; frame 1 fetched again |
| Clear RAM | 0 entries / 0 bytes; the displayed picture stays as transient state and is not relabelled reusable |
| P2 to P1 | No hidden reuse; the first P1 frame performs a network request |
| Invalid response for frame 6 | Rejected before commit with a visible error; 0 entries retained |
| Delayed network (1.2 s) | Pending marker shown while the previous frame stays displayed; final frame committed |
| Close, reopen, logout | No late frame responses in the observation window; reopen starts from an empty cache |
| P0 after the P2 work | Opens and renders |

Controlled prefetch comparison (trace 1, 5, 6, 7, 8; fresh sessions; 700 ms pacing; prefetch switched off and settled at the end): with prefetch off there were five demand requests; with it on there were two demand and three prefetch requests, and frames 6–8 were served from RAM (three useful completed prefetch hits). The total was five frame requests in both arms, because the same frames were fetched, only earlier. No speedup is claimed.

Limits: the delay was applied to the network response only, so this is not delayed-decode evidence. No total browser memory, GPU, physical paint, TTFB, backend I/O, OPFS, encryption or freshness result was obtained. The browser harness printed target-closed noise at teardown after the evidence was written, and the intentionally intercepted frame request appears as an aborted request. Three harness-only corrections were needed before the successful runs; no application source changed. Timing values are in-page selection-to-source and selection-to-commit boundaries only.

### 2026-11-01 — Plaintext OPFS storage adapter implemented

Implemented a standalone adapter (`nativeFramePlaintextOpfs.ts`) with `open`, `get`, `put`, `invalidate`, `clearOwned`, `drain`, `close`, `recover`, `snapshot`, `eventsSince`, `activate` and `capabilities`, plus the immutable 11-frame digest table (`nativeFrameFixtureDigests.ts`). The adapter performs no fetching, decoding or presentation, and no viewer, session or mode code changed.

Behaviour recorded: exact `native-frame-v1` payloads behind the 80-byte header; opaque aliases only in names, snapshots and events; the 64 KiB / 8 KiB / 8-entry and 128 KiB root caps with peak-byte reservation before any temporary file is created; one active and one queued write with queue-full rejection; failed deletes keep their bytes counted against the root cap; replacement keeps the old complete entry until the new one is published; a bounded root scan (32 namespaces, 64 files); bounded, revisioned events (`queued`, `writing`, `committed`, `invalidated`, `deleting`, `deleted`, `rejected`, `error`, `cleared`) in which `committed` follows confirmed publication only.

Verification: 145 focused tests passing (the 119 existing plus 26 new), type-check, lint and build passing, with the existing configuration and chunk-size warnings. The digest table was compared with the generated manifest when it was present, and the comparison reports "not run" instead of passing when the manifest is absent; both branches were exercised.

Limits: every test used an in-memory OPFS substitute. Real browser OPFS, quota, Web Locks, persistence and crash behaviour were not exercised. The substitute always provides `move()`, so the no-`move()` publication path was never tested.

### 2026-11-01 — Adapter review findings

A read-only review of the adapter and its tests found peak-usage, residue and accounting gaps. None showed a false `committed` state.

| Finding | Severity | Summary |
|---|---|---|
| F1 | High | The no-`move()` fallback reserves one new file while temporary and final copies coexist, so it can exceed the root cap near the limit |
| F2 | High | A failed or partial fallback can leave an unindexed final file that recovery does not remove and that still counts against the cap |
| F3 | High | A write that outlives the 5 s drain can leave files after clear or close, and late cleanup failures are not added to the failed-delete total |
| F4 | Medium | A failed `open` can leave a live lease that blocks another owner until it expires |
| F5 | Medium | Recovery does not distinguish busy, skipped or scan-limit outcomes from "nothing to recover" |
| F6 | Medium | The 128 KiB root cap is global, so the first live namespace has an advantage; two live namespaces are not tested |
| F7 | Medium | An active temporary file can be counted twice (observed bytes and reservation) |
| F8 | Low/Medium | `activate` changes in-memory ownership before the lease write succeeds |
| F9 | Low | Own-temporary-file recovery assumes exclusive namespace ownership when locks are unavailable |

The existing tests still support the absence of false committed residency. They do not cover the fallback residue, late-write cleanup residue, failed-open lease, busy and scan-limit results, two live namespaces, activation failure or the no-lock temporary-file case.

### Planned — dual-window policy and trimmed real-browser check

Pure dual-window policy (approved, not implemented): a decision function over actual descriptors that distinguishes desired, actual and planned states. RAM keeps 1/1 and OPFS 3/3, each clamped at the instance ends; leaving the RAM window removes only the RAM copy, and leaving the OPFS window plans deletion of the disk copy only. Pending writes and deletes reserve bytes without creating or removing residency, and a planned deletion stays present until confirmed. The root-cap projection is a peak physical figure (external bytes, owned entries, pending deletes and writes, incoming and replacement copies). Pressure order is outside the window, then farthest, then least recently used, then a stable tie-break. Missing neighbours are never fetched or published, and a stale or wrong-context plan is rejected before any pressure decision.

Real-browser check (approved in trimmed form, not run): a temporary browser profile and loopback origin, limited to smoke namespaces under the owned root, exercising the native OPFS handles only. It stores, reads and verifies one synthetic frame, rejects a wrong digest, clears and closes, and records whether `move()` is available. The sanitised result is kept before the temporary directory is removed. The fallback arm is held until the `move()` decision above.

### Next checkpoint — 2026-11-01

Implement the pure dual-window policy, run the trimmed real-browser check, then decide the publication fallback and correct the adapter's High findings (with F4, F5 and F7 and tests for two live namespaces and activation failure). Only then integrate the OPFS tier into the viewer as P3, add the committed-state residency visuals and Clear OPFS / Clear Both controls, cover the asynchronous lifecycle cases, and optionally extend the one-neighbour prefetch to P3. Live dual-tier evidence follows, compared against the RAM-only baseline with equal traces, pacing and termination.

Standing limits: the plaintext tier is not encryption, access control, freshness or secure erasure; leftovers after a crash and the shared root cap are accepted limitations; real OPFS behaviour is unverified until the browser check; encryption, key custody, study-level policy and shared reuse remain separate, deferred work. No performance, total-memory, physical-paint or eight-image/500 ms claim is made.

## Continuation — 2026-11-01 (P3 integration and browser evidence)

This continuation supersedes the "planned" and "not yet implemented" states in the earlier 2026-11-01 snapshot, and resolves the open fallback question in ID-23. Earlier entries keep their original scope.

### Current snapshot

| Area | State | Verification / limit |
|---|---|---|
| Implementation vehicle | Same `feat/dissertation-frame-baseline` branch | Local commits, none pushed: `aa74628` (fixture digest verification), `24d8a3e` (plaintext OPFS adapter), `9eba97b` (dual-window policy), `3ec80a4` (P3 session integration), `fdc82dc` (P3 entry and status), `a9ab048` (P3 residency strip and clear controls), `5c99d43` (P3 Clear RAM), `bcd6a56` (snapshot freshness fix). Browser evidence taken at `bcd6a56` |
| Plaintext OPFS adapter | Implemented, move-only; real-browser smoke passed | Chromium 153, temporary loopback origin and profile, owned root only; replacement of an existing entry verified |
| Dual-window policy | Implemented; unit-tested | Pure planner: no fetching, no storage mutation |
| P3 session integration | Implemented; Node-tested and browser-verified | Read order RAM, verified OPFS, network |
| P3 viewer: entry, status, strip, clear controls | Implemented; SSR-tested and browser-verified | Interaction paths exercised in the browser run, not in component tests |
| P3 real-browser evidence | Complete for the approved arms | All pre-registered request and tile predictions matched; synthetic data only |
| Focused frontend suite | 223 tests passing | Successive runs of one growing suite, not additive |
| Optional P3 prefetch | Not started | Separate decision |
| Encryption, key custody, study policy, shared reuse | Deferred | Unchanged |

Test progression of the focused suite: 145 (adapter) to 164 (dual-window policy, +19) to 175 (adapter fix, +11) to 189 (P3 session, +14) to 191 (default-configuration trace and counter-integrity tests) to 197 (entry and status, +6) to 216 (strip, +19) to 218 (Clear RAM, +2) to 223 (snapshot freshness, +5).

### ID-24 — Require native move() for plaintext publication

Decision: remove the copy-and-delete publication fallback. `open` reports move support and returns unavailable when `FileSystemFileHandle.move()` is missing; the viewer then enters the labelled degraded RAM-only state. The earlier forced no-move comparison arm is cancelled.

Reason: the fallback caused both the unreserved peak (copy and original coexisting near the root cap) and the orphan-final residue. The target browser supports `move()`, which the real-browser smoke confirmed. Removing the fallback is smaller and safer than reserving a second copy and adding an orphan-recovery protocol.

Consequence: publication peak is temp plus the previous final during a replacement, and `temporaryFileBytes` is 0 on the normal write path. A failed move on a new entry deletes the temporary file and the final name only because the final did not exist before; a failed replacement deletes only the temporary file, re-verifies the old entry (size, header, context, checksum, payload digest) and keeps it indexed. No `committed` event follows a failure. The same change fixed the other review findings: close and clear return `failed: write-in-flight` instead of claiming cleanup, keep the lock and lease, and allow a retry; `open` holds the lock throughout and writes the lease last; actual scanned bytes and unmaterialised reservations are counted separately; recovery returns `ok`, `skipped`, `partial` or `failed` with a reason; `activate` changes ownership only after the lease write succeeds; two live namespaces share the root cap without deleting each other. The multi-tab assumption for own-temporary-file recovery without locks (F9) remains documented. A move is a browser operation, not a transaction; no crash-atomicity claim is made.

### ID-25 — Plan retention separately from execution, and retry a blocked publication once

Decision: a pure policy plans RAM 1/1 and OPFS 3/3 retention over actual committed descriptors, pending reservations and external root bytes. The session executes the plan: delete, wait for confirmed deletion, replan once, then publish once. A publication still blocked after that replan is dropped with a visible capacity reason. There are no timers and no loops.

Reason: separating the decision from the effect keeps the policy deterministic and testable, and planned deletions cannot count as free space in the same plan because the bytes are only free after confirmation.

Consequence: a jump across the window needs a delete, confirm, replan sequence, and its publication may be dropped; this is visible through a counter and is not retried in the background. Window maintenance never fetches and never publishes bytes it was not given. Pending deletes stay counted and ineligible until confirmed; failed deletes remain counted.

### ID-26 — Report the origin of a P3 result separately from the legacy source field

Decision: the true origin of a completed selection is `opfsSource` and `lastSource` (`network`, `ram` or `opfs`), with RAM promotion as a separate boolean. The legacy `source` field stays `network` or `ram` so the existing viewer types are unchanged, and an OPFS hit reads as `ram` there.

Reason: an OPFS hit promoted to RAM must not be relabelled as a RAM hit, and it is not a network request. The first implementation reported `lastSource` as `ram` on an OPFS hit; a test written to check counter integrity exposed this and it was corrected.

Consequence: an OPFS hit counts as a RAM lookup miss, not a RAM hit, and leaves the network and demand counters unchanged. Evidence tables must use `opfsSource` or `lastSource` for origin and treat the legacy field as a compatibility value.

### ID-27 — Maintain windows after every completed selection and reconcile storage from adapter truth

Decision: after every completed P3 selection, including RAM and OPFS hits, run the policy and execute only its confirmed deletions. The session's OPFS map is reconciled from adapter results, events and snapshots, so a late write that completes after navigation, close or logout still updates the map even though it cannot affect the display, RAM promotion or user-facing counters.

Reason: a physically committed file that the session ignores becomes an untracked orphan, and a disk tier that is only trimmed on network writes would keep outside-window entries indefinitely.

Consequence: a late committed entry outside the OPFS window is planned for deletion. Forward all-hit navigation is impossible once the ahead entries are deleted, so OPFS hits occur only on revisits within three frames after the 1/1 RAM window has moved on. The demonstration trace is therefore `1,2,3,4,5,2`. A single publication is pending at a time; a newer payload may supersede one that has not started, an active adapter write always completes, and a payload arriving while a write is active is dropped as `publication-busy` and counted.

### ID-28 — Paint the P3 view from committed state and show an unavailable disk tier explicitly

Decision: realise the visual contract of ID-22 with a bounded 24-tile dual-tier strip, painted only from adapter-committed frames. The session exposes frame numbers and window bounds only (no aliases, keys, UIDs or payloads). In degraded and cleanup-required states the disk tier is text-labelled and hatched "unavailable/unknown" rather than grey, and the cleanup-required text says that plaintext frames may still be present on disk until the retry succeeds. The RAM-only strip is hidden in P3. A permanent banner names the mode as plaintext OPFS, synthetic and not encrypted, with no lock or shield icon.

Reason: an unobserved disk tier must not read as an empty one, and a pending write, window membership or intent must not read as residency.

Consequence: independent badges show pending write (`S`), pending delete (`X`), storage error, OPFS read in progress and network pending, alongside requested and displayed markers and the two window outlines. A failed delete keeps the physical entry, shows the error and marks it ineligible, with no deletion claim. Clear RAM, Clear OPFS and Clear Both act on the owned namespace only; Clear Both reports a partial state if OPFS cleanup fails after the RAM clear. Clear RAM keeps the disk tier.

### ID-29 — Publish P3 view state from one point and test the emitted snapshot, not only recomputed metrics

Decision: every P3 state change that affects a derived view field emits through a single publish helper, and tests compare the last emitted subscription snapshot with a freshly derived one after each awaited operation in the P3 scenarios.

Reason: the first real-browser smoke showed the pending-write badge and counter staying stale after the write had committed. The session cleared the pending publication without emitting a new snapshot, and the viewer renders from the last emitted snapshot. The Node tests had read freshly computed metrics, so they could not see it. The browser run was stopped at that defect, which was not fixed inside the evidence task, and was restarted from the smoke after a separate fix.

Consequence: all audited mutation paths now emit (pending publication, pending write and delete, committed map and counters, dropped and failed publications, reading and network-pending state, error and ineligible frames, adapter event reconciliation, Clear OPFS, close and retry, enter and exit, promotions, frame-body counters). The viewer's metrics export now includes `lastSource`. The stale badge did not recur in any later arm.

### 2026-11-01 — Real-browser adapter smoke

Chromium 153, temporary loopback origin and temporary profile, owned root only, synthetic frame. Wrong-digest write rejected; a correct 8,192-byte native frame stored as 8,272 file bytes and read back exact and verified; a read without a digest was skipped; clear and close removed the namespace; no console or page errors. After the adapter fix the smoke was rerun: a second write of the same key returned `replaced`, with one final file and no temporary file, and the second read was exact and verified. Move support was recorded as available; the no-move arm was not run, by decision ID-24.

Limits: one browser build, a loopback origin rather than the application origin, no reload persistence, quota stress, multi-tab or crash behaviour.

### 2026-11-01 — P3 browser evidence

Setup: frontend-only rebuild and restart; backend, volumes, fixture and index untouched; existing headless Chromium 153 at 1440×900; a fresh profile per arm; local traffic only; synthetic data and credentials supplied through the environment. Predictions were written before each run and not edited afterwards. Frame-body requests are counted from mode entry.

| Step (P3, RAM 1/1, OPFS 3/3) | Cumulative requests | Tiles after settle |
|---|---:|---|
| 1 | 1 | 1 both |
| 2 | 2 | 1, 2 both |
| 3 | 3 | 1 disk only; 2, 3 both |
| 4 | 4 | 1, 2 disk only; 3, 4 both |
| 5 | 5 | 1 absent (deleted); 2, 3 disk only; 4, 5 both |
| back to 2 (one slider change) | 5 | 2 both (OPFS hit, promoted to RAM); 3, 4, 5 disk only; 1 absent |

- Two runs of the P3 main trace matched. OPFS bytes were 8,192 native / 8,272 file / 8,389 root at one entry and 32,768 / 33,088 / 33,205 at four entries; `publication-busy` was 0 and pending writes and deletes were 0 after settle.
- Clear RAM after step 5 left RAM at 0 entries and the disk tier at four; revisiting 2 was an OPFS hit with promotion and still 5 requests.
- The P2 control (same trace, same viewport and pacing, prefetch off, two runs) needed 6 frame-body requests, because frame 2 had left the 1/1 RAM window. The difference is a cache-state difference, not a speed result.
- Degraded start (move removed by a test-only script) showed the labelled RAM-only state, a storage error, a hatched disk tier, no disk or both tiles, and one request.
- Mode exit and logout left the owned namespace absent, and the next P2 selection was a network request.
- A forced namespace-removal failure (test-only script) produced the cleanup-required state with the plaintext-on-disk warning; after the failure was lifted, the retry closed the session and the namespace was absent.
- All successful arms ended with no owned namespace left; no console or page errors.

Differences recorded: one aborted frame request with no matching response in the first P3 main run, with no effect on any counter, tile or displayed frame and not reproduced in the second run; its cause is not established because the frame number was not retained. Two harness-only count-check defects were corrected and are not application results. Aborted `plugins` requests are classified as non-frame. Settle times of roughly 1.4 to 2.0 s per selection are publication-lag notes, and elapsed time between selections differs across arms because the P3 arms wait for settling, so no timing comparison is made.

Limits: real browser OPFS and UI, synthetic data only. One browser build, one fixture, a loopback origin. No reload persistence, multi-tab, crash, quota-stress, performance, total-memory, physical-paint, encryption, freshness, authorisation, clinical or secure-erasure result. Plaintext frames in the profile are readable by anyone with access to it.

### Updates to deferred items

| Item | Change |
|---|---|
| Plaintext OPFS control | Now implemented and browser-verified within the synthetic profile; supersedes its earlier deferred status for this control only |
| Ciphertext OPFS, AES-GCM envelope, key custody, study policy, shared reuse | Unchanged and deferred; the plaintext control does not establish any of them |
| Optional P3 prefetch | Not started; needs a separate decision and equal-trace comparison against the RAM-only prefetch result |
| Multi-tab coordination, crash atomicity, quota behaviour | Not tested; still outside the claims |

### Unresolved concerns

- The implementation commits exist only locally, and the detailed evidence is untracked; there is no remote backup of either. Remote availability is not asserted.
- The aborted frame request in the first P3 main run is unexplained. A further run with the frame number recorded is proposed to classify it.
- The 128 KiB root cap remains global with a first-live-namespace advantage, and own-temporary-file recovery without locks assumes exclusive namespace ownership. Both are documented limits, not fixed.
- A crash can leave plaintext in the profile until a later recovery pass; deletion is application cleanup, not media sanitisation.

### Next checkpoint — 2026-11-01 (later)

Classify the aborted-request anomaly with one more run, and back up the local commits and evidence. Then decide, against the presentation deadline, between the optional P3 prefetch comparison and freezing the demonstration (rehearsal order, screenshots, and the equal-trace comparison of P1, P2 and P3). The encrypted tier, key custody, study policy and shared reuse remain separate, deferred work. No performance claim is made until a defined workload is measured.
