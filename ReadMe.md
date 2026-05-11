# Daily Dev Mix

> A Spotify companion web application that builds personalized, activity-based playlists from your real listening behavior — not algorithmic guesses.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Application Flow](#application-flow)
- [Pages](#pages)
- [Tech Stack](#tech-stack)
- [Architecture](#architecture)
- [Database Schema](#database-schema)
- [Spotify API Integration](#spotify-api-integration)
- [Prerequisites](#prerequisites)
- [Running Locally](#running-locally)
- [Running Tests](#running-tests)
- [Project Structure](#project-structure)
- [Contributors](#contributors)
- [Deployed Application](#deployed-application)

---

## Overview

Daily Dev Mix is a full-stack web application built on top of the Spotify Web API. The core idea is simple: most music recommendation systems tell you what to listen to based on aggregate listening history or demographic data. Daily Dev Mix takes a different approach — it watches what you actually listen to while doing a specific activity, and builds a playlist of artist-based recommendations from that session's data.

The result is a set of playlists that are genuinely tied to context. The music you listen to while working out is different from what you listen to while studying, even if it's the same artist. Daily Dev Mix captures that distinction by separating listening sessions by user-defined "vibes" and building each playlist independently over time.

---

## Features

- **Spotify OAuth 2.0 authentication** — users sign in with their existing Spotify account, no separate registration required
- **Vibe-based session tracking** — users define named activities (vibes) and start a listening session tied to that vibe
- **Real-time playback display** — the active session page polls the Spotify API to show currently playing track data live during the session
- **Artist-based playlist generation** — when a session ends, the app queries the artists heard during that session and uses Spotify's Recommendations API to generate a curated playlist of new tracks
- **Persistent playlist building** — each successive session under the same vibe appends to and refines the same playlist, building accuracy over time
- **Playlist pushed directly to Spotify** — generated playlists are written back to the user's Spotify account and accessible from any Spotify client
- **Session history** — past sessions are stored and accessible from the user's profile

---

## Application Flow

```
Landing Page
    │
    ▼
Login / Spotify OAuth Authorization
    │
    ▼
Dashboard — select or create a vibe
    │
    ▼
Active Session — listen to music, playback tracked in real time
    │
    ▼
Session End — artist data queried, Spotify Recommendations API called
    │
    ▼
Playlist Page — generated playlist displayed, pushed to Spotify account
    │
    ▼
Profile / Logout
```

---

## Pages

### Landing Page
The entry point of the application. Introduces the concept and prompts the user to connect their Spotify account.

### Login / Spotify Authorization
Redirects the user through Spotify's OAuth 2.0 authorization flow. The user grants the application permission to read playback state, manage playlists, and access their profile. On successful authorization, Spotify returns an access token which is stored server-side in the user's session.

### Dashboard
The main hub after login. Displays the user's existing vibes and allows them to create new ones. Each vibe represents a named activity (e.g. "studying", "gym", "commute"). Selecting a vibe and clicking Start Session begins tracking.

### Active Session
The live tracking page. While the session is active, the application polls the Spotify API at regular intervals to capture the current playback state — track name, artist, album art — and logs each track to the database tied to the current session and vibe. The user can end the session at any time.

### Playlist Page
Displayed after a session ends. Shows the playlist generated from the session's artist data. Tracks are selected via the Spotify Recommendations API using the artists heard during the session as seed data. The playlist is automatically created in the user's Spotify account.

### Profile
Displays the user's session history organized by vibe, with links to associated playlists.

### Logout
Clears the server-side session and redirects to the landing page.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Handlebars (server-side rendering) |
| Backend | Node.js, Express |
| Database | PostgreSQL |
| Authentication | Spotify OAuth 2.0 with server-side session persistence |
| External API | Spotify Web API |
| Testing | Mocha + Chai |
| Containerization | Docker, Docker Compose |
| Deployment | Railway |

---

## Architecture

The application follows an MVC pattern built on Node.js and Express. Handlebars handles all page rendering server-side, keeping the frontend tightly coupled to the backend without requiring a separate client-side framework.

### Request Lifecycle

1. User requests arrive at Express routes defined across multiple route files
2. Route handlers interact with the PostgreSQL database via parameterized queries
3. Spotify API calls are made through a dedicated integration layer that manages access tokens stored in the server-side session
4. Data is passed to Handlebars templates for rendering and returned to the client

### Session Management

Spotify access tokens are stored server-side using Express session middleware. This keeps tokens off the client entirely and ensures authenticated context persists across requests without re-authorization. Each request to a protected route validates that a valid session token exists before proceeding.

### Spotify Integration Layer

All communication with the Spotify Web API is handled through a centralized service layer. This covers:
- OAuth authorization and token exchange
- Playback state polling during active sessions
- Artist seed extraction from session track logs
- Recommendations API querying
- Playlist creation and track population in the user's Spotify account

### Containerization

Docker Compose orchestrates two containers locally — the Node.js application server and the PostgreSQL database. The database is initialized on first run using an SQL init script that creates all required tables and seeds any default data. The same Docker configuration forms the basis of the Railway deployment.

---

## Database Schema

The database is organized around four core entities:

**users** — stores Spotify user ID, display name, and any persisted token data  
**vibes** — named activity categories created by users, linked to a user ID  
**sessions** — individual listening sessions, each tied to a user and a vibe with start/end timestamps  
**tracks** — individual track records logged during a session, storing Spotify track ID, track name, and artist name

This structure allows the application to reconstruct the full listening history for any vibe, extract the artists involved, and pass them as seed data to the Spotify Recommendations API at session end.

---

## Spotify API Integration

The application uses the following Spotify API endpoints:

| Endpoint | Purpose |
|---|---|
| `/authorize` | OAuth 2.0 authorization redirect |
| `/api/token` | Access token exchange |
| `/v1/me` | Fetch authenticated user profile |
| `/v1/me/player/currently-playing` | Poll current playback state during session |
| `/v1/recommendations` | Generate track recommendations from artist seeds |
| `/v1/users/{id}/playlists` | Create a new playlist in the user's account |
| `/v1/playlists/{id}/tracks` | Add recommended tracks to the generated playlist |

Required OAuth scopes: `user-read-playback-state`, `user-read-currently-playing`, `playlist-modify-public`, `playlist-modify-private`, `user-read-private`, `user-read-email`

---

## Prerequisites

- [Docker](https://www.docker.com/) and Docker Compose
- [Node.js](https://nodejs.org/) (v18 or later)
- A valid Spotify account
- Spotify Developer credentials — register an app at the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard) to obtain a Client ID and Client Secret

---

## Running Locally

```bash
# Clone the repository
git clone https://github.com/GregLowman/Daily-Dev-Mix.git
cd Daily-Dev-Mix/ProjectSourceCode

# Install dependencies
npm install
```

Create a `.env` file in the `ProjectSourceCode` directory with the following:

```
SPOTIFY_CLIENT_ID=your_client_id
SPOTIFY_CLIENT_SECRET=your_client_secret
SPOTIFY_REDIRECT_URI=http://localhost:3000/auth/callback
SESSION_SECRET=your_session_secret
```

Then start the application:

```bash
docker-compose up --build
```

Navigate to `http://localhost:3000` to access the application.

---

## Running Tests

Tests run automatically on `docker-compose up`. To run them independently:

```bash
npm test
```

The test suite uses Mocha as the test runner and Chai for assertions, covering core route behavior and API integration points.

---

## Project Structure

```
Daily-Dev-Mix/
├── ProjectSourceCode/          # Application source
│   ├── index.js                # Express app entry point and route definitions
│   ├── package.json
│   ├── docker-compose.yaml
│   ├── railway.json            # Railway deployment config
│   ├── init_data/
│   │   └── 00_create.sql       # Database schema and seed data
│   ├── init_db.sh              # Database initialization script
│   ├── resources/
│   │   ├── js/                 # Client-side JavaScript
│   │   └── css/                # Stylesheets
│   └── views/                  # Handlebars templates
│       ├── layouts/
│       └── pages/
├── MilestoneSubmissions/       # Course milestone deliverables
├── TeamMeetingLogs/            # Sprint meeting notes
└── README.md
```

---

## Contributors

This was a collaborative team project. Contributions by area:

| Contributor | GitHub | Contributions |
|---|---|---|
| Greg Lowman | [GregLowman](https://github.com/GregLowman) | Spotify API integration, Railway deployment (app + database), initial Handlebars shell, code structure, project management |
| Devang Pandey | [DevangPandey1](https://github.com/DevangPandey1) | Web development, frontend implementation |
| Dylan Long | [dylo3261](https://github.com/dylo3261) | Web development, frontend implementation |
| Ian Martin | [IanMartin110](https://github.com/IanMartin110) | Database design, Mocha + Chai testing, Handlebars templates |
| Matthew Aldridge | [Matthew-Aldridge](https://github.com/Matthew-Aldridge) | Database design, documentation, milestone reports |
| Vera Zaric | [vera-z05](https://github.com/vera-z05) | Page content and captions, project presentation, milestone reports |

---

## Deployed Application

[https://daily-dev-mix-production.up.railway.app](https://daily-dev-mix-production.up.railway.app)

> **Note:** A valid Spotify account is required to use the application. The Spotify Developer app is currently in development mode, which limits access to pre-approved accounts. If you would like access for testing purposes, please open an issue or reach out directly.
