# ADR-0001: Programmatic monitoring of London Aquatics Centre pool availability

- **Status:** Accepted (Tier 1); Proposed (Tier 2)
- **Date:** 2026-09-04
- **Deciders:** Owner/operator (single user)
- **Supersedes:** —
- **Superseded by:** —

## Context

I hold an Everyone Active membership at London Aquatics Centre (LAC), Queen Elizabeth
Olympic Park, and want to know — without manually opening an app — when I can swim.

"Availability" turns out to be two distinct questions that happen to share a phrase:

1. **Programme availability.** Which sessions run on a given day, in which pool, and on
   how many lanes. At LAC this is the more decision-relevant question: 50m fitness
   swimming in the Competition Pool is frequently reduced to a subset of the ten lanes,
   or displaced entirely by schools, galas and Dive London use. A session running on 5
   lanes versus 10 materially changes whether it's worth the trip.
2. **Capacity availability.** How many bookable spaces remain in a session, or how long
   the waiting list is. Everyone Active's own booking documentation describes a grey
   badge showing spaces left and a red badge showing waiting-list count when full, so
   this figure exists in their booking payloads.

These two questions have very different access characteristics. (1) is served publicly
with no authentication. (2) sits behind member login in a booking system whose interface
contract is undocumented and subject to change without notice.

Constraints and drivers:

- **Low operational burden.** This is a personal utility, not a product. It must survive
  months of neglect and be cheap to fix when it breaks.
- **Terms of service.** Automated access to the authenticated booking system is, on most
  readings, outside Everyone Active's terms. Read-only, low-frequency polling from a
  single legitimate member account is a materially different posture from automated
  booking or high-volume scraping, and the design should stay firmly on the former side.
- **No credentials at rest in a shared place.** Anything touching login needs a
  credential story.
- **Breakage should be loud, not silent.** A monitor that quietly stops finding sessions
  is worse than no monitor.

## Decision

Adopt a **two-tier architecture**, shipping Tier 1 now and treating Tier 2 as a separate,
later decision gated on reconnaissance.

### Tier 1 (Accepted): scrape the public Active In Time timetable

The LAC pool programme is published through an Active In Time embeddable timetable widget
that accepts a date parameter:

```
https://www.activeintime.com/en-gb/embeddable_timetable/15981?selected_date=YYYY-MM-DD
```

Timetable ID `15981` covers the LAC pools, with facility filters for Competition Pool,
Diving Pool and Training Pool, and a session-type filter. A `.pdf` variant of the same
path exists.

- Poll once daily for a rolling 7-day window.
- Parse rows into `{date, start, end, session, facility, lanes}`.
- Persist the parsed result as JSON; diff against the previous run.
- Notify only on change (session added, removed, retimed, or lane count altered).
- Treat a zero-row parse as an error condition, not an empty timetable.

Parsing keys off the time-range cell pattern (`HH:MM - HH:MM`) rather than CSS class
names, since the widget markup has changed historically and class names are the most
fragile available anchor.

### Tier 2 (Proposed, not yet decided): authenticated spaces-remaining

Deferred pending a manual reconnaissance step: open the member booking flow in browser
DevTools (Network → Fetch/XHR), log in, navigate to swim booking for LAC, and record the
request that returns the session list with capacity figures.

The outcome of that reconnaissance decides between two options recorded below. No code
until then — guessing at the interface is the expensive path.

## Considered options

### For Tier 1

| Option | Verdict |
| --- | --- |
| **Scrape Active In Time widget** | **Chosen.** No auth, stable date-parameterised URL, structured rows including lane counts. |
| Scrape the `.pdf` variant | Rejected. Same data behind a harder parse. Useful only as a cross-check if the HTML changes shape. |
| Scrape `everyoneactive.com` centre pages | Rejected. Marketing pages linking out to the same widget; extra hop, no extra data. |
| Manual transcription into a calendar | Rejected. The LAC timetable varies week to week; a static copy goes stale silently. |

### For Tier 2

**Option A — replay the JSON API.** The Everyone Active Android app is
`com.innovatise.everyoneactive`, an Innovatise white-label build over a
leisure-management backend, which implies a JSON interface behind the app rather than
server-rendered HTML. Capture the bearer token or session cookie plus venue/site ID and
date parameter, then replay with an HTTP client.

- *Good:* fast, cheap to run, exact capacity numbers, trivially schedulable.
- *Bad:* undocumented private interface; breaks without warning on redeploy; token
  refresh mechanics unknown until inspected.

**Option B — Playwright headless browser.** Drive the member portal, read the
spaces-remaining badges from the DOM.

- *Good:* no need to reverse the interface; tolerates backend changes that preserve the
  UI; the request pattern is indistinguishable from ordinary member use.
- *Bad:* heavier runtime, slower, needs a browser in the execution environment, more
  moving parts to break on CSS churn.

**Option C — do nothing; check the app manually.** The honest baseline. Tier 1 already
answers the question that actually drives my decisions most days, so Tier 2 may never
clear the cost/benefit bar.

## Consequences

**Positive**

- Tier 1 ships without credentials, without ToS ambiguity, and without a browser
  runtime. It answers the lane-count question that the official app arguably surfaces
  less clearly than the widget does.
- Change-only notification means the steady state is silence.
- Tiering means a Tier 2 breakage never takes Tier 1 down with it.

**Negative / accepted costs**

- Tier 1 depends on a third-party widget's markup. Expect to fix the parser
  occasionally; the zero-row error guard converts this from silent failure into a visible
  one.
- Tier 1 cannot tell me whether a bookable session is full. If LAC moves lane swimming
  to capacity-limited booking, Tier 1's value drops and Tier 2 becomes necessary rather
  than optional.
- Lane counts are as-published, not as-actual. Pool operations override the timetable
  regularly and no data source reflects that.

**Operational guardrails (binding on both tiers)**

- Read-only. No automated booking, no automated waiting-list joins.
- Tier 1: once daily. Tier 2, if built: no more than one poll per 10–15 minutes, single
  account, backoff on error rather than retry-hammer.
- Identifiable `User-Agent` rather than browser impersonation.
- Credentials via environment variables or a secrets store, never in the repository.

## Implementation notes

- Scheduling: cron on an always-on box, or a GitHub Actions `schedule` trigger if the
  state file is committed back or cached.
- Notification: `ntfy.sh` or a Telegram bot — lowest friction to a UK handset, no SMTP
  reputation problems.
- State: a single JSON file keyed by `(date, start, end, session, facility)`. Comparing
  full session tuples rather than counts catches retimings, which a count-only diff
  misses.

## Open questions

1. Do the widget's facility and session-type filters work as query parameters? If so,
   Tier 1 can request only 50m fitness swimming and shrink the parse surface.
2. Does the widget expose a JSON or iCal representation alongside `.html` and `.pdf`?
   Unverified; would remove the parser fragility entirely and is worth ten minutes of
   probing before investing further in HTML parsing.
3. Is LAC lane swimming actually capacity-booked, or walk-in on membership? This
   determines whether Tier 2 has any value at all and should be answered before any
   Tier 2 work.

## References

- Active In Time LAC pool timetable widget, ID `15981`
- Everyone Active booking guidance (grey/red spaces and waiting-list badges)
- Everyone Active Android package `com.innovatise.everyoneactive`
- Companion implementation: `lac_timetable.py`
