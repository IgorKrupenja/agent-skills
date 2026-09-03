---
name: new-life
description: Personal "New Life" events for the soul (culture/leisure, NOT work). Crawl Igor's bookmarked sources, list candidates in chat, add events to the private "New Life" Google Calendar, and remember what to skip — both single events and whole banned series. Use when Igor asks to show / review "new life" events; when he pastes ANY event link (Facebook, Instagram, tiks.me, Fienta, …) and asks to add it to his personal / private / "личный" calendar — including a bare "добавь в личный календарь <ссылка>"; or when he wants to ban an event series.
---

# New Life — Personal Events Skill

Runs in: **local** (needs the browser + the local Vivaldi bookmarks file).

A dead-simple, personal counterpart to the tallinn.dev event skills — but for **culture & leisure** ("для души"), not IT/work. No Coda, no labels, no publishing. Just:

**crawl bookmarked sources → list in chat → add the picks to a private calendar → remember the rejects.**

Plus a standalone entry point that needs no crawl: **Igor pastes an event link → it lands on the calendar, fully filled in.** Same calendar, same formatting rules, same `state.json` bookkeeping — that's exactly why it lives in this skill and not a separate one.

Everything is private: a personal Google Calendar and a git-ignored `state.json`. Nothing is published anywhere.

## Prerequisites

Always load env first (some values contain spaces, so use `set -a`, not `export $(...)`):

```bash
set -a && source "${SKILLS_DIR:-$HOME/.claude/skills}/.env" && set +a
```

| Variable                    | Meaning                                                              |
| --------------------------- | ------------------------------------------------------------------- |
| `BOOKMARKS_FILE`            | Path to the Chromium/Vivaldi bookmarks JSON                         |
| `NEW_LIFE_BOOKMARKS_FOLDER` | Folder path inside bookmarks, `/`-separated (e.g. `New Life/Events`) |
| `NEW_LIFE_CALENDAR_ID`      | Target Google Calendar ID (the private "New Life" calendar)          |
| `NEW_LIFE_EVENT_COLOR`      | `gog` event color id 1–11 (so events stand out). `6` = Tangerine     |
| `NEW_LIFE_TIMEZONE`         | IANA timezone, e.g. `Europe/Tallinn`                                 |
| `GOOGLE_PLACES_API_KEY`     | Google Places key for `goplaces` (venue name → full address)         |

State file: **`new-life/state.json`** (git-ignored). If missing, create it from `state.example.json`.

The calendar was created once with:
`gog calendar create-calendar "New Life" --timezone "Europe/Tallinn"`
and colored in the sidebar with `gog calendar subscribe "$NEW_LIFE_CALENDAR_ID" --color-id 6`. You don't need to recreate it.

---

## The three things Igor will ask

### A) "Show / review new life events" → crawl

1. **Get today's date first** (avoid year mistakes):
   ```bash
   date +"%Y-%m-%d %A %Z"
   ```

2. **Read source URLs fresh** from the bookmarks folder (the list changes between runs):
   ```bash
   FIRST="${NEW_LIFE_BOOKMARKS_FOLDER%%/*}"   # e.g. "New Life"
   LAST="${NEW_LIFE_BOOKMARKS_FOLDER##*/}"    # e.g. "Events"
   jq -r --arg first "$FIRST" --arg last "$LAST" '
     [.. | objects | select(.type=="folder" and .name==$first)][0]
     | [.. | objects | select(.type=="folder" and .name==$last)][0]
     | .children[]? | select(.type=="url") | "\(.name)\t\(.url)"
   ' "$BOOKMARKS_FILE"
   ```

3. **Crawl every source.** Open each URL in the browser, dismiss cookie/login popups, read the snapshot, and extract event candidates based on what's actually on the page (don't hardcode per-site logic).
   - **Never skip a source** because it's noisy or you "already have enough". Every bookmark is there on purpose. If a page needs login, ask Igor to log in.
   - **Every candidate MUST have a URL.** If you can see a title/date but no link, click into it / read the `href` before moving on.
   - Collect across ALL sources before showing anything:
     ```
     candidate = { title, date, url, source_url, location? }
     ```
   - `source_url` = the bookmark the candidate came from (needed for series bans).

4. **Filter against `state.json`** (see [Filtering](#filtering-what-not-to-show)). Drop:
   - anything already in `added` or `declined` (by URL), and
   - anything matching a `banned_series` entry.

5. **Present ALL survivors** as one numbered table in chat:

   | # | Date | Title | Where | Source | Link |
   |---|------|-------|-------|--------|------|

   Then **report what was filtered** — never drop silently:
   > Скрыл 4: 2 уже добавлены, 1 ранее отклонён, 1 из забаненной серии «Open Mic».

   Criteria for now: **show everything** (no taste filtering yet). This will be refined over time.

### B) Put an event on the calendar

**Two entry points, one pipeline:**

- **B1 — from the crawl table:** "добавь 1, 3, 5" → you already have the candidate's URL from step A.
- **B2 — from a bare link:** "добавь в личный календарь `<URL>`" (any event link: Facebook, Instagram, tiks.me, Fienta, …). No crawl needed, no `state.json` filtering — Igor already decided. Go straight to step 1.

Both paths run the **same 8 steps** below. Never shortcut B2 just because it's a one-off.

#### The four mandatory fields

An event is **not** ready to create until all four are filled. None of them may be silently left blank:

| Field | Rule if you can't find it |
| ----- | ------------------------- |
| **Date + time** | Never guess the year — see step 0. If only a start time is given, assume **2 hours** and say so in your report. |
| **Location** | Dig: FB events put the venue in a separate block, not in the body text. Online event → `Online`. **Distributed events** (yard-sale days, city-wide festivals, tours across a district) legitimately have no single address — use the district as given (`Nõmme, Tallinn, Estonia`) and keep any route/map link in the description rather than forcing a fake street address. |
| **Price** | Look hard (see step 3), then **default to `0€`. Never ask Igor about price** — just note in your report that it wasn't stated. |
| **Language** | Language the event is actually held in. If not stated, infer from the description text. Bilingual → `EE / ENG`. |
| **Booking** | Does *attending* require booking ahead? See step 3c. Default when nothing indicates it: **no booking**. |
| **Full description** | Complete original text, never summarized or translated. |

#### Steps

**0. Get today's date first** — avoids year errors on "15 сентября" with no year:
```bash
date +"%Y-%m-%d %A %Z"
```

**1. Normalize the URL.** Share links (`facebook.com/share/…`, `?rdid=…`, `&mibextid=…`, `/events/s/…`) are redirect wrappers. Open the link, then take the **canonical** URL from the address bar — for Facebook that's `https://www.facebook.com/events/<id>/`. Store and display the canonical one; the wrapper rots and is unreadable.

**Two link shapes that are NOT wrappers — never strip them:**
- **FB recurring events** use `facebook.com/events/<series_id>/<occurrence_id>/`. The second id *is* the date Igor picked. Dropping it lands on the series and silently changes which occurrence you add. Keep both segments. (The page shows the sibling dates as a row of chips — useful for confirming which one is live.)
- **Fienta series** live at `fienta.com/[ru/]s/<slug>` and list several dates, each its own page at `fienta.com/[ru/]<slug>-<id>`. A series link is **not** an event: open it, pick the date, and check availability — sold-out dates are marked (`Распродано` / `Sold out`) and must not be added. If several dates are open and Igor didn't say which, add the nearest available one and tell him the others exist.

**2. Extract full content with Playwright.** Open the page in the browser, dismiss cookie/login walls, read the snapshot.
- **Facebook: always click "See more"** before extracting — FB truncates the body by default.
- **Facebook mangles links inside the description — twice.** The visible text is cut with `…` (`https://open.spotify.com/artist/4fJ6…`) and the `href` is a `l.facebook.com/l.php?u=<percent-encoded>&fbclid=…` redirect. Neither is usable. Pull the real URL from each `<a>` and decode it:
  ```js
  [...node.querySelectorAll('a')].map(a => {
    const u = new URL(a.href);
    const real = u.hostname.endsWith('facebook.com') && u.searchParams.get('u')
      ? new URL(decodeURIComponent(u.searchParams.get('u'))) : new URL(a.href);
    ['fbclid','__cft__[0]','__tn__','si','utm_id'].forEach(p => real.searchParams.delete(p));
    return { shown: a.innerText.trim(), real: real.href };
  })
  ```
  Substitute the decoded URLs back into the description text. This is restoring what the author wrote, not editing it.
- **Content policy:** do **not** summarize, do **not** translate, keep original line breaks, emphasis and emojis (incl. math-bold like `𝐌𝐔𝐔𝐒𝐈𝐊𝐀`). This is a private calendar — the full original text is the whole point.
- Drop only FB's own UI chrome that lands in `innerText`: the trailing `See less` and the city tag link after it.
- **Quoting:** write the assembled description to a temp file and pass `--description "$(cat file)"`. Inlining multi-line text with emoji into the shell mangles it.

**3. Find the price.** Check, in this order:
   1. the ticket/price block FB and ticketing sites render **outside** the description text;
   2. the body text — `10€`, `10 EUR`, `tasuta`, `free`, `vaba sissepääs`, `annetus` / donation, `at the door`, `eelmüük`;
   3. the ticket-vendor link (Fienta / tiks.me / Piletilevi) — open it if the price isn't on the event page itself.

   Then normalize to a **numeric-first** form:
   - Paid → `15€`.
   - **Choice-based tiers** (anyone may pick — pay-what-you-want, early bird vs door) → keep the range: `1–10€`, `12€ / 15€ kohapeal`.
   - **Eligibility-based tiers** (student, child, pensioner) → use the **regular adult price only**: a tour at `18€ regular / 12€ student / 0€ kids` is `18€`, never `0–18€`. Igor pays the adult price; folding in discounts he can't claim makes the number a lie.
   - **Quantity bundles** (`Двойной билет 18€`, group of 4) → also the single-ticket price: `10€`, not `10–18€`. A bundle is more tickets, not a cheaper option — and Igor is usually going alone.
   - Free (`tasuta`, `vaba sissepääs`, `free entry`) → **`0€`**.
   - Donation-based → `annetus`.
   - **Nothing found after all three → `0€`.** Do NOT ask Igor about price, ever — most of what he adds is free-entry anyway. Just say in your report that the page didn't state it, so he can spot a wrong guess.

**3b. Determine the language.** What language the event is actually held in — matters most for talks, screenings and discussions. If the page states it, use that; otherwise infer from the description text. Short codes: `EE`, `RU`, `ENG`. Bilingual/multilingual → `EE / ENG`. When you inferred rather than read it, say so in your report.

> ⚠️ **Do not trust Fienta's `inLanguage` field.** It reflects the *page locale* (from the `/et/`, `/ru/` URL segment), not the language the event is held in. A Russian-language stand-up club listed at `fienta.com/et/…` reports `inLanguage: "et"` while its whole description — and its audience — is Russian. Always cross-check against the description text and the organizer's own profile; the description wins.

**3c. Decide whether booking is required.** The question is strictly: **must Igor do something in advance in order to attend?** If yes → the title gets a trailing `[BOOK]`. If no → nothing is added.

Signals that it IS required:
- a ticket / registration CTA on the event itself (`Get Tickets`, `Register`, `Osta pilet`), or a link to Fienta / Piletilevi / Eventbrite / Luma / a registration form;
- the body says `registreeru`, `eelregistreerimine`, `kohtade arv on piiratud`, `limited seats`, `RSVP`, `sign up`, `регистрация`, `запись обязательна`.

**Two false positives that show up constantly — neither means `[BOOK]`:**
1. **Vendor / performer booking.** Markets and fairs routinely say `Müügikoha broneerimine` (booking a *stall*), `Kui soovid tulla kodukohvikut pidama, kirjuta…`, `apply to play`. That's for people who want to *sell or perform*, not to visit. Visitors walk in freely.
2. **Host-page ticket buttons.** In FB's "Meet your hosts" block, each host Page can carry its own `Get tickets` CTA. That belongs to the Page, not to this event. Only a ticket/registration element attached to the event itself counts.
3. **Registration for a sub-activity.** A free fair can contain one thing that needs signing up — a kids' fun run, a workshop slot, a tournament. `Registreerimine on vajalik` scoped to that sub-activity does not make the event itself booked; Igor can still just walk in.

When in doubt, ask yourself who the instruction is addressed to — the audience, or the people running a table. Only the former counts. Default is **no booking**.

**4. Resolve the location** to a full street address:
```bash
goplaces search "Venue Name, Tallinn" --api-key "$GOOGLE_PLACES_API_KEY" --json
```
Use the `address` field. Sanity-check the `name` in the result actually matches the venue — Places happily returns a plausible wrong bar. Also glance at `business_status`: `CLOSED_PERMANENTLY` on a venue hosting a future event means you resolved the wrong place.

**5. Check for duplicates** before creating:
```bash
gog calendar events "$NEW_LIFE_CALENDAR_ID" --from 2026-09-01 --to 2026-09-02 --all-pages --json
```
Compare start time (few-minutes tolerance) + title. Also check the URL against `state.json` `added`. If it's a duplicate → report it and **stop**, don't create a second copy.

> ⚠️ **`gog calendar events` paginates at 10 by default** and just returns a `nextPageToken` — the missing events are invisible unless you look for it. A multi-day range silently truncates, so a duplicate check over a busy week can report "nothing there" while the duplicate sits on page 2. Always pass **`--all-pages`** (or `--max`), and treat any listing without it as unreliable.

**6. Create the event.**

**Title format:** `Event Title · <price>` plus a trailing ` [BOOK]` only when booking is required.

```
PLURRR · 0€                      ← free, just show up
Kontsert · 10–15€                ← paid, no booking
Keelekohvik · 0€ [BOOK]          ← free but you must register
Workshop · 25€ [BOOK]            ← paid and booked
```

Always a middot `·`, never an em dash; always a numeric price, never the word `tasuta`. `[BOOK]` is uppercase, in square brackets, always last — and **absent entirely** when booking isn't needed (no `[NO BOOK]`, no empty brackets).

**Description format — three header lines, then a blank line, then the full original text:**

```
<LANGUAGE>      ← line 1: EE / RU / ENG / EE / ENG
<PRICE>         ← line 2: 0€ / 15€ / annetus
<CANONICAL_URL> ← line 3
                ← blank
<FULL ORIGINAL DESCRIPTION>
```

```bash
gog calendar create "$NEW_LIFE_CALENDAR_ID" \
  --summary "Event Title · 15€" \
  --from "YYYY-MM-DDTHH:MM:SS" \
  --to   "YYYY-MM-DDTHH:MM:SS" \
  --timezone "$NEW_LIFE_TIMEZONE" \
  --location "Telliskivi tn 62, 10412 Tallinn, Estonia" \
  --event-color "$NEW_LIFE_EVENT_COLOR" \
  --source-url "<CANONICAL_EVENT_URL>" \
  --description "EE
15€
<CANONICAL_EVENT_URL>

<FULL ORIGINAL DESCRIPTION>"
```
- Times: local `YYYY-MM-DDTHH:MM:SS` + `--timezone` (Google applies DST correctly). For all-day: `--all-day --from YYYY-MM-DD --to YYYY-MM-DD` (end = next day).
- `--location` takes the **resolved address** from step 4. (`--location-search "Venue, City"` also works and resolves internally, but resolving yourself lets you verify the match first.)
- Add `-n/--dry-run` to inspect the exact payload before writing anything. Note it prints a `Dry run: would calendar.create` line **before** the JSON — strip it (`tail -n +2`) before piping to `jq`.
- `--json` wraps the created event in an `event` key: read `.event.htmlLink`, not `.htmlLink`. Top-level `jq` on it silently yields `null`, which looks like a failed create when it actually succeeded.

**7. Record it** in `state.json` `added` — for B2 links too, otherwise the crawler will re-suggest the event next week. (Schema below.)

**8. Report back**: title, date/time, resolved address, price, and the event's `htmlLink`. Flag any assumption you made (guessed 2h duration, ambiguous venue match, price taken from the ticket vendor rather than the event page).

### C) "Not interested in 2, 4" / "Ban series 6" → remember the skip

- **Single event** ("не интересно", "skip"): append to `declined`.
- **Whole series** ("забань серию", "больше не предлагай такое"): append to `banned_series`. The match key is the event's **normalized title** (see below), scoped to its `source_url`. Igor can also give a custom phrase ("забань всё с 'карнавал осьминогов'") — normalize that phrase instead.
- After editing, confirm in one line what will now be hidden.

---

## Filtering: what NOT to show

`state.json` is the memory of rejects. Use this normalization for both **creating** a ban key and **matching** candidates, so they line up:

```python
import re
def normalize(t):
    t = (t or "").lower()
    t = re.sub(r'[#№]', ' ', t)
    t = re.sub(r'\d+', ' ', t)               # drop numbers: vol 12, years, dates
    t = re.sub(r'[^\w\s]', ' ', t, re.UNICODE)  # drop punctuation/emoji, keep RU/ET/EN words
    return re.sub(r'\s+', ' ', t).strip()
```

A candidate is **hidden** when any of these is true:
- its `url` is in `added` (by url) or `declined` (by url);
- there is a `banned_series` entry `b` where `b.source` is `"*"` **or** equals the candidate's `source_url`, **and** `b.match` is contained in `normalize(candidate.title)`.

Ready-to-run filter (write the crawled candidates to a temp JSON array first):

```python
import json, re, sys
def normalize(t):
    t=(t or "").lower(); t=re.sub(r'[#№]',' ',t); t=re.sub(r'\d+',' ',t)
    t=re.sub(r'[^\w\s]',' ',t,flags=re.UNICODE); return re.sub(r'\s+',' ',t).strip()

state = json.load(open(sys.argv[1]))            # state.json
cands = json.load(open(sys.argv[2]))            # [{title,date,url,source_url,location}]
seen  = {e["url"] for e in state["added"]} | {e["url"] for e in state["declined"]}
bans  = state["banned_series"]

show, hidden = [], []
for c in cands:
    if c["url"] in seen:
        hidden.append((c, "already added/declined")); continue
    nt = normalize(c["title"])
    hit = next((b for b in bans
                if (b["source"] in ("*", c.get("source_url"))) and b["match"] in nt), None)
    if hit:
        hidden.append((c, f"banned series «{hit['label']}»")); continue
    show.append(c)

print(f"SHOW {len(show)} / HIDE {len(hidden)}")
for c,why in hidden: print("  hidden:", c["title"], "—", why)
print(json.dumps(show, ensure_ascii=False, indent=2))
```

---

## `state.json` schema

```jsonc
{
  "added":    [ { "url": "...", "title": "...", "date": "2026-06-29", "price": "0€", "lang": "EE", "book": false, "added_at": "2026-06-27" } ],
  "declined": [ { "url": "...", "title": "...", "declined_at": "2026-06-27" } ],
  "banned_series": [
    {
      "match":  "open mic",                          // normalized phrase to match in titles
      "label":  "Open Mic @ Erinevate Tubade Klubi", // human-readable, for reports
      "source": "https://www.facebook.com/erinevatetubadeklubi/", // bookmark scope, or "*" for global
      "banned_at": "2026-06-27"
    }
  ]
}
```

Use `date +%F` for the `*_at` stamps. Edit the file directly (read → modify JSON → write); keep it valid JSON.

---

## Quality checklist

**Crawling (A):**
- ✅ Sourced env; ran `date` first
- ✅ Read bookmarks **fresh**; crawled **every** source; no source skipped
- ✅ Every candidate has a URL
- ✅ Filtered against `added` + `declined` + `banned_series`
- ✅ Reported how many were hidden and why (no silent drops)

**Adding (B) — applies to both the crawl picks and a pasted link:**
- ✅ Canonical URL (FB share/`rdid` wrapper resolved to `/events/<id>/`)
- ✅ All six mandatory fields present: date+time, location, **price**, **language**, **booking**, full description
- ✅ Full original text — no summary, no translation, FB "See more" expanded
- ✅ Location resolved via `goplaces`; result name actually matches the venue
- ✅ Duplicate check done (calendar for that day + `state.json` `added`)
- ✅ Title is ` · <price>` (numeric, `0€` not `tasuta`), with ` [BOOK]` only if attending needs booking
- ✅ Description header is language / price / URL, then a blank line, then the full text
- ✅ Created with `--event-color "$NEW_LIFE_EVENT_COLOR"` and `--timezone "$NEW_LIFE_TIMEZONE"`
- ✅ Recorded adds/declines/bans back into `state.json`
- ✅ Reported `htmlLink` + every assumption made

## Pitfalls

- ❌ Skipping a source because there are "enough" candidates already
- ❌ A candidate with no URL
- ❌ **Asking Igor what the price is** — he explicitly doesn't want the question; unfindable means `0€`
- ❌ Writing `tasuta` / `free` in the title instead of `0€`, or joining with `—` instead of ` · `
- ❌ Forgetting the language line — it's line 1 of every description
- ❌ **Tagging `[BOOK]` off vendor-side booking** (`Müügikoha broneerimine`) or a host Page's own `Get tickets` button — neither applies to attending
- ❌ Adding `[NO BOOK]` or empty brackets when no booking is needed — the tag is simply absent
- ❌ **Leaving `--location` empty** because the body text didn't mention a venue — FB keeps it in a separate block
- ❌ Saving the FB share wrapper (`/share/…?mibextid=…`) instead of the canonical `/events/<id>/` URL
- ❌ Treating a pasted link (B2) as a shortcut — it runs the same 8 steps
- ❌ Summarizing/translating the description, or forgetting FB "See more"
- ❌ Date-only `--from/--to` for a timed event (use `THH:MM:SS` + `--timezone`)
- ❌ Forgetting to record an add/decline/ban → it gets suggested again
- ❌ Committing `state.json` (it's git-ignored — keep it that way)

> Note: the sibling `events-add` skill warns that date-only `--from/--to` breaks `gog calendar events`. That was true for an older `gog`; on v0.30.0 both date-only and RFC3339 work for listing. Date-only in `create` still needs `--all-day`.
