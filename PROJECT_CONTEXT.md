# PROJECT CONTEXT: ALEX TEREX PERSONAL SHOWCASE & HEADLESS CMS

> **Usage for AI Assistant:**  
> Read this document in full before performing any changes, migrations, or feature implementations. It contains the complete architectural blueprint, design invariants, synchronization algorithms, and editorial rules for this project.

---

## 1. Executive Summary & Core Philosophy

- **Subject:** Official portfolio, bio-link, and video showcase for film director Aleksandr Terekhov (Alex Terex) — `FILMMAKER`.
- **Public URL:** `https://archaeoterex.github.io/`
- **Target Repository:** `archaeoterex/archaeoterex.github.io`
- **Architecture Paradigm:** **Zero-backend, Git-as-CMS architecture.**
  - There is no SQL/NoSQL database server or traditional cloud backend.
  - The repository's `index.html` file serves simultaneously as the live public site (served via GitHub Pages) and the single source of truth (the database).
  - The Control Panel (Admin) interacts directly with the GitHub Contents REST API v3 using a Personal Access Token (`Contents: read & write`), pulling the remote HTML, parsing it via DOMParser, applying user edits, and pushing an atomic commit back to GitHub.
- **Tone & Persona:** Minimalist, cinematic, brutalist-editorial, strictly no fluff.

---

## 2. Core Operational & Design Invariants

These rules are strict principles derived directly from `index.html` and `admin.html`:

1. **Preserve Production Compatibility & Content Immutability:**
   - Any tooling, refactoring, or layout adjustment must preserve full two-way compatibility with `index.html`.
   - **CRITICAL DATA RULE:** Code/design updates must NEVER overwrite or reset live user content (video cards, links, bio texts) with template defaults. Content in the remote `.video-grid` is authoritative and must be preserved during all HTML updates.
2. **Absolute Ban on Emojis & Colored Icons:**
   - Strictly NO emoji characters (e.g., ⚡, 🚀, ✅, ✕, 📥) in any UI element, buttons, toasts, toolbars, or code.
   - All iconography must be pure monochrome SVGs using `currentColor` or muted design tokens (`#8b949e`).
   - Button labels must consist of pure, clean text without icons, symbols, or accessories.
3. **Telegraphic Micro-Notifications (No Sprawling Banners):**
   - Strictly forbid multi-line alert banners, sprawling popups, or descriptive notification prose.
   - Status toasts must be strictly 1–2 words in uppercase (e.g., `SAVED`, `PUBLISHED`, `PUSHING...`, `TOKEN REQUIRED`, `FAILED`).
   - Notifications must be discreet, compact, centered, non-blocking, and auto-dismiss after 2–3 seconds without auxiliary links.
4. **Minimalist Telegraphic Copy:** All site copy, labels, and placeholders must remain concise, sharp, and focused on film showcase essentials without filler text or preambles.
5. **Consistent Tactile Neo-Brutalist Aesthetics:**
   - Dark GitHub-inspired base (`#0d1117`, `#21262d`, `#161b22`, border `#30363d`).
   - Signature cinnabar drop-shadow (`5px 5px 0px #cb4c3e`) with instant physical press transitions.
   - Smooth card/button radius hierarchy (`--radius-card: 14px`, `--radius-btn: 10px`/`14px`, `--radius-sq: 18px`, header action pills `999px`).
6. **Admin / Showcase Separation:**
   - The public website (`index.html`) is a standalone showcase containing **zero** links, buttons, or metadata referencing the administrative control panel.
   - The Control Panel is a decoupled interface managing content via the GitHub REST API using the client's Personal Access Token.


## 3. Architecture & Data Flow

```
                      +--------------------------------------+
                      |       GitHub Repository              |
                      | (archaeoterex/archaeoterex.github.io)|
                      +--------------------------------------+
                                  ^              |
             PUT /contents/index.html            | GET /contents/index.html
             (Base64 commit with SHA)            | (Base64 payload)
                                  |              v
+---------------------------------+----------------------------------+
|                     CONTROL PANEL / ADMIN                          |
|  - In-browser DOMParser parsing index.html                         |
|  - Granular dirty state tracking (dirty Set + videosDirty)         |
|  - LocalStorage caching (config, token, cached HTML, dirty flags)  |
|  - Instant Blob-based live preview (with <base href>)              |
+--------------------------------------------------------------------+
                                  |
                           Generates / Patches
                                  v
+--------------------------------------------------------------------+
|                      PUBLIC SHOWCASE (index.html)                  |
|  - Self-contained, zero-dependency, static HTML5/CSS3             |
|  - Strict Content-Security-Policy (CSP)                            |
|  - 1-line clamped H1, single subtitle, 4 square contact links      |
|  - Responsive YouTube-nocookie video grid (1 col mobile, 3 desktop)|
|  - Desktop-only hover-sliding title overlay captions               |
+--------------------------------------------------------------------+
```

### 3.1 Data Synchronization & Patching Mechanics
The CMS reads and writes directly into the DOM structure of `index.html`:
- **Parsing (`parseSiteConfig`):**
  - `H1` text content: splits on `/` to extract dual naming (`Aleksandr Terekhov / Alex Terex`).
  - `.subtitle`: extracts role (e.g., `FILMMAKER`).
  - `.square-grid a`: regex-matches URLs for Instagram (`instagram.com`), IMDb (`imdb.com`), LinkedIn (`linkedin.com`), and Email (`mailto:`).
  - `.video-grid .video-card`: extracts `iframe[src]` (converting embed IDs to `https://www.youtube.com/watch?v={id}`) and `.video-caption-text` (truncated to 45 chars).
- **Patching (`patchHtml`):**
  - Only fields flagged in the `dirty` set are mutated into the fetched remote DOM tree.
  - If videos are edited (`videosDirty = true`), `.video-grid` cards are rebuilt using `buildVideoCards()`.
  - Injects or updates `.video-caption` CSS style block if missing.
  - Serializes to `<!DOCTYPE html>\n` + `doc.documentElement.outerHTML`.
- **Concurrency & Conflict Handling:**
  - Employs GitHub file `sha`.
  - If GitHub responds with `409 Conflict`, it re-fetches the remote file SHA and re-applies the dirty patch before retrying.
  - If remote videos differ from local baseline (`!sameVideos()`), prompts the user for explicit overwrite confirmation before committing.
- **In-Memory Live Preview:**
  - Compiles patched HTML in memory into a `Blob([preview], { type: "text/html;charset=utf-8" })`.
  - Injects `<base href="https://archaeoterex.github.io/">` into the preview `<head>` so relative assets (`avatar.jpg`) render identically to the live site.
  - Opens via a secure `URL.createObjectURL(blob)` window.

---

## 4. Visual Design System & Tokens

### 4.1 Color Palette
| Token Variable | Hex Value | Usage / Meaning |
| :--- | :--- | :--- |
| `--bg-body` | `#0d1117` | GitHub Dark primary background |
| `--bg-btn` | `#21262d` | Button surface background |
| `--bg-card` / `--bg-caption` | `#161b22` | Card body, admin tab surface, video caption bar |
| `--border-color` / `--border`| `#30363d` | Strict structural border lines |
| `--accent-shadow` / `--accent`| `#cb4c3e` | Cinnabar / terracotta accent; hard drop shadow, active tabs, `/` slash |
| `--accent-hover` | `#db5344` | Hover state for accent buttons |
| `--text-main` | `#f0f6fc` | High-contrast off-white primary text |
| `--text-sub` / `--btn-icon` | `#8b949e` | Muted cool gray for subtitles, secondary labels, icons |
| `--btn-focus` | `#58a6ff` | Accessibility focus ring |
| `--active-bg` | `#30363d` | Selected tab background in admin |

### 4.2 Tactile Physical Press Mechanics
The signature button aesthetic relies on an isometric hard shadow that compresses on interaction:
- **Default:** `box-shadow: 5px 5px 0px var(--accent-shadow); transform: translate(0, 0);`
- **Hover:** `box-shadow: 3px 3px 0px var(--accent-shadow); transform: translate(2px, 2px); color: #ffffff;`
- **Active / Pressed:** `box-shadow: 0px 0px 0px var(--accent-shadow); transform: translate(5px, 5px); color: #ffffff;`
- **Motion Preference:** Fully disabled under `@media (prefers-reduced-motion: reduce)`.

### 4.3 Typography & Header Constraints
- **Font Stack:** `system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif`
- **Single-Line H1 Enforcement:**
  - `white-space: nowrap;` is mandatory.
  - Fluid sizing: `font-size: clamp(1.05rem, 4.8vw, 1.45rem);` ensures the full dual name fits on a single line across mobile screens without wrapping.
  - The slash `/` is wrapped in `<span class="slash" aria-hidden="true">/</span>` with color `#cb4c3e` and bold weight.
- **Subtitle:** `font-size: 0.85rem; letter-spacing: 0.5px; text-transform: uppercase; font-weight: 600; color: #8b949e;`.

---

## 5. Component Breakdown & Functional Rules

### 5.1 Avatar & Side Satellite Button
- **Avatar:** Path `avatar.jpg` (1000x1000 square source, monochrome portrait). Rendered size `110px x 110px`, circular (`border-radius: 50%`), `border: 2px solid #30363d`.
- **Side Satellite Button (`.btn-side`):**
  - Shape: Perfect circle (`border-radius: 50%`), compact size `60px x 60px` (reduced by 20% from initial 74px draft).
  - Position: Right of avatar (`left: calc(100% + 14px); bottom: 22px;`), elevated by 20% of the avatar photo height (22px) from the bottom boundary.
  - Colors & Styling: Transparent background (`background-color: transparent`), delicate thin gray border (`1px solid #30363d`), zero shadow (`box-shadow: none`).
  - Icon Color & Transition: Default icon color matches button icons (`#8b949e`), transitioning to warm ivory `#F4F1EA` on hover with subtle border illumination and background tint (without scaling/enlargement).
  - Icon Geometry: Scaled SVG glyph (viewBox `0 0 70 104.6`, rendered `21px x 31px`), fill `currentColor`.
  - Centering: Perfectly centered inside the circular button via Flexbox (`align-items: center; justify-content: center;`) with zero artificial offsets.
  - Target: Configurable via admin panel (default: `https://elsewire.news`). Opens in new tab (`target="_blank" rel="noopener noreferrer"`).
  - Hover Tooltip: Single styled floating badge (`.btn-side-tip`) displaying `ELSEWIRE` in uppercase off-white font (`#f0f6fc`), card surface background (`#161b22`), border (`#30363d`), positioned above button (`bottom: calc(100% + 7px)`). The native browser `title` attribute is intentionally omitted to prevent duplicate tooltips.
  - Responsive alignment: Because it is positioned absolutely within an `inline-block` container, the avatar itself remains perfectly centered on the screen axis on all screen sizes.

### 5.2 Social & Contact Links (Square Grid)
- Fixed 4-column grid (`grid-template-columns: repeat(4, 1fr)`), max container width `400px`.
- Aspect ratio: `1 / 1` square buttons (`--radius-sq: 18px`).
- Services in strict left-to-right order:
  1. **Instagram** (`https://instagram.com/archaeoterex`) — opens in new tab (`target="_blank" rel="noopener noreferrer"`).
  2. **IMDb** (`https://www.imdb.com/name/nm9440554/`) — opens in new tab. Custom SVG geometry for crisp logo rendering.
  3. **LinkedIn** (`https://www.linkedin.com/in/alexterex/`) — opens in new tab.
  4. **Email** (`mailto:alexterex@proton.me`) — **No `target="_blank"`** to prevent blank browser tab spawns.

### 5.3 Video Showcase (`video-grid`)
- **Embed Standard:** Privacy-enhanced YouTube embed (`https://www.youtube-nocookie.com/embed/{id}`).
- **Layout Behavior:**
  - Mobile / Tablets (< 1260px): Single-column stack, width 100%, 16:9 ratio.
  - Desktop (>= 1260px): Exactly 3 columns of `400px` each (`grid-template-columns: repeat(3, 400px)`), with the center video vertically aligning with the top bio column.
- **Desktop Hover-Slide Caption Bar:**
  - Displayed only on pointer-fine desktop devices (`@media (hover: hover) and (pointer: fine)`).
  - Occupies bottom 22% of the card height (`min-height: 38px`).
  - Dark background (`#161b22`), top border (`#30363d`), text colored in accent `#cb4c3e`.
  - On hover or focus within: smoothly slides down (`transform: translateY(100%)`) out of sight to let the viewer see the video thumbnail/controls unhindered.
  - Hard constraint: Caption text length capped at **45 characters**.

### 5.4 Control Panel Tab Structure
1. **Profile:** Name (max 80 chars) and Title (max 60 chars).
2. **Links:** Direct input fields for Instagram, IMDb, LinkedIn, Email, and Elsewire (Side Circle Link).
3. **Videos:** Dynamic re-orderable list with up/down arrows, deletion, addition, URL parsing, and caption length counter (`X/45`).
4. **GitHub:** Personal Access Token input with reveal/hide toggle, stored exclusively in client `localStorage`.

---

## 6. Content Security Policy (CSP) & Web Standards

The public `index.html` strictly enforces a security header meta tag:
```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'none'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https://archaeoterex.github.io https://i.ytimg.com; frame-src https://www.youtube-nocookie.com https://www.youtube.com; connect-src 'self'; object-src 'none'; base-uri 'self';">
```
- No external CDNs or untrusted remote scripts.
- Preconnect and dns-prefetch hints for `https://www.youtube-nocookie.com` and `https://i.ytimg.com`.
- Strict OpenGraph and Twitter card metadata for cinematic social sharing.
- Monogram favicon embedded as an inlined vector SVG data URI.

---

## 7. Migration & Expansion Guidelines for Future Turns

When working in this project or extending it:
1. **Preserve Compatibility:** Any refactoring to React, TypeScript, or modern frameworks must generate or support the exact same `index.html` output structure and GitHub synchronization protocol.
2. **No Secret Leaks:** Never commit GitHub personal access tokens or private credentials to git or public client bundles.
3. **Admin Panel Isolation:** The admin interface must remain decoupled from the public page, operating via GitHub API authentication without exposing management capabilities to general visitors.
4. **Refer to this Document First:** Before proposing structural changes, consult Section 2 (Invariants) and Section 4 (Design System).
