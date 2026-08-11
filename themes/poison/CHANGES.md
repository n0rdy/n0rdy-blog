# Local changes to Poison

Changes made in this fork, on top of upstream `lukeorth/poison` @ `07485e8`
(2024-09-05, last upstream commit).  Earlier local changes — the "My Products"
section, `nofollow` links by default, absolute `og:image`/`twitter:image` URLs,
the `mailto:` double-slash fix — are in the pre-vendoring git bundle.

## 2026-08-11 — vendoring and modernisation

### KaTeX is now opt-in

Previously every page loaded the full KaTeX library, whether or not it
contained maths: ~265KB of JavaScript and ~38KB of CSS on every request.

It is now gated behind `math: true` in a page's front matter, or `math = true`
under `[params]` site-wide.  Pages without maths ship **2.5KB of JS and 15.5KB
of CSS**; pages with `math: true` get exactly what they got before.

This also defuses a latent hazard: the renderer treats a bare `$` as an inline
maths delimiter, which is unhelpful on a blog that discusses shell variables
and bcrypt hashes.

### Fixed: KaTeX web fonts never loaded

`assets/css/lib/katex.css` referenced its fonts as `url(fonts/...)`.  Because
the stylesheet is served from the concatenated bundle at `/css/`, those
resolved to `/css/fonts/...`, which does not exist — the fonts live at
`/fonts/`.  Any page using maths rendered with fallback system faces.  Paths
are now absolute.

### Fixed: article metadata was on the wrong pages

`layouts/_partials/head/meta.html` guarded its article tags with
`.Page.IsNode`, which is true for *list* pages and false for regular pages.
`og:type=article`, `article:published_time` and `article:author` were therefore
emitted on `/posts/` and were entirely absent from actual posts.  Now gated on
`.IsPage`, with `og:type=website` elsewhere.

### Fixed: empty description tags on every post

Posts carry no `description` in front matter, and there was no fallback, so
every post shipped `<meta name="description" content="" />` — repeated across
`og:description`, `twitter:description` and `itemprop`.  Descriptions now fall
back to the page summary, plainified and truncated to 160 characters.

### Fixed: duplicate `/posts/page/N/` URLs

`meta.html` called `.Paginate` purely to emit `rel="prev"`/`rel="next"` links.
Because `layouts/list.html` renders the full year-grouped list and ignores the
paginator, this generated `/posts/page/2/` and `/posts/page/3/` — each serving
a complete copy of the post list under a different URL.  The call is gone.

### Fixed: unclosed `<nav>` in pagination

`layouts/_partials/pagination.html` opened `<nav id="page-nav">` and closed
only the surrounding `<div>`.  Also removed the `T "Prev"` / `T "Next"` lookups,
which resolved to empty strings as the theme ships no i18n files.

### Titles for taxonomy pages

`/tags/` and `/tags/<term>/` fell through the title logic and were all titled
with the raw site title.  They now follow the same `brand - title` pattern as
everything else.  `.Scratch` has been replaced with ordinary variables.

### Removed `<base href>`

`meta.html` emitted `<base href="{{ .Permalink }}">`, silently re-anchoring
every relative URL on the page.  All content links here are already absolute or
root-relative, so this only ever created the opportunity for surprise.

### Fonts: woff2 only, with `font-display: swap`

Dropped the `.woff` fallbacks (woff2 is universally supported since 2016) and
the no-op `local('')` entries, and added `font-display: swap` so text renders
in a fallback face instead of staying invisible during font load.

### Removed dead weight

- `static/katex/` (2.9MB) — a complete second copy of KaTeX, referenced by
  nothing.  The theme serves KaTeX from `assets/`.
- `static/fonts/*.woff`, `static/fonts/*.ttf` (1.2MB) — superseded by woff2.
- `layouts/_partials/head/favicon.html` — empty and unreferenced.

### Hugo 0.146+ template layout

Migrated to the current template system: `layouts/_partials/`,
`layouts/_shortcodes/`, `layouts/_markup/`, and top-level `baseof.html`,
`home.html`, `list.html`, `single.html` in place of `layouts/_default/` and
`layouts/index.html`.  Output is byte-identical; `theme.toml` now declares
`min_version = "0.146.0"`.

### Markup tidying

Dropped the XHTML `xmlns` attribute, replaced the `http-equiv` content-type
meta with `<meta charset="utf-8">` as the first element in `<head>`, removed
the obsolete `language`/`type` attributes from `<script>`, quoted the `favicon`
and social meta attribute values, dropped the always-empty `fb:admins` tag, and
removed the non-existent `og:article:published_time` property.
