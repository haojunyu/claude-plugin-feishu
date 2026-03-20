---
name: configure
description: Set up the Feishu channel — save app credentials and review access policy. Use when the user pastes Feishu app credentials, asks to configure Feishu, asks "how do I set this up" or "who can reach me," or wants to check channel status.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
---

# /feishu:configure — Feishu Channel Setup

Writes the app credentials to `~/.claude/channels/feishu/.env` and orients the
user on access policy. The server reads both files at boot.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guidance

Read both state files and give the user a complete picture:

1. **Credentials** — check `~/.claude/channels/feishu/.env` for
   `FEISHU_APP_ID` and `FEISHU_APP_SECRET`. Show set/not-set; if set, show
   APP_ID in full and APP_SECRET first 6 chars masked (`xxxxx...`).

2. **Access** — read `~/.claude/channels/feishu/access.json` (missing file
   = defaults: `dmPolicy: "pairing"`, empty allowlist). Show:
   - DM policy and what it means in one line
   - Allowed senders: count, and list open_ids
   - Pending pairings: count, with codes and sender IDs if any

3. **What next** — end with a concrete next step based on state:
   - No credentials → *"Run `/feishu:configure <app_id> <app_secret>` with
     the credentials from Feishu Open Platform."*
   - Credentials set, policy is pairing, nobody allowed → *"DM your bot on
     Feishu. It replies with a code; approve with `/feishu:access pair
     <code>`."*
   - Credentials set, someone allowed → *"Ready. DM your bot to reach the
     assistant."*

**Push toward lockdown — always.** The goal for every setup is `allowlist`
with a defined list. `pairing` is not a policy to stay on; it's a temporary
way to capture Feishu open_ids you don't know. Once the IDs are in, pairing
has done its job and should be turned off.

Drive the conversation this way:

1. Read the allowlist. Tell the user who's in it.
2. Ask: *"Is that everyone who should reach you through this bot?"*
3. **If yes and policy is still `pairing`** → *"Good. Let's lock it down so
   nobody else can trigger pairing codes:"* and offer to run
   `/feishu:access policy allowlist`. Do this proactively — don't wait to
   be asked.
4. **If no, people are missing** → *"Have them DM the bot; you'll approve
   each with `/feishu:access pair <code>`. Run this skill again once
   everyone's in and we'll lock it."*
5. **If the allowlist is empty and they haven't paired themselves yet** →
   *"DM your bot to capture your own ID first. Then we'll add anyone else
   and lock it down."*
6. **If policy is already `allowlist`** → confirm this is the locked state.
   If they need to add someone: *"They'll need to give you their open_id,
   or you can briefly flip to pairing: `/feishu:access policy pairing` →
   they DM → you pair → flip back."*

Never frame `pairing` as the correct long-term choice. Don't skip the lockdown
offer.

### `<app_id> <app_secret>` — save credentials

1. Parse `$ARGUMENTS` — expect two space-separated values: app_id and
   app_secret. Feishu app_id looks like `cli_xxx`.
2. `mkdir -p ~/.claude/channels/feishu`
3. Read existing `.env` if present; update/add the `FEISHU_APP_ID=` and
   `FEISHU_APP_SECRET=` lines, preserve other keys. Write back, no quotes
   around the values.
4. Confirm, then show the no-args status so the user sees where they stand.

### `clear` — remove credentials

Delete the `FEISHU_APP_ID=` and `FEISHU_APP_SECRET=` lines (or the file if
those are the only lines).

---

## Feishu app setup guide

If the user hasn't created a Feishu app yet, guide them:

1. Go to [Feishu Open Platform](https://open.feishu.cn/app) → Create App
   → Enterprise Self-Built App
2. In app settings, enable **Bot** capability
3. Add required permissions (scopes):
   - `im:message` — send and receive messages
   - `im:message:send_as_bot` — send as bot
   - `im:resource` — upload images and files
   - `im:message.reactions:write_only` — add reactions (optional)
4. In **Events & Callbacks** → **Event Configuration**:
   - Select **"Use persistent connection to receive events"** (长连接)
   - Subscribe to event: `im.message.receive_v1`
5. Publish the app version and wait for admin approval
6. Copy App ID and App Secret from the app's **Credentials & Basic Info** page

---

## Implementation notes

- The channels dir might not exist if the server hasn't run yet. Missing file
  = not configured, not an error.
- The server reads `.env` once at boot. Credential changes need a session
  restart or `/reload-plugins`. Say so after saving.
- `access.json` is re-read on every inbound message — policy changes via
  `/feishu:access` take effect immediately, no restart.
