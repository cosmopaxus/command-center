# Command Center sync (GitHub)

Command Center keeps its data in one JSON file in a **private** GitHub repo, so your phone, your Mac and your bots all see the same list.

- App (public, code only): `https://github.com/cosmopaxus/command-center`
- Data (private): `https://github.com/cosmopaxus/command-center-data`, file `data/command-center.json`
- Format: `command-center/v1` (same file the app's Export JSON makes)

Nothing secret lives in the public repo. The token lives only in the browser of the device you pasted it into.

---

## 1. One-time setup for you (about 2 minutes)

You need **one fine-grained token**, pasted once per device (phone, Mac).

1. GitHub → your profile picture → **Settings**.
2. Left sidebar, bottom → **Developer settings**.
3. **Personal access tokens → Fine-grained tokens → Generate new token**.
4. Token name: `command-center` (anything).
5. Expiration: **1 year** (or a custom date). GitHub will email you before it expires.
6. Repository access: **Only select repositories** → choose **`command-center-data`** and nothing else.
7. Permissions → **Repository permissions** → **Contents: Read and write**. (Metadata: Read-only is added automatically. Leave everything else at No access.)
8. **Generate token**. **Copy it now**: GitHub shows it once.
9. Open Command Center → **Settings → Sync**. Leave owner `cosmopaxus` and repo `command-center-data`. Paste the token. Tap **Test connection** (should say Connected), then **Save & connect**.
10. Do the same on your other device. Each device needs its own paste (the token is not stored in the repo or the app).

The chip at the top shows the state: **Synced**, **Saving…**, **Offline**, **Sync error** (tap it for the details), or **Local only** (no token yet).

If a token leaks or you lose a phone: GitHub → Settings → Developer settings → Fine-grained tokens → **Delete** it, make a new one, paste it again. The token can only touch `command-center-data`, nothing else.

### What the app does for you
- Saves to the device first (works offline), then pushes to GitHub about 2 seconds after the last change.
- Pulls when you open the app, when you come back to the tab, and every minute while it is open.
- If your phone and a bot both changed the file, it merges **by item id**. For the same item the **newer `updated_at` wins**. Items added on either side are kept. Nothing is silently dropped. An item you deleted stays deleted unless someone edited it after.
- Refuses to sync to a **public** repo (it checks).
- Never mixes lanes. Each item keeps its own `lane` id (Evoke and Evolve stay separate).

---

## 2. For bots (Bottley, Chief of Staff, Evoke Ops and so on)

Use the GitHub connector with the account that has access to `cosmopaxus/command-center-data`. No Supabase, no extra service.

### Read
`get_file_contents` → owner `cosmopaxus`, repo `command-center-data`, path `data/command-center.json`. Keep the returned **sha**.

### Write (one safe way)
1. Read the file and note its `sha`.
2. Change only what you need in the JSON.
3. `create_or_update_file` with the **same sha**. If GitHub answers 409/422 (someone else wrote first), read again and redo your change on the new copy. Do not force it.
4. Keep the whole file valid JSON, `"format": "command-center/v1"`.

**Rules for every row you add or change**
- Set `updated_at` to the current time as an ISO string (`2026-10-26T21:55:00.000Z`). The merge uses it. If you forget, your edit can lose to an older copy.
- New rows need a unique `id` (for example `it_` + random) and `created_at`.
- Keep `lane` pointing at an existing lane id. Do not move items between Evoke and Evolve lanes unless Cosmo asked.
- Do not put tokens, passwords or card numbers in this file. No dosing copy.
- Do not delete rows you did not create. Set `status` to `done` or `discarded` instead.

### Item shape (`items[]`)
```json
{
  "id": "it_abc123",
  "owner": "cosmo",
  "visibility": "personal",
  "created_at": "2026-10-26T21:50:00.000Z",
  "updated_at": "2026-10-26T21:55:00.000Z",
  "title": "Short title",
  "lane": "evoke",
  "kind": "task",
  "status": "open",
  "due": "2026-10-30",
  "bot": null,
  "note": "",
  "link": null,
  "pinned": false,
  "pinned_at": null,
  "since": null,
  "inbox": false
}
```
- `kind`: `task` or `decision`.
- `status`: `open`, `in_process`, `scheduled`, `blocked`, `deadline`, `waiting`, `waiting_you`, `done`, `discarded`, `note`.
- `due`: `YYYY-MM-DD` or `null`.
- `bot`: a bot id from `bots[]` or `null`.
- `inbox`: `true` puts it in the lane's inbox until Cosmo sorts it.
- `waiting` and `waiting_you` items use `since` (`YYYY-MM-DD`) to count days waiting.

### Update an item
Find it by `id`, change fields, set `updated_at` to now, write the file as above.

### Add an item
Append an object to `items[]` with a new `id`, a real `lane`, `created_at` and `updated_at` both now.

### Send Cosmo a message through the outbox (Bottley relays it)
Append a row to `outbox[]`. Bottley (or Cosmo) reads pending rows and relays them. Mark one sent by setting `status` to `sent`, `sent_at` to now, and `updated_at` to now.
```json
{
  "id": "ob_k3j9x2",
  "owner": "cosmo",
  "visibility": "personal",
  "created_at": "2026-10-26T21:55:00.000Z",
  "updated_at": "2026-10-26T21:55:00.000Z",
  "to_bot": "bot_evoke_ops",
  "to_bot_name": "Evoke Ops",
  "lane": "evoke",
  "text": "What the bot should do, in plain words.",
  "status": "pending",
  "sent_at": null
}
```
- `to_bot` / `to_bot_name` must match an entry in `bots[]`. Use the bot's `id` and `name`.
- `status`: `pending` or `sent`.

### Bot status cards (`bots[]`)
A bot can update its own card: `last_activity_at`, `last_note`, and `snapshot[]` (a few `{status, text, date, link, item_id}` lines). Set `updated_at` too.

### Top-level shape
```
{ "format": "command-center/v1", "meta": {...}, "settings": {"theme": "dark"},
  "lanes": [...], "priorities": {...}, "bots": [...], "items": [...], "outbox": [...], "notes": [...] }
```
Do not rename or remove top-level keys. `meta.sample: true` marks example data; the app clears that flag when Cosmo clears the examples.

---

## 3. Moving servers
The app is one static file, `index.html`. Copy it to any static host (Cloudflare Pages, Netlify, your own server). Sync still talks to GitHub directly from the browser, so nothing else moves. To change where data lives, change owner/repo in Settings → Sync, or swap the adapter in `index.html` (see README.md).

## 4. Honest limits
- The token is in the browser's localStorage on that device. Anyone who can use that browser profile can use it. That is why it is scoped to one private repo with Contents only.
- There is no live push. Other devices see changes within about a minute, or when you open the app / tap the chip.
- GitHub API rate limit is 5,000 requests per hour per token. The app uses about 1 per minute while open.
