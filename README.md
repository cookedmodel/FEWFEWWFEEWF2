# gmail_allias

Self-hosted service: add a Gmail account once, spin up as many aliases as you
want from it (`+tag` and dot variants — same inbox, no server-side creation
needed), then read and parse the mail those aliases receive through a REST API.
Includes per-alias filtering and automatic extraction of verification codes and
confirmation links.

## What it does

- **Add account** — stores a Gmail App Password, encrypted at rest (Fernet).
- **Generate aliases** — `+tag` (readable / random / per-service) or dot variants.
- **Read mail** — list & fetch messages over IMAP, parsed to clean JSON.
- **Filter by alias** — uses Gmail's `X-GM-RAW deliveredto:` so you see exactly
  what landed on `you+netflix@gmail.com`, even when the visible `To:` differs.
- **Grab the code** — `GET /code` returns the newest verification code + links.

## Setup

Gmail dropped "less secure apps", so IMAP needs an **App Password**:

1. Enable 2-Step Verification on the account.
2. Go to Google Account → Security → App passwords → create one (16 chars).
3. Use that value as `app_password` (spaces are stripped automatically).

```bash
pip install -r requirements.txt
cp .env.example .env        # optional: set API_TOKEN, etc.
python run.py               # -> http://127.0.0.1:8000  (docs at /docs)
```

Set `API_TOKEN` in `.env` to require `Authorization: Bearer <token>` on every
route. Leave it empty for open localhost use.

## API

| Method | Path | Purpose |
|--------|------|---------|
| POST   | `/accounts` | add/update account (verifies IMAP login) |
| POST   | `/accounts/bulk` | paste-import `email:password` lines (optional verify) |
| GET    | `/accounts` | list accounts (`?domain=`, `?status=`) |
| DELETE | `/accounts/{email}` | remove account |
| POST   | `/accounts/{email}/check` | IMAP-verify one account now, update health |
| GET    | `/stats` | stock report: totals + per-domain counts (`?domain=`) |
| POST   | `/accounts/{email}/aliases/generate` | make aliases |
| GET    | `/accounts/{email}/aliases` | list stored aliases (`?service=`) |
| PATCH  | `/accounts/{email}/aliases/{alias}` | tag alias with `service`/`note` |
| GET    | `/accounts/{email}/messages` | list mail (`?alias=`, `?unread=`, `?since=`, `?search=`, `?limit=`) |
| GET    | `/accounts/{email}/messages/{uid}` | full parsed message + body |
| GET    | `/accounts/{email}/code` | newest verification code (`?alias=`) |

## Stock / monitoring (v1.1)

- **Bulk import** — `POST /accounts/bulk` with
  `{"text":"a@gmail.com: pass1\nb@gmail.com;pass2","verify":true}`.
  Separators `:`, `;` or space; spaces inside App Passwords are stripped;
  every line is IMAP-verified and reported back per-line.
- **Health monitor** — a background thread IMAP-verifies every account every
  `CHECK_INTERVAL` seconds (default 900, `0` disables) and stores
  `status` (`active` / `invalid` / `unknown`) + `last_checked` per account.
- **Stock report** — `GET /stats` returns totals and a per-domain breakdown
  (`total` / `active` / `invalid` / `unknown` / `aliases_issued`);
  `GET /stats?domain=gmail.com` filters to one domain.

## Server deploy

Live at `http://5.22.209.163:443` (Debian 13, `/opt/gmailer`, venv + systemd):

- `gmailer-api.service` — uvicorn on `0.0.0.0:443` (port 8000 is DPI-throttled
  on this route, 443 passes clean).
- `gmailer-bot.service` — the Telegram bot (token in `.env`).
- Manage: `systemctl restart gmailer-api` / `gmailer-bot`, logs via
  `journalctl -u gmailer-api -f`.
- Auth: `Authorization: Bearer $API_TOKEN` from `/opt/gmailer/.env`.

### Examples

Add the account:

```bash
curl -X POST localhost:8000/accounts \
  -H 'Content-Type: application/json' \
  -d '{"email":"yourname@gmail.com","app_password":"abcd efgh ijkl mnop","label":"main"}'
```

Generate 20 per-service aliases:

```bash
curl -X POST localhost:8000/accounts/yourname@gmail.com/aliases/generate \
  -H 'Content-Type: application/json' \
  -d '{"count":20,"kind":"plus","style":"service","service":"reddit"}'
```

List mail that arrived on one alias:

```bash
curl 'localhost:8000/accounts/yourname@gmail.com/messages?alias=yourname+reddit-ab12@gmail.com&limit=10'
```

Pull the latest verification code for a signup:

```bash
curl 'localhost:8000/accounts/yourname@gmail.com/code?alias=yourname+reddit-ab12@gmail.com'
```

## Alias styles

- `kind=plus, style=word` → `you+swiftraven42@gmail.com`
- `kind=plus, style=rand` → `you+k7f9q2ab@gmail.com`
- `kind=plus, style=service` (with `service`) → `you+reddit-ab12@gmail.com`
- `kind=dot` → `y.o.u@gmail.com` (Gmail ignores dots)

## Layout

```
app/
  config.py    paths, env, IMAP endpoint
  crypto.py    Fernet encrypt/decrypt for app-passwords
  aliases.py   +tag / dot generation, canonicalisation
  db.py        SQLite: accounts + aliases
  parser.py    RFC822 -> JSON, code/link extraction
  gmail.py     IMAP client, alias-aware search (X-GM-RAW)
  models.py    pydantic schemas
  main.py      FastAPI routes
run.py         uvicorn entry point
```

## Telegram bot

A multi-user aiogram 3 front-end lives in `bot/`. Users pick a site, get a
fresh alias, and the bot polls that alias's mailbox in the background and edits
the order card in place the moment a message lands — pulling out the code or
link and offering the full parsed JSON as a file.

- **Multi-user + isolated** — every order is tied to a `user_id`; card actions
  (open message / open JSON) verify ownership, so no one sees another user's mail.
- **True concurrency** — blocking IMAP checks run on worker threads
  (`asyncio.to_thread`) behind a semaphore, so many orders are polled at once
  without stalling the event loop.
- **Live card** — the order message edits itself: waiting → received, code/link
  filled in, JSON button appears.
- **Admin panel** (`/admin`, ids in `ADMIN_IDS`) — add Gmail mailboxes
  (email + App Password, verified on add), view stats, remaining capacity,
  per-account usage, and broadcast to all users.
- **Premium emoji** — all text uses `<tg-emoji>` custom emoji; all buttons carry
  `icon_custom_emoji_id` (no inline unicode in labels).

Run it:

```bash
cp .env.example .env   # set BOT_TOKEN and ADMIN_IDS
python run_bot.py
```

Bot layout:

```
bot/
  config.py      env (token, admins, poll interval, budget, price)
  emojis.py      premium custom-emoji registry + pe() helper
  storage.py     users / orders / budgets (shares the core SQLite file)
  keyboards.py   reply + inline keyboards (premium icons only)
  texts.py       HTML message + order-card builders
  services.py    order lifecycle, threaded IMAP poller, card updates
  states.py      FSM states
  handlers/      user.py (buy/card) + admin.py (panel)
  main.py        bot wiring + background poller
run_bot.py       entry point
```

## Alias modes (bot Settings → ⚙️ Настройки)

Sites that dedupe by canonical Gmail address (Wallapop, most big platforms)
strip both `+tags` and dots, so those aliases get rejected as "already used".
The bot offers three modes; pick per how strict the target is:

| Mode | Address | Accepted by | Isolation |
|------|---------|-------------|-----------|
| `plus`   | `you+tag@gmail.com` | lenient sites | `deliveredto:` (exact) |
| `dot`    | `y.o.u@gmail.com`   | more sites, but canonicalisers still block | To-header match |
| `domain` | `anything@yourdomain.com` | **everywhere** — truly distinct addresses | To-header match |

### Catch-all domain setup (the mode that always works)

1. Point a domain's email at a catch-all that forwards into one inbox you added
   to the pool. Free option — **Cloudflare Email Routing**:
   - Add the domain to Cloudflare → **Email → Email Routing → Enable**.
   - Verify a destination address (e.g. `bs.toke075@gmail.com`).
   - **Catch-all → Send to → that address → Save.**
2. In the bot: `Админ-панель → ⚙️ Настройки → Домен catch-all`, then
   **Задать домен** (`yourdomain.com`) and **Задать инбокс** (the pool mailbox
   the domain forwards into).

Now every order mints `random@yourdomain.com`; the mail forwards into the inbox
with the domain address preserved in the `To` header, and the bot matches it
exactly — so addresses are unlimited, unique, accepted by any site, and still
isolated per user.

## Notes

- App passwords are encrypted with a key from `APP_KEY`, or auto-generated to
  `data/.app_key` on first run. Back that key up — losing it means re-adding
  accounts.
- IMAP access is read-only in this build (messages are fetched with
  `BODY.PEEK`, so nothing is marked as read).
- Sending *from* an alias (Gmail "Send mail as") is a separate flow — say the
  word and it can be added on top of this via the Gmail API.
```
