---
name: minimax-music-playlist
description: >
  Generate personalized music playlists by analyzing the user's music taste from local
  apps (Apple Music, QQ Music, Spotify, NetEase) and generation feedback history. Triggers
  on any request involving playlist generation, music taste profiling, or personalized
  music recommendations. Supports multilingual triggers — match equivalent phrases in
  any language.
license: MIT
metadata:
  version: "2.0"
  category: creative
---

# MiniMax Music Playlist — Personalized Playlist Generator

Scan the user's local music libraries, build a taste profile, generate a personalized
playlist, and create an album cover. This skill is designed for both agent and direct
user invocation — adapt interaction style to context.

## Prerequisites

- **mmx CLI** — music & image generation. Install: `npm install -g mmx-cli`. Auth: `mmx auth login --api-key <key>`.
- **Python 3** — for scanning scripts you write on the fly (stdlib only, no pip).
- **Audio player** — `mpv`, `ffplay`, or `afplay` (macOS built-in).

## Language

Detect the user's language from their message. **All user-facing text must be
in the same language as the user's prompt** — do not mix languages. If the user
writes in Chinese, all output (profile summary, theme suggestions, playlist plan,
playback info) must be fully in Chinese. If in English, all in English.

All `mmx` generation prompts should be in English for best quality.
Each song's lyrics language follows its genre (K-pop → Korean, J-pop → Japanese, etc.),
NOT the user's UI language.

---

## Workflow

```
1. Scan local music apps → 2. Build taste profile → 3. Plan playlist
→ 4. Generate songs (mmx music) → 5. Generate cover (mmx image) → 6. Play → 7. Save & feedback
```

---

## Step 1: Scan Local Music Sources

Discover which music apps are installed and extract listening data. You should write
your own scanning scripts or commands — the approach below is guidance, not prescription.

**Known macOS music app data locations:**

| App | Where to look | Data format |
|-----|--------------|-------------|
| Apple Music | `osascript` to query Music.app | AppleScript returns track name, artist, album, genre, play count |
| QQ Music | `~/Library/Containers/com.tencent.QQMusicMac*/Data/` | SQLite DB + plist files |
| Spotify | `~/Library/Application Support/Spotify/` | LevelDB cache, URL cache, osascript for current track |
| NetEase | `~/Library/Containers/com.netease.163music/Data/Documents/storage/` | JSON files (webdata, tracks, FM queue), Cache.db |

**What to extract from each source:**
- Track names + artist names (primary signal)
- Playlist names and membership (e.g., a playlist named "Chinese Traditional" tells you genre preference)
- Play counts if available (weight frequently played tracks higher)
- Scene/mood tags if available (NetEase has these)
- Search history if accessible

**Approach:**
1. Check which apps are installed (`ls` the data paths)
2. For each installed app, figure out how to read its data (SQLite, JSON, plist, osascript)
3. Write a Python script using only stdlib to extract tracks and artists
4. If no apps found, ask the user to describe their taste manually

**Privacy rule:** Never show raw track lists to the user. Only show aggregated stats.

---

## Step 2: Build Taste Profile

From the scanned data, build a taste profile covering:

- **Genre distribution** — what styles the user listens to (e.g., J-pop 20%, R&B 15%, Classical 10%)
- **Mood tendencies** — emotional tone preferences (melancholic, energetic, calm, romantic, etc.)
- **Vocal preference** — male vs female voice ratio
- **Tempo preference** — slow / moderate / upbeat / fast distribution
- **Language distribution** — zh, en, ja, ko, etc.
- **Top artists** — most listened artists

**How to infer genre/mood from artist names:**
Most raw data only has artist + track names without genre tags. To enrich this:
- Use `mmx text chat --message "<prompt>" --quiet` to batch-query artist genres
- Keep batches small (10 artists per request) for reliable results
- Cache results to `<SKILL_DIR>/data/artist_cache.json` to avoid re-querying
- **Use concurrent requests** (e.g., ThreadPoolExecutor) to speed up lookups
- **Retry failed batches** — if a batch returns invalid JSON or errors, retry it
  once before skipping. Network/model glitches are common; a retry usually succeeds

**Profile caching:**
- Save profile to `<SKILL_DIR>/data/taste_profile.json`
- If a profile less than 7 days old exists, reuse it (offer rescan option)
- If older or missing, rebuild

**Show user a summary:**
```
Your Music Profile:
  Sources: QQ Music 145 | Apple Music 230 | Spotify 20 | NetEase 197
  Genres: J-pop 20% | R&B 15% | Classical 10% | Indie Pop 9%
  Moods: Melancholic 25% | Calm 20% | Romantic 18%
  Vocals: Female 65% | Male 35%
  Top artists: Faye Wong, Ryuichi Sakamoto, Taylor Swift, Jay Chou, Taeko Onuki
```

If invoked by an agent with clear parameters, skip the confirmation and proceed.
If invoked by a user directly, ask if the profile looks right before continuing.

---

## Step 3: Plan Playlist

**Ask the user for a theme/scene before generating.** This is the one
interactive step in the workflow. All other steps run autonomously.

If the theme was already provided in the invocation (e.g., the agent or user
said "generate a late night chill playlist"), use it directly and skip the question.
Otherwise, ask:

```
What theme would you like for your playlist? Here are some suggestions:

- "Late night chill" — relaxing slow songs
- "Commute" — upbeat and energizing
- "Rainy day" — melancholic & cozy
- "Surprise me" — random based on your taste

Or tell me your own vibe!
```

Once the user picks a theme, proceed automatically through generation, cover,
playback, and saving — no further confirmations needed.

Determine playlist parameters:
- **Theme/mood** — from user input, or default to top mood from profile
- **Song count** — from user input, or default to 5
- **Genre mix** — weighted by profile, with variety

**Per-song lyrics language** follows genre:

| Genre | Lyrics language |
|-------|----------------|
| K-pop, Korean R&B/ballad | Korean |
| J-pop, city pop, J-rock | Japanese |
| C-pop, Chinese-style, Mandopop | Chinese |
| Western pop/indie/rock/jazz/R&B | English |
| Latin pop, bossa nova | Spanish/Portuguese |
| Instrumental, lo-fi, ambient | No lyrics (`--instrumental`) |

Embed language naturally into the mmx prompt via vocal description:
- Good: `"A melancholy Chinese R&B ballad with a gentle introspective male voice, electric piano, bass, slow tempo"`
- Bad: `"R&B ballad, melancholy... sung in Chinese"`

**Show the playlist plan before generating.** Display each song with two lines:
the first line shows genre, mood, and vocal/language tag; the second line shows
the generation prompt description. Example:

```
Playlist Plan: Late Night Chill (5 songs)

1. Neo-soul R&B — introspective  English/male vocal
   A mellow neo-soul R&B ballad with warm baritone, electric piano, smooth bass

2. Lo-fi hip-hop — dreamy  Instrumental
   Dreamy lo-fi with sampled piano, vinyl crackle, soft electronic drums

3. Smooth jazz — romantic  English/female vocal
   Silky female voice, saxophone, piano, romantic starlit night

4. Indie folk — melancholic  English/male vocal
   Tender male voice, acoustic guitar, harmonica, quiet solitude

5. Ambient electronic — calm  Instrumental
   Soft synth pads, gentle arpeggios, dreamy atmosphere
```

After showing the plan, proceed directly to generation — no confirmation needed.
The user has already chosen the theme; the plan is shown for transparency, not approval.

---

## Step 4: Generate Songs

Use `mmx music generate` to create all songs. **Generate concurrently** (up to 5 in parallel).

```bash
# Example: 5 songs in parallel
mmx music generate --prompt "<english_prompt_1>" --lyrics-optimizer \
  --out ~/Music/minimax-gen/playlists/<name>/01_desc.mp3 --quiet --non-interactive &
mmx music generate --prompt "<english_prompt_2>" --instrumental \
  --out ~/Music/minimax-gen/playlists/<name>/02_desc.mp3 --quiet --non-interactive &
# ... more songs ...
wait
```

**Key flags:**
- `--lyrics-optimizer` — auto-generate lyrics from prompt (for vocal tracks)
- `--instrumental` — no vocals
- `--vocals "<description>"` — vocal style (e.g., "warm Chinese male baritone")
- `--genre`, `--mood`, `--tempo`, `--instruments` — fine-grained control
- `--quiet --non-interactive` — suppress interactive output for batch mode
- `--out <path>` — save to file

**File naming:** `<NN>_<short_desc>.mp3` (e.g., `01_rnb_midnight.mp3`)

**Output directory:** `~/Music/minimax-gen/playlists/<playlist_name>/`

If a song fails, **retry once** before skipping. Log the error and continue with the rest.

---

## Step 5: Generate Album Cover

Generate the album cover **concurrently with the songs** (Step 4), not after.
Launch the `mmx image generate` call in parallel with the song generation calls.

Craft a prompt that reflects the playlist's theme, mood, and genre mix. The image
should feel like an album cover — artistic, evocative, not literal.

```bash
mmx image generate \
  --prompt "<cover description based on playlist theme and mood>" \
  --aspect-ratio 1:1 \
  --out-dir ~/Music/minimax-gen/playlists/<playlist_name>/ \
  --out-prefix cover \
  --quiet
```

**Prompt guidance:**
- Abstract/artistic style works best for album covers
- Reference the dominant mood and genre (e.g., "dreamy late-night cityscape, neon reflections, lo-fi aesthetic")
- Do NOT include text or song titles in the image prompt
- Aspect ratio should be 1:1 (square, standard album cover)

---

## Step 6: Playback

Detect an available player and play the playlist in order:

| Player | Command | Controls |
|--------|---------|----------|
| mpv | `mpv --no-video <file>` | `q` skip, Space pause, arrows seek |
| ffplay | `ffplay -nodisp -autoexit <file>` | `q` skip |
| afplay | `afplay <file>` | Ctrl+C skip |

Play all `.mp3` files in the playlist directory in filename order.
Only play the songs generated in this session — if the directory has old files
from a previous run, clean them out first or filter by the known filenames.
If no player is found, just show the file paths.

---

## Step 7: Save & Feedback

Save playlist metadata to `<playlist_dir>/playlist.json`:
```json
{
  "name": "Late Night Chill",
  "theme": "late night chill",
  "created_at": "2026-04-11T22:00:00",
  "song_count": 5,
  "cover": "cover_001.png",
  "songs": [
    {"index": 1, "filename": "01_rnb_midnight.mp3", "prompt": "...", "rating": null}
  ]
}
```

If the user is present, ask for feedback (per-song or overall). Update the
taste profile's feedback section with liked/disliked genres and prompts to
improve future playlists.

---

## Replaying Playlists

If asked to play a previous playlist: `ls ~/Music/minimax-gen/playlists/`, show
available ones, and play the selected one.

---

## Notes

- **Agent vs user invocation**: The theme/scene question (Step 3) is the single
  interactive touchpoint. If the theme is already provided in the invocation,
  skip the question. Everything else runs autonomously.
- **No hardcoded scripts**: Write scanning/analysis scripts on the fly as needed.
  Use Python stdlib only. Cache results to avoid redundant work.
- **Skill directory**: `<SKILL_DIR>` = the directory containing this SKILL.md file.
  Data/cache files go in `<SKILL_DIR>/data/`.
- **All mmx prompts in English** for best generation quality.
