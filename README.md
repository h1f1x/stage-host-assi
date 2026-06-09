# stage-host-assi

A phone-friendly **stage host assistant** for conference MCs — built for events
like [Berlin Buzzwords](https://berlinbuzzwords.de). It walks the host through a
single room's day, one swipe at a time: the agenda, a per-talk overview, a ready
-to-read **Anmoderation** (intro) script with a hand-over line, and a live
countdown that flashes and chimes as a talk's time runs out.

The app ships **no programme of its own** — `index.html` is pure UI + logic and
loads the schedule for one room/one day from a JSON file at runtime.

## Live

- **App (current, v2):** https://h1f1x.github.io/stage-host-assi/v2/
- **Original (v1):** https://h1f1x.github.io/stage-host-assi/

Open the app, tap the ⚙️ gear, and pick a room from the **schedule chooser**.
You can also link straight to a room with the `data` query parameter (path is
relative to `v2/index.html`):

```
https://h1f1x.github.io/stage-host-assi/v2/?data=data/kesselhaus-2026-06-09.json
```

## What the host gets, per talk

- **Agenda** — every session of the day, with the current/next talk highlighted.
- **Overview** — title, speaker, role, tags, quick links, and a "starts in" timer.
- **Anmoderation** — a short, plain-language intro to read out, plus one upbeat
  hand-over line. A 🔊 button reads it aloud (browser text-to-speech).
- **Countdown** — remaining time with screen-flash + audio alerts at 10 and 5
  minutes, and a fullscreen timer for the speaker. A time offset in settings
  keeps it in sync if the schedule is running late.

## Project layout

```
index.html                  original (v1) single-file app
v2/
  index.html                current app (UI + logic only)
  styles.css                styling
  schedule.schema.json      JSON Schema for a schedule file
  SCHEMA.md                 data-format guide + the LLM authoring prompt
  data/
    index.json              the room list shown in the chooser
    <room>-<date>.json      one file per room per day
PRD-stage-host-deck.pdf     product brief
.github/workflows/          GitHub Pages deploy (on push to main)
```

Deployment is automatic: pushing to `main` publishes the repo root to GitHub
Pages.

## Adding a schedule

One file = **one room, one day**. Drop a new JSON file into `v2/data/`, add it to
`v2/data/index.json` so it appears in the chooser, push to `main`, and it's live.

The full field-by-field format (and a machine-readable JSON Schema you can
validate against) lives in **[`v2/SCHEMA.md`](v2/SCHEMA.md)** — that file is the
source of truth for the data format.

### LLM prompt (build a schedule from a programme page)

Paste the prompt below into an LLM that can browse the web (or paste the page
text yourself), fill in the three placeholders at the top, and save the result as
`v2/data/<room>-<date>.json`. This mirrors the prompt in
[`v2/SCHEMA.md`](v2/SCHEMA.md); see that file for the authoritative version and
the exact schema.

````text
You are building a JSON schedule for a conference "Stage Host" app. Produce data
for EXACTLY ONE room on ONE day.

INPUTS
- PROGRAMME_URL: <paste the conference programme/agenda URL>
- ROOM: <exact room/stage name, e.g. "Frannz Salon">
- DATE: <the day to extract, ISO format, e.g. 2026-06-09>

TASK
1. From PROGRAMME_URL, find every session in ROOM on DATE, in chronological order.
   If a session has its own page, open it for details (speaker role, links, photo).
2. For each session capture: start, end (24h "HH:MM"; if only a duration is given,
   compute end = start + duration; end should include Q&A time), type
   (e.g. "Talk", "Short Talk", "Keynote"), title, speaker (join multiple with ", "),
   role ("Role · Affiliation"), tags (tracks/topics; make the LAST tag a duration
   label like "Talk · 40 min"), url (session page), links (LinkedIn / blog / profile
   as {l,u}), and photo (full image URL if available, else omit).
3. Also WRITE two host scripts per talk, in simple, warm, spoken English:
   - "intro": 3–6 short sentences. Welcome the room, then explain in plain language
     what the talk is about and why it matters, and one credible detail about the
     speaker. No marketing fluff, no jargon, no buzzword soup.
   - "hand": ONE upbeat sentence that hands over to the stage and ENDS with the
     speaker's name (e.g. "… please welcome Celeste Horgan!").
4. Identify breaks between sessions in ROOM (name + start + end) and set "afterTalk"
   to the number (n) of the talk each break follows.
5. Identify the day's closing item (label, time, room) if there is one.

OUTPUT
- Return ONE JSON object and nothing else (no prose, no code fence).
- It MUST validate against this schema (key points):
  meta{event,room*,date*,dayLabel,agendaTitle,avatarBase}, talks[]* with
  {n,start*,end*,type*,title*,speaker*,role,tags[],photo,url,intro*,hand*,links[{l,u}]},
  breaks[]{afterTalk*,name*,start*,end*}, closing{label*,time*,room}.  (* = required)
- Times are 24h "HH:MM". Number talks n = 1,2,3,… in start-time order.
- Do NOT include a "next" field — the app derives it from the ordered talks/breaks.
````
