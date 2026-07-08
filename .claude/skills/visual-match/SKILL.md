---
description: Screenshot a page on the dev server, screenshot a reference page or URL, compare them visually, identify style gaps, apply fixes, and loop until they match. Invoke with a local route (e.g. /projects/project-10) and optionally a reference route or URL.
---

# Skill: Visual Match

Takes screenshots of a new or changed page on the local dev server, compares them against a reference page (another route on the same server, a built/served copy, or an external URL), identifies style divergence, applies code fixes, and repeats until the pages are visually indistinguishable.

**Invoke with:** a local route to check (e.g. `/projects/project-10`) and optionally a reference route or URL to match against (defaults to an equivalent existing project page, e.g. `/projects/project-9`).

---

## Capture workspace

Playwright and ffmpeg-static must be installed somewhere on the machine. Check first:

```powershell
# Preferred: reuse the existing capture workspace if present
Test-Path "C:\Users\kaixi\AppData\Local\Temp\capture_temp"
```

If missing, create it:

```powershell
$CAP = "C:\Users\kaixi\AppData\Local\Temp\capture_temp"
New-Item -ItemType Directory -Force $CAP
Set-Location $CAP
npm init -y
npm install playwright ffmpeg-static
npx playwright install chromium
```

All screenshot scripts go in `$CAP`. Screenshots are saved to `$CAP\screenshots\`.

---

## Phase 1 — Ensure the dev server is running

```powershell
# Check if something is already listening on 3000 or 3001
$port = if ((Get-NetTCPConnection -LocalPort 3001 -ErrorAction SilentlyContinue)) { 3001 }
        elseif ((Get-NetTCPConnection -LocalPort 3000 -ErrorAction SilentlyContinue)) { 3000 }
        else { $null }
```

If no server is up, start it in the background:

```powershell
Set-Location "c:\Users\kaixi\OneDrive\桌面\website\kaixiangkh7.github.io"
npm run dev   # run_in_background = true; wait for "Ready" in output
```

Note the port — use it as `BASE_URL` for all captures (e.g. `http://localhost:3001`).

---

## Phase 2 — Determine the reference

| Situation | Reference to use |
|---|---|
| New project page (`/projects/project-N`) | Existing project page on the same server, e.g. `/projects/project-9` |
| New homepage card | The homepage `/#projects` section (screenshot existing cards vs. new card) |
| New component in a section | The same section on the live site (https://kaixiangkh7.github.io) |
| User specified a URL | Use that URL directly |

If the user specified a reference route or URL, use it. Otherwise apply the defaults above.

---

## Phase 3 — Capture screenshots

Write and run `$CAP\capture_compare.mjs`. Capture at **two breakpoints** every time:

| Breakpoint | Viewport | Filename suffix |
|---|---|---|
| Desktop | 1440 × 900 | `_desktop.png` |
| Mobile | 390 × 844 | `_mobile.png` |

```js
// capture_compare.mjs
import { chromium } from 'playwright';
import { mkdirSync } from 'fs';

const CAP   = 'C:\\Users\\kaixi\\AppData\\Local\\Temp\\capture_temp';
const OUT   = `${CAP}\\screenshots`;
const BASE  = 'http://localhost:3001';          // adjust port
const ROUTE = '/projects/project-10';           // page to check
const REF   = '/projects/project-9';            // reference page

mkdirSync(OUT, { recursive: true });

const VIEWPORTS = [
  { name: 'desktop', width: 1440, height: 900  },
  { name: 'mobile',  width: 390,  height: 844  },
];

const browser = await chromium.launch({ headless: true });

for (const vp of VIEWPORTS) {
  // Subject
  {
    const ctx  = await browser.newContext({ viewport: { width: vp.width, height: vp.height } });
    const page = await ctx.newPage();
    await page.goto(`${BASE}${ROUTE}`, { waitUntil: 'networkidle', timeout: 30000 });
    await page.waitForTimeout(1500);
    await page.screenshot({ path: `${OUT}\\subject_${vp.name}.png`, fullPage: true });
    await ctx.close();
  }
  // Reference
  {
    const ctx  = await browser.newContext({ viewport: { width: vp.width, height: vp.height } });
    const page = await ctx.newPage();
    await page.goto(`${BASE}${REF}`, { waitUntil: 'networkidle', timeout: 30000 });
    await page.waitForTimeout(1500);
    await page.screenshot({ path: `${OUT}\\reference_${vp.name}.png`, fullPage: true });
    await ctx.close();
  }
}

await browser.close();
console.log('Screenshots saved to', OUT);
```

Run:
```powershell
Set-Location "C:\Users\kaixi\AppData\Local\Temp\capture_temp"
node capture_compare.mjs
```

---

## Phase 4 — Compare

Read both screenshots with the Read tool (Claude can view images). Examine them side by side for every item in the checklist below.

### Visual parity checklist

| Category | What to check | Expected |
|---|---|---|
| **Typography** | Body font family, weight | Nunito, `font-light` (300) |
| **Typography** | Section h2 | `text-2xl font-normal text-primary` (cyan `#00D4FF`) |
| **Typography** | Card title | `text-xl font-medium` |
| **Color** | Primary accents | Cyan `#00D4FF` only — no stray blue/teal/green |
| **Color** | Background | Pure white `#ffffff` |
| **Spacing** | Section padding | `py-24` (96px) top/bottom |
| **Spacing** | Container width | `max-w-[1400px]`, centered |
| **Images** | Hero height | `h-[50vh]`, `object-cover` |
| **Images** | GIFs in Key Screens | Full-width, no gray margins, `object-contain shadow-lg` |
| **Images** | Thumbnail | Dark-framed, fills card, no stretching |
| **Components** | Tags/badges | Pill style: `px-3 py-1 border border-border/50` |
| **Components** | Buttons | `PrimaryButton` / `SecondaryButton` — no ad-hoc `<button>` |
| **Components** | Card hover | `shadow-xl -translate-y-2` on card, `scale-110` on image |
| **Animations** | Transitions | `transition-all duration-300 ease-out` everywhere |
| **Mobile** | Stack layout | Single column, full-width images, correct padding |

---

## Phase 5 — Report differences

For each divergence found, record:

```
Element:   <what it is>
Subject:   <what it looks like now>
Reference: <what it should look like>
File:      <file path> line ~N
Fix:       <exact Tailwind class change or JSX change>
```

If everything matches → report "Visual parity achieved. No changes needed." and stop.

---

## Phase 6 — Apply fixes

For each divergence, apply the minimal change:

- Prefer changing or adding Tailwind classes over any other approach.
- Do **not** add inline `style={{}}`.
- Do **not** introduce new CSS rules — use classes from `app/globals.css` or standard Tailwind.
- One fix at a time if changes are interdependent; otherwise batch them.

---

## Phase 7 — Loop

After applying fixes, go back to **Phase 3** and re-capture. Repeat until the checklist passes completely at both breakpoints.

Stop when:
- All checklist items pass, OR
- A remaining difference is intentional/justified — call it out explicitly ("intentional: hero image is taller on this page because it uses a wide panoramic asset").

---

## Phase 8 — Report to user

Summarize:
1. What was different (list each item that needed fixing)
2. What was changed (file + line)
3. What was intentional/justified and left as-is
4. Screenshot paths for final reference (user can open them to verify)

---

## Common failure modes

| Symptom | Cause | Fix |
|---|---|---|
| Screenshot is blank / all white | Page not finished loading | Increase `waitForTimeout` to 3000ms; check the port is correct |
| Mobile screenshot cuts off | `fullPage: false` | Always pass `fullPage: true` |
| Colors look wrong in screenshot | headless rendering difference | Acceptable — compare structure/layout/class names in source instead |
| Reference route 404s | Server not serving that route | Check `lib/projects-data.ts` has the slug; restart dev server |
| `networkidle` timeout | Page has infinite polling | Switch to `waitUntil: 'domcontentloaded'` + longer `waitForTimeout` |
| Node ESM error | `.mjs` not supported | Add `"type": "module"` to capture_temp's `package.json` |
