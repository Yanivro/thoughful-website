# Thoughtful — Lesson Engine Build Brief
## Instructions for Claude Code: What to build now, what comes later.

*This is the master task document. Read it fully before writing any code. It tells you what to build, in what order, and — critically — what NOT to build yet.*

---

## Project Context

Thoughtful is a native iOS app (Expo/React Native + NativeWind + Supabase) that teaches emotional intelligence through gamified lessons. The lesson engine is the core of the product — it renders 8 different interactive lesson formats from structured JSON data stored in Supabase.

### Reference Documents (read before building)

| Document | What it covers | Priority |
|---|---|---|
| `thoughtful_lesson_engine_spec.md` | Database schema, JSON content schemas for all 8 formats, format router pattern, caching strategy | **PRIMARY — this is your build spec** |
| `thoughtful_lesson_design.md` | Lesson philosophy, format definitions, worked examples for Peek L1 and L2 | Read for context on what each format *does* |
| `thoughtful_quiz_spec.md` | Quiz + algorithm + path definitions | Reference for path/dimension data when seeding |
| `thoughtful_design_principles.md` | Product philosophy, character definitions, design principles | Reference for tone/copy decisions |

### Tech Stack

- **Frontend:** Expo SDK, React Native, NativeWind (Tailwind CSS), React Navigation, Reanimated
- **Backend:** Supabase (PostgreSQL + Auth + Edge Functions)
- **State:** React hooks + Supabase realtime (no Redux)
- **Animations:** React Native Reanimated for lesson transitions, typing indicators, XP celebrations

---

## Build Stages

### ═══════════════════════════════════════
### STAGE 1: Database + Types (Build Now)
### ═══════════════════════════════════════

**Goal:** Supabase tables exist, TypeScript types are defined, seed data is inserted.

#### 1.1 — Create Supabase tables

Create all 6 tables exactly as specified in `thoughtful_lesson_engine_spec.md` Section 2:

- `paths` — 6 rows (one per character/dimension)
- `levels` — rows for built paths only (peek, still, dash × 3 levels each = 9 rows)
- `lessons` — JSONB content column, format-specific payloads
- `user_lesson_progress` — tracks choices, XP, completion per user per lesson
- `user_patterns` — claimed/understood/practiced stages
- `journal_entries` — insight journal

**Include all indexes specified in the spec.** The `idx_lessons_path_level` index is critical for performance.

#### 1.2 — Create TypeScript types

Define types for all content schemas in a single file: `src/types/lessons.ts`

Types to define:
- `LessonFormat` union type (8 formats)
- `ChatMessage`, `Option` (shared types)
- One interface per format content: `RecognitionTapContent`, `PerspectiveFlipContent`, `LiveSimContent`, `ContextSwitchContent`, `BeforeAfterContent`, `TagMessageContent`, `MirrorMomentContent`
- `LessonResult` (what a renderer returns on completion)
- `Lesson` (the full database row type)
- `UserPattern` with stages

**Do NOT create a type for `ToneMixerContent` yet.** See Deferred Items below.

All schemas are fully specified with examples in `thoughtful_lesson_engine_spec.md` Sections 3–4.

#### 1.3 — Seed Peek Level 1

Insert the 3 Peek L1 lessons + 1 mirror moment using the exact JSON from the spec's worked examples:

- Lesson 1: `recognition_tap` — "Does this sound familiar?" (the "I'm fine" scenario)
- Lesson 2: `perspective_flip` — "You asked. They said I'm fine."
- Lesson 3: `live_sim` — "Jordan texted." (4 branches with per-branch consequences)
- Mirror: `mirror_moment` — "twice this week someone said I'm fine..."

Also seed the `paths` table with all 6 paths and the `levels` table with Peek Level 1.

**Validation:** After seeding, run a SELECT query to confirm the JSONB content is valid and all foreign keys resolve.

---

### ═══════════════════════════════════════
### STAGE 2: Core Renderers (Build Now)
### ═══════════════════════════════════════

**Goal:** The 3 lesson formats used in Peek L1 render correctly from Supabase JSON data.

Build these components in this order (hardest first to de-risk):

#### 2.1 — LiveSimRenderer (most complex — build first)

This is the dating-sim-style branching conversation. Requirements:
- Display the incoming message with speaker name
- Show 4 reply options as tappable cards
- On tap: 400ms selection feedback, then show typing indicator for `consequence_delay_ms` (typically 600ms)
- Typing indicator animates (three dots), then the consequence message appears
- Below the consequence, show the optional `narration` text in italic
- Then show the per-branch `feeling` question with its specific options
- On feeling tap: show the `naming` takeaway text
- Return `LessonResult` with the choice, feeling, XP, and the `insight_label_override` if present

**Animation details:**
- Typing indicator: three dots pulsing (use Reanimated)
- Message appearance: slide up + fade in, like a real chat app
- Selection: selected option gets a subtle highlight, others fade slightly

**Critical rule from design doc:** Consequences must feel like real text conversations. The delay is not cosmetic — it creates the emotional beat of waiting for a reply.

#### 2.2 — RecognitionTapRenderer

Simpler than LiveSim. Requirements:
- Display the `moment` (either a chat conversation or scenario text)
- Show the `recognition` prompt and options as tappable cards
- On tap: if `followup` exists, transition to the followup question
- On followup tap (or if no followup): show the `naming` text
- Return `LessonResult`

**Key interaction:** The transition from moment → recognition should feel seamless. No screen change. The question fades in below the conversation.

#### 2.3 — PerspectiveFlipRenderer

Very similar to RecognitionTap but with a perspective badge. Requirements:
- Display the `badge_text` ("you're on the other side now") as a small chip/tag above the conversation
- Display the flipped `conversation`
- Show the `question` prompt and options
- On tap: show the `naming` text
- Return `LessonResult`

**Design note:** The badge is subtle — not a big banner. A small rounded pill, dimmed color, above the conversation bubble.

#### 2.4 — LessonScreen (Format Router)

The single screen that renders any lesson. Requirements:
- Accept a `lessonId` prop
- Fetch lesson data from Supabase (with caching — see spec Section 5)
- Read the `format` field
- Route to the correct renderer component
- Handle `onComplete` callback: save progress, save journal entry, award XP, navigate to next lesson

Implementation follows the pattern in spec Section 4 exactly.

#### 2.5 — Lesson Flow Logic

- `useLessonData(lessonId)` hook — fetches from Supabase, caches locally
- `useLevelLessons(pathId, levelId)` hook — fetches all lessons for a level, ordered by position
- `saveLessonProgress(lessonId, result)` — inserts into `user_lesson_progress`
- `saveJournalEntry(lesson, result)` — inserts into `journal_entries`
- `awardXP(amount)` — updates user's total XP (compute from progress table, don't store separately)
- `navigateToNext(lesson)` — determines next lesson in level, or level completion

---

### ═══════════════════════════════════════
### STAGE 3: Remaining Renderers (Build Now)
### ═══════════════════════════════════════

**Goal:** All free-tier lesson formats can render. This unblocks full content population.

Build in this order:

#### 3.1 — BeforeAfterRenderer
Two message versions displayed side by side (or stacked on narrow screens). Question about what shifted between them. Straightforward — similar structure to RecognitionTap.

#### 3.2 — ContextSwitchRenderer
Same message evaluated across 4 relationship contexts. Each context shows independently with its own tap options. Consider a horizontal swipe between contexts or a vertical scroll.

#### 3.3 — TagMessageRenderer
Display a conversation. Show dimension tags as tappable chips below. Multiple selection allowed. Feedback shows what they caught + what they missed. Needs a "done" button to submit selections.

#### 3.4 — MirrorMomentRenderer
Special format — appears after the last lesson in a level. Requires:
- Character animation display area
- Observation text (with template variable injection from user's actual choices — see Stage 5)
- Exactly 3 options: claim / explore / disclaim
- On "claim": create entry in `user_patterns` table with stage='claimed'
- On "explore": navigate to depth journey (DEFERRED — for now, show "coming soon" state)
- On "disclaim": note it silently, still unlock next level

**Template variable injection is deferred to Stage 5.** For now, use the static observation text from the JSON. Dynamic personalization comes later.

---

### ═══════════════════════════════════════
### STAGE 4: Content Population (Build Now, parallel with Stage 3)
### ═══════════════════════════════════════

**Goal:** All MVP lesson content exists as JSON in Supabase.

This stage is content work, not engineering. It can run in parallel with Stage 3.

#### Required content for MVP:

**Peek path (Learning to Open Up) — 12 lessons + 3 mirrors:**
- Level 1: "The Cost of the Wall" — 3 lessons + mirror (ALREADY SEEDED in Stage 1)
- Level 2: "One True Thing" — 4 lessons + mirror (partially designed in lesson design doc)
- Level 3: "Letting Someone In" — 4 lessons + mirror (format outline exists in design doc)

**Still path (Finding Your Ground) — 12 lessons + 3 mirrors:**
- Level 1–3: Need full lesson design and JSON creation

**Dash path (Stepping Forward) — 12 lessons + 3 mirrors:**
- Level 1–3: Need full lesson design and JSON creation

**Total MVP content:** ~36 lessons + 9 mirror moments = 45 JSON documents

#### Content creation workflow:
1. Design the lesson (scenario, options, consequences, takeaway)
2. Write the JSON matching the format schema
3. Validate JSON against the schema rules in spec Section 9
4. Insert into Supabase `lessons` table
5. Test in the app — lesson appears on next cache refresh

---

### ═══════════════════════════════════════
### STAGE 5: Pattern Tracking + Polish (Build Now, after Stages 2-3)
### ═══════════════════════════════════════

**Goal:** The invisible curriculum works — patterns are tracked, mirror moments are personalized, constellation updates.

#### 5.1 — Pattern Tracking System

Track user choices silently in `user_lesson_progress`:
- Which `choice_type` (grounded/shadow/middle) was selected per lesson
- Count shadow choices per dimension across a level
- Detect "first shifts" (user who previously chose shadow makes a grounded choice)

Update `user_patterns` table:
- On mirror moment "claim" → insert pattern with stage='claimed'
- On depth journey completion → update stage to 'understood' (DEPTH JOURNEY ITSELF IS DEFERRED)
- On first shift detection → update stage to 'practiced', trigger celebration

#### 5.2 — Mirror Moment Template Variables

The `mirror_moment` content JSON contains template variables like `{{shadow_count}}`. Before rendering, inject actual values from `user_lesson_progress`:
- `{{shadow_count}}` — number of shadow choices in this level
- `{{grounded_count}}` — number of grounded choices in this level

**Keep this simple for MVP.** Only support these two variables initially. More sophisticated personalization (referencing specific lesson titles, specific choices) is deferred.

#### 5.3 — Constellation Brightness Updates

After each lesson completion, recalculate dimension engagement scores:
- Count lessons completed per dimension
- Count reflections answered per dimension
- Count patterns claimed per dimension
- Map to brightness levels per the quiz spec (0-4 = dimmed, 5-7 = medium, 8-10 = bright)

**Important:** Constellation brightness is based on *engagement*, not on choosing the "right" answer. Shadow choices with completed reflections count the same as grounded choices.

#### 5.4 — Insight Journal UI

Simple list view of all `journal_entries` for the current user:
- Each entry shows: `insight_label`, `takeaway`, `created_at`
- Tapping an entry shows the full takeaway + optional user note
- User can add/edit their own note on any entry
- Entries grouped by path, ordered by date

---

## Deferred Items — DO NOT BUILD YET

These are explicitly out of scope for the current build. They are documented here so nothing is forgotten, but they should not be implemented until the stages above are complete and tested.

### ─── Deferred: ToneMixer Renderer ───

**What:** Format 7 from the lesson design doc. Three sliders (warmth, directness, openness) that dynamically update a draft message. User experiments with combinations and sends their preferred version.

**Why deferred:** 
1. It's the only format that requires 8-27 message variants per lesson — significantly more content work per lesson than any other format.
2. It's premium-only per the design doc, so no free-tier user is blocked.
3. The slider interaction pattern is the most complex UI in the entire app — three synchronized sliders with real-time text updates and smooth animations.
4. Without user feedback on the simpler formats, it's unclear whether the Tone Mixer adds enough value to justify the content + engineering cost.

**When to build:** After MVP launch, once there's user data on premium conversion and format engagement. If users love the Live Sim format and want more interactivity, Tone Mixer becomes the clear next format to build.

**What's already done:** The TypeScript type (`ToneMixerContent`) schema is designed in the lesson engine spec. The Supabase schema supports it — no database changes needed. The `lessons` table already accepts `format: 'tone_mixer'`. Only the renderer component and content need to be created.

**Holding state:** If a lesson with `format: 'tone_mixer'` is encountered by the router, display a "coming soon" card. Do not crash.

---

### ─── Deferred: Depth Journey Flow ───

**What:** When a user taps "tell me more" on a mirror moment, an AI-guided conversation helps them explore *why* the pattern exists and what it protects. This moves a pattern from "claimed" to "understood."

**Why deferred:**
1. It requires AI (Supabase Edge Function calling Claude API) — adds cost and complexity.
2. The design doc describes it as "always optional and always user-initiated" — no user is blocked by its absence.
3. The prompt engineering for the depth journey needs careful work to match the product voice (warm, specific, non-clinical).

**When to build:** After the free-tier lesson engine is stable and conversion to premium is being tested.

**What's already done:** The `user_patterns` table has the `stage` field and `the_protection` / `the_cost` text fields ready. The mirror moment renderer has the "tell me more" option. Only the actual conversational AI flow needs building.

**Holding state:** When "tell me more" is tapped on a mirror moment, show a brief message: "depth journeys are coming soon. for now, your pattern is saved." Still award the XP. Still claim the pattern.

---

### ─── Deferred: Weekly Narrative ───

**What:** Once a week, an AI-generated reflection on the user's actual choices that week. Written in second person, warm, specific. Example from the design doc: *"This week you explored the Peek avoidance pattern four times. You stayed on the safe side of the door three times — and once, when Jordan texted, you let one true thing through."*

**Why deferred:**
1. Requires AI generation (Edge Function + Claude API call).
2. Requires enough lesson data accumulated to be meaningful — users need at least a week of engagement.
3. The prompt needs to reference specific lesson names, choices, and patterns — this is the most complex AI prompt in the product.

**When to build:** After 2-4 weeks of real user data exists. The weekly narrative is a retention mechanic, not an activation mechanic — it's most valuable for returning users, not new ones.

**What's already done:** The `user_lesson_progress` table stores all the data needed to generate the narrative. No new tables required.

**Holding state:** No placeholder needed — users won't see a "weekly narrative" slot until it exists.

---

### ─── Deferred: AI-Powered Lesson Formats ───

**What:** Several premium lesson formats require real-time AI responses:
- **Rewrite It** — user writes their own version of a message, AI gives feedback on tone/clarity/honesty
- **Message Lab** — open-ended message writing with AI coach response
- **Tone Mixer with AI Response** — AI generates a contextually authentic reply to the user's mixed message
- **Pattern Journal with AI Reflections** — AI responds to claimed patterns with a short observation

**Why deferred:** All require Edge Functions, Claude API calls, and careful prompt engineering. They are the premium tier value prop ("the free tier shows you your patterns; the premium tier has a conversation with you about them"). Building them before the free tier is solid would be premature.

**When to build:** After premium tier pricing and paywall are implemented. These are the features that justify the subscription.

**What's already done:** Nothing in the codebase yet, but the lesson design doc fully describes each format's interaction model.

---

### ─── Deferred: Constellation Brightness Calculation (full formula) ───

**What:** The full formula for how dimension brightness changes based on engagement across lessons. The quiz spec defines initial brightness from quiz scores. The lesson engine needs to *update* brightness as the user progresses.

**Why deferred for full formula:** The basic version (Stage 5.3) covers MVP — simple counts of engagement per dimension. The *full* formula should consider:
- Recency weighting (recent lessons matter more than old ones)
- Pattern stage progression (practicing a pattern should boost brightness more than claiming)
- Cross-dimension effects (some lessons touch multiple dimensions)

**When to build:** After there's enough user data to calibrate the weights. Building a sophisticated formula before real usage data exists is guesswork.

**What's already done:** Stage 5.3 implements the basic version. The constellation display component from the quiz already exists and reads brightness values — it just needs an updated input source.

---

### ─── Deferred: Interstitial Animations Between Lessons ───

**What:** Character animations and motivational copy shown between lesson clusters (described in quiz spec as interstitials between clusters). For the lesson engine: transitions between lessons within a level, and level completion celebrations.

**Why deferred:** The lesson engine works without them. They're polish, not function. Building animations before the core interaction loop is proven is a distraction.

**When to build:** After all 8 renderers are working and content is populated. This is a polish pass.

**Holding state:** Simple "next lesson" transition — fade out, fade in. No character animation.

---

## Roadmap Summary

```
NOW (Stages 1-5)
├── Database schema + types
├── Seed Peek L1 lessons
├── 7 format renderers (all except ToneMixer)
├── Lesson flow engine (routing, XP, journal)
├── Basic pattern tracking
├── Content population (45 lessons across 3 paths)
└── Insight journal UI

NEXT (after MVP is stable)
├── ToneMixer renderer + content
├── Depth journey (AI-powered, pattern → understood)
├── Premium paywall + RevenueCat integration
├── AI lesson formats (Rewrite It, Message Lab)
└── Interstitial animations

LATER (after real user data)
├── Weekly narrative (AI-generated)
├── Full constellation brightness formula
├── Pattern journal with AI reflections
├── Phase 2 paths (Flare, Scout, Swell — full content)
└── Community features
```

---

## Constraints and Rules

1. **Lessons are data, not code.** Never hardcode lesson content in components. Always read from Supabase JSON.
2. **No lesson deletion.** Set `is_active = false` to retire. Never DELETE a row — progress records need the FK.
3. **XP for engagement, not correctness.** Shadow choice = 8 XP. Grounded choice = 10 XP. Never 0.
4. **One format per renderer.** Don't combine formats in one component. Each format has its own file.
5. **Cache lessons locally** after first fetch. Invalidate on app foreground by checking `max(updated_at)`.
6. **Never show scores, percentages, or "correct/incorrect"** anywhere in the UI. This is a design principle, not a suggestion.
7. **Typing indicators in LiveSim** are not optional. The delay creates the emotional experience. Do not skip them.
8. **Options must render as tappable cards** — never radio buttons, never checkboxes. The physicality of the tap matters.
9. **Auto-advance after selection:** 400ms delay, then transition. Consistent across all formats.
10. **Takeaway text** is always rendered in a distinct style — smaller, italic or different weight, feels like a journal entry. Never looks like a heading or a CTA.

---

*Last updated: March 2026*
*Owner: Yaniv Rose*
*Hand this document + the lesson engine spec to Claude Code to begin building.*
