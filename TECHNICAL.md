# Tombola - Technical Design

This document explains the engineering and design decisions behind Tombola. It complements the README (what the tool does) and Supabase guide (how it's wired) by recording *why* things are built the way they are.

---

## Architecture

A single static HTML file with vanilla JS. Two themes, two modes, one engine. Persistence via Supabase + RLS for signed-in users, localStorage for anonymous use.

No build step, no framework, no bundler. The whole thing is one `.html` file that can be opened directly in a browser. This was deliberate - Tombola is small enough that adding React or a build pipeline would be more overhead than value, and the static-only approach means it deploys identically to any other static asset on the personal site.

The two modes share the core engine and most of the persistence layer but have separate UI sections and separate themes. Mode switching is a CSS class toggle on `<body>` plus showing/hiding two `<section>` elements.

---

## The picking engine

The whole tool reduces to one function:

```js
function weightedPick(options, weights) {
  const total = weights.reduce((a,b) => a+b, 0);
  if (total === 0) return options[Math.floor(Math.random() * options.length)];
  let r = Math.random() * total;
  for (let i = 0; i < options.length; i++) {
    r -= weights[i];
    if (r <= 0) return options[i];
  }
  return options[options.length - 1];
}
```

Cumulative-weight sampling. Roll a uniform random number scaled to the total weight, then walk through subtracting until it goes negative. This is the standard, correct implementation.

All three general modes (equal, manual, fair-rotation) and the captain logic call `weightedPick` with different weight arrays:

- **Equal**: every weight is 1
- **Manual**: weights from user-entered numbers
- **Fair-rotation**: `1 / (count + 1)` where count is how many times this option has been picked

The inverse-frequency formula is intentionally simple. It guarantees never-picked options get the highest weight (1.0), keeps weights strictly positive (never zero, so any option always has a non-zero chance), and decays at a reasonable rate (after 4 picks, weight is 0.2). More sophisticated approaches (Elo-style updates, beta distributions, time-decay) were considered and rejected as overengineering for the actual problem.

---

## Why three modes, not one configurable picker

A single mode with knobs ("how much should history affect picks?") was tempting because it's more flexible. It was rejected because:

1. The three modes solve genuinely different problems (one-off random, weighted draw, fair rotation over time). Collapsing them into one would force users to understand what they want before they can choose.
2. The scheme cards are also the explanation. Picking "Fair-rotation" is faster than reading a paragraph about history-weighted random selection.
3. Most users only want one of the three. The card UI makes the choice explicit rather than buried in settings.

---

## Captain mode is "fair-rotation with rules"

Captain mode is fundamentally the same algorithm as fair-rotation general mode, with three additions:

1. **GK pairing**: if Captain 1 is a goalkeeper, Captain 2 must also be a goalkeeper. If there's only one GK present, the algorithm repicks Captain 1 from outfield only.
2. **Last-session block**: last week's captains are excluded from the eligible pool by default (toggleable).
3. **Coin toss**: after captains are picked, a separate uniform-random call decides who picks teams first.

Keeping captain as its own mode (rather than a configurable preset of general) is a UX choice. The football audience doesn't want to learn that captain is "fair-rotation with two extra toggles." The chalk-on-pitch theme also makes it visually distinct, which signals "this is its own thing."

---

## Persistence: Supabase + RLS, with localStorage fallback

Anonymous users get localStorage. Signed-in users get Supabase. The data shape is the same in both cases, so the same UI code works against either.

**Why Supabase over a custom backend**: it's already in use for The Assortment, magic-link auth is already configured, and Row Level Security solves the "lock my captain instance to me" requirement without writing any auth/authz code. Adding a new service costs nothing here.

**Why RLS instead of application-level checks**: defense in depth. Even if a bug in the client somehow tried to read another user's data, the database refuses. With application-only checks, a compromised or misbehaving client could see anything in the table.

**Why magic link over OAuth**: low friction, no password to manage, no third-party identity dependency, already working for The Assortment.

---

## Two themes, one engine

The site has two genuinely different aesthetics - restrained 70s board-game for general, chalk-on-pitch for captain. This is implemented as two sets of CSS custom properties scoped to a body class (`.theme-general` and `.theme-captain`). Switching modes swaps the class.

Almost all visual properties resolve through variables: `--bg`, `--ink`, `--accent`, `--rule`, etc. The captain theme also adds two `::before`/`::after` decorations (paper grain, pitch halfway line) on body.

This pattern was chosen over separate stylesheets because the two themes share 90% of their layout and the only thing that really differs is colour, type, and a few borders. Variables let those swap atomically.

---

## UX decisions worth recording

### Step-by-step "soft flow"
Steps reveal sequentially as you complete them, but all stay editable. Locked steps are dimmed at 32% opacity and `pointer-events: none`. The number badges (1, 2, 3 for general; I, II, III for captain) give the page rhythm and make the order obvious.

This was iterated heavily. Earlier versions tried a single configuration panel (overwhelming for first-time users) and a strict step-by-step gate (felt rigid for repeat use). The current "soft" version splits the difference.

### Logging is opt-in for general mode
After a fair-rotation pick, the user gets a "Log this pick" button. Clicking it confirms before adding to history. Earlier versions auto-logged, but this conflated two actions (the *act* of picking and the *commitment* to that pick). Sometimes you pick, look at the result, and decide to pick again rather than commit.

Captain mode logging is also explicit (a "Log session" button after each pick), for the same reason plus the added need for a date input.

### Live weight readout
For fair-rotation with a saved list, a "Current chances" panel shows each option with its pick count and current probability percentage. Updates in real-time as you toggle "exclude last picked."

This was missing from an earlier iteration and felt opaque - users couldn't see *why* fair-rotation was about to make a particular choice likely. Adding it back made the algorithm's behaviour legible.

### The "save this list" call-to-action
Fair-rotation requires a saved list to track history. The first time a user clicks Fair-rotation without a saved list, an inline message appears with a clickable "Save this list →" link. This was simpler than the auto-popup prompt earlier versions tried.

### Coin toss animation
A real 3D-rotating CSS animation, not text. Each face shows the captain's name. The winning side ends facing the camera. This is the one visual flourish that earns its complexity - for a Sunday-morning sports tool, a static "Kabir wins" text would be less satisfying than the moment of suspense as the coin spins.

---

## What's deliberately not built

- **CSV import/export**: would be useful for backup but adds UI complexity. Migrating the existing captain log is a one-time SQL paste, not a recurring need.
- **Edit/delete captain log entries**: intentional. Append-only prevents accidental corruption. Manual SQL edits remain the safety valve.
- **Attendance-aware captain weights**: would require logging who played each session, not just captains. The data doesn't exist for the historical sessions and collecting it retroactively isn't feasible.
- **Sharing lists between users**: would need a join table and policy updates. Not a use case yet.
- **Real-time multi-device sync**: Supabase realtime subscriptions are available but unnecessary for single-user tools.
- **Streak detection / "you've been picked 3 weeks running"**: was in scope at one point, cut as feature-creep that didn't change behaviour, just narrated it.

---

## File structure (as deployed)

```
tools/tombola/
├── index.html       # the whole app
├── README.md        # public-facing
├── TECHNICAL.md     # this file
```

The HTML is monolithic on purpose - inline CSS, inline JS, single file. Splitting into separate files would require either a build step or HTTP overhead. For a tool this size, neither is worth it.

---

## Where the engine could be reused

The picking engine (`weightedPick` + `inverseWeight` + `parseLines`) is roughly 25 lines and depends on nothing. If a future tool needs the same primitives - say, a contest entry picker, a chore rotation, a meal planner - those lines lift directly.

The Supabase schema generalises to anything that's "lists with options and a history dict." The `tombola_lists` table is essentially a key-value store with extra metadata; new tools can either share it (with a `tool_id` column to namespace) or copy the pattern.

---

## Lineage

This started as a Python CLI for picking football captains on Sunday mornings. The CLI worked but was ugly and tied to one machine. The web version generalised the engine (captains became a special case of fair-rotation), added a second mode for friends with non-football use cases, and replaced the CSV with proper persistence. The CLI still works if you want it locally; the web version is for everyone else.