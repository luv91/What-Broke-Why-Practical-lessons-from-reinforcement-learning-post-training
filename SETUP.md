# Deploying the landing page to GitHub Pages

Two ways to ship this, depending on whether you want a build step.

- **`index.html`** — the whole site in one file. Drop it in the repo, turn on Pages, done.
- **`BookLanding.jsx`** — the same page as a React component, if you would rather build it properly with Vite.

Both contain the identical page, decision trees included. No UI library, no Tailwind — every style is inline.

## Fastest path: no build step at all

Drop `index.html` at the root of the repo. In **Settings → Pages**, set the source to the `main` branch, root folder. Done — the site is live at `https://<username>.github.io/<repo>/`.

That file is fully self-contained: React, Babel, and fonts load from CDNs, and the whole page (including the decision trees) is inline. Nothing to install, nothing to compile.

The tradeoff is that Babel compiles in the browser on each load, which adds roughly a second on first paint. If that bothers you, use the Vite path below instead.

## Faster path (Vite + GitHub Pages)

```bash
npm create vite@latest What-Broke-Why-Practical-lessons-from-reinforcement-learning-post-training -- --template react
cd What-Broke-Why-Practical-lessons-from-reinforcement-learning-post-training
npm install
npm install --save-dev gh-pages
```

Copy `BookLanding.jsx` into `src/` (it is one self-contained file — the decision trees are inside it), then replace `src/App.jsx`:

```jsx
import BookLanding from "./BookLanding";
export default function App() {
  return <BookLanding />;
}
```

In `vite.config.js`, set the base path to the repo name:

```js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  base: "/What-Broke-Why-Practical-lessons-from-reinforcement-learning-post-training/",   // must match the repo name exactly
});
```

Add the deploy scripts to `package.json`:

```json
"scripts": {
  "dev": "vite",
  "build": "vite build",
  "predeploy": "npm run build",
  "deploy": "gh-pages -d dist"
}
```

Then:

```bash
npm run deploy
```

In the repo settings under **Pages**, set the source to the `gh-pages` branch. The site goes live at `https://<username>.github.io/What-Broke-Why-Practical-lessons-from-reinforcement-learning-post-training/`.

## Links — already wired to this repo

1. **`REPO`** — set to `https://github.com/luv91/What-Broke-Why-Practical-lessons-from-reinforcement-learning-post-training`.
2. **`GH_PROFILE`** — set to `https://github.com/luv91`.
3. **The DOI** is wired to `10.5281/zenodo.21115798`, pulled from the PDF's own copyright page. The "Read the book" button and the Zenodo footer link both use it.

All three are set in `index.html` (lines 31–34) and `BookLanding.jsx` (lines 8–11); change them only if the repo or username changes.

## Fonts

Space Grotesk, IBM Plex Serif, and IBM Plex Mono load from Google Fonts via an `@import` inside the component. They're all open source. To self-host instead, download from Google Fonts and swap the `@import` for `@font-face` rules.

## What's in the design

- **The hero trace** is the Part IV entropy story drawn live: it collapses, explodes, then damps under Clip-Cov. Zone labels fade in as the line reaches them. It respects `prefers-reduced-motion` — the animation is skipped and the full curve renders immediately.
- **The journey accordion** numbers 01–05 because the programs genuinely are a sequence — each break forced the next. The numbering carries real information.
- **The symptom router** is filterable. Typing "entropy" or "reward" narrows the table live. It mirrors Map 0.3 from the book.
- **Colors are signals, not decoration.** Red marks what broke, green marks what held, amber marks the explosion phase. Same meaning everywhere on the page.

## Accessibility floor

Keyboard focus is visible on every interactive element, the accordion uses `aria-expanded`, the SVG carries an `aria-label` describing the trace, and the layout is responsive down to mobile.
