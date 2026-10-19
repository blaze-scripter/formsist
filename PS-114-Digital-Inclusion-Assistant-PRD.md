# PRD: Digital Inclusion Assistant (PS-114)

## Problem Statement
First-time or low-confidence internet users struggle to complete online forms, digital payments, service portals, and account processes — often abandoning tasks, making irreversible errors, or oversharing sensitive data (Aadhaar, OTPs, bank details) to third parties for help. This excludes them from government and financial services they're entitled to and creates fraud exposure. An AI-guidance layer that sits on top of any website, walks users step by step, and actively blocks unsafe data-sharing directly addresses this gap.

## Goals
1. Reduce form abandonment rate on guided sessions by 50% vs. unassisted baseline.
2. Reduce user-reported "I didn't understand this field" incidents by 60%.
3. Achieve <2% incidence of sensitive-data exposure to unintended fields/screenshots during guided sessions.
4. 70% of users complete a full form flow (start to submit) without external human help.
5. Support 3 languages (Hindi, English, Telugu) with voice guidance at parity of feature coverage.
6. Reduce missed-deadline application failures (expired forms, lapsed renewals) among tracked applications to near zero.

## Non-Goals
- Not building a form-filling autopilot that submits on the user's behalf without explicit confirmation at each step (liability + trust risk).
- Not storing or transmitting documents to any backend server in v1 — local-only profile store (avoids becoming a data-breach target, which defeats the "prevent oversharing" goal).
- Not scraping/altering arbitrary third-party site logic beyond overlay guidance — no auto-fill injection into fields that could break site validation (v1 restricts to visual guidance + copy-paste assist, not DOM injection, unless the target site is on an allow-list).
- Not building a full document storage/locker system — only enough profile fields to guide, not host scanned IDs.
- Not covering payment execution (OTP entry, UPI PIN) — assistant guides up to that point, then explicitly steps back for security-sensitive fields.
- Not requesting standing camera/`getUserMedia` permission or running a live in-app camera preview — image capture goes through the OS-native single-shot picker only, to limit permission scope and trust risk.

## Personas
- **Primary user**: First-time/low-confidence internet user (elderly, rural, first-generation smartphone/PC user), low digital literacy, may be more comfortable in Hindi/Telugu, may have low reading speed → benefits from voice.
- **Secondary user**: A family member/community volunteer who sets up the profile once for the primary user.
- **Tertiary (future)**: Government/NGO field agent using this as an assisted-service tool at a kiosk.

## User Stories
- As a first-time user, I want the assistant to read out each form field in my language so I understand what's being asked before I answer.
- As a first-time user, I want to load any government/service website inside the assistant so I don't need to leave the guided environment.
- As a first-time user, I want my name/address/basic details to auto-suggest from a local profile so I don't retype them every time.
- As a first-time user, I want to be warned before I paste/type something that looks like an OTP, PIN, or Aadhaar number into a field that shouldn't need it.
- As a first-time user, I want to upload a photo of a document and have it auto-compressed so the upload doesn't fail.
- As a first-time user, I want to be told which document I'm missing and how to apply for it, if a form requires something I don't have.
- As a first-time user, I want to switch between Hindi, English, and Telugu at any point without losing my progress.
- As a volunteer/family member, I want to set up the profile once so the elder in my family can reuse it independently later.
- As a first-time user filling a paper form, I want to photograph or upload an image of the physical form so the assistant can guide me on what to write in each section, field by field.
- As a first-time user, I want to choose whether to snap a photo, pick from gallery, or upload a file — without granting the app full/ongoing camera access.
- As a first-time user, I want to be notified before a submitted application's deadline (payment due, document resubmission, renewal) expires so I don't lose my progress or eligibility.
- As a first-time user, I want to see all my in-progress and past applications in one place with their current status and any upcoming deadline.

## System Architecture (high level)
- **Shell app**: Web app (or PWA) hosting an `<iframe>` that loads the target third-party site.
- **Overlay layer**: A floating assistant panel/highlight-overlay rendered above the iframe (same-origin proxy or browser-extension-style content script — see Open Questions on iframe cross-origin limits).
- **Local profile store**: Browser local storage / IndexedDB, encrypted at rest, never synced to a server by default.
- **Guidance engine**: Rule-based step sequences per known site/form template (v1), with an LLM fallback to generate generic step-by-step guidance for unrecognized sites by reading visible DOM text/labels.
- **Sensitive-field detector**: Pattern-matching (regex + heuristics) library flagging Aadhaar/PAN/OTP/bank-account-like inputs, cross-referenced against field context.
- **Voice module**: TTS for guidance output; STT for hands-free navigation commands ("next", "read again", "go back").
- **Image compressor**: Client-side compression (canvas-based resize/re-encode) before any upload.
- **Physical form vision module**: OCR + layout detection on a photographed/uploaded paper form image, mapped to a guided field-by-field walkthrough (what to write, in what script/language) without requiring the user to type into a digital replica.
- **Application tracker & notification engine**: Local record per application (site/form name, submission date, known deadline/expiry, current status) with a local scheduler (browser notifications / OS push if installed as PWA) that fires reminders before expiry.

## Requirements

### Must-Have (P0)
1. **Iframe site loader** — Load an arbitrary URL inside the app shell with a persistent assistant panel alongside it.
   - AC: Given a valid URL, when loaded, then the site renders inside the iframe and the assistant panel remains visible and interactive.
   - AC: Given a site blocks iframe embedding (X-Frame-Options/CSP), then the assistant shows a clear fallback message and alternative (e.g., open in new tab with guidance sidebar).
2. **Local user profile for autofill** — Name, DOB, address, phone, email, and other reusable non-sensitive fields stored locally.
   - AC: Given a profile field exists locally, when the user reaches a matching form field, then the assistant offers a one-tap suggestion (not auto-injected without confirmation).
   - AC: Given no network connection, then profile read/write still works.
3. **Step-by-step tutorial per form field** — Highlight the active field, explain what's needed in plain language, in the selected language.
   - AC: Given the user is on a field the assistant recognizes, then a plain-language explanation is shown/read aloud before input.
4. **Sensitive-info guardrail** — Detect and warn before an OTP/PIN/full Aadhaar/bank-account number is entered into a field outside a verified secure context.
   - AC: Given text matching a sensitive pattern is about to be submitted/pasted, then the assistant interrupts with a warning requiring explicit user confirmation to proceed.
   - AC: Given the field is the site's own verified OTP field, then no false-positive warning is shown (context-aware).
5. **Built-in image compressor** — Automatically compress photos before upload to meet common size/format limits.
   - AC: Given a user selects a photo for upload, when file size exceeds a threshold, then it is compressed client-side and the user sees before/after size.
6. **Multilingual UI + guidance (Hindi, English, Telugu)** — All assistant copy and voice guidance available in all 3 languages, switchable mid-session.
   - AC: Given a language is selected, then all assistant panel text, tutorials, and voice output use that language without losing form progress.
7. **Voice guidance (TTS)** — Read out instructions and field explanations aloud.
   - AC: Given voice mode is on, then each new step is read aloud automatically; user can replay or mute.
8. **Physical form guidance via image (camera capture, gallery, file upload)** — User can bring in a photo of a paper form through three input paths, and the assistant reads it and guides them field by field on what to write and where.
   - AC: Given the user taps "capture," then the OS-native camera capture UI opens for a single photo only (via `<input type="file" capture>` / native file picker) — the app never requests standing `getUserMedia` camera permission or keeps a live camera feed running in-app.
   - AC: Given the user taps "gallery" or "files," then the OS-native picker opens and only the selected image is handed to the app — no access to the rest of the photo library/filesystem.
   - AC: Given an image is provided by any of the three paths, then it is run through OCR/layout detection and the assistant produces a step-by-step "write this here" walkthrough (read aloud + on-screen highlight on the image) in the selected language.
   - AC: Given the OCR confidence is low (blurry photo, handwriting-heavy form), then the assistant asks the user to retake/reselect rather than guessing.
   - AC: Given the form language differs from the user's selected language, then the assistant translates field labels for explanation purposes only — it does not alter what the user physically writes on the paper form.
9. **Application tracking & expiry notifications** — Track every application the user starts/submits through the assistant and alert them before it expires.
   - AC: Given a user completes or submits a form flow (digital or physical), then it is logged locally with a name, date, status (draft/submitted/needs action), and deadline if known.
   - AC: Given an application has a known expiry/renewal/payment-due date, then the assistant sends a local reminder notification at configurable intervals before that date (e.g., 7 days, 1 day).
   - AC: Given the user opens the app, then a dashboard lists all tracked applications sorted by nearest deadline first, with status clearly visible.
   - AC: Given no deadline is known for a form, then the assistant prompts the user (or volunteer) to enter one manually, or flags it as "no deadline set."
   - AC: Given the device is offline when a reminder is due, then the notification still fires using locally scheduled OS-level notifications (no server round-trip required).

### Nice-to-Have (P1)
1. **Voice input (STT) for navigation** — "next step," "repeat," "read this field" as spoken commands.
2. **Missing-document suggestion flow** — If a required field/document is unavailable, show what govt document is needed and link/steps to apply for it (e.g., no PAN → steps to apply for PAN).
3. **Progress save/resume** — Save form progress locally so a user can leave and return.
4. **Site-template library** — Pre-built guided flows for high-traffic portals (e.g., common govt service sites) vs. generic LLM-generated guidance for unknown sites.
5. **Accessibility mode** — Larger text, high-contrast theme, screen-reader compatibility.

### Future Considerations (P2)
1. Assisted-service kiosk mode for NGO/field agents with multi-user profile switching.
2. Expand language set beyond Hindi/English/Telugu.
3. Community-sourced site templates (crowdsourced guided flows, moderated).
4. Family/caregiver remote-assist mode (screen-share style co-browsing with consent).

## Success Metrics
**Leading (days–weeks post-launch)**
- Form completion rate per guided session (target: 70%+)
- Sensitive-data warning trigger rate & user override rate (track for tuning false positives, target <10% override)
- Avg. time-to-complete vs. unassisted baseline
- Language distribution of sessions (validates multilingual investment)

**Lagging (weeks–months)**
- Repeat usage rate (return without a helper) — target 40% of users complete a second form independently within 30 days
- % of tracked applications with a reminder sent before deadline vs. missed silently — target 100% coverage for applications with a known deadline
- Reported fraud/data-leak incidents — target 0
- User-reported confidence increase (survey-based)

## Open Questions
- **Engineering**: Can we reliably overlay guidance on arbitrary third-party sites given iframe sandboxing and X-Frame-Options restrictions? Do we need a companion browser extension instead of a pure iframe for full DOM access? (Blocking)
- **Legal/Compliance**: What are the data-localization/consent requirements for even locally-stored PII under India's DPDP Act? (Blocking before profile feature ships)
- **Design**: How do we visually differentiate "our overlay" from "the actual government site" to avoid look-alike phishing concerns? (Blocking)
- **Data**: What's the actual list of top N government/service portals to prioritize for the site-template library? (Non-blocking, needed for P1)
- **Engineering**: STT accuracy for Telugu/Hindi in noisy/low-end-device environments — is an on-device model feasible, or do we need a cloud STT (which conflicts with the local-only privacy goal)? (Blocking for P1 voice input)
- **Engineering**: Can OCR run on-device (to keep photographed forms local/private) with acceptable accuracy on handwritten/regional-language forms, or does it require a cloud OCR call — and if cloud, what's retained/logged on that vendor's side? (Blocking before physical-form feature ships)
- **Product**: Most govt/service portals don't expose a machine-readable deadline — how much of "expiry known" relies on OCR-detected dates (e.g., from a confirmation receipt/screen) vs. the user/volunteer manually entering it? Sets expectations for how much of the tracker is truly automatic. (Non-blocking, needed before P0 sign-off)

## Timeline Considerations
- **Phase 1 (v1, P0 only)**: Iframe shell, local profile, static tutorial overlays for 2–3 pilot govt portals, sensitive-field guardrail, image compressor, 3-language UI + TTS. Target: MVP demo-ready.
- **Phase 2 (P1)**: STT voice input, missing-document guidance, progress save/resume, expand site template library.
- **Phase 3 (P2)**: Kiosk mode, caregiver co-browsing, broader language support.
- Dependency: sensitive-field detector accuracy directly gates public pilot launch — cannot ship P0 without it passing a false-negative review.
