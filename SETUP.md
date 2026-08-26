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

Until the Metrics workflow completes once, `assets/metrics.isocalendar-{dark,light}.svg` and
`assets/metrics.achievements-{dark,light}.svg` are broken images, and the snake URLs 404 because the
`output` branch doesn't exist yet. Both are expected on day one.

With no `METRICS_TOKEN` the stat card renders 3 tiles instead of 6 — it degrades, it doesn't fail.

## Regenerating the art

```bash
pip install pillow
```

### Portrait

The source photo is a lectern shot with a bright LED wall behind it. Rendered straight, the
background dominates and `--equalize` measures the curtain instead of the face, so the face
collapses into one flat white mass. Two extra steps fix it, and both matter:

```bash
# 1. cut the background out - the alpha channel becomes a subject mask, so --equalize
#    measures only him and nothing is drawn outside the silhouette
pip install rembg onnxruntime
python -c "
from PIL import Image; from rembg import remove, new_session
Image.open('me.JPG').crop((963,0,2263,1300)).save('me-crop-tight.png')
remove(Image.open('me-crop-tight.png'), session=new_session('u2netp')).save('me-cut-tight.png')"

# 2. the light theme needs the subject's tones inverted. Dot size tracks brightness, so on a
#    white page a bright face becomes big dark dots - a photographic negative. Inverting the
#    subject (and NOT using --invert, which would also invert the black background into a full
#    grid of dots) puts the ink on his hair and shirt where it belongs.
python -c "
from PIL import Image, ImageOps
im = Image.open('me-cut-tight.png').convert('RGBA'); a = im.split()[3]
inv = ImageOps.invert(Image.merge('RGB', im.split()[:3])).convert('RGBA'); inv.putalpha(a)
inv.save('me-cut-tight-inv.png')"

# 3. render each theme from its own input
python scripts/dotify.py me-cut-tight.png     -o assets/portrait --cols 88 --equalize --detail 0.9 --contrast 0.95 --reveal
python scripts/dotify.py me-cut-tight-inv.png -o assets/portrait --cols 88 --equalize --detail 0.9 --contrast 0.95 --reveal
# keep portrait-dark.svg from the first run and portrait-light.svg from the second
```

`--detail 0.9 --contrast 0.95` instead of the guide's `--detail 0.5`: at 0.5 his face still
saturated. Cols 88 keeps both files ~232 KB; 100 pushed the light one past 500 KB.

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
- **Metrics assets are rendered twice.** `lowlighter/metrics` draws a light card by default,
  which floats on a dark profile. The isocalendar and achievements steps each run twice, with
  `config_theme: light` and `config_theme: dark`, so the `<picture>` blocks have something real
  to choose between. `metrics.habits` and `metrics.languages` are still generated but unused.
- **Text-only social badges.** Simple Icons no longer carries `linkedin` or `yahoo`, so
  `logo=linkedin` renders a badge with no icon. A row where two of five silently lose their
  logo reads as broken, so none of them have one.
