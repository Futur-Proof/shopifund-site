# ADR-001: Gamification Architecture for the Application Wizard

**Status:** Accepted  
**Date:** 2026-05-15  
**File:** `apply.html`  
**Framework:** Yu-kai Chou's Octalysis (8 Core Drives, 4 Player Types, Experience Phases)

---

## Context

Loan application forms have one of the highest abandonment rates in financial services — typically 60–85% drop before completing. The conventional cause is friction: too many fields, too much perceived risk, and a bureaucratic frame that makes applicants feel like they are being evaluated rather than served.

The shopifund application wizard was designed to invert this. The goal was not a shorter form, but a more engaging one — one that uses gamification to lower the perceived cost of each step while simultaneously qualifying leads and routing them silently into the revenue waterfall (gold / silver / bronze tiers).

The design draws from Yu-kai Chou's Octalysis framework. Every mechanic in the current implementation maps to at least one named Core Drive. The player type distribution of a typical SMB loan applicant audience skews Achiever-heavy with a significant Explorer minority, which shaped the visual design priorities.

---

## Decision

Apply Octalysis gamification across the full 13-step wizard with the following principles:

1. **Priority-first opening** — Step 1 is a strategic choice (maximize capital / fund fast / best rate), not a data-collection step. This activates CD3 (Empowerment) immediately and reframes the entire experience.
2. **Silent lead scoring** — Qualify applicants at Step 2 without surfacing a tier label. Status is revealed through elevated UI treatment, not a score.
3. **Escalating commitment** — Step sequence deliberately increases input type difficulty: card tap → dropdown → number → free text.
4. **Conditional gating** — High-friction steps (bank connection, Step 12) are shown only when the data is operationally necessary.
5. **Endgame ritual** — The loading screen is a designed experience, not a spinner. It narrates value creation before results appear.

---

## Current Implementation

### Core Drive Activation

**CD1 — Epic Meaning and Calling**  
The form is framed as "discover what your business qualifies for," not "apply for a loan." The loading screen copy ("Scanning your matches... Checking lender eligibility across the network") positions shopifund as working on the applicant's behalf. The applicant is already in the network — they are uncovering their place in it.

**CD2 — Development and Accomplishment** *(dominant drive)*  
- `updateProgress()` computes `Math.max(7, Math.round((currentStep / totalSteps) * 100))` — bar never starts at zero
- Three milestone dots (Your Goal → Your Business → Your Match) shift pending → active (amber) → done (teal + checkmark) via `updateMilestones()`
- `.selected`, `.answered`, `.yes-selected` classes applied immediately on each card click — Instant Feedback on every micro-decision
- `div.affirmation.show` at Step 11: "Great. Now we can match you faster." — a phase-boundary Trophy signal

**CD3 — Empowerment of Creativity and Feedback**  
`selectPriority()` stores the Step 1 choice then calls `updateSubLabels()`, which maps `formData.priority` to the `PRIORITY_SUBLABELS` object and injects a priority-specific sublabel into every subsequent step header. The user's first decision reshapes the narrative framing of the entire form downstream.

**CD4 — Ownership and Possession**  
`scoreLeadTier()` runs silently inside `answerQualifier()` after each binary qualifier answer. It scores two questions (6+ months operating, $10K+/month revenue) and assigns `formData.leadTier` as gold / silver / bronze without surfacing the label. What the user sees:

- **Gold** (2 yes): `#gold-callout` — dark teal gradient, amber text, star icon. "You qualify for our top lender network."
- **Silver** (1 yes): no callout, Continue button appears
- **Bronze** (0 yes): `#bronze-callout` — reassurance copy, community framing

Results page: `submitForm()` branches the heading. Gold → "[BizName], you have 5 matches." Bronze → "[Name], here are your matches."

**CD5 — Social Influence and Relatedness**  
Three `.social-proof` injections at peak friction points:
- Step 6 (before revenue input): "Business owners like yours averaged $187,000 in matched offers this quarter."
- Step 13 (email gate): "Join 4,200+ businesses matched this week."
- Step 2 bronze path: community reassurance, not exclusion language

**CD6 — Scarcity and Impatience**  
Results page `.results-window`: amber clock icon, "Your match window is active for 48 hours." No literal countdown timer. Loss aversion activated through language, not fabricated mechanics. "No credit pull" trust badges appear on Steps 7, 8, 11, 13 — anxiety removal at each major abandonment risk point.

**CD7 — Unpredictability and Curiosity**  
`animateLoading()` ticks a counter from 0 to 47 using `Math.ceil(Math.random()*4)` increments at 80ms, simulating a live scan across a lender network. Five loading-step items check off at 600ms intervals. Randomized increments prevent the animation from feeling scripted even though the endpoint (47 lenders, 5 matches) is fixed.

Secondary: `toggleTip()` / `#credit-tip` on Step 7 — a Glanceable Curiosity mechanic. The "Why do we ask this?" link is always visible but never required. `toggleDetails()` and the About/Reviews tab system on results cards extend the same pattern into the Endgame phase.

**CD8 — Loss and Avoidance** *(restrained)*  
48-hour match window language on results. No fake countdown timers. No "2 spots left" fabrication. Loss aversion operates through anxiety removal (no credit pull, no commitment) rather than manufactured scarcity — appropriate for a financial product where trust is the primary conversion variable.

---

### Player Type Mapping

| Player Type | Want | How served |
|---|---|---|
| **Achiever** | Win, complete, max out | Progress bar starts at 7%, ticks on every step; gold callout signals elite access; amber `#1 for Your Profile` ranked badge on top lender card |
| **Explorer** | Discover, understand, probe edges | `toggleTip()` Step 7, `toggleDetails()` + About/Reviews tabs on lender cards — all optional depth layers, never gating |
| **Socializer** | Connection, belonging | "$187K averaged" and "4,200+ businesses this week" create a peer community being entered |
| **Killer** | Status relative to others | Gold callout is visually elevated (dark teal gradient, star), ranked badge establishes hierarchy, amber CTA differentiates the single best action |

---

### Experience Phase Coverage

**Discovery** — Handled outside `apply.html`. `URLSearchParams` reads `?amount=` from the query string, pre-populating `formData.loanAmount` from the hero dropdown on `index.html`. Continuity from first intent signal into the wizard.

**Onboarding (Steps 1–2)** — Zero typing. Three-card tap → two binary buttons → auto-advance at 220ms delay. Progress bar ticks twice. Gold callout fires at end of Step 2. Interaction pattern (tap, advance) is learned before any effort is asked.

**Scaffolding (Steps 3–12)** — Escalating Commitment: card taps → dropdowns → number input → free text. Each step raises investment cost of the previous one. Step 12 (bank connection) is conditionally gated by `needsBankStep()` — only shown when `credit ∈ [fair, poor]` AND `timeline === within-week`. All other paths skip silently from Step 11 → 13.

**Endgame (Loading + Results)** — Loading screen is a ~3.5s ritual. All milestones flip `.done` and progress bar hits 100% inside `submitForm()` before the results panel renders. Visual closure (complete bar, all dots green) signals the work is done and the reward has been earned before the applicant sees it.

---

### Named Techniques → Code

| Technique | Location |
|---|---|
| Progress Bar | `.progress-bar-fill`, `updateProgress()` |
| Milestone Unlock | `updateMilestones()`, three `.milestone` elements |
| Instant Feedback | `.selected`, `.answered`, `.yes-selected` on click |
| Social Proof | Steps 6 and 13 inline, Step 2 callouts |
| Status Points | `scoreLeadTier()`, `#gold-callout` / `#bronze-callout` reveal |
| Scarcity / Loss Aversion | `.results-window` 48-hour copy |
| Mystery Box | `animateLoading()` randomized counter |
| Glanceable Curiosity | `toggleTip()` / `#credit-tip` Step 7, `toggleDetails()` on results |
| Escalating Commitment | Step sequence: card → dropdown → number → text |
| Conditional Gating | `needsBankStep()`, Step 12 bypass |
| Personalization | `formData.bizName` injected at results heading |
| Trophy / Affirmation | `div.affirmation.show` Step 11 |
| Ranked Badge | `.lender-rank-badge` + `.ranked-first` amber border |

---

## Known Gaps in Current Implementation

**1. Silent tier is underused downstream**  
`scoreLeadTier()` generates a gold/silver/bronze classification that only branches in two places: the Step 2 callout and the results heading. Lender card order is static. The ranked badge always goes to the same lender regardless of tier. The scoring infrastructure supports more differentiation than the UI currently expresses.

**2. Loading screen copy is tier-agnostic**  
`ls-3-text` ("Matching your profile to top lenders") reads the same for gold and bronze. Bronze applicants could receive a different narrative during the load ("Finding lenders who work with growing businesses") to warm their expectation before results appear.

**3. No re-engagement hook**  
If a user abandons mid-wizard, there is no recovery path. Email is collected only at Step 13. A partial-progress save (localStorage) with a re-entry URL would allow abandoned sessions to resume.

**4. Results are static**  
All five lender cards are hardcoded. In a live system they would be dynamically ordered by tier, loan amount match, and product fit. The current UI architecture supports this but the data layer does not exist yet.

**5. No time-pressure mechanic on the form itself**  
CD6 (Scarcity) only fires post-submission. A soft "Lenders are reviewing profiles submitted today — yours is next in queue" indicator early in the wizard would extend the urgency frame further up the funnel.

---

## Future-State Options

These are ranked by implementation complexity and expected impact on conversion rate and lead quality.

---

### Option A — Dynamic Lender Ordering by Lead Tier *(high impact, medium effort)*

**What:** Sort the five lender cards at runtime based on `formData.leadTier`, `formData.loanAmount`, `formData.industry`, and `formData.timeline`. Gold tiers see lenders with lower factor rates first. Bronze tiers see lenders with higher approval rates first.

**How:** Replace the hardcoded lender card HTML with a `LENDERS` data array. `submitForm()` filters and sorts the array before rendering. Each lender object carries `minTier`, `minRevenue`, `industries[]`, `fastFund: bool`.

```js
const LENDERS = [
  { id: 'shield', name: 'Shield Funding', minTier: 'bronze', fastFund: true, factorRate: '1.15–1.45', ... },
  { id: 'kapitus', name: 'Kapitus', minTier: 'silver', fastFund: false, factorRate: '1.10–1.35', ... },
  // ...
];

function rankLenders() {
  return LENDERS
    .filter(l => tierRank(formData.leadTier) >= tierRank(l.minTier))
    .sort((a, b) => {
      if (formData.timeline === 'within-week') return a.fastFund ? -1 : 1;
      if (formData.priority === 'rate') return parseFloat(a.factorRate) - parseFloat(b.factorRate);
      return 0;
    });
}
```

**Impact:** Personalizes the endgame to every applicant's declared priority (Step 1) and silent score (Step 2). Achievers and Killers see a more credible ranking system. Converts the results page from a placeholder into a functional matching output.

---

### Option B — localStorage Session Resume *(high impact, low-medium effort)*

**What:** Persist `formData` and `currentStep` to `localStorage` after every `nextStep()` call. On page load, check for a saved session and offer to resume.

**How:**

```js
function saveSession() {
  localStorage.setItem('sf_session', JSON.stringify({ step: currentStep, data: formData }));
}

function loadSession() {
  const saved = localStorage.getItem('sf_session');
  if (!saved) return;
  const { step, data } = JSON.parse(saved);
  if (step > 1 && step < 14) {
    // Show resume banner
    document.getElementById('resume-banner').classList.add('show');
    document.getElementById('resume-btn').onclick = () => {
      Object.assign(formData, data);
      currentStep = step;
      showStep('step-' + step);
      updateProgress();
    };
  }
}
```

Resume banner design: amber pill at top of Step 1, "You left off at Step [N]. Continue where you stopped →". Dismiss clears the saved session.

**Octalysis:** CD2 (Accomplishment — progress is preserved), CD8 (Loss Avoidance — "don't lose your progress"). Also directly recovers abandoned sessions, which in a financial product likely represent 40–60% of all form starts.

---

### Option C — Real Countdown Timer on Results *(medium impact, low effort)*

**What:** Replace the 48-hour static language with a live countdown initialized from session start time.

**How:**

```js
function startMatchTimer() {
  const expiry = Date.now() + 48 * 60 * 60 * 1000;
  sessionStorage.setItem('sf_match_expiry', expiry);
  tickTimer(expiry);
}

function tickTimer(expiry) {
  const el = document.getElementById('match-timer');
  const interval = setInterval(() => {
    const rem = expiry - Date.now();
    if (rem <= 0) { clearInterval(interval); el.textContent = 'Expired'; return; }
    const h = Math.floor(rem / 3600000);
    const m = Math.floor((rem % 3600000) / 60000);
    el.textContent = `${h}h ${m}m remaining`;
  }, 60000);
  tickTimer(expiry); // immediate first tick
}
```

Display: amber clock icon + "47:52 remaining" inline in `.results-window`. Persists across page reloads via `sessionStorage`.

**Octalysis:** CD6 (Scarcity) elevated from language to live mechanic. A visible, decaying number creates stronger urgency than copy alone without crossing into fabricated scarcity (the 48-hour window is a real operational constraint — advisors do deprioritize cold leads).

**Risk:** If the timer reaches zero and nothing actually expires, trust erodes. Only implement if the backend enforces the window.

---

### Option D — Streak and Return Mechanic for Multi-Session Applicants *(medium impact, medium effort)*

**What:** Some applicants will not complete in one session (bank statement upload is an async task). Add a streak mechanic for applicants who return within 24 hours to complete.

**How:**  
On Step 12 (bank connection), if the user selects "Upload PDFs manually" and closes the wizard, save partial state to `localStorage` with a timestamp. On return within 24 hours, surface a streak badge: "Back within 24 hours — your application is still active."

Streak state:
```js
const lastVisit = localStorage.getItem('sf_last_visit');
const streak = lastVisit && (Date.now() - lastVisit) < 86400000;
if (streak) {
  document.getElementById('streak-badge').classList.add('show'); // "You're back — let's finish this."
}
localStorage.setItem('sf_last_visit', Date.now());
```

**Octalysis:** CD2 (Accomplishment — returning earns recognition), CD8 (Loss Avoidance — completing before session expires). Particularly effective for Explorer and Socializer player types who need more time to deliberate.

---

### Option E — Dynamic Copy Based on Priority × Industry *(high impact, high effort)*

**What:** Extend `PRIORITY_SUBLABELS` into a two-dimensional matrix. Current: `priority → sublabel`. Future: `priority × industry → sublabel`.

Current sublabel for `amount` priority, Step 5: "Finding you the lender with the highest offer."  
With industry context (captured at Step 4):

```js
const CONTEXTUAL_SUBLABELS = {
  'amount': {
    'restaurant': 'Restaurant cash advances up to $500K — based on your daily card volume.',
    'construction': 'Equipment financing with no down payment for licensed contractors.',
    'healthcare': 'Healthcare receivables advance against outstanding insurance claims.',
    'default': 'Finding you the lender with the highest offer.'
  },
  'speed': {
    'restaurant': 'Same-week MCA funding — common for food service businesses.',
    'default': 'Fast match in progress.'
  }
};
```

`updateSubLabels()` would consult `formData.industry` alongside `formData.priority` to pull the most specific copy available, falling back to `default`.

**Octalysis:** CD3 (Empowerment — the form feels like it knows the applicant's industry), CD5 (Social Proof — industry-specific copy implies others like them have been matched). Also reduces cognitive load: applicants in specialized verticals don't have to map generic copy to their specific situation.

**Implementation note:** This requires 13 priority × industry combinations across 13 steps = up to 169 copy strings. Build a content management layer (even a simple JSON file) rather than expanding the inline JS object.

---

### Option F — Adaptive Step Ordering Based on Priority *(high impact, high effort)*

**What:** Currently the step order is fixed (1–13 minus optional Step 12). Future: reorder steps based on Step 1 priority choice.

- `priority === speed` → move timeline step (Step 11) to Step 3, before business data. Fastest path to a funding decision.
- `priority === rate` → move credit step (Step 7) earlier. Rate matching requires credit context sooner.
- `priority === amount` → keep current order (revenue and business age are the primary amount signals).

**How:** Replace `currentStep++` with a `STEP_SEQUENCE` map per priority:

```js
const STEP_SEQUENCES = {
  speed:  [1, 2, 3, 11, 4, 5, 6, 7, 8, 9, 10, 12, 13],
  rate:   [1, 2, 3, 4, 7, 5, 6, 8, 9, 10, 11, 12, 13],
  amount: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13]
};

let stepSequence = STEP_SEQUENCES['amount'];

function nextStep() {
  const idx = stepSequence.indexOf(currentStep);
  const nextIdx = idx + 1;
  if (nextIdx >= stepSequence.length) return;
  currentStep = stepSequence[nextIdx];
  if (currentStep === 12 && !needsBankStep()) {
    currentStep = stepSequence[nextIdx + 1];
  }
  updateProgress();
  showStep('step-' + currentStep);
}
```

`selectPriority()` sets `stepSequence` as its last action.

**Octalysis:** CD1 (Epic Meaning — the form is literally shaped by your stated goal), CD3 (Empowerment — choice has real structural consequences, not just copy consequences). This is the deepest gamification commitment in the list because it requires the step sequence itself to carry meaning.

**Risk:** `prevStep()` must traverse the sequence in reverse. Progress bar percentages must be computed against sequence position, not step number. QA surface area expands significantly.

---

### Option G — Social Proof Personalization via Industry *(medium impact, low-medium effort)*

**What:** The current social proof strings are static. Extend them to branch on `formData.industry` once Step 4 is completed.

```js
const INDUSTRY_SOCIAL_PROOF = {
  'restaurant': 'Restaurant owners averaged $143,000 in matched offers last quarter.',
  'construction': 'Contractors in your region averaged $218,000 — primarily equipment financing.',
  'healthcare': 'Healthcare practices averaged $165,000 in matched working capital.',
  'retail': 'Retail businesses averaged $97,000 — mostly lines of credit for inventory.',
  'default': 'Business owners like yours averaged $187,000 in matched offers this quarter.'
};

function updateSocialProof() {
  const copy = INDUSTRY_SOCIAL_PROOF[formData.industry] || INDUSTRY_SOCIAL_PROOF['default'];
  document.querySelectorAll('.social-proof-dynamic').forEach(el => el.textContent = copy);
}
```

Call `updateSocialProof()` at the end of `nextStep()` when `currentStep > 4`.

**Octalysis:** CD5 (Social Influence — industry-peer comparison is more credible than generic comparison), CD4 (Ownership — the form recognizes what kind of business you run).

---

### Option H — Micro-Celebration on Milestone Completion *(low impact, low effort)*

**What:** Add a brief CSS animation when a milestone dot flips to `.done` — a pulse ring, a checkmark pop, or a confetti burst scoped to the milestone element.

```css
@keyframes milestone-pop {
  0%   { transform: scale(1); }
  40%  { transform: scale(1.4); }
  70%  { transform: scale(0.9); }
  100% { transform: scale(1); }
}
.milestone.done .milestone-dot {
  animation: milestone-pop 0.4s ease forwards;
}
```

For a stronger effect: a 12-particle confetti burst using a small canvas overlay scoped to the milestone element's bounding rect, triggered via JS at phase transition.

**Octalysis:** CD2 (Accomplishment — visual confirmation of phase completion is a Trophy mechanic). The cost is < 20 lines. The psychological payoff at each milestone boundary is disproportionate to the implementation effort.

---

### Option I — Behavioral Nudge on Idle / Hesitation *(medium impact, medium effort)*

**What:** Detect when a user pauses on a step for more than 20 seconds without advancing. Surface a contextual nudge.

```js
let hesitationTimer;

function resetHesitation() {
  clearTimeout(hesitationTimer);
  hesitationTimer = setTimeout(showNudge, 20000);
}

const NUDGES = {
  7: "Most businesses qualify even with fair credit — lenders focus on cash flow.",
  12: "Bank connection takes 30 seconds via Plaid. Manual upload works too.",
  13: "Your info is encrypted and never sold to third parties."
};

function showNudge() {
  const msg = NUDGES[currentStep];
  if (!msg) return;
  const el = document.getElementById('step-nudge');
  el.textContent = msg;
  el.classList.add('show');
}
```

Nudge is a small teal pill below the primary input, auto-hides after 6 seconds or on next interaction.

**Octalysis:** CD8 (Loss Avoidance — addresses the specific fear causing hesitation), CD7 (Curiosity — provides information the user was probably about to search for). Most valuable at Steps 7 (credit) and 12 (bank connection), the two highest-friction steps.

---

## Consequences

The current implementation is a complete, functional foundation across all eight Core Drives and all four player types. It is production-ready as a static wizard.

The gap between current and future state is a backend integration gap, not a gamification design gap. Options A (dynamic lender ordering), B (session resume), and D (return streak) are the highest-leverage next steps because they address the two biggest conversion leakage points: abandonment and lead qualification depth. They can be implemented without a backend — A via a JS data array, B and D via `localStorage`.

Options E and F (dynamic copy matrix, adaptive step ordering) require the most effort but would produce the most personalized experience in the financial lead-gen category. They should be scoped once the product has real lender partner data to populate the content.
