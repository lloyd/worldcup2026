# CLAUDE.md — World Cup 2026 Prediction League site

Operating manual for this repo. Read this fully before changing anything. It is written so a
future session can do the two common jobs end‑to‑end **from a single dropped‑in file**:

1. **Drop in a screenshot / PDF / printout of the group chat → update the charts.**
2. **Drop in a player photo → (re)generate that player's chibi.**

Both jobs end with: commit to `main` → auto‑deploy to `gh-pages` → verify live.

- **Live site:** https://lloyd.github.io/worldcup2026/
- **Repo / remote:** `git@github.com:lloyd/worldcup2026.git` (public)
- **What it is:** one self‑contained `index.html` (all CSS, data, JS, and hand‑built SVG charts
  inline; no external/runtime dependencies) telling the story of the Yarris × Hilaiel family's
  iMessage World Cup‑2026 score‑prediction league. Plus `assets/chibi/*.webp` character art.

---

## 1. Repo layout

```
index.html                    EVERYTHING: <style>, the data object `const D`, the JS that
                              renders charts, the SVG generators. Edit this to change data/visuals.
assets/chibi/<name>.webp      One chibi per player (jon, george, jake, lainie, gabe, sophie, lloyd).
.github/workflows/deploy.yml  On push to main → peaceiris publishes repo root to the gh-pages branch.
.nojekyll                     Tell Pages not to run Jekyll.
.gitignore                    Keeps secrets + raw photos OUT of this public repo (see §6).
README.md                     Public-facing blurb + publish instructions.
CLAUDE.md                     This file.
```

There is no build step. `index.html` is the deployed artifact verbatim.

---

## 2. The data model (all inside `index.html`, the `const D = { … }` object)

The single source of truth for the charts is `D`. Grep anchors: `const D = {`, the cast loop
`/* ---------- CAST`, the podium loop `/* ---------- hero scoreboard`.

| Field | Meaning |
|---|---|
| `D.players` | array of the 7 names, canonical order. |
| `D.color` | name → hex (each player's signature color; used by every chart + their kit). |
| `D.cum[name]` | **cumulative** points after each matchday. Index `0` = start (0), `1..N` = end of MD1..MD_N. **This is the backbone — everything else derives from it.** |
| `D.dayLabel`, `D.dayDate` | `["","MD1",…]` and `["","Jun 11",…]`, indexed to match `D.cum`. |
| `D.fp[name]` | per‑player card data: `role`, `pts` (final total), `rank`, `avg` (avg goals predicted), `draw` (draw rate), `fav`/`favN` (signature scoreline + count), `quote`, `fp` (one‑line stat blurb, may contain `<b>`). |
| `D.boldest` | list of `{g, by, d, m, v}` — highest‑goal single predictions (`v:"whiff"` shows a red tag). |
| `D.chat` | banter hall of fame: `{s, me, t}` (`me:true` = Lloyd = right/blue bubble). |

**Derived at runtime (do NOT store):** per‑day points = `D.cum[p][d] - D.cum[p][d-1]` (the heatmap
and spotlight compute this live). Final standings, podium, lineup order = sort by `D.fp[p].rank`
(or by `D.cum[p][last]`).

### The reconciliation invariant (must always hold)

The family's official scores are **Jake's green‑header spreadsheet screenshots**
("Ranked Score / Ranked Players / Place"). Those numbers ARE `D.cum[*][lastIndex]`.

> For every matchday, the per‑day deltas must sum **exactly** to each player's final total.
> i.e. `sum(D.cum[p][d]-D.cum[p][d-1] for d in 1..N) == D.cum[p][N]` (trivially true if `cum` is
> internally consistent). When you add a new day, the new `cum` values come straight from the
> newest standings screenshot, so the deltas are automatically correct. **Never hand‑invent
> per‑day points** — always take cumulative totals from a real standings screenshot.

### Scoring rules (locked in the chat, shown in the "Rulebook" section)

- **2 pts** — correct result (you called the winner, or the draw).
- **3 pts** — correct result **+ one team's exact score**.
- **5 pts** — perfect scoreline.
- Multipliers (still dormant — everything so far is group stage, scored ×1):
  **×2** knockouts · **×3** quarters · **×4** final.

When the knockouts start, the deltas will already reflect multipliers (they come from the
spreadsheet), so no scoring math is needed here — but update the "Rulebook" copy if a multiplier
goes live, and consider noting it in the narrative.

---

## 3. Player roster (identity, color, role, chibi outfit + prop)

Keep these stable so chibis stay a consistent set. `name.toLowerCase()` is the asset filename.

| Name | Color (hex) | Role label | Identity notes (for likeness) | Kit color | Chibi prop / pose |
|---|---|---|---|---|---|
| Jon | `#F5C518` gold | The Closer | older man, glasses, grey hair/beard; reigning champ | gold | small gold crown + holds a trophy |
| George | `#FF3D7F` pink | The Frontrunner | young man, short dark hair | hot pink | raised "well, actually" finger + microphone |
| Jake | `#46C7E8` cyan | The Statistician | young man | cyan/teal | clipboard with a tiny chart + pencil |
| Lainie | `#B388FF` violet | The Matriarch | woman, warm smile (the mom) | purple + scarf | coffee mug, one hand cheering |
| Gabe | `#FF9F45` orange | The Commissioner | **teenage boy, curly dark hair** | orange | referee whistle on a lanyard + phone |
| Sophie | `#5FD08A` mint | The Rookie ("Frogaroo") | young woman | mint green | friendly wave + tiny green frog pin |
| Lloyd | `#6E8BFF` periwinkle | The Data Nerd | man, full dark beard (the chat owner; his texts are the right/blue bubbles) | periwinkle indigo | round glasses + laptop showing a rising chart |

---

## 4. JOB A — Update the charts from a new chat screenshot / PDF / printout

Trigger: the user drops a chat image, a multi‑image set, or a PDF (e.g. on `~/Desktop`).

1. **Get readable images.** For a PDF, render pages to PNG:
   ```bash
   python3 -c "import fitz; d=fitz.open('FILE.pdf'); [p.get_pixmap(matrix=fitz.Matrix(2,2)).save(f'/tmp/p{i+1:02d}.png') for i,p in enumerate(d)]"
   ```
   For a single screenshot, just read it. (Tools available: `pymupdf`/`fitz`, `pdftotext`, `PIL`.)
2. **Find the newest standings screenshot** (green header: *Ranked Score / Ranked Players / Place*).
   Read the exact integer total for **every** player. Zoom/crop if unsure — these numbers are the
   whole ballgame, so verify them. This is the new matchday's `D.cum` column.
3. **Anchor it in THIS World Cup only.** Ignore any Copa‑America‑2024 content (dated "Jul 2024").
   The WC‑2026 chat runs from Jun 11, 2026. Each match‑day adds one index to `D.cum`.
4. **Append the new day** to every `D.cum[name]` array (same order as `D.players`), and extend
   `D.dayLabel` (`"MD9"`) and `D.dayDate` (the date). If a screenshot is mid‑day/partial, label it
   honestly.
5. **Refresh derived copy** that names specific numbers/story beats:
   - `D.fp[name].pts` and `.rank` (new totals + ranking).
   - Hero podium = top 3 (`/* ---------- hero scoreboard`): the `order`, `tag`, and the count‑up
     `data-to` values come from `D.cum`, so they update automatically — but re‑check the `tag`
     strings and any hard‑coded headline numbers.
   - Narrative sections that cite figures (e.g. "The Race", "Daily Damage", "The Swing / MD6"):
     the section headings and call‑outs are hand‑written prose. Update them if the story changed
     (a new lead change, a new biggest day, etc.). Grep the section eyebrows `01 — `, `02 — `, …
   - New banter worth featuring → add to `D.chat`.
   - New predictions (optional) → only matter for `D.fp` fingerprints (`avg`, `draw`, `fav`).
     Recompute from the predictions if you want them exact; otherwise leave fingerprints as‑is.
6. **Validate** before shipping:
   - Final totals (`D.cum[*][last]`) match the screenshot exactly.
   - Each day's deltas are non‑negative and look sane (nobody loses points).
   - Ranks in `D.fp` match the sorted finals.
7. **Render‑verify, then ship** (see §7 and §8).

> Heavy re‑import (rebuilding the whole dataset from a long chat export): fan out vision agents
> over the page images to extract standings tables, predictions, results‑revealing banter, and
> quotes; then reconcile per §2. Cross‑check that computed deltas reproduce the screenshot totals.

---

## 5. JOB B — (Re)generate a chibi from a player photo

Trigger: user drops a photo (typically `~/Desktop/players/<name>.<ext>`) and says make/redo a chibi.
We use **gpt-image-2** (image‑edit, likeness‑preserving) via the `one` repo's CLI + the **gototo-dev**
GCP creds. Quality **`low`** is the house default — fast and looks great.

**Prereqs (one‑time per session):**
```bash
# OpenAI key from the gototo-dev secret manager (needs gcloud ADC; account lloyd@goheadlands.com).
export OPENAI_API_KEY="$(gcloud secrets versions access latest --secret=openai-api-key --project=gototo-dev)"
# …or: source ~/dev/one/go/env.sh   (also sets many other secrets)

# Build the image-edit CLI from the one repo:
( cd ~/dev/one && go build -o /tmp/stickerexp ./go/cmd/stickerexp )
# stickerexp flags: -ref <photo> -prompt <prompt.txt> -out <out.png> -quality low|medium|high
# (It hardcodes size 1024x1024, opaque background, PNG. cmd/imgen is the text-only generator.)
```

**Steps:**
1. **Normalize the photo** (EXIF‑rotate, downscale ≤1280):
   ```bash
   python3 -c "from PIL import Image,ImageOps; im=ImageOps.exif_transpose(Image.open('SRC')).convert('RGB'); im.thumbnail((1280,1280)); im.save('/tmp/ref.png')"
   ```
2. **Write the prompt** to a file. Use the shared template + the player's tail from §3. Template:
   ```
   Turn the person in this photo into a cute CHIBI mascot sticker. Keep their face, hairstyle,
   facial hair, skin tone and glasses clearly recognizable so the sticker obviously looks like THIS
   person. Chibi proportions: big round head about half the total height, large friendly expressive
   eyes, small simple rounded body, full body standing in a slight 3/4 pose, little soccer cleats.
   Soft cel shading, thick clean dark outline, bright flat colors, thick white die-cut sticker
   border around the whole character. Background: ONE solid flat light-gray (#cfd3d8) color filling
   the entire background — no scenery, no gradient, no transparency checkerboard, no text. Character
   centered, full body in frame with a little margin. <PER-PLAYER TAIL: kit color + role prop>
   ```
   ⚠️ gpt-image-2 ignores "transparent background" (it paints a literal checkerboard). **Always**
   ask for a solid flat background and key it out in step 4.
3. **Generate** (parallelize when doing several — one `&` per player):
   ```bash
   /tmp/stickerexp -ref /tmp/ref.png -prompt /tmp/<name>.txt -out /tmp/<name>.png -quality low
   ```
4. **Key out the background → transparent, trim, export webp** to the repo:
   ```python
   from PIL import Image; from collections import deque
   def keyout(path):
       im=Image.open(path).convert("RGBA"); w,h=im.size; px=im.load(); bg=px[0,0]
       near=lambda a,b,t=60: all(abs(a[i]-b[i])<=t for i in range(3))
       seen=bytearray(w*h); q=deque()
       for s in [(0,0),(w-1,0),(0,h-1),(w-1,h-1)]:
           i=s[1]*w+s[0]
           if not seen[i]: seen[i]=1; q.append(s)
       while q:
           x,y=q.popleft(); r,g,b,a=px[x,y]
           if near((r,g,b),bg):
               px[x,y]=(r,g,b,0)
               for dx,dy in((1,0),(-1,0),(0,1),(0,-1)):
                   nx,ny=x+dx,y+dy
                   if 0<=nx<w and 0<=ny<h:
                       i=ny*w+nx
                       if not seen[i]: seen[i]=1; q.append((nx,ny))
       bb=im.getbbox(); return im.crop(bb) if bb else im
   im=keyout("/tmp/<name>.png")
   H=460; im=im.resize((int(im.width*H/im.height),H), Image.LANCZOS)
   im.save("assets/chibi/<name>.webp","WEBP",quality=82,method=6)   # ~30KB
   ```
   Flood‑fill works because the chibi has a thick white die‑cut border, so the fill stops at the
   border and never eats interior whites (eyes, kit trim).
5. **Verify the likeness** (read the new webp / composite it on a dark panel) before shipping. Redo
   the prompt if it drifts. Then §7 + §8.

The page wires chibis automatically: cast cards, the team‑lineup strip, and the top‑3 podium all
build `src="assets/chibi/${name.toLowerCase()}.webp"` in JS — just dropping a correctly‑named webp
into `assets/chibi/` updates the site.

---

## 6. Secrets & privacy — do NOT leak to this public repo

- **Never commit** the OpenAI key or raw family photos. `.gitignore` already excludes `.env*`,
  `*.key`, `characters/raw/`, `photos/`, `secrets/`. Keep generation scratch outside the repo
  (e.g. `/tmp` or the session scratchpad) or under an ignored dir.
- The key lives only in `gcloud` secret manager (`gototo-dev`) — fetch it into an env var per
  session; don't echo it, don't write it to disk in the repo.
- The committed chibis are stylized cartoons (intended for the public page). The original photos
  are not — keep them local.

---

## 7. Local render‑verify (always look before shipping)

Headless Chrome is available. Use `--force-prefers-reduced-motion` so the scroll‑reveal sections
(`.rv`) are all visible in a static full‑page shot (otherwise they stay hidden in headless):
```bash
CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
"$CHROME" --headless=new --disable-gpu --hide-scrollbars --force-prefers-reduced-motion \
  --virtual-time-budget=4000 --window-size=1280,7800 \
  --screenshot=/tmp/page.png "file://$PWD/index.html"
```
Also sanity‑check the inline JS parses:
```bash
node -e "const fs=require('fs');const m=fs.readFileSync('index.html','utf8').match(/<script>([\s\S]*)<\/script>/);new Function(m[1]);console.log('JS OK')"
```

---

## 8. Deploy & verify live

```bash
git add -A
git commit -m "…"            # commits are co-authored; see existing history for the trailer
git push origin main          # → .github/workflows/deploy.yml publishes root to gh-pages
```
Then confirm it's actually live (Pages CDN lags the Action by ~30–60s):
```bash
gh run list --repo lloyd/worldcup2026 --limit 3                 # deploy + pages-build should be success
gh api repos/lloyd/worldcup2026/pages/builds/latest -q .status  # → "built"
curl -s -o /dev/null -w "%{http_code}\n" --retry 20 --retry-delay 6 --retry-all-errors \
  https://lloyd.github.io/worldcup2026/assets/chibi/jon.webp     # → 200
```
For a definitive check, headless‑screenshot the **live URL** (not just the local file) and look.

Pages source = the **gh-pages branch** (root). The workflow keeps gh-pages in sync from main; you
normally never touch gh-pages by hand.

---

## 9. Gotchas / conventions

- **`index.html` is body content + a `<head>` wrapper.** It's a fully standalone document (works on
  Pages with relative `assets/` paths). A separate copy was once published as a claude.ai *Artifact*,
  but that sandbox blocks local image files (CSP) — **the GitHub Pages site is canonical** for the
  chibis.
- **Charts are hand‑built inline SVG**, generated by the JS in `index.html` from `D`. No chart libs.
- **Reveal animation:** sections use `.rv` + an IntersectionObserver. Respects
  `prefers-reduced-motion`. There's a small timeout fallback so nothing stays hidden if IO misfires.
- **Player → file mapping is `toLowerCase()`** (`Jon` → `assets/chibi/jon.webp`).
- **Chibi background:** request a solid flat bg + key it out; never rely on the model for real
  transparency.
- Keep the **palette** intentional: floodlit‑pitch dark green ground (`--ground:#0B1511`), pink‑boots
  accent (`--accent:#FF3D7F`, a literal running joke in the chat), spreadsheet‑gold for 1st
  (`--gold:#F5C518`). Numbers everywhere are tabular monospace (the "data nerd" identity).
```
