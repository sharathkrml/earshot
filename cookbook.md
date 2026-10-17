<div align="center">

# 🍳 The earshot cookbook

**A field guide for giving eyes-and-ears to a text-only agent.**

*CPU only · no API key · two open models cached in `.cache/hf`*

</div>

---

## The 10-second pitch

You cannot hear `film.mp4`. This can:

```bash
earshot describe hit.wav                    # what is it?
earshot match hit.wav "a single key press" # does it match?
earshot timeline film.mp4                  # what happens, and when?
earshot sheet film.mp4 -o sheet.png        # …draw it
earshot forge whoosh --dir out             # make one and improve it by ear
```

That last one is the weird one. Hold on.

---

## 👁 Look first

This is your actual launch film, drawn by one command. Everything below this picture is the same idea in text form.

![sheet of earshot_launch.mp4](img/launch_sheet.png)

```bash
earshot sheet earshot_launch.mp4 -o sheet.png --width 1800
```

Reading it top to bottom:

| row | what it is |
|---|---|
| **frames** | the picture, sampled — for an agent that has eyes |
| **sections** | what kind of music/sound, with `?` when it is guessing |
| **events** | every onset, coloured by kind, tagged by CLAP. Pale + `?` = weak hint |
| **spectrogram** | the texture: where the energy lives in Hz |
| **waveform** | the shape: transients, decays, silence |
| **loudness** | LUFS over time; dashed red = integrated target |
| **sync / beats** | green diamond = onset within 1 frame of a cut, amber = within 100 ms, red = nothing |

And the one-line verdict at the top, which is the whole point:

> 41 events · **100% of cuts have an onset within 1 frame** (chance 20%) · **0% land on a strong accent** · **0% on the beat grid**

That is a mixing note. Something is on every cut; nothing hits hard; nothing lines up with the music.

### Zoom in, get more

![6 second zoom, detail high](img/launch_zoom.png)

```bash
earshot sheet film.mp4 --from 0 --to 6 --detail high -o zoom.png
```

Same 6 seconds, up to 96 events and 16 AudioSet labels instead of 12. `--detail low|normal|high` is the only speed dial that matters.

---

## 🔎 The five questions

### 1. What is this?

```bash
earshot describe whoosh.wav
earshot describe whoosh.wav --top 20 --json --out w.json
```

Calibrated tags by category (source, material, action, character), AudioSet labels, plus duration, LUFS, true peak,
attack/decay ms, brightness, bandwidth, band energy.

> Tags are hints. `?` = the excerpt is mostly bed. On busy mixes, believe `z` from `match` over this prose.

### 2. What are the facts?

```bash
earshot measure film.mp4 --light      # fast: no onsets, no pitch
earshot measure track.wav --music     # + tempo, beats, key
```

No models at all. LUFS, true peak, LRA, noise floor, clipped samples, attack/decay, brightness, bandwidth, onsets.
**Every number here is ground truth.** When you want a fact, ask `measure`, not `describe`.

### 3. Does it match what I asked for?

```bash
earshot match hit.wav "an impact" "a notification" "a whoosh"
earshot match hit.wav "an impact" --alts "a notification" "a click" "a riser"
earshot rank "a deep cinematic boom impact" ./sfx/*.wav --top 5
```

`z` = how far your description stands out among earshot's whole vocabulary, scored on that same sound:

| z | verdict |
|---|---|
| ≥ 3 | strong |
| 2.5 – 3 | plausible |
| 2 – 2.5 | weak |
| < 2 | no match |

`--alts` is a forced choice: uncalibrated, but much sharper at breaking ties.

### 4. Which moment?

```bash
earshot find field.wav "a single bird call" --top 5
earshot find field.wav "wind and leaves"  --kind texture
earshot find field.wav "footsteps"         --kind event --win 1.5
```

`--kind event` = onsets (something happened). `--kind texture` = sliding windows (something is going on).
`--kind auto` reads the query. Then feed the winner to anything else:

```bash
earshot describe field.wav --start 42.0 --end 42.6
```

That chain — **find → describe/match/compare on the trim** — is the single most useful habit here. A 4-minute
ambience confuses any model. The 0.4 s event inside it does not.

### 5. Which one, and why?

```bash
earshot ab v1.wav v2.wav --query "a bright logo sting"
earshot ab v1.wav v2.wav --refs ./pro_stings/
earshot compare my_pop.wav ./refs/pops/*.wav -k 5
```

`ab` = two files, in words: closer to the description, closer to the refs, and how they differ (louder, brighter,
longer decay). `compare` = one file against a set of *your* references: nearest neighbours, typicality (fits the
set, or an outlier), and the measured differences. That last sentence is a mixing note.

---

## 🎬 Listen through a whole film

```bash
earshot timeline film.mp4 --json --out t.json
earshot timeline film.mp4 --cues cues.json      # {"fps":60,"events":[{"f":90,"kind":"cut"}]}
earshot timeline film.mp4 --cues "1.5,3.2,5.0" # or just seconds
earshot timeline film.mp4 --from 30 --to 50    # zoom; results are cached
```

Returns a **listening transcript** — what a sound editor writes after one playback: events with time, level, kind and
tags; gaps and stop-downs; loudest window; sections; beat grid; picture cuts and sync stats.

Then `sheet` draws it (above). Give a sheet to a vision-capable agent and you have given it ears *and* eyes.

---

## 🏆 Is it any good?

```bash
earshot critique film.mp4                 # vs the 7 professional launch films it ships with
earshot critique film.mp4 --fast         # skip AudioSet + realness passes
```

Per metric: status `ok` / `low` / `high`, a weight, why it matters, and **concrete actions with timestamps**
("soften 0:04–0:06 by ~2 dB"). Overall: launch-grade / close / needs work / not launch-grade.

Score against films *you* like:

```bash
earshot profile ref_a.mp4 ref_b.mp4 ref_c.mp4 -o my_profile.json
earshot critique my_film.mp4 --profile my_profile.json
```

Median, p10–p90, min–max → your film is judged against a distribution, not one file. Most reusable thing in the
repo if you make videos.

---

## 🎹 Catch the fake

```bash
earshot realness render.wav --ast
earshot realness ./renders/ --json
```

Real recording, or code-synthesized sound (oscillators, filtered noise, beeps)? `p_real`, a verdict, evidence in
words, and per-window verdicts for anything over 6 s — which is how you find the one second that gave you away.

Held-out ROC-AUC **0.92** with whole sound families held out. ~1 in 5 clips get "uncertain", which is the honest
answer more often than an agent wants.

---

## 🔧 Make sounds in code, judged by ear

`forge` is an evolution strategy over DSP parameters: render candidates → score them by listening → keep the winners
→ mutate → repeat. Twenty recipes ship ready:

```
click  downlifter  glitch  hover  impact  keypress  notify  pluck  pop  popin
riser  shimmer  sting  subdrop  swipe  swish  toggle  typing  whoosh  zip
```

```bash
earshot forge --list
earshot forge pop --dir out/pop --budget 60 --pop 12 --variants 4
open out/pop/report.md out/pop/compare.png ; open out/pop/audition.wav
```

| in the folder | what it is |
|---|---|
| `pop_01.wav`, `pop_02.wav`… | the kept sounds, best first |
| `report.md` / `report.json` | every pick's score, what it sounds like, artifacts flagged |
| `compare.png` | the picks, drawn |
| `audition.wav` | all picks stitched in a row — one file to A/B against yourself |

### What came out

Ten keeps — 2 pops, 4 whooshes, 4 impacts — stitched into one file and drawn by earshot itself:

![ten forge sounds](img/forge_montage.png)

Read the labels: the models call the impacts *"ocean waves"* and *"thunderstorm"*. That is not a bug, it is the
honest answer for a 6-second boom with no picture — and it is exactly why you believe `z` over the prose.

Each search also drops a `compare.png`: spectrogram, envelope decay and 1/3-octave balance of every keeper.

![forge impact compare](img/forge_impact.png)

The title says *"made (left) vs nearest real reference"* and the right half is **empty** — this machine has no
reference recordings to compare against. Hand `--refs` a folder of real booms and that half fills in.

### Steer it with words

```bash
earshot forge impact --target "a deep cinematic boom impact" \
  --alts "an explosion" "a drum hit" "a sub drop" --dir out/impact --budget 200
```

`--alts` is the most satisfying flag in the tool: **describe the failure you keep hearing** and the search routes
around it.

### Give it real references

```bash
earshot forge whoosh --refs ~/sfx/real_whooshes/ --mode real --dir out/whoosh
```

> ⚠️ On this machine `forge --list` ends with `refsets: none on this machine` — the benchmark's licensed recordings
> can't be redistributed. So by default the critic runs **without** references and tells you so. It still works; with
> refs it gains typicality and realness as scoring dimensions.

### Dials

| flag | effect |
|---|---|
| `--budget N` | candidates scored (default 160) — the quality dial |
| `--pop N` `--variants N` | population / how many diverse winners |
| `--reps N` | renders per candidate, ranked by mean: 2 = half as lucky, 2× slower |
| `--robust N` | re-render each keeper on N new seeds: catches sounds that only work on one seed |
| `--mode real\|designed\|auto` | with refs, does realness *count* or just get reported |
| `--tone W` | weight of the model-free tonal-balance penalty |
| `--strategy es\|cma` | evolution strategy (default, beat CMA-ES here) or CMA-ES |
| `--seed S` | reproducibility |
| `--check DIR` | re-render every pick from `report.json`, compare to the WAVs within 1e-6 |

```bash
earshot forge --check out/pop      # run this once just to see the answer be yes
```

### One page for the whole catalogue

```bash
earshot gallery out/ --title "my sfx" -o sfx.html
earshot gallery out/ --embed-compressed -o portable.html     # one self-contained file
earshot gallery out/ --compare r2:'out/*/final_r2' r3:'out/*/final'
```

![forge gallery page](img/gallery.png)

Static page: players, spectrograms, scores, A/B reference rows. The fastest way to review 200 generated sounds with a
human — or hand the whole catalogue to another agent in one file.

---

## 🔬 Turn a recording into code

`fit` is the reverse: it listens to a real recording and writes you a **recipe** — a `.py` DSP file — that
resynthesizes something like it. Analysis by synthesis.

```bash
ffmpeg -i reference.mp4 -vn ref.wav
earshot fit ref.wav -o ref.py --kind auto        # auto | modal | spectral | sines | hybrid
earshot fit ref.wav --dir out/fit --report --describe
earshot fit --morph a.wav b.wav --amount 0.5 -o morph.py   # 0 = A, 1 = B
earshot fit ref.wav --forge ref.py --dir out/variations    # then let forge improve it
```

`modal` = resonant modes (the wood in a knock) · `spectral` = filtered partials · `sines` = pure tones ·
`hybrid --layers sines,modal,spectral`. `--report` adds the honest quality floor and the CLAP cosine between the
original and the resynthesis. Then open the `.py`, change one number, listen again.

---

## 🤖 Hand the ears to an agent

```bash
earshot mcp --tools                    # list all 13 tools, don't start the server
claude mcp add earshot --scope user -e EARSHOT_THREADS=3 -- \
  /path/to/earshot/.venv/bin/python -m earshot mcp --warm
```

`measure` `describe` `match` `rank` `find` `compare` `ab` `realness` `timeline` `sheet` `critique` `doctor`
`llm_critique` — after that the agent can listen on its own, which is the entire point of the project.

Two skills ship too, already linked into `~/.claude/skills/`: **`earshot`** (how to listen well) and **`real-sound`**
(a full licensed-recording soundtrack process for code-made films).

### Or in Python

```python
import earshot
earshot.describe('whoosh.wav')
earshot.find('typing.ogg', 'a single isolated key press')
earshot.timeline('soundtrack.wav', video='film.mp4')
from earshot import api        # api.rank, api.critique, api.realness_many, ...
```

Batch 400 files in **one** process instead of paying a model load 400 times.

---

## ⚡ Speeding up

| knob | effect |
|---|---|
| `EARSHOT_THREADS` | torch CPU threads (default `min(4, cores/2)`) |
| `--detail low\|normal\|high` | timeline/sheet model budget — the biggest real speedup |
| `--light` | `measure` without onsets/HPSS/pitch |
| `--fresh` | recompute a timeline (otherwise results are cached, so zooms are free) |
| `EARSHOT_PRECISION` | `bf16` (only on CPUs with native bf16) or `fp32` |
| `EARSHOT_HOME` | cache root: models, embeddings, decoded audio |
| `EARSHOT_OFFLINE=1` | never touch the network |
| `EARSHOT_DEBUG=1` | full traceback instead of the one-line error |
| `--json --out f.json` | on everything. Build your own dashboards out of these |

Measured here (8-core Apple silicon):

| | |
|---|---|
| CLAP into RAM | 5.4 s, once per process |
| one 10 s window | 1.0 s (CLAP) · 1.6 s (AudioSet) |
| forge search, budget 60 | ~1 min |
| 498 tests, no models | 2 m 29 s |

Models load **per process**. Batch into one command.

---

## 🗺 How good is it? (all of it measured, all of it here)

| claim | number | proof |
|---|---|---|
| "strong" match names the right kind of sound | **98%** (exact variant 61%) | [`eval/semantics/RESULTS.md`](eval/semantics/RESULTS.md) |
| real vs code-synthesized, families held out | ROC-AUC **0.92** | [`eval/realness/RESULTS.md`](eval/realness/RESULTS.md) |
| effects placed in a 60 s mix are detected | **93%** (89% of the quiet ones under music) | [`eval/timeline/RESULTS.md`](eval/timeline/RESULTS.md) |
| launch scorecard in range | **86%** of metrics | [`eval/launch/RESULTS.md`](eval/launch/RESULTS.md) |
| forge search | per-category, seed banks, blind tests | [`eval/forge/RESULTS.md`](eval/forge/RESULTS.md) |

Every headline number with the command that produced it: [`docs/VALIDATION.md`](docs/VALIDATION.md).
What it **cannot** judge: [`docs/LIMITS.md`](docs/LIMITS.md) — read that one first.

The eval data is licensed media and deliberately not in the repo. Point the scripts at your own with
`eval/local_paths.json` (gitignored) or env vars — see [`eval/README.md`](eval/README.md). To build a launch bar from
films you like you need none of that: `earshot profile` + `critique --profile`.

---

## 🩹 Gotchas I hit myself

**`OSError: laion/larger_clap_general does not appear to have a file named pytorch_model.bin`** — you ran a command
while the 1.4 GB download was still going, or a download died halfway.

```bash
rm -rf .cache/hf/models--*     # partial cache only
earshot doctor --load          # ~5 min, want "problems: []"
```

Root cause is a real bug: `earshot/models/clap.py:44` decides "models are cached" from *directory existence*, and a
download creates that directory immediately — so a half-downloaded cache flips earshot into permanently-offline mode
and the download fallback can never run.

**`NOTE: Redirects are currently not supported in MacOs`** — torch, harmless, every run.

**`earshot` not found** — it isn't on your PATH: `.venv/bin/earshot`, or `source .venv/bin/activate`.

**Tags look confidently wrong** — that's a busy mix, and the tool is telling you so. Facts: `measure`. Judgements: `z`.
Moment: `find`. Read [`docs/LIMITS.md`](docs/LIMITS.md) before trusting it with anything that matters.

---

## 💸 What you could actually build with this

**Sell the ears. Never the sounds.**

That is the whole business thesis in one line. Recordings are somebody else's property — this repo refuses to even
redistribute its own reference set (`eval/README.md`, and `--refs bench` is hard-refused). What you own is the
*measuring*: loudness, timing, sync, "is this the kind of sound you asked for", "does this sound synthetic". That is a
QA instrument, and QA is what gets paid for.

### The one constraint you cannot engineer around

You cannot ship audio whose provenance you cannot prove. So: `forge` output is yours by construction (it is DSP out of
a `.py`), everything else needs a licence you can point at. `docs/LIMITS.md` is blunt about the other half — it cannot
judge taste, emotion or licences, so never sell its verdict as taste. Sell the numbers.

### Sellable things, ranked by how fast money moves

| the product | what you run | who pays |
|---|---|---|
| **Launch-film sound QC** | `critique film.mp4 --profile my_profile.json` | every studio, agency and indie launcher |
| **Brand sound kit** | `forge <recipe>` per category → `gallery --embed-compressed` → one HTML link | startups shipping video weekly |
| **Sound in CI** | `critique --json` + an exit code on every render commit | teams with more videos than hours |
| **Buy-side pack screening** | `realness` + `compare` across a pack before you licence it | SFX libraries, marketplaces, buyers |
| **Rival teardown** | `timeline` + `compare --refs` across 20 competitor launch films | marketing / positioning consultants |
| **Agent tooling** | the MCP server + the two skills, packaged | anyone building AI video agents |

### The three that would move fastest

**1. A QC pass, sold per film.** Charge for the scorecard, not the mix: loudness, true peak, dynamic range, hit
density, sound-on-cuts, how close it sits to the reference profile, and the timestamps to fix. Setup is `earshot
profile` on seven films you respect, which makes your bar *yours* — and that is the part clients pay for. Ten films a
month at a few hundred each is a full income out of one profile JSON and one command.

**2. A brand sound kit, generated in an afternoon.** Pick a brand's 12 sounds (`click`, `hover`, `toggle`, `notify`,
`whoosh`, `impact`, `riser`, `sting`…), forge each with `--target` and `--alts` written in the brand's own words,
iterate on the `report.md`, ship the `gallery` page. Do it from the *reference* side: point `--refs` at the brand's
existing library and the search starts matching its house sound instead of the recipe's default. Licensing is a
non-issue because there is no licence. Priced per brand, not per sound.

**3. The before/after is the marketing.** This is the one nobody else in AI video has: `sheet` produces a picture of
a film that is legible, surprising and honest — *"100% of cuts have a hit, 0% land on an accent"* is a clip and a
headline at the same time. Before/after reels, scorecard screenshots, verdict cards: the instrument is also the
content engine, and the numbers are what make it shareable instead of cringe.

### What will not make money

- **Selling the verdicts as taste.** `z ≥ 3` means "this is the kind of sound you asked for", not "this is good". Sell
  the timestamps, let the human decide.
- **Shipping forge renders as premium library content.** Great for UI, motion, stingers, braams, transitions — real
  foley and real rooms are still code, and `realness` says so: about a fifth of clips come back "uncertain", and its
  own numbers show it calls 5.5% of real clips synthesized.
- **Uploading anything to `earshot llm`** without written consent. It sends audio to Google.

### Start on Monday

```bash
earshot profile a.mp4 b.mp4 c.mp4 d.mp4 -o brand_profile.json        # your bar
earshot critique client_film.mp4 --profile brand_profile.json        # the deliverable
earshot forge pop --target "our brand's pop: round, warm, no click" \
  --alts "a wet mouth click" "a cork" --dir out/brand --budget 300
earshot gallery out/ --embed-compressed -o brand_kit.html            # the catalogue
```

---

## 🗺 Where to go next

1. `find` something in a long recording, then `describe --start/--end` the winner — watch how much sharper it gets.
2. `ab` your two favourite takes of one sound. Stop guessing which is better.
3. `forge impact --alts "an explosion" "a drum hit" --budget 400`. Read the report. It gets more honest.
4. `fit` a sound you love, open the generated `.py`, change one number.
5. `sheet` a film, hand the picture to a vision model, ask it to critique the *look* of the sound.
6. `critique` the same film, then `profile` the films you love and score it again.
7. Register the MCP server and stop doing any of this by hand.
8. `realness --ast` across a folder of your renders and find out which family of sounds you keep faking.

<div align="center">

*measure for facts · `z` for judgements · `find` for the moment · `forge` for making*

</div>
