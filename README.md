# Liberty-Classic-TV-
Matrix Stripper  System Architecture
Frontend
The frontend is built with React 18, TypeScript, Vite, shadcn/ui (Radix UI), and Tailwind CSS. It adheres to Material Design principles, emphasizing data density and a professional aesthetic, with support for custom theming (light/dark modes). State management is handled by TanStack Query and React hooks, while client-side routing uses Wouter.

Backend
The backend utilizes Node.js with Express.js, providing a RESTful API for CRUD operations. It manages M3U file uploads, robust parsing with regex, episode validation, bulk updates, duplicate detection, and M3U/JSON export capabilities. It also integrates with GitHub for repository synchronization and Archive.org for content import.

Data Storage
PostgreSQL, provided by Neon (a serverless database), is used for persistent data storage. Drizzle ORM facilitates type-safe interactions with the database. The schema includes tables for detailed episodes information and an archive_holding_queue for tracking Archive.org items. Drizzle Kit is used for schema migrations.

Core Features
The application supports M3U parsing and metadata extraction, episode management with CRUD operations, bulk updates, and renumbering, and URL validation with duplicate detection. It includes integration with Archive.org for content import and searching, and MS NOW for HLS news clip browsing. The Channel Player offers a universal full-page media playback experience with auto-advance, Smart Shuffle, and an Aria Schedule Drawer. A Master HLS 24-hour Linear Broadcast Scheduler generates deterministic broadcast streams. Features like the Cyclical Variety Engine ensure diverse episode playback, preventing repeats, while the Dead-Air Watchdog prevents playback interruptions. The system also includes a Program Director for content mixing and export options in M3U, JSON, HTML, and TXT formats. UI/UX elements feature a Workbench layout with a collapsible Shadcn Sidebar, ContextBar, SelectionBar, and a dark cyberpunk-inspired theme.

Two-Paddock Architecture (Player 1 / Player 2 Separation)
Player 1 and Player 2 run as strictly independent scheduling systems — each player has its own clock, break grid, and event routing. They share the same ingestion pipeline, episode DB, and filler pool, but nothing else.

Player 1 (TV News Player — 24h Linear Broadcast)

Clock: Pattern 1 — breaks at :05 (1 min), :29 (4 min), :57:50 (2 min + 10s station ID) per hour
Implemented in server/manifest.ts via BREAK_SCHEDULE constant (72 anchors/day)
Server-driven hard-slice schedule; followers sync via /api/watchdog/position
Receives FORCE_INJECT with targetPlayer: "all" only
REPACK events carry targetPlayer: "player1" — Player 2 ignores them
Player 2 (Live Player 2 — AJ Broadcast / VoD)

Clock: Pattern 2 — watchdog :00/:15/:30/:45 grid; AJ mode 900s NTD breaks
Implemented in server/watchdog.ts + client/src/pages/live-player-2.tsx
AJ-mode FORCE_INJECT carries targetPlayer: "player2" — Player 1 never receives 900s breaks
LP2 filters incoming SSE events by targetPlayer field; discards anything not addressed to it
Shared infrastructure (safe to share)

Episode ingestion, DB (episodes table with allowedPlayers column for future asset tagging)
Filler pool, last-aired cache, round-robin cursor
Live HLS pool, Archive.org import, holding queue
AJ Broadcast Mode (Live Player 2)
Sequential playback of Alex Jones Network full-hour MP4s with mandatory 15-minute NTD HLS breaks. Implemented in server/aj-pool.ts, server/watchdog.ts, server/routes.ts, and client/src/pages/live-player-2.tsx.

Feed source: https://rss.alexjones.media/hourly-mp4-HD.html (primary) — full-hour MP4s at https://ajn.archives.pub/hourly-mp4/HD/. Shows: Alex Jones Mon–Fri (4 h/day), War Room Mon–Fri (3 h/day), Alex Sun (2 h), TNT Radio Sun (2 h), Sortor Sun (1 h).

Date-window waterfall (applied each 4 h refresh):

Last 24 h — live/fresh run during weekday broadcasts
Last 7 days — weekend/holiday fill
All HD entries — beyond 7-day window
Legacy segments feed — opt-in via AJ_LEGACY_FALLBACK=true env var
STATIC_HD_FILENAMES — 35-file hardcoded guard
NTD breaks: watchdog fires FORCE_INJECT at wall-clock 15-min boundaries emitting breakDurationSec: 900. LP2 resumes AJ file at exact pre-break offset. Poll-fallback path skips breaks in AJ mode (SSE is authoritative). NTD URL selected by /ntd|newtangdynasty|ntdtv/i pattern match — deterministic regardless of pool reorder.

Cursor persistence: POST /api/aj-pool/cursor {fileIdx, offsetSec} — synced every 30 s and on break start. Creation-time wall-clock anchoring on fresh start (no saved cursor).

External Dependencies
Neon Database: Serverless PostgreSQL database.
GitHub API (Octokit): For interacting with GitHub repositories.
Archive.org: Used for content searching, browsing, and importing media.
NPM Packages: Key packages include @neondatabase/serverless, @octokit/rest, drizzle-orm, @tanstack/react-query, @radix-ui/*, tailwindcss, zod, express, multer, wouter, cheerio, and hls.js.
