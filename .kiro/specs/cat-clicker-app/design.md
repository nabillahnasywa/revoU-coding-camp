# Design Document: Cat Clicker App

## Overview

The Cat Clicker App is a single-page web application built with HTML5, Tailwind CSS, and Vanilla JavaScript. The core loop is simple: the user pets an animated cat, which responds with delightful visual reactions, particle effects, and sound. Engagement is sustained through a combo system, mood system, milestone unlocks, and a persistent pet counter.

The entire app lives in a single `index.html` file with inline or linked CSS/JS. No build tools, no frameworks — just a fast, self-contained experience that loads instantly.

### Key Design Principles

- **Immediate feedback**: Every pet triggers a visible reaction within 100ms
- **Variety**: Weighted random selection prevents repetition
- **Progression**: Milestones and unlocks give users reasons to keep going
- **Accessibility**: Keyboard, touch, and reduced-motion support throughout
- **Performance**: 60fps particle animations using CSS or Canvas

---

## Architecture

The app follows a simple event-driven architecture with a central state object. All game logic reads from and writes to this state, and a lightweight render loop syncs the DOM to state changes.

```
┌─────────────────────────────────────────────────────┐
│                     index.html                      │
│                                                     │
│  ┌──────────┐   ┌──────────┐   ┌─────────────────┐ │
│  │  Input   │   │  State   │   │   Render / DOM  │ │
│  │ Handler  │──▶│  Store   │──▶│    Updater      │ │
│  └──────────┘   └──────────┘   └─────────────────┘ │
│       │              │                              │
│       │         ┌────┴──────────────────────┐       │
│       │         │       Subsystems          │       │
│       │         │  - ReactionEngine         │       │
│       │         │  - ParticleSystem         │       │
│       │         │  - MoodEngine             │       │
│       │         │  - ComboTracker           │       │
│       │         │  - MilestoneManager       │       │
│       │         │  - AudioManager           │       │
│       │         │  - PersistenceManager     │       │
│       │         └───────────────────────────┘       │
└─────────────────────────────────────────────────────┘
```

### Data Flow

1. User input (click / tap / keyboard) → `InputHandler.onPet()`
2. `InputHandler` calls `StateStore.registerPet()`
3. `StateStore` updates counter, combo, mood, checks milestones
4. `StateStore` emits events consumed by subsystems
5. Subsystems update the DOM / canvas / audio
6. `PersistenceManager` writes changed state to localStorage

---

## Components and Interfaces

### StateStore

Central state container. All subsystems read from it; only `StateStore` methods mutate it.

```js
StateStore = {
  // Counters
  sessionPets: Number,
  allTimePets: Number,

  // Mood
  mood: 'calm' | 'happy' | 'excited' | 'overwhelmed',
  recentPetTimestamps: Number[],   // rolling window for rate calc

  // Combo
  comboActive: Boolean,
  comboCount: Number,
  lastPetTimestamp: Number,

  // Reactions
  reactionPool: Reaction[],
  reactionHistory: String[],       // last 10 reaction IDs for frequency check

  // Milestones
  unlockedMilestones: Set<Number>,

  // Sound
  soundEnabled: Boolean,

  // Methods
  registerPet(),
  getMood(),
  getWeightedReaction(),
  checkMilestone(),
  persist(),
  restore(),
}
```

### InputHandler

Attaches event listeners to the cat element. Normalises mouse, touch, and keyboard events into a single `onPet()` call.

```js
InputHandler = {
  attach(catElement, onPet),
  // Listens for: click, touchstart, keydown (Space/Enter when focused)
}
```

### CatAnimator

Controls CSS class toggling on the cat SVG/element to drive animations.

```js
CatAnimator = {
  playIdle(),
  playPetAnimation(index),      // 0-4, picks one of 5 pet animations
  playCelebration(),            // milestone animation
  playExcited(),                // combo state
  playOverwhelmed(),
  playEntrance(),
}
```

### ReactionEngine

Selects and displays reactions. Enforces the 40% frequency cap.

```js
ReactionEngine = {
  trigger(reaction, position),
  selectReaction(mood, reactionPool, history): Reaction,
  // Weighted random with history-based dampening
}
```

### ParticleSystem

Renders floating particles. Supports both CSS animation and Canvas fallback.

```js
ParticleSystem = {
  emit(x, y, type, count, rate),
  setRate(multiplier),   // 1x normal, 3x combo, 0x overwhelmed
  clear(),
}
```

### MoodEngine

Calculates mood from recent petting rate.

```js
MoodEngine = {
  calculate(recentPetTimestamps): Mood,
  // Calm:       < 1 pet/10s
  // Happy:      1–3 pets/10s
  // Excited:    4–9 pets/10s
  // Overwhelmed: ≥ 10 pets/10s
}
```

### ComboTracker

Tracks rapid petting sequences.

```js
ComboTracker = {
  onPet(timestamp): { comboActive, comboCount, justEnded, justStarted },
  WINDOW_MS: 2000,
  THRESHOLD: 5,
}
```

### MilestoneManager

Checks thresholds and fires milestone events.

```js
MilestoneManager = {
  THRESHOLDS: [10, 50, 100, 500, 1000],
  check(allTimePets, unlockedMilestones): Milestone | null,
  getUnlockReaction(milestone): Reaction,
}
```

### AudioManager

Wraps the Web Audio API / `<audio>` elements. Respects the sound toggle.

```js
AudioManager = {
  play(clipId),
  setEnabled(bool),
  isEnabled(): Boolean,
}
```

### PersistenceManager

Reads/writes to localStorage.

```js
PersistenceManager = {
  save(key, value),
  load(key, defaultValue),
  KEYS: { ALL_TIME_PETS, SOUND_ENABLED }
}
```

---

## Data Models

### Reaction

```js
{
  id: String,           // unique identifier e.g. "purr"
  label: String,        // display text e.g. "Purrrr~"
  particleType: String, // "hearts" | "stars" | "sparkles" | "fish" | "notes"
  audioClip: String,    // clip ID, or null
  weight: Number,       // base selection weight (1–10)
  moods: String[],      // which moods this reaction belongs to
  unlockMilestone: Number | null,  // null = available from start
}
```

### Milestone

```js
{
  threshold: Number,    // pet count that triggers this milestone
  label: String,        // e.g. "10 Pets!"
  unlockReactionId: String,
  celebrationDuration: Number,  // ms, between 2000 and 4000
}
```

### Mood

```js
type Mood = 'calm' | 'happy' | 'excited' | 'overwhelmed'
```

### AppState (persisted subset)

```js
{
  allTimePets: Number,
  soundEnabled: Boolean,
}
```

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system — essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Pet counter always increments by exactly 1

*For any* app state, when a pet is registered, the session pet counter SHALL increase by exactly 1 and no more.

**Validates: Requirements 1.1**

### Property 2: Reaction frequency cap

*For any* sequence of 10 consecutive pet reactions drawn from the reaction pool, no single reaction ID SHALL appear more than 4 times (40%).

**Validates: Requirements 3.2**

### Property 3: Mood is a pure function of recent petting rate

*For any* array of pet timestamps, `MoodEngine.calculate()` SHALL return the same Mood value every time it is called with the same input — and the returned Mood SHALL match the rate thresholds defined in the spec (Calm < 1/10s, Happy 1–3/10s, Excited 4–9/10s, Overwhelmed ≥ 10/10s).

**Validates: Requirements 7.1, 7.2**

### Property 4: Combo activates at threshold

*For any* sequence of pet timestamps, the combo state SHALL be active if and only if 5 or more pets were registered within any 2-second window ending at the most recent pet.

**Validates: Requirements 6.1**

### Property 5: Persistence round-trip

*For any* all-time pet count and sound preference, saving to localStorage and then restoring SHALL produce values equal to the originals.

**Validates: Requirements 4.3, 4.4, 8.3**

### Property 6: Milestone triggers exactly once per threshold

*For any* sequence of pet registrations that crosses a milestone threshold, the milestone event SHALL fire exactly once — not zero times and not more than once — regardless of how many pets are registered in a single batch.

**Validates: Requirements 5.1, 5.2**

### Property 7: Reaction pool grows monotonically after milestones

*For any* sequence of milestone events, the size of the reaction pool SHALL be greater than or equal to its size before the milestone was reached.

**Validates: Requirements 5.4**

### Property 8: Overwhelmed mood suppresses particles

*For any* app state where the mood is Overwhelmed, the particle emission rate SHALL be 0 for the 3-second suppression window.

**Validates: Requirements 7.6**

### Property 9: Reactions always come from the correct mood subset

*For any* mood state (Calm, Happy, Excited, or Overwhelmed), every reaction selected by `ReactionEngine.selectReaction()` SHALL have that mood listed in its `moods` array.

**Validates: Requirements 7.3, 7.4, 7.5**

### Property 10: Muted state suppresses all audio

*For any* reaction triggered while `soundEnabled` is false, `AudioManager.play()` SHALL not produce any audio output — regardless of which reaction or clip is involved.

**Validates: Requirements 8.4**

---

## Error Handling

### localStorage Unavailable

Some browsers block localStorage in private/incognito mode. `PersistenceManager` wraps all reads/writes in try/catch. If unavailable, the app runs in-memory only — the counter resets on refresh but the session still works fully.

### Audio Playback Blocked

Browsers require a user gesture before playing audio. `AudioManager` defers all playback until after the first pet interaction. If `play()` throws (e.g., missing clip), it logs a warning and continues silently.

### Animation Performance

If `requestAnimationFrame` is unavailable (very old browsers), the `ParticleSystem` falls back to CSS-only animations. The `prefers-reduced-motion` media query disables non-essential particle effects and transitions.

### Invalid State Recovery

On app load, `PersistenceManager.load()` validates that restored values are of the expected type and within sane ranges (e.g., `allTimePets >= 0`). Corrupt or unexpected values are discarded and replaced with defaults.

---

## Testing Strategy

### Unit Tests (example-based)

Focus on specific behaviors and edge cases:

- `ReactionEngine.selectReaction()` returns a reaction from the correct mood subset
- `MoodEngine.calculate()` returns correct mood for boundary rate values (e.g., exactly 1/10s, exactly 10/10s)
- `MilestoneManager.check()` returns null when no new threshold is crossed
- `MilestoneManager.check()` returns the correct milestone when a threshold is first crossed
- `PersistenceManager` returns the default value when localStorage is empty
- `ComboTracker.onPet()` does not activate combo for 4 pets in 2 seconds
- `ComboTracker.onPet()` activates combo for exactly 5 pets in 2 seconds

### Property-Based Tests

The feature involves pure logic functions (mood calculation, combo detection, reaction selection, persistence round-trips, milestone firing) that are well-suited to property-based testing. The recommended library is **fast-check** (JavaScript).

Each property test runs a minimum of **100 iterations**.

Tag format: `// Feature: cat-clicker-app, Property N: <property_text>`

| Property | Test Description |
|---|---|
| P1: Pet counter increments by 1 | Generate arbitrary state, call `registerPet()`, assert counter increased by exactly 1 |
| P2: Reaction frequency cap | Generate sequences of 10 reactions via `selectReaction()`, assert no ID appears > 4 times |
| P3: Mood is pure function of rate | Generate arbitrary timestamp arrays, assert `calculate()` is deterministic and matches thresholds |
| P4: Combo activates at threshold | Generate timestamp sequences, assert combo state matches 5-in-2s rule |
| P5: Persistence round-trip | Generate arbitrary `{allTimePets, soundEnabled}`, save then restore, assert equality |
| P6: Milestone fires exactly once | Generate pet sequences crossing thresholds, assert milestone fires exactly once |
| P7: Reaction pool grows after milestone | Generate milestone sequences, assert pool size is non-decreasing |
| P8: Overwhelmed suppresses particles | Generate overwhelmed state, assert emission rate is 0 during suppression window |
| P9: Reactions from correct mood subset | Generate any mood + reaction selection, assert returned reaction has that mood in its moods array |
| P10: Muted state suppresses audio | Generate any reaction while soundEnabled=false, assert AudioManager.play() produces no output |

### Integration / Smoke Tests

- App loads without JS errors in Chrome, Firefox, Safari
- Cat element is visible and centered on 320px and 1440px viewports
- Sound toggle persists across page reload
- All-time counter persists across page reload
- Milestone overlay appears and disappears within expected duration
- Keyboard (Space/Enter) triggers a pet reaction

### Accessibility Checks

- All interactive elements have ARIA labels (manual + axe-core scan)
- Color contrast verified with browser DevTools
- Reduced-motion mode tested by toggling `prefers-reduced-motion` in DevTools
