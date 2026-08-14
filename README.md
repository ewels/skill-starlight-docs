# starlight-docs

A Claude Code skill for writing [Astro Starlight](https://starlight.astro.build/) docs on Astro 7 / Starlight 0.41+ / Sätteri, and for making a docs site readable by AI agents.

## Install

```sh
git clone <this-repo> ~/.claude/skills/starlight-docs
```

Claude loads it when you edit `.md`/`.mdx`/`.mdoc` in a Starlight project, touch `astro.config.mjs` or `ec.config.mjs`, or ask for docs to be made LLM-friendly.

## Contents

- `SKILL.md` — non-negotiables, quick decisions, how to verify.
- `references/` — Expressive Code meta strings, built-in components, page conventions, llms.txt, the Sätteri Markdown pipeline.
- `scripts/refresh.sh` — installed versions, plus a live npm compatibility check for every plugin the skill names.
- `scripts/check-fences.mjs` — audits Expressive Code meta strings. `astro build` validates none of it.

```sh
./scripts/refresh.sh
node scripts/check-fences.mjs src/content/docs
```

Version numbers in `references/` are an August 2026 snapshot. Where they disagree with the scripts, the scripts are right.
