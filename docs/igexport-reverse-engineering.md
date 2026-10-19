# igexport.com: how the tools work

Reverse-engineering notes on nine igexport.com tools, written as groundwork for
this repo's own exporter. They cover how the site is built, the API its pages
call, how it gates requests, and how each tool turns an Instagram link or
username into downloadable files.

- **Snapshot:** 2026-10-19, Vercel deployment `dpl_6bvAxyX7xvY5swftXDsw4pV7xDmj`.
- **Pages studied:** `/en/video-download/` and these pages under `/id/`:
  `ig-story-download`, `instagram-highlights-downloader`, `reels-download`,
  `video-download`, `audio-download`, `threads-download`, `unfollow-checker`,
  `instagram-profile-viewer`.
- **Method:**
  1. Download each page and every JS chunk it references, then reformat them
     with prettier 3.3.3.
  2. Read the code that calls the API.
  3. Send one request to each `/api/*` route with the public `@instagram`
     account (17 requests in total).
- **Limits:**
  - The backend code is not visible. Everything about what the server does
    upstream is inferred from response fields and CDN URL parameters.
  - Instagram hosts were blocked from the analysis machine, so nothing was
    checked against Instagram directly.

**How each claim is known.** *code* means read in the shipped JS. *observed*
means seen in a live response. *inferred* means a conclusion not directly
confirmed.

**Coordinates.** References look like `chunk:line (module N)`. The chunk is a
file under `/_next/static/chunks/`, the line is in the prettier-formatted copy,
and the module is the Turbopack module ID. The same module appears in several
chunks with the same code at slightly different line offsets, for example
module `171085` in `aa066b08…`, `c9b66458…`, `b7d4cab1…` and `34d8a848…`. Line
numbers here refer to `aa066b08…`.

---

## 1. Summary

1. **Stack.** igexport is a Next.js App Router site built with Turbopack,
   hosted on Vercel behind Cloudflare. The browser never calls Instagram's API
   itself. Every lookup goes to igexport's own `/api/*` routes, and the server
   returns simplified JSON with direct Instagram CDN links.
2. **Request gating.** Two groups of routes behave differently:
   - **Open:** `ig-posts`, `ig-reels`, `ig-photo` and `threads-media`. The page
     calls them with plain `fetch`, and they answered a bare curl client with
     200.
   - **Gated:** `ig-profile`, `ig-stories`, `ig-highlight*`, `ig-avatar`,
     `ig-audio-page`, `ig-search`, `ig-reposts`, `threads-profile` and
     `recent-follow/preview`. The page calls them through `clearedFetch`, a
     Cloudflare Turnstile wrapper. Without clearance, Cloudflare answers with a
     403 block page.
3. **Media.**
   - Videos and full-size images are direct links to `*.cdninstagram.com`.
   - Thumbnails go through a third-party image proxy, `d.rapidcdn.app/thumb`.
     It takes an HS256 JWT holding the CDN URL and a `TelegramBot (like
     TwitterBot)` user-agent to fetch it with.
4. **Client-side tools.**
   - **Audio → MP3:** the browser downloads the reel MP4 from the CDN, decodes
     it with `OfflineAudioContext`, and encodes 128 kbps MP3 with lamejs in a
     Web Worker. No server transcodes anything.
   - **Unfollow checker:** parses Instagram's own "Download your information"
     export (ZIP or JSON) in the browser. It makes no network calls.
5. **Upstream (inferred).** The `source` numbers in responses and a `source=3`
   ("free only") switch point to several upstream data sources. Posts, reels
   and photos come back with CDN URLs tagged `urlgen_source: "www"`, which
   points to anonymous scraping of instagram.com. Stories and highlights are
   raw Instagram objects with private-API field names, wrapped in a
   `{data: {items, additional_data}}` envelope.

---

## 2. Architecture

| Layer | Evidence | Known via |
| --- | --- | --- |
| Next.js App Router, Turbopack chunks (`globalThis.TURBOPACK.push`) | `turbopack-415a22705560ec8a.js`; `self.__next_f` RSC payload in every page | code |
| Vercel origin | `x-vercel-id` and `x-vercel-cache` on `ig-posts`, `ig-reels`, `ig-photo` responses | observed |
| Cloudflare in front of everything | `server: cloudflare` and `cf-ray` on every response; `/` returns 307 to `/en/` | observed |
| Some heavy I/O routes run off Vercel, on Cloudflare Workers | Comment in the shipped worker source `/_next/static/media/mp3-encoder.worker.43c53976.ts`: *"the I/O-heavy routes were moved off Vercel for cost reasons (see `wrangler.jsonc`)"*. `threads-media` returned a Cloudflare `error code: 502` with no `x-vercel-id`. | code + observed; which routes moved is inferred |
| Supabase (auth and tables) for story notifications | Story page RSC props hold `notifyConfig: {url: "https://<ref>.supabase.co", key: "sb_publishable_…"}`; the client calls `/auth/v1/*` and `/rest/v1/*` | code |
| Ads and analytics | AdSense `ca-pub-5931171682448636`; `trackEvent` (module `375679`) calls `window.gtag("event", …)` on every result | code |
| Mobile-app upsell | Module `35169`: Play `com.recentapp`, App Store `id6744271488` ("Recent Follow") | code |

---

## 3. Request gating: Cloudflare Turnstile pre-clearance

`clearedFetch` is defined in `0f3a8f67c17eb8bc.js:394` (module `696734`). Its
steps (*code*):

1. On first idle (`requestIdleCallback`, 3 s timeout), it loads
   `https://challenges.cloudflare.com/turnstile/v0/api.js?render=explicit` and
   renders a widget:
   - `sitekey` `0x4AAAAAAE_uYqg1CvyYGOGF`
   - `appearance: "interaction-only"`
   - `action: "api"`

   The widget sits in a fixed `div[data-api-clearance]` and only becomes
   visible as a full-screen overlay if Cloudflare asks for interaction.
2. A successful solve is cached for 25 minutes (`15e5` ms). No token is added
   to API requests. Clearance is carried by Cloudflare's own cookie
   (Turnstile pre-clearance). *(The cookie mechanism is inferred from the
   absence of any token header.)*
3. A request first waits for clearance: up to 6 s, or 90 s if the user is
   being asked to interact. Then it calls `fetch`.
4. If the response is `403` with `cf-mitigated: challenge`, it forces a fresh
   solve (20 s budget) and retries once.
5. If it is still challenged, it creates a fake
   `503 {"error":"rate_limited","code":"rate_limited"}` response and sends the
   `api_clearance_fail` analytics event. This is why the UI reports "rate
   limited" when the real cause is a failed Turnstile check.

**Observed:** curl with no clearance got `403` and Cloudflare's "Attention
Required!" HTML on every route that goes through `clearedFetch`. No bypass was
attempted.

---

## 4. API contract

All routes are `GET` with query strings unless noted. "Status → reason" is how
the client maps HTTP status codes to its UI error states.

| Route | Gate | Params | Response (fields the client reads) | Status → reason | Known via |
| --- | --- | --- | --- | --- | --- |
| `/api/ig-posts/` | open; client aborts after 27 s | `username` | `{ok, source, via, username, posts:[{shortcode, kind:"video"\|"image"\|"carousel", thumbnailUrl, items:[{type, url, thumbnailUrl, filename}]}]}`. Returned 12 posts. | not ok → `null` | observed `aa066b08:913` |
| `/api/ig-reels/` | open | `url` (post URL), `videoOnly=1` | `{ok, source, media:{videoUrl, thumbnailUrl, filename}}` | 404/502 not_found; 429/503 rate_limited | observed `aa066b08:508` |
| `/api/ig-photo/` | open | `url` | `{ok, source, media:{shortcode, items:[{type:"image", url, thumbnailUrl:null, filename}]}}`. A carousel returned 3 items. | same as `ig-reels` | observed `aa066b08:565` |
| `/api/threads-media/` | open | `url` (threads.com or threads.net post) | `{ok, media:{items:[…]}}` | 429/503 rate_limited; 404 private_post | code `aa066b08:294`; a fake URL got a Cloudflare 502 |
| `/api/ig-profile/` | Turnstile | `username`, optional `source=3` ("freeOnly") | `{data:{pk, full_name, is_private, follower_count, following_count, hd_profile_pic_versions[], hd_profile_pic_url_info.url, profile_pic_url, profile_pic_url_hd, profile_pic_url_original}}` | 404 not_found; else unavailable | code `aa066b08:32` |
| `/api/ig-stories/` | Turnstile | `username` | `{data:{items:[story], additional_data:{user}, user}}` (story fields below) | 404/502 not_found; else rate_limited | code `aa066b08:955` |
| `/api/ig-highlight/` | Turnstile (not probed) | `highlight_id=highlight:<digits>` | `{data:{items:[story], additional_data:{title, user}}}` | 404/502 not_found | code `aa066b08:1251` |
| `/api/ig-highlight-tray/` | Turnstile | `username` | `{items:[{id, title, coverUrl}]}` | not ok → `null`, shown as rate_limited | code `aa066b08:116`, `881e6191:935-975` |
| `/api/ig-avatar/` | Turnstile | `username` | `{ok, profile:{…}}` or `{ok:false, error:"private_account"}` | 429/503 rate_limited | code `aa066b08:655` |
| `/api/ig-audio-page/` | Turnstile | `id` (10–20 digit audio ID) | `{track:{url, cover, filename, kind, title, artist, durationMs, spotifyUrl, full, reelCode, reelOwner}}` | 400/404 not_found; else rate_limited | code `aa066b08:193` |
| `/api/ig-search/` | Turnstile | `q`, optional `kind=typo` | `{users:[…]}` | not ok → `null` | code `aa066b08:1478` |
| `/api/ig-reposts/` | Turnstile | `username`, `page` | `{items:[…], more}` | 404 not_found; 403 private; 400 invalid_input; 410 expired | code `aa066b08:1375`; not called by these 9 pages |
| `/api/threads-profile/` | Turnstile | `username`, `page` | `{profile, posts:[…], more}` | same as `ig-reposts` | code `aa066b08:1425`; not called by these 9 pages |
| `/api/recent-follow/preview/` | Turnstile | `u`, optional `from` | `{ok, …}` or `{error}` | `http_<status>` | code `1d7dfa14:1288` |
| `/api/q-log/` | plain `fetch`, `POST` JSON (not probed) | body `{page, reason, input_kind, value, w:5}` | — | — | code `0f3a8f67:676` |
| `/api/notify-story/code-hint/` | plain `fetch`, `POST` JSON (not probed) | `{email, locale}` | — | — | code `f0a0af5d:2488` |
| `/api/notify-story/welcome/` | plain `fetch`, `POST` JSON with `Authorization: Bearer <supabase access token>` (not probed) | — | — | — | code `f0a0af5d:2879` |

**Story item → UI item.** The function `b()` at `aa066b08:1147` maps each story
item as follows (*code*):

- `isVideo` = `media_type == 2` or a non-empty `video_versions`.
- `url` = `video_versions[0].url` for videos, otherwise
  `image_versions.items[0].url`.
- `thumbnailUrl` = `thumbnail_url`, falling back to `url`.
- `timestamp` = `taken_at`.
- `taggedUsers` = the users in `reel_mentions[].user`, deduplicated by
  `id`/`pk`.

Items are sorted newest first.

---

## 5. Input classification

`classifyInput` lives at `0f3a8f67:23-204` (module `280999`). Zero-width
characters are stripped first, then the input is tested in this order:

| Kind | Rule |
| --- | --- |
| `threads_url` | `threads.(com\|net)/(@user/post\|t\|share)/<id>` |
| `threads_profile_url` | `threads.(com\|net)/@user` |
| `highlight_url` | `instagram.com/stories/highlights/<digits>`, giving `highlight:<digits>`. Or `instagram.com/s/<b64>`, where the base64url segment decodes to `highlight:<digits>`. |
| `story_url` | `instagram.com/stories/<username>` |
| `audio_page_url` | `/reel(s)/audio/<10–20 digits>` |
| `post_url` | `instagram.com/[user/](reel\|reels\|p\|tv)/<code>` |
| `share_url` | `instagram.com/share/<code>`, excluding `share/p\|reel\|reels\|tv` |
| `profile_url` | `instagram.com/<name>`, unless `<name>` is a reserved path such as `p`, `reel`, `stories`, `explore` or `accounts` |
| `at_username` | `@name` (1–30 characters) |
| `bare_code` | 5–20 characters matching `[A-Za-z0-9_-]` |

Two helpers sit beside the classifier:

- **`isPrivatePostUrl`** treats a shortcode of 28 or more characters as a
  private-account post and refuses it before any network call.
- **`normalizeReelUrl`** rewrites `reels/` to `reel/`. A bare code becomes
  `https://www.instagram.com/reel/<code>/`.

**Resolver.** `resolveIgInput(text, policy)` at `aa066b08:118` (module
`171085`) dispatches on the kind. Each page passes a **policy**:

- `bareMeans`: `"shortcode"` or `"username"`
- `accountMeans`: `"stories"`, `"profile"` or `"avatar"`
- `mediaPrefers`: `"video"` or `"photo"`
- `accountPosts`: `"video"` or `"all"`
- `audioPage`: `"resolve"`

**Posts.** Post and share URLs try `ig-reels` and `ig-photo` in the order
`mediaPrefers` sets. **Bare input.** A bare string tries the policy's
interpretation first and falls back to the other one. An 11-character code
containing an uppercase letter is treated as a shortcode even when the policy
prefers usernames.

---

## 6. The nine tools

| Page | Policy (source) | Flow |
| --- | --- | --- |
| `/en/video-download/`, `/id/video-download/` | `{bareMeans:"shortcode", accountMeans:"stories", mediaPrefers:"video"}` (`a376208d:748`) | Post URL → `ig-reels`, falling back to `ig-photo`. Highlight URL → `ig-highlight`. Profile or `@user` → `resolveIgAccount` (stories). An audio-page URL is handed off to `/{lang}/audio-download/?url=…` (module `999306`). |
| `/id/reels-download/` | `{…, mediaPrefers:"video", accountPosts:"video"}` (RSC props in the HTML) | Same as above. For a username, it also calls `ig-posts` in parallel and keeps only video posts. |
| `/id/ig-story-download/` | `{bareMeans:"username", accountMeans:"stories", mediaPrefers:"video"}`; free-text input uses `bareMeans:"shortcode"` (`f0a0af5d:2972-2976`) | `ig-stories`. If there are no stories and the response lacks enough profile data, it falls back to `ig-profile` to tell "private" apart from "no stories". It adds `source=3` when the stories response already marked the account private. Every page whose policy has `accountMeans:"stories"` shares this logic (`aa066b08:942-1146`). Also uses `ig-highlight-tray`, `ig-search` (name and typo search) and `recent-follow/preview`, and offers story-notification sign-up (section 8). |
| `/id/instagram-highlights-downloader/` | none (custom component, `881e6191:725-790`) | Highlight URL → `ig-highlight`. Username → `ig-highlight-tray`, then the user picks an item → `ig-highlight`. |
| `/id/audio-download/` | `{bareMeans:"shortcode", accountMeans:"stories", mediaPrefers:"video", audioPage:"resolve"}` (`29a7cba7:1382`) | Reel → `ig-reels` → MP4 → MP3 in the browser (section 7). Audio-page URL → `ig-audio-page` (gives the track's source reel and a Spotify link). Takes a `?url=` parameter and submits it automatically. |
| `/id/threads-download/` | `{bareMeans:"shortcode", accountMeans:"stories", mediaPrefers:"video"}` (RSC) | Threads URL → `threads-media`. Instagram input behaves as on the video page. |
| `/id/instagram-profile-viewer/` | `{bareMeans:"username", accountMeans:"profile", mediaPrefers:"photo"}` (`410418b1:898`) | Calls `ig-profile` and `ig-posts` in parallel. Private or not found → stop. Otherwise shows the profile header and the post grid. |
| `/id/unfollow-checker/` | client-only (`07e5197c:205-600`) | Upload 1–2 Instagram data exports. Details below. |

**Unfollow checker.** The page accepts `.zip` (opened with JSZip, loaded
lazily) or `.json` (*code*).

- **Files read:** `followers(_N).json` and `following.json`. It reads each
  entry's `string_list_data[0].{value, href, timestamp}`, falling back to
  `title` or the username in the `href`. The function `E()` also handles the
  newer `{relationships_following: […]}` wrapper by taking the first array
  value.
- **Rejected inputs:** HTML-format exports and identical archives get their
  own error messages.
- **One export:** shows `following − followers`, the accounts that don't
  follow back.
- **Two exports:** sorts them by the latest timestamp and shows followers lost
  and gained.
- **Network:** nothing is uploaded. The only network calls are analytics
  events.

---

## 7. Media delivery

- **Direct CDN links (observed).** Videos (`ig-reels`, `ig-posts`) and
  full-size images (`ig-photo`) are signed `scontent-*.cdninstagram.com` URLs.
  - Video `efg` decodes to
    `{"vencode_tag":"xpv_progressive.INSTAGRAM.CLIPS.C3.720.dash_baseline_1_v1", …, "urlgen_source":"www"}`.
    These are 720p progressive MP4s, not the highest-quality DASH renditions.
  - Photo `efg` decodes to
    `{"vencode_tag":"CAROUSEL_ITEM.xpids.1080.sdr.regular_photo.C3"}`.
- **Thumbnail proxy (observed).** `thumbnailUrl` in `ig-posts` is
  `https://d.rapidcdn.app/thumb?token=<JWT>`. The JWT header is
  `{"typ":"JWT","alg":"HS256"}` and its payload is:

  ```json
  {"url": "https://scontent-…cdninstagram.com/…jpg?…",
   "headers": {"user-agent": "TelegramBot (like TwitterBot)"},
   "iat": 1792432038}
  ```

  The signing key lives on the server, so this proxy can't be reused without
  it.
- **Downloading.** `saveFile` (`aa066b08:1625`) fetches the URL, turns it into
  a Blob and clicks an `<a download>` with the name `igexport-<code>.mp4/.jpg`.
  If `fetch` fails (CORS), it falls back to `window.open`.
- **MP3 conversion (code).** Two pieces do the work:
  - The page (`29a7cba7:1020-1185`) streams the MP4 with a progress bar and
    calls `new OfflineAudioContext(2, 1, 44100).decodeAudioData(...)`.
  - The worker (`/_next/static/media/mp3-encoder.worker.43c53976.ts`,
    shipped as TypeScript source with comments) runs
    `new Mp3Encoder(channels, sampleRate, 128)` from `@breezystack/lamejs` in
    blocks of 1152×40 samples.

  The worker's comment says Instagram's CDN sends
  `access-control-allow-origin: *`, which is what makes this possible.

---

## 8. Other client behaviour

- **Failed-query logging.** `logFailedQuery` (`0f3a8f67:657`) sends the raw
  text the user typed to `/api/q-log/`. It samples about 1 in 5 failures and
  skips `rate_limited`, `wrong_tool` and `audio_page_rate_limited`.
- **Story notifications** (`f0a0af5d`, story page only):
  1. Sign-in is an email OTP through Supabase GoTrue (`/auth/v1/signup`,
     `/otp`, `/verify`, `/token?grant_type=refresh_token`), preceded by
     `/api/notify-story/code-hint/`.
  2. Watched accounts are rows in `notify_targets`:
     `{user_id, username, kind:"story"}`.
  3. Preferences are rows in `notify_prefs`:
     `{user_id, status:"active", locale, app:"web"}`.
  4. A heartbeat calls `rpc/notify_touch_active`.
  5. `/api/notify-story/welcome/` is called with the Supabase bearer token.

  Sending the alerts happens on the server and isn't visible.
- **Rate-limit circuit breaker.** After two `ig-profile` failures in a row, the
  client stops calling it for 5 minutes (`aa066b08:13-25`).

---

## 9. What the backend probably calls upstream (inferred)

None of this is verified, because the server code isn't visible and Instagram
was unreachable from the analysis machine.

- **Several providers behind each route.** Every response carries
  `source: <n>` (2 for `ig-posts`, `ig-reels` and `ig-photo`), and
  `ig-profile` accepts `source=3` to mean "free only".
- **Posts, reels and photos come from scraping instagram.com.** `ig-posts`
  says `"via":"direct"`, and the CDN URLs carry `urlgen_source:"www"`. That
  fits anonymous scraping of instagram.com's web surface: the GraphQL post and
  timeline queries, or `?__a=1`-style JSON. The `TelegramBot` user-agent in the
  thumbnail JWT suggests they rely on Instagram serving link-preview bots
  without a login.
- **Stories, highlights and profiles look like a third-party scraping API.**
  These responses pass through Instagram private-API objects (`video_versions`,
  `taken_at`, `reel_mentions`, `hd_profile_pic_versions`,
  `hd_profile_pic_url_info`). However, images arrive as `image_versions.items`
  rather than Instagram's own `image_versions2.candidates`, inside a
  `{data:{items, additional_data}}` envelope. That shape matches commercial
  Instagram scraper APIs rather than Instagram's raw responses. Highlight IDs
  use the `highlight:<id>` form those APIs accept. Which provider it is was not
  confirmed.

---

## 10. Not verified / open questions

- **Gated routes.** Their response bodies come only from the fields the client
  code reads, because every one returned a Cloudflare 403. Full payloads would
  need a real browser session; no bypass was attempted.
- **`threads-media`** returned a Cloudflare 502 for a made-up URL. A real
  Threads post was not tested.
- **`fetchReposts` and `fetchThreadsProfile`** exist but aren't called by
  these 9 pages. They're likely used elsewhere on the site.
- **Instagram CDN behaviour** was not checked from here: the CORS header, how
  long signed URLs last (`oe`), and the user-agent rules. All
  `*.instagram.com`, `*.cdninstagram.com` and `threads.*` hosts were blocked
  by the analysis machine's network policy.

## Reproducing

```bash
UA='Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36'
curl -sSL -A "$UA" https://igexport.com/id/reels-download/ -o page.html
grep -oE '_next/static/chunks/[A-Za-z0-9_.-]+\.js' page.html | sort -u   # then fetch + prettier each
curl -sS -A "$UA" 'https://igexport.com/api/ig-posts/?username=instagram'
curl -sS -A "$UA" 'https://igexport.com/api/ig-reels/?url=https%3A%2F%2Fwww.instagram.com%2Freel%2F<code>%2F&videoOnly=1'
curl -sS -A "$UA" 'https://igexport.com/_next/static/media/mp3-encoder.worker.43c53976.ts'
```

Chunk hashes change with each deployment. Find the module by its export names
(`classifyInput`, `clearedFetch`, `resolveIgInput`) rather than by line
number.
