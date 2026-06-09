# Stage Host — schedule data format (v2)

The v2 app is **content-free**: `index.html` ships only the UI + logic and loads
the actual programme from a JSON file at runtime. One file = **one room, one day**.

- Machine-readable schema: [`schedule.schema.json`](schedule.schema.json) (JSON Schema 2020-12)
- Example / current data: [`data/frannz-salon-2026-06-09.json`](data/frannz-salon-2026-06-09.json)

## Loading a schedule

By default the app loads `data/frannz-salon-2026-06-09.json`. Point it at any
other file with the `data` query parameter (path is relative to `index.html`):

```
https://h1f1x.github.io/stage-host-assi/v2/?data=data/my-room-2026-06-10.json
```

Drop a new JSON file into `v2/data/`, push, and link to it with `?data=`. That's it.

> Note: opening `index.html` from `file://` will fail because browsers block
> `fetch` of local files. Serve it over HTTP (GitHub Pages already does).

## What you author vs. what is derived

You only describe **what exists**; the app computes the flow:

| You provide          | Derived automatically at load                                            |
|----------------------|--------------------------------------------------------------------------|
| `talks` (in order)   | each talk's **"Als Nächstes"** pointer (next talk / break / closing)      |
| `breaks`             | the break rows on the agenda + the "after the break" pointer             |
| `closing`            | the closing row on the agenda + the final pointer                        |
| `meta.date`          | all countdowns                                                           |
| `talks[].start/end`  | the agenda time-range and session count ("6 Sessions · 09:30 – 15:30")   |

So **do not** hand-write a `next` field — it is generated from the ordered list.

## Structure (summary)

```jsonc
{
  "meta": {
    "event": "Berlin Buzzwords",
    "room": "Frannz Salon",          // required — shown (uppercased) on the agenda pill
    "date": "2026-06-09",            // required — ISO YYYY-MM-DD, drives countdowns
    "dayLabel": "Dienstag · 9. Juni 2026",
    "agendaTitle": "Deine Talks heute",
    "avatarBase": "https://program.berlinbuzzwords.de/media/avatars/"
  },
  "talks": [
    {
      "n": 1,                        // optional; defaults to position
      "start": "09:30", "end": "09:50",   // required, 24h "HH:MM"; end includes Q&A
      "type": "Short Talk",          // required
      "title": "…",                  // required
      "speaker": "Celeste Horgan",   // required
      "role": "Sr. OSS Developer Advocate · Snowflake",
      "tags": ["Data Science", "Scale", "Short Talk · 20 min"],
      "photo": "https://…/KVW9PZ_xJNiKyO.webp",  // full URL, or bare filename + meta.avatarBase
      "url": "https://…/session/…/",
      "intro": "Few short, plain sentences the host reads before the talk.", // required
      "hand":  "One upbeat line, ending with the speaker's name!",           // required
      "links": [{ "l": "LinkedIn", "u": "https://…" }]
    }
  ],
  "breaks": [
    { "afterTalk": 2, "name": "Frühstückspause", "start": "10:40", "end": "11:10" }
  ],
  "closing": { "label": "Closing Session", "time": "17:10", "room": "Kesselhaus" }
}
```

Field-by-field rules live in [`schedule.schema.json`](schedule.schema.json). Validate with any
JSON-Schema tool, e.g.:

```bash
npx ajv-cli validate -s v2/schedule.schema.json -d v2/data/your-file.json --spec=draft2020
```

---

## LLM prompt — build a schedule file from a programme webpage

Paste the prompt below into an LLM **that can browse the web** (or paste the page
HTML/text yourself). Fill in the three placeholders at the top.

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

After the model returns the JSON, save it as `v2/data/<room>-<date>.json`, push,
and open `…/v2/?data=data/<room>-<date>.json`.
