# Audio Feature & Legal Audio Preview Sources for J-Pop, Anison, Vocaloid & English Pop

**Research Date:** 2026-11-17  
**Target Schema:** [`data/songs.csv`](../data/songs.csv) & [`src/recommender.py`](../src/recommender.py) (`id,title,artist,genre,mood,energy,tempo_bpm,valence,danceability,acousticness,popularity,duration_ms,release_year`)  
**Scope:** Free tools/APIs only, suitable for a student AI class final project covering Japanese pop (J-pop / anison / idol / Vocaloid) and English pop.

---

## Executive Summary

1. **Spotify Web API is dead for new student apps.** On **November 27, 2024**, Spotify blocked new and Development Mode apps (`403 Forbidden`) from `GET /v1/audio-features`, `GET /v1/audio-analysis`, `GET /v1/recommendations`, related artists, and `preview_url`. Follow-up changes in **May 2025** (Extended Quota restricted to registered businesses with $\ge$ 250k MAUs) and **February 2026** (Development Mode requires an active Spotify Premium account, caps apps at 5 allowlisted users, removes `popularity` from track objects, and caps `/search` at `limit=10`) rule out Spotify even for basic popularity or catalog lookup in student pipelines.
2. **Best free legal audio source:** **iTunes Search API** (`country=JP` for J-pop/anison/Vocaloid, `country=US` for English pop) provides the widest legal catalog of 30-second previews (`.m4a` AAC) plus exact `trackTimeMillis`, `releaseDate`, and `primaryGenreName`, with **Deezer API** (30s `.mp3` `preview`, `rank`, `release_date`, `duration`) as a secondary source where domestic Japanese licensing exists. **YouTube / `yt-dlp` must not be used** due to explicit YouTube ToS and API Policy prohibitions on downloading and isolating audio streams, as well as PO Token / bot-gate fragility.
3. **Best free audio-feature stack:**
   - **Local analysis with `Essentia` + `essentia-tensorflow`** (or **`librosa` + `pyloudnorm`** for DSP-only features without semantic valence/mood) paired with **iTunes/Deezer 30s previews** decoded transiently in memory/temp buffers via `ffmpeg` (respecting stream-only / non-redistribution preview terms).
   - **ReccoBeats API** (`api.reccobeats.com/v1`) offers free, unauthenticated Spotify-compatible feature retrieval (given Spotify IDs/ISRCs) and direct 30s audio upload (`POST /v1/analysis/audio-features`), though with unverified long-term SLA and unverified direct title search.
   - **Last.fm API** (`track.getInfo`, `track.getTopTags`) fills the missing **`popularity`** (`listeners`/`playcount`) and **`genre`/`mood`** folksonomy tags (subject to notifying `partners@last.fm` for academic/research use per ToS).
   - **AcousticBrainz** (shut down in 2022) and **Kaggle Spotify datasets** (frozen ~2020–2022) are useful only for historical/benchmark backfills and miss all 2023–2026 J-pop/anison/Vocaloid releases.

---

## 1. Spotify Web API Restriction Verification (2024–2026)

### Findings from Official Spotify Developer Announcements

| Milestone | Effective Date | Primary Source | Exact Impact on Student / New Apps |
|---|---|---|---|
| **Web API Deprecations & Access Cut** | Nov 27, 2024 | [Spotify Developer Blog (Nov 27, 2024)](https://developer.spotify.com/blog/2024-11-27-changes-to-the-web-api) | New apps and existing Development Mode apps lost access (`403 Forbidden`) to `GET /v1/audio-features`, `GET /v1/audio-analysis`, `GET /v1/recommendations`, `GET /v1/artists/{id}/related-artists`, Featured/Category Playlists, algorithmic/editorial playlists, and `preview_url` (marked Deprecated on track objects). Only apps with pre-existing approved Extended Quota Mode were grandfathered. |
| **Extended Quota Mode Lockout** | May 15, 2025 (announced Apr 15, 2025) | [Spotify Developer Blog (Apr 15, 2025)](https://developer.spotify.com/blog/2025-04-15-updating-the-criteria-for-web-api-extended-access) | Individual developers and hobbyists are excluded from Extended Quota Mode. Applicants must be an **established registered business entity** operating an active launched service with **at least 250,000 Monthly Active Users (MAUs)**. |
| **Development Mode Lockdown** | Feb 11, 2026 (new apps) / Mar 9, 2026 (existing apps) | [Spotify Developer Blog (Feb 6, 2026)](https://developer.spotify.com/blog/2026-02-06-update-on-developer-access-and-platform-security) & [Feb 2026 Migration Guide](https://developer.spotify.com/documentation/web-api/tutorials/february-2026-migration-guide) | Development Mode now requires the app owner to hold an active **Spotify Premium** subscription, caps each app at **5 allowlisted users**, **removes the `popularity` field** from track/album/artist response objects, restricts bulk multi-get endpoints, and reduces `GET /search` max `limit` from `50` to `10` (default `5`). |

### Student Project Verdict
**Cannot be used.** A new student app in 2026 is locked into Development Mode and cannot access `audio-features`, `audio-analysis`, `preview_url`, or even `popularity`. Even using Spotify `/search` solely to fetch Spotify Track IDs for downstream services like ReccoBeats now requires an active Spotify Premium subscription and caps search pages at 10 items.

---

## 2. Structured Comparison Table

| Source | Category | Free / Auth | Rate Limits | J-Pop / Anison / Vocaloid Coverage | ToS / Legal Restrictions | `data/songs.csv` Columns Covered | Python Effort & Key Packages | Verification Status |
|---|---|---|---|---|---|---|---|---|
| **Spotify Web API** | Feature & Metadata API | Free tier requires **Premium** owner (Dev Mode); OAuth / Client Creds | App-owner quota; Dev Mode `/search` max `limit=10` | High catalog (metadata), **0% audio features** for new apps | `403` on audio endpoints; scraping/caching prohibited | `title`, `artist`, `duration_ms`, `release_year` (`popularity` removed in Dev Mode Feb 2026) | Low (`spotipy` / `requests`), **unusable for features** | **Verified** (Spotify Dev Blog 2024–2026) |
| **AcousticBrainz** | Frozen MIR Archive (Dumps / Legacy API) | Free, **CC0** public domain; no auth | Live API obsolete/unreliable; local dumps unlimited | **Low/Incomplete**: 0% post-June 2022 releases; sparse dōjin/Vocaloid pre-2022; Kanji/Romaji MBID mismatch | CC0 public domain dumps (~900–1000 GB full archive) | `tempo_bpm`, partial `energy` (loudness), `danceability`, `acousticness`, partial `valence`/`mood` | **High** (huge `.tar.zst` dumps or MBID lookup via MusicBrainz `1 req/s`) | **Verified** (MetaBrainz Blog & Downloads) |
| **`librosa` (+ `pyloudnorm`)** | Local DSP Signal Analysis | Free, **ISC** (`librosa`) / **MIT** (`pyloudnorm`) | Local CPU bound | **100% of provided audio** (needs external audio/previews); octave errors on fast anison | Local processing of legally obtained audio | `tempo_bpm`, `energy` proxy (RMS/LUFS), spectral features (**no** native `valence`, `danceability`, `acousticness`, `mood`) | **Medium** (`librosa>=1.0`, `soundfile`, `pyloudnorm`, system `ffmpeg` for `.m4a`) | **Verified** (`librosa` docs, GitHub Releases, PyPI) |
| **`Essentia` + `essentia-tensorflow`** | Local DSP + Deep ML Inference | Free for non-commercial (**AGPLv3** code; **CC BY-NC-ND 4.0** model weights) | Local CPU bound | **100% of provided audio**; ML models trained on Western/MSD/Discogs data (domain shift on Vocaloid/anison) | Non-commercial/academic OK under AGPLv3 + CC BY-NC-ND 4.0; commercial license required for commercial apps | `tempo_bpm`, `energy` (LUFS/RMS), `danceability`, `acousticness`, `valence` (rescaled 1–9), `mood` probs | **Medium–High** (`essentia-tensorflow` wheels on Linux/macOS arm64; poor native Windows support) | **Verified** (UPF Essentia Reference & Models Docs) |
| **ReccoBeats API** | Free Spotify-style Feature & Upload API | Free, **No auth** | Unspecified numeric cap (`429` + `Retry-After` header) | Catalog includes J-pop/anison (verified `ETERNAL BLAZE` 45 hits); **100% via `POST /v1/analysis/audio-features`** on 30s uploads (<=5 MB) | "As-is", no SLA; abusive traffic IP-blocked | `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness` (via `/audio-features`), plus `popularity`, `duration_ms` (via `/track/search`) | **Low** (`requests` + `ffmpeg` if converting `.m4a`->`.mp3`/`.wav` for upload) | **Verified (live HTTP `200 OK`)** on `/v1/artist/search`, `/v1/track/search`, & `/v1/track/{id}/audio-features` (`POST /analysis` untested) |
| **GetSongBPM API** | BPM / Audio Metadata API | Free **API key** required | **3,000 req/hr** (1-hr auto-block if exceeded) | **Weak–Moderate**: skewed toward Western music; spotty on niche anison/Vocaloid | **Mandatory live backlink** to `GetSongBPM.com` on public site/app or key suspended | `tempo_bpm`, `danceability` (0–100), `acousticness` (0–100), `genre` (**no** `energy`, `valence`, `popularity`, `duration_ms`, `release_year`) | **Low** (`requests`), requires hosting public backlink URL | **Verified** (GetSongBPM API Docs) |
| **Last.fm API** | Folksonomy Tags & Popularity API | Free **API key** required + `User-Agent` | Soft ~**5 req/sec** (Error `29` on burst) | **Strong** for popular J-pop/anison/Vocaloid; fragmented Kanji/Romaji names | Non-commercial free; **academic/research requires emailing `partners@last.fm` prior to use** | `popularity` (`listeners`/`playcount` log-scaled), `genre` & `mood` (noisy user tags), `duration_ms` (often `0`) | **Low** (`requests` / `pylast`), tag normalization needed | **Verified** (Last.fm API Docs & ToS) |
| **Kaggle Spotify Datasets** | Static Pre-Scraped CSVs | Free (Kaggle account); mixed/unclear licenses | Static download | **Moderate**: `114k` set has `j-pop`, `j-idol`, `anime`; **0% post-2022** releases; Vocaloid scarce | Kaggle Open Database / "Unknown"; upstream Spotify ToS prohibits bulk redistribution | `114k`: all numeric features + `popularity` + `duration_ms` (**missing `release_year`**); `1.2M`: has `year`, **missing `popularity` & `genre`** | **Low** (`pandas`), useful only for baseline/prototyping | **Verified** (Kaggle Dataset Schemas) |
| **iTunes Search API** | 30s Audio Preview & Metadata | Free, **No auth** | ~**20 calls/min** soft limit per IP | **Best legal J-pop/anison/Vocaloid coverage** when querying **`country=JP`** (`country=US` misses domestic labels) | **Stream-only** (no downloading/saving/caching); promo use w/ Apple Store badge & *"courtesy of iTunes"* attribution | `title`, `artist`, `genre` (`primaryGenreName`), `duration_ms`, `release_year`, + 30s `.m4a` `previewUrl` | **Low–Medium** (`requests` + `ffmpeg` to decode `.m4a` AAC in memory/temp) | **Verified (live HTTP `200 OK`)** on `country=JP` (`ETERNAL BLAZE` returned `previewUrl`, `trackTimeMillis`) |
| **Deezer API** | 30s Audio Preview & Metadata | Free, **No auth** for public catalog | **50 req / 5 sec** per IP (Error `4`) | **Moderate**: good for major J-pop/anime crossovers; weaker on Japan-exclusive idols/Vocaloid vs iTunes JP | Metadata caching OK; **image caching prohibited**; audio governed by Deezer Developer ToS (no redistribution/storage) | `title`, `artist`, `duration_ms`, `release_year`, `popularity` (`rank`), `tempo_bpm` (**often `0`**), + 30s `.mp3` `preview` | **Low** (`requests`; `.mp3` loads cleanly via `soundfile`>=1.1 or `ffmpeg`) | **Verified (live HTTP `200 OK`)** on `/search` (`ETERNAL BLAZE` returned `.mp3` `preview`, `rank`, `isrc`) |
| **YouTube / `yt-dlp`** | Full Audio Extraction | Free / Cookie / PO Token hacks | ~300 req/hr guest limit before bot block | **Near 100%** (MVs, Niconico mirrors, live cuts, Vocaloid) | **Prohibited** by YouTube ToS & API Developer Policies (no downloading, no separating audio from video) | Full audio if extracted (**do not use**) | **High friction** (PO Tokens, BotGuard `403`, account ban risk) | **Verified** (YouTube ToS & Google Developer Policies) |

---

## 3. Detailed Evaluation: Audio-Feature Sources & Datasets

### 3.1 AcousticBrainz (MetaBrainz / MTG-UPF)

* **Primary Sources:** [MetaBrainz Shutdown Blog (Feb 16, 2022)](https://blog.metabrainz.org/2022/02/16/acousticbrainz-making-a-hard-decision-to-end-the-project/), [AcousticBrainz Downloads](https://acousticbrainz.org/download), [MetaBrainz Community Update](https://community.metabrainz.org/t/acousticbrainz-submissions-data-dumps-and-next-steps/53872), [AcousticBrainz Genre Dataset](https://mtg.github.io/acousticbrainz-genre-dataset/).
* **Status:** MetaBrainz announced the shutdown on **February 16, 2022**, ended submissions in **June 2022**, and published final static archives in **July 2022**. While legacy read-only web endpoints (`/api/v1/<mbid>/low-level` and `/high-level`) intermittently remained online past target retirement dates, MetaBrainz engineers flagged the legacy infrastructure as obsolete (bot traffic/SSL instability) and subject to complete retirement.
* **Data Scale & Static Dumps:** ~29.4–29.5 million submissions covering ~7.5 million unique MusicBrainz Recording IDs (`MBID`s), released under **CC0** (public domain) as Zstandard (`.tar.zst`) archives:
  * Full low-level JSON dumps (~589 GB compressed) and high-level JSON dumps (~39 GB compressed; ~900–1000 GB total archive).
  * 100k sample dumps and smaller tabular CSV extracts (BPM, loudness, dynamic complexity).
* **J-Pop / Anison / Vocaloid Coverage:**
  * **Hard 2022 cutoff:** Zero coverage for any song released after June 2022 (e.g., 2023–2026 anime/J-pop hits).
  * **MBID Dependency & Script Friction:** Every track must first be mapped to a MusicBrainz **Recording MBID** (via the MusicBrainz WS/2 API, capped at **1 req/sec** per IP with a mandatory descriptive `User-Agent` header, or a local MusicBrainz PostgreSQL mirror). Japanese Kanji/Kana vs. Romaji inconsistencies, single vs. album recording splits, and sparse crowd-submissions for indie Vocaloid/dōjin tracks significantly reduce hit rates.
  * **Genre taxonomy bias:** As documented in Bogdanov et al. (ISMIR 2019), editorial metadata (`Discogs`/`AllMusic`) skews Western, whereas `j-pop`/`anime` labels appear mostly in `Last.fm`/`tagtraum` folksonomy splits.
* **ToS / Legal:** Completely permissive **CC0** public domain for the static dumps.
* **Python Implementation & `songs.csv` Mapping:**
  * Low-level JSON: `rhythm.bpm` $\rightarrow$ `tempo_bpm`; `lowlevel.average_loudness` & `lowlevel.dynamic_complexity` $\rightarrow$ proxy for `energy`.
  * High-level JSON: `highlevel.danceability.all.danceable` $\rightarrow$ `danceability`; `highlevel.mood_acoustic.all.acoustic` $\rightarrow$ `acousticness`; `highlevel.mood_happy.all.happy` / `mood_sad` / `mood_aggressive` / `mood_relaxed` $\rightarrow$ partial proxy for `valence` & `mood` (note: built with older Essentia v2.1-beta4 SVM/Gaia models, not modern neural embeddings).
  * Effort is high if downloading multi-hundred-GB `.tar.zst` dumps (`zstandard`, `musicbrainzngs`), and brittle if relying on the legacy read-only API.

---

### 3.2 Local Signal Analysis: `librosa` (+ `pyloudnorm`)

* **Primary Sources:** [librosa Official Docs](https://librosa.org/), [librosa GitHub Releases](https://github.com/librosa/librosa/releases), [librosa on PyPI](https://pypi.org/project/librosa/), [pyloudnorm on PyPI](https://pypi.org/project/pyloudnorm/).
* **Native Features Computed:**
  * **Energy / Loudness:** `librosa.feature.rms` computes frame-wise root-mean-square amplitude. `librosa` does **not** compute perceptual integrated loudness (LUFS) natively; pairing it with **`pyloudnorm`** (MIT license, ITU-R BS.1770-4) provides true integrated loudness in LUFS (`pyln.Meter(sr).integrated_loudness(data)`).
  * **Tempo / Rhythm:** `librosa.onset.onset_strength` computes spectral-flux onset envelopes; `librosa.feature.tempo` and `librosa.beat.beat_track` estimate BPM and beat frames.
  * **Spectral & Tonal Features:** `librosa.feature.spectral_centroid`, `spectral_bandwidth`, `spectral_rolloff`, `spectral_flatness`, `zero_crossing_rate`, `mfcc`, `chroma_stft`/`chroma_cqt`, `tonnetz`.
* **What `librosa` Cannot Compute Natively:** Spotify-style high-level perceptual attributes—**`valence`**, **`danceability`**, **`acousticness`**, **`mood`**, and **`genre`**. Estimating those with `librosa` alone requires extracting low-level features (MFCCs, chroma, spectral contrast, HPSS) and training or running your own classifier/regressor.
* **J-Pop / Anison / Vocaloid Considerations:**
  * Works on **100%** of tracks for which you supply an audio clip.
  * **BPM Octave Errors:** Fast anison/Vocaloid tracks (160–200+ BPM) are frequently detected at half-time (80–100 BPM) by default `librosa.feature.tempo` priors (`start_bpm=120.0`) unless `start_bpm` / tempogram bounds are tuned.
* **Rate Limits, Auth & ToS:** Runs 100% offline with zero rate limits or auth. **ISC License** (`librosa`) and **MIT License** (`pyloudnorm`)—maximally permissive for academic or commercial code.
* **Python Implementation & Codec Gotcha:**
  * `librosa` deprecated `audioread` in `v0.10.0` and **removed `audioread` fallback in `v1.0.0`** (2025/2026 stable release), relying on `soundfile` (`libsndfile`). Because `libsndfile` does **not** natively decode iTunes `.m4a` (AAC) previews, Python pipelines must decode `.m4a` via external **`ffmpeg`** (e.g., `subprocess` piping raw WAV/PCM into `soundfile`/`numpy`) before calling `librosa`.
  * Mapping to `data/songs.csv`: `tempo_bpm` (direct), `energy` (normalized RMS or sigmoid/min-max scaled LUFS combined with onset density/spectral centroid), **missing** `valence`, `danceability`, `acousticness`, `mood`, `genre`, `popularity`, `duration_ms`, `release_year`.

---

### 3.3 Local Signal Analysis: `Essentia` & `essentia-tensorflow`

* **Primary Sources:** [Essentia Official Site](https://essentia.upf.edu/), [Algorithm Reference](https://essentia.upf.edu/reference/index.html) ([`LoudnessEBUR128`](https://essentia.upf.edu/reference/std_LoudnessEBUR128.html), [`Danceability`](https://essentia.upf.edu/reference/std_Danceability.html), [`RhythmExtractor2013`](https://essentia.upf.edu/reference/std_RhythmExtractor2013.html), [`KeyExtractor`](https://essentia.upf.edu/reference/std_KeyExtractor.html), [`DynamicComplexity`](https://essentia.upf.edu/reference/std_DynamicComplexity.html)), [Pre-trained Models Docs](https://essentia.upf.edu/models.html), [PyPI `essentia-tensorflow`](https://pypi.org/project/essentia-tensorflow/), [macOS Wheels](https://essentia.upf.edu/downloads/python-wheels/macosx/arm64/).
* **Native DSP Algorithms (`essentia.standard`):**
  * `LoudnessEBUR128` (EBU R128 integrated loudness in LUFS, loudness range, momentary/short-term max), `DynamicComplexity`, `RMS`, `Energy`.
  * `RhythmExtractor2013` (BPM, beat ticks, beat confidence), `OnsetRate`, `SpectralCentroidTime`, `KeyExtractor` (HPCP-based key & major/minor scale).
  * `Danceability` (Detrended Fluctuation Analysis / DFA): outputs values typically in **`[0, ~3]`** (higher = more rhythmically stable/danceable), requiring rescaling (e.g., `min(val / 3.0, 1.0)`) to match `data/songs.csv`'s `[0.0, 1.0]` range.
* **High-Level TensorFlow Models (`essentia-tensorflow`):**
  * **Backbone Embeddings:** `discogs-effnet` (trained on Discogs style taxonomy) and `msd-musicnn` (trained on Million Song Dataset).
  * **Semantic Heads for `songs.csv`:**
    * `deam-msd-musicnn` / `emomusic-msd-musicnn`: predict continuous **valence** and **arousal** on a **`[1, 9]`** scale $\rightarrow$ rescale `(val - 1.0) / 8.0` to `[0.0, 1.0]` for `valence` (and optionally blend arousal with LUFS for `energy`).
    * `danceability-discogs-effnet` and `mood_acoustic-discogs-effnet`: output `[0.0, 1.0]` probabilities directly usable for `danceability` and `acousticness`.
    * `mood_happy`, `mood_sad`, `mood_aggressive`, `mood_relaxed`, `mood_party` (Discogs-EffNet heads): map cleanly to categorical `mood` labels (`happy`, `chill`, `intense`, `melancholic`, etc.).
* **J-Pop / Anison / Vocaloid Coverage & Caveats:**
  * Computes features for **100%** of supplied audio files.
  * **Training Domain Shift:** `msd-musicnn` was trained on the Million Song Dataset (predominantly Western tracks pre-2011), and DEAM/EmoMusic are small Western emotion benchmarks; synthetic Vocaloid vocals and high-energy minor-key anison (sad lyrics/minor harmony with fast euphoric instrumentation) can exhibit calibration drift on `valence` and `mood_acoustic`. Empirically calibrating or inspecting outputs on 15–20 benchmark J-pop/anison tracks is recommended.
* **Rate Limits, Auth & ToS:** 100% local, no rate limits or auth. **Licensing caveat:** Essentia code is **AGPLv3**, and pre-trained TensorFlow weights are **CC BY-NC-ND 4.0** (non-commercial). Fine for a student AI class project, but requires a commercial license from MTG-UPF for commercial deployment.
* **Python Implementation & Platform Effort:**
  * Pre-built `essentia-tensorflow` wheels exist on PyPI for Linux `x86_64` and macOS (`x86_64` and Apple Silicon `arm64`, matching supported Python minor versions). Native Windows `essentia-tensorflow` support is limited/experimental (Windows core wheels require `os.add_dll_directory()` for FFmpeg DLLs, or running via Docker `mtgupf/essentia`).
  * `MonoLoader` uses FFmpeg under the hood on Unix/macOS when compiled with FFmpeg, though transcoding `.m4a` to `.wav` via `ffmpeg` avoids loader quirks.

---

### 3.4 Other Free APIs & Open Datasets

#### ReccoBeats API
* **Primary Sources:** [ReccoBeats Docs](https://reccobeats.com/docs), Base URL `https://api.reccobeats.com/v1`.
* **Capabilities (Verified Live):**
  * **Title & Artist Search (`GET /v1/track/search`, `GET /v1/artist/search`):** Live verification confirmed **`GET /v1/track/search?searchText=ETERNAL+BLAZE`** works directly (`200 OK`, returning `45` paginated hits without auth). Each item includes `id` (ReccoBeats UUID), `trackTitle`, `artists`, `durationMs` (`307546`), `isrc` (`JPKI00507727`), Spotify `href` (`https://open.spotify.com/track/6FghEVkOL58IWEkRFqfKks`), `availableCountries`, and `popularity` (`38`). **`GET /v1/artist/search?searchText=Nana+Mizuki`** also returned `200 OK` (`5` hits).
  * **Catalog Feature Lookup (`GET /v1/track/{id}/audio-features`, `GET /v1/track?ids=...`):** Accepts ReccoBeats UUIDs, **Spotify Track IDs**, or ISRCs. Live call `GET /v1/track/47f0c7ff-7dc7-4490-aeb3-3d58c1ba1670/audio-features` (*ETERNAL BLAZE*) returned **`200 OK`** with: `acousticness: 0.191`, `danceability: 0.363`, `energy: 0.938`, `instrumentalness: 0.0`, `key: 2`, `liveness: 0.326`, `loudness: -3.36`, `mode: 0`, `speechiness: 0.091`, `tempo: 155.057`, `valence: 0.404`. Also exposes `GET /v1/artist/{id}/track`, `GET /v1/album/search`, and `GET /v1/track/recommendation`.
  * **Direct Audio Extraction (`POST /v1/analysis/audio-features`):** Accepts `multipart/form-data` (`audioFile`) in MP3, OGG/Vorbis, AIFF/AIFC, or WAV up to **5 MB** (analyzes the first **30 seconds**, 44.1 kHz recommended) and returns Spotify-style features even for songs missing from the catalog. (iTunes `.m4a` files must be converted to `.mp3`/`.wav` first.)
* **J-Pop / Anison Coverage:** Verified live hits for major J-pop/anison catalog tracks (*ETERNAL BLAZE*, plus covers/remixes); **100% fallback** via `POST /v1/analysis/audio-features` on 30s previews (the `POST` upload endpoint was not tested live here).
* **Auth, Rate Limits & ToS:** Completely free, **no API key or auth**. Rate limits are enforced dynamically via `429 Too Many Requests` + `Retry-After` header (exact request ceiling not published). Terms state data is provided **"as-is"/"as-available" with no SLA**, and abusive traffic may be IP-blocked. Good as a fast prototyping/validation helper, risky as a single point of failure for a graded demo without local `librosa`/`Essentia` fallback.
* **`songs.csv` Mapping & Effort:** Low effort (`requests`). `/v1/track/{id}/audio-features` fills `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness` (already scaled `[0.0, 1.0]` except `tempo` BPM), while `/v1/track/search` also supplies `popularity` and `duration_ms` (**missing** `mood`, `genre`, `release_year`).

#### GetSongBPM API
* **Primary Sources:** [GetSongBPM API Docs](https://getsongbpm.com/api), Base URL `https://api.getsong.co/`.
* **Capabilities:** `/search/` (`type=song|artist|both`) and `/song/` return `tempo` (BPM), `time_sig`, `key_of`, `open_key`, `danceability` (`0–100`), `acousticness` (`0–100`), and `genres`.
* **J-Pop / Anison Coverage:** Spotty outside major global crossovers (YOASOBI, Kenshi Yonezu, Ado); community feedback highlights Western-heavy indexing and weak coverage for Japanese-script titles, indie anison, and Vocaloid.
* **Auth, Rate Limits & ToS:** Requires a free API key (`?api_key=` or `X-API-KEY`), capped at **3,000 requests/hour** (exceeding triggers an automatic 1-hour block). **Mandatory backlink policy:** You must host a visible, verifiable backlink to `GetSongBPM.com` on a live website/app URL submitted during API registration, or your key risks suspension.
* **`songs.csv` Mapping & Effort:** Low Python effort (`requests`), high administrative annoyance for a local class repo unless you deploy a public Streamlit/GitHub Pages URL with the required backlink. Only fills `tempo_bpm`, `danceability` (`/ 100.0`), `acousticness` (`/ 100.0`), and partial `genre`; **missing `energy`, `valence`, `mood`, `popularity`, `duration_ms`, `release_year`**.

#### Last.fm API
* **Primary Sources:** [Last.fm `track.getTopTags`](https://www.last.fm/api/show/track.getTopTags), [Last.fm API ToS](https://www.last.fm/api/tos), Endpoint `https://ws.audioscrobbler.com/2.0/`.
* **Capabilities:**
  * `track.getInfo` returns `listeners`, `playcount`, `duration` (ms—often `0` for long-tail tracks), and top tags.
  * `track.getTopTags` returns crowd-sourced folksonomy tags with frequency weights (`count` 0–100).
* **J-Pop / Anison / Vocaloid Coverage:** Very strong for listener popularity and rich genre/vibe tags (`j-pop`, `anime`, `anisong`, `vocaloid`, `idol`, `energetic`, `happy`, `uplifting`, `electronic`) on popular tracks, though Japanese vs. Romaji artist/title spelling splits scrobble counts unless `autocorrect=1` or canonicalized aliases are used.
* **Auth, Rate Limits & ToS:** Free API key required + descriptive `User-Agent`. Community-documented soft limit is ~**5 requests/sec** (HTTP Error `29` on bursts). **Important ToS Clause:** [Last.fm API ToS](https://www.last.fm/api/tos) explicitly states: *"If you are planning to use our API for commercial or research/academic purposes, or have any questions about the license terms, please drop us a line prior to use via email at partners@last.fm."* (Students using it in a class/academic project should send that notification email before bulk harvesting.)
* **`songs.csv` Mapping & Effort:** Low effort (`requests`). Best free source for **`popularity`** (e.g., log-scaling `listeners` across your dataset to `0–100`) and **`genre`/`mood`** tag mapping, with **zero** numeric audio features (`energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`).

#### Kaggle Spotify Datasets
* **Primary Sources:**
  * [`maharshipandya/spotify-tracks-dataset`](https://www.kaggle.com/datasets/maharshipandya/spotify-tracks-dataset) (~114,000 tracks, 125 genres): includes `track_genre` values **`j-pop`**, **`j-idol`**, **`j-rock`**, **`anime`**, plus `popularity`, `duration_ms`, `danceability`, `energy`, `loudness`, `acousticness`, `valence`, `tempo`. **Missing:** `release_year` and `mood`, plus **0% post-2022 releases** and no dedicated `vocaloid` genre. License listed as Open Database (contents owned by original authors / derived from Spotify API).
  * [`rodolfofigueroa/spotify-12m-songs`](https://www.kaggle.com/datasets/rodolfofigueroa/spotify-12m-songs) (1.2M+ tracks, `tracks_features.csv`): includes `year`/`release_date`, `duration_ms`, and Spotify audio features. **Missing:** `popularity`, `genre`, `mood`, and post-2020/2021 releases; Kaggle license listed as **Unknown**.
  * [`vatsalmavani/spotify-dataset`](https://www.kaggle.com/datasets/vatsalmavani/spotify-dataset) (~170k tracks, 1921–2020): includes `year`, `popularity`, and audio features, **missing** `genre`, `mood`, and post-2020 J-pop depth.
* **ToS / Legal:** Redistributing Spotify API exports conflicts with Spotify's Terms against bulk data redistribution, though downloading Kaggle CSVs for offline student experimentation is common practice.
* **`songs.csv` Mapping & Effort:** Lowest effort (`pandas.read_csv`), great for **English pop baseline tracks (pre-2022)** or **sanity-checking Essentia/librosa calibration** against historical Spotify ground truth, **insufficient alone** for modern 2023–2026 J-pop/anison/Vocaloid.

---

## 4. Detailed Evaluation: Legal/Free Audio & 30-Second Preview Sources

Because `Essentia` and `librosa` require raw audio waveforms, you need a legal way to obtain 30-second previews (or full audio) for signal extraction.

### 4.1 iTunes Search API (`previewUrl`)

* **Primary Sources:** [iTunes Search API Archive Docs](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/iTuneSearchAPI/index.html), Endpoint `https://itunes.apple.com/search` & `https://itunes.apple.com/lookup`.
* **Audio Format:** `previewUrl` serves a **30-second `.m4a` file** (AAC-LC stereo, 44.1 kHz, typically **128 kbps**).
* **J-Pop / Anison / Vocaloid Coverage (`country=JP` vs. `country=US`):**
  * **`country=JP` is mandatory for Japanese tracks:** Domestic Japanese labels (e.g., King Records, Pony Canyon, Sony Music Japan exclusives, Lantis, Vocaloid/dōjin distributions on KarenT/TuneCore Japan) frequently restrict distribution to Japan or use native Kanji/Kana titles only indexed in `country=JP`.
  * **`country=US` for English pop & global crossovers:** Misses Japan-exclusive singles and returns Romanized/translated titles for international acts (e.g., YOASOBI, Kenshi Yonezu, Ado).
  * **Recommended dual-store strategy:** Query `country=JP` first for J-pop/anison/Vocaloid candidates, falling back to `country=US` for English pop (or cross-store matching).
* **Auth & Rate Limits:** **No API key or auth required.** Soft rate limit of **~20 requests/minute** per IP (unwritten/empirical operational ceiling; no SLA). Space requests by $\ge 3\text{s}$ and cache metadata JSON locally.
* **ToS / Legal Restrictions:** Apple's Search API & Affiliate promotional guidelines require that `previewUrl` streams be **stream-only** (*"must not be downloaded, saved, cached, or synchronized"*), used solely to promote iTunes/Apple Music content proximate to an Apple Store badge with *"provided courtesy of iTunes"* attribution. **Practical implication for a student pipeline:** Fetch the `previewUrl` bytes into an in-memory buffer (`io.BytesIO`) or transient `tempfile` piped to `ffmpeg`, compute the numeric features (`energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`), discard the audio bytes immediately, and **do not commit or redistribute `.m4a` clips in Git**.
* **Python Implementation & `songs.csv` Mapping:**
  * Metadata returned directly (**verified live `200 OK`** on `term=ETERNAL+BLAZE+Nana+Mizuki&country=JP`, returning `水樹奈々` with `.plus.aac.p.m4a` `previewUrl`, `trackTimeMillis: 312487`, `primaryGenreName: "J-Pop"`, and `releaseDate` — note `limit=1` matched a 2025 live album version first, so disambiguate live vs. studio versions when ingesting): `trackName` (`title`), `artistName` (`artist`), `primaryGenreName` (`genre`, e.g., `"J-Pop"`, `"Anime"`, `"Pop"`), `trackTimeMillis` (`duration_ms`—exact full track length, not 30s!), `releaseDate[:4]` (`release_year`).
  * Audio decode caveat: Since `librosa>=1.0` removed `audioread`, pipe `.m4a` through **`ffmpeg`** (`ffmpeg -i pipe:0 -f wav -ar 44100 -ac 1 pipe:1`) before feeding NumPy arrays to `librosa`/`Essentia`.

---

### 4.2 Deezer API (`preview`)

* **Primary Sources:** [Deezer API: Track](https://developers.deezer.com/api/track), [Deezer API: Search](https://developers.deezer.com/api/search), [Deezer Terms of Use](https://developers.deezer.com/termsofuse), Base URL `https://api.deezer.com`.
* **Audio Format & Extra Track Fields:**
  * `GET /search?q=...` or `GET /search/track?q=...` returns `preview` (30-second **MP3** URL), `duration` (seconds $\rightarrow \times 1000$ for `duration_ms`), and `rank` (Deezer popularity integer).
  * `GET /track/{id}` additionally exposes `release_date` (`YYYY-MM-DD` $\rightarrow$ `release_year`), `bpm` (`float`), and `gain` (ReplayGain dB `float`).
* **J-Pop / Anison / Vocaloid Coverage & `"bpm": 0` Caveat:**
  * Covers mainstream global J-pop and major anime themes, but lags behind iTunes `country=JP` on Japan-exclusive idols, older King Records/Lantis anison, and independent Vocaloid releases due to regional licensing.
  * **Frequent `"bpm": 0` bug:** Community issues confirm Deezer's catalog frequently returns `"bpm": 0` (missing) on Asian/J-pop and even mainstream tracks—never rely on Deezer's `bpm` without local DSP fallback.
* **Auth & Rate Limits:** **No auth required** for public catalog searches and track endpoints. Rate limit is **50 requests per 5 seconds** per IP (returns Quota error code `4` if exceeded).
* **ToS / Legal Restrictions:** Free for personal apps/websites under Deezer Developer Terms of Use. Deezer's FAQ explicitly allows caching basic text metadata (titles, IDs), **forbids caching images**, and restricts unlogged API audio access to 30s previews (no permanent audio storage/redistribution).
* **Python Implementation & `songs.csv` Mapping:**
  * Low effort (`requests`). Live check on `GET /search?q=ETERNAL+BLAZE+Nana+Mizuki&limit=1` returned **`200 OK`** (`total: 5`) with `id: 2230713757`, `isrc: "JPKI00507727"`, `duration: 307`, `rank: 156254`, `artist.name: "水樹奈々"`, and a signed `.mp3` `preview` URL (`cdnt-preview.dzcdn.net`). 30s MP3 previews are directly decodable by `ffmpeg` or modern `libsndfile >= 1.1` / `soundfile` without AAC codec issues.
  * Note that `rank` is an unbounded platform-specific integer (often $10^5\text{–}10^6$), so it must be log-normalized or percentile-ranked to `[0, 100]` for `data/songs.csv`'s `popularity` column.

---

### 4.3 YouTube / `yt-dlp`

* **Primary Sources:** [YouTube Terms of Service](https://www.youtube.com/t/terms), [YouTube API Services Developer Policies](https://developers.google.com/youtube/terms/developer-policies), [`yt-dlp` GitHub Repository](https://github.com/yt-dlp/yt-dlp).
* **Coverage:** Effectively near-100% for J-pop, anison, idol live performances, Niconico-crossposted Vocaloid tracks, and English pop.
* **ToS & Legal Restrictions (Blocking):**
  * **YouTube ToS ("Permissions and Restrictions"):** Explicitly forbids users from *"access[ing], reproduc[ing], download[ing], distribut[ing]... any part of the Service or any Content except: (a) as expressly authorized by the Service; or (b) with prior written permission from YouTube and, if applicable, the respective rights holders,"* and forbids circumventing technical copy-protection measures.
  * **YouTube API Services Developer Policies:** Explicitly prohibit *"separating, isolating, or modifying the audio or video components"* of YouTube audiovisual content or offering/extracting standalone audio streams/MP3s.
* **Operational / Technical Fragility:** Unauthenticated `yt-dlp` extraction routinely hits `"Sign in to confirm you're not a bot"` / HTTP `403` blocks (guest rate caps ~300 req/hr, datacenter/VPN IP filtering), forcing fragile workarounds like Node.js PO Token (BotGuard) plugins or `--cookies-from-browser` (which risks personal Google account suspension).
* **Verdict:** **Do not use.** Violates YouTube ToS and carries high grading/legal/account-ban risk when iTunes Search API (`country=JP`) + Deezer API solve preview sourcing cleanly.

---

## 5. Exact Column Mapping Strategy for `data/songs.csv`

[`src/recommender.py`](../src/recommender.py) strictly expects `energy`, `valence`, `danceability`, and `acousticness` in `[0.0, 1.0]`, `popularity` in `[0, 100]` (`int`), `duration_ms` (`int`), `release_year` (`int`), `tempo_bpm` (`float`), and exact lowercase strings for `genre` and `mood`.

| `data/songs.csv` Column | Recommended Primary Free Source | Fallback / Normalization Rule |
|---|---|---|
| `id` | Auto-increment `int` | Sequential `1..N` (preserve external IDs in a separate lookup/provenance file if needed) |
| `title`, `artist` | Seeded curated list (or iTunes `trackName` / `artistName`) | Query iTunes `country=JP` (J-pop/anison/Vocaloid) & `country=US` (English pop) |
| `genre` | Seeded taxonomy / iTunes `primaryGenreName` / Last.fm `track.getTopTags` | Normalize lowercase (`j-pop`, `anime`, `vocaloid`, `idol`, `pop`, `rock`) to match `UserProfile.favorite_genre` |
| `mood` | `Essentia` TF mood classifiers (`mood_happy`, `mood_sad`, `mood_aggressive`, `mood_relaxed`, `mood_party`) + Last.fm tags | Map highest-scoring head / tag to repo mood vocabulary (`happy`, `intense`, `chill`, `moody`, `melancholic`, `nostalgic`, etc.) |
| `energy` | `Essentia` `LoudnessEBUR128` (or `pyloudnorm` LUFS) + `RMS` / onset density + `deam`/`emomusic` arousal | Combine min-max/sigmoid-scaled integrated LUFS (e.g., clipping $[-25, -4]$ LUFS $\rightarrow [0, 1]$) with arousal (`(arousal - 1)/8`) |
| `tempo_bpm` | `Essentia.RhythmExtractor2013` or `librosa.feature.tempo` | Cross-check against Deezer `bpm` (if `> 0`) or GetSongBPM; watch for $0.5\times / 2\times$ octave errors on fast (>160 BPM) anison |
| `valence` | `Essentia` `deam-msd-musicnn` or `emomusic-msd-musicnn` (or ReccoBeats `/v1/analysis/audio-features` check) | Rescale `[1, 9]` prediction via `(val - 1.0) / 8.0` clamped to `[0.0, 1.0]` |
| `danceability` | `Essentia` `danceability-discogs-effnet` (preferred) or `Essentia.Danceability` DFA | If using DFA `Danceability` (`[0, ~3]`), normalize `min(dfa / 3.0, 1.0)`; TF model already outputs `[0, 1]` |
| `acousticness` | `Essentia` `mood_acoustic-discogs-effnet` | Probability in `[0.0, 1.0]` |
| `popularity` | Last.fm `track.getInfo` (`listeners`) / Deezer `rank` | Log-scale `listeners` across catalog via min-max to `int` in `[0, 100]` |
| `duration_ms` | **Metadata only:** iTunes `trackTimeMillis` or Deezer `duration * 1000` | **Never** use preview clip length (`30000` ms) |
| `release_year` | **Metadata only:** iTunes `releaseDate[:4]` or Deezer `release_date[:4]` | Integer year (`int`) |

### Critical Engineering Caveat: 30-Second Preview Bias
iTunes and Deezer previews typically sample a **30-second high-energy highlight (often the chorus / サビ)** rather than the quiet intro/verse. Consequently:
1. Features computed from a 30s chorus preview will skew slightly higher in `energy` and `danceability` than full-track Spotify averages—apply consistent scaling across **both** J-pop and English pop tracks using the **same** 30s preview pipeline so recommender comparisons remain apples-to-apples.
2. Always populate `duration_ms` from metadata (`trackTimeMillis` / Deezer `duration`), never from `len(y) / sr`.
3. Do not store or commit downloaded `.m4a`/`.mp3` files in the repository; stream them into temporary memory/buffers for feature extraction and discard immediately.

---

## 6. Unverified Items & Limitations

Single-track live probes verified ReccoBeats (`/v1/track/search`, `/v1/artist/search`, `/v1/track/{id}/audio-features`), iTunes Search (`country=JP`), and Deezer (`ETERNAL BLAZE`). The following operational points remain **unverified** directly and should be tested in a batch smoke test over the full seed playlist:
1. **ReccoBeats `POST /v1/analysis/audio-features` & batch hit rate:** Audio upload extraction was not tested live, and bulk J-pop/anison/Vocaloid catalog recall across all 100+ seed tracks remains to be measured.
2. **ReccoBeats numeric rate limit:** Exact requests-per-minute threshold before triggering `429 Too Many Requests` (undocumented in `reccobeats.com/docs`).
3. **Exact J-pop / Vocaloid recall across AcousticBrainz, GetSongBPM, iTunes JP, and Deezer:** Quantitative hit rates across the full seed playlist (including Vocaloid/VTuber/idol tracks) could not be measured in a single-track probe.
4. **Local `essentia-tensorflow` wheel match:** Compatibility against the exact Python version inside this repo's `.venv` (whether macOS/Linux wheels match the active interpreter without pinning Python 3.10/3.11/3.12).
