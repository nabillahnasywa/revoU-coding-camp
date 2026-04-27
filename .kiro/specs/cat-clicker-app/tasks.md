# Implementation Plan: Cat Clicker App

## Overview

Build a self-contained `index.html` single-page app using HTML5, Tailwind CSS, and Vanilla JavaScript. Implementation proceeds in layers: scaffold → state/persistence → core interaction → reactions/particles → mood/combo → milestones → audio → accessibility/polish.

## Tasks

- [x] 1. Scaffold the HTML shell and load dependencies
  - Create `index.html` with Tailwind CSS CDN link and a `<script>` block for all JS
  - Add semantic landmark elements: main container, cat stage, counter display, mood indicator, sound toggle
  - Apply base Tailwind classes for layout (flex, centered, full-height) and pastel background gradient
  - Add cat placeholder element with `tabindex="0"` and ARIA label for keyboard focus
  - _Requirements: 9.1, 9.4, 10.1, 10.2, 10.4_

- [ ] 2. Implement PersistenceManager and StateStore
  - [-] 2.1 Implement `PersistenceManager` with `save()`, `load()`, and try/catch for unavailable localStorage
    - Validate restored values (type check, `allTimePets >= 0`) and fall back to defaults on corruption
    - _Requirements: 4.3, 4.4, 8.3, 8.5_

  - [ ]* 2.2 Write property test for persistence round-trip
    - **Property 5: Persistence round-trip**
    - **Validates: Requirements 4.3, 4.4, 8.3**

  - [~] 2.3 Implement `StateStore` with all fields and `restore()` / `persist()` methods
    - Wire `PersistenceManager` into `restore()` on init and `persist()` after every mutation
    - _Requirements: 4.3, 4.4, 8.3, 8.5_

  - [ ]* 2.4 Write property test for pet counter increment
    - **Property 1: Pet counter always increments by exactly 1**
    - **Validates: Requirements 1.1**

- [ ] 3. Implement InputHandler and core pet registration
  - [~] 3.1 Implement `InputHandler.attach()` listening for `click`, `touchstart`, and `keydown` (Space/Enter)
    - Ensure events outside the cat element do not call `onPet()`
    - _Requirements: 1.1, 1.5, 9.2, 9.3_

  - [~] 3.2 Implement `StateStore.registerPet()` to increment `sessionPets`, `allTimePets`, update `lastPetTimestamp`, and push to `recentPetTimestamps`
    - Call `persist()` after each registration
    - _Requirements: 1.1, 4.3_

  - [~] 3.3 Render the pet counter and animate it on change (scale/bounce CSS transition)
    - Display both session and all-time totals
    - _Requirements: 4.1, 4.2, 4.5_

- [ ] 4. Implement CatAnimator and entrance animation
  - [~] 4.1 Create the cat SVG/CSS character with idle, pet (×5), celebration, excited, and overwhelmed animation classes
    - Use CSS `@keyframes` for each animation; respect `prefers-reduced-motion` by wrapping non-essential animations in a media query
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 9.6, 10.5_

  - [~] 4.2 Implement `CatAnimator` methods: `playIdle()`, `playPetAnimation(index)`, `playCelebration()`, `playExcited()`, `playOverwhelmed()`, `playEntrance()`
    - `playEntrance()` runs once on load (0.5–1.5 s), then transitions to `playIdle()`
    - Return-to-idle timer (2 s after last pet) managed here
    - _Requirements: 1.3, 1.4, 2.2, 2.3, 2.4, 10.5_

- [ ] 5. Implement MoodEngine and mood display
  - [~] 5.1 Implement `MoodEngine.calculate(recentPetTimestamps)` as a pure function using the four rate thresholds
    - Calm < 1/10 s · Happy 1–3/10 s · Excited 4–9/10 s · Overwhelmed ≥ 10/10 s
    - _Requirements: 7.1, 7.2_

  - [ ]* 5.2 Write property test for mood as pure function of rate
    - **Property 3: Mood is a pure function of recent petting rate**
    - **Validates: Requirements 7.1, 7.2**

  - [~] 5.3 Integrate `MoodEngine` into `StateStore.registerPet()` and update the mood indicator element (emoji/label)
    - Trigger `CatAnimator.playOverwhelmed()` and set particle suppression flag when mood is Overwhelmed
    - _Requirements: 7.2, 7.6, 7.7_

  - [ ]* 5.4 Write unit tests for MoodEngine boundary values
    - Test exactly 1/10 s, exactly 3/10 s, exactly 9/10 s, and exactly 10/10 s edge cases
    - _Requirements: 7.1, 7.2_

- [ ] 6. Implement ReactionEngine and reaction display
  - [~] 6.1 Define the initial `reactionPool` array with at least 10 `Reaction` objects covering all four moods
    - Assign weights, particle types, and mood arrays per the data model
    - _Requirements: 3.1, 7.3, 7.4, 7.5_

  - [~] 6.2 Implement `ReactionEngine.selectReaction(mood, reactionPool, history)` with weighted random and 40% frequency cap
    - Filter pool to current mood subset before selection; dampen reactions appearing in last 10 history entries
    - _Requirements: 3.2, 7.3, 7.4, 7.5_

  - [ ]* 6.3 Write property test for reaction frequency cap
    - **Property 2: Reaction frequency cap**
    - **Validates: Requirements 3.2**

  - [ ]* 6.4 Write property test for reactions from correct mood subset
    - **Property 9: Reactions always come from the correct mood subset**
    - **Validates: Requirements 7.3, 7.4, 7.5**

  - [~] 6.5 Implement `ReactionEngine.trigger()` to show the text label near the cat (fade out after 1 s)
    - Position label relative to click/tap coordinates
    - _Requirements: 3.4_

- [ ] 7. Implement ParticleSystem
  - [~] 7.1 Implement `ParticleSystem.emit(x, y, type, count, rate)` using CSS animations (Canvas fallback if `requestAnimationFrame` unavailable)
    - Spawn particle DOM elements with randomised trajectory keyframes; remove after animation ends
    - _Requirements: 3.3, 10.3_

  - [~] 7.2 Implement `ParticleSystem.setRate(multiplier)` and `clear()`
    - Rate 0 suppresses all emission; rate 3× used during combo
    - _Requirements: 6.2, 7.6_

  - [ ]* 7.3 Write property test for overwhelmed mood suppressing particles
    - **Property 8: Overwhelmed mood suppresses particles**
    - **Validates: Requirements 7.6**

- [ ] 8. Implement ComboTracker
  - [~] 8.1 Implement `ComboTracker.onPet(timestamp)` returning `{ comboActive, comboCount, justEnded, justStarted }`
    - Combo activates when ≥ 5 pets in any 2-second window; ends when no pet for 2 s
    - _Requirements: 6.1_

  - [ ]* 8.2 Write property test for combo activation at threshold
    - **Property 4: Combo activates at threshold**
    - **Validates: Requirements 6.1**

  - [ ]* 8.3 Write unit tests for ComboTracker edge cases
    - Test 4 pets in 2 s (no combo), exactly 5 pets in 2 s (combo), and combo-end summary label
    - _Requirements: 6.1, 6.4_

  - [~] 8.4 Integrate ComboTracker into `StateStore.registerPet()` and wire visual effects
    - On combo start: call `CatAnimator.playExcited()` and `ParticleSystem.setRate(3)`
    - On combo end: display combo summary label; reset rate to 1×
    - On combo ≥ 20: trigger rare reaction
    - _Requirements: 6.2, 6.3, 6.4, 6.5_

- [~] 9. Checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 10. Implement MilestoneManager
  - [~] 10.1 Implement `MilestoneManager.check(allTimePets, unlockedMilestones)` for thresholds [10, 50, 100, 500, 1000]
    - Return the crossed `Milestone` object or `null`; use `unlockedMilestones` Set to prevent re-firing
    - _Requirements: 5.1, 5.2_

  - [ ]* 10.2 Write property test for milestone fires exactly once
    - **Property 6: Milestone triggers exactly once per threshold**
    - **Validates: Requirements 5.1, 5.2**

  - [~] 10.3 Add milestone-unlocked reactions to the pool and implement `MilestoneManager.getUnlockReaction()`
    - Each milestone adds ≥ 1 new reaction; pool size must be non-decreasing
    - _Requirements: 5.4_

  - [ ]* 10.4 Write property test for reaction pool grows monotonically after milestones
    - **Property 7: Reaction pool grows monotonically after milestones**
    - **Validates: Requirements 5.4_**

  - [~] 10.5 Implement milestone overlay UI: full-screen overlay with celebration message, unlock announcement, and auto-dismiss after 2–4 s
    - Trigger `CatAnimator.playCelebration()` during overlay
    - _Requirements: 5.2, 5.3, 5.5_

- [ ] 11. Implement AudioManager and sound toggle
  - [~] 11.1 Implement `AudioManager` wrapping Web Audio API / `<audio>` elements; defer first play until after user gesture
    - `play()` is a no-op when `soundEnabled` is false; log warning on missing clip
    - _Requirements: 3.5, 8.4_

  - [ ]* 11.2 Write property test for muted state suppresses all audio
    - **Property 10: Muted state suppresses all audio**
    - **Validates: Requirements 8.4**

  - [~] 11.3 Implement sound toggle button: toggle `soundEnabled` in StateStore, persist preference, update button icon/label
    - Default to muted on first load (no saved preference)
    - _Requirements: 8.1, 8.2, 8.3, 8.5_

- [ ] 12. Wire all subsystems together in the main init function
  - [~] 12.1 Write `init()` that calls `StateStore.restore()`, attaches `InputHandler`, starts `CatAnimator.playEntrance()`, and renders initial UI state
    - Ensure `registerPet()` calls `MoodEngine`, `ComboTracker`, `MilestoneManager`, `ReactionEngine`, `ParticleSystem`, `AudioManager`, and `CatAnimator` in the correct order
    - _Requirements: 1.2, 2.2, 4.4, 8.5_

  - [ ]* 12.2 Write integration smoke tests
    - Verify app loads without JS errors, cat is visible, counter renders, sound toggle works, and keyboard Space/Enter triggers a reaction
    - _Requirements: 1.1, 1.2, 4.1, 8.1, 9.3_

- [ ] 13. Accessibility and responsive polish
  - [~] 13.1 Add ARIA labels to all interactive controls (cat, sound toggle, any buttons)
    - Verify focus ring is visible on keyboard navigation
    - _Requirements: 9.4_

  - [~] 13.2 Audit color contrast for all text elements (≥ 4.5:1) and fix any failures
    - _Requirements: 9.5_

  - [~] 13.3 Test and fix layout at 320 px and 2560 px viewport widths; ensure no horizontal scroll
    - Verify cat scales responsively using Tailwind responsive classes
    - _Requirements: 2.5, 9.1_

- [~] 14. Final checkpoint — Ensure all tests pass
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Property tests use **fast-check** (min 100 iterations each); tag format: `// Feature: cat-clicker-app, Property N: <property_text>`
- Unit tests complement property tests — both are needed for full coverage
- The entire app lives in a single `index.html`; no build tools required
