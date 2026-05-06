# Installation Guide

This guide sets up the complete Spotify Clone project for local development.

The repository contains two separate Node applications:

- `Spotify_clone_backend`: Express backend for Spotify OAuth and selected Spotify Web API proxy routes
- `Spotify_clone_frontend`: React/Vite frontend for the UI and Spotify Web Playback SDK

Do not commit real `.env` files or Spotify credentials.

## Prerequisites

Install these before starting:

- Node.js and npm. Use a current LTS Node.js version; Node 20 or newer is recommended for this Vite/React/TypeScript stack.
- A Spotify account.
- A Spotify Developer Dashboard app.
- Spotify Premium for playback through the Spotify Web Playback SDK.
- Network access to:
  - `https://accounts.spotify.com`
  - `https://api.spotify.com`
  - `https://sdk.scdn.co`

## 1. Install Project Dependencies

From the repository root:

```powershell
cd "C:\Users\LENOVO\Projects\Personal Projects\Spotify_Clone"
```

Install backend dependencies:

```powershell
cd Spotify_clone_backend
npm install
```

Install frontend dependencies:

```powershell
cd ..\Spotify_clone_frontend
npm install
```

`npm install` uses each app's `package-lock.json`, so new developers should not need to install individual packages by hand.

## 2. Create Spotify Credentials

In the Spotify Developer Dashboard:

1. Create an app.
2. Copy the app's Client ID.
3. Copy the app's Client Secret.
4. Add this exact redirect URI to the app:

```text
http://127.0.0.1:5000/auth/callback
```

5. If the app is in a restricted/development mode, add each developer's Spotify account as an allowed user/tester.
6. Use a Spotify Premium account when testing playback.

The backend requests these scopes during login:

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

## 3. Configure Backend Environment

Create or update:

```text
Spotify_clone_backend/.env
```

Use this shape:

```env
PORT=5000
SPOTIFY_CLIENT_ID=your_spotify_client_id
SPOTIFY_CLIENT_SECRET=your_spotify_client_secret
REDIRECT_URI=http://127.0.0.1:5000/auth/callback
FRONTEND_URI=http://localhost:5173

# Present in the local env file, but not currently used by the code.
SUPABASE_URL=
SUPABASE_KEY=
```

Important: the current backend code reads `REDIRECT_URI`, not `SPOTIFY_REDIRECT_URI`. If your existing env file contains `SPOTIFY_REDIRECT_URI`, duplicate the same value into `REDIRECT_URI` or rename the variable.

The backend keeps `SPOTIFY_CLIENT_SECRET` server-side. Never add Spotify secrets to frontend env files.

## 4. Configure Frontend Environment

Create or update:

```text
Spotify_clone_frontend/.env
```

Use:

```env
VITE_BACKEND_URL=http://127.0.0.1:5000
```

This must match the backend origin used by the browser. The frontend calls auth routes with `credentials: include`, so cookies must be set and sent for this same backend origin.

## 5. Run the Project

Start the backend in one terminal:

```powershell
cd "C:\Users\LENOVO\Projects\Personal Projects\Spotify_Clone\Spotify_clone_backend"
npm run dev
```

The backend should run at:

```text
http://localhost:5000
```

Start the frontend in a second terminal:

```powershell
cd "C:\Users\LENOVO\Projects\Personal Projects\Spotify_Clone\Spotify_clone_frontend"
npm run dev
```

Vite should run at:

```text
http://localhost:5173
```

Open `http://localhost:5173`, click "Log In with Spotify", approve the scopes, and return to the app.

## 6. Verify the Setup

Backend health check:

```powershell
Invoke-WebRequest http://127.0.0.1:5000/ | Select-Object -ExpandProperty Content
```

Expected response contains:

```text
The Spotify Clone Backend is running
```

Frontend verification:

```powershell
cd "C:\Users\LENOVO\Projects\Personal Projects\Spotify_Clone\Spotify_clone_frontend"
npm run build
```

Backend test:

```powershell
cd "C:\Users\LENOVO\Projects\Personal Projects\Spotify_Clone\Spotify_clone_backend"
npm test
```

Frontend test:

```powershell
cd "C:\Users\LENOVO\Projects\Personal Projects\Spotify_Clone\Spotify_clone_frontend"
npm test
```

Note: `vite.config.ts` currently points Vitest setup to `./src/test/setup.js`, while the repo contains `src/test/setup.ts`. If frontend tests fail because the setup file cannot be resolved, align that config with the TypeScript setup file.

## Available Scripts

Backend scripts from `Spotify_clone_backend/package.json`:

| Command | Purpose |
|---|---|
| `npm run dev` | Starts the backend with `nodemon --exec ts-node server.ts` |
| `npm start` | Starts the backend with `ts-node server.ts` |
| `npm test` | Runs Jest tests |

Frontend scripts from `Spotify_clone_frontend/package.json`:

| Command | Purpose |
|---|---|
| `npm run dev` | Starts the Vite dev server |
| `npm run build` | Builds the frontend for production |
| `npm run preview` | Serves the production build locally |
| `npm run lint` | Runs ESLint |
| `npm test` | Runs Vitest tests |

## Dependency Inventory

### Backend Runtime Dependencies

Installed from `Spotify_clone_backend/package.json`:

```text
axios
cookie-parser
cors
dotenv
express
express-rate-limit
helmet
```

### Backend Development Dependencies

```text
@types/cookie-parser
@types/cors
@types/express
@types/jest
@types/node
@types/supertest
jest
nodemon
supertest
ts-jest
ts-node
typescript
```

### Frontend Runtime Dependencies

Installed from `Spotify_clone_frontend/package.json`:

```text
@reduxjs/toolkit
react
react-dom
react-redux
react-router-dom
```

### Frontend Development Dependencies

```text
@eslint/js
@testing-library/jest-dom
@testing-library/react
@types/node
@types/react
@types/react-dom
@types/spotify-web-playback-sdk
@vitejs/plugin-react
eslint
eslint-plugin-react-hooks
eslint-plugin-react-refresh
globals
happy-dom
typescript
vite
vitest
```

### External Runtime Dependencies

These are not installed through npm but are required at runtime:

```text
Spotify Accounts API
Spotify Web API
Spotify Web Playback SDK
Spotify Premium account for playback
Browser support for credentialed cross-origin requests and cookies
```

## Troubleshooting

### Login redirects but the frontend is still logged out

Check:

- `Spotify_clone_backend/.env` contains `REDIRECT_URI`, not only `SPOTIFY_REDIRECT_URI`.
- The Spotify Dashboard redirect URI exactly matches `http://127.0.0.1:5000/auth/callback`.
- `FRONTEND_URI=http://localhost:5173`.
- `VITE_BACKEND_URL=http://127.0.0.1:5000`.
- Browser devtools show auth cookies being accepted for `127.0.0.1`.

The backend currently sets cookies with `secure: true` and `sameSite: 'none'`. Some local HTTP/browser combinations may reject or withhold those cookies.

### CORS errors

Make sure the frontend is opened at the same origin as `FRONTEND_URI`. With the default config, use:

```text
http://localhost:5173
```

Do not open the Vite app on a different host or port unless you also update `FRONTEND_URI`.

### Invalid redirect URI

The URI in three places must match the local backend callback:

- Spotify Developer Dashboard redirect URI
- `Spotify_clone_backend/.env` `REDIRECT_URI`
- The backend URL that is actually running

Default value:

```text
http://127.0.0.1:5000/auth/callback
```

### Playback fails with Premium or account errors

The Spotify Web Playback SDK requires Spotify Premium. The logged-in Spotify account must be Premium and allowed to use the developer app if the app is not public.

### Home page is empty

The Home page shows saved tracks from `https://api.spotify.com/v1/me/tracks`. If the logged-in account has no liked songs, the page can appear empty even though auth works.

### Search works but Library or Playlist pages fail

Search goes through the backend. Library and playlist detail pages call Spotify directly from the browser. Check browser network errors for Spotify `401`, `403`, or missing scope responses.

## Development Notes

- Backend and frontend each have their own `package.json`, `package-lock.json`, `.env`, and `.gitignore`.
- Keep `SPOTIFY_CLIENT_SECRET` only in `Spotify_clone_backend/.env`.
- Do not put backend-only secrets into `VITE_` variables. Vite exposes `VITE_` variables to browser code.
- Use `npm install` in both app directories after pulling dependency changes.
- Restart both dev servers after changing `.env` files.
