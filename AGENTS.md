# Language, Writing, and UI Rules (ASD-STE100 + Anti-Slop)

This file is the **single source of truth** for language, documentation, code comments, and UI rules in this repository.

**Context Budget Rule (Strict):**
- **DO NOT** call `read_file` on any `antislop` or `anti-slop` `SKILL.md` files (`C:\Users\andre\.gemini\config\plugins\antislop\**` or `C:\Users\andre\.gemini\skills\anti-slop\**`) unless the user explicitly asks to run a formal `after` audit.
- Apply all rules directly from this document in `during` mode (`antislop active: during (session override)`).

---

## 1. ASD-STE100 (Simplified Technical English)

1. **One word, one meaning, one part of speech.**
   - Use one approved term per domain concept (`transaction` instead of `tx`, `sale record`, or `payment item`; `merchant` instead of `vendor` or `seller`).
   - Keep each technical word to one grammatical role (`validate` as verb, `valid` as adjective, `check` strictly for the `GET /check/:merchant_id` endpoint).
2. **Sentence length and paragraph limits.**
   - **Procedural sentences (steps, runbooks, CLI commands):** Maximum **20 words**. Write one instruction per sentence.
   - **Descriptive sentences (code comments, README, UI copy, chat):** Maximum **25 words**. Maximum **6 sentences** per paragraph.
   - **Relaxed exception for `docs/**`:** Architecture and specification documents inside `docs/` are exempt from the 25-word sentence cap and 6-sentence paragraph cap so complex technical trade-offs, SQL queries, and state transitions can be explained without artificial splitting. All other ASD-STE100 and Anti-Slop rules (active voice, concrete verbs, zero buzzwords, no em dashes) still apply to `docs/**`.
3. **Noun cluster limit.**
   - Maximum **3 consecutive nouns**. Unpack longer clusters with prepositions or hyphens (`query for transaction status` instead of `transaction status query`; `hash collision on idempotency-key` instead of `idempotency key hash collision`).
4. **Imperative mood and explicit conditions.**
   - Start procedural steps with an imperative verb (`Run pnpm test`, `Set MYSQL_HOST`, `Verify the HTTP 200 status`).
   - Place the condition before the action (`If merchant_id returns HTTP 404, reject the sale with HTTP 404`).
5. **Quantify all tolerances and intervals.**
   - Never write `periodically`, `quickly`, `soon`, or `retry a few times`. State exact bounds (`every 30 seconds`, `timeout after 2000 ms`, `maximum 3 retries with 100 ms base backoff`).

---

## 2. Prose, Chat, and Documentation (`anti-slop` + `antislop-copywriting`)

1. **Start with the content.** Omit greetings, apologies, preambles, transition filler (`Let's dive in`), and sign-off invitations (`Let me know if you have questions`).
2. **Ban the em dash (`R-02`).** Never output `—` or spaced `--` in prose, docs, comments, or UI text. Use a period, comma, colon, or parentheses.
3. **Ban promotional and AI vocabulary (`R-16`).** Never use `robust`, `seamless`, `scalable`, `powerful`, `crucial`, `cutting-edge`, `production-grade`, `modern`, `comprehensive`, `elegant`, `streamlined`, `unlock`, `elevate`, `empower`, `delve`, `pivotal`, `testament`, `landscape`, or `journey`. State measurable bounds instead (`1 external call per 30s`, `p99 < 50ms`, `HTTP 422 on SHA-256 mismatch`).
4. **Use active voice and concrete verbs.** Name the actor and action (`PM Service validates merchant_id`). Replace nominalizations (`perform a validation`) with direct verbs (`validate`, `read`, `reject`, `insert`, `measure`). Do not give human verbs (`understands`, `decides`, `wants`) to inanimate objects.
5. **Ban AI rhythm tells (`R-36`).** No negative parallelism (`It is not just X, it is Y`), forced rule-of-three lists, staccato fragments, theatrical takeaways (`Let that sink in`), aphorisms (`X is the language of Y`), or temporal openings (`In today's fast-paced world`).
6. **Formatting discipline (`R-04`, `R-36`).** Use sentence case in headings and list items (never capitalize every word). Do not use decorative emojis (`🚀`, `✅`, `💡`) in headings, bullets, UI, or code. Avoid mechanical bolding of every key term or inline-header bullets that repeat the sentence.
7. **Verifiable evidence only (`C-5`, `R-17`, `R-36`, `R-38`).** Never invent metrics, benchmarks, customer names, or compliance claims. Mark unmeasured values as `not measured` or `[REAL DATA]`.

---

## 3. Code & Comment Hygiene (`antislop-code`)

1. **Explain why, never what (`R-31`).** Write a comment only when the code cannot show the constraint (protocol rule, concurrency lock, retry bound, hardware/OS quirk). Delete comments that restate the next line, variable, or function signature.
2. **Ban decorative comment formatting.** No ASCII banners (`// =======`), ALL-CAPS headers, step narration (`// Step 1:`), end markers (`} // end if`), decorative emojis, or vague `// TODO: improve this` notes.
3. **Length cap.** Maximum 1 to 2 sentence-case lines per logical block. Never write multi-sentence essay comments.
4. **Precision in code.** No unrequested abstractions, wrapper classes, or speculative configuration flags.

---

## 4. Admin Panel UI, Accessibility, and Layout (`apps/admin-panel`)

1. **Dials & Visual Identity (`R-01`, `R-20`, `R-29`, `R-37`).**
   - Lock operational dials to **ENERGY 1 (Calm) / RHYTHM 2 (Balanced) / MOTION 1 (Hover states only)**.
   - Maximum 2–3 core colors + 1 deliberate accent color. No blue-purple gradients, neon glows, background grids (`R-07`), pill-shaped everything (`R-11`), or glassmorphism on more than 1 element (`R-10`).
   - Every visual choice (font, color, card vs. table, shadow) must serve hierarchy or operator speed (`R-31`).
2. **Operator-First Dashboard Hierarchy (`C-3`, `R-05`, `R-20`).**
   - Build each view around the operator's action (inspecting circuit breakers, triggering 30s clearing batch, auditing idempotency leases, running load simulator).
   - Do not use a generic 4-stat-card template, Bento grid mosaic, fake terminal window, or decorative pulsing status dots (`R-13`, `R-19`). Status indicators must reflect live state.
3. **Real Telemetry & Honest Placeholders (`R-17`, `R-18`, `R-23`, `R-38`).**
   - Bind all numbers, deltas, charts, and tables to live backend endpoints or deterministic simulator output.
   - Never hardcode fake deltas (`+12% this week`), fictional activity feeds (`Sarah Chen`), or `John Doe` table rows. Use real data or explicit `[REAL DATA]` / input-format placeholders.
   - Title every chart with the specific question it answers (`Failed jobs per minute, last 15m`), never `Overview` or `Performance`.
4. **Functional Completeness & UI States (`C-2`, `C-4`, `R-24`, `R-26`, `R-27`, `R-34`).**
   - Every button, link, filter, and toggle must execute real logic. No dead links or non-functional controls.
   - Every data view must implement explicit, text-labeled **Empty** (cause + action to populate), **Loading** (what is loading), and **Error** (what failed + retry action) states.
   - If a theme toggle exists, both light and dark modes must pass all contrast and layout checks (`R-34`).
5. **Accessibility (`antislop-human`: `R-25`, `R-32`, `R-39`).**
   - **WCAG 2.2 AA Contrast:** Minimum `4.5:1` for normal text and `3:1` for large text (`18px+`), UI boundaries, chart segments, and `:focus-visible` outlines. Verify via `(L1 + 0.05) / (L2 + 0.05)`; never eyeball grey-on-grey.
   - **Keyboard & Semantics:** Logical `Tab` order, `Enter`/`Space` activation, `Escape` to close modals, native `<button>` for actions and `<a href>` for navigation, visible `<label>` on all inputs, `aria-label` on icon-only controls, `cursor: pointer` on enabled controls, `cursor: not-allowed` on disabled controls, and `@media (prefers-reduced-motion: reduce)` support.
   - Never communicate status by color alone; pair color with text or an icon.
6. **Responsive Reflow (`antislop-layoutmobile`: `R-03`, `R-35`, `R-40`).**
   - Define content-driven breakpoints across phone, tablet (`600px–1024px`), and desktop so grids restack cleanly without stretched single columns or cramped multi-columns.
   - Use fluid typography (`clamp()`), dynamic viewport units (`dvh`, never `100vh`), relative widths (`minmax()`, `min-width: 0`), zero horizontal overflow, and `200%` zoom support without clipping.
   - Enforce minimum `44x44px` tap targets with clear gaps and tap equivalents for any hover interaction.
   - Size all media elements explicitly (`width`/`height` or `aspect-ratio`) and verify interactive controls before completion (`R-35`).
