# TRD: Digital Inclusion Assistant (PS-114)
Companion to PRD: PS-114-Digital-Inclusion-Assistant-PRD.md

## 1. Scope & Constraints
- All processing is **on-device**. No AI-as-a-service / cloud LLM / cloud OCR / cloud STT calls in v1 — this is a hard constraint, not a preference, to keep PII (Aadhaar, forms, profile data) off any third-party server.
- No standing camera permission (`getUserMedia`) — image capture only via OS-native single-shot picker (`<input type="file" capture>` / native share-sheet equivalent on mobile shells).
- Target platform: Progressive Web App (PWA) first, wrapped in a thin native shell (Capacitor/Cordova-style) later if app-store distribution or deeper OS notification/file-picker access is needed.
- Target devices: low-to-mid-end Android phones (majority of primary persona), Chrome/Edge desktop as secondary.
- Offline-first: profile, tracker, and guidance must work with zero network connectivity. Iframe site-loading obviously requires network, but nothing else should depend on it.

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      Shell App (PWA)                     │
│                                                            │
│  ┌───────────────┐   ┌───────────────────────────────┐   │
│  │  Assistant     │   │   Content Frame                │   │
│  │  Panel (UI)    │   │  ┌───────────────────────────┐ │   │
│  │  - highlights  │◄──┼──┤ <iframe> 3rd-party site    │ │   │
│  │  - voice ctrl  │   │  │   (Mode A: web form)       │ │   │
│  │  - warnings    │   │  └───────────────────────────┘ │   │
│  │  - tutorial    │   │  ┌───────────────────────────┐ │   │
│  │                │   │  │ Captured image viewer      │ │   │
│  │                │   │  │   (Mode B: physical form)  │ │   │
│  └───────┬────────┘   │  └───────────────────────────┘ │   │
│          │            └───────────────────────────────┘   │
│  ┌───────▼─────────────────── Form Agent (core) ───────┐  │
│  │  perceive() → decide() → act()                       │  │
│  └───────┬───────────────┬───────────────┬─────────────┘  │
│          │               │               │                 │
│  ┌───────▼─────┐ ┌───────▼──────┐ ┌──────▼───────────┐    │
│  │ Knowledge   │ │ Guardrail    │ │ OCR Module        │    │
│  │ Base (JSON) │ │ (regex rules)│ │ (Tesseract.js WASM)│    │
│  └─────────────┘ └──────────────┘ └───────────────────┘    │
│          │               │               │                 │
│  ┌───────▼───────────────▼───────────────▼─────────────┐  │
│  │      Local Data Layer (IndexedDB, encrypted)          │  │
│  │  - profile store   - tracker store   - templates      │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────┐  │
│  │ TTS/STT     │  │ Image      │  │ Notification        │  │
│  │ (Web Speech)│  │ Compressor │  │ Scheduler            │  │
│  │             │  │ (Canvas)   │  │ (Notifications API/  │  │
│  │             │  │            │  │  Service Worker)     │  │
│  └────────────┘  └────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

## 3. Core Module: Form Agent

### 3.1 Responsibility
Deterministic orchestrator. No ML inference. Coordinates perception (OCR text / DOM field / user tap), decision (rule table + guardrail check), and action (speak / highlight / warn / log) across both Web-Form Mode and Physical-Form Mode.

### 3.2 Interfaces
```ts
interface FieldState {
  id: string;
  labelVariants: string[];       // e.g. ["Aadhaar No.", "ఆధార్ సంఖ్య", "आधार संख्या"]
  type: "sensitive" | "profile" | "free_text" | "document_upload" | "date";
  guidanceKey: string;           // lookup key into KnowledgeBase
  validation?: string;           // regex string, optional
  next?: string;                 // id of next field, forms a linked list/graph
}

interface Decision {
  text: string;                  // guidance text in current language
  blocking: boolean;             // true => show modal, require confirm
  fieldId: string;
  suggestion?: string;           // profile autofill suggestion, if applicable
}

class FormAgent {
  perceive(input: OCRBlock | DOMFieldEvent | VoiceCommand): FieldState;
  decide(field: FieldState, context: SessionContext): Decision;
  act(decision: Decision): void;
  step(input): FieldState["next"];
}
```

### 3.3 Matching strategy (perceive)
- Web-Form Mode: input source depends on active tier — Tier 1 (native Android) supplies accessibility-tree nodes; Tier 2 (default/cross-platform) supplies OCR text blocks from a screenshot; Tier 0 (allow-listed sites) supplies same-origin DOM field events. See §7 for the full tier breakdown.
- Physical-Form Mode: OCR output blocks (text + bounding box) matched against `labelVariants` via fuzzy string match (Levenshtein distance threshold, e.g. ≤2 for short labels) — not embeddings, not ML classification.
- Unmatched field → generic fallback guidance ("Fill this as shown on the form; ask a trusted person if unsure") rather than guessing.

### 3.4 Decision table (decide)
- `type: sensitive` → route to Guardrail module first; only proceed to guidance text if guardrail clears or user confirms override.
- `type: profile` → look up matching key in local Profile Store, attach as `suggestion`.
- `type: free_text` / `date` → pull explanation string from Knowledge Base by `guidanceKey` + current language.
- `type: document_upload` → route to Image Compressor + Missing-Document flow (P1).

### 3.5 State persistence
- Session state (current field index, form id, language) persisted to IndexedDB every step → enables "Progress save/resume" (P1) with no extra engineering later since it's built into the agent loop from day one.

## 4. Knowledge Base (Guidance Content)

### 4.1 Format
Static JSON, bundled with app + updatable via versioned content packs (no server dependency required to function, but allows content updates without app store re-release if hosted as static JSON on a CDN — read-only fetch, not AI inference).

```json
{
  "aadhaar_explain": {
    "en": "This is your 12-digit Aadhaar number, found on your Aadhaar card.",
    "hi": "यह आपकी 12 अंकों की आधार संख्या है, जो आपके आधार कार्ड पर मिलती है।",
    "te": "ఇది మీ 12 అంకెల ఆధార్ నంబర్, ఇది మీ ఆధార్ కార్డ్‌పై ఉంటుంది."
  }
}
```

### 4.2 Site/Form Templates (P0: 2–3 pilot templates)
```json
{
  "template_id": "epfo_uan_activation",
  "match": { "url_pattern": "*.epfindia.gov.in/*" },
  "fields": [ /* array of FieldState */ ]
}
```
- Template authoring is manual for v1 pilot portals; template library growth is a P1/ops task, not an engineering blocker.

## 5. Sensitive-Field Guardrail

### 5.1 Detection rules (regex + heuristics, all local)
| Pattern | Regex (illustrative) | Action |
|---|---|---|
| Aadhaar (12 digit) | `^\d{4}\s?\d{4}\s?\d{4}$` | Block + confirm modal |
| PAN | `^[A-Z]{5}\d{4}[A-Z]$` | Warn + confirm modal |
| OTP (4–6 digit, short-lived context) | `^\d{4,6}$` + field-context check | Block unless field is flagged `trusted_otp_field` in template |
| Bank account (8–18 digit numeric, near "account" label) | contextual, not pure regex | Warn + confirm modal |

### 5.2 Context-awareness
- Each site template can mark specific fields `trusted_otp_field: true` (verified first-party OTP box) to suppress false positives, per PRD AC.
- Guardrail always logs (locally only) trigger + user decision (proceed/cancel) for the false-positive tuning metric in the PRD — never logs the actual sensitive value itself, only field-type + outcome.

### 5.3 Failure mode
- Default-deny posture: if guardrail is uncertain, it warns rather than stays silent (false positives acceptable, false negatives are not).

## 6. OCR / Physical Form Module

### 6.1 Engine
- Tesseract.js (WASM), running in a Web Worker to avoid blocking UI.
- Language packs: `eng`, `hin`, `tel` bundled or lazy-loaded on first use per selected language.

### 6.2 Pipeline
1. Image intake (capture/gallery/file) → passed to Image Compressor first (resize to OCR-friendly resolution, not over-compressed to the point of losing text legibility — separate compression profile for "upload to portal" vs. "OCR read").
2. Deskew/contrast normalization (basic canvas image processing).
3. Tesseract OCR pass → text blocks with bounding boxes + confidence scores.
4. Confidence check: if avg. confidence < threshold (e.g. 60%), prompt retake instead of guessing (per PRD AC).
5. Text blocks → Form Agent `perceive()`.

### 6.3 Layout mapping
- Bounding boxes let the UI draw an on-screen highlight over the actual photographed form region being explained (visual + audio guidance together).

## 7. Web-Form Overlay Module

### 7.1 Known constraint
- Cross-origin iframes (most real govt/bank sites) block parent-frame DOM access (same-origin policy) and many set `X-Frame-Options: DENY` / `frame-ancestors` CSP, preventing embedding entirely. This is the PRD's top blocking open question, and it rules out a pure-iframe approach for most real target sites.

### 7.2 Tiered approach

**Tier 1 (Android native app, preferred) — Screen Overlay + Accessibility Service**
- `SYSTEM_ALERT_WINDOW` permission draws the assistant panel/floating bubble on top of any app, including the regular mobile browser with the real site loaded — no embedding, so X-Frame-Options/CSP is irrelevant.
- `AccessibilityService` reads the visible view hierarchy/text of the foreground browser to identify field labels/positions, feeding `FormAgent.perceive()` directly (more reliable than OCR, no image processing needed).
- Requires a native Android app (PWA cannot request these permissions) — ship as a thin native wrapper (Capacitor/Cordova) hosting the same web-based UI/agent code, plus a native plugin bridging overlay + accessibility events into the JS layer.
- **Platform limit**: Android-only. iOS does not permit third-party overlay/accessibility access at this level — iOS users fall to Tier 2.
- **Trust/privacy handling**: accessibility service access is powerful and sensitive (Play Store review scrutiny is high, and misuse risk is real). Mitigations required before launch:
  - Overlay/accessibility reading active **only** while the user has explicitly started a guided session (visible on-screen indicator whenever active).
  - Never log, store, or transmit read screen content — it is consumed in-memory by `perceive()` and discarded per step.
  - Explicit, plain-language consent screen explaining what is read and why, before first use.
  - Play Store accessibility-permission declaration must map 1:1 to this guided-form-filling use case (no scope creep risk during review).

**Tier 2 (cross-platform fallback, default for PWA/iOS/desktop) — Screenshot + OCR**
- Target site opens in a normal tab/window; assistant panel runs alongside as a screenshot-and-tap guide.
- User-triggered or app-captured visible-area screenshot (with permission) is OCR'd via the same pipeline as Physical-Form Mode (§6), feeding `FormAgent.perceive()` with text blocks instead of accessibility-tree nodes.
- Works everywhere (any OS/browser) but is slower per-step and less precise than Tier 1 (OCR confidence limits, manual screenshot trigger friction).

**Tier 0 (opportunistic, allow-listed sites only)**
- If specific partner/govt sites permit iframe embedding or provide an API/partnership integration, use direct DOM-aware overlay (original iframe plan) for those sites only — best experience where available, but not something to depend on broadly.

### 7.3 Perception-source abstraction
`FormAgent.perceive()` takes any of three input shapes — accessibility-tree node (Tier 1), OCR text block (Tier 2/Physical-Form), or DOM field event (Tier 0) — and normalizes them to the same `FieldState` before `decide()`/`act()` run. This keeps the agent core identical across all three tiers; only the perception adapter differs.

### 7.4 Rollout sequencing
- Ship Tier 2 first (cross-platform, no native app, fastest to pilot).
- Build Tier 1 as the native Android app once Tier 2 validates the guidance/UX flow — treat it as a v2 platform investment given Play Store review lead time and native-plugin engineering effort.
- Tier 0 is opportunistic, pursued in parallel via partnerships, not a P0 engineering dependency.

## 8. Local Data Layer

### 8.1 Storage
- IndexedDB for structured data (profile, tracker, session state, templates cache).
- Encryption: Web Crypto API (AES-GCM), key derived from a user-set local PIN/passphrase (or device-level secure storage if wrapped in native shell) — never transmitted, never recoverable remotely (trade-off: lost PIN = lost local data, must be communicated clearly to user/volunteer during setup).

### 8.2 Schemas

**Profile Store**
```ts
interface UserProfile {
  id: string;
  fields: { key: string; value: string }[]; // name, dob, address, phone, email — NOT sensitive IDs
  preferredLanguage: "en" | "hi" | "te";
  createdBy: "self" | "volunteer";
}
```

**Application Tracker**
```ts
interface TrackedApplication {
  id: string;
  formName: string;
  mode: "web" | "physical";
  startedAt: string;       // ISO date
  submittedAt?: string;
  status: "draft" | "submitted" | "needs_action" | "completed";
  deadline?: string;        // ISO date, nullable
  deadlineSource: "ocr_detected" | "manual" | "unknown";
  reminderIntervals: number[]; // days before deadline, e.g. [7,1]
}
```

### 8.3 No sync by default
- No backend account system in v1 (matches PRD non-goal). Optional local backup/export (encrypted file the user can save to their own Drive/SD card) is a P1/P2 consideration, not v1.

## 9. Voice Module (TTS/STT)

- **TTS**: Web Speech API `SpeechSynthesis`. Language/voice availability for Hindi/Telugu varies by device OS — must runtime-check `speechSynthesis.getVoices()` and gracefully degrade to text-only + a "voice not available in this language on your device" notice if unsupported.
- **STT (P1)**: Web Speech API `SpeechRecognition` where supported (Chrome-based); fallback to on-screen buttons ("Next", "Repeat", "Back") on unsupported browsers/devices — STT is an enhancement, never the only way to navigate.

## 10. Image Compressor

- Canvas-based resize + re-encode (JPEG/WebP) client-side, before any upload to a third-party site.
- Two profiles: **Upload profile** (aggressive compression to meet common portal size limits, e.g. <200KB) vs. **OCR-read profile** (preserve resolution/contrast for text legibility, used internally only, never uploaded anywhere).

## 11. Notification / Tracker Scheduler

- PWA installed → use Service Worker + Notifications API + `periodicSync` (where supported) or a foreground check-on-open fallback for browsers without background sync support (notable Android/iOS PWA limitations — flag as platform risk).
- On each app open, scheduler also does a catch-up check against all `TrackedApplication.deadline` values in case background sync didn't fire (defensive, since background reliability varies by OS/browser).
- All scheduling logic is local; no server-side push/cron dependency in v1.

## 12. Non-Functional Requirements
- **Performance**: OCR pass on a mid-range Android device should complete in <8s per image; app shell should be usable on 2GB RAM devices.
- **Offline**: Profile, tracker, guidance content, and physical-form OCR must all function fully offline (only iframe web-form loading needs network).
- **Privacy**: No PII leaves the device in v1 — verified via a data-flow audit before pilot launch (ties to PRD legal/compliance open question).
- **Accessibility**: Text scaling to at least 200%, color-contrast ratio ≥ 4.5:1, all interactive elements reachable without precise fine-motor input (large tap targets, per persona's likely device/skill level).
- **Localization**: All static strings externalized to translation files from day one, not hardcoded — Hindi/Telugu additions to Knowledge Base content follow the same pattern.

## 13. Testing Strategy
- **Guardrail**: regression test suite of known sensitive-pattern strings (positive) and known safe strings that resemble them (negative), tracking false-negative rate at zero tolerance before pilot sign-off.
- **OCR accuracy**: benchmark set of real (anonymized/synthetic) sample forms in all 3 languages, tracking field-recognition rate and confidence-threshold tuning.
- **Offline mode**: full test pass with network disabled except iframe-loading screens.
- **Device matrix**: test on at least 2 low-end Android devices + 1 mid-range + desktop Chrome, given persona skews toward older/cheaper hardware.

## 14. Open Technical Decisions (mirrors PRD Open Questions, engineering detail added)
1. Timing of native Android app (Tier 1 overlay+accessibility) investment vs. staying PWA-only longer on Tier 2 — depends on pilot feedback on Tier 2 speed/precision (§7).
2. Play Store review risk/timeline for the accessibility-service permission — needs early legal/compliance + store-policy review, not left to submission time.
3. On-device OCR language-pack size/load-time trade-off for Hindi/Telugu on low-end devices — needs a device benchmark spike.
4. PIN/passphrase recovery UX for encrypted local storage — no data recovery by design; needs a clear onboarding warning flow.
5. Background notification reliability across Android/iOS PWA constraints — may require native-shell wrapping earlier than planned if push reliability is unacceptable (ties into the Tier 1 native-app decision above).

## 15. Phase Mapping (engineering sequencing, mirrors PRD phases)
- **Phase 1**: Form Agent core loop, Knowledge Base (2–3 templates), Guardrail, Profile Store, Image Compressor (upload profile), TTS, IndexedDB + encryption, Tracker (manual deadline entry only), Tier 2 (screenshot+OCR) web-form guidance.
- **Phase 2**: OCR module hardening + Physical-Form Mode, STT, OCR-detected deadlines, background notification scheduling.
- **Phase 3**: Native Android app for Tier 1 (screen overlay + accessibility service), kiosk/multi-profile mode, encrypted local export/backup, Tier 0 partner integrations.
