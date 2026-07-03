# CLAUDE.md — Engagement Unlocked: Listen and Lead

Read this file fully before making any change. It is the design bible and decision
history for this project. When a request conflicts with something here, flag the
conflict rather than silently overriding it.

## What this is

A single-file, browser-based serious game (`index.html`, no dependencies, no build
step) about leading an Employee Listening Center of Excellence at a fictional food
manufacturer (Meridian Foods — Bell Creek, Riverside, and Prairie Junction plants,
1,400 employees). The player runs five annual survey cycles (plus an optional Year 0
tutorial), making quarterly decisions, investing in a capability tree, and managing
four resources. Deployed via GitHub Pages.

**This game is also an engine.** The architecture is deliberately content-agnostic:
swapping the content layer (situations, characters, tree, metric names) should yield
a new professional-decision-making game without touching the loop. Never couple
content to architecture — e.g., no difficulty-specific trees, no situation logic
hardcoded into the renderer.

## Architecture (locked — do not restructure)

- **`S`** — state object: all game state, mutations, helpers (`canBuild`,
  `checkReq`, `resolveOpt`, `applyQueued`, `checkStagnation`, `drawYearSituations`,
  `getPreSurvey`, `findSitById`, arc + rec methods)
- **`R`** — renderer: pure render functions, reads S, writes DOM via
  `this.app().innerHTML`
- **`C`** — controller: event handlers and flow control
- **`AUDIO`** — Web Audio API sounds (node click, grade sting, zone shift), gated
  behind `S.soundOn` toggle

**Phase flow:** `diff` → `ob` → `game` (quarter loop) → `presurvey` → `reveal`
(Year 0 only) → `postsurvey` → next year … → `closing` (after Year 5 postsurvey).

**Quarter flow:** decision → build (tree) → resolve. Q4 resolve leads to presurvey.

## Core resources

| Resource | Nature | Notes |
|---|---|---|
| Organizational Trust | 0–100 standing, cannot be spent | Zones: Skeptical 0–19, Cautious 20–39, Building 40–59, Engaged 60–79, Champion 80–100. Floor of 5 (no death spiral). Building was added between Cautious and Engaged so a strong-grade year has somewhere to land besides a stale-feeling Cautious label — see balance notes below. |
| Political Capital | Spendable currency | Earned passively per quarter by zone rate + grade bonus at year end. Spent on gated situation options and recommendations. |
| Budget | Annual money | See budget model below. |
| Capacity | Slots per quarter | 1 per decision option that costs a slot, 1 per tree node built. Easy 4 / Normal 3 / Hard 2. |

## Balance targets (verified by simulation — re-simulate before changing)

- **Trust trajectory:** Easy reaches Champion with A/S play. **Normal should land
  shy of Engaged or just at Engaged with good (B/A) play; Champion should be rare
  and require sustained A/S.** Hard fights to reach Cautious or lands at Building
  with sustained good play; Engaged is the stretch-goal ending (rare, not zero).
- **Trust sources by weight:** survey grade shift (primary: S+10 A+7 B+4 C+1 D−2
  F−5), gated decision options (+3 to +6 each, requires nodes/PC/zone), node
  effects (SE track mostly), ambient events (texture only; pool weights
  champion .65 / engaged .55 / building .50 / cautious .45 / skeptical .20;
  most deltas ±1).
- **`TRUST_ANNUAL_CAP` (`easy:null, normal:10, hard:13`) throttles Normal/Hard.**
  Simulation showed the three positive sources above (decisions ~8–12/yr,
  grade-shift ~7–8/yr, node effects ~3/yr) each independently approached the old
  "~8/yr decision cap" note below, so capping decisions alone wasn't enough —
  Normal/great hit Champion 100% of the time even with grade-shift and node
  trust cut hard. The fix instead shares ONE annual budget for positive trust
  across every source (decisions, ambient, node effects, grade-shift, rec
  landings); losses are never capped. Scaling happens in `applyQueued` /
  `applyCappedTrust` *before* anything is displayed, and `applyQueued` sums all
  of a quarter's trust into a single `adjTrust` call (not one per entry) so the
  floor/ceiling clamp can only ever fire once — otherwise a bad quarter's later
  entries could get silently absorbed by an earlier floor-out while the
  displayed chips still summed to something larger. `checkStagnation` pushes its
  penalty into `qCons` and runs *before* `applyQueued` for the same reason —
  it must not apply its own separate `adjTrust` call. Do not reintroduce a
  decisions-only cap; re-simulate before changing `TRUST_ANNUAL_CAP` values.
  Hard's cap moved 11→13 when the Building zone was added — Engaged's floor
  moved from 50 to 60, and 11 could no longer reach it even as a rare
  stretch under sustained great play; re-check this any time zone
  boundaries change again, not just when the cap itself changes.
- **Budget model:** flat base per year (Easy $50K / Normal $40K / Hard $25K) +
  10% carry-over of unspent (rest "went to overhead") + flat grade delta
  (S+$8K A+$5K B+$2K C 0 D−$3K F−$6K) + Skeptical penalty −$2K. Floor $5K.
  No multipliers, no compounding surpluses.
- **Node costs are fixed and do NOT scale with budget/difficulty.** Fixed prices
  against different purchasing power IS the difficulty lever. Total tree ≈ $413K
  (doubled from an original $207K — simulation showed $207K vs. Normal's ~$200K
  5-yr budget let good play buy nearly the entire tree, ~15 nodes, versus the
  10–12 target). Normal 5-yr budget ≈ $200K → 10–12 nodes with good play;
  Hard ≈ $125K+ → 8–9.
- **PC economy:** zone rates skeptical 1 / cautious 3 / building 4 / engaged 5 /
  champion 8 per quarter; grade bonuses at year end. Rec tiers cost 18/12/6 PC
  (see below). Rec landing probability by zone: 20/40/58/75/90%.

## Investment tree (16 nodes, 4 tracks × 4 levels)

Tracks: Infrastructure (blue #4A9EFF), Manager Capacity (amber #F5A623),
Communications (teal #26C6DA), Stakeholder Engagement (purple #B39DDB).

- **Phase locking:** L1 available Year 0+, L2 Year 1+, L3 Year 2+, L4 Year 3+
  (`minYear = phase − 1`). This is what spreads investment across the game.
- **Every node has an `effect` object** (small direct ap/part/trust bump applied in
  `applyQueued`) *and* acts as a requirement gate for situation options. Both must
  stay true — a node whose description promises an effect that isn't wired in code
  is a bug (this happened once across all 16 nodes; don't reintroduce it).
- **Planned/approved:** gate the SE track (all levels) behind Trust ≥ Cautious (25),
  as a trust-zone rule, not a difficulty rule. Rationale: "you don't have the
  credibility to propose stakeholder engagement yet." Only SE, only behind
  Skeptical — do not gate more tracks or higher zones (over-coupling).

## Situation system

- Year 0: 4 hardcoded tutorial situations (`SITS`), consequences visible in Q1 only.
- Years 1–5: `YEAR_POOLS[year]` of 12 situations each; Fisher-Yates draw of 4 per
  playthrough. Year 5 pool is legacy-themed.
- **One situation per GAME (not per year) is guaranteed to draw.** `EXEC_DISAGREEMENT_IDS`
  lists 5 candidate exec-disagreement situations, one per year (`y1-j` Delia/Desmond,
  `y2-c` Raj/Delia, `y3-c` Raj/Desmond, `y4-c` Warren/Delia, `y5-d` Desmond/Delia).
  `S.init` picks ONE at random per playthrough into `S.guaranteedAnchorId`.
  `drawYearSituations(yr)` forces that single id into its year's draw if
  `yr` matches; every other year (including the other 4 candidate years) draws
  fully at random from its normal 12-situation pool — the other candidates are
  ordinary pool members that may or may not get drawn. This guarantees the
  player sees *a* version of the exec-friction beat exactly once across the
  5-year game, with which pairing varying by playthrough, rather than
  guaranteeing all 5 every game (that was tried and reverted — it made the
  device routine instead of a rare beat). If adding more candidate situations,
  append their ids to `EXEC_DISAGREEMENT_IDS` rather than flagging them
  `anchor:true` directly — anchor status is now assigned dynamically per game,
  not authored statically on the situation object.
- Pre-survey decision per year (`YEAR_PRESURVEY`); consequence revealed AFTER the
  choice, never before. Each year's best option also carries a small `effect.trust`
  (routed through `S.applyCappedTrust`) so the presurvey choice visibly connects to
  the main trust loop instead of only moving Participation.
- **4 balance-tested shapes** (3 narrative skins each): `relationship`,
  `resource_tradeoff`, `data_integrity`, `capacity_mechanic`. New situations should
  reuse a shape's cost/payoff profile; write new fiction, not new math. (Grown from
  2 skins/shape to 3 in one pass — verify pool counts stay a multiple of 4 across
  all 5 years if adding more, and re-run `sim.js` to confirm the larger draw pool
  doesn't shift trust/grade distributions before merging.)
- Special option effects: `variance` (±1 trust swing with narrated outcome),
  `pcGain`, `guaranteeNextRec`, `contractorHire` (cap decay not yet implemented),
  `treeUnlock`, 2-slot high-commitment options (peak arc moments, trust +6).
- **Situation content may vary slightly by difficulty only as small bespoke flavor**
  (e.g., 2–3 Hard-only situations that require low-trust context to make sense).
  Do NOT build parallel pools per mode — triples authoring and balance surface.

## Hard rules (violating these has burned us before)

1. **No free stuff.** Every tree node has a nonzero cost. Every meaningful option
   has a cost or a tradeoff.
2. **Hidden consequences.** Options show costs, not outcomes (exception: Year 0 Q1
   tutorial, explicitly labeled). Pre-survey consequences show after choice.
3. **Text must match mechanics.** If UI copy says a thing does X, the code must do
   X. Audit `unlocks` text and option notes when changing effects.
4. **Never reference unbuilt systems in content.** Two situations once referenced
   "recommendation slots" before recs existed — they were replaced.
5. **Apostrophes inside single-quoted JS strings break the file.** Use `&#39;`,
   rewrite the sentence, or escape properly. Syntax-check after every content edit.
6. **Use `onclick` attributes, not post-render `getElementById` handler loops**
   (the latter caused an unclickable-presurvey bug).
7. **Guard `Object.keys(x.tree)` style calls** — `tree` can be null at phase
   boundaries; use `(x.tree||{})`.
8. **Anything shown while a basket selection is pending must display real values
   plus pending as a separate annotation** — never pre-subtract (caused the
   "phantom $1K budget" confusion).

## Systems added late (know they exist before "adding" them)

- **Character arcs:** `CHAR_ARCS` — 7 characters × 4 stages; advance via relevant
  track investment (1 stage per 2 nodes) + key decision IDs. Stage changes swap
  ambient event pools. People panel (left panel + postsurvey grid) always shows
  all seven with role + stage.
- **Org tiers, deliberately NOT a second trust resource:** `CHARS[id].tier` is
  `'exec'` (Delia CHRO, Raj CFO, Desmond COO, Warren CEO) or `'plant'` (Marcus/
  Bell Creek, Yolanda/Riverside, Priya/Prairie Junction — one rep per location).
  All 3 plant reps share the same title, **Plant Manager** (Yolanda and Priya
  were originally "Supervisor," a level below Marcus — retitled so the tier
  reads as peers, not a mixed rank). Organizational Trust stays the single
  tracked resource; do not fork it per tier
  or per character — that would require re-deriving the whole balance model
  (zones, budget formula, PC rates, rec landing odds all key off the one number)
  and was explicitly rejected in favor of this lighter approach. `tierIds(tier)` /
  `tierStatus(tier)` (module-level helpers, just above the recommendations system)
  derive a display-only label (Early/Mixed/Building/Strong) from the *average arc
  stage* of that tier's characters — nothing new is tracked, it's a read of
  existing `S.charArcs` values. People panel (both renders) and the Company modal's
  Leadership section all group by tier with this derived label in the header, so a
  CFO and a plant manager read as different altitudes without a second gauge.
  `priya`/`desmond`/`warren` currently have empty `advanceOn.decisions` arrays (no
  situations reference them by id yet, so their arcs only advance via passive tree
  investment through `checkTreeArcAdvance`) — a future situations pass could give
  them decision-based bonus advancement the way Marcus/Delia/Raj/Yolanda have.
- **Company identity / About page:** `COMPANY_INFO` (name, product tagline, founding
  year, HQ blurb, mission, vision, 4 named values, history timeline, `plants` array
  with one entry per site — opened year + personality blurb, rendered in a
  "Locations" section) plus a `bio` field per `CHARS` entry — pure content layer,
  swap this object (and the bios) to reskin the game per the "this game is also an
  engine" rule at the top of this file. Rendered by `R.companyModal()` /
  `C.openCompany()`, reachable from a button on the difficulty-select screen, the
  in-game nav ("Company"), and by clicking any entry in the People panel (left
  panel or postsurvey grid). The 4 stated values
  (Integrity, People First, Continuous Improvement, Ownership) are referenced by
  name in a couple of situations (`y1-d`, `y3-f`) so decisions occasionally cite the
  company's own stated values directly, not just abstract trust/AP math — keep new
  values-tie-in situations reusing an existing shape's cost/effect numbers, per the
  situation-shape rule below, rather than inventing new mechanics for the callback.
  The 5 candidate situations in `EXEC_DISAGREEMENT_IDS` (see Situation system
  above) frame two execs disagreeing with each other in the scenario text (not
  just with the player), rotating the pairing by id rather than reusing the
  same two people every time — a lightweight way to make the exec cast feel
  like they have relationships with each other, not just with the CoE lead;
  the underlying options/numbers are unchanged from before each rewrite. If
  adding more candidates, keep rotating pairings rather than defaulting back
  to Raj/Delia every time.
- **Recommendations:** postsurvey review, max 2/year, tiered PC cost by diagnostic
  signal — high 18 PC (full land prob), medium 12 (×0.85), low 6 (×0.60 **and**
  trust dip if it lands: acting on the wrong signal backfires). Landing prob by
  zone: 20/40/58/75/90%. Explicit skip button with flavor note. Results resolve at
  NEXT year's postsurvey. `getRecDiagnostic()` reads real game state (5-arg call:
  `dimId, ap, part, year, treeBuilt` — an earlier version had a 6-arg signature with
  an unused `charArcs` param that shifted `treeBuilt` out of alignment with both call
  sites, silently zeroing the Manager Effectiveness dimension's signal; fixed, don't
  reintroduce an unused param ahead of `treeBuilt`) and picks the high/medium/low
  signal. Each dimension maps to a real stat + delta (`REC_DIMENSIONS[i].score/delta`
  — e.g. Manager Effectiveness → AP +5); `getRecReason()` explains *why* the signal
  is what it is in plain language citing actual ap/part/year/node values, and the
  issue-time card also states the target stat/delta so the payoff is visible before
  spending PC. Landed high/medium recs apply their real delta + trust +2 + PC +3;
  landed low-signal recs apply nothing but trust −1; a **deprioritized (unlanded)
  high/medium-signal rec queues into `S.recFallout`** and surfaces once, as a normal
  cause-chip on the resolve screen of a random quarter the *following* year (via
  `genAmbient()`, which checks `recFallout` before the regular ambient pool) — a
  real, data-supported ask that went unaddressed has an in-game consequence (small
  ap/part hit + trust −1, text keyed by dimension in `REC_FALLOUT_TEXT`), not just a
  quieter number at postsurvey. The "Recommendation Results" card names the exact
  stat/delta/trust/PC change per outcome instead of generic "scores improving" text.
- **Stagnation:** if something was buildable but nothing was built → −1 trust; if
  the decision also cost zero slots → −2. Never fires in Year 0 or when nothing
  was affordable.
- **Engagement** recalculates quarterly: `round(ap*0.55 + part*0.45)`.
- **yearHistory** snapshots `{year, ap, part, eng, grade}` at each postsurvey —
  feeds the trending table (left panel) and closing-screen SVG line charts.
- **Sound:** three programmatic sounds, toggle in nav, never autoplay before a
  user gesture (AudioContext init on first interaction).
- **Tree nodes are compact by default:** icon, name, cost/slot chips (or a lock
  message), nothing else. A `Details ▾` toggle per track (`S.treeDetailsOpen`,
  `C.toggleTreeDetails`) reveals each node's `unlocks` description text; built
  nodes get a small top-right checkmark badge (`.tn-check`, reused from the
  build-phase "selected" indicator — the two states are mutually exclusive so
  sharing the class is safe) instead of an inline "&#10003; Built" line. This is
  duplicated logic across two render sites — the inline build-phase tree
  (`R.build`) and the "View full tree" modal (`R.treeModal`) — both read the
  same `S.treeDetailsOpen` object, so a toggle made in one place is remembered
  in the other. `C.toggleTreeDetails` must re-render whichever one is actually
  on screen (`R.go()` for the inline tree, `R.treeModal()` if the modal element
  exists) rather than unconditionally opening the modal — got this wrong once.

## UI/UX conventions

- **Visual identity: "Foreman's Gauge."** Warm charcoal/brass instrument-panel
  aesthetic, not the generic dark-dev-tool look (electric blue/purple/teal,
  IBM Plex Mono, emoji icons) it started as. All ~90 hardcoded accent colors
  were consolidated into the existing `--amber/--blue/--green/--red/--purple/
  --teal` root variables (names kept for the ~200 call sites that reference
  them; values now hold brass/patina/sage/rust/gold/clay respectively — see
  the comment above `:root` in index.html) plus `color-mix(in srgb, var(--x)
  N%, transparent)` for tint/glow backgrounds, so a future palette change is
  one edit, not a file-wide find-replace.
- **Shape language communicates content type**, not just decoration:
  `--r-narr` (16px, rounded) on situation/decision cards — human/narrative
  moments; `--r-rep` (6px, semi-rounded) on resolve/postsurvey/reveal/closing
  cards — data/report moments; the mechanical shell (topbar, trust bar,
  resource strip, tree, buttons, modal) stays on the original sharp `--r/--rs/
  --rl` scale. Don't blur this — a card's radius should tell you what kind of
  content is in it before you read it.
- **Trust bar is a zone-tinted fluid gauge**, not a flat progress bar — see
  `ZONE_FLUID` in index.html and the `.trust-fill`/`.resolve-bar-fill`/
  `.ob-bar-fill` shared `::before`/`::after` rules. The gradient + meniscus
  highlight are `mix-blend-mode:soft-light` so they tint to whichever zone
  gradient sits underneath rather than sitting on top as a flat white patch —
  keep any future bar-style edits going through that same shared selector
  group, not a one-off `.trust-fill` override, or the other two bars silently
  drift out of sync.
- **Icons are inline SVG via the `II()` helper**, not emoji — see the icon
  fields on `NODES`/`TRACKS`/`CHARS`/`CHAR_ARCS`. Sized in `em` so they scale
  with whatever font-size their container already sets; adding a new node/
  character means calling `II('<path .../>')`, not picking an emoji.
- Contrast floors: `--t2:#C7BFAE` (9.4:1 on `--bg`), `--tm:#948A76` (5.0:1 on
  `--bg` / 4.6:1 on `--panel`) — both re-verified WCAG AA after the palette
  swap. Do not darken these, and re-check contrast with the actual `--bg`/
  `--panel` values (not eyeballed) if either changes again.
- Resource strip (Budget / Capacity dots / PC) always visible under the trust bar.
  Mobile additionally gets the score strip (AP/Part/Eng + Y·Q) — hidden on desktop
  via `@media(max-width:680px)` only.
- Minimal reading: resolve screen = trust bar + one-line consequence chips +
  compact BUILT chip + pulse. No toasts/popups duplicating summary info.
- Nav: `? Guide` (reopens onboarding as overlay), sound toggle, `⟳ New Game`.
- Reveal screen exists for Year 0 only; Years 1–5 go straight to postsurvey.
- Year 5 postsurvey button routes to `closing` (never "Begin Year 6").

## Owner preferences (Luke)

- Information density over padding; will flag verbose text and redundant screens.
- Wants incremental, targeted fixes — not wholesale rewrites — but will approve a
  full-system redesign when simulation shows the model is wrong (budget redesign).
- Expects simulation/playtesting evidence before and after balance changes.
- Catches mechanical-integrity gaps fast (text promising unwired effects).
- Accessibility matters: this may be shared with non-gamer HR colleagues. A future
  "streamlined mode" (orthogonal to difficulty: fewer systems surfaced, suggested
  picks) is on the idea list — difficulty = harder numbers, streamlined = less
  system management. Keep those concepts separate.

## Workflow

- Single file; syntax-check after every edit:
  `node -e "…new Function(js)…"` against the extracted `<script>` body.
- Check for duplicate DOM ids after render changes.
- Balance changes: write a quick Python/Node simulation of the 5-year trajectory
  first, compare against the targets above, then implement.
- Deployed at `lukeamadore.github.io/UnlockEngagement` (repo file must be
  `index.html`). Commit small, descriptive changes.

## Known gaps / backlog

- `contractorHire` grants +1 cap but the 2-quarter decay is not implemented.
- Recurring costs referenced in some copy (e.g., quarterly briefing slot) charge
  once, not recurring.
- FTE hire (Engaged-zone option) is narrative-only; no systemic capacity effect.
- SE-track trust gate (approved above) not yet implemented.
- Possible "streamlined mode" toggle (see Owner preferences).
- Verify no node is unreachable-in-principle on Hard with perfect play; adjust
  that node's price individually if so (not a scaling system).
