# Design hard rules

Source: [OpenAI "Designing Delightful Frontends with GPT-5.4"](https://developers.openai.com/blog/designing-delightful-frontends-with-gpt-5-4) (Mar 2026) plus gstack design methodology.

## Classifier (pick before evaluating)

- **MARKETING / LANDING** — hero-driven, brand-forward, conversion-focused → Landing Page Rules
- **APP UI** — workspace-driven, data-dense, task-focused → App UI Rules
- **HYBRID** — Landing rules on hero/marketing sections, App UI rules on functional sections

## Hard rejection (instant-fail if ANY apply)

1. Generic SaaS card grid as first impression
2. Beautiful image with weak brand
3. Strong headline with no clear action
4. Busy imagery behind text
5. Sections repeating the same mood statement
6. Carousel with no narrative purpose
7. App UI made of stacked cards instead of layout

## Litmus checks (YES/NO)

1. Brand/product unmistakable in first screen?
2. One strong visual anchor present?
3. Page understandable by scanning headlines only?
4. Each section has one job?
5. Are cards actually necessary?
6. Does motion improve hierarchy or atmosphere?
7. Would design feel premium with all decorative shadows removed?

## Landing page rules

- First viewport reads as one composition, not a dashboard
- Brand-first hierarchy: brand > headline > body > CTA
- Typography: expressive, purposeful — no default stacks (Inter, Roboto, Arial, system)
- No flat single-color backgrounds — gradients, images, or subtle patterns
- Hero: full-bleed, edge-to-edge
- Hero budget: brand, one headline, one supporting sentence, one CTA group, one image
- No cards in hero. Cards only when the card IS the interaction
- One job per section
- Motion: 2–3 intentional motions minimum
- Color: CSS variables, avoid purple-on-white defaults, one accent by default
- Copy: product language, not design commentary. If deleting 30% improves it, keep deleting

## App UI rules

- Calm surface hierarchy, strong typography, few colors
- Dense but readable, minimal chrome
- Organize: primary workspace, navigation, secondary context, one accent
- Avoid dashboard-card mosaics, thick borders, decorative gradients, ornamental icons
- Copy: utility language — orientation, status, action
- Cards only when the card IS the interaction
- Section headings state what the area is or what the user can do

## Universal rules

- Define CSS variables for the color system
- No default font stacks as the primary typeface
- One job per section
- Cards earn their existence
- NEVER use small, low-contrast type (body < 16px or contrast < 4.5:1)
- NEVER use placeholder-as-the-only-label
- ALWAYS preserve visited vs unvisited link colors
- NEVER float headings between paragraphs (heading closer to the section it introduces)
