# Discord plugin: the laris-co patch

The official `discord@claude-plugins-official` plugin listens only in channels listed one by one in
`access.json`, and it never opens threads. The team wants this instead: **tag a bot in any room, and it
answers in a thread on that message**. This folder keeps the code we run, so any machine can install it
and anyone can compare it with upstream.

```
plugin/discord/
  upstream-0.0.4/   the plugin exactly as shipped (server.ts, package.json, .mcp.json), for comparison
  server.ts         what we run: upstream plus the patch below
  laris-co.patch    diff -u upstream-0.0.4/server.ts server.ts
```

Install on a machine with `just plugin-patch`, then `/mcp` reconnect (or restart) each `--channels` session.
`just plugin-patch check` only reports. The script replaces a cached `server.ts` only if it is upstream,
a prior hand patch it knows about, or already ours. Anything else it leaves alone.

## What the patch adds (all opt-in in access.json)

- `guilds: { "<guildId>": <policy> }` sets a default for every room in that guild that is not listed in `groups`.
- `threadOnMention: true` (per policy): a tag in a plain channel opens a thread on that message, and the
  message reaches Claude with the thread as `chat_id`. If two bots are tagged in one message, the second one
  joins the thread the first one opened.
- In a thread the bot opened, follow-ups need no tag.
- **`!thread <name>`** opens a thread called `<name>` on that message, and the message reaches Claude
  from inside it, so the bot answers there. A bare **`!thread`** opens one with a placeholder name. The message
  arrives with `thread_new`, and the agent names the thread with the **`rename_thread`** tool once it knows the topic.
- **Mode toggle per room:** `!thread on` switches the room to thread mode, and the bot reacts 🧵.
  `!thread off` switches it back to answering in the room, and the bot reacts 💬. The setting is saved in
  `<state dir>/modes.json`, so it survives restarts. When a room has no setting, `threadOnMention` decides
  (the laris-co default is off, so answers stay in the room). Bots that only answer tags need the tag on
  the command (`@bot !thread on`). Each bot keeps its own mode.
- `!here` unlocks a room, like waking a phone screen. Tag the bot with `!here`, for example
  `@LarisCo !here what's on today`. While the room is unlocked, that bot answers in the room, not in a
  thread, and hears every allowed message without a tag. Each message resets the idle timer. After
  10 minutes with no messages the room locks again. `!here 30m` or `!here 2h` sets a longer timer for
  that unlock. `!lock` locks the room now. The bot reacts 🔓 / 🔒. Only the tagged bot unlocks.
  The words and the default timer can be changed in access.json:
  `"here": { "unlock": "!here", "lock": "!lock", "idleMinutes": 10 }`.
- `historyBackfill: N` (per policy): on a tag, the plugin reads back through the room until it reaches
  this bot's own last message, or N messages. It hands those lines to Claude as `meta.context`, oldest
  first, so the bot knows what was said before it was tagged. Senders not on the allowlist are marked
  `[unverified]`. The idea comes from Hermes (`DISCORD_HISTORY_BACKFILL`).
- Every guild message reaches Claude with `tagged="true"` or `"false"` in meta. When the bot answers in a
  plain channel with `reply_to`, the plugin opens a thread on that message and answers there (only where
  `threadOnMention` is on, and not while the room is `!here` unlocked). With `requireMention: false`, a bot
  can read every message and still keep its answers in threads.
- `forwardTo: <worktree path>` (per policy): the room belongs to a worktree body of this oracle. The main
  body keeps the only gateway connection and types each message into the herdr agent whose cwd is that
  path, then reacts 📨. The worktree answers with `just say <chat_id> "<text>"`, which is send-only REST
  and does not listen. If no agent runs there, the message is delivered to the main body as usual. In
  config.yaml, write the path relative to the main checkout (`wt/<slug>`); `access-sync` makes it absolute.
- **Rooms per oracle, routed by the host.** An oracle lists its rooms in `oracles/<name>/config.yaml`
  `rooms:` (`id`, `body`, `what`). `access-sync` for the host bot (LarisCo, `host: true`) turns each room
  into a route: `forwardTo` = the body (a worktree path, matched on agent cwd in any running herdr session,
  or an exact `session:pane`), `answerAs` = the oracle's bot, and `answerMention` = its `owner`. The body
  answers with `BOT=<oracle> just say`. `just room-map <name>` posts and pins a map in the oracle's main
  channel showing thread → body → status, and edits that same message on later runs.
- `answerCmd` (set by access-sync): the full send-only command (`BOT=<bot> <laris-team>/tools/discord-say.sh`),
  so a forwarded body in another repo (neo-oracle, calliope-oracle) can answer without the laris-team justfile.
- `roleCall: [<role id>]`: a message from an `allowFrom` user that tags one of these roles (Laris Co: All Oracles)
  reaches the bot in any room, even one its policy denies, and the bot may answer in that room. A bot that is not
  running does not answer: a roll call never wakes a sleeping bot.
- **Roll call in a thread joins it.** A role tag (All Oracles or a team role) inside a thread makes every awake
  bot holding that role join the thread. After that, every message there from an `allowFrom` user reaches those
  bots without a tag. `!leave` in the thread makes every bot leave, and `!kick @Bot [@Bot…]` makes only the tagged bots leave
  (each one reacts 👋). Joined threads are saved in
  `<state dir>/joined.json`, so they survive restarts.
- `deny: true` on a group means never listen there, even if the guild default would.
- A thread can be allowed by its own id. This is the earlier hand patch from 2026-10-17 10:40, now folded in.

Without these keys, the plugin behaves exactly like upstream.

## Per-bot access

Each bot still has its own `access.json` in its own state dir (`~/.claude/channels/<name>/`).
`just access-sync <name>` writes it from the repo:

1. `oracles/access-defaults.yaml`: the shared default (Laris Co Ltd: tag gets a thread, Nat allowed)
2. `oracles/<name>/config.yaml` `access:` block: whatever this bot needs that is different
3. `oracles/<name>/access.jsonl`: append-only allow/deny rows

Keep the defaults small and put the differences in each bot's `config.yaml`. Example: a bot that should
hear #media-team without a tag, and never read #บัญชี:

```yaml
access:
  groups:
    "<channel-id>": { requireMention: false }
    "<other-channel-id>": { deny: true }
```

## Concerns and open questions

- **Plugin updates overwrite the cache.** After an update, `just plugin-patch check` reports UNKNOWN.
  Copy the new upstream into `upstream-<ver>/`, re-apply `laris-co.patch` to it, and commit.
- **One body per bot.** Only the main checkout runs `--channels`. A worktree never connects as the bot.
  If a worktree should get only some of the work, route a thread to it (`maw atlas route` / `team-threads`)
  instead of giving it an `access.json`.
- **Every machine needs the patch.** A bot on white runs white's plugin cache. If it is unpatched, it ignores
  `guilds` and listens only in the rooms listed in `groups`.
- **Threads someone else opened** (by Nat, or by another bot) still need a tag on each message. Should a bot
  keep listening in a thread once it has been tagged there? Not decided.
- **Thread names** are the first 90 characters of the message. That may be too long or too noisy.
- **Who may tag.** The default `allowFrom` is Nat only. Adding teammates is a per-room question.
- **`!here` state is in memory.** A restart or `/mcp` reconnect locks every room. While a room is unlocked,
  every allowed message reaches that bot, so keep the timer short in busy rooms.
- **No token no longer means a failed plugin (fixed in the patch).** Upstream exits when there is no
  `DISCORD_BOT_TOKEN`. Every session without a bot (worktrees, oracles with no bot) then records
  `plugin:discord:discord` as failed in the global `~/.claude/mcp-needs-auth-cache.json`, and for 15
  minutes every new session skips Discord, bots too. The patch makes a token-less session connect as an
  empty server with no tools. Found by Athena.
- **A plugin that fails once stays off for 15 minutes.** Claude Code records the failure in
  `~/.claude/mcp-needs-auth-cache.json` and skips `plugin:discord:discord` until it expires. If a bot is
  silent after a restart, delete that entry (or run `/mcp` and reconnect) before you look elsewhere.
  Found by Athena.
- **Token resets: verify before use.** After a reset, the token read from the page is not always the newest
  one. Check it with `GET /users/@me` (`maw atlas whoami`) before you launch the body.
- **Bots ignore bots** (upstream rule). Oracles cannot tag each other into a thread over Discord. They use
  `maw hey` / herdr for that.
- **Untyped access.** The patch reads `ownerId` and `guildId` off the channel at runtime. It builds with
  `bun build`, but nobody has type-checked it with `tsc`.
