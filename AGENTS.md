# Repository Instructions

@/Users/heyandras/.codex/RTK.md

## Project

Landing page for [Jean](https://jean.build), built with Astro, Svelte and Tailwind CSS v4. Use `bun` (`bun run dev`, `bun run build`). CI uses `npm ci`, so keep `package-lock.json` in sync with `bun.lock` after dependency changes (`npm install --package-lock-only`).

## Styling

- Same "graphite" design as coolify.io and the Coolify app. Tokens live in `src/styles/global.css` (Tailwind v4 `@theme`): `app`, `panel`, `surface`, `raised`, `fg`, `fg-dim`, `fg-faint`, `hairline`, `coollabs` purple, `warning` yellow.
- Shared classes: `.btn` + `.btn-primary` / `.btn-neutral` (+ `.btn-lg`), `.card`, `.eyebrow` (global.css), `.navbar-link` and `.text-link` (Layout.astro).
- Self-hosted Geist Sans / Geist Mono variable fonts in `public/fonts/`.
- Icons: Reicon. Deep-import one icon and render it with the wrapper: `import Icon from "../components/Icon.astro"; import Rocket from "reicon/icons/Rocket";` then `<Icon icon={Rocket} class="size-4" />` (Svelte: `Icon.svelte`). Brand and agent logos stay inline SVGs.

## Screenshots

- Homepage gallery data: `src/data/screenshots.js` (first entry is the hero image). Viewer: `src/components/ScreenshotGallery.astro`.
- Images are real Jean app screenshots in dark mode, 1440x900 viewport at 2x, saved as WebP in `public/images/screenshots/` (`ffmpeg -i shot.png -c:v libwebp -quality 82 shot.webp`). Do not show tokens, secrets or private IPs.
