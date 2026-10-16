# Frontend review — config-driven sports (2026-10-17)

Built strictly against `hitnrun-api/API_CONTRACT.md` (read-only, not edited).
**Not committed. Not pushed.** Babel Standalone pin `7.23.10` unchanged in both files.

```
 admin/index.html | 1020 ++++++++++++++++++++++++++++++++++++++++++------------
 index.html       |  428 +++++++++--------------
 2 files changed, 961 insertions(+), 487 deletions(-)
```
(`git diff --stat`; this file is new and untracked.)

## Checks run

| Check | Result |
|---|---|
| Babel transpile, `@babel/standalone@7.23.10`, presets `env`+`react`, both files | ✅ both OK |
| Pin `@babel/standalone@7.23.10/babel.min.js` present | ✅ 1× in each file |
| Leftover hardcoded sports (`SPORTS`, `SPORTS_TURF`, `sportColor`, `net1..3`, `/3 free`, `₹1,200`, `₹400`, sport names) | ✅ none in either file |
| Customer site in Chrome against **live** API (read-only: `getSports`, `getAvailability`) | ✅ 6 sports incl. 🎯 Bowling Machine Practice "₹650 / hr · 20 overs", slots "2 left", duration capped, 6 price tiles, merged Bookings row |
| Customer site with first `getSports` forced to fail | ✅ retry state in sports step + pricing, hero shows "—"; Retry recovers. No stale list. |
| Admin in Chrome against a **mocked** `fetch` (scratchpad copy only, no admin key, no real writes) | ✅ every tab, drawer, Spot/Edit modals, Sports tab flows (see below) |
| Real bookings created | **None** |

The admin browser test found and fixed one runtime crash I had introduced
(`BookingDetailDrawer` used the sport lookup before declaring it).

## Customer site (`index.html`)

- **Sports from `getSports`**, rendered in server order (= `sort_order`). Card: icon, name, tagline,
  `₹{price} / hr · {unit_label}`. Inactive/hidden/archived sports never arrive, so never render.
- **Failure state**: if `getSports` fails → "Couldn't load sports · Retry" in step 2 and in the pricing
  section; hero price shows "—". No fallback list exists anywhere.
- **Availability via `getAvailability`** (one call per date covers every sport; per-date promise cache,
  revisits refresh in background, next 2 days prefetched). Slot cards show "N left"
  (now also visible on mobile). Customer picks a **time**; we send that hour's `slot_id` and the server
  assigns the lane.
- **Multi-hour kept**, options `1…max_slots_per_booking`, disabled beyond the hour's `max_duration`.
- **Prices from the server**: slots show the server's hourly price; duration cards and the button show a
  display-only total = server hourly price × hours ("Lock in 2 hrs · ₹1,300"). The ticket shows the
  server's `amount` only (old browser-computed fallback removed).
- `createBooking` now sends `sport_id`. On `SLOT_TAKEN` / `SPORT_FULL` the error is shown verbatim and
  availability refreshes.
- Ticket, calendar link and share text use `sport_name` + icon (never `sport_id`).
- Returning-customer banner: `lookupCustomer.last_sport` is now a sport_id → mapped to the public name,
  omitted if not public.
- Hero: "Slots open today" = hours today where any sport has `available > 0` (was a net-practice
  heuristic); "From ₹{lowest price} / hour"; generic copy. Meta/OG description generic.
- Features card "3 nets, 1 turf" → "Pick your game". PricingCallout = one tile per sport from config.
- Venue: Net-practice + Badminton phone rows merged into one **Bookings** row.
- Deleted dead code: `SPORTS`, `PASS_PLANS`, `PassCallout` (+ its CSS), `ConfigError` + placeholder checks,
  unused `.price-tile-list` CSS.

## Admin (`admin/index.html`)

### Config everywhere
- On login loads `listSportsAdmin` + `listResourcesAdmin` into a `ConfigCtx`. `cfg.sport(id)` returns
  name/icon/colour (archived included → **old bookings of archived sports keep their name + icon**);
  unknown ids fall back to the raw stored value. `cfg.resourceLabel(id)` gives "Net 2" etc.
- **Spot booking**: sport picker from active (incl. walk-in-only) sports with price; slots via
  `getSlots` + adminKey (see note); "N left"; hours `1…max_slots_per_booking`; Amount optional —
  blank lets the server compute; sends `sport_id`. Hardcoded kind toggle/₹ removed.
- **EditBookingModal**: can change sport (`sport_id` sent only if changed); booking's own sport stays
  selectable even if off/archived; slot list per sport; hours up to `max(max_slots_per_booking, current)`;
  Amount optional (blank = server rule); hardcoded `max_duration: 3` and amount maths removed.
- **Filters**: sport chips from config (+ archived sports that still have bookings); filter by sport_id.
- **Stats**: new "today by sport" chip row from `stats.sports_today` (click → filtered list).
- **Print**: sets `data-print-date` (date/filter + sport + printed time) — **fixes blank print header**;
  sport column prints icon + name from config; chip row hidden in print.
- Table/mobile cards/drawer/history/notifications/leads/revenue "By Sport" all use config names + icons.
- Table sub-label and drawer pill show the **resource label** (e.g. "Badminton 2") — **fixes
  badminton bookings showing "🏏 Turf"**.
- LeadModal sport list from config (interest types and pass `PLANS` untouched, as approved).

### Availability tab
- Columns = every **active resource** from config (plus any under maintenance that still holds a
  booking/block that day) — **fixes badminton courts missing**. Rows show resource label and, when
  held, the sport's icon + name and customer. Subtitle lists the resources. Filters work across all.

### New "⚙️ Sports" tab
1. **Sports** — card per sport: icon, name, price/unit, resource pool, concurrency, max hours, status
   pill (Active / Hidden / Walk-in only / Archived), on/off toggle, Edit.
   Turning off calls `getSportImpact` first → "3 upcoming bookings will still be honoured. New online
   bookings will stop." → `setSportActive`.
   **Drawer** (create/edit): name, slug (auto-suggested from name on create, read-only after), icon with
   suggestions, tagline, unit label, price/hr, "Resources it can use" checkboxes (pool order preserved),
   "Max at the same time" (blank = all), max hours, "Show on website", sort order, "Slot length: 1 hour
   (fixed)". Inline validation mirrors the contract table; server errors shown verbatim. **Archive**
   requires typing the sport name. Archived sports in a collapsed list with **Restore**.
2. **Resources** — row per resource: label, used-by sports, "Available / Under maintenance" toggle
   (`getResourceImpact` confirm before turning off), "Shares space with" chips saved via
   `setResourceConflicts` (Save/Undo when changed). **Add resource** (id + label) → "Generate slots now"
   → `generateSlotsForResource`.
3. **Change log** — last 100 `listAudit` entries in plain language, e.g.
   "Bowling Machine Practice price 650 → 700, 17 Oct, 9:14 PM" (IST).

Verified in the mocked run: status pills, impact confirm texts, toggle payloads, inline errors,
`"Sport id \"dupe\" already exists"` shown verbatim, slug read-only on edit, archive button disabled
until the exact name is typed, conflicts payload, create resource → generate slots, change-log wording.

## Decisions (all approved by owner, 2026-10-17)

1. **Total before booking** — approved: display-only server hourly price × hours ("Lock in 2 hrs ·
   ₹1,300"); the server's `amount` stays final on the ticket.
2. **Admin uses deprecated `getSlots` (with adminKey)** in Spot booking / Edit — approved for now,
   because `getAvailability` ignores adminKey and so omits walk-in-only sports. Backend will add admin
   support to `getAvailability` later; the frontend swap is small after that.
3. **Editing a booking whose sport is now off/archived**: `getSlots` returns nothing for inactive sports,
   so only the current slot is listed (duration / sport change still work). A note explains this.
4. **Sport colours** in admin are now assigned by `sort_order` position from an 8-colour palette, so
   existing colours may shift.
5. `updateSport` sends all editable fields (including unchanged ones); the backend audit diff handles
   that, and the change log ignores unchanged fields.
6. Not touched: React is still loaded as unpinned `react@18` (only the Babel pin was in scope).
