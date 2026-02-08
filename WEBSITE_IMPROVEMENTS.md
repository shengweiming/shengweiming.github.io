# Website Improvement Ideas

This is a prioritized list of improvements for your personal academic website.

## 1) Clarify academic identity and purpose (high impact)
- Replace the generic page title (`My Site with Tabs`) with your name and role (e.g., `Weiming Sheng | Philosophy, Statistics, and ML`).
- Add a one-sentence research statement near the top that explains your core agenda.
- Add your institution and current position for immediate context.

## 2) Fix information architecture and tab labeling (high impact)
- The nav includes a `Writing` tab, but the panel content is currently `Teaching`; align labels and section IDs so navigation is unambiguous.
- Consider a top-level structure optimized for academics:
  - About
  - Research / Publications
  - Teaching
  - Notes / Writing
  - CV
  - Contact

## 3) Improve contact and professional links (high impact)
- Add direct links to Google Scholar, ORCID, CV PDF, and (if applicable) X/Twitter or LinkedIn.
- Use a clickable email link (`mailto:`) and consider obfuscation if spam becomes an issue.

## 4) Expand research visibility (high impact)
- For each project/publication, add:
  - venue/status (preprint, conference, journal)
  - 1–2 sentence plain-language summary
  - links: paper, code, slides, data (if available)
- Add selected works first, then a full publications list (or link to a BibTeX-backed page).

## 5) Add credibility assets (medium impact)
- Add a downloadable CV.
- Add short bio variants (50-word and 150-word versions) for conference pages.
- Add a headshot for recognizability.

## 6) Strengthen accessibility and semantics (medium impact)
- Ensure tab UI updates keyboard semantics fully (`tabindex`, `hidden`, and focus order) so non-mouse navigation is robust.
- Add a “Skip to main content” link.
- Ensure heading hierarchy remains consistent (single `h1`, then section `h2`s).

## 7) Improve metadata and discoverability (medium impact)
- Add SEO/social metadata:
  - `<meta name="description" ...>`
  - Open Graph and Twitter card tags
  - canonical URL
- Add favicons and a web manifest.

## 8) Performance and maintainability (medium impact)
- Split CSS/JS into separate files as the site grows.
- Add basic analytics (privacy-friendly option like Plausible) to understand what visitors read.
- Add a changelog or "last updated" timestamp to signal freshness.

## 9) Optional next-step enhancements (nice to have)
- Add a writing/notes feed (RSS).
- Add a tiny search over publications and notes.
- Add dark/light mode toggle while preserving accessibility contrast.

## Suggested implementation order
1. Title/metadata + tab/section alignment + contact links
2. Research/publications section with structured entries
3. CV + professional profile links
4. Accessibility polishing
5. Visual refinements and optional features
