---
name: good-designer
description: Enforce intentional, human-crafted UI/UX design and eliminate generic AI aesthetic patterns. Use whenever designing, styling, building, reviewing, or refactoring user interfaces, web applications, components, layouts, or dashboards.
---

# Good Designer: Anti-AI UI / UX Design

Design and build interfaces that feel intentionally crafted by a seasoned product designer, not spit out by generic AI templates.

```
Clear → Functional → Consistent → Intentional → Polished
NOT: Fancy → Decorative → Trendy → Animated → AI-looking
```

Every visual decision must serve the product, its content, and its users.

---

## Operating Modes

### 1. Build Mode (Default)
When generating or implementing UI:
* Enforce all 7 Pillars silently and authoritatively in code and markup.
* Establish semantic design tokens and strict spatial rhythm before writing layout code.
* Do not clutter conversation with aesthetic excuses; deliver calm, functional, production-ready interfaces.

### 2. Audit Mode
When asked to review, critique, or refactor an existing interface:
* Inspect the interface against **Pillar 2 (The Anti-AI Fingerprint)**.
* Apply **Gate 2 (The "Remove 20%" Test)** to identify decorative bloat, redundant cards, and unnecessary containers.
* Provide an actionable, prioritized refactoring plan that strips away slop and restores visual hierarchy.

---

## The 7 Core Pillars

### Pillar 1: The Prime Directive (Intentionality & Whitespace)
Before introducing any visual element, border, container, or styling rule, verify:
> *Does this element directly improve usability, hierarchy, comprehension, navigation, or feedback?*

If the answer is no: **Do not add it.**
* **Whitespace is deliberate design:** An unfilled area is not an invitation to add clutter. Never add decorative shapes, fake metrics, or filler widgets to solve perceived "emptiness."
* **Eliminate decorative containers:** Do not invent sections or UI elements solely to make a layout appear complex or sophisticated.

---

### Pillar 2: The Anti-AI Fingerprint (Banned Aesthetic Slop)
AI models gravitate toward overused defaults found in generic templates. Reject these defaults outright:

1. **Zero Gratuitous Gradients:**
   * Use solid colors by default.
   * Ban purple-to-blue/cyan neon gradients, gradient text, gradient borders, radiant backgrounds, and blurred gradient mesh blobs.
   * Gradients are permitted only when an established brand identity explicitly requires them.
2. **Zero AI Symbolism & Buzzwords:**
   * Never decorate interfaces with sparkles, magic wands, brain icons, glowing neon borders, or robot graphics.
   * Never label features with empty buzzwords like "Smart", "Intelligent", "Magic", or "AI-Powered" unless describing explicit technical functionality with accurate product copy.
3. **Zero Card-Nesting Addiction:**
   * Stop wrapping every paragraph, control, and list item in a bordered card.
   * Avoid "card inside card inside card" layouts.
   * Ban universal giant rounded corners (`32px` on everything) and ubiquitous giant pill buttons.
4. **Zero Faux-Depth & Glassmorphism:**
   * Avoid blurred translucent panels (`backdrop-filter`), frosted glass navigation, and transparent glowing panels.
   * Avoid giant floating drop shadows and colored glowing halos.
   * Achieve elevation through clean surface contrast, crisp borders, and deliberate spacing.
5. **Zero Invented Badges:**
   * Do not decorate elements with generic tags ("Pro", "Premium", "Popular", "Featured", "Trending") unless backed by real system status.

---

### Pillar 3: Layout & Information Architecture
Content dictates layout—never force content into predetermined AI templates.

* **Ban the Cookie-Cutter SaaS Layout:** Reject the default sequence of `[Centered Hero → Logo Banner → 3 Feature Cards → Testimonials → Pricing Cards → Giant Footer]` unless user research proves it fits the product's specific problem.
* **Purposeful Asymmetry:** Do not force identical heights, symmetrical card widths, or 3-column grids when the content lengths naturally differ. Let hierarchy and scanning behavior guide proportions.
* **Dashboards for Operational Decisions:** Dashboards must answer critical operational questions. Never default to four generic KPI cards, a meaningless area chart, and a "Welcome back" banner. Select widgets based on actionable workflows.
* **Data Belongs in Tables:** Dense, structured data belongs in aligned, scannable, sortable tables with clear column headers—not in repetitive grids of floating cards.

---

### Pillar 4: Design System Foundations

#### 1. Color
* Build on a restrained, semantic token architecture:
  * `canvas` (base background)
  * `surface` (elevated panels/containers)
  * `text-primary` & `text-muted` (high-contrast content)
  * `border` (subtle structural separation)
  * `primary` (dominant brand/action color)
  * `destructive` & `success` (status communication)
* Never introduce arbitrary saturated colors or neon accents just to fill a visual void.

#### 2. Typography
* Limit to 1 primary typeface, optionally paired with 1 complementary typeface.
* Restrict to 3–4 font weights and a consistent mathematical type scale.
* Never use oversized display headings that push key interactive content below the fold.
* **Script Sensitivity:** Tailor typographic properties to the script language. For scripts with high ascenders or deep descenders (e.g. Arabic, Indic), expand line-height (e.g., 1.6–1.8) to prevent diacritic clipping and maintain effortless scanning.

#### 3. Spacing & Rhythm
* Anchor spacing to a strict mathematical scale (e.g., 4px / 8px grid).
* Use spacing—not borders or colored boxes—as the primary mechanism to communicate relationship:
  * Keep related elements close (tight internal padding).
  * Give distinct sections generous external margins.
* Never use arbitrary margins, unpredictable gaps, or magic pixel numbers.

#### 4. Geometry & Elevation
* Use a consistent, proportional radius scale:
  * Small controls / inputs: small radius (e.g., 4px–6px)
  * Cards / panels: moderate radius (e.g., 8px–12px)
  * Modals / dialogs: structured radius (e.g., 12px–16px)
  * Pills / circles: strictly for avatars, status dots, or actual pill tags.
* Prioritize flat hierarchy with 1px border contrast over exaggerated floating drop shadows.

---

### Pillar 5: Interaction, States & Motion

* **Subtle, Purposeful Hover States:**
  * Hover feedback must be subtle (e.g., slight background tint, border tone shift, or text color change).
  * Never make elements violently jump, lift, or enlarge on hover (no universal card-scaling transforms).
* **Restrained Motion:**
  * Motion exists to communicate state transitions and spatial continuity, not entertainment.
  * Keep transitions fast and crisp (<200ms).
  * Respect `prefers-reduced-motion` at all times.
  * Ban continuous looping animations, ambient glowing pulses, and universal scroll fade-in effects.
* **Complete State Design:**
  * **Loading States:** Use targeted skeletons or inline indicators; never animate entire screens gratuitously.
  * **Empty States:** Clearly communicate: (1) What is empty, (2) Why it is empty, and (3) The specific action to populate it. Avoid decorative illustrations.
  * **Error States:** Provide inline, descriptive error messages explaining what went wrong and how to correct it.

---

### Pillar 6: Honest Copy & Authentic Data

* **Action-Oriented Verbs:**
  * Button labels must clearly state what happens on click (`Save changes`, `Create project`, `Download CSV`, `Delete workspace`).
  * Ban vague or lazy labels (`Submit`, `Proceed`, `Get Started`, `Click Here`).
* **Zero Marketing Hype:**
  * Eliminate empty phrases: *"Supercharge your workflow"*, *"Unlock your potential"*, *"The future of productivity"*, *"Seamless next-generation platform"*.
  * Write clear, objective prose describing what the system does.
* **Zero Fabricated Social Proof:**
  * Never invent fake customer counts ("Trusted by 10,000+ teams"), fake user reviews, star ratings, or imaginary company logos.
  * If real proof does not exist, omit the section entirely and let the product speak for itself.

---

### Pillar 7: The 3 Audit Gates (Verification)

Before declaring any UI finished, run it through all three gates:

#### Gate 1: The "Could This Be Any Product?" Test
> *If the logo is removed, does this interface look like a generic SaaS template?*
* If yes, the design lacks domain specificity. Re-evaluate the information architecture, vocabulary, and primary workflows to reflect the specific problem domain. Do not solve this with decorative art.

#### Gate 2: The "Remove 20%" Test
> *Can ~20% of the visual elements (borders, background tints, extra cards, decorative badges) be removed without hurting usability?*
* If usability is unaffected or improved, delete those elements permanently.

#### Gate 3: The "Squint & Keyboard/WCAG" Test
1. **The Squint Test:** Squint at the screen so text becomes unreadable.
   * Can you still identify the primary action?
   * Is the most important section immediately distinct from secondary information?
   * If everything blends into a uniform gray or competes equally, fix visual hierarchy.
2. **The Keyboard & WCAG Test:**
   * Is full navigation possible using only the `Tab` and arrow keys?
   * Are focus rings crisp and visible?
   * Does body text meet WCAG 2.2 contrast standards against its background?
   * Are interactive elements native semantic tags (`<button>`, `<a>`, `<nav>`, `<input>`) rather than clickable `<div>`s?

---

## Pre-Flight Checklist

Before presenting or shipping UI, verify each checkbox:

- [ ] **Functional Necessity:** Every visual element has a documented purpose.
- [ ] **No AI Slop:** No purple/cyan gradients, glowing borders, sparkle icons, or glassmorphism.
- [ ] **Structural Restraint:** No card-nesting addiction or decorative badges.
- [ ] **Domain-Specific Layout:** Layout reflects the real user workflow, not a generic template.
- [ ] **Semantic Color & Spacing:** Strict token palette and mathematical spacing scale applied.
- [ ] **Script-Aware Typography:** Type scale is disciplined; line-heights prevent clipping.
- [ ] **Subtle Interaction:** Hover states are calm; transitions are short (<200ms).
- [ ] **Complete States:** Empty, loading, and error states are fully articulated.
- [ ] **Honest Content:** No fake testimonials, fabricated metrics, or hype copy.
- [ ] **Passed All 3 Gates:** Passed the "Any Product" test, the 20% reduction test, and the Squint/WCAG audit.
