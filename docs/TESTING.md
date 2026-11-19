# Regression testing and evidence

## What was verified

The inspected model regression harness completed **3,526,424 assertion evaluations** in a verified run on November 18, 2026. This number counts calls to an assertion helper, not independent test cases, randomized trials, user sessions, or a coverage percentage.

The commercial source and harness are private. This showcase documents the testing system and verified result; readers cannot build or reproduce the suite from this repository. There is no public CI status badge or public test workflow.

## How a run starts

In the private project, test.sh compiles production model and geometry files together with Tests/main.swift using swiftc with optimization enabled. It then executes the generated command-line program. It does not launch SwiftUI, an iPhone simulator, or Apple-service integration tests.

The compiled production files are TeamHistory.swift, CityVenue.swift, CountryModel.swift, WorldTour.swift, GameModel.swift, SyncModel.swift, CircuitCatalog.swift, and CircuitGeometry.swift. Naming these files documents the boundary; their contents are not included here.

The harness's assertion helper increments a counter and terminates on failure. Scenario code creates careers, supplies explicit race seeds, advances/settles sessions, serializes values, and checks expected results. Parameter sweeps repeat assertions over many inputs. Some expensive loops aggregate violations and assert once at the end.

## Why the count is 3,526,424

The original final numerical summary reports 3,526,416 before eight additional career-deletion assertions. A temporary instrumented copy of the harness counted the full execution and recorded **3,526,424**. Application source was not changed for that instrumentation.

| Assertion family | Evaluations |
| --- | ---: |
| Sampled road width remains in range | 515,520 |
| Adjacent width samples transition smoothly | 515,520 |
| Both road edges avoid folding backward | 1,031,040 |
| Both road edges stay within map bounds | 1,031,040 |
| All other assertion-helper calls | 433,304 |
| **Total** | **3,526,424** |

The four road families make up approximately 87.7% of the total. They evaluate 720 samples for each of 716 stored geometry entries, including historical compatibility entries. Those entries are not additional advertised locations or race weekends.

The published game scope is seven tours, 64 distinct locations, and 322 scheduled championship weekends. Retaining older geometry allows previously saved races to keep their route after catalog changes.

## Coverage by engineering concern

| Concern | Representative checks |
| --- | --- |
| Economy and settlement | Affordable upgrades, bounded funds/wear, no premature settlement, repeated settlement rejection |
| Determinism | Equal input and explicit seed produce matching generated timing in tested scenarios |
| Strategy | Compound selection, early pit calls, stint extension, pending orders, completed-lap preservation |
| Opponents | Persisted strategy identities, difficulty-dependent planning, bounded hardware development |
| Driver careers | Experience growth, training, market refresh, retirement, identity persistence |
| Save integrity | JSON round trips, supported legacy data, malformed snapshot rejection, in-progress route continuity |
| Cloud policy | Equal/one-sided/divergent baseline decisions, unsupported/oversized/invalid snapshots |
| Geometry | Lap seam continuity, sampled distance uniformity, coarse segment crossings, width/edge safety |
| Traffic | Completed/aborted passes, forward progress, lateral smoothness, containment, spacing, pit merges |
| Progression | Championship gates, exhibitions, achievements, historical credit, career reset file selection |

Traffic sweeps run sampled races across the stored layouts. They inspect positions and car pairs, accumulate failures, and assert on those totals. The many inner comparisons are not all individual counted assertions. One verified sweep recorded 649 completed passes and 829 abandoned attempts; those are results of its synthetic workload, not population-wide gameplay statistics.

Some progression scenarios force wins to exercise unlock paths. They establish rule correctness rather than proving that natural gameplay difficulty is balanced.

## Reproducibility and its limits

Race scenarios use explicit seeds with the custom SeededRandom generator. Strategy data and generated race inputs are retained for save/resume checks. The harness also tests rebuilding transient traffic after clearing the cache.

The verified run used optimized Swift compilation on an arm64 Mac with Apple Swift 6.4.2. Temporary file fixtures use incidental unique names. Cross-toolchain and cross-architecture bit-for-bit equivalence has not been established. Normal gameplay chooses new weekend seeds; a saved career retains generated results and inputs rather than a complete seed-only replay specification.

The results describe the inspected application snapshot. Reproducing them requires the private sources and harness; future releases may change the assertion count.

## Performance evidence is separate

An existing synthetic traffic benchmark was recompiled against the inspected model. A single optimized run prepared a ten-car, 72-lap workload with five scheduled stops per car in **0.244 seconds** on the verification Mac; all cars completed the scheduled distance.

This measures model preparation for that synthetic input. It is not an iPhone benchmark, a sustained frame-rate measurement, a cold-start distribution, or a comparison establishing a speedup. Separate scenery rendering, saving, memory, thermal, and UI scheduling costs are outside that measurement.

## What this suite does not establish

- AppleServices and ExpansionStore are not compiled into this harness. The cloud decision model and purchase progression policy checks are not transport or transaction integration tests.
- There is no automated StoreKit refund/restore, CloudKit multi-device race, or Game Center account-switch suite here.
- SwiftUI navigation, accessibility behavior, rendering appearance, lifecycle interruptions, and physical-device smoothness require separate checks.
- No line/branch coverage metric, Thread Sanitizer result, device energy profile, or controlled long-running performance distribution is claimed.
- Fail-fast execution does not provide the isolation or full failure reporting of a test framework.

Recommended next steps are service injection and fault tests, lifecycle/UI automation, CI for the private model suite, and device profiling. These are proposed improvements, not features represented as already implemented.
