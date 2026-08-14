# Share cards (og:image)

**Every Starlight site needs these.** Starlight emits every OpenGraph tag except the image: `og:title`, `og:description`, `og:url`, `og:site_name`, `og:locale` and `twitter:card: summary_large_image` all come for free, and then the card renders as a blank grey box in Slack, Discord, Teams, iMessage, Bluesky, LinkedIn and X. A docs link is shared far more often than it is searched for. Treat a missing `og:image` like a missing favicon: not a nice-to-have, and about an hour of work once.

There is no official Starlight plugin for this. The tool is `astro-og-canvas` — same author as Starlight, used by Starlight's own docs — plus a `Head` component override. Two files and three config lines.

## Contents

- [The two halves](#the-two-halves)
- [The image route](#the-image-route)
- [The Head override](#the-head-override)
- [Using the project logo](#using-the-project-logo)
- [Fonts](#fonts)
- [Pages the route cannot enumerate](#pages-the-route-cannot-enumerate)
- [Design that survives the platforms](#design-that-survives-the-platforms)
- [Verifying](#verifying)
- [Gotchas](#gotchas)

---

## The two halves

1. **A route that renders PNGs** — one per page, prerendered into `dist/og/…png` at build time.
2. **A `Head` override that points at them** — Starlight will not do it for you, and the URL must be absolute.

Both halves are needed. Generating images nobody links to is the usual failure, and it is silent.

## The image route

```ts
// src/pages/og/[...route].ts
import { getCollection } from 'astro:content';
import { OGImageRoute } from 'astro-og-canvas';

const entries = await getCollection('docs');

export const { getStaticPaths, GET } = await OGImageRoute({
  // Keys become the route: id `guides/setup` → /og/guides/setup.png
  pages: Object.fromEntries(entries.map((entry) => [entry.id, entry.data])),

  getImageOptions: (_path, page) => ({
    title: page.title,
    description: page.description ?? '',
    logo: { path: 'src/assets/og-logo.png', size: [960] },
    bgGradient: [
      [27, 31, 42],
      [42, 51, 78],
    ],
    border: { color: [242, 198, 65], width: 16, side: 'inline-start' },
    padding: 60,
    font: {
      title: { color: [244, 245, 248], size: 58, lineHeight: 1.3 },
      description: { color: [170, 178, 196] },
    },
  }),
});
```

Cards are cached in `node_modules/.astro-og-canvas` and keyed on the options, so rebuilds are cheap and only changed pages re-render.

`description ?? ''` is not defensive noise: Astro's `strictest` tsconfig sets `exactOptionalPropertyTypes`, so passing `string | undefined` into the optional `description` is a type error. `astro check` catches it, `astro build` does not.

## The Head override

```astro
---
// src/components/Head.astro
import Default from '@astrojs/starlight/components/Head.astro';
import { getCollection } from 'astro:content';

const ids = new Set((await getCollection('docs')).map((entry) => entry.id));
const { id } = Astro.locals.starlightRoute;
const image = new URL(
  `${import.meta.env.BASE_URL}/og/${ids.has(id) ? id : 'index'}.png`.replace(/\/{2,}/g, '/'),
  Astro.site,
);
---

<Default><slot /></Default>
<meta property="og:image" content={image} />
<meta property="og:image:width" content="1200" />
<meta property="og:image:height" content="630" />
<meta name="twitter:image" content={image} />
```

```js
// astro.config.mjs
starlight({
  components: { Head: './src/components/Head.astro' },
})
```

- **`site` must be set in `astro.config.mjs`.** Crawlers reject a relative `og:image`. `new URL(path, Astro.site)` is the whole trick, and it is also why this breaks on a site that never set `site`.
- **Handle the base path.** `import.meta.env.BASE_URL` already carries it; the `/{2,}` squash stops `//og/…` when the base ends in a slash.
- The override runs for **every** page Starlight renders, including plugin-injected routes that use `StarlightPage` — that is what makes the fallback below work.
- Starlight already emits `twitter:card: summary_large_image`, so don't repeat it. `og:image:width`/`height` are optional but stop some clients from laying out the card before they have fetched it.

## Using the project logo

Put the site's wordmark on the card. It is the one element that makes a card recognisable at thumbnail size, and it costs one option.

**CanvasKit decodes bitmaps only — PNG, JPEG, WebP — never SVG.** Most projects only have an SVG logo, so rasterise it. Do that at build time rather than committing a second copy that drifts from the real logo:

```ts
import sharp from 'sharp';

// `density` renders the SVG well above the target width, so fine detail in the
// mark survives the downscale. sharp's default of 72dpi renders it at its
// intrinsic size and the result looks pixellated at card scale.
const logo = 'node_modules/.astro/og-logo.png';
await sharp('src/assets/logotype-dark.svg', { density: 288 }).resize({ width: 960 }).png().toFile(logo);
```

Do this at module top level in the route file — it runs once per build, before any image is drawn. `sharp` is already in the dependency tree of most Astro sites (it is the default image service), so this adds nothing.

Use the **dark-mode variant** of a logo on a dark card: the light variant is dark ink meant for a white page and it will disappear.

## Fonts

CanvasKit has no system fonts and no bundled font. Text is drawn with what you give it:

```ts
fonts: [
  'https://api.fontsource.org/v1/fonts/michroma/latin-400-normal.ttf',
  'https://api.fontsource.org/v1/fonts/noto-sans/latin-400-normal.ttf',
],
font: { title: { families: ['Michroma', 'Noto Sans'] } },
```

- The `families` stack works like CSS: first family wins, later ones fill in missing glyphs. Every family named must be loaded by the top-level `fonts` array.
- Omit `fonts` and the package fetches Noto Sans 400 from `api.fontsource.org` for you. Either way **the first build on a machine needs network**, which matters for offline or air-gapped builds. Files cache in `node_modules/.astro-og-canvas`.
- **Font files must be TTF or OTF.** The `@fontsource/*` npm packages ship only WOFF/WOFF2, so use the `api.fontsource.org/v1/fonts/<family>/<subset>-<weight>-normal.ttf` URLs even when the package is installed for the site itself.
- `weight: 'Bold'` with only a 400 file loaded gives a synthesised fake bold. Load the 700 file, or set the weight to 400 and let size carry the emphasis.
- Reusing the site's display font on the card ties the two together; if that font is also used for headings on the site, one `@fontsource` install plus a `customCss` rule on `h1#_top` covers both.

## Pages the route cannot enumerate

Injected routes — `starlight-openapi`, `starlight-pydocs`, `starlight-blog` archives, anything a plugin generates — are **not** content collection entries. The image route cannot see them, so it cannot make a card for each one. There is no clever workaround for a static build: a `getStaticPaths` needs the list up front.

Give them the site card instead. That is what the `ids.has(id) ? id : 'index'` fallback in the override does: pages with a generated card get theirs, everything else gets the home page's. A site-branded card beats a grey box, and the title and description in the unfurl are still page-specific because Starlight sets those from the page's own frontmatter.

Only invest more when a plugin's pages are the ones people actually share.

## Design that survives the platforms

- **1200 × 630** — the size every platform crops toward. `astro-og-canvas` defaults to it.
- **Keep text away from the edges.** Some clients crop to squarer aspects on mobile. `padding: 60` and a bottom-heavy layout survive it.
- **Test the longest title in the sidebar, not a short one.** Display and wide-set faces overflow far sooner than a UI sans; drop the title `size` until the longest one wraps to two lines at most.
- **Dark backgrounds read better** in the dark-themed clients most people share into, and hide the JPEG-ish artefacts some platforms add.
- **One accent** — a border on a single edge, in the brand colour — is enough to make cards recognisable in a feed.
- Contrast still matters: description text at ~40px in a mid grey on a dark ground should stay above 4.5:1.

## Verifying

`astro build`, then actually look at the PNGs — the build cannot tell you a title overflowed or a logo went blurry:

```sh
npx astro build
ls dist/og                                            # cards exist
grep -o '<meta property="og:image"[^>]*>' dist/guides/*/index.html
open dist/og/index.png dist/og/guides/<longest-title>.png
```

Check three things by eye: the longest title, a page with no `description`, and the logo at 1:1 (zoom in — aliasing on a logo mark is the most common defect and it is invisible at page scale).

An end-to-end assertion is worth having, because a renamed collection id breaks the link silently:

```ts
test('every page points at a share card that exists', async ({ page, request }) => {
  await page.goto('guides/setup/');
  await expect(page.locator('meta[property="og:image"]')).toHaveAttribute('content', /\/og\/guides\/setup\.png$/);
  expect((await request.get('og/guides/setup.png')).status()).toBe(200);
});
```

Before announcing a launch, paste a URL into the actual target — Slack and Bluesky both render immediately and both cache hard, so check before the link is shared, not after.

## Gotchas

1. **No `site` in the config → no valid card.** Relative `og:image` URLs are ignored by every crawler.
2. **The base path must be in the URL.** A site under `/project/` that emits `/og/x.png` 404s for the crawler.
3. **Caches are aggressive.** Slack, X and LinkedIn cache an unfurl for days. During development, change the image URL (or use their debug tools) rather than trusting a re-paste.
4. **`entry.id` is the key, and it is stable-ish.** Renaming a file changes the id, which changes the card URL — harmless, but a hard-coded URL anywhere else will rot.
5. **Don't commit a rasterised logo.** It drifts from the SVG the site uses. Generate it in the route module.
6. **Don't hand-roll this with Satori/resvg or a headless browser.** More dependencies, worse fonts, slower builds. `astro-og-canvas` is ~40 kB and does the whole job.
7. **A splash / hero home page has no `h1`,** so the home card's title comes from frontmatter `title` — check it says something, not just the site name repeated.
8. **Run `./scripts/refresh.sh` before pinning a version.** `astro-og-canvas` is not a Starlight plugin and declares no Starlight peer, so it does not go stale the same way, but check it all the same.
