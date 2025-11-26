# UCHATTR

> UCHATTR is a lightweight, self-hosted chat platform with public rooms, private rooms/DMs, polls, server-authoritative mini-games (battles), and replay support.

This README summarizes key user features, admin/developer notes, setup & run instructions, and quick testing steps. It has been updated to reflect recent behavior changes (battle timer, poll persistence, replay UX, and opt-out preferences).

## Quick start

- Install dependencies:

  `npm install`

- Start locally:

  `node server.js`

- (Optional) Start with PM2 using the included ecosystem file:

  `pm2 start ecosystem.config.js`

The server listens for HTTP and WebSocket traffic (see `server.js`). Static client files are served from the `public/` directory.

## Key user-facing changes (recent)

- Battles now use a fixed maximum duration: a 10-minute timer (set when a battle starts). Clients receive `timeoutExpiresAt` in `battle_state` so UIs show a synchronized countdown.
- When a battle ends the room is converted to a public, read-only replay room (clears `allowed_users`, sets `read_only=1`, and clears `is_battle` so live banners hide). The room is scheduled for deletion 1 hour after match end (stored in `pending_room_deletes`).
- System messages are now persisted (they no longer auto-delete client-side).
- Users can opt out of battle announcements via the `block_battles` preference; the server omits battle-related system messages from broadcasts and filters history returned to opted-out users.
- The replay toggle was moved out of the battle banner into the system announcement link — clicking the announcement requests the persisted replay history and renders it in-room.
- Polls are fully persisted (DB-backed), support multi-word options (pipe-separated or quoted), and are removed from the UI when their linked persisted message is deleted.

## Quick feature reference

Polls

- Create: `/poll <question> <option 1> <option 2> ...` — supports pipe-separated or quoted arguments for multi-word content.
- Server persists polls and votes. Clients receive `poll_update` broadcasts. Admins can delete/close polls and clients will remove the poll UI on `poll_deleted` or when the underlying message record is deleted.

Battles (/fight)

- Start: `/fight <username>` — creates a private battle room for two participants.
- Timer: Battles have a fixed 10-minute max duration from start (server-synchronized). The server computes winner on timeouts and records results.
- Persistence: Rolls are recorded into `battle_history` and `battle_sessions` to support replay. Battle rooms are converted to read-only replay rooms on end and scheduled for deletion after 1 hour.
- Opt-out: Users with `block_battles = 1` will not receive battle-related system messages or replay announcements.

System messages

- Persisted in the `messages` table. Battle-related system messages are tagged with `is_battle = 1` so history and broadcasts can filter them for opted-out users.

Unread behavior

- System messages and polls do not mark rooms as unread (they won't create red-dot unread indicators for normal users).

## Privacy & E2EE notes

- Regular DMs use E2EE. Battle rooms are intentionally not E2EE so the server can persist roll history for replay. The client disables E2EE for `is_battle` rooms.
- Passwords are hashed with bcrypt. Auth tokens are stored in the `users.token` column for session identification.

## Storage / Database (concise)

- SQLite3 (`uchattr-app.db`). Important tables: `users`, `rooms`, `messages` (now includes `is_battle`), `polls`/`poll_options`/`poll_votes`, `battle_sessions`, `battle_history`, `pending_room_deletes`.

## Admin & moderation notes

- Admin endpoints exist for poll deletion/closure and for managing rooms. Moderators should use `/admin.html`.
- The server will not force client room switches; instead it notifies clients (`update_rooms`, `notification`) and clients decide whether to join.

## Developer notes

- Key server file: `server.js` — handles HTTP routes, WebSocket messages, battle engine, room lifecycle, emoji uploads, and admin actions.
- Client: `public/index.html`, `public/scripts.js` — look for `joinRoom`, `battle_state`, `battle_log`, `poll_update`, `poll_deleted`, and `block_battles` handling.

## How to test critical flows (short)

1. Register two users or open two browsers and log in.
2. Use `/fight <user>` to challenge and accept; verify the private battle room is created.
3. Play the match; observe `battle_state` and `battle_log` updates. Confirm `timeoutExpiresAt` is present.
4. After the match ends, verify the room becomes read-only and a public replay announcement appears in General (unless the receiver has `block_battles` enabled).
5. Click the replay announcement to request persisted rolls from the server and verify the client renders the `battle_log` entries.

## Deployment tips

- Run `npm install` and start with `node server.js` or via PM2.
- For production, use a reverse proxy (Nginx) with TLS and enable proper file permissions for `public/emojis`.

## Where to look in code

- Server: `server.js` — `createDMRoomForUsers`, `performBattleRoll`, `endBattleAndCleanup`, `sendSystemMessage`.
- Client: `public/scripts.js` — `renderPoll`, `updatePollUI`, `appendSystemMessage`, `appendReplayAnnouncement`, and unread-dot logic.
