# Wotagei (ヲタ芸 / 打ち曲) Song Data Sources Research

**Context:** Replacing the 18-row synthetic `data/songs.csv` dataset (`id,title,artist,genre,mood,energy,tempo_bpm,valence,danceability,acousticness,popularity,duration_ms,release_year`) with a real recommender dataset centered on **wotagei** (ヲタ芸 — cyalume/lightstick cheering choreography) and **uchikyoku** (打ち曲 — tracks well-suited for wotagei routines), while supporting broader J-pop, anison, Vocaloid, and English-language crossover tracks.

**Verification & Methodology Note:**
- Primary sources were investigated via domain-restricted web searches **and** direct unsandboxed HTTP verification checks against live endpoints (`wikiwiki.jp/neosr/*`, `wikiwiki.jp/wotagei/*`, `wikiwiki.jp/otasugei/*`, `snapshot.search.nicovideo.jp`, `vocadb.net/api`, `api.animethemes.moe`, and `anison.info/data/download.html`).
- Sources whose live responses were parsed directly (`wikiwiki.jp/neosr/打ち曲`, `wikiwiki.jp/neosr/ヲタ芸の基本的な構成`, Niconico Snapshot Search API v2, VocaDB API, `anison.info/data/download.html`) are marked **Verified (live HTTP)**; APIs/docs checked via primary documentation and search without direct live payload parsing (Uta-Net HTML, X API v2 live tweet counts, MusicBrainz live tag counts, AniDB, and `api.animethemes.moe` which returned `HTTP 503`) have any unfetched counts or timestamps explicitly marked **Unverified**.

---

## 1. Executive Summary & Fit for `data/songs.csv`

1. **No single free source provides both curated wotagei suitability labels (`打ち曲`) and continuous audio features (`energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`).**
   - Community wikis and blogs (`wikiwiki.jp/neosr/打ち曲`, `wotageitimes.com`, `note.com`) identify *which* tracks are wotagei favorites (and sometimes human-estimated BPM), but lack APIs, normalized IDs, and continuous perceptual/audio features.
   - Structured music/anison/Vocaloid APIs (**VocaDB**, **MusicBrainz**, **AnimeThemes**, **Niconico Snapshot Search API v2**) supply track IDs, titles, artists, durations, release years, and popularity/tag metadata at scale, but **lack audio features** and mostly **lack explicit `打ち曲` tags** (aside from noisy performance-video tags on Niconico).
2. **Best Free Candidate Pipeline for `data/songs.csv`:**
   - **Wotagei Seed & Community Signals:** Curated lists from `ヲタ芸 Wiki*` (`https://wikiwiki.jp/neosr/打ち曲`, verified live `200 OK`, 90 song entries), `wotageitimes.com`, `note.com` anikura BPM guides, and `Niconico Snapshot Search API v2` (`ヲタ芸` exact tag verified live at `1,594` videos).
   - **Structured Track Catalog & Metadata:** **VocaDB API** (best coverage of Vocaloid/utaite/indie J-pop, includes `lengthSeconds`, `publishDate`, `favoritedTimes`, `tags`, and Niconico/YouTube `pvs`), **AnimeThemes API** (anison OP/ED catalog with `anime.year`, plus direct `.ogg` audio streams; note transient `503` observed during check), **Anison Generation** (`anison.info/data/download.html` ZIP/CSV archives), and **MusicBrainz** (broad J-pop + English crossover coverage with `length` in ms and `first-release-date`).
   - **Audio Feature Gap:** Columns `energy`, `tempo_bpm`, `valence`, `danceability`, and `acousticness` must be filled by downstream signal analysis or audio-feature APIs (covered separately in `research/audio-feature-sources.md`).

---

## 2. Summary Comparison Table

All 13 columns of `data/songs.csv` are evaluated:
`id`, `title`, `artist`, `genre`, `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`, `popularity`, `duration_ms`, `release_year`.

| Source | Free / Paid | Access Method | `data/songs.csv` Column Coverage | Missing `data/songs.csv` Columns | Language Coverage | Last-Updated Date | Verification Status |
|---|---|---|---|---|---|---|---|
| **Japanese Fan Wiki: `wikiwiki.jp/neosr/打ち曲` (and `otasugei`)** | Free | Manual web browse (no API/dump; `otasugei` anti-bot `429` gate; `wikiwiki.jp/wotagei/` returns `404`) | Title-only list on `打ち曲` (`title` for 90 entries, 89 distinct); linked move/song subpages exist | `id`, `artist` (absent on `打ち曲` index), `genre`, `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`, `popularity`, `duration_ms`, `release_year` | Japanese-only (includes select English-title anison/J-rock entries like `I SAY YES`, `The Beginning`) | `打ち曲`: **2024-05-11 19:56:32**; `ヲタ芸の基本的な構成`: **2021-04-18 15:30:49** | **Verified (live HTTP `200 OK`)** for `neosr/打ち曲` & `neosr/ヲタ芸の基本的な構成`; `wikiwiki.jp/wotagei/` **`404`**; `otasugei/打ち曲` **`429`** |
| **AtWiki (`w.atwiki.jp`) Wotagei / Anikura Pages** | Free | Manual web browse (no API/dump) | Very sparse prose: occasional `title`, `artist` in event/call articles | All numeric & ID columns (`id`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`, `popularity`, `duration_ms`, `release_year`), structured `genre`/`mood` | Japanese-only | **Unverified** (no live scrape; event wikis contain legacy + ongoing edits) | **Verified** existence of `アニソン系クラブイベントWiki@関東` & `アニヲタWiki(仮)`; dedicated wotagei song DB **Not Found / Unverified** |
| **Uta-Net (`uta-net.com`)** | Free (browse); **scraping prohibited** | Manual web browse only (no public API; ToS explicitly bans scraping) | `title`, `artist`, `popularity` (proxy: `表示回数`/favorites), `release_year` (from `発売日`) | `id` (only URL slug), `genre`, `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`, `duration_ms`; no wotagei tags | Mostly Japanese (includes select English songs licensed in Japan) | Daily lyric catalog updates; Terms updated Aug 2023 (**Verified**) | **Verified** via `uta-net.com` docs/pages; live page fetch **Unverified** |
| **Niconico Snapshot Search API v2** | Free (no key; `User-Agent` required) | REST JSON GET API (`api.search.nicovideo.jp` / `snapshot.search.nicovideo.jp`) | `id` (`contentId`), `title` (noisy video title), `genre` (`genre`/`tags`), `popularity` (`viewCounter`, `mylistCounter`, `likeCounter`), `duration_ms` (`lengthSeconds*1000`), `release_year` (`startTime` upload year — proxy only) | `artist` (unstructured in title/desc), `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness` | Primarily Japanese (tracks tagged `ヲタ芸` / `打ち曲` / `VOCALOID`) | Snapshot **`last_modified`: `2026-11-17T06:38:10+09:00`** (daily 05:00 JST); docs updated **2026-04-15** | **Verified (live HTTP `200 OK`)**: exact tag `ヲタ芸` returned `totalCount: 1594` (top hit `sm12278556`, `647,120` views) |
| **Niconico Pedia (`dic.nicovideo.jp`)** | Free | Manual web browse (no API) | Prose only: occasional exemplar `title`/`artist` inside `ヲタ芸`/`PPPH` articles (`打ち曲` article does not exist) | All structured/numeric fields | Japanese-only | Ongoing wiki edits; exact timestamp **Unverified** | **Verified**: `ヲタ芸`/`PPPH` exist; `/a/打ち曲` confirmed non-existent |
| **X (Twitter) (`#打ち曲`, `#ヲタ芸`)** | **Paid** for API (Pay-Per-Use `$0.005`/read; Free read retired); Free manual browse | Manual search (X API v2 requires credits; scraping violates ToS) | Unstructured text mentions of `title`, `artist`, occasional `tempo_bpm` | All structured/numeric fields | Multilingual (mostly Japanese for `#打ち曲`) | Real-time user posts (continuously updated) | **Verified** X API v2 Pay-Per-Use pricing via `docs.x.com` & search; live tweets **Unverified** |
| **Public Community Curations (Wotagei Times, `note.com`, YouTube, Spotify, Google Sheets, GitHub)** | Free (manual / platform ToS apply) | Manual HTML reading, playlist exports, ad-hoc sheets | `title`, `artist`, partial `tempo_bpm` (BPM bands or human BPM notes), informal `genre` | `id`, `mood`, `energy`, `valence`, `danceability`, `acousticness`, `popularity`, `duration_ms`, `release_year` | Primarily Japanese (`wotageitimes`, `note.com`); multilingual on YouTube/WWC | `wotageitimes` survey **2019-11-22**; WWC `2026` rules active; `note.com`/playlists ongoing | **Verified** for `wotageitimes.com`, `note.com`, `wotagei.org`; no canonical open CSV found (**Unverified** sheets) |
| **MusicBrainz** | Free (`CC0 1.0` core; `CC BY-NC-SA 3.0` tags/genres/ratings) | REST WS v2 API (XML/JSON, 1 req/s) + twice-weekly JSON/PG dumps | `id` (MBID), `title`, `artist` (`artist-credit`), `genre` (`genres`/`tags`), `mood` (very sparse tags), `popularity` (sparse `rating`), `duration_ms` (`length`), `release_year` (`first-release-date`) | `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`; no active `wotagei` genre/series | Multilingual (global + Japanese) | Live continuous edits; JSON dumps updated **Wednesdays & Saturdays** | **Verified** via `musicbrainz.org` docs; live `tag/wotagei` count **Unverified** |
| **VocaDB (+ UtaiteDB / TouhouDB)** | Free (open source; CC metadata) | REST JSON API (`/api/songs`, OpenAPI/Swagger); community DB dumps on GitHub | `id`, `title` (`name`), `artist` (`artistString`), `genre` (`tags`, `songType`), `mood` (partial `tags`), `popularity` (`favoritedTimes`, `ratingScore`), `duration_ms` (`lengthSeconds*1000`), `release_year` (`publishDate`), plus `pvs` URLs | `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`; `tags?query=wotagei` returns **`totalCount: 0`** | Multilingual (Japanese, Romaji, English) | Live continuous edits (active); third-party dump currency **Unverified** | **Verified (live HTTP `200 OK`)** on `/api/songs` (`fields=PVs,Artists,Tags`) and `/api/tags?query=wotagei` (`0` hits) |
| **Anison Generation (`anison.info`)** | Free (**personal, non-commercial use only**) | Web search + 4 downloadable CSV ZIPs (`download/program.zip`, `anison.zip`, `sf.zip`, `game.zip`); no official API | `id` (`楽曲ID`, `番組ID`), `title` (`楽曲名`), `artist` (`歌手名`), `genre` (`番組分類` & `摘要(OP/ED別)` only), `release_year` (`放映開始日` in `program.zip`) | `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`, `popularity`, `duration_ms`; no wotagei tags | Japanese-only | `download.html` `Last-Modified: 2023-09-05`; CSVs updated monthly on **5th/10th/15th/20th** | **Verified (live HTTP `200 OK`)** on `download.html` with exact column lists & ZIP links; internal CSV encoding **Unverified** |
| **AnimeThemes (`api.animethemes.moe`)** | Free (open source) | REST JSON:API + GraphQL + server DB dumps (~90 req/min limit) | `id`, `title` (`song.title`), `artist` (`artists.name`), `genre` (only OP/ED type), `release_year` (`anime.year`), plus `.ogg` audio URLs | `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`, `popularity`, `duration_ms` (audio is TV-size ~90s; `duration` field absent from observed schema) | Multilingual (Japanese/Romaji titles, English docs) | Continuous (active anime seasons); exact date **Unverified** | **Verified** via `animethemes.moe` & GitHub; live probe returned **`HTTP 503`** (Cloudflare/nginx challenge or temporary outage) |
| **AniDB (`anidb.net`)** | Free (non-commercial; strict client reg & caching) | HTTP XML API (1 req / >=2s), UDP API, `animetitles.xml.gz` | Minimal for songs: anime `id`/`titles`/`startdate`/`tags` only; song tracks not cleanly exposed in HTTP `request=anime` XML | `title` (song), `artist`, `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`, `popularity`, `duration_ms` (song), `release_year` (song) | Multilingual (`x-jat`, `ja`, `en`) | Continuous anime catalog edits; exact date **Unverified** | **Verified** HTTP XML schema scope via `wiki.anidb.net` / Go bindings; unsuitable for song datasets |

---

## 3. Detailed Per-Source Findings

### 3.1 Japanese Fan Wikis

#### A. WIKIWIKI: `wikiwiki.jp/neosr/打ち曲`, `wikiwiki.jp/otasugei/`, and `wikiwiki.jp/wotagei/`
* **URLs & Live Status:**
  * Live verified pages: **`https://wikiwiki.jp/neosr/打ち曲`** (**`200 OK`** — *打ち曲 - ヲタ芸 Wiki\**) and **`https://wikiwiki.jp/neosr/ヲタ芸の基本的な構成`** (**`200 OK`** — *ヲタ芸の基本的な構成 - ヲタ芸 Wiki\**).
  * Other WIKIWIKI paths checked: `https://wikiwiki.jp/otasugei/%E6%89%93%E3%81%A1%E6%9B%B2` returned **`HTTP 429`** (*アクセス確認中 - WIKIWIKI* anti-bot gate), and `https://wikiwiki.jp/wotagei/` (plus subpages) returned **`HTTP 404 Not Found`** (despite older search snippets).
* **Free or Paid:** Free to read.
* **Access Method:** Manual HTML web browsing (WIKIWIKI provides no JSON API or bulk dataset dump; automated requests may hit `429` anti-bot verification on some subdomains).
* **Fields Available vs. `data/songs.csv`:**
  * On live `https://wikiwiki.jp/neosr/打ち曲`: **90 song entries** (89 distinct titles; `Crossing Field` appears twice; 3 `?` uncreated stubs) arranged alphabetically (`ア`–`ワ`, `the`) containing **song titles only** (e.g., `ETERNAL BLAZE`, `Endless Story`, `Only My Railgun`, `Synchrogazer`, `Crossing Field`, `空奏列車`, `地球最後の告白を`, `ローリンガール`, `ブラック★ロックシューター`, `ルカルカ★ナイトフィーバー`, `千本桜`, `六兆年と一夜物語`, `The Beginning`). No artist, BPM, release year, or numeric audio features are tabulated on the index page itself.
  * **Missing on index page:** `id`, `artist`, `genre`, `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`, `popularity`, `duration_ms`, `release_year` (though move/song subpages exist in the wiki menu).
* **Languages:** Japanese wiki UI/headings (`〈ア行から始まる曲〉`), listing both Japanese and select English-title tracks (`I SAY YES`, `The Beginning`, `Joint`, `Paradise Lost`).
* **Last-Updated Date:** Verified from live page HTML: `https://wikiwiki.jp/neosr/打ち曲` `Last-modified: 2024-05-11 (土) 19:56:32`; `https://wikiwiki.jp/neosr/ヲタ芸の基本的な構成` `Last-modified: 2021-04-18 (日) 15:30:49`.
* **Verification Status:** `wikiwiki.jp/neosr/打ち曲` and `wikiwiki.jp/neosr/ヲタ芸の基本的な構成` **Verified (live HTTP `200 OK`)**; `wikiwiki.jp/wotagei/` verified dead (**`404`**); `wikiwiki.jp/otasugei/打ち曲` gated (**`429`**, contents **Unverified**).

#### B. AtWiki (`w.atwiki.jp`) and Seesaa Wiki (`seesaawiki.jp`) Wotagei Pages
* **URLs:** Scattered across `w.atwiki.jp` wikis such as **アニソン系クラブイベントWiki@関東** and **アニヲタWiki(仮)** (`ヲタ芸` / song call articles), plus idol call pages on `seesaawiki.jp`. No dedicated, structured `w.atwiki.jp` or `seesaawiki.jp` database of `打ち曲` with BPM/song tables was found.
* **Free or Paid:** Free.
* **Access Method:** Manual HTML browsing only.
* **Fields Available vs. `data/songs.csv`:** Unstructured prose mentioning occasional `title` and `artist` (e.g., `ロマンティック 浮かれモード` by Miki Fujimoto, Vocaloid/anison club anthems) and event-type context (`打ちバカ`, `原曲`). All numeric and ID columns of `data/songs.csv` are **missing**.
* **Languages:** Japanese-only.
* **Last-Updated Date:** **Unverified** (live pages not fetched; `アニソン系クラブイベントWiki@関東` covers historical and ongoing events).
* **Verification Status:** **Verified** existence of event/encyclopedia coverage on `w.atwiki.jp`; structured song-list wiki on AtWiki/Seesaa **Unverified / Not Found**.

---

### 3.2 Uta-Net (歌ネット)
* **URLs:** `https://www.uta-net.com/`, song pages `https://www.uta-net.com/song/<id>/`, Terms of Service `https://www.uta-net.com/info/kiyaku.html`.
* **Free or Paid:** Free for browser viewing; **commercial/automated extraction prohibited**.
* **Access Method:** Manual web search/browse only.
  * **No Public API:** Uta-Net provides **no official public API** (searches on `uta-net.com` for `API` match only songwriter credits).
  * **Strict Anti-Scraping ToS:** The Terms of Service (`https://www.uta-net.com/info/kiyaku.html`, revised **August 2023**) explicitly forbid reproducing, transmitting, or **scraping (`スクレイピング`)** any site content. Lyrics are delivered via streaming mode under JASRAC and NexTone licenses.
* **Fields Available vs. `data/songs.csv`:**
  * `title` (`曲名`)
  * `artist` (`歌手名`; plus `作詞`, `作曲`, `編曲`)
  * `release_year` (derivable from `発売日`)
  * `popularity` (proxy: `表示回数` lyric view count and `お気に入り` favorites count)
  * `id` (site-internal numeric URL slug only; not an ISRC/MBID)
  * **Missing:** `genre`, `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`, `duration_ms`. **Crucially, Uta-Net has no `ヲタ芸` or `打ち曲` classification.**
* **Languages:** Primarily Japanese (J-pop, anison, idol, Vocaloid), plus select English-lyric tracks commercially distributed in Japan and licensed via JASRAC/NexTone.
* **Last-Updated Date:** Catalog updated continuously with new commercial releases; Terms revision verified as **August 2023 (`2023年8月`)**.
* **Verification Status:** **Verified** via `uta-net.com` search index and Terms summary; direct HTML DOM scraping deliberately not pursued due to ToS prohibition and sandbox proxy block.

---

### 3.3 Niconico (ニコニコ動画 Snapshot Search API v2 & ニコニコ大百科)

#### A. Niconico Video Snapshot Search API v2 (ニコニコ動画 スナップショット検索API v2)
* **URLs:**
  * Official docs: `https://site.nicovideo.jp/search-api-docs/snapshot` (and `.html`)
  * Search endpoints: `https://api.search.nicovideo.jp/api/v2/snapshot/video/contents/search` / `https://snapshot.search.nicovideo.jp/api/v2/snapshot/video/contents/search` (HTTP retired December 2023; FQDN updated March 1, 2024)
  * Version check: `https://snapshot.search.nicovideo.jp/api/v2/snapshot/version`
* **Free or Paid:** Free, no API key or authentication required. Requires a descriptive `User-Agent` header (up to 40 chars recommended) or `_context` parameter.
* **Access Method:** REST JSON `GET` API.
  * Supports exact tag matching via `targets=tagsExact` for `q=ヲタ芸` or `q=打ち曲`, pagination up to `_limit=100` and `_offset=100000` (partition larger queries using `filters[startTime][gte/lte]`), sorting (`_sort=-viewCounter`), and JSON filters (`jsonFilter`).
  * **Rate / Etiquette Limits:** Wait at least as long as the previous response duration between sequential requests; wait **>= 5 minutes** if HTTP `503` occurs.
* **Fields Available vs. `data/songs.csv`:**
  * `id` -> `contentId` (e.g., `sm9`, though this is a **video ID**, not a canonical recording ID)
  * `title` -> `title` (**caveat:** performance video title, e.g., `"【ヲタ芸】<Song Name> 打ってみた"`, requiring regex/NLP parsing to isolate song title)
  * `artist` -> **Unstructured** (must be extracted from `title` or `description`; `userId`/`channelId` identifies the *choreographer/uploader*, not the musician)
  * `genre` -> `genre` (Niconico video category, e.g., `ダンス` / `音楽・サウンド`), `categoryTags`, `tags` (space-delimited tags including `ヲタ芸`, `打ち曲`, `VOCALOID`, `アニソン`)
  * `popularity` -> `viewCounter`, `mylistCounter`, `likeCounter`, `commentCounter`
  * `duration_ms` -> `lengthSeconds * 1000` (**caveat:** performance video length, which may be a cut, intro, or medley rather than the studio track duration)
  * `release_year` -> `startTime` (ISO 8601 upload timestamp — reflects when the *wotagei video* was uploaded, **not** when the underlying song was released)
  * `contentType` -> `'long'` or `'short'` (added **2026-04-15**)
  * **Missing:** `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`.
  * **Tag Coverage Caveat:** The tag `ヲタ芸` has **1,500+ videos** (plus many more under `ヲタ芸打ってみた` / `サイリウムダンス`), whereas `打ち曲` as a standalone video tag is **much sparser**. Because most hits are choreography videos rather than official music uploads, Niconico works best as a **popularity signal / co-occurrence miner** ("which songs do performers actually dance to?") paired with VocaDB or MusicBrainz for clean song metadata.
* **Languages:** Primarily Japanese (some multilingual Vocaloid/NCS/English titles in descriptions).
* **Last-Updated Date:** Snapshot version endpoint verified live at `last_modified: "2026-11-17T06:38:10+09:00"` (refreshes daily at **05:00 JST**); official API changelog records updates on **2024-03-01** (FQDN change) and **2026-04-15** (`contentType` field added).
* **Verification Status:** **Verified (live HTTP `200 OK`)** against `snapshot.search.nicovideo.jp` (`q=ヲタ芸&targets=tagsExact` returned `meta.totalCount = 1594`; top result `sm12278556` with `647,120` views, `sm3864` with `518,627` views).

#### B. Niconico Pedia (ニコニコ大百科 — `dic.nicovideo.jp`)
* **URLs:** `https://dic.nicovideo.jp/a/%E3%83%B2%E3%82%BF%E8%8A%B8` (`ヲタ芸`), `PPPH`. Notably, **`https://dic.nicovideo.jp/a/%E6%89%93%E3%81%A1%E6%9B%B2` (`打ち曲`) does not exist as a standalone article** as of 2026.
* **Free or Paid:** Free.
* **Access Method:** Manual HTML web browse (no API).
* **Fields Available vs. `data/songs.csv`:** Prose definitions, history (1980s–2003 evolution), and move descriptions (`OAD`, `PPPH`, `ロマンス`, `サンダースネイク`), citing a small number of canonical songs (`ロマンティック 浮かれモード`, `Fire◎Flower`). No structured song fields.
* **Languages:** Japanese-only.
* **Last-Updated Date:** Ongoing community wiki edits; exact page edit timestamp **Unverified**.
* **Verification Status:** **Verified** (`ヲタ芸` and `PPPH` exist; `打ち曲` confirmed absent).

---

### 3.4 X (Twitter) Lists & Public Community Curations / Spreadsheets

#### A. X (formerly Twitter) Hashtags, Lists & X API v2
* **Scope:** Community members share daily song picks, BPM notes, and event setlists under hashtags `#打ち曲`, `#今日の打ち曲`, `#打ち曲リスト`, and `#ヲタ芸`, as well as user-curated X Lists.
* **Free or Paid:**
  * **Manual browsing:** Free (logged-in account required for many search views).
  * **Programmatic API (X API v2, `https://docs.x.com`):** **Paid.** X discontinued its legacy fixed Free and Basic ($100–$200/mo) plans for new developers and transitioned to **Pay-Per-Use** pricing: **$0.005 per post read** for general search/reads ($0.001/owned read, 3M reads/month cap; legacy Free tier also lacked search/read access). Automated scraping violates X Terms of Service.
* **Access Method:** Manual browse only for a zero-budget class project.
* **Fields Available vs. `data/songs.csv`:** Unstructured free-text mentions of `title`, `artist`, and occasional user-noted `tempo_bpm` or move suitability. All structured/numeric fields are missing.
* **Languages:** Multilingual, overwhelmingly Japanese for `#打ち曲` / `#ヲタ芸`.
* **Last-Updated Date:** Real-time continuous posts (live social feed); no static dataset version.
* **Verification Status:** X API v2 Pay-Per-Use pricing **Verified** via search citations referencing `docs.x.com`; live hashtag post volume **Unverified**.

#### B. Public Community Blogs, Rankings, Playlists, Spreadsheets & GitHub
* **Identified Primary Community Sources:**
  * **Wotagei Times (`https://wotageitimes.com`):** Features move encyclopedia (`技辞典`) and community-voted survey rankings like 「好きな打ち曲ランキング！」 (published **2019-11-22**), plus articles discussing typical fast-wotagei BPM (**170–200 BPM**) and `4x8` (32-count) structure, listing canonical tracks such as `地球最後の告白を` (KEMU VOXX), `ローリンガール` (wowaka), `空奏列車` (Orangestar), `インフェルノ` (Mrs. GREEN APPLE), `ピースサイン` (Kenshi Yonezu), `シルエット` (KANA-BOON), `Crossing Field` (LiSA), and `ブラック☆ロックシューター` (ryo).
  * **`note.com` Anikura / Wotagei Curations:** DJ and performer essays (e.g., 「【アニクラ】覚えておくと楽しい打ち曲紹介①」) categorizing tracks into actionable BPM bands: `<=129 BPM` (slow/deliberate), `130–160 BPM` (sweet-spot anison/anthem wotagei), and `161+ BPM` (high-speed/intense).
  * **Competition Rules & Mixes (`https://wotagei.org/wwc`, YouTube, Spotify):** **Wotagei World Cup (WWC)** (`wotagei.org/wwc`, running through **WWC2026**) uses **Free Choice of Music (`自由曲制`)** for video qualifiers and organizer-selected random music for live battles. User-created YouTube mixes and Spotify playlists (`打ち曲`, `ヲタ芸`) gather popular tracks.
  * **Google Sheets & GitHub:** Ad-hoc DJ BPM Google Sheets circulate via X profiles, `note.com`, and YouTube DJ-mix descriptions, **but no single canonical, permanently maintained open-licensed Google Sheet CSV of wotagei songs was verified in public search indices**. GitHub repositories for wotagei (`himetuki/seu_wotagei_competition`, `himetuki/WotaGeiCam`, `randomList/confusedList.json`) are competition tools/etiquette lists rather than song-feature datasets.
* **Free or Paid:** Free.
* **Access Method:** Manual curation / copy-paste (HTML prose, YouTube descriptions, Spotify web playlists).
* **Fields Available vs. `data/songs.csv`:** `title`, `artist`, partial `tempo_bpm` (human-estimated BPM or BPM band), informal `genre` (Vocaloid, anison, J-rock, NCS). Missing `id`, `mood`, `energy`, `valence`, `danceability`, `acousticness`, `popularity`, `duration_ms`, `release_year`.
* **Languages:** Primarily Japanese for wikis/blogs; multilingual (including English NCS/EDM) on YouTube/WWC.
* **Last-Updated Date:** `wotageitimes.com` survey verified as **2019-11-22**; `wotagei.org/wwc` rules reflect **2026 (`WWC2026`)**; individual `note.com`/Spotify playlists vary (**Unverified** per-list dates).
* **Verification Status:** `wotageitimes.com`, `note.com`, `wotagei.org/wwc`, and GitHub repos **Verified** via search index; public Google Sheet URLs **Unverified**.

---

### 3.5 MusicBrainz
* **URLs:**
  * API Docs & Root: `https://musicbrainz.org/ws/2/`
  * Database / JSON Dumps: `https://musicbrainz.org/doc/MusicBrainz_Database/Download` and `https://musicbrainz.org/pub/musicbrainz/data/json-dumps/`
  * Genre / Tag Pages: `https://musicbrainz.org/genre/j-pop`, `https://musicbrainz.org/tag/vocaloid`, `https://musicbrainz.org/tag/anisong`, `https://musicbrainz.org/tag/wotagei`
* **Free or Paid:** **Free.** Core metadata (artists, recordings, releases, works, series, relationships) is **CC0 1.0 (Public Domain)**; supplementary community data (**Tags, Genres, Ratings, Annotations**) is **CC BY-NC-SA 3.0** (free for non-commercial academic use).
* **Access Method:**
  * **REST API (`/ws/2/recording`):** Supports lookup, browse, and Lucene search (`&fmt=json`). Strictly rate-limited to **1 request/second** per IP and requires a descriptive `User-Agent` header (requests without it receive `403`). Use `inc=tags+genres+artist-credits+ratings+series-rels`.
  * **Bulk Data Dumps:** Full PostgreSQL and JSON dumps updated **twice weekly (Wednesdays and Saturdays)**.
* **Fields Available vs. `data/songs.csv`:**
  * `id` -> Recording MBID (UUID)
  * `title` -> Recording `title`
  * `artist` -> `artist-credit`
  * `genre` -> `genres` (official list includes `j-pop`, `denpa`) and folksonomy `tags` (`vocaloid`, `anisong`, `anime song`, `moe song`)
  * `mood` -> Very sparse user tags only; no dedicated mood field
  * `popularity` -> `rating` (`inc=ratings`, 0–5 average + vote count — **sparse coverage** on Japanese catalog tracks)
  * `duration_ms` -> `length` (exact integer milliseconds)
  * `release_year` -> `first-release-date` (`YYYY-MM-DD` / `YYYY`)
  * **Missing:** `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness` (MusicBrainz does not store signal features; those previously lived in AcousticBrainz, which shut down in 2022).
  * **Wotagei & Series Caveat:** **`wotagei` / `ヲタ芸` / `打ち曲` is NOT an established genre or Series entity.** `Series` organizes album collections, tours, and media franchises—not cheering choreography lists. If `wotagei` appears in tags or notes, it is sparse (primarily on live idol concert videos).
* **Languages:** Multilingual (Japanese native scripts + romanized aliases + global Western/English music).
* **Last-Updated Date:** Live database edited continuously; JSON/DB dumps generated **twice weekly (Wednesdays & Saturdays)**.
* **Verification Status:** API, schema, licensing, and dump schedule **Verified** against `musicbrainz.org` docs; exact count of entities tagged `tag/wotagei` **Unverified** (blocked from direct live API fetch).

---

### 3.6 Anison & Vocaloid Databases

#### A. VocaDB (+ Sibling Engines UtaiteDB & TouhouDB)
* **URLs:**
  * Web & Swagger: `https://vocadb.net/`, `https://vocadb.net/swagger`, API `https://vocadb.net/api/songs`
  * Source Code / Contracts: `https://github.com/VocaDB/vocadb`
  * Sibling databases on same engine/API: **UtaiteDB** (`https://utaitedb.net/`) for *utattemita* / J-pop covers, and **TouhouDB** (`https://touhoudb.com/`) for Touhou Eurobeat / Denpa / Vocal arrangements.
* **Free or Paid:** **Free.** Open-source platform (`VocaDB/vocadb`); user-contributed metadata broadly follows Creative Commons terms (e.g., CC BY-SA noted on sibling TouhouDB; underlying audio/PVs remain copyrighted by rightsholders).
* **Access Method:**
  * **REST JSON API:** No API key required for read endpoints; custom `User-Agent` and polite request throttling required to avoid automated WAF/IP blocks. Supports pagination (`start`, `maxResults`, `getTotalCount=true`), sorting (`SongSortRule`: `PublishDate`, `FavoritedTimes`, `RatingScore`), and `fields=Tags,PVs,Artists,WebLinks`.
  * **Dumps:** No official, routine bulk CSV/SQL dump download endpoint on VocaDB's API/wiki (to limit server strain); community GitHub mirrors exist (e.g., `blueset/vocaloid-database-dump`, `hoshizora.sql`) with **unverified** freshness.
* **Fields Available vs. `data/songs.csv` (`SongForApiContract`):**
  * `id` -> `id` (integer)
  * `title` -> `name` / `defaultName` (multilingual)
  * `artist` -> `artistString` / `artists`
  * `genre` -> `tags` (rich subgenre tags: Vocaloid, EDM, rock, eurobeat, denpa, etc.) + `songType` (`Original`, `Cover`, `Remix`, `Remaster`)
  * `mood` -> Partial via descriptive user tags (where applied)
  * `popularity` -> `favoritedTimes` and `ratingScore`
  * `duration_ms` -> `lengthSeconds * 1000`
  * `release_year` -> `publishDate` (ISO 8601)
  * **Bonus Field:** `pvs` (array of linked Niconico `sm...`, YouTube, and SoundCloud PV IDs — ideal for cross-referencing Niconico `ヲタ芸` videos or fetching audio!)
  * **Missing:** `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`.
  * **Wotagei Tag Reality:** Targeted search and live API query `GET https://vocadb.net/api/tags?query=wotagei` (`200 OK`) both returned **`totalCount: 0`** (`items: []`), confirming Vocaloid wotagei favorites must be identified via external seed lists or Niconico tag cross-matching. Live `GET /api/songs` queries (`fields=PVs,Artists,Tags`) returned `200 OK` with full `pvs` (including `NicoNicoDouga` `sm...` and `Youtube` IDs), `lengthSeconds`, `publishDate`, `favoritedTimes`, and `ratingScore`.
* **Languages:** Multilingual (Japanese, Romaji, English).
* **Last-Updated Date:** Live continuously edited community database; exact community dump currency **Unverified**.
* **Verification Status:** **Verified (live HTTP `200 OK`)** on `/api/songs` (`地球最後の告白を` ID `13615`, `ブラック★ロックシューター` ID `1323`) and `/api/tags?query=wotagei` (`totalCount: 0`).

#### B. Anison Generation (`anison.info`)
* **URLs:** `http://anison.info/`, CSV download `http://anison.info/data/download.html`, README `http://anison.info/data/readme.php`.
* **Free or Paid:** Free for **personal/non-commercial use** (`download.html` explicitly prohibits commercial use and redistribution beyond citation: *「本データの商用上の利用はご遠慮ください」「本データの引用を超える転載はご遠慮ください」*).
* **Access Method:** Web search and 4 downloadable ZIP-compressed CSV archives on `download.html` (`download/program.zip`, `download/anison.zip`, `download/sf.zip`, `download/game.zip`). **No official first-party REST API**.
* **Fields Available vs. `data/songs.csv` (verified directly on `download.html`):**
  * `program.zip` (updated **5th of each month**, early morning): `番組ID,番組分類,ゲーム種別,番組名,バングミメイ,番組名補,バングミメイホ,放映話数,年齢制限,放映開始日` (provides `release_year` via `放映開始日`).
  * `anison.zip` (10th), `sf.zip` (15th), `game.zip` (20th): `番組ID,番組分類,番組名,摘要(OP/ED別),放映順,楽曲ID,楽曲名,歌手名` (note: explicitly states lyricist/composer/arranger credits are excluded from CSV exports).
  * Mapping: `id` -> `楽曲ID`/`番組ID`, `title` -> `楽曲名`, `artist` -> `歌手名`, `genre` -> `番組分類` & `摘要(OP/ED別)` only (not musical genre), `release_year` -> `放映開始日` (via join on `番組ID`).
  * **Missing:** `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`, `popularity`, `duration_ms`, and **no wotagei/打ち曲 tags**.
* **Languages:** Japanese-only.
* **Last-Updated Date:** `download.html` header `Last-Modified: Tue, 05 Sep 2023 20:46:06 GMT`; table states monthly CSV generation on the **5th / 10th / 15th / 20th** of each month (exact timestamp inside ZIP files **Unverified** without unpacking ZIPs).
* **Verification Status:** **Verified (live HTTP `200 OK`)** on `http://anison.info/data/download.html` (terms, column schemas, monthly schedule, and ZIP URLs confirmed); internal ZIP CSV character encoding **Unverified**.

#### C. AnimeThemes (`animethemes.moe`)
* **URLs:**
  * Site & Docs: `https://animethemes.moe/`, `https://docs.animethemes.moe/`, `https://github.com/AnimeThemes/animethemes-api-docs`, `https://github.com/AnimeThemes/animethemes-server`
  * Endpoints: `https://api.animethemes.moe/` (REST JSON:API) and `https://graphql.animethemes.moe/` (GraphQL)
* **Free or Paid:** Free, open-source.
* **Access Method:** Public unauthenticated REST JSON:API (`/anime`, `/song`, `/audio`, `/video`) and GraphQL API; rate limit commonly documented by clients at **~90 requests/minute** (returns `429 Too Many Requests` on burst). Server architecture (`animethemes-server`) also includes automated database dump infrastructure (`DUMP_DISK`). Provides direct CDN links to extracted `.ogg` Opus audio tracks (`a.animethemes.moe`).
* **Fields Available vs. `data/songs.csv`:**
  * `id` -> `song.id` / `animetheme.id`
  * `title` -> `song.title`
  * `artist` -> `artists.name` (via `performances`)
  * `genre` -> Only theme classification (`OP` / `ED` / `IN`), not musical genre
  * `release_year` -> `anime.year`, `anime.season`
  * `duration_ms` -> **Gap / Caveat:** AnimeThemes hosts **TV-size (~90s) OP/ED cuts**, not full-length songs. Moreover, inspected `Video`/`Audio` JSON schemas expose `basename`, `filename`, `path`, `size`, and `mimetype` (`audio/ogg`, `video/webm`), **without an explicit `duration` attribute confirmed**.
  * **Missing:** `mood`, `energy`, `tempo_bpm`, `valence`, `danceability`, `acousticness`, `popularity`, and **no wotagei tags**.
* **Languages:** Multilingual (Japanese/Romaji song and anime titles; English API/UI).
* **Last-Updated Date:** Actively updated with seasonal anime themes; exact latest commit/dump timestamp **Unverified**.
* **Verification Status:** REST/GraphQL endpoints and `Video`/`Audio` file attributes **Verified** via `animethemes.moe` and GitHub; live `GET https://api.animethemes.moe/song` probe returned **`HTTP 503 Service Temporarily Unavailable`** (Cloudflare/nginx challenge or temporary outage), leaving live rate-limit headers and `duration` field **Unverified**.

#### D. AniDB (`anidb.net`)
* **URLs:** `https://wiki.anidb.net/w/API`, `http://api.anidb.net:9001/httpapi`, bulk title dump `animetitles.xml.gz`.
* **Free or Paid:** Free for **non-commercial use**.
* **Access Method:** HTTP XML API (strict **1 request per >=2 seconds**, mandatory registered `client` & `clientver`, mandatory local daily caching to avoid IP bans), UDP API, and `animetitles.xml.gz` XML dump.
* **Fields Available vs. `data/songs.csv`:**
  * AniDB models **anime series, episodes, characters, and video files**, not standalone music tracks.
  * The HTTP XML `request=anime` schema returns `<anime id>`, `<type>`, `<episodecount>`, `<startdate>`, `<enddate>`, `<titles>`, `<episodes>`, and weighted anime-level `<tags>`, **without structured song BPM, duration, artist, or wotagei tags**.
  * **Verdict:** **Unsuitable** as a song source for `data/songs.csv`.
* **Languages:** Multilingual (`x-jat` romanized, `ja`, `en`).
* **Last-Updated Date:** Continuously updated anime catalog; exact date **Unverified**.
* **Verification Status:** HTTP API XML schema and rate limits **Verified** via `wiki.anidb.net` / client schemas.

---

## 4. English-Language Tracks in Wotagei & Oshi-katsu Song Lists

English-language tracks **do** appear in wotagei and oshi-katsu ecosystems, but **under sharply different circumstances** depending on whether the list serves **online cyalume dance choreography**, **anison/VTuber club culture**, **traditional live idol venues**, or **broader social-media oshi-katsu**.

### 4.1 Four Circumstances Where English-Language Tracks Appear

1. **YouTube / Online Cyalume Dance Videos Using Copyright-Safe EDM (NCS — NoCopyrightSounds):**
   * **Who & Where:** Major Japanese wotagei crews (such as 「ゼロから打ち師始めます。」, `GinyuforcE`, `North Accel`) and international competitors in `CYALUME DANCE WORLD BATTLE` / `Wotagei World Cup` (`wotagei.org/wwc`, which uses **Free Choice of Music / `自由曲制`** for video qualifiers) frequently choreograph routines to **NoCopyrightSounds (NCS)** tracks.
   * **Why English Tracks Appear:**
     1. **Copyright / Content ID Safety:** NCS grants royalty-free YouTube/Twitch usage with attribution in the video description, avoiding automated copyright strikes or audio mutes that hit commercial J-pop/anison uploads.
     2. **Musical Structure Match:** NCS genres (House, Drumstep, Dubstep, Drum & Bass) deliver crisp 4-beat phrasing and sharp **build-up -> drop** transitions that map cleanly onto wotagei technique sequences (`ツクヨミ`, `ロザリオ`, `サンダースネイク`, `アマテラス`).
   * **Verified Exemplars:** *Cartoon - On & On (feat. Daniel Levi)*, *Janji - Heroes Tonight (feat. Johnning)*, *Warriyo - Mortals (feat. Laura Brehm)*, *Lost Sky - Fearless pt. II (feat. Chris Linton)*, *Lost Sky - Dreams pt. II (feat. Sara Skinner)*, *Different Heaven & EH!DE - My Heart*, *Lensko - Let's Go!*, *Alan Walker - Fade / Spectre*, and tracks by *Elektronomia*, *Tobu*, and *Jim Yosef*.

2. **Mainstream Western EDM & High-Energy Electronic Pop ("Universal Drops"):**
   * **Why English Tracks Appear:** Tracks at **120–140+ BPM** with four-on-the-floor kicks and euphoric choruses mirror the structural *takamari* (pre-chorus tension -> chorus release) of anison, making them easy to count in `4x8` (32-beat) wotagei blocks and appealing to global audiences on YouTube/TikTok without a language barrier.
   * **Verified Exemplars:** **Avicii** (*Wake Me Up*, *The Nights*), **Zedd** (*Clarity feat. Foxes*, *Beautiful Now*), **Ariana Grande feat. Zedd** (*Break Free*), **The Chainsmokers** (*Closer feat. Halsey*), **5 Seconds of Summer** (*Teeth*), and **Porter Robinson** (*Language*, *Shelter* — heavily cross-pollinated with Japanese anime/club culture).

3. **English-Lyric Tracks Native to Japanese Anime, Game & VTuber Culture:**
   * **Why English Tracks Appear:** Even within domestic Japanese anikura (`アニクラ`) and wotagei scenes, English vocals are common whenever the track originates from a Japanese anime, rhythm game, or VTuber franchise.
   * **Verified Exemplars:** **Hiroyuki Sawano / SawanoHiroyuki[nZk]** (English-vocal anime OSTs/themes such as *Attack on Titan*, *Kill la Kill*, *Promare*), **MYTH & ROID** (aggressive English-mixed anison themes), **Mili** (*Deemo* / anime themes), and **Hololive English** (`hololive English -Myth-`, etc.) original EDM/pop tracks embraced in live penlight/call culture.

4. **Oshi-katsu (推し活) BGM Lists vs. Traditional Live-Venue Wotagei (Important Distinction):**
   * **Oshi-katsu (推し活) Song Lists:** On TikTok, Instagram Reels, YouTube vlogs, and Spotify "推し活 洋楽" playlists, **English Western pop appears frequently** — e.g., **Carly Rae Jepsen** (*Call Me Maybe*, *I Really Like You*), **Taylor Swift** (*Shake It Off*, *Cruel Summer*), **Ariana Grande** (*No Tears Left to Cry*), **One Direction** (*What Makes You Beautiful*), **Billie Eilish** (*BIRDS OF A FEATHER*), **The Neighbourhood** (*Sweater Weather*), and **Sabrina Carpenter**.
   * **Why & How It Differs from Wotagei:** Oshi-katsu BGM selection is driven by **lyrical devotion** (love songs projected onto one's *oshi* / `推しの概念`), **aesthetic mood** for fancam/merchandise Reels, or **pre-concert hype** — regardless of whether the tempo/beat supports `OAD`, `PPPH`, or `サンダースネイク`.
   * **Traditional Live Idol-Venue & Domestic Anikura `打ち曲`:** By contrast, traditional Japanese wotagei/call lists (`wikiwiki.jp/wotagei/`, `wotageitimes.com` survey, domestic anikura DJ lists) remain **overwhelmingly Japanese** (anison, Vocaloid, idol pop, J-rock), with English tracks appearing primarily through Channels #1–#3 above.

### 4.2 Design Implication for the Recommender
If the project dataset merges **Japanese wotagei/anison/Vocaloid tracks** with **English pop/EDM tracks**, a tag or metadata column (or genre/source indicator) should distinguish **performative wotagei compatibility** (high-BPM / strong build-up-to-chorus or EDM drop) from **general oshi-katsu pop** (mid-tempo lyrical pop like *Sweater Weather* or *Call Me Maybe*), so the recommender does not treat all English oshi-katsu tracks as traditional `打ち曲`.

---

## 5. Primary Source URLs Cited

* **WIKIWIKI Wotagei Pages:** `https://wikiwiki.jp/neosr/%E6%89%93%E3%81%A1%E6%9B%B2` (`200 OK`), `https://wikiwiki.jp/neosr/%E3%83%B2%E3%82%BF%E8%8A%B8%E3%81%AE%E5%9F%BA%E6%9C%AC%E7%9A%84%E3%81%AA%E6%A7%8B%E6%88%90` (`200 OK`), `https://wikiwiki.jp/otasugei/%E6%89%93%E3%81%A1%E6%9B%B2` (`429`), `https://wikiwiki.jp/wotagei/` (`404`)
* **Uta-Net:** `https://www.uta-net.com/`, `https://www.uta-net.com/info/kiyaku.html`
* **Niconico:** `https://site.nicovideo.jp/search-api-docs/snapshot`, `https://api.search.nicovideo.jp/api/v2/snapshot/video/contents/search`, `https://snapshot.search.nicovideo.jp/api/v2/snapshot/video/contents/search`, `https://snapshot.search.nicovideo.jp/api/v2/snapshot/version`, `https://dic.nicovideo.jp/a/%E3%83%B2%E3%82%BF%E8%8A%B8` (verified absent: `https://dic.nicovideo.jp/a/%E6%89%93%E3%81%A1%E6%9B%B2`)
* **X API:** `https://docs.x.com`
* **Community Blogs & Competitions:** `https://wotageitimes.com`, `https://wotagei.org/wwc`
* **MusicBrainz:** `https://musicbrainz.org/ws/2/`, `https://musicbrainz.org/doc/MusicBrainz_Database/Download`, `https://musicbrainz.org/pub/musicbrainz/data/json-dumps/`, `https://musicbrainz.org/genre/j-pop`, `https://musicbrainz.org/tag/vocaloid`, `https://musicbrainz.org/tag/anisong`, `https://musicbrainz.org/tag/wotagei`
* **VocaDB Ecosystem:** `https://vocadb.net/api/songs`, `https://vocadb.net/swagger`, `https://github.com/VocaDB/vocadb`, `https://utaitedb.net/`, `https://touhoudb.com/`
* **Anison Generation:** `http://anison.info/data/download.html`, `http://anison.info/data/readme.php`
* **AnimeThemes:** `https://animethemes.moe/`, `https://api.animethemes.moe/`, `https://graphql.animethemes.moe/`, `https://docs.animethemes.moe/`, `https://github.com/AnimeThemes/animethemes-api-docs`, `https://github.com/AnimeThemes/animethemes-server`
* **AniDB:** `https://wiki.anidb.net/w/API`, `http://api.anidb.net:9001/httpapi`
