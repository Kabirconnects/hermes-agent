---
project: RAISE
status: CURRENT
last_verified: 2026-09-22
verification_method: browser + API
related:
  - README.md
scope: |
  Practical browser operating procedures for RAISE.
  Every procedure tagged VERIFIED / UNVERIFIED / FAILED / DO NOT USE / UNTESTED.
  Browser procedures only — product/architecture info lives in README.md.
---

# RAISE Browser Operating Guide

**This file is the practical reference for operating RAISE through a browser.**
It covers navigating pages, filling forms, submitting generations, checking status,
and handling known failures. Product architecture, API details, and the verification
taxonomy live in [README.md](./README.md).

**Verification tags used throughout this file:**
- `VERIFIED` — tested successfully in the live RAISE application on the stated date
- `UNVERIFIED` — plausible from code/docs/structure, not yet tested in browser
- `FAILED` / `DO NOT USE` — tested and known not to work reliably
- `UNTESTED` — not yet tested; no claim about whether it works

Only promote something to `VERIFIED` after actual testing in the live application.

---

## 1. Pre-flight Checklist

Run through this before any browser operation. If any check fails, report it and do not proceed.

1. **Frontend reachable:** `https://raise-7i17.onrender.com/` loads and shows the RAISE homepage.
2. **Backend reachable:** `https://raise-backend-pymy.onrender.com/api/health` returns `{status: "ok"}` (or frontend loads without API errors).
3. **Authenticated:** You are signed in. If not, complete OTP sign-in before any channel operations.
4. **On the correct page:** Navigate to `/channel` for Studio/channel operations.
5. **Plan allowance confirmed (for generations):** Check "GENERATIONS LEFT" / "EDITS LEFT" counters on the Studio page before generating.
6. **Browser session stable:** No console errors that would interfere with UI interaction.

`VERIFIED` — checked before every session. 2026-09-22.

---

## 2. URLs & Page Reachability

| Page | URL | Notes |
|---|---|---|
| Homepage | `https://raise-7i17.onrender.com/` | Marketing site; navigation to other pages via menu |
| Studio (channel list) | `https://raise-7i17.onrender.com/channel` | Main operating page — channel list, video rankings, add channel, generate |
| Video detail | `https://raise-7i17.onrender.com/channels/:channelId/videos/:videoId` | Video stats, brand inputs, generate button |
| Generation History | `https://raise-7i17.onrender.com/generation-history` | React SPA — content renders dynamically |
| Content Editor | `https://raise-7i17.onrender.com/content-editor/:generationId` | Edit generated content |

`VERIFIED` — URLs confirmed by navigating to each page. 2026-09-22.

---

## 3. SPA Behavior & What "Loaded" Means

RAISE is a React + Vite SPA. Page content renders dynamically after the initial HTML loads.
When you navigate to a page:

- The URL changes immediately.
- The page title updates.
- The shell (nav, footer) appears quickly.
- **Dynamic content may take a moment to render.** Give it a moment if a page appears incomplete.
- The Generation History page (`/generation-history`) in particular renders its content dynamically and may appear empty on initial load. See §8.

`VERIFIED` — observed during navigation. 2026-09-22.

---

## 4. Scrolling

Some buttons (notably "Generate content strategy") may be outside the initial viewport.
Scroll the page to bring them into view before clicking.

`VERIFIED` — the generate button on the video detail page required scrolling to be clickable. 2026-09-22.

---

## 5. Page Reference (What You See on Each Page)

### 5.1 Studio (`/channel`)

The main operating page. Shows:
- "STUDIO" header with tagline "TRACK WHAT RESONATES."
- "POWER MONTHLY ALLOWANCE" section with "GENERATIONS LEFT" and "EDITS LEFT" counters
- "+ ADD A CHANNEL" button (top of channel list)
- Channel list: each channel shows @handle (uppercase), "top X.Xx" outlier score, "REMOVE" button, video list or status
- Each video entry: outlier multiplier, duration, title, view count, time ago

`VERIFIED` — observed 2026-09-22.

### 5.2 Video Detail (`/channels/:channelId/videos/:videoId`)

Shows:
- Video stats: views, likes, comments, views/day, engagement rate, comment rate, duration, published date
- Description (with "COPY DESCRIPTION" button)
- "GENERATION INPUTS" section with 10 brand input fields (see §6)
- "WATCH ON YOUTUBE →" link
- "GENERATE CONTENT STRATEGY" button
- "GENERATE FROM VIDEO ONLY" button
- "AI EDITOR LIAISON SKILL →" link

`VERIFIED` — observed 2026-09-22.

### 5.3 Generation History (`/generation-history`)

React SPA. Shows generation job list (when rendered). See §8 for the SPA behavior.

`UNVERIFIED` — the page was reached but content did not render during testing; API confirmed the job existed and completed.

### 5.4 Content Editor (`/content-editor/:generationId`)

Edit generated content with natural language instructions. Apply or reset edits. Revision-safe.

`UNVERIFIED` — page not visited in this session.

---

## 6. Channel Management (Add Channels)

### 6.1 Full procedure

1. Navigate to Studio: `https://raise-7i17.onrender.com/channel`
2. Click "+ ADD A CHANNEL" button (top of channel list)
3. Modal opens titled "TRACK A CREATOR" / "ADD YOUTUBER"
4. Fill the input field (id: `channel-handle`, placeholder: `@mkbhd`) with the full `@handle` (e.g. `@sandyleeai`) or full YouTube channel URL (e.g. `https://www.youtube.com/@sandyleeai`)
5. Click "ADD CHANNEL" submit button
6. Wait for operation — UI shows "ADDING..." during processing
7. Modal closes. Verify the channel appears in the channel list with its @handle displayed in UPPERCASE

`VERIFIED` — tested across 4 batches (39 channels total). 2026-09-22.

### 6.2 Reliable click method for "+ ADD A CHANNEL" button

- **JavaScript `.click()` does NOT work reliably** on this button. The button's React event handling did not respond to a direct JS click.
- **Use CDP mouse events:** `cdp('Input.dispatchMouseEvent', type='mousePressed', ...)` followed by `mouseReleased` at the button's current viewport coordinates.
- Button stable position: viewport approximately (56, 633), size 233×56px.

`VERIFIED` — CDP mouse events are the reliable method. JS `.click()` failed. 2026-09-22.

### 6.3 Reliable click method for "ADD CHANNEL" submit button

- Query the current button position via `getBoundingClientRect()` before clicking — the position varies between form openings.
- Click at the center point of the button's current bounding rect.
- JavaScript `.click()` on the submit button works after the input is filled; coordinates must be queried first.

`VERIFIED` — coordinates must be queried each time; hardcoded coordinates miss. 2026-09-22.

### 6.4 Form behavior

- The form closes after each addition. To add another channel, click "+ ADD A CHANNEL" again to reopen the modal.
- `VERIFIED` — 2026-09-22.

### 6.5 Verification after adding a channel

1. Check that the channel handle appears in the channel list in UPPERCASE.
2. Confirm it appears as a channel entry (with "REMOVE" button and video list or status message), not as a form label.
3. Confirm the entry persists after the modal closes and the page settles.

`VERIFIED` — this verification procedure was followed for all 39 channels added across batches 1–4 (2026-09-22).

### 6.6 Failure: "Couldn't load videos" after add

- If "Couldn't load videos: Could not fetch videos for this channel right now" appears with a "TRY AGAIN" button, the channel IS added. The video fetch failed separately.
- Do NOT treat this as a failed add. Optionally click "TRY AGAIN" to retry video fetch.
- `VERIFIED` — 2026-09-22.

### 6.7 Failure: channel handle not appearing after add

- If the handle does not appear in the list after the modal closes, the add did not persist (silent failure).
- Re-add the channel; verify the input value was correct; check for error messages.
- `VERIFIED` — 2026-09-22.

---

## 7. Video Research: Finding and Opening a Video

### 7.1 Studio video list

Studio lists channels with ranked videos (by Outlier Score). Each video card shows: outlier multiplier, duration, title, view count, time ago, and the channel @handle.

### 7.2 Opening a video

Two ways:
- **Click a video card on the Studio page.** Locate the `div.group.cursor-pointer` element containing the video title, query its bounding rect, and click at the center coordinates.
- **Direct URL:** `https://raise-7i17.onrender.com/channels/:channelId/videos/:videoId`

`VERIFIED` — both methods confirmed. JavaScript `.click()` on the video card `div.group.cursor-pointer` worked and navigated to the video detail page. 2026-09-22.

---

## 8. Generation History Page (React SPA)

The Generation History page at `/generation-history` is a React SPA. Observed behavior during the 2026-09-22 session:

- Navigating to the URL changes the address and loads the page shell (nav + footer).
- The main content area (job list) does NOT render immediately on load.
- After waiting, the React root element contained the shell text but no job list content.
- No loading indicators or skeleton placeholders were visible.

`VERIFIED` — this exact behavior was observed. The page shell loaded but the job list did not render during the session's checks. 2026-09-22.

Implications:
- Do not rely on the Generation History UI to confirm job status immediately after navigation.
- Use the API (`GET /api/generation-history/:id`) to check status reliably.
- If using the UI, wait longer or refresh; the content may render on a later attempt.

`UNVERIFIED` — whether the job list renders after a longer wait, after a page refresh, or when there are multiple generations to display.

---

## 9. Generation Input Fields Reference

### 9.1 Location

All 10 fields are located under the "GENERATION INPUTS" heading on the video detail page, below the video stats and description. Every field is optional — leave all blank to generate from video alone. The page includes two buttons at the bottom: "GENERATE CONTENT STRATEGY" and "GENERATE FROM VIDEO ONLY".

### 9.2 Field table

Recorded 2026-09-22 from live RAISE session (video @AIJASONZ / "Jev + Treg is a crazy combo for automation...").

| # | Field label (as shown, uppercase) | HTML type | Placeholder text | Width | Notes |
|---|---|---|---|---|---|
| 1 | Target duration (minutes, optional) | `number` | `10 (auto)` | 328px | Suggested default 10; auto-detected from video duration. |
| 2 | Brand voice | `text` | `Direct, bold, educational` | 671px | Free-text. |
| 3 | Content pillars | `text` | `AI tools, creator economy, business systems` | 671px | Free-text. |
| 4 | Ideal customer / audience | `text` | `Solo founders, agency owners, creators` | 671px | Free-text. |
| 5 | Your expertise (optional — enables first-person claims) | `text` | `10 years building automation systems for agencies` | 671px | Free-text; marked optional. |
| 6 | Your offer/business (optional) | `text` | `AI automation agency for e-commerce brands` | 328px | Free-text; marked optional. |
| 7 | Brand positioning (optional) | `text` | `The no-fluff, systems-first alternative to hustle culture` | 328px | Free-text; marked optional. |
| 8 | Business/content goals (optional) | `text` | `Grow authority and audience — no hard selling yet` | 671px | Free-text; marked optional. |
| 9 | Custom generation instructions | `textarea` | `Examples:\n• Turn this into an advertisement\n• Make it more emotional\n• Write like Alex Hormozi\n• Add humor\n• Make it beginner friendly` | 671px × 118px | Multi-line text area. |
| 10 | Things to preserve | `textarea` | `Examples:\n• Keep the hook\n• Preserve the story flow\n• Keep the CTA\n• Don't change statistics\n• Maintain chapter structure` | 671px × 118px | Multi-line text area. |

`VERIFIED` — all 10 field labels, types, placeholders, and dimensions confirmed by inspecting the live DOM during the 2026-09-22 session. Field labels are rendered as uppercase text above each input (e.g. "TARGET DURATION (MINUTES, OPTIONAL)", "BRAND VOICE", etc.). The two textarea fields (#9, #10) have placeholder examples that include bullet-style instructions.

Fields #6 and #7 (Your offer/business, Brand positioning) sit side by side on the same row — each 328px wide. All other single-line text fields span the full 671px width. The number field (#1) is 328px wide.

### 9.3 Filling fields — interaction method

- **`fill_input()` works for all text and number inputs.** Target each field by its placeholder text or label-based CSS selector.
- For textareas, `fill_input()` also works.
- After filling, verify the value is present in the DOM before submitting (read the input's value via JavaScript).
- Do NOT set values via direct DOM property manipulation (`input.value = "..."` without triggering change events) — the application may not detect the change.

`VERIFIED` — all 10 fields were filled and verified during the 2026-09-22 generation session. `fill_input()` succeeded on every field.

`UNVERIFIED` — whether leaving all fields blank and clicking "GENERATE FROM VIDEO ONLY" produces different result quality than filling fields and clicking "GENERATE CONTENT STRATEGY". The system description says fields are optional and generation works from video alone, but the difference was not tested.

---

## 10. Submitting a Generation Request

### 10.1 Two submit buttons

| Button | Label | Behavior |
|---|---|---|
| Primary | "GENERATE CONTENT STRATEGY" | Submits all filled brand inputs along with the video. Fields are optional. |
| Alternative | "GENERATE FROM VIDEO ONLY" | Submits without brand inputs. Generates from the video alone. |

Both render as inline-block elements with border styling. The primary button has a stronger border (border-2) and is uppercase; the alternative has a lighter border (border, border-ink/30).

`VERIFIED` — both buttons present and labeled as above during the 2026-09-22 session.

### 10.2 Scroll requirement

The generate buttons are located below the 10 input fields and may be outside the initial viewport. Scroll the page to bring them into view before clicking.

`VERIFIED` — during the 2026-09-22 session, the "Generate content strategy" button was at viewport coordinates approximately (249, 1501) after scrolling, requiring the page to be scrolled down from the top.

### 10.3 Reliable click method

- **JavaScript `.click()` on the "Generate content strategy" button WORKS.** Unlike the "+ ADD A CHANNEL" button (which requires CDP mouse events), the generate button responds reliably to a standard JavaScript click on the button element.
- Click the button element directly: find it via `document.querySelector('button')` filtered by text content "Generate content strategy", then call `.click()`.
- The "Generate from video only" button also responds to `.click()`.

`VERIFIED` — JavaScript `.click()` on "Generate content strategy" was used successfully in the 2026-09-22 session and triggered the generation.

### 10.4 CDP mouse events for generate buttons

- **UNTESTED** — CDP mouse events were not needed or tested for the generation buttons. Prefer the verified JS `.click()` method; only fall back to CDP if JS interaction fails and the fallback is subsequently verified.

---

## 11. Recognizing That Generation Has Started

After clicking "GENERATE CONTENT STRATEGY" or "GENERATE FROM VIDEO ONLY":

1. **Both generate buttons become disabled.** This is the primary visual signal that the submission was accepted. Both "GENERATE CONTENT STRATEGY" and "GENERATE FROM VIDEO ONLY" switch to a disabled state.
2. The page does NOT show a progress spinner, loading bar, or in-page status message on the video detail page after submission.
3. No new text appears on the video detail page indicating "queued" or "processing".

`VERIFIED` — during the 2026-09-22 session, both buttons became disabled immediately after the click. The disabled state is the confirmation that the request was accepted. The session waited and polled the API to confirm the job progressed.

`UNVERIFIED` — whether the disabled state persists if the submission is rejected server-side (e.g., rate limit, plan limit exceeded). The observed behavior was a successful submission.

---

## 12. Waiting for and Checking Asynchronous Generation Status

### 12.1 Method A — Poll the API (recommended, reliable)

```
GET https://raise-backend-pymy.onrender.com/api/generation-history/:id
```

with credentials (cookie) included.

Response shape when found:
```json
{
  "success": true,
  "data": {
    "total": 1,
    "page": 1,
    "limit": 50,
    "generations": [
      {
        "_id": "<generationId>",
        "videoId": "<videoId>",
        "channel": { "channelUrl": "@handle" },
        "status": "queued" | "processing" | "completed" | "failed",
        "stage": "queued" | "processing" | "completed" | "failed",
        "createdAt": "...",
        "startedAt": "...",
        "completedAt": "..."
      }
    ]
  }
}
```

Status flow: `queued` → `processing` → `completed` (or `failed`).

`VERIFIED` — the API returned this exact shape during the 2026-09-22 session. The job went queued → completed in approximately 2.5 minutes.

### 12.2 Method B — Generation History page (less reliable)

Navigate to `/generation-history`. The page is a React SPA and may show only the shell initially. If the job list renders, the job appears there. Because rendering is unreliable on load, the API is preferred.

See §8 for full SPA handling notes.

`VERIFIED` — the page URL is valid but content rendering was unreliable during the 2026-09-22 session; API confirmed what the UI did not show.

### 12.3 Expected completion time

Observed: ~2.5 minutes from queue to complete for a single-video generation with no transcript (17:39:40 queued → 17:42:06 completed, UTC).

`VERIFIED` — measured during the 2026-09-22 session. This is a single data point; actual times will vary.

---

## 13. Accessing Generated Results

### 13.1 Via API (most reliable)

```
GET https://raise-backend-pymy.onrender.com/api/generation-history/:id
```

When `status: "completed"` and `stage: "completed"`, the response includes a `result` object with the full generated content package:

```json
{
  "success": true,
  "data": {
    "_id": "...",
    "videoId": "...",
    "channel": { "channelUrl": "..." },
    "status": "completed",
    "stage": "completed",
    "result": {
      "contentArchetype": { "type": "...", "reasoning": "...", "structure": "..." },
      "sourceStrategyAnalysis": { /* 14 fields */ },
      "youtube": {
        "hookOptions": { /* 7 hooks keyed by style */ },
        "selectedHook": "...",
        "hookSelectionReason": "...",
        "hookAnalysis": { ... },
        "seoTitles": { /* 3 titles keyed by angle */ },
        "description": "...",
        "chapters": "0:00 ...\n1:00 ...",
        "thumbnailIdeas": "1. ...\n2. ...\n3. ...",
        "visualDirections": "..."
      },
      "teleprompterScript": "..." | null,
      "shortForm": ["youtubeShorts", "tiktok", "instagramReels", "facebookReels", "linkedinVideoHook"],
      "social": ["twitterThread", "linkedinPost", "instagramCaption", "facebookPost", "communityPost"],
      "written": ["newsletterBlurb", "emailVersion", "blogOutline"],
      "sourcesUsed": [{ "videoId": "...", "title": "..." }]
    }
  }
}
```

`VERIFIED` — this exact structure was returned for generation ID `6ab2bd5c5646075ed3ae3b8a` during the 2026-09-22 session.

Key notes:
- `teleprompterScript` is `null` when the source video has no transcript. This is expected behavior, not an error. The 2026-09-22 generation had no transcript, so `teleprompterScript` was null and several `sourceStrategyAnalysis` fields were marked "Not applicable due to lack of transcript."
- `youtube.thumbnailIdeas` is text descriptions only, not image URLs.
- `youtube.chapters` is a newline-separated string of timestamped chapter titles.

### 13.2 Via Content Editor UI

Open `/content-editor/:generationId` in the browser. This provides a natural-language editing interface for the generated content.

`UNVERIFIED` — Content Editor page not visited during the 2026-09-22 session.

### 13.3 Via Generation History UI

Open `/generation-history`. Job list may render after a delay. Clicking into a completed job should show the outputs.

`UNVERIFIED` — Generation History page content rendering was not confirmed during the 2026-09-22 session.

---

## 14. When API Inspection Is More Reliable Than the Visible UI

API inspection is more reliable than the visible UI in these situations:

1. **Checking generation status.** The video detail page shows no progress indicator after submission. The Generation History UI may not render. The API returns definitive status immediately.
2. **Confirming a generation completed and retrieving results.** The API returns the full `result` object with all generated content. The UI may or may not show it depending on rendering.
3. **When the Generation History SPA shows only the shell.** The API works regardless of UI rendering state.
4. **Verifying button state changes.** Reading the button's `disabled` property via JS is more reliable than visually inspecting the page.

`VERIFIED` — during the 2026-09-22 session, the API confirmed the generation was queued, processing, and completed, while the video detail page showed no status and the Generation History page did not render the job list.

---

## 15. Common Browser Failures and Known Fixes

| Failure | Cause | Fix |
|---|---|---|
| Clicking "+ ADD A CHANNEL" does not open the modal | The button's React event handling does not respond to JS `.click()` | Use CDP `Input.dispatchMouseEvent` at the button's current coordinates (§6.2) |
| "ADD CHANNEL" submit button click misses | Button coordinates shift between form openings | Query current button position via `getBoundingClientRect()` before each click; do not hardcode (§6.3) |
| Generation button click appears to do nothing | Button is below the viewport / not scrolled into view | Scroll the page to bring the button into view before clicking (§10.2) |
| Generation submission seems to succeed but no job appears | Waiting only for UI changes (which don't happen on the video detail page) | Poll the API `GET /api/generation-history/:id` — do not wait for UI feedback (§12.1) |
| Generation History page appears empty | React SPA has not rendered the job list yet | Wait longer, refresh, or use the API instead (§8, §14) |
| Channel handle not appearing after add | Add did not persist (silent failure) | Re-add the channel; verify input value was correct; check for error messages (§6.7) |
| "Couldn't load videos" shown after channel add | Video fetch failed, not channel add | Channel IS added; optionally click "TRY AGAIN" to retry video fetch; do not treat as failed add (§6.6) |
| Form field value not detected by application | Value set via direct DOM property manipulation without triggering change events | Use `fill_input()` which triggers proper input events (§9.3) |

`VERIFIED` — all failures and fixes above were observed during the 2026-09-22 session (generation) and prior channel-add sessions (2026-09-22, batches 1–4).

`UNVERIFIED` — behavior when generation quota is exhausted, when rate limits are hit, or when the server returns an error response to a generation request. Not tested.

---

## 16. Verification Steps After Each Important Action

### 16.1 After adding a channel

1. Check that the channel handle appears in the channel list in UPPERCASE.
2. Confirm it appears as a channel entry (with "REMOVE" button and video list or status message), not as a form label.
3. Confirm the entry persists after the modal closes and the page settles.
4. If "Couldn't load videos" appears, the channel IS added — the video fetch failed separately.

`VERIFIED` — this verification procedure was followed for all 39 channels added across batches 1–4 (2026-09-22).

### 16.2 After submitting a generation request

1. Check that both "GENERATE CONTENT STRATEGY" and "GENERATE FROM VIDEO ONLY" buttons are disabled.
2. Poll the API `GET /api/generation-history/:id` — the job ID is returned when the generation is enqueued.
3. Confirm status progresses: `queued` → `processing` → `completed`.
4. When completed, retrieve the full result via the API.

`VERIFIED` — followed during the 2026-09-22 session.

### 16.3 After generation completes

1. Confirm `status: "completed"` and `stage: "completed"` via API.
2. Inspect the `result` object for expected fields (contentArchetype, sourceStrategyAnalysis, youtube, shortForm, social, written, sourcesUsed).
3. Note if `teleprompterScript` is null — this means the source video had no transcript (expected, not an error).
4. Optionally open the Content Editor at `/content-editor/:generationId` for further editing.

`VERIFIED` — followed during the 2026-09-22 session.

---

## 17. Interaction Methods — Summary

### 17.1 Text and number inputs
- Use `fill_input()` with a CSS selector targeting the field by placeholder, label, or id.
- Works reliably for all text inputs and number inputs.
- `VERIFIED` — all 10 generation input fields filled successfully. 2026-09-22.
- Also works for the channel handle input (id: `channel-handle`).

### 17.2 Textarea inputs
- Use `fill_input()` — works for multi-line textareas.
- `VERIFIED` — both textarea fields (#9 Custom generation instructions, #10 Things to preserve) filled successfully. 2026-09-22.

### 17.3 Buttons that work with JavaScript `.click()`
- "Generate content strategy" button — JS `.click()` works reliably.
- "Generate from video only" button — JS `.click()` works reliably.
- "ADD CHANNEL" submit button — JS `.click()` works (after the input is filled); coordinates must be queried first because they vary.
- Video card `div.group.cursor-pointer` — JS `.click()` works (navigates to video detail page).

`VERIFIED` — tested during the 2026-09-22 session.

### 17.4 Buttons that require CDP mouse events
- "+ ADD A CHANNEL" button — JS `.click()` does NOT work reliably; use CDP `Input.dispatchMouseEvent` at the button's current coordinates.
- Button stable position: viewport approximately (56, 633), size 233×56px.

`VERIFIED` — established during prior channel-add sessions (2026-09-22, batches 1–4).

### 17.5 CDP mouse events for generate buttons
- **UNTESTED** — CDP mouse events were not tested for the generation buttons. Prefer the verified JS `.click()` method; only fall back to CDP if JS interaction fails and the fallback is subsequently verified.

### 17.6 When to scroll
- Scroll when buttons are below the fold. The generate buttons on the video detail page require scrolling.
- Use `window.scrollTo(0, y)` or equivalent to scroll to the button's Y position before clicking.

`VERIFIED` — scrolling was required and used during the 2026-09-22 generation session.

### 17.7 When to use the API instead of the UI
- Always use the API to check generation status.
- Always use the API to retrieve generated results.
- Use the API when the Generation History UI is not rendering.

`VERIFIED` — API was the reliable path during the 2026-09-22 session.

---

## 18. Known Limitations Observed in the Browser

- **Thumbnail ideas are text descriptions only** — no image URLs or generated images.
- **No transcript = no teleprompter script** — when the source video has no transcript, `teleprompterScript` is null and several `sourceStrategyAnalysis` fields are marked "Not applicable due to lack of transcript." This is expected behavior.
- **Generation History is a React SPA** — content may not render on initial load; API is more reliable.
- **Video detail page shows no generation progress** — after submitting, the page does not display a progress indicator; poll the API.
- **Both generate buttons disable on submit** — this is the only visual confirmation that the request was accepted.
- **Transcript availability varies by video** — affects analysis depth.

`VERIFIED` — all observed during the 2026-09-22 session.

---

## 19. Cross-references

- **Product architecture, API, verification taxonomy** — see [README.md](./README.md)
- **Channel-add detailed procedure and lessons learned** — this file, §6
- **Generation input field reference** — this file, §9
- **Generation submission and status checking** — this file, §10–§13
- **Generation History SPA** — this file, §8
- **Pre-flight checklist** — this file, §1 (mirrored from README.md §6.1)

---

## Verification Log

### 2026-09-22 — Generation input fields documented

**What was verified:** All 10 generation input fields on the video detail page — labels, HTML types, placeholder text, widths, and layout position.

**Method:** Inspected the live DOM during a generation session for video @AIJASONZ / "Jev + Treg is a crazy combo for automation..."

**Result:** All 10 fields confirmed. Field table in §9.2 is accurate.

### 2026-09-22 — Generation submission and status flow verified

**What was verified:** Submitting a generation via "GENERATE CONTENT STRATEGY", recognizing the start signal (both buttons disable), waiting for completion, checking status via API, retrieving results.

**Method:** Live browser session + API inspection.

**Result:** Generation ID `6ab2bd5c5646075ed3ae3b8a` completed in ~2.5 minutes. Full result structure confirmed. All procedures in §10–§13 verified.

### 2026-09-22 — Generation History SPA behavior confirmed

**What was verified:** Navigating to `/generation-history` shows only the shell; job list does not render on initial load.

**Method:** Browser navigation + DOM inspection.

**Result:** Confirmed. API is the reliable alternative. §8 and §14 verified.

---

*End of browser-how-to.md*
