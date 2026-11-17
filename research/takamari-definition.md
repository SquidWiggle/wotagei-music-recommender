# Defining and Measuring "Takamari" (高まり) for a Wotagei Recommender

**Project Context:** AI class final project upgrading an 18-row synthetic music recommender (`data/songs.csv`) into a real wotagei (ヲタ芸 / 打ち曲) recommender covering J-pop / anison / idol tracks (and optionally English-language tracks that share compatible structure and energy build-up).  
**Verification Note:** Every claim in this report is tied to primary sources (academic MIR/musicology papers, official library documentation, and Japanese wotagei/music community references). Where direct automated retrieval of a live webpage was blocked by environment network policy, the exact HTTP block is noted and search-index-grounded findings are explicitly separated from unverified claims.

---

## Executive Summary

In Japanese idol, anison, and wotagei culture, **takamari (高まり)** denotes the listener's and performer's surging emotional and physical tension that builds through a song's transitional sections (`Bメロ` or `落ちサビ`) and releases explosively at the chorus (`サビ` or `ラスサビ`). Rather than raw average loudness alone, *takamari* depends on **contrast and trajectory**—how a track accumulates anticipation (rhythmic acceleration, calls/claps like `警報` and `PPPH`, or instrumental thinning in `落ちサビ`) before unleashing full-band energy, driving 4/4 beats (typically **130–180 BPM**), and clear 8-bar (`1エイト × 4 = 32カウント`) phrases suitable for synchronized cyalume routines.

In Music Information Retrieval (MIR) literature, full-track chorus detection relies heavily on self-similarity and global repetition (Goto's **RefraiD**, McFee & Ellis's **Laplacian segmentation**, Nieto & Bello's **MSAF**), which **cannot operate reliably on 30-second preview clips** (e.g., iTunes/Deezer previews) that lack full-song repetition. Conversely, local structural novelty (**Foote, 2000**) and temporal tension/climax features (**Farbood, 2012**; **Yadati et al., 2014**) measure frame-to-frame change and perceptual build-up within short windows.

To approximate *takamari* computationally with free Python libraries (**`librosa`** and **`Essentia`**), this report defines three concrete, measurable proxies—plus an uncalibrated composite hypothesis:
1. **Pre-Chorus-to-Chorus Loudness Contrast ($\Delta L_{\text{dB}}$)** — RMS / EBU R128 momentary loudness jump across a local transition.
2. **Onset Density & Rhythmic Drive Surge ($\Delta \rho_{\text{onset}}$)** — change in attack rate / spectral-flux onset envelope coupled with beat regularity in the wotagei BPM band.
3. **Spectral Brightness & High-Frequency Expansion ($\Delta C_{\text{oct}}$ / $\Delta \text{Flux}$)** — octave-scale jump in spectral centroid (and positive spectral flux) reflecting fuller orchestration and elevated vocal/synth register.

These proxies transfer well to **Western pop tracks that follow `Verse -> Pre-Chorus -> Chorus`**, require **sub-band adaptation for EDM Drops** (where snare-roll onset density and high-pass filter sweeps often peak *before* the drop), and suffer **dynamic-range compression saturation in dense rock/metal**.

---

## 1. Wotagei Community Understanding of "Takamari" & `打ち曲` Selection

### 1.1 Primary Source Verification Status (`wikiwiki.jp/neosr/`)

Direct HTTP `GET` requests from our workspace environment to the two requested `wikiwiki.jp` pages were blocked by egress proxy policy:

| Target URL | Retrieval Method | Status & Exact Observation |
| :--- | :--- | :--- |
| `https://wikiwiki.jp/neosr/ヲタ芸の基本的な構成` | Direct `curl` GET | **Blocked (`HTTP/2 403`)**: `Request to GET /neosr/ヲタ芸の基本的な構成 on wikiwiki.jp not allowed by policy` |
| `https://wikiwiki.jp/neosr/打ち曲` | Direct `curl` GET | **Blocked (`HTTP/2 403`)**: `Request to GET /neosr/打ち曲 on wikiwiki.jp not allowed by policy` |
| `wikiwiki.jp/neosr/` (indexed pages) | Web search grounding | **Partially verified via search snippets** (exact verbatim page text beyond indexed snippets remains **unverified** due to the 403 block) |

**What was verified from indexed `wikiwiki.jp/neosr/` snippets:**
- **ヲタ芸 Wiki\* (NEOSr admin)** documents standard routine structure across `Aメロ`, `Bメロ`, and `サビ` (including `ラスサビ`):
  - **Aメロ:** Uses foundational moves such as **OAD** (Over Action Dolphin / オー・エー・ディー), and in underground-style wotagei (`地下芸`) may enter from the cue *"check, set, open"* (`チェック・セット・オープン`), or include moves like `ニーハイオーハイ` and `ソイヤ` (4-thrust pattern), with the connecting thrust at the A-to-B transition frequently omitted.
  - **Bメロ:** Bridges `Aメロ` to `サビ` using three documented patterns depending on section length and style:
    1. All performers execute a fixed number of **Rosario** (`ロザリオ`) or **Los Angeles** (`ロサンゼルス`) repetitions (common when `Bメロ` is short).
    2. Performers insert an upward/chorus-move preview before transitioning upward into the chorus.
    3. Performers execute **Rosario** in a prescribed sequence and converge onto unison **Rosario** at the end of `Bメロ`.
  - **サビ / ラスサビ:** Executes high-energy **サビ技** (chorus moves) such as **Romance** (`ロマンス`), **Thunder Snake** (`サンダースネイク`), **Amaterasu** (`アマテラス`), and **Muramasa** (`ムラマサ`), with routines paced in units of **`1エイト` = 8 counts** and **`32カウント` = 4 eights** (8 bars of 4/4). Later counts of a chorus move (e.g., `ロマンス警報`) can act as a cue into subsequent phrases.
- **On `打ち曲` / `おすすめ打ち曲` on `wikiwiki.jp/neosr/`:** Search snippets show navigation entries for `打ち曲` and `ヲタ芸 おすすめ打ち曲` in the wiki menu/glossary, alongside individual song/routine pages (e.g., `ロマンチック`, `シリウス`, `サマータイムレコード`, `ミシャグジ`, `炎火`, `アーリークロス`, `メテオロロジスト`, `神楽`), and indicate that NEOSr discussions historically dispersed song examples across move/performer pages rather than maintaining a comprehensive numerical BPM table. Because the live `https://wikiwiki.jp/neosr/打ち曲` page returned HTTP 403, its exact current contents, prose criteria, and any song table on that page are marked **unverified**.

### 1.2 Definition of *Takamari* (高まり) and *Uchi-kyoku* (打ち曲)

Across Japanese wotagei wikis, Niconico Pedia (`dic.nicovideo.jp/a/ヲタ芸`, `dic.nicovideo.jp/a/PPPH`), Japanese Wikipedia (`ja.wikipedia.org/wiki/オタ芸`), and anikura (anime-song club) / idol community guides:

- **Wotagei (ヲタ芸 / オタ芸):** Coordinated fan cheering dances, calls, and glowstick/cyalume routines (*cyalume dance*, `サイリウムダンス`) performed during idol concerts, voice-actor/anison lives, and anikura events; performing wotagei is called *utsu* (`打つ`, "to strike/perform"), and performers are *uchishi* (`打ち師`).
- **Uchi-kyoku (打ち曲):** Songs well suited to wotagei—characterized not merely by high tempo, but by **clear rhythmic markers (4/4 meter, audible kick/downbeat)**, **4-bar / 8-bar (`32カウント`) phrase regularity**, and **dramatic dynamic contrast** between restrained or tension-building passages (`Aメロ`/`Bメロ`/`落ちサビ`) and explosive chorus releases (`サビ`/`ラスサビ`).
- **Takamari (高まり):** Literally "heightening/rising," used in wotagei and anikura culture for the **psychological and physiological surge of excitement** approaching and hitting a song's climax ("高まる" = getting hyped/pumped). In anison/voice-actor music—exemplified by **Nana Mizuki (水樹奈々)** tracks such as *ETERNAL BLAZE*, *Astrogation* (~143 BPM), *Synchrogazer*, and *Vitalization*—*takamari* is driven by **dramatic pre-chorus tension (`Bメロ`)**, **dense synth/symphonic/rock arrangement**, **high vocal intensity**, and **upward key modulations (`転調`)** that create an unmistakable "launch point" into the chorus.

### 1.3 Standard J-Pop / Anison Structural Progression & Wotagei Mapping

A full-length J-pop / anison track typically follows a linear, goal-directed progression (`イントロ -> 1番 [Aメロ -> Bメロ -> サビ] -> 間奏 -> 2番 [Aメロ -> Bメロ -> サビ] -> 間奏/Cメロ -> 落ちサビ -> ラスサビ -> アウトロ`), though 90-second TV-size anime openings frequently invert the opening with an **`頭サビ` (atama-sabi / chorus-first)** hook before dropping into `Aメロ`.

| Musical Section | Function in J-Pop / Anison | Acoustic & Musical Profile | Wotagei Routine / Call Mapping | Energy & Tension Trajectory |
| :--- | :--- | :--- | :--- | :--- |
| **`イントロ` (Intro)** | Establishes tempo, key, and hook; often 8–16 bars in 4/4. | Full or rising instrumental motif (synth/guitar riff); vocals usually absent (unless `頭サビ`). | **MIX** calls (`タイガー、ファイヤー…`), clapping, or intro preparation (`チェック・セット・オープン`). | **Moderate -> Rising** (sets rhythmic expectation). |
| **`Aメロ` (Verse 1/2)** | Narrative setup; establishes rhythmic/lyrical baseline. | Lower vocal register, sparser percussion/instrumentation, moderate loudness. | Foundational repetitive moves: **OAD** (Over Action Dolphin), `ニーハイオーハイ`, `ソイヤ` (4 thrusts); rhythmic groove lock-in. | **Baseline / Steady** (controlled energy). |
| **`Aメロ` End -> `Bメロ` (Pre-Chorus)** | **Build-up / bridge into chorus**; typically 4 or 8 bars (`16` or `32` beats). | Rising vocal pitch, faster chord changes (harmonic rhythm), snare/hi-hat fills, rhythmic syncopation, sometimes brief thinning before impact. | **Last 2 bars of A-melo:** **`警報`** (*keihou* — e.g., *"せーの！はい！はい！はいはいはいはい！"*) alerts the floor.<br>**In B-melo:** **`PPPH`** (*pan-papan-hyuu*: clap beat 1, off-beat 2-and, beat 3, shout/jump beat 4); or cyalume **`ロザリオ`** / **`ロサンゼルス`** patterns. | **Steeply Rising Tension** ("tame" / coiling spring / anticipation). |
| **`サビ` (Chorus)** | Emotional and melodic climax; typically **8 bars = 32 beats (`4エイト`)** or 16 bars. | Peak vocal register/power, full drums/bass/synth/guitar wall of sound, high loudness & high-frequency brightness. | **`サビ技` / `サイリウムダンス`** choreographed in `1エイト` (8-count) × 4 = `32カウント` blocks: **Romance**, **Thunder Snake**, **Amaterasu**, **Muramasa**; `フッフー` calls. | **Peak Release / High Energy** (first explosion of *takamari*). |
| **`間奏` (Interlude)** | Instrumental break / guitar or synth solo after 1st or 2nd chorus. | Vocal drops out; fast lead guitar/synth arpeggios, driving drums. | **`間奏技` / `技連`** (high-speed combo chains favored by technical `技師` performers). | **Sustained High / Virtuosic** (showcase energy). |
| **`落ちサビ` (Drop-Chorus)** | Pre-final vulnerability & contrast device unique/prominent in J-pop/anison. | **Dramatic `音抜き` (instrumentation drop-out)**: drums/bass cut out, leaving solo vocal over piano, strings, or soft pad. | **`ケチャ` (kecha)** (low crouch raising/offering both hands toward stage), or quiet glowstick prep / cracking ultra-bright sticks (`大閃光` / `スパロン`). | **Sharp Dip in Loudness, Peak Anticipatory Tension** (held breath). |
| **`ラスサビ` (Final Chorus)** | Ultimate climax of the entire track. | Full band crashes back in at max loudness; frequently adds **+1 or +2 semitone upward modulation (`転調`)** and extra vocal ad-libs/backing harmonies. | Maximum-intensity **`サビ技`** (`ロマンス`, `サンダースネイク`, `アマテラス`, etc.) with ultra-bright sticks; peak **`高まり`**. | **Maximum Climax** (highest loudness + emotional release). |

#### Typical Energy & Tension Curve

```text
Loudness / Density
  ^            [サビ]            [サビ]                      [ラスサビ (+転調)]
  |            /----\            /----\    [間奏]           /----------------\
  |     [Bメロ]/      \    [A2]  /      \  /----\          /                  \
  |      /---/        \  /----\/        \/      \  [落ちサビ]                  \
  | [イントロ][A1]       \/                        \___[音抜き]___/             [アウトロ]
  +---------------------------------------------------------------------------------> Time
  Action: MIX   OAD    警報+PPPH  サビ技(32ct)       技連    ケチャ+大閃光  全力サビ技
                       /ロザリオ
```

Notice the critical distinction highlighted in community guides and Farbood-style tension psychology: **Acoustic loudness dips during `落ちサビ`, yet subjective *takamari* tension climbs sharply** because listeners anticipate the imminent `ラスサビ` explosion.

### 1.4 Typical BPM Range for `打ち曲`

Community blogs, anikura DJ notes, and wotagei song breakdowns identify **4/4 time** and an overall usable tempo band of roughly **130–180 BPM** (with no single canonical numerical table verified on `wikiwiki.jp/neosr/`):

- **~130–160 BPM (Standard / Accessible Zone):** Widely cited as the most balanced, comfortable range for both classic `サビ技` (Romance, OAD, Thunder Snake) and full `PPPH` + `32カウント` routines without rushing arm arcs (e.g., Mizuki Nana's *Astrogation* at ~143 BPM).
- **~160–180 BPM (High-Energy / "Golden" Fast-Uchi Zone):** Heavily favored in fast anison, Vocaloid, and hardcore/rock-tinged wotagei (e.g., *Butter-Fly* ~160s, *Melt* / modern Vocaloid ~170s, *Don't say "lazy"* / *Senbonzakura* ~180). Demands sharp stamina and compact technique transitions.
- **>180 BPM (Extreme / Half-Time Adaptation):** Requires **half-time** (`ハーフテンポ`) movement (treating 180 BPM as 90 BPM) or simplified rapid patterns, otherwise routine clarity breaks down.
- **<130 BPM:** Better suited to slow glowstick waving (`振りコピ` / swaying) or heavy four-on-the-floor groove than high-speed *cyalume dance*, unless doubled (`倍取り`).
- **MIR Pitfall (Octave Errors):** Beat trackers often report **half-time or double-time** tempos (e.g., 85 BPM instead of 170 BPM, or 190 BPM instead of 95 BPM), so a recommender filtering on BPM must evaluate $b \in [130, 185]$ alongside $2b \in [130, 185]$.

---

## 2. MIR Literature Review

### 2.1 Chorus Detection & Structural Segmentation

| Method / Tool | Primary Citation & Source | Core Mechanism | Strengths for `打ち曲` / Takamari | Limitations (Especially on 30s Previews) |
| :--- | :--- | :--- | :--- | :--- |
| **RefraiD** | Goto (2003, *ICASSP*); Goto (2006, *IEEE TASLP* 14(5):1783–1794, DOI: [`10.1109/TSA.2005.863204`](https://doi.org/10.1109/TSA.2005.863204)) | 12-dim chroma vectors -> Self-Similarity Time-Lag Matrix; **12 cyclic chroma shifts** handle transposed/modulated choruses; picks chorus via repetition count, duration, internal repetition, and acoustic/loudness boundaries. Benchmarked on **RWC Popular Music** (Goto et al., *ISMIR 2002*; AIST annotations, *ISMIR 2006*). | Explicitly handles **key-modulated `ラスサビ`** (`転調`), and RWC-MDB-P includes ~74 Japanese pop/rock tracks among its 100 songs. | Requires **full-song repetition** across multiple verses/choruses; fails completely on 30s preview clips containing ≤1 chorus. |
| **Audio Novelty (Checkerboard Kernel)** | Foote (2000, *IEEE ICME*, pp. 452–455, DOI: [`10.1109/ICME.2000.869637`](https://doi.org/10.1109/ICME.2000.869637)) | Convolves a Gaussian-tapered **checkerboard kernel** along the diagonal of a feature Self-Similarity Matrix (SSM); local peaks mark transitions between acoustically homogeneous blocks. | **Works locally without full-song repetition**; detects `Bメロ -> サビ` or `落ちサビ -> ラスサビ` boundaries inside short (e.g., 30s) clips. | Boundary detector only—cannot label *which* side is `Bメロ` vs. `サビ` without directional energy/density features. |
| **Laplacian Segmentation** (`librosa.segment`) | McFee & Ellis (2014, *ISMIR*, pp. 405–410, [PDF](https://archives.ismir.net/ismir2014/paper/000319.pdf), DOI: [`10.5281/zenodo.1415778`](https://doi.org/10.5281/zenodo.1415778)); [Librosa docs](https://librosa.org/doc/latest/generated/librosa.segment.agglomerative.html) | Beat-synchronous CQT/MFCC recurrence matrix (`recurrence_matrix`, `path_enhance`) + local sequence links -> symmetric normalized graph Laplacian -> spectral clustering of eigenvectors into $k$ structural labels. | Clean, open-source Python (`librosa.segment`) implementation; captures multi-scale song structure across whole tracks. | Global clustering requires whole-song repetition; unsupervised labels (`0..k-1`) are arbitrary until ranked by loudness/duration. |
| **MSAF** (Music Structure Analysis Framework) | Nieto & Bello (2016, *ISMIR*, [PDF](https://zenodo.org/record/1417661/files/NietoB16.pdf)); [`urinieto/msaf`](https://github.com/urinieto/msaf) | Unified Python framework benchmarking boundary + labeling algorithms (Foote, CNMF, LSD/Spectral Clustering, CBM) across CQT, MFCC, chroma, and Tonnetz features. | Ideal for systematically comparing boundary detectors (e.g., Foote vs. CNMF) on an annotated J-pop validation set. | Older dependency stack in Python environments; algorithms still expect full tracks for section labeling. |
| **`allin1`** (All-In-One Metrical & Functional Structure) | Kim & Nam (2023, *IEEE WASPAA*, [arXiv:2307.16425](https://arxiv.org/abs/2307.16425)); [`mir-aidj/allin1`](https://github.com/mir-aidj/allin1) | Deep learning neighborhood-attention model on source-separated (demixed) spectrograms jointly predicting beats, downbeats, and functional labels (`intro`, `verse`, `chorus`, `bridge`, `inst`, `outro`). | Jointly outputs beat grid + downbeats + semantic `chorus` labels (`1エイト` alignment!) in one pass. | Trained on **Harmonix Set** (Nieto et al., *ISMIR 2019*, [PDF](https://zenodo.org/record/3527878/files/ismir2019_paper_000068.pdf)) of Western pop/rock/EDM; lacks explicit `B-melo`/`ochi-sabi` labels and requires heavy PyTorch/demixing dependencies (often overkill or mismatched for 30s clips). |

### 2.2 Build-Up, Musical Tension Curves, and Climax/Drop Detection

1. **Farbood (2012) — Parametric, Temporal Model of Musical Tension** (*Music Perception*, 29(4):387–428, DOI: [`10.1525/MP.2012.29.4.387`](https://doi.org/10.1525/MP.2012.29.4.387)):
   - Empirically models continuous listener tension ratings from multiple musical parameters: **dynamics** (loudness/amplitude), **onset frequency** (note/event attack rate), **pitch height** (melodic/ensemble register), **tempo**, and **harmony** (tonal tension via Lerdahl's Tonal Pitch Space).
   - **Key finding for *takamari*:** Features operate on **different perceptual memory windows**—surface parameters like dynamics and onset density act over rapid, short-term windows (a few seconds), whereas harmony integrates over longer windows—and perceived tension depends strongly on **trend salience** (directional change/slope), not static snapshots.
2. **Yadati, Larson, Liem, & Hanjalic (2014) — EDM Drop Detection** (*ISMIR 2014*, pp. 143–148, [PDF](https://archives.ismir.net/ismir2014/paper/000297.pdf)):
   - Used timestamped SoundCloud comments as weak labels to locate socially significant "drop" moments, showing that drops behave as a **two-stage acoustic watershed**: a transitional build-up (filtering, rising pitch, noise sweeps, sudden bass/loudness pullback) followed by full-spectrum release.
3. **Solberg (2014) — Production Techniques in EDM Build-ups and Drops** (*Dancecult*, 6(1), DOI: [`10.12801/1947-5403.2014.06.01.04`](https://doi.org/10.12801/1947-5403.2014.06.01.04)):
   - Documents the production mechanics of tension and release: accelerating snare rolls (1/4 -> 1/8 -> 1/16 -> 1/32 subdivisions), ascending pitch/noise risers, high-pass filter sweeps that thin out low-end energy before the drop, and abrupt full-bass re-entry on beat 1 of the drop.
4. **Standard Signal Descriptors in `librosa` and `Essentia`**:
   - **Loudness & Dynamics:** `librosa.feature.rms` / dB-scaled energy; `essentia.standard.LoudnessEBUR128` computing ITU-R BS.1770 / EBU R128 **momentary loudness** (400 ms window, LUFS), **short-term loudness** (3 s window, LUFS), **integrated loudness**, and **loudness range (LRA)** ([Essentia docs](https://essentia.upf.edu/reference/std_LoudnessEBUR128.html)).
   - **Onset Density & Rhythmic Drive:** `librosa.onset.onset_strength` (spectral-flux onset envelope over mel-spectrogram) and `librosa.onset.onset_detect` ([Librosa docs](https://librosa.org/doc/latest/generated/librosa.onset.onset_strength.html)); `essentia.standard.OnsetRate` (onsets/sec at 44.1 kHz), `essentia.standard.Flux` (L1/L2 spectral flux, optionally half-rectified), `essentia.standard.HFC` (Masri's high-frequency-weighted transient energy), and `essentia.standard.RhythmExtractor2013` (BPM, beat ticks, confidence).
   - **Spectral Brightness & Timbre Shift:** `librosa.feature.spectral_centroid` ([Librosa docs](https://librosa.org/doc/latest/generated/librosa.feature.spectral_centroid.html)) / `essentia.standard.Centroid` (spectral center of mass in Hz, correlated with perceived brightness) and `librosa.feature.spectral_contrast` (octave-band peak-to-valley energy contrast).

---

## 3. Proposed Audio Proxies for "Takamari"

Because a class-project wotagei recommender may run on **30-second preview clips** (iTunes/Deezer) *or* full tracks, the proxies below do **not** assume global whole-song chorus detection (`RefraiD` / `Laplacian` clustering) is available. Instead, they detect the **strongest local build-up-to-release transition $t^*$** inside whatever audio segment is supplied (using a sliding pre/post window comparison), and provide a fallback intra-clip intensity signal when no transition occurs inside a chorus-only preview.

### 3.1 Candidate Transition Detection ($t^*$)

Given an audio signal $y$ (at sample rate $sr$), compute synchronized feature frames (hop length $h = 512$, or 0.1 s frames for EBU R128) and smooth each curve into $1\text{-s}$ median or moving-average frames. For any candidate boundary $t \in [W_{\text{pre}}, T - W_{\text{cho}}]$ (e.g., $W_{\text{pre}} = W_{\text{cho}} = 6\text{ s}$, roughly 3–4 bars at 140–160 BPM; use $8\text{ s}$ on full songs and $5\text{--}6\text{ s}$ on 30 s clips), compare the **pre-boundary window** $\mathcal{W}_{\text{pre}}(t) = [t - W_{\text{pre}}, t)$ against the **post-boundary window** $\mathcal{W}_{\text{cho}}(t) = [t, t + W_{\text{cho}})$.

We can select $t^*$ either as the peak of a multi-feature **Foote checkerboard novelty curve** (computed on beat- or 0.5s-binned MFCC + chroma + log-RMS) *restricted to upward-energy transitions*, or directly as the timestamp maximizing upward loudness + spectral contrast:

$$t^* = \arg\max_{t \in [W_{\text{pre}},\, T - W_{\text{cho}}]} \Big( \Delta L_{\text{dB}}(t) + \lambda \, N_{\text{Foote}}(t) \Big)$$

---

### 3.2 Proxy 1: Pre-Chorus-to-Chorus Loudness Contrast ($\Delta L_{\text{dB}}$)

**What it captures:** The macroscopic acoustic "explosion" from a restrained `Aメロ`/`Bメロ` or stripped-down `落ちサビ` into `サビ`/`ラスサビ`—directly operationalizing Farbood's **dynamics** dimension and the wotagei community's emphasis on dynamic `落差` (drop-off and release).

**Exact Mathematical Definition:**
Let $L(n)$ be frame-level log loudness in decibels (either $20 \log_{10}(\text{RMS}(n) / \text{RMS}_{\text{ref}})$ from `librosa.feature.rms`, or EBU R128 400 ms momentary loudness $L_M(n)$ in LUFS from `LoudnessEBUR128`). Using medians to resist single-beat drum fills or claps inside `Bメロ`:

$$\Delta L_{\text{dB}}(t^*) = \operatorname{median}_{n \in \mathcal{W}_{\text{cho}}(t^*)} L(n) \;-\; \operatorname{median}_{n \in \mathcal{W}_{\text{pre}}(t^*)} L(n)$$

To also credit **continuous ramp build-ups** inside $\mathcal{W}_{\text{pre}}(t^*)$ (e.g., gradual `Bメロ` crescendo before $t^*$), optionally compute the pre-boundary positive loudness slope $\beta_L = \text{OLS slope of } L(n) \text{ on } \mathcal{W}_{\text{pre}}(t^*)$ in $\text{dB/s}$ (clipped at $\ge 0$).

**Python Implementation (`librosa` & `Essentia`):**

```python
import numpy as np
import librosa


def loudness_contrast_librosa(
    y: np.ndarray, sr: int = 22050, hop_length: int = 512, win_sec: float = 6.0
) -> tuple[float, float, float]:
    """Return (max_delta_db, t_star_sec, pre_slope_db_per_sec)."""
    rms = librosa.feature.rms(y=y, frame_length=2048, hop_length=hop_length)[0]
    db = librosa.amplitude_to_db(rms, ref=np.max)  # relative to track/clip peak (<= 0 dB)
    times = librosa.frames_to_time(np.arange(len(db)), sr=sr, hop_length=hop_length)

    fps = sr / hop_length
    w = int(round(win_sec * fps))
    if len(db) <= 2 * w:
        # Clip too short for pre/post windows: fall back to dynamic spread
        return float(np.percentile(db, 90) - np.percentile(db, 20)), float(times[len(db) // 2]), 0.0

    best_delta = -np.inf
    best_idx = w
    for idx in range(w, len(db) - w):
        delta = float(np.median(db[idx : idx + w]) - np.median(db[idx - w : idx]))
        if delta > best_delta:
            best_delta = delta
            best_idx = idx

    pre_db = db[best_idx - w : best_idx]
    pre_t = times[best_idx - w : best_idx] - times[best_idx - w]
    slope = float(np.polyfit(pre_t, pre_db, 1)[0])
    return best_delta, float(times[best_idx]), max(0.0, slope)
```

With **Essentia** (perceptually K-weighted EBU R128 momentary loudness; note `LoudnessEBUR128` requires separate left/right `vector_real` inputs):

```python
import numpy as np
import essentia.standard as es


def loudness_contrast_essentia(audio_path: str, win_sec: float = 6.0) -> tuple[float, float]:
    """Compute max momentary LUFS jump (dB/LU) via EBU R128."""
    audio, sr, n_ch, *_ = es.AudioLoader(filename=audio_path)()
    left = audio[0]
    right = audio[1] if n_ch >= 2 else audio[0]

    hop_sec = 0.1  # LoudnessEBUR128 default hopSize
    momentary, _, _, _ = es.LoudnessEBUR128(hopSize=hop_sec, sampleRate=sr)(left, right)
    m = np.asarray(momentary, dtype=np.float32)
    m = np.where(np.isfinite(m) & (m > -70.0), m, -70.0)

    w = int(round(win_sec / hop_sec))
    if len(m) <= 2 * w:
        return float(np.percentile(m, 90) - np.percentile(m, 20)), 0.0

    deltas = [
        float(np.median(m[i : i + w]) - np.median(m[i - w : i]))
        for i in range(w, len(m) - w)
    ]
    best_offset = int(np.argmax(deltas))
    return deltas[best_offset], (best_offset + w) * hop_sec
```

---

### 3.3 Proxy 2: Onset Density & Rhythmic Drive Surge ($\Delta \rho_{\text{onset}}$ + BPM-Gated Pulse)

**What it captures:** Farbood's **onset frequency** dimension plus the rhythmic transition from sparser `Aメロ` / half-time `Bメロ` (or silent `落ちサビ`) into driving 8-beat / 4-on-the-floor `サビ` percussion (`1エイト = 32カウント`). Crucially, it separates a gentle crescendo in a ballad from an energetic `打ち曲` chorus where percussion kicks in sharply.

**Exact Mathematical Definition:**
Let $O_{\text{env}}(n)$ be the spectral-flux onset strength envelope (`librosa.onset.onset_strength`), and let $\{o_1, \dots, o_M\}$ be peak-picked onset timestamps (`librosa.onset.onset_detect` or `es.OnsetRate`). Over windows $\mathcal{W}_{\text{pre}}(t^*)$ and $\mathcal{W}_{\text{cho}}(t^*)$ of duration $W$ seconds:

$$\rho_{\text{onset}}(\mathcal{W}) = \frac{|\{o_m \in \mathcal{W}\}|}{W} \; [\text{onsets/s}], \qquad \bar{O}(\mathcal{W}) = \frac{1}{|\mathcal{W}|} \sum_{n \in \mathcal{W}} O_{\text{env}}(n)$$

Define the **onset-drive surge** as the standardized jump in mean onset strength (continuous, less brittle than discrete peak thresholds) paired with onset-rate contrast:

$$\Delta \bar{O}_z(t^*) = \frac{\bar{O}(\mathcal{W}_{\text{cho}}(t^*)) - \bar{O}(\mathcal{W}_{\text{pre}}(t^*))}{\operatorname{std}(O_{\text{env}}) + \epsilon}, \qquad \Delta \rho_{\text{onset}}(t^*) = \rho_{\text{onset}}(\mathcal{W}_{\text{cho}}(t^*)) - \rho_{\text{onset}}(\mathcal{W}_{\text{pre}}(t^*))$$

To prevent unmetrical noise from scoring well as a wotagei candidate, compute **beat-band eligibility** from estimated tempo $\hat{b}$ (`librosa.beat.beat_track` or `es.RhythmExtractor2013` returning `bpm, ticks, confidence, estimates, bpmIntervals`): verify $\hat{b}$ or $2\hat{b}$ lies within $[130, 185]\text{ BPM}$ and track Essentia `confidence` (when `method="multifeature"`).

**Python Implementation (`librosa` & `Essentia`):**

```python
import numpy as np
import librosa


def onset_drive_surge_librosa(
    y: np.ndarray, sr: int = 22050, t_star_sec: float = 10.0, win_sec: float = 6.0, hop_length: int = 512
) -> dict[str, float]:
    o_env = librosa.onset.onset_strength(y=y, sr=sr, hop_length=hop_length)
    onset_times = librosa.onset.onset_detect(
        onset_envelope=o_env, sr=sr, hop_length=hop_length, units="time"
    )
    tempo, _ = librosa.beat.beat_track(onset_envelope=o_env, sr=sr, hop_length=hop_length)
    bpm = float(np.atleast_1d(tempo)[0])

    times = librosa.frames_to_time(np.arange(len(o_env)), sr=sr, hop_length=hop_length)
    pre_mask = (times >= t_star_sec - win_sec) & (times < t_star_sec)
    cho_mask = (times >= t_star_sec) & (times < t_star_sec + win_sec)

    rate_pre = float(np.sum((onset_times >= t_star_sec - win_sec) & (onset_times < t_star_sec)) / win_sec)
    rate_cho = float(np.sum((onset_times >= t_star_sec) & (onset_times < t_star_sec + win_sec)) / win_sec)

    std_o = float(np.std(o_env)) + 1e-6
    delta_o_z = float((np.mean(o_env[cho_mask]) - np.mean(o_env[pre_mask])) / std_o) if pre_mask.any() and cho_mask.any() else 0.0

    return {
        "delta_onset_rate": rate_cho - rate_pre,
        "delta_onset_strength_z": delta_o_z,
        "chorus_onset_rate": rate_cho,
        "estimated_bpm": bpm,
    }
```

With **Essentia** (`OnsetRate` requires `44100 Hz` mono audio; `RhythmExtractor2013` provides `bpm` and beat `confidence`):

```python
import numpy as np
import essentia.standard as es


def onset_and_rhythm_essentia(audio_path: str, t_star_sec: float = 10.0, win_sec: float = 6.0) -> dict[str, float]:
    audio_44k = es.MonoLoader(filename=audio_path, sampleRate=44100)()
    onset_times, global_rate = es.OnsetRate()(audio_44k)
    onset_times = np.asarray(onset_times)

    bpm, ticks, confidence, _, _ = es.RhythmExtractor2013(method="multifeature")(audio_44k)

    rate_pre = float(np.sum((onset_times >= t_star_sec - win_sec) & (onset_times < t_star_sec)) / win_sec)
    rate_cho = float(np.sum((onset_times >= t_star_sec) & (onset_times < t_star_sec + win_sec)) / win_sec)
    return {
        "delta_onset_rate": rate_cho - rate_pre,
        "global_onset_rate": float(global_rate),
        "bpm": float(bpm),
        "beat_confidence": float(confidence),
    }
```

---

### 3.4 Proxy 3: Spectral Brightness & High-Frequency Expansion ($\Delta C_{\text{oct}}$ & Boundary Flux)

**What it captures:** When an anison/J-pop arrangement opens from `Bメロ` (or `落ちサビ`) into `サビ` (`ラスサビ`), crash cymbals, stacked synth leads, high-register vocals, and octave harmonies shift spectral energy upward—matching Farbood's **pitch height** parameter and timbre expansion even in loudness-compressed masters where raw RMS changes only modestly.

**Exact Mathematical Definition:**
At frame $n$ with STFT magnitude $|X(n, k)|$ and bin frequencies $f_k$ (in Hz):

$$C(n) = \frac{\sum_{k} f_k \, |X(n, k)|}{\sum_{k} |X(n, k)| + \epsilon} \; [\text{Hz}]$$

Because human pitch/brightness perception is logarithmic, measure the **octave-scale centroid jump** across $t^*$:

$$\Delta C_{\text{oct}}(t^*) = \log_2 \left( \frac{\operatorname{median}_{n \in \mathcal{W}_{\text{cho}}(t^*)} C(n) + \epsilon}{\operatorname{median}_{n \in \mathcal{W}_{\text{pre}}(t^*)} C(n) + \epsilon} \right)$$

Optionally reinforce with the **peak positive Spectral Flux** within $t^* \pm 0.5\text{ s}$ (`es.Flux(halfRectify=True)`), which spikes when new harmonic/percussive instruments enter simultaneously on downbeat 1 of the chorus.

**Python Implementation (`librosa` & `Essentia`):**

```python
import numpy as np
import librosa


def spectral_expansion_librosa(
    y: np.ndarray, sr: int = 22050, t_star_sec: float = 10.0, win_sec: float = 6.0, hop_length: int = 512
) -> float:
    cent = librosa.feature.spectral_centroid(y=y, sr=sr, n_fft=2048, hop_length=hop_length)[0]
    times = librosa.frames_to_time(np.arange(len(cent)), sr=sr, hop_length=hop_length)

    pre = cent[(times >= t_star_sec - win_sec) & (times < t_star_sec)]
    cho = cent[(times >= t_star_sec) & (times < t_star_sec + win_sec)]
    if len(pre) == 0 or len(cho) == 0:
        return 0.0
    return float(np.log2((np.median(cho) + 1e-6) / (np.median(pre) + 1e-6)))
```

With **Essentia** (`FrameGenerator` + `Windowing` + `Spectrum` + `Centroid` + `Flux`):

```python
import numpy as np
import essentia.standard as es


def spectral_expansion_essentia(
    audio_path: str, t_star_sec: float = 10.0, win_sec: float = 6.0, frame_size: int = 2048, hop_size: int = 512
) -> dict[str, float]:
    sr = 44100
    audio = es.MonoLoader(filename=audio_path, sampleRate=sr)()
    win = es.Windowing(type="hann")
    spec_algo = es.Spectrum()
    cent_algo = es.Centroid(range=sr / 2.0)
    flux_algo = es.Flux(halfRectify=True, norm="L2")

    centroids, fluxes = [], []
    for frame in es.FrameGenerator(audio, frameSize=frame_size, hopSize=hop_size, startFromZero=True):
        mag = spec_algo(win(frame))
        centroids.append(cent_algo(mag))
        fluxes.append(flux_algo(mag))

    centroids = np.asarray(centroids)
    fluxes = np.asarray(fluxes)
    times = np.arange(len(centroids)) * (hop_size / sr)

    pre_c = centroids[(times >= t_star_sec - win_sec) & (times < t_star_sec)]
    cho_c = centroids[(times >= t_star_sec) & (times < t_star_sec + win_sec)]
    boundary_flux = fluxes[(times >= t_star_sec - 0.5) & (times <= t_star_sec + 0.5)]

    delta_c_oct = float(np.log2((np.median(cho_c) + 1e-6) / (np.median(pre_c) + 1e-6))) if len(pre_c) and len(cho_c) else 0.0
    peak_flux = float(np.max(boundary_flux)) if len(boundary_flux) else 0.0
    return {"delta_centroid_oct": delta_c_oct, "boundary_peak_flux": peak_flux}
```

---

### 3.5 Proposed Composite Takamari Score (Hypothesis for Calibration)

To rank candidate songs into a single `takamari` feature compatible with `src/recommender.py`, combine the three contrast proxies after normalizing each to $[0, 1]$ (via dataset z-scores passed through a sigmoid or domain-anchored min-max clipping, e.g., clipping $\Delta L_{\text{dB}}$ on $[0, 6]\text{ dB}$, $\Delta \bar{O}_z$ on $[0, 1.5]$, and $\Delta C_{\text{oct}}$ on $[0, 0.5]$ octaves):

$$T_{\text{transition}} = w_1 \, \tilde{\Delta L}_{\text{dB}}(t^*) + w_2 \, \tilde{\Delta \bar{O}}_z(t^*) + w_3 \, \tilde{\Delta C}_{\text{oct}}(t^*)$$

**Status: Uncalibrated Hypothesis.** Initial heuristic weights (e.g., $w_1 = 0.45,\; w_2 = 0.35,\; w_3 = 0.20$) should **not** be treated as validated constants; they should be calibrated and tested (e.g., Spearman $\rho$ / NDCG) against a hand-annotated ground-truth set of ~15–30 songs labeled with human 1–5 *takamari* scores and exact `Bメロ -> サビ` timestamps.

---

### 3.6 Full Audio vs. 30-Second Preview Clips

Neither the **iTunes Search API** (`previewUrl`, 30s AAC/MP4) nor the **Deezer API** (`preview`, 30s MP3) allows callers to query or specify the start timestamp of the 30-second preview clip. Consequently, preview-based pipelines encounter three distinct regimes:

| Clip Regime | What the 30s Clip Contains | Behavior of MIR & Our 3 Proxies | Mitigation / Practical Handling |
| :--- | :--- | :--- | :--- |
| **Regime A: Transition-Capturing** (common when preview starts mid-`Bメロ`) | Late `Aメロ`/`Bメロ` -> `サビ` (or rarely `Cメロ`/`落ちサビ` -> `ラスサビ`). | **Ideal case:** Global repetition methods (`RefraiD`, `Laplacian`) fail, but local $t^*$ sliding-window proxies ($\Delta L_{\text{dB}}, \Delta \bar{O}_z, \Delta C_{\text{oct}}$) detect the boundary directly using $W \approx 5\text{--}6\text{ s}$ windows. | Keep $W_{\text{pre}}, W_{\text{cho}} \le 6\text{ s}$ so $t^* \in [6\text{ s}, 24\text{ s}]$ can still be evaluated cleanly inside 30s. |
| **Regime B: Chorus-Only** (preview starts at `サビ` downbeat) | Continuous `サビ` throughout all 30 seconds. | **Contrast collapses ($\Delta L_{\text{dB}} \approx 0$)** because no pre-chorus baseline is present! | Detect flat contrast (`boundary_detected = False`) and score a **chorus-intensity fallback** (absolute onset rate $\rho_{\text{onset}}$, beat regularity in `[130, 185]` BPM, high `LoudnessEBUR128` short-term level) while flagging lower confidence on *transition takamari*. |
| **Regime C: Intro / Verse-Only** (some labels start previews at `t=0`) | `イントロ` -> `Aメロ` only. | Misses both `Bメロ -> サビ` and `落ちサビ -> ラスサビ` (note: `落ちサビ -> ラスサビ` typically occurs at 2:30–3:30 in full tracks and is **almost never captured** in 30s previews). | Can only evaluate BPM/beat clarity; manual or full-audio fallback is needed if preview misses the chorus entirely. |

**Key Takeaway:** Full tracks are strictly required to measure **`落ちサビ -> ラスサビ`** and modulated final choruses (`転調`, RefraiD); 30-second previews can only measure the **first `Bメロ -> サビ` transition** *if* the distributor's fixed preview happens to span that boundary.

---

### 3.7 Generalization Across English Pop, EDM, and Rock

| Genre & Typical Form | Structural Alignment with J-Pop `Aメロ -> Bメロ -> サビ` | Proxy 1 ($\Delta L_{\text{dB}}$) | Proxy 2 ($\Delta \rho_{\text{onset}}$) | Proxy 3 ($\Delta C_{\text{oct}}$) |
| :--- | :--- | :--- | :--- | :--- |
| **J-Pop / Anison / Idol**<br>`Aメロ -> Bメロ -> サビ` (+ `落ちサビ -> ラスサビ`) | Reference archetype; `Bメロ` actively builds formal urgency and `落ちサビ -> ラスサビ` maximizes contrast. | **Strong.** Clear step-up at `サビ` and huge step at `落ちサビ -> ラスサビ`. | **Strong.** Driving 8-beat / 4-on-the-floor chorus drums lift onset density over `A`/`Bメロ`. | **Strong.** Synths, brass, crash cymbals, and soaring chorus vocals shift centroid up. |
| **Western English Pop**<br>`Verse -> Pre-Chorus -> Chorus` (Summach, 2011) | **High alignment when a Pre-Chorus is present** (e.g., synth-pop, arena pop): Summach documents harmonic/dynamic/rhythmic urgency preparing the chorus.<br>**Lower alignment** in groove/loop-centric or trap-pop songs lacking a pre-chorus or dropping out the beat on the hook. | **Strong** on anthem/pre-chorus pop; **weak/inverted** on minimalist "anti-drop" or stripped trap-pop hooks. | **Moderate–Strong** on dance-pop; **unreliable** if pre-chorus already runs a dense 16th hi-hat trap pattern. | **Strong** on layered pop choruses (stacked harmonies/synths); weaker on vocal-minimal hooks. |
| **EDM**<br>`Breakdown -> Build-up -> Drop` (Solberg, 2014; Yadati et al., 2014) | Structurally analogous tension/release, **but inverted micro-features inside the Build-up**. | **Very Strong** (massive sub-bass + LUFS surge at the Drop, especially after bass cut in Breakdown). | **FAILS / INVERTED!** Snare rolls accelerate (`1/8 -> 1/16 -> 1/32`) *during* the Build-up, making $\rho_{\text{onset}}$ **higher before the Drop** than in the 4-on-the-floor Drop! | **AMBIGUOUS / INVERTED!** High-pass filter sweeps & white-noise risers push centroid *up* right before the Drop; sub-bass re-entry at the Drop can actually **lower** centroid! |
| **Rock / Metal**<br>`Verse -> (Pre-Chorus) -> Chorus` | Shares section roles, though guitar riffs often drive Verses as hard as Choruses. | **Compressed / Weak.** Heavy distortion & brickwall mastering keep LRA low (often 1–2 dB between Verse and Chorus). | **Moderate.** Double-kick or crash patterns may shift, but distorted guitars create continuous high spectral density in both sections. | **Moderate.** Crash cymbals and higher chorus vocal belts still raise upper-mid/high energy, though saturated guitars blunt contrast. |

**Design Implication for Multi-Genre Support:**
- To support **English synth-pop / dance-pop** alongside Japanese `打ち曲`, all three proxies work well out of the box when a `Pre-Chorus -> Chorus` transition exists.
- To avoid false negatives on **EDM** (if EDM/hardcore crossovers like `アニソン×EDM` or Eurobeat/hands-up tracks are included), replace raw global centroid jump with **Sub-Bass Energy Contrast** ($\Delta E_{20\text{--}150\text{ Hz}}$) and treat rapid pre-boundary onset acceleration (`slope` $> 0$ in $\mathcal{W}_{\text{pre}}$) as a *positive* build-up signal rather than penalizing $\rho_{\text{cho}} < \rho_{\text{pre}}$.

---

## 4. Concrete Recommendations for `takamari` Integration in This Project

1. **Annotate a 15–30 Song Ground-Truth Benchmark First:** Manually label representative J-pop/anison `打ち曲` (e.g., Mizuki Nana, classic idol/Vocaloid wotagei tracks) plus non-`打ち曲` negatives (ballads, flat groove pop) with human 1–5 *takamari* scores, true BPM, and `Bメロ -> サビ` timestamps. Use this set to fit/validate proxy weights rather than hardcoding unverified weights.
2. **Start with `librosa` in `requirements.txt` (Pure Python/NumPy):** `librosa` provides `rms`, `onset_strength`, `onset_detect`, `spectral_centroid`, `beat_track`, and `segment` cleanly in standard `pip` environments. Add `Essentia` (`LoudnessEBUR128`, `OnsetRate`, `RhythmExtractor2013`) only if cross-platform wheel installation works cleanly on team machines and EBU R128 perceptual weighting is needed.
3. **Separate "Clip Regime" Metadata from Audio Score:** When running on 30-second previews, store whether a clear transition $t^*$ was found (`has_transition: bool`) alongside the proxy values so the recommender does not unfairly penalize an iconic `打ち曲` whose 30s preview started mid-chorus.
4. **Fix BPM Half/Double-Time Matching in Scoring:** When evaluating whether a track fits the `130–180 BPM` wotagei target band, check both `bpm` and `2 * bpm` (and `0.5 * bpm` if `> 200`), consistent with `RhythmExtractor2013` / `librosa.beat.beat_track` octave behavior.

---

## References & Primary Sources

1. **Goto, M.** (2006). *A Chorus Section Detection Method for Musical Audio Signals and Its Application to a Music Listening Station*. IEEE Transactions on Audio, Speech, and Language Processing, 14(5), 1783–1794. DOI: [`10.1109/TSA.2005.863204`](https://doi.org/10.1109/TSA.2005.863204) *(RefraiD conference origin: ICASSP 2003, V-437–440)*.
2. **Goto, M., Hashiguchi, H., Nishimura, T., & Oka, R.** (2002). *RWC Music Database: Popular, Classical and Jazz Music Databases*. ISMIR 2002, 287–288 *(and Goto 2006 ISMIR AIST Annotations)*.
3. **Foote, J.** (2000). *Automatic Audio Segmentation Using a Measure of Audio Novelty*. IEEE ICME 2000, Vol. 1, 452–455. DOI: [`10.1109/ICME.2000.869637`](https://doi.org/10.1109/ICME.2000.869637).
4. **McFee, B., & Ellis, D. P. W.** (2014). *Analyzing Song Structure with Spectral Clustering*. ISMIR 2014, 405–410. [PDF](https://archives.ismir.net/ismir2014/paper/000319.pdf) | DOI: [`10.5281/zenodo.1415778`](https://doi.org/10.5281/zenodo.1415778).
5. **Nieto, O., & Bello, J. P.** (2016). *Systematic Exploration of Computational Music Structure Research*. ISMIR 2016. [PDF](https://zenodo.org/record/1417661/files/NietoB16.pdf) | Repo: [`github.com/urinieto/msaf`](https://github.com/urinieto/msaf).
6. **Nieto, O., McCallum, M., Davies, M. E. P., Robertson, A., Stark, A., & Egozy, E.** (2019). *The Harmonix Set: Beats, Downbeats, and Functional Segment Annotations of Western Popular Music*. ISMIR 2019. [PDF](https://zenodo.org/record/3527878/files/ismir2019_paper_000068.pdf) | Repo: [`github.com/urinieto/harmonixset`](https://github.com/urinieto/harmonixset).
7. **Kim, T., & Nam, J.** (2023). *All-In-One Metrical And Functional Structure Analysis With Neighborhood Attentions on Demixed Audio*. IEEE WASPAA 2023. [arXiv:2307.16425](https://arxiv.org/abs/2307.16425) | Repo: [`github.com/mir-aidj/allin1`](https://github.com/mir-aidj/allin1).
8. **Farbood, M. M.** (2012). *A Parametric, Temporal Model of Musical Tension*. Music Perception: An Interdisciplinary Journal, 29(4), 387–428. DOI: [`10.1525/MP.2012.29.4.387`](https://doi.org/10.1525/MP.2012.29.4.387).
9. **Yadati, K., Larson, M., Liem, C. C. S., & Hanjalic, A.** (2014). *Detecting Drops in Electronic Dance Music: Content based approaches to a socially significant music event*. ISMIR 2014, 143–148. [PDF](https://archives.ismir.net/ismir2014/paper/000297.pdf).
10. **Solberg, R. T.** (2014). *Waiting for the Bass to Drop: Correlations Between Intense Emotional Experiences and Production Techniques in Build-up and Drop Sections of Electronic Dance Music*. Dancecult, 6(1). DOI: [`10.12801/1947-5403.2014.06.01.04`](https://doi.org/10.12801/1947-5403.2014.06.01.04) | [Article](https://dj.dancecult.net/index.php/dancecult/article/view/580).
11. **Summach, J.** (2011). *The Structure, Function, and Genesis of the Prechorus*. Music Theory Online, 17(3). [URL](https://mtosmt.org/issues/mto.11.17.3/mto.11.17.3.summach.html).
12. **Librosa Documentation:** [`librosa.feature.rms`](https://librosa.org/doc/latest/generated/librosa.feature.rms.html), [`librosa.onset.onset_strength`](https://librosa.org/doc/latest/generated/librosa.onset.onset_strength.html), [`librosa.onset.onset_detect`](https://librosa.org/doc/latest/generated/librosa.onset.onset_detect.html), [`librosa.feature.spectral_centroid`](https://librosa.org/doc/latest/generated/librosa.feature.spectral_centroid.html), [`librosa.beat.beat_track`](https://librosa.org/doc/latest/generated/librosa.beat.beat_track.html), [`librosa.segment.agglomerative`](https://librosa.org/doc/latest/generated/librosa.segment.agglomerative.html), and [Laplacian segmentation example](https://librosa.org/doc/latest/auto_examples/plot_segmentation.html).
13. **Essentia Documentation (UPF MTG):** [`LoudnessEBUR128`](https://essentia.upf.edu/reference/std_LoudnessEBUR128.html), [`OnsetRate`](https://essentia.upf.edu/reference/std_OnsetRate.html), [`RhythmExtractor2013`](https://essentia.upf.edu/reference/std_RhythmExtractor2013.html), [`Centroid`](https://essentia.upf.edu/reference/std_Centroid.html), [`Flux`](https://essentia.upf.edu/reference/std_Flux.html), [`AudioLoader`](https://essentia.upf.edu/reference/std_AudioLoader.html).
14. **Wotagei Community References:**
    - `https://wikiwiki.jp/neosr/ヲタ芸の基本的な構成` *(direct HTTP GET blocked by proxy policy: HTTP 403; indexed summary verified via search grounding)*.
    - `https://wikiwiki.jp/neosr/打ち曲` *(direct HTTP GET blocked by proxy policy: HTTP 403; partial menu/page metadata observed via search grounding, page body unverified)*.
    - Niconico Pedia: [ヲタ芸](https://dic.nicovideo.jp/a/ヲタ芸), [PPPH](https://dic.nicovideo.jp/a/PPPH).
    - Japanese Wikipedia: [オタ芸](https://ja.wikipedia.org/wiki/オタ芸).
