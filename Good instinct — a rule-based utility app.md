Good instinct — a rule-based utility app for elderly/low-literacy users has zero business looking like a generic AI-generated SaaS landing page. Here's a concrete design plan grounded in what this actually is (a trusted, official-feeling helper tool, not a startup product).

**Avoid these specific tells** (per the design brief for this category):
- Cream background + terracotta accent (#F4F1EA / #D97757) — the most obvious "AI made this" signature right now.
- Dark background + neon-green/vermilion accent.
- Rounded SaaS cards with identical soft grey shadows on everything.
- ALL-CAPS eyebrow labels, middle-dot meta text ("Step · 2 · of · 5"), arrows appended to buttons ("Continue →").

**Ground it in the actual subject: government forms, trust, low literacy**

**Color** — think official documents, not tech products:
- `#1B3A5C` — deep indigo-navy (used sparingly, for primary actions/headers — evokes govt letterhead, not decoration)
- `#F7F5F0` — warm paper white (not stark white, not cream-with-terracotta-tell)
- `#2D2D2D` — near-black text (not tinted #0B0B0B)
- `#C2410C` — a burnt marigold, used ONLY for warnings/guardrail alerts (Aadhaar warning etc.) — reserve it, don't decorate with it
- `#3F7D4F` — a muted green for "confirmed/done" states
- Borders: solid 1–2px, not soft box-shadows — this reads more like a printed form than a floating card.

**Type** — legibility over personality, since this serves low-reading-speed users:
- One typeface family that has full Devanagari + Telugu coverage: **Noto Sans** (or Noto Sans Devanagari/Telugu variants) — this isn't a stylistic choice so much as a functional requirement, and it also sidesteps the generic-Inter-everywhere look.
- Large base size (18–20px minimum for body, not 16px) — your actual users need this, not just aesthetics.
- No display/headline face at all — a second typeface adds nothing here and every extra visual flourish is one more thing competing with comprehension.

**Layout** — literal, not clever:
- Single column, full-width tap targets, generous spacing — mimics filling a paper form, which is deliberately familiar to the audience rather than "appy."
- Step numbering (1, 2, 3...) is actually justified here — it's a real sequence (field-by-field), not decoration.
- No card grid, no dashboard — one thing on screen at a time, like a physical form being filled one line at a time.

**Motion**
- Only one moment of motion: the field highlight when guidance moves to a new field (a soft box outline appearing around the matched region on the photographed form). No hover transitions, no fade-ins on scroll — this app doesn't scroll much and users won't hover.

**Copy tone**
- Plain, direct instructions in the user's own words, not system language: "Write your Aadhaar number here" not "Enter Aadhaar field value." No marketing language, no "Let's get started!" — trust here comes from sounding official and calm, not friendly-startup.

Want me to rewrite the build prompt from earlier with these tokens baked in, so the generated HTML comes out looking like this from the first pass?