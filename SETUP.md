# How this profile is built

Not rendered on the profile — this file is the build notes for `README.md`.

Adapted from Gargi Bhardwaj's *"Build a GitHub profile that isn't a wall of badges"*
([github.com/gargibhardwaj24](https://github.com/gargibhardwaj24)). The three generators in
`scripts/` are hers; the palette, the portrait pipeline, the content and the layout are not.

## What still needs doing on GitHub

The local half is finished and committed. These are the switches only you can flip:

- [ ] Push to `https://github.com/tabishaliansari/tabishaliansari` (the repo already exists — its
      current README is the old emoji one, and this replaces it).
- [ ] **Settings → Actions → General → Workflow permissions → Read and write.** All three
      workflows commit files back; without this the charts workflow fails at `push` and the snake
      workflow can't create its `output` branch at all.
- [ ] **`METRICS_TOKEN` secret.** Generate at `github.com/settings/tokens` →
      **Generate new token (classic)** — a fine-grained token will *not* work. Scope: `read:user`
      (add `repo` if you want private contributions counted). Save it under
      Settings → Secrets and variables → Actions with the name `METRICS_TOKEN` exactly.
- [ ] Actions tab → run all three workflows once by hand (`Run workflow`).
- [ ] Check the profile in **both** GitHub themes (Settings → Appearance) and on your phone.

Until the Metrics workflow completes once, `assets/metrics.isocalendar.svg` and
`assets/metrics.achievements.svg` are broken images, and the snake URLs 404 because the
`output` branch doesn't exist yet. Both are expected on day one.

With no `METRICS_TOKEN` the stat card renders 3 tiles instead of 6 — it degrades, it doesn't fail.

## Regenerating the art

```bash
pip install pillow
```

### Portrait

Rendered in colour: `--color` takes each dot's fill from the photo itself. Two things the
guide's recipe doesn't cover had to be solved for this particular photo.

**The background.** It's a lectern shot against a bright LED wall. Rendered straight, the wall
dominates and `--equalize` measures the curtain instead of the face, so the face collapses into
one flat white mass. Cutting the background out turns the alpha channel into a subject mask, so
`--equalize` measures only him and nothing is drawn outside the silhouette:

```bash
pip install pillow rembg onnxruntime
python -c "
from PIL import Image; from rembg import remove, new_session
Image.open('me.JPG').crop((963,0,2263,1300)).save('me-crop-tight.png')
remove(Image.open('me-crop-tight.png'), session=new_session('u2netp')).save('me-cut-tight.png')"
```

`u2netp` is the 4.7 MB model — rembg's default is a 1 GB download and the small one is plenty
for a mask this coarse.

**Both themes from one file.** The guide says `--color` works on both themes because the fills
come from the photo. That holds for an evenly-lit photo on a light background; it does not hold
here. Dot *size* tracks brightness, so on a white page his black hair and navy shirt become
tiny dots that vanish and you get a face with no head. `--bg` fixes it by giving the portrait
its own panel, so the same file reads identically in either theme:

```bash
python scripts/dotify.py me-cut-tight.png -o assets/portrait \
  --cols 116 --equalize --detail 0.9 --contrast 0.95 --reveal --color --bg "#0a0a0a"
```

That writes a single `assets/portrait.svg` — no `<picture>` block needed, and the panel colour
matches the portfolio site's dark surface.

`--detail 0.9 --contrast 0.95` instead of the guide's `--detail 0.5`: at 0.5 his face still
saturated. Cols 116 renders at 396 KB and holds up at the 420 px the README displays it at;
88 was fine at 300 px but goes visibly coarse when scaled past that.

Two rejected alternatives, in case you want to revisit:

- **No `--bg`.** Great on dark, weak on light — vivid face with no silhouette behind it.
- **A `--color --invert` file for the light theme.** Strong silhouette, but almost no colour
  survives, because the colour lives in the highlights and halftone puts little ink there.

### Radar and cards

```bash
python scripts/radar.py --data assets/skills.json -o assets/radar --values
python scripts/cards.py --user tabishaliansari --projects assets/projects.json --out assets
```

Edit `assets/skills.json` (self-rated, 0–100) and `assets/projects.json` (featured repos) —
the Charts and cards workflow redraws both on every push to either file, and daily at 03:30.

## Deliberate departures from the guide

- **No live language radar.** 89.7% of the language bytes across these repos are Jupyter
  Notebook, because committed notebook outputs are base64 images. The chart came out as one
  spike with six dots around it, and excluding notebooks was worse — it made a data engineer
  look like a JavaScript developer. The step is commented out in `.github/workflows/radar.yml`
  with instructions; strip outputs with `nbstripout` and it becomes worth having.
- **`cards.py` accepts `owner/repo`.** Two of the four featured projects live in orgs
  (`DA-workshop-101`, `Vibe-Coding-by-Tabish`) and the original only looked at repos owned by
  `--user`. An entry can also set `"name"` to override the card title and filename, which is
  how the Alzheimer's repo becomes `ADAPT`.
- **`ADAPT` has `"language": "Python"` pinned.** The repo's byte count is dominated by a
  bundled JS frontend; the model, training and inference API are Python. Stars and forks are
  still live.
- **Monochrome, not green.** The palette in all three scripts is bone `#ededed` on dark and ink
  `#0a0a0a` on light, matching the portfolio site. Accent colour lives in exactly one place per
  script — the `THEMES` dict at the top.
- **Metrics assets are single files, in the action's default styling.** `lowlighter/metrics`
  has **no theming input** — there is no `config_theme`, and nothing in its `action.yml`
  controls light/dark. An earlier version of this workflow passed `config_theme`, which the
  action rejects as an unsupported option; it failed on the first step every run and produced
  no files at all. The only theming lever the action does expose is `extras_css`, if the default
  card ever needs to be darkened to match the portrait's `#0a0a0a` panel.
- **Social badges: brand colours, no logos.** Simple Icons no longer carries `linkedin` or
  `yahoo`, so `logo=linkedin` renders a badge with no icon at all. A row where two of five
  silently lose their logo reads as broken, so none of them carry one. Colours are brand-accurate
  except Vercel's and X's pure `#000000`, which is invisible against GitHub's `#0d1117` dark
  canvas — both are lifted to `#24292F`, which reads as black on light and as a visible chip on
  dark. Portfolio sits first so the two dark chips aren't adjacent.
