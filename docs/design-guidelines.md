# Design Guidelines — PTG Sokół Turystyczna

Design system extracted on 2026-10-17 from <https://www.turystycznaptg.com.pl/>, the website of the tourism section of PTG „Sokół” (Gmina Czernichów). The site is a WordPress blog running the stock **Twenty Ten** theme, so most values below are that theme's defaults as rendered on the live homepage (1280px viewport).

## Assets

| File | Description |
|---|---|
| [`../assets/homepage.png`](../assets/homepage.png) | Full-page homepage screenshot (no overlays) — the visual brand reference |
| [`../assets/design-tokens.json`](../assets/design-tokens.json) | Structured design tokens |

Not available from the site:

- **Logo** — the site has no graphic logo; the brand mark is the text title „PTG Sokół Turystyczna” (see [Logo usage](#logo-usage)).
- **Favicon** — no `<link rel="icon">`, and `/favicon.ico` returns 404.
- **Custom fonts** — none; the site uses system font stacks only.
- **Header image** — a fern photo (`/wp-content/themes/twentyten/images/headers/fern.jpg`, 940×198, a default Twenty Ten header). It was not saved locally; see the screenshot for reference.

## Colors

The palette is almost monochrome: black structure, grey text, one blue link colour and an orange-red hover.

| Token | Hex | Usage |
|---|---|---|
| `brand.primary` | `#000000` | Site title, entry titles, nav bar background, thick structural rules |
| `brand.accent` / `text.link` | `#0066CC` | All links (content and sidebar) |
| `brand.hover` / `text.linkHover` | `#FF4B33` | Link hover/active state |
| `background.default` | `#FFFFFF` | Page and content background |
| `background.light` | `#F5F5F5` | Calendar day cells, subtle fills |
| `background.nav` | `#000000` | Main navigation bar |
| `background.navHover` | `#333333` | Hovered nav item |
| `background.button` | `#32373C` | Buttons (e.g. „Szukaj”) |
| `text.primary` | `#333333` | Body copy in content area |
| `text.secondary` | `#666666` | Default body text, widget titles, tagline, footer credit |
| `text.caption` | `#222222` | Calendar caption |
| `text.tableHeader` | `#40464D` | Calendar weekday headers |
| `text.muted` | `#AAAAAA` | Nav links (idle), calendar day numbers |
| `text.onDark` | `#FFFFFF` | Text on buttons and hovered nav items |
| `border.strong` | `#000000` | 4px rules above header image and footer |
| `border.input` | `#949494` | Text inputs |
| `border.subtle` | `#DDDDDD` | Table cells |

No dedicated error or success colours are present on the site; define them with the group if the app needs them.

## Typography

Two system font stacks, no web fonts:

- **Serif (default body):** `Georgia, "Bitstream Charter", serif` — body text, content, widget titles, buttons, inputs.
- **Sans (headings & UI):** `"Helvetica Neue", Arial, Helvetica, "Nimbus Sans L", sans-serif` — site title, entry titles, navigation, calendar, footer site name.

No custom fonts were downloaded (none are served by the site).

**Weights:** 400 (regular), 700 (bold). Italic is used for the tagline and footer credit.

| Token | Size | Usage |
|---|---|---|
| `xs` | 11px | Calendar weekday headers |
| `sm` | 12px | Base body size, sidebar lists, inputs, buttons, tagline |
| `nav` | 13px | Navigation links |
| `md` | 14px | Calendar caption, footer site name |
| `base` | 16px | Article/page content |
| `lg` | 18px | Widget titles („Ostatnie posty”, „Archiwum”) |
| `xl` | 21px | Entry / page title (bold, sans) |
| `2xl` | 30px | Site title (bold, sans) |

**Line heights:** 18px for 12px text (the theme's vertical rhythm), 24px for 16px content, 1.3 for entry titles, 36px for the site title, 38px for the nav bar.

## Spacing

The theme works on an **18px vertical rhythm** (list item margin, footer padding) with a 4px-based horizontal scale.

| Token | Value | Typical use |
|---|---|---|
| `1` | 4px | Thin gaps, rule thickness |
| `2` | 8px | Input padding |
| `3` | 12px | Nav list left offset, content top padding |
| `5` | 20px | Wrapper side padding, content left margin |
| `6` | 24px | Paragraph bottom margin |
| `8` | 36px | Content bottom margin |
| `10` | 40px | Space between nav bar and main content |
| `rhythm` | 18px | List items, footer padding |

**Layout:** a fixed 940px page on a white background, centred. Content column 640px on the left, sidebar 220px on the right. Footer widgets use 220px columns.

## Border radius

| Token | Value | Usage |
|---|---|---|
| `none` | 0px | Everything — buttons, inputs, images, nav |

The style is strictly square-cornered; do not introduce rounded corners.

## Components

**Header** — site title (30px bold sans, black) on the left and italic grey tagline („Strona sekcji turystycznej”) on the right, 30px top padding. Below: a full-width 940×198 photo framed by a 4px black top border and 1px black bottom border.

**Navigation bar** — solid black bar, 38px high, directly below the header image. Links are 13px sans, `#AAAAAA`, padded `0 10px`. Hover: background `#333333`, text white. No uppercase, no underline.

**Entry / page** — title 21px bold sans black; body 16px Georgia, `#333333`, 24px line height, 24px between paragraphs. Links blue `#0066CC`, underlined, hover `#FF4B33`.

**Sidebar widgets** — titles 18px Georgia regular, `#666666`. Lists 12px with blue links and small square bullets.

**Button** — `#32373C` background, white 12px text, padding `10px 18px`, no border, square corners.

**Input** — white background, `1px solid #949494`, padding 8px, square corners; placed inline with the button (10px gap).

**Calendar** — 14px bold caption; weekday headers 11px `#40464D` with `#DDDDDD` borders; day cells `#F5F5F5` with `#AAAAAA` numbers.

**Footer** — separated by a 4px black top rule, 18px padding. Footer widgets in 220px columns; bottom line with site name (14px bold sans, black) on the left and italic grey credit on the right.

## Logo usage

There is no image logo. Use the **text wordmark** „PTG Sokół Turystyczna” set in the sans stack, 30px, weight 700, colour `#000000` on white. In the footer it appears smaller (14px bold). On a dark background (e.g. the black nav bar) use white `#FFFFFF`. If the app needs a graphic logo or favicon, it has to be designed and agreed separately — none exists on the source site.

## Visual style summary

The site is a plain, content-first community blog with a classic early-2010s WordPress look. It relies on a strict black/white/grey structure, Georgia for readable body copy and Helvetica-style sans for titles and navigation. The only colour comes from the nature photography in the header (ferns in green and blue) and the blue links with their orange-red hover. Corners are square, borders are bold black rules, and decoration is minimal — the tone is modest, practical and outdoorsy rather than commercial.
