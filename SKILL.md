---
name: starlight-docs
description: Author and edit Astro Starlight documentation on the current stack (Astro 7, Starlight 0.41+, Sätteri), and make it machine-readable for AI agents. Covers the full Expressive Code meta-string syntax (line highlighting, ranges, labelled markers, diff syntax, collapsible sections, line numbers, frames, titles), Starlight's built-in MDX components (Tabs, Steps, FileTree, Card, CardGrid, LinkCard, LinkButton, Aside, Badge, Icon, Code), the Sätteri Markdown pipeline and why remark/rehype plugins no longer run, LLM-friendliness (llms.txt, raw Markdown routes, page action buttons), and og:image share cards, which every site needs and Starlight does not generate. Use whenever writing or editing .md, .mdx or .mdoc files in a Starlight project, whenever a site needs social share cards, OpenGraph images or link previews, whenever a code block needs more than a bare language tag, whenever content would suit a built-in component, and whenever touching astro.config.mjs or ec.config.mjs. Also use when asked to make docs agent-friendly or LLM-readable, when a remark or rehype plugin stops working, when upgrading Astro or Starlight, or when a code block renders wrongly.
---

# Starlight docs authoring

Starlight renders code blocks through **Expressive Code**, not plain Shiki. Most of its power lives in the *meta string* — everything after the language on the opening fence. That syntax is easy to get wrong from memory, so **read `references/expressive-code.md` before writing any code block that needs more than a language tag**.

## Read these when relevant

| File | Read it when |
| --- | --- |
| `references/expressive-code.md` | Any code block with highlighting, line ranges, diffs, titles, frames, collapsed sections, line numbers, word wrap, or the `<Code>` component. **This is the one that matters most.** |
| `references/components.md` | Reaching for Tabs, Steps, FileTree, Cards, Asides, Badges, Icons, LinkButtons — or deciding whether a component is warranted at all. |
| `references/conventions.md` | Structuring a new page, writing frontmatter, choosing `.md` vs `.mdx` vs `.mdoc`, sidebar placement, prose style. |
| `references/llm-friendly.md` | Making the site machine-readable: llms.txt, per-page `.md` routes, page action buttons, MCP, discovery signposting, writing prose that survives Markdown conversion. |
| `references/markdown-pipeline.md` | Anything touching `markdown.processor`, remark/rehype plugins, Sätteri features or plugin ports, or an Astro 6 → 7 upgrade. Read this before editing Markdown pipeline config. |
| `references/share-cards.md` | `og:image` share cards: generating them, the `Head` override, logos, fonts, and pages the route can't enumerate. **Every site needs these** — check before calling a site finished. |

| `scripts/refresh.sh` | **Run this, don't read it.** Installed versions and live plugin compatibility. See below. |
| `scripts/check-fences.mjs` | **Run this, don't read it.** Audits every Expressive Code meta string in a content tree. Run it after writing or editing code blocks — `astro build` validates none of this. |

Don't read all six reference files by default. Read what the task needs.

## Start here: refresh, don't recall

This skill's version-sensitive facts (plugin versions, peer ranges, config options, component props) go stale within weeks. **Before recommending or installing any plugin, or citing any version, run:**

```sh
./scripts/refresh.sh          # installed versions + live plugin compatibility (~5s)
./scripts/refresh.sh --docs   # also mirrors canonical doc sources to /tmp/starlight-upstream
```

It reads `package.json` for what's installed, queries npm for every plugin this skill names, and prints a verdict per plugin: **OK** (declares Astro 7 / Starlight ≥0.41, or depends on `@astrojs/markdown-satteri`), **CHECK** (installs but never declared Astro 7), **DEAD** (excludes Astro 7 — do not use). With `--docs` it mirrors the Starlight, Expressive Code and Astro Markdown doc sources as Markdown from `raw.githubusercontent.com`, which is authoritative and needs no HTML scraping.

**Prefer script output over anything written in this skill.** The `references/` files carry the reasoning and the decisions; the script carries the current facts. When they disagree, the script wins.

If the network is blocked, say so plainly and use the bundled references, telling the user they are a snapshot from **August 2026** (Astro 7.2.2, Starlight 0.41.7, Sätteri default). Never present snapshot version numbers as current without having refreshed.

## The stack to target

Astro 7, Starlight 0.41+, Sätteri, `.md` by default and `.mdx` only where components are needed. Astro 7 (June 2026) brought a Rust compiler that errors on unclosed tags instead of auto-correcting them; Starlight 0.41 dropped Astro 6 entirely. **Sätteri is the Markdown processor to use** — it needs no configuration, and Starlight and Expressive Code both support it natively. If the project has remark/rehype plugins, **keep `unified()` rather than porting them** — Astro supports it as a first-class processor and switching back is one install plus one config line. See the decision tree in `references/markdown-pipeline.md`; most such plugins are redundant on Starlight and can simply be deleted.

When a project is behind, say so and offer the upgrade rather than quietly writing to the old APIs.

## Non-negotiables

1. **`.md` files cannot use components.** Components require `.mdx` (imported) or `.mdoc` (Markdoc preset, no import). If a `.md` page needs `<Tabs>`, rename it to `.mdx` first and add the import. Asides are the exception — the `:::note` directive syntax works in plain `.md`.
2. **Import from `@astrojs/starlight/components`**, not `astro-expressive-code/components`, in a Starlight project. Getting this wrong is the single most common failure.
3. **Line numbers in markers count raw source lines, starting at 1.** They are unaffected by `startLineNumber`. Count the actual lines in the fence, including blank ones.
4. **Never invent meta attributes.** If it isn't in `references/expressive-code.md`, it doesn't exist. Expressive Code silently ignores unknown meta, so a typo produces a plain block with no error — check the rendered output rather than assuming it worked.
5. **Run `./scripts/refresh.sh` before naming any plugin or version.** A plugin's README, star count and polished docs site tell you nothing about whether it works — `starlight-contextual-menu` reads as current and has been stuck on Astro 5 since October 2025. `peerDependencies` is the only reliable signal, and the script reads it for you. Never recommend a plugin on reputation alone.
6. **Only one thing may own the Markdown routes.** `starlight-dot-md`, `starlight-md-txt` and `starlight-page-actions` can all serve raw `.md`; two together collide. Same for `llms.txt`, which both `starlight-llms-txt` and `starlight-page-actions` generate.
7. **Plugin-injected routes are invisible to other plugins.** Pages generated by `starlight-openapi`, `starlight-pydocs` and friends are injected routes, not content collection entries. `starlight-links-validator` reports every link into them as broken until you `exclude` their base, and `starlight-llms-txt` omits them from `llms-full.txt` entirely — bridge that with `optionalLinks`. Neither failure is obvious from the error message, and the llms.txt one is silent.
8. **Every site gets share cards.** Starlight emits every OpenGraph tag *except* `og:image`, so out of the box every shared link unfurls as a blank grey box. A site is not finished without them. If a project has no `og:image`, say so and offer to add them — `astro-og-canvas` plus a `Head` override, about an hour. See `references/share-cards.md`.
9. **Optional plugins must be installed.** `collapse=`, `collapseStyle=`, `showLineNumbers` and `startLineNumber` need `@expressive-code/plugin-collapsible-sections` / `@expressive-code/plugin-line-numbers` in the config. Before using them, check `ec.config.mjs` or the `expressiveCode` block in `astro.config.mjs`. If absent, either add the plugin or use a different approach — don't emit meta that will be silently dropped.

## Quick decisions

**Showing a change to code?** `diff` with `lang="js"` to keep syntax highlighting, or `ins={}` / `del={}` line markers. Never hand-write `+`/`-` prefixes in a normally-tagged block. **A from-scratch example is not a change** — green `+` gutters there are wrong, and a plain `{}` is what you want.

**Long example with boilerplate?** `collapse={1-8, 20-24}` beats trimming the example down and beats `// ...` placeholders. Never collapse five lines or fewer — the summary costs more than it saves.

**Snippet that leaves the reader asking "where does this go?"** Show the whole config file and collapse everything else: `collapse={1-8, 18-23}` plus a marker on the option. They get the option *and* its nesting. Better than `title=` alone, which names the file but not the position in it.

**Explaining specific lines?** Put a comment in the code and highlight the range: `// Name the package and point griffe at its source.` with `{10-15}`. Not a labelled marker — those render as a caption box beside the code, read as site chrome, and vanish when anyone copies the block. Save labels for annotations that have no line to attach to.

**Marker on a long line?** Check it isn't scrolling out of view — the code column is only ~60–65 characters wide with sidebar and ToC. Break the line, or add `wrap`. Don't apply that budget to unmarked lines; splitting an idiomatic import across four lines to satisfy it is worse than the long line.

**Terminal commands?** Just use `sh`/`bash` and let the frame detect itself. Add `frame="none"` when a page has many one-line commands and the frames get noisy.

**File path on a code block?** `title="src/pages/index.astro"` on the fence, or a file-name comment in the first four lines. Prefer `title` — it's explicit and survives copy-paste.

**Two ways to do the same thing (npm/pnpm/yarn, JS/TS)?** `<Tabs syncKey="pkg">` so the reader's choice persists across the page and site.

**A sequence the reader must follow in order?** `<Steps>` around an ordered list. Not bold "Step 1" headings.

**Project layout?** `<FileTree>`, not a fenced ASCII tree.

**Caveat or warning?** Aside directive (`:::caution`). Reserve `:::danger` for genuine data loss or security risk.

**Asked about remark/rehype plugins, or something Markdown-y stopped working?** Sätteri doesn't run them, and fails silently. Don't propose porting anything: either the plugin is redundant on Starlight (delete it) or the project should stay on `unified()` (one install, one config line). Decision tree in `references/markdown-pipeline.md`.

**Setting up, reviewing or finishing a site?** Check it has `og:image` share cards. It almost certainly doesn't — Starlight generates every other OpenGraph tag, so the omission is easy to miss until a link unfurls grey in Slack. `astro-og-canvas` plus a `Head` override; recipe in `references/share-cards.md`.

**Asked to make the docs agent-friendly?** Two plugins and a signpost: `starlight-llms-txt` for site-level indexes and `starlight-page-actions` for copy/open buttons plus raw `.md` routes — then a visible link to `/llms.txt`. **Not `starlight-contextual-menu`**, which is stuck on Astro 5 despite being the most-recommended option. Generating the files without linking to them is the usual failure. On a site hosting several products, `customSets` is required rather than optional, or retrieval answers about the wrong product. MCP is a later step, and extending an existing product MCP usually beats a docs-only one. See `references/llm-friendly.md`.

**Writing prose on a site with llms.txt?** Front-load the answer, keep essential information in body text rather than asides (asides are stripped from `llms-small.txt` by default), and write `description` frontmatter on every page.

## Verifying

Three layers, and the first two are cheap:

```sh
node path/to/scripts/check-fences.mjs src/content/docs   # meta strings
npx astro build                                          # MDX, links, types
```

`check-fences.mjs` re-derives every marker and collapse range from the final file contents and reports ranges outside the fence, long labels landing on a non-blank line, markers buried inside collapsed ranges, plugin attributes with no plugin installed, marked lines wide enough to scroll out of view, and `ins=`/`del=` on blocks that show no change. Counting fence lines by hand is where this goes wrong; don't.

The build catches missing MDX imports, a `<TabItem>` without a label, broken internal links (with `starlight-links-validator`) and type errors.

**Then open a browser.** A green build is much weaker evidence than it feels. On a real project this session, every one of these passed the build and Prettier and the type-checker: marked code clipped off the right edge of the code column, a sidebar carrying five identically-named "Overview" links, a labelled marker rendering as chrome nobody wanted, and a sample presented as real generated output that had been quietly doctored so a marker had a blank line to sit on. Only looking at the page found them.

```sh
agent-browser open "http://localhost:4321/base/guides/your-page/"
agent-browser eval "(()=>{document.querySelectorAll('.expressive-code')[1].scrollIntoView({block:'center'});return 'ok'})()"
agent-browser screenshot /tmp/block.png
```

Look at every block carrying markers or `collapse=`, both theme colours if you touched CSS.

Add `starlight-links-validator` to CI for internal link checking; 0.25.3 supports Starlight 0.41 / Astro 7 and is Sätteri-aware.

## When the references and reality disagree

Trust reality. Re-run `./scripts/refresh.sh --docs` and read from `/tmp/starlight-upstream`, which is the upstream Markdown source rather than a scrape. If a path in the script 404s, upstream has moved a file — fetch `https://raw.githubusercontent.com/withastro/starlight/main/docs/src/content/docs/` or the Expressive Code equivalent to find it, and tell the user the script needs a path updated.

Starlight's own docs have no `llms.txt` (still an open request, withastro/starlight#3007), so raw GitHub is the fastest authoritative route. Where a project has `starlight-llms-txt` installed, its generated `llms-full.txt` is a good source for that project's own content.
