# Spotify Clone

A pixel-faithful Spotify UI clone built with React 19, Vite, Tailwind CSS 4, and React Router DOM v7 — featuring a fully functional music player with real audio playback, a clickable seek bar, previous/next track navigation, live MM:SS timestamp display, per-album dynamic background color gradients, and client-side routing between the home view and individual album pages. All music and album data is bundled locally — no external music API required.

---

## What This Project Does

The app replicates Spotify's three-panel layout: a left sidebar for navigation, a central display area for browsing, and a persistent bottom player bar. Users can browse 6 album cards and 8 song cards on the home screen. Clicking any song card immediately starts audio playback. Clicking any album card navigates to that album's detail page, which lists all 8 songs in a numbered track table — clicking any row starts that song. The bottom player shows the current track image and name, play/pause toggle, previous/next buttons, a live seek bar, and real-time current/total timestamps — all wired to a single `<audio>` element managed through React Context.

---

## Architecture — How the App Is Structured

The layout in `App.jsx` is a fixed 3-row structure:

```jsx
<div className='h-screen bg-black'>
  <div className='h-[90%] flex'>      {/* Top 90% — Sidebar + Display */}
    <Sidebar />
    <Display />
  </div>
  <Player />                           {/* Bottom 10% — persistent player bar */}
  <audio ref={audioRef} src={track.file} preload='auto' />
</div>
```

A single `<audio>` element lives in `App.jsx`. Its `ref` (`audioRef`) is stored in `PlayerContext` and shared across all components — `Player.jsx` controls it via `audioRef.current.play()` and `audioRef.current.pause()`, and `PlayerContext` reads its `currentTime` and `duration` properties to drive the seek bar and timestamps.

---

## Data — `src/assets/assets.js`

All music and album data is defined as two exported JavaScript arrays — no database, no API.

**`albumsData` — 6 albums:**

| id | Name | Background Color |
|---|---|---|
| 0 | Top 50 Global | `#2a4365` (dark blue) |
| 1 | Top 50 India | `#22543d` (dark green) |
| 2 | Trending India | `#742a2a` (dark red) |
| 3 | Trending Global | `#44337a` (dark purple) |
| 4 | Mega Hits | `#234e52` (dark teal) |
| 5 | Happy Favorites | `#744210` (dark orange) |

Each album object contains `id`, `name`, `image`, `desc`, and `bgColor`. The `bgColor` is used in `Display.jsx` to set the album page's gradient background.

**`songsData` — 8 songs:**
Each song object contains `id`, `name`, `image`, `file` (one of 3 bundled `.mp3` files), `desc`, and `duration` (a display string like `"3:00"`). Songs 0, 3, 6 use `song1.mp3`; songs 1, 4, 7 use `song2.mp3`; songs 2, 5 use `song3.mp3` — 3 actual audio files serve all 8 song entries.

---

## PlayerContext — `src/context/PlayerContext.jsx`

All audio playback state and control functions are managed here and shared globally via React Context.

**Refs:**

| Ref | Points to | Purpose |
|---|---|---|
| `audioRef` | `<audio>` in `App.jsx` | Direct DOM control — `.play()`, `.pause()`, `.currentTime` |
| `seekBg` | Seek bar background `div` in `Player.jsx` | Used to get `offsetWidth` for click-position calculation |
| `seekBar` | Seek bar fill `<hr>` in `Player.jsx` | Width is set directly as a percentage to reflect playback progress |

**State:**

| State | Initial value | Purpose |
|---|---|---|
| `track` | `songsData[0]` | The currently loaded song object |
| `playStatus` | `false` | Whether audio is currently playing |
| `time` | `{ currentTime: {second:0, minute:0}, totalTime: {second:0, minute:0} }` | MM:SS values displayed in the player |

**Functions:**

**`play()` / `pause()`**
Directly call `audioRef.current.play()` or `audioRef.current.pause()` and update `playStatus`.

**`PlayWithId(id)`**
```js
await setTrack(songsData[id]);
await audioRef.current.play();
setPlayStatus(true);
```
Sets the track to the song at the given index, immediately plays it, and marks status as playing. Called from both `SongItem` (home screen cards) and `DisplayAlbum` (track table rows).

**`previous()` / `next()`**
Check that `track.id > 0` (or `< songsData.length - 1`) before decrementing/incrementing, preventing out-of-bounds navigation. Then set the new track and play immediately.

**`seekSong(e)`**
```js
audioRef.current.currentTime =
  (e.nativeEvent.offsetX / seekBg.current.offsetWidth) * audioRef.current.duration;
```
Calculates the click position as a fraction of the seek bar's total width, multiplies by the song's total duration in seconds, and sets `currentTime` directly on the audio element — jumping playback to that position.

**`ontimeupdate` handler (inside `useEffect`):**
```js
audioRef.current.ontimeupdate = () => {
  seekBar.current.style.width =
    Math.floor(audioRef.current.currentTime / audioRef.current.duration * 100) + "%";
  setTime({
    currentTime: {
      second: Math.floor(audioRef.current.currentTime % 60),
      minute: Math.floor(audioRef.current.currentTime / 60)
    },
    totalTime: {
      second: Math.floor(audioRef.current.duration % 60),
      minute: Math.floor(audioRef.current.duration / 60)
    }
  });
};
```
This fires continuously as audio plays. The seek bar fill width is set as a direct DOM style mutation (not React state) for performance — avoiding a re-render on every audio tick. The `time` state is updated with `Math.floor` values to display clean integer seconds and minutes.

---

## Components

### `Sidebar.jsx`
Left panel — `w-[25%]` of the full height. Contains two `bg-[#121212]` dark boxes:
- Top box: Home icon (`navigate('/')`) and Search icon — navigation links
- Bottom box: "Your Library" header with stack and plus icons, a "Create your first playlist" CTA card, and a "Find podcasts" CTA card — both with white pill-shaped buttons. Hidden entirely on screens below `lg` breakpoint (`hidden lg:flex`).

### `Display.jsx`
Center panel — `w-[75%]` on large screens, `w-[100%]` on small. Contains a `<Routes>` block with 2 routes:
- `/` → `<DisplayHome />`
- `/album/:id` → `<DisplayAlbum />`

On every render, `Display.jsx` reads `location.pathname` to check if the current route contains `"album"`. If yes, it extracts the album ID from the last character of the path (`pathname.slice(-1)`), looks up that album's `bgColor` from `albumsData`, and applies a CSS gradient as an inline style:
```js
displayRef.current.style.background = `linear-gradient(${bgColor}, #121212)`;
```
This gives each album page its own unique color gradient fading to Spotify's dark background. The home page always gets a flat `#121212`.

### `DisplayHome.jsx`
Renders inside the `/` route. Shows two horizontally scrollable rows:
- "Featured Charts" — maps `albumsData` into `<AlbumItem>` cards
- "Today's biggest hits" — maps `songsData` into `<SongItem>` cards

### `AlbumItem.jsx`
A card component with `min-w-[180px]`, album image, name, and description. On click, calls `navigate('/album/${id}')` — routing to that album's detail page.

### `SongItem.jsx`
A card component identical in structure to `AlbumItem`. On click, calls `PlayWithId(id)` from `PlayerContext` — immediately starts playback without navigating.

### `DisplayAlbum.jsx`
Renders inside the `/album/:id` route. Uses `useParams()` to get the album ID, then looks up `albumsData[id]`. Displays the album image, name (large `text-7xl` on desktop), description, and a fake stats line ("1,23,235 likes || 50 songs, about 2 hr 30 min"). Below that, a 4-column grid (`grid-cols-3 sm:grid-cols-4`) lists all `songsData` entries as numbered rows. Each row has: track number, thumbnail + song name, album name, date added ("5 days ago"), and duration. Clicking any row calls `PlayWithId(item.id)`.

### `Navber.jsx`
An in-display top bar (not a fixed page navbar — it renders inside `DisplayHome` and `DisplayAlbum`). Contains left/right back-forward navigation arrows that call `navigate(-1)` and `navigate(1)`. On the right: "Explore Premium" pill (hidden on mobile), "Install App" pill, and a purple circle avatar with the letter "P".

### `Player.jsx`
The persistent bottom bar — `h-[10%]` of the screen, `bg-black`. Three sections:

**Left section** (hidden below `lg`): Current track thumbnail (`w-12`) and name + first 12 characters of description.

**Center section:** 5 playback control icons (shuffle, prev, play/pause toggle, next, loop) and the seek bar below them. The seek bar background (`seekBg ref`) is `w-[60vw] max-w-[500px] bg-gray-300 rounded-full`. The seek bar fill (`seekBar ref`) is a `<hr>` styled as `h-1 border-none w-0 bg-green-800 rounded-full` — its width is updated directly via the `ontimeupdate` handler. Timestamps (`time.currentTime.minute:time.currentTime.second`) flank both sides.

**Right section** (hidden below `lg`): 8 control icons — plays, mic, queue, speaker, volume, a white slider div, mini-player, and zoom. These are display-only — present in the UI but not wired to any audio functionality.

---

## Tech Stack

| Technology | Version | Role |
|---|---|---|
| React | 19.0.0 | Component UI, `useContext`, `useRef`, `useState`, `useEffect` |
| Vite | 6.2.0 | Dev server, HMR, production bundler |
| Tailwind CSS | 4.1.3 | All layout and styling — no separate CSS files |
| React Router DOM | 7.6.0 | Client-side routing — `/` and `/album/:id` routes, `useNavigate`, `useParams`, `useLocation` |

---

## Project Structure

```
Spotifyclone/
├── src/
│   ├── components/
│   │   ├── Sidebar.jsx         # Left nav — Home, Search, Your Library, CTA cards
│   │   ├── Display.jsx         # Center panel — Routes, per-album dynamic gradient background
│   │   ├── DisplayHome.jsx     # Home view — Featured Charts (albums) + Today's Hits (songs) rows
│   │   ├── DisplayAlbum.jsx    # Album detail — header, 4-col numbered song table, click-to-play
│   │   ├── Navber.jsx          # In-display top bar — back/forward nav, Explore Premium, avatar
│   │   ├── Player.jsx          # Bottom bar — seek bar, timestamps, play/pause/prev/next controls
│   │   ├── AlbumItem.jsx       # Album card — image, name, desc, navigates to /album/:id on click
│   │   └── SongItem.jsx        # Song card — image, name, desc, calls PlayWithId on click
│   ├── context/
│   │   └── PlayerContext.jsx   # All audio state — track, playStatus, time, seek, prev/next, play/pause
│   ├── assets/
│   │   ├── assets.js           # albumsData (6 albums), songsData (8 songs), all icon imports
│   │   ├── song1.mp3           # Audio file used by songs 0, 3, 6
│   │   ├── song2.mp3           # Audio file used by songs 1, 4, 7
│   │   ├── song3.mp3           # Audio file used by songs 2, 5
│   │   └── img1.jpg–img16.jpg  # Album and song cover images
│   ├── App.jsx                 # Root layout — Sidebar + Display (90%) + Player (10%) + audio element
│   ├── main.jsx                # React DOM entry, wraps App in PlayerContextProvider + BrowserRouter
│   └── index.css               # Minimal global styles
├── index.html                  # Vite HTML entry
├── vite.config.js              # Vite + React + Tailwind plugin config
└── package.json                # Dependencies and scripts
```

---

## Getting Started

**Prerequisites:** Node.js 18+

**1. Clone the repository**
```bash
git clone https://github.com/tripathipawan/Spotifyclone.git
cd Spotifyclone
```

**2. Install dependencies**
```bash
npm install
```

**3. Start the development server**
```bash
npm run dev
```

**4. Build for production**
```bash
npm run build
```

No API keys or environment variables required — all data and audio files are bundled locally.

---

## Repository

[https://github.com/tripathipawan/Spotifyclone](https://github.com/tripathipawan/Spotifyclone)
