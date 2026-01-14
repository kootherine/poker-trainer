# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Overview

This repository is a single-page, static web app for training Texas Hold'em pre-flop starting-hand decisions. Everything (markup, styles, and logic) lives in `index.html`; there is no build system, package manager, or test framework configured.

## How to run and develop

Because this is a pure static site, you develop by editing `index.html` and reloading it in a browser.

- **Open directly in a browser (quickest path)**
  - On macOS: `open index.html`
  - On other systems: open `index.html` from your file explorer or drag it into a browser.
- **Serve via a simple local HTTP server (recommended when editing frequently)**
  - Using Python 3: `python3 -m http.server 8000`
  - Then visit `http://localhost:8000/index.html` in your browser.

There are **no project-defined commands** for build, lint, or tests:

- No `package.json` or similar manifest is present.
- No test runner or linter configuration exists.
- Running a single test is not applicable until a test framework is added.

If you introduce a toolchain later (e.g., npm + a bundler or test framework), add the key commands here so future agents can use them.

## High-level architecture

### File layout

- `index.html` — the entire application:
  - `<style>` in the `<head>` defines layout and visual design.
  - `<body>` contains the overlays, main trainer UI, and all interactive elements.
  - A single `<script>` tag at the bottom contains all game logic in plain browser JavaScript.

There are no external JS/CSS dependencies or modules; all functions and state are in the global script scope.

### Core UI structure

Key UI blocks inside `index.html`:

- **Welcome overlay (`#welcome`)**: Intro modal shown only on the first visit, controlled via `localStorage` key `poker-trainer-visited`.
- **Position info overlay (`#position-info`)**: Click/tap on the displayed position name to show a description of that position, using the `positionInfo` map.
- **Main container (`.container`)**:
  - Scoreboard (`#correct`, `#total`, `#accuracy`, `#sizing-accuracy`).
  - Situation description (`#action-description`) summarizing preflop action.
  - Position display (`#position`).
  - Game info row (`#pot-size`, `#to-call`).
  - Two card elements (`#card1`, `#card2`) showing rank/suit, colored red/black based on suit.
  - Hand name (`#hand-name`).
  - Primary action buttons (`#btn-raise`, `#btn-call`, `#btn-fold`).
  - Raise sizing panel (`#sizing-panel`, `#sizing-buttons`, `#btn-back`) for choosing a specific raise size after selecting Raise.
  - Feedback area (`#feedback`, with title/subtitle/explanation elements) shown after each decision.
  - Next-hand button (`#next-btn`) to deal a new hand.
  - Streak indicator (`#streak`) for consecutive fully-correct decisions.

### Game state and data model

The script defines core constants and state:

- **Static data**
  - `ranks`, `suits`: card ranks and suits for random dealing.
  - `positions`: player positions (UTG, Middle, Cutoff, Button, Small Blind, Big Blind).
  - `scenarios`: a fixed list of preflop action patterns (e.g., folded to you, limpers, raises/3-bets) with metadata like `raisesAhead`, `callersAhead`, `pot`, `toCall`, and `lastRaise`.
  - `handRankings`: a coarse strength map (`1` = premium through `5` = weakest) keyed by hand type strings such as `AA`, `AKs`, `KQo`, etc.
- **Mutable state**
  - Global counters: `correct`, `total`, `sizingCorrect`, `sizingTotal`, `streak`.
  - Current situation: `currentHand`, `currentPosition`, `currentScenario`.
  - Correct answers for the current hand: `correctAnswer` (`'raise' | 'call' | 'fold'`) and implied sizing band via `getCorrectSizing`.

`currentHand` is an object `{ rank1, rank2, suit1, suit2, handKey, suited }`, where `handKey` is something like `AKs` or `TT` produced by `getHandKey`.

### Core logic functions

High-level responsibilities of the main functions:

- **Hand representation and naming**
  - `getHandKey(rank1, rank2, suited)`: normalizes two ranks into an ordered key (e.g. `AKs`, `QJo`, `99`) used to look up strength in `handRankings`.
  - `getHandName(rank1, rank2, suited)`: returns a user-facing name for the hand (e.g. `Pocket Queens`, `Ace-King Suited`).

- **Classification helpers**
  - `isLate(position)`: true for `Button` or `Cutoff`.
  - `isBlind(position)`: true for `Small Blind` or `Big Blind`.

- **Decision policy**
  - `getCorrectAction(handKey, position, scenario)`: returns the recommended high-level action (`'raise'`, `'call'`, or `'fold'`) based on hand strength, position, and scenario:
    - Strongest hands (ranking 1–3) are generally raises, with adjustments for facing raises/3-bets.
    - Limper scenarios treat Big Blind specially (never folding when checking is free).
    - Mid-strength hands in late position or blinds may become raises or calls that would be folds elsewhere.
  - `getCorrectSizing(scenario)`: returns `{ amount, min, max }` for correct raise sizing given the scenario (open-raise, raise vs limpers, 3-bet, 4-bet). Other code uses this to judge if a chosen raise size is acceptable.
  - `getRaiseSizingOptions(scenario)`: generates four button options for the sizing panel (amount + label like `Standard (3x)`), tuned to whether the player is open-raising, isolating limpers, 3-betting, or 4-betting.

- **Interaction utilities**
  - `addTapHandler(el, cb)`: wraps `touchstart`/`touchmove`/`touchend` and `click` to create a mobile-friendly tap handler that avoids accidental taps while allowing desktop clicks.

- **UI mode switching**
  - `showRaiseSizing()`: hides the main action buttons, populates `#sizing-buttons` with scenario-specific options from `getRaiseSizingOptions`, wires them to call `makeDecision('raise', amount)`, and shows the sizing panel.
  - `hideSizingPanel()`: hides the sizing panel and restores the main action buttons.

- **Feedback and scoring**
  - `makeDecision(action, size)`: core function called when the user chooses Fold/Call or a Raise size. It:
    - Increments attempt counters and, for raises, sizing counters.
    - Compares the chosen action to `correctAnswer` and, if applicable, checks the raise size against `getCorrectSizing`.
    - Updates scoreboard values and accuracy percentages.
    - Chooses and renders feedback styles (`correct`, `partial`, or `incorrect`) and text via `getExplanation` and `getSizingExplanation`.
    - Manages the streak count and shows/hides the appropriate UI sections (feedback vs action buttons vs Next Hand).
  - `getExplanation(...)`: returns a human-readable explanation of *why* the correct action is preferred, using hand class (`premium`, `strong`, etc.), scenario, and position.
  - `getSizingExplanation(scenario, chosen, correct)`: explains recommended raise size and whether the user's size was too small/large.

- **Hand dealing and scenario selection**
  - `dealHand()`: sets up each new quiz round by:
    - Dealing two distinct random cards (`r1/s1`, `r2/s2`) and constructing `currentHand` and its `handKey`.
    - Randomly picking `currentPosition` and `currentScenario`, with corrections:
      - UTG always uses the "folded_to_you" scenario (no prior action).
      - Big Blind never gets "folded_to_you"; at least one limper is enforced.
    - Computing `correctAnswer` from `getCorrectAction`.
    - Updating the DOM: card faces, position label, hand name, situation text, pot, and effective `to-call` amount.
      - `to-call` is adjusted for blinds already posted so SB/BB show only the remaining amount to call in non-free-check cases.
      - If the Big Blind has a free check (no raises), the Call button label becomes `Check` and the Fold button is hidden.
    - Resetting feedback and panel visibility for the new hand.

- **Position info overlay**
  - `positionInfo`: map from position name to `{ title, text }` shown in the info overlay.
  - `showPositionInfo()` / `hidePositionInfo()`: control the overlay using `positionInfo[currentPosition]`.

- **Welcome overlay and initialization**
  - `showWelcome()` / `hideWelcome()`: manage the first-time welcome modal via `localStorage`.
  - At the bottom of the script, event handlers are attached via `addTapHandler` for all interactive elements, `showWelcome()` is called, and then `dealHand()` is invoked to initialize the first hand.

### Extensibility notes

- All logic is in a single script block; when adding new features, prefer extracting coherent helpers (e.g., additional scenario generators or hand-ranking rules) into small functions within the same script, or move logic into a separate JS file referenced from `index.html`.
- If you introduce additional game modes (e.g., postflop decisions or different blind structures), you will likely add new scenario definitions and extend `getCorrectAction`, `getCorrectSizing`, and explanation functions to handle those paths consistently.
