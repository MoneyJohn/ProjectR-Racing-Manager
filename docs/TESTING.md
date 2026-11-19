# How the regression checks work

## The verified result

A local run on **November 18, 2026** completed **3,526,424 assertion evaluations**. That is the number of calls to the harness’s assertion helper, not millions of independently written tests, user sessions, or a coverage percentage.

The application and test sources are private. This document explains the harness and its result, but the suite cannot be built or run from this showcase. There is no public CI workflow.

## Starting a run

In the private project, `test.sh` compiles the production model and geometry files together with `Tests/main.swift`, using `swiftc -O`. It then runs the resulting command-line executable. The suite does not launch SwiftUI or a simulator, and it does not connect to Apple services.

The production files it compiles are `TeamHistory.swift`, `CityVenue.swift`, `CountryModel.swift`, `WorldTour.swift`, `GameModel.swift`, `SyncModel.swift`, `CircuitCatalog.swift`, and `CircuitGeometry.swift`.

Each assertion-helper call increments a counter and stops the program if its condition fails. Scenarios create careers, use explicit race seeds, advance and settle races, round-trip saves through JSON, and check the results. Other checks loop over combinations of inputs or sampled points. Some loops collect violations and make one assertion on the total.

## Where the count comes from

The harness prints a summary at **3,526,416**, then runs eight more career-deletion assertions. A temporary instrumented copy counted the complete run as **3,526,424**. The application source was unchanged by that instrumentation.

| Assertion family | Evaluations |
| --- | ---: |
| Road width stays in range | 515,520 |
| Neighboring width samples change smoothly | 515,520 |
| Both road edges avoid folding backward | 1,031,040 |
| Both road edges stay inside map bounds | 1,031,040 |
| Other assertion-helper calls | 433,304 |
| **Total** | **3,526,424** |

The four road-geometry checks account for about **87.7%** of the total. They run at 720 sample points for each of 716 stored geometry entries, including older layouts kept for save compatibility. Repeating an assertion at many points is useful for finding geometry problems, but it does not mean each evaluation is a separately designed test.

Those stored entries are distinct from the advertised scope of seven tours, 64 locations, and 322 scheduled championship weekends. Older geometry lets an in-progress race keep its route after a catalog update.

## What gets checked

| Area | Examples |
| --- | --- |
| Economy and results | Upgrade costs, bounded funds and wear, unfinished-race rejection, repeated-settlement rejection |
| Determinism | Matching timing for the same tested inputs and seed |
| Strategy | Tire changes, early pit calls, stint extension, pending commands, preservation of completed laps |
| Opponents | Saved strategy profiles, difficulty-dependent planning, bounded component development |
| Drivers | Experience growth, training, market refresh, retirement, saved identities |
| Saves | JSON round trips, supported older saves, invalid snapshot rejection, in-progress route compatibility |
| Cloud decision rules | Equal saves, one-sided changes, divergent changes, invalid and oversized snapshots |
| Geometry | Lap seams, sample spacing, coarse segment crossings, road widths, edge bounds |
| Traffic | Passing and abandoned attempts, forward movement, lane smoothness, containment, spacing, pit merges |
| Progression | Championship gates, exhibitions, achievement eligibility, history, reset-file selection |

Traffic sweeps simulate races across the stored layouts. They compare car positions, accumulate violations, and assert on those totals; the inner comparisons are not all counted as separate assertions. One verified synthetic sweep recorded 649 completed passes and 829 abandoned attempts.

Some progression scenarios force wins to reach later unlocks. Those checks verify progression rules, not whether the game’s natural difficulty is balanced.

## Repeating the run

The race scenarios supply explicit seeds to `SeededRandom`. Save/resume checks retain generated frames, strategy data, and traffic inputs. The harness also checks that rebuilding transient traffic after clearing its cache restores the expected progress.

The verified run used optimized compilation with Apple Swift 6.4.2 on an arm64 Mac. Temporary file fixtures have incidental unique names. Bit-for-bit equivalence across toolchains and architectures has not been established. In normal gameplay, weekends use new seeds and saves retain generated race data rather than a seed-only replay specification.

Reproducing the result requires the private sources and harness. The total describes that application snapshot and can change as the suite grows.

## A separate performance check

An existing synthetic traffic benchmark was also compiled against the inspected models. In one optimized run on the verification Mac, a ten-car, 72-lap workload with five scheduled stops per car was prepared in **0.244 seconds**. Every car completed the scheduled distance.

This is a local model-preparation measurement, not an iPhone rendering benchmark or evidence of a consistent frame rate. It does not measure scenery drawing, saving, UI scheduling, thermal behavior, or memory use, and it is not a controlled before/after speedup comparison.

## Gaps in the testing

- `AppleServices` and `ExpansionStore` are not compiled into this harness. Testing cloud decision rules or purchase progression policy does not test the real CloudKit transport or StoreKit transactions.
- There is no automated StoreKit refund/restore, multi-device CloudKit conflict, or Game Center account-switch suite here.
- Navigation, accessibility, appearance, app interruptions, and device smoothness need separate UI and physical-device checks.
- There is no reported line/branch coverage, Thread Sanitizer result, energy profile, or long-running performance distribution.
- The harness stops at the first failure rather than isolating cases and reporting all failures like a test framework.

Possible next steps include injectable services for failure tests, UI/lifecycle automation, CI for the private harness, and device profiling. These are improvements to work toward, not existing coverage.
