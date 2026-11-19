# Architecture and engineering decisions

This document describes the commercial application's inspected implementation. Source filenames and type names identify the basis for the description; those source files are not distributed in this showcase.

## Boundaries and state ownership

ProjectR has one native application target. Its architecture uses an observable application store, value-based domain models, SwiftUI presentation, and concrete service objects. The boundaries below are logical, not independent Swift packages.

| Responsibility | Types and original source locations |
| --- | --- |
| Application state and coordination | GameStore in ProjectRApp.swift |
| Career rules and race data | Career, RaceSession, LapFrame, Setup in GameModel.swift |
| Weekend planning and progression | WeekendPlan, RaceStrategy, PitWallState and model extensions in WorldTour.swift |
| Traffic and overtaking | TrafficInput, TrafficPhase, TrafficRun, TrafficCache in GameModel.swift |
| Track geometry | CircuitGeometry and GeometryCache in CircuitGeometry.swift; route catalog in CircuitCatalog.swift |
| Screens and playback state | RaceView and RacePlaybackState in RaceViews.swift |
| Drawing and scenery caching | CircuitArtwork and CircuitTextureCache in Artwork.swift |
| Cloud, identity, and achievements | AppleServices in AppleServices.swift; CareerSnapshot and SyncDecision in SyncModel.swift |
| Purchase ownership | ExpansionStore in ExpansionStore.swift; implementation withheld |

SwiftUI receives GameStore through the environment. GameStore is main-actor isolated and observable. Career and RaceSession carry domain state as value types, while view-local state handles presentation concerns. The model harness compiles without SwiftUI, although geometry types remain a dependency.

This is not presented as formal Clean Architecture, reducer-based state management, or a fully modular engine. Concrete service construction and large model files make a small project direct to navigate, but limit substitution and independent ownership.

## Three stages of racing

### 1. Intended pace and strategy

Career.qualify produces qualifying order and per-lap cumulative timing. Inputs include car performance and condition, setup, driver attributes and preparation, tire degradation, selected difficulty, conditions, and seeded variation.

RivalRaceStrategy retains distinct team profiles and stint decisions. Strategy planning uses cost estimates; higher difficulty evaluates candidate thresholds. RivalDevelopment carries component development across championship weekends. Driver growth combines race experience, training focus, and development milestones.

These are domain rules and heuristic planning, not machine learning or a physically complete car model. Weather is selected for a race weekend rather than continuously simulated.

### 2. Spatial traffic

TrafficRun turns intended durations and pit schedules into a sampled ten-car race. It advances at a fixed 0.25-second simulation step. Its finite-state model covers following, seeking an opening, pulling out, running alongside, merging, abandoning an attempt, and pitting.

Road width and bend lookahead constrain passing opportunities. Nearby cars influence lane choice, longitudinal separation, steering, braking, and merge safety. Pit service participates in traffic movement. Finish-line crossing times feed future classification frames through RaceSession.applyTraffic.

The implementation is an arcade traffic simulation with tuned constraints. It is not a rigid-body physics engine or a formal collision-free proof for arbitrary inputs.

### 3. Playback and presentation

TrafficRun.at interpolates stored samples for display. RaceSession.positions maps playback progress to model time and car positions. The main race uses a Combine timer and monotonic elapsed time; it does not run the full race calculation at display cadence.

TimelineView supplies workshop, guide, and podium animation schedules where used. The requested scheduling intervals are not measured frame-rate guarantees.

## Strategy changes without rerolling

PitWallState keeps baseline frames and the evolving stop/order plan. RaceSession.queue, advanceStrategyLap, and rebuildStints apply commands and recompute future stint effects. Revealed lap frames remain intact. Persisted rival plans and baseline incidents avoid a fresh random draw when the player changes strategy.

This makes an interrupted weekend resumable and prevents strategy controls from becoming a way to repeatedly roll for better results. It does not promise cross-version reconstruction solely from a random seed: generated frames, strategies, and traffic inputs are the persisted replay material.

## Concurrency: calculate, validate, commit

GameStore.qualifyWeekend and editRace take value snapshots and perform CPU work using detached tasks. On completion, the main-actor store compares its current revision with the revision captured when work started. An obsolete result is rejected rather than overwriting newer career state.

Different guards address different failure modes:

| Mechanism | Purpose |
| --- | --- |
| Main-actor observable state | Serializes access to application and presentation state |
| Preparation gate | Prevents overlapping race preparation/edit operations |
| Career revision | Rejects stale background calculations |
| Cloud operation gate | Prevents overlapping synchronization decisions |
| Account-generation token | Ignores obsolete cloud responses after account/state changes |
| Game Center player checks | Avoids applying a response to a different authenticated player |
| Cancellable debounce tasks | Coalesces local-save and achievement synchronization requests |
| NSLock-protected caches | Synchronizes shared geometry and traffic cache dictionaries |

Main-actor isolation alone is not enough: asynchronous functions can suspend and allow intervening changes. Revision and identity checks address that logical reentrancy.

The caches build expensive values outside locks and recheck before insertion. This shortens lock ownership but allows duplicate computation on simultaneous misses. The project uses Swift 5 language mode; it is not claimed to have fully enforced Swift 6 strict-concurrency isolation. There are no custom actor types in the inspected application.

## Rendering and memory tradeoffs

CircuitGeometry rounds authored corners and creates approximately equal-distance route samples. Geometry and width tables are initialized lazily. Traffic runs precompute bend and passing-opening lookups to avoid repeated work in the simulation loop.

CircuitArtwork separates static scenery from the car Canvas. CircuitTextureCache rasterizes scenery through ImageRenderer and uses NSCache with a configured 24 MiB cost limit and 16-image count limit. Recent traffic samples are retained in a three-run cache. HUD observation uses coarser progress updates than car playback.

These techniques reduce repeated preparation and drawing. Tradeoffs include the memory cost of full-race traffic samples, possible first-use rasterization work on the main actor, retained geometry tables, and advisory rather than hard NSCache limits. No sustained-device-FPS or battery claim is made.

## Career storage and cloud reconciliation

Careers are Codable JSON snapshots. Validation checks supported formats, ranges, identities, finite timing values, and race/progression structure. Supported migrations retain compatibility. Disk writes use atomic replacement and iOS file protection. A cloud restore validates the replacement and backs up the existing local file first.

Cloud synchronization uses a private CloudKit database. SHA-256 hashes cover normalized, sorted-key career JSON; snapshot timestamps and device descriptions are not the comparison payload.

| Comparison with the last shared baseline | Action |
| --- | --- |
| Local and remote match | No transfer |
| Only local exists | Upload |
| Only remote exists | Download |
| Both exist without a baseline | Ask the player |
| Only local changed | Upload |
| Only remote changed | Download, subject to active-race protection |
| Both changed | Ask the player |

Before accepting a choice, AppleServices.resolve checks the local hash, fetches the current server record, checks the local hash again after suspension, and compares server record change tags. Uploads use conditional record writes. A server conflict causes another review rather than an unconditional overwrite.

This is whole-career selection, not field-level merging or timestamp-based last-write-wins. SHA-256 is a change fingerprint, not encryption, a signature, or an anti-cheat boundary.

## Apple service boundaries

StoreKit 2 manages non-consumable ownership with verified transactions, entitlement refresh, update handling, and restoration. GameKit manages authentication and monotonic achievement reporting. AuthenticationServices supplies an optional Apple profile stored in Keychain. Cloud synchronization uses the iCloud account independently of that profile.

Public documentation describes capabilities only. Product identifiers, production configuration, purchase logic, and account-management implementation are not included in this repository.

## Reliability limits and next engineering steps

- Resume returns to successfully saved checkpoints; the view's within-lap playback fraction is not persisted.
- Failed local writes report an error but do not roll back the in-memory career.
- Unreadable files are preserved on a best-effort basis; there is no automatic backup-selection interface.
- Optional cloud operations can retry on later activity, but there is no durable background synchronization worker.
- Whole-career storage and bounded cloud snapshot size need continued evaluation as careers grow.
- Service protocols and injectable transports would enable automated fault-injection tests.
- Strict-concurrency auditing and physical-device profiling would strengthen evidence beyond the current model harness.
