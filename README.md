# Spotify Clone Architecture

This document describes the current codebase structure and runtime behavior. The project is split into two independent TypeScript applications:

- `Spotify_clone_frontend`: React 19, Vite, Redux Toolkit, React Router
- `Spotify_clone_backend`: Node.js, Express, TypeScript, Spotify OAuth proxy

The frontend owns the UI, player controls, route-level pages, and Spotify Web Playback SDK lifecycle. The backend owns Spotify OAuth code exchange, refresh-token handling, CORS, rate limiting, and the Spotify API calls that should be proxied through a trusted server.

## Project Layout

```text
Spotify_Clone/
|-- README.md
|-- INSTALLATION.md
|-- Spotify_clone_backend/
|   |-- server.ts
|   |-- package.json
|   |-- tsconfig.json
|   |-- jest.config.js
|   |-- .env
|   |-- .env.example
|   |-- config/
|   |   `-- endpoints.ts
|   |-- controllers/
|   |   |-- authController.ts
|   |   `-- spotifyController.ts
|   |-- routes/
|   |   |-- authRoutes.ts
|   |   `-- spotifyRoutes.ts
|   `-- tests/
|       `-- app.test.ts
`-- Spotify_clone_frontend/
    |-- package.json
    |-- vite.config.ts
    |-- tsconfig*.json
    |-- .env
    |-- public/
    `-- src/
        |-- main.tsx
        |-- App.tsx
        |-- types.d.ts
        |-- context/
        |   `-- PlayerContext.tsx
        |-- hooks/
        |   |-- useSpotifyAuth.ts
        |   `-- useSpotifySDK.ts
        |-- store/
        |   |-- store.ts
        |   `-- playerSlice.ts
        |-- pages/
        |   |-- Home.tsx
        |   |-- Search.tsx
        |   `-- Library.tsx
        |-- components/
        |   |-- Header.tsx
        |   |-- Sidebar.tsx
        |   |-- BottomPlayer.tsx
        |   |-- SongCard.tsx
        |   `-- PlaylistDetails.tsx
        `-- test/
            `-- setup.ts
```

## Runtime Topology

```text
Browser on http://localhost:5173
  |
  | Frontend API calls with VITE_BACKEND_URL
  v
Express backend on http://127.0.0.1:5000
  |
  | OAuth authorize, token exchange, refresh, liked songs, search
  v
Spotify Accounts API and Spotify Web API

Browser also talks directly to Spotify for:
  - Web Playback SDK script: https://sdk.scdn.co/spotify-player.js
  - Playback commands: PUT https://api.spotify.com/v1/me/player/play
  - User playlists and playlist details
```

The backend exists because `SPOTIFY_CLIENT_SECRET` must stay out of browser code. Playback commands still run from the browser because they use the logged-in user's access token and target the SDK device running in that same browser session.

## Frontend Architecture

### Entry Point

`src/main.tsx` mounts the React app and wraps it in:

1. Redux `Provider` from `react-redux`
2. `PlayerProvider` from `src/context/PlayerContext.tsx`
3. `App`

This gives every route access to persisted player state through Redux and playback/auth actions through context.

### Routes

`src/App.tsx` uses `BrowserRouter` with these client routes:

| Route | Component | Purpose |
|---|---|---|
| `/` | `Home` | Shows the user's liked songs with infinite scroll |
| `/search` | `Search` | Searches Spotify tracks through the backend |
| `/library` | `Library` | Lists the user's Spotify playlists directly from Spotify |
| `/playlist/:id` | `PlaylistDetails` | Shows playlist metadata and tracks directly from Spotify |

`Sidebar` is always visible inside the main layout. `BottomPlayer` is always mounted below the route content and only renders when a current track exists.

### State Model

The frontend has three state layers:

| Layer | Location | Stores |
|---|---|---|
| Redux Toolkit | `src/store/playerSlice.ts` | `currentTrack`, `isPlaying`, `position`, `duration`, SDK readiness, `deviceId` |
| React Context | `src/context/PlayerContext.tsx` | Auth functions, playback commands, SDK state, player error message |
| Component state | Pages/components | Search text, loading flags, fetched songs/playlists, local progress |

`playerSlice` persists part of the player state to `localStorage` under `spotify_player_state`. This lets the app restore the last known track and position after reload. SDK readiness and device ID are not restored from storage because they are tied to the current browser SDK connection.

### Authentication Hook

`src/hooks/useSpotifyAuth.ts` owns token discovery and refresh from the frontend perspective.

On first load it calls:

```text
GET {VITE_BACKEND_URL}/auth/token
credentials: include
```

The backend reads HttpOnly cookies and returns the current access token and expiry timestamp as JSON. The hook stores those values in React state.

Every 60 seconds, if the token will expire in less than five minutes, the hook calls:

```text
POST {VITE_BACKEND_URL}/auth/refresh
credentials: include
```

Logout calls:

```text
POST {VITE_BACKEND_URL}/auth/logout
credentials: include
```

Then it clears `spotify_player_state`, clears local auth state, and redirects to `/`.

### Spotify SDK Hook

`src/hooks/useSpotifySDK.ts` owns the Spotify Web Playback SDK lifecycle:

1. Waits until an access token exists.
2. Defines `window.onSpotifyWebPlaybackSDKReady`.
3. Injects `https://sdk.scdn.co/spotify-player.js` if it is not already present.
4. Creates `new window.Spotify.Player({ name: 'My Spotify Clone', getOAuthToken, volume: 0.5 })`.
5. Registers SDK event listeners:
   - `ready`: stores `device_id`, marks the player ready, attempts auto-resume from `localStorage`
   - `not_ready`: clears readiness and device ID
   - `player_state_changed`: syncs current track, paused state, position, and duration into Redux
   - SDK error listeners: log initialization, authentication, account, and playback failures
6. Calls `player.connect()`.
7. Disconnects the SDK player on cleanup.

Playback requires Spotify Premium because Spotify's Web Playback SDK only supports Premium playback.

### Player Context

`src/context/PlayerContext.tsx` combines `useSpotifyAuth` and `useSpotifySDK` into one app-level context.

It exposes:

```text
token
login()
logout()
playTrack(trackOrUri, pos_ms?, devId?, contextUri?)
pauseTrack()
resumeTrack()
seek(pos_ms)
togglePlay()
nextTrack()
previousTrack()
currentTrack
isPlaying
deviceId
isReady
position
duration
playerError
```

`playTrack` handles both track objects and Spotify URI strings. It optimistically updates Redux before calling Spotify, then sends:

```text
PUT https://api.spotify.com/v1/me/player/play?device_id={deviceId}
Authorization: Bearer {accessToken}
Content-Type: application/json
```

For individual tracks the body is:

```json
{ "uris": ["spotify:track:..."] }
```

For playlist or album context the body is:

```json
{
  "context_uri": "spotify:playlist:...",
  "offset": { "uri": "spotify:track:..." }
}
```

Errors are shown through a temporary fixed-position player error message.

### Page and Component Responsibilities

| File | Responsibility |
|---|---|
| `Home.tsx` | Fetches liked songs from the backend route named `/api/spotify/new-releases`; supports offset/limit pagination and `IntersectionObserver` infinite scroll |
| `Search.tsx` | Debounces the search input by 500 ms and calls backend `/api/spotify/search?q=...` |
| `Library.tsx` | Calls Spotify directly at `/v1/me/playlists?limit=50` and renders playlist cards |
| `PlaylistDetails.tsx` | Calls Spotify directly for playlist metadata and playlist items; passes playlist context URI into `SongCard` |
| `SongCard.tsx` | Renders one track and toggles play/pause through `usePlayer()` |
| `BottomPlayer.tsx` | Reads Redux player state, shows current track metadata, controls playback, and tracks local progress between SDK updates |
| `Header.tsx` | Shows login/logout buttons based on whether a token is available |
| `Sidebar.tsx` | Provides links to Home, Search, and Library |

## Backend Architecture

### Server Setup

`Spotify_clone_backend/server.ts` creates the Express app and applies middleware in this order:

1. `helmet()` for baseline HTTP security headers
2. `express-rate-limit` with 100 requests per IP per 15 minutes
3. `cors({ origin: FRONTEND_URI, credentials: true })`
4. `express.json()`
5. `cookieParser()`
6. `/api/spotify` routes
7. `/auth` routes
8. `/` health/root response

The app starts listening only when `NODE_ENV !== 'test'`, which lets `supertest` import the Express app during Jest tests without opening a network port.

### Backend Routes

| Method | Route | Controller | Purpose |
|---|---|---|---|
| `GET` | `/` | `server.ts` | Basic backend health response |
| `GET` | `/auth/login` | `login` | Redirects the browser to Spotify's authorize URL |
| `GET` | `/auth/callback` | `callback` | Exchanges Spotify's authorization code for tokens, stores cookies, redirects to frontend |
| `GET` | `/auth/token` | `getToken` | Returns access token and expiry from cookies |
| `POST` | `/auth/refresh` | `refresh` | Uses refresh-token cookie to get a new access token |
| `POST` | `/auth/logout` | `logout` | Clears auth cookies |
| `GET` | `/api/spotify/new-releases` | `getNewReleases` | Fetches the user's liked tracks from Spotify `/v1/me/tracks` |
| `GET` | `/api/spotify/search` | `searchSpotify` | Searches Spotify tracks |

### Spotify Endpoint Configuration

`config/endpoints.ts` centralizes Spotify URLs:

```text
TOKEN_URL: https://accounts.spotify.com/api/token
BASE_URL: https://api.spotify.com/v1
NEW_RELEASES: https://api.spotify.com/v1/me/tracks
FEATURED_PLAYLISTS: https://api.spotify.com/v1/browse/featured-playlists
CATEGORIES: https://api.spotify.com/v1/browse/categories
SEARCH: https://api.spotify.com/v1/search
```

Important naming note: `NEW_RELEASES` and `/api/spotify/new-releases` currently return saved tracks from `/v1/me/tracks`, not Spotify new releases. The UI labels this data as "Your Liked Songs".

### OAuth Flow

The backend implements Spotify Authorization Code Flow.

1. User clicks "Log In with Spotify".
2. Frontend redirects to `{VITE_BACKEND_URL}/auth/login`.
3. Backend builds `https://accounts.spotify.com/authorize` with:
   - `response_type=code`
   - `client_id`
   - `redirect_uri`
   - requested scopes
4. Spotify redirects back to `REDIRECT_URI`, normally `http://127.0.0.1:5000/auth/callback`.
5. Backend exchanges the authorization code at `https://accounts.spotify.com/api/token`.
6. Backend stores `access_token`, `refresh_token`, and `expires_at` in cookies.
7. Backend redirects the browser to `FRONTEND_URI`, normally `http://localhost:5173/`.
8. Frontend calls `/auth/token` with credentials to hydrate auth state.
9. Frontend refreshes tokens through `/auth/refresh` when expiry is near.

Requested Spotify scopes:

```text
streaming
user-read-email
user-read-private
user-library-read
user-library-modify
user-read-playback-state
user-modify-playback-state
playlist-read-private
playlist-read-collaborative
```

### Cookie Behavior

Auth cookies are created with:

```ts
{ httpOnly: true, secure: true, sameSite: 'none' }
```

This keeps cookies unavailable to browser JavaScript, but the current frontend still receives the access token through `/auth/token` because browser-side Spotify SDK and Web API calls need a bearer token.

The `secure: true` and `sameSite: 'none'` settings are production-oriented. During local HTTP development, cookie behavior depends on browser handling of secure cookies on localhost-style origins. If login appears to succeed but the frontend has no token, inspect whether the cookies were accepted and sent back to `http://127.0.0.1:5000`.

## Environment and Configuration

### Backend Environment

Current backend code reads these variables:

```text
PORT=5000
SPOTIFY_CLIENT_ID=your_spotify_client_id
SPOTIFY_CLIENT_SECRET=your_spotify_client_secret
REDIRECT_URI=http://127.0.0.1:5000/auth/callback
FRONTEND_URI=http://localhost:5173
```

The local backend `.env` also contains:

```text
SUPABASE_URL=...
SUPABASE_KEY=...
SPOTIFY_REDIRECT_URI=...
```

Those names are not currently read anywhere in the code. If an existing env file only has `SPOTIFY_REDIRECT_URI`, duplicate or rename it to `REDIRECT_URI` for the current backend to use it.

### Frontend Environment

The frontend reads:

```text
VITE_BACKEND_URL=http://127.0.0.1:5000
```

Every frontend call to the Express backend uses this value. It must point to the same origin that sets the auth cookies.

## Data Flow Examples

### Liked Songs on Home

```text
Home.tsx
  -> GET {VITE_BACKEND_URL}/api/spotify/new-releases?offset=0&limit=20
     Authorization: Bearer {token}
  -> spotifyController.getNewReleases
  -> GET https://api.spotify.com/v1/me/tracks
  -> map saved-track wrapper objects into frontend SpotifyTrack shape
  -> render SongCard list
```

### Search

```text
Search.tsx
  -> debounce input for 500 ms
  -> GET {VITE_BACKEND_URL}/api/spotify/search?q={query}
     Authorization: Bearer {token}
  -> spotifyController.searchSpotify
  -> GET https://api.spotify.com/v1/search?type=track&limit=10&q={query}
  -> map track items into frontend SpotifyTrack shape
  -> render SongCard list
```

### Playing a Track

```text
SongCard click
  -> PlayerContext.playTrack(track, 0, undefined, contextUri?)
  -> optimistic Redux update
  -> PUT https://api.spotify.com/v1/me/player/play?device_id={deviceId}
  -> Spotify SDK emits player_state_changed
  -> useSpotifySDK syncs SDK state back to Redux
  -> BottomPlayer updates metadata, controls, and progress
```

## Testing and Quality Tooling

### Backend

The backend uses:

- Jest
- ts-jest
- Supertest
- TypeScript strict mode

`tests/app.test.ts` imports the Express app and checks the root endpoint.

Commands:

```powershell
cd Spotify_clone_backend
npm test
```

### Frontend

The frontend uses:

- Vite build tooling
- Vitest
- Testing Library
- happy-dom
- ESLint
- TypeScript strict mode

Commands:

```powershell
cd Spotify_clone_frontend
npm run build
npm run lint
npm test
```

Current test-config note: `vite.config.ts` references `./src/test/setup.js`, while the repository contains `src/test/setup.ts`. If Vitest cannot resolve setup files, that config should be aligned with the TypeScript setup file.

## Current Risks and Follow-Up Work

| Area | Current state | Risk or improvement |
|---|---|---|
| OAuth cookies | Cookies use `secure: true` and `sameSite: 'none'` | Local HTTP development may fail to retain cookies in some browsers |
| Access token exposure | `/auth/token` returns the access token to the frontend | Required for current SDK/API design, but still exposes the bearer token to browser runtime |
| Route naming | `/api/spotify/new-releases` returns liked songs | Rename route/config to avoid future confusion |
| Direct Spotify calls | Library, playlist details, and playback call Spotify from the browser | Backend cannot centrally normalize errors or rate-limit those paths |
| Supabase env | Supabase variables exist in `.env` | No Supabase integration is currently implemented |
| Frontend test setup | Vite config points to `setup.js` while file is `setup.ts` | Vitest setup resolution may need correction |

## Summary

The current app is a two-service Spotify client:

- React renders the UI, keeps player state in Redux, and runs the Spotify Web Playback SDK.
- Express protects the Spotify client secret, performs OAuth code exchange, refreshes tokens, and proxies liked-song/search data.
- Spotify remains the source of truth for authentication, playback state, playlists, and track catalog data.

The most important operational dependency is correct OAuth configuration: `REDIRECT_URI`, `FRONTEND_URI`, `VITE_BACKEND_URL`, and the redirect URI registered in the Spotify Developer Dashboard must all agree with the origins used in the browser.
