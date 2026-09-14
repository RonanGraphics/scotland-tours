---
name: walking-tour
description: Generates a standalone HTML self-guided walking tour page for a place in Scotland and commits it so GitHub Pages serves it. Use whenever the user asks for a walking tour, an audio tour, a guided walk or a tour of stops around a place. Triggers include "we're at Edinburgh Castle, give me a walking tour with 7 stops", "walking tour of Kirkwall", "tour around Inverness, 5 stops, history", "make a tour of where we are", "give me a walk with 8 stops near Stirling Castle", "food tour of Glasgow".
---

# Walking tour generator

Builds one self-contained page per tour from `template/tour.html`, writes it to `tours/<slug>.html`,
adds it to `index.html`, and pushes. GitHub Pages serves it at
`https://ronangraphics.github.io/scotland-tours/tours/<slug>.html`.

Follow the steps in order. Do not skip step 9.

## 1. Parse the request

Pull these out of what the user said:

| Field | Rule |
|---|---|
| anchor | REQUIRED. A place name, a landmark, or coordinates. If the user says "where we are" and gives a lat/lon, use those numbers directly as the anchor and skip step 2's geocode. If there is no anchor at all, ask for one and stop. |
| stops | Default 6. Maximum 12. "a few" = 5, "lots" = 10. |
| flavour | One of history, food, kids, nature, architecture. Default general. |
| radius | Default 1200 metres. "wider" / "spread out" / "we have all day" = 2500. |

## 2. Geocode the anchor

Wikipedia first:

```
https://en.wikipedia.org/w/api.php?action=query&prop=coordinates|pageimages|extracts&exintro=1&explaintext=1&piprop=original&titles=<Title>&format=json&redirects=1
```

Read `query.pages.<id>.coordinates[0].lat` / `.lon`.

If that returns no coordinates (or a missing page), fall back to Nominatim:

```
https://nominatim.openstreetmap.org/search?q=<query>,Scotland&format=json&limit=1
```

Try the **WebFetch** tool first. If WebFetch is unavailable, use `curl -s`. Nominatim requires a
User-Agent header when called with curl, and it is good manners on the Wikipedia API too:

```bash
curl -s -A "scotland-tours/1.0 (personal walking tour generator)" "<url>"
```

URL-encode the pipe characters as `%7C` when using curl.

## 3. Find candidate stops

Geosearch around the anchor:

```
https://en.wikipedia.org/w/api.php?action=query&list=geosearch&gscoord=<lat>|<lon>&gsradius=<radius>&gslimit=50&format=json
```

Then batch-fetch details for the candidates in one call (pipe-separated page ids, 20 at a time):

```
https://en.wikipedia.org/w/api.php?action=query&prop=coordinates|pageimages|extracts|description&exintro=1&explaintext=1&exlimit=20&piprop=original|thumbnail&pithumbsize=800&pageids=<id|id|...>&format=json
```

Now pick the N most interesting for the flavour. This is judgement, not a filter:

- **Prefer** landmarks, castles, churches, museums, monuments, memorials, historic streets and
  squares, bridges, viewpoints, notable buildings, parks.
- **Skip** disambiguation pages, council and corporate offices, bus and railway stations, companies,
  research centres, schools and university departments, people, ships, and any "List of ..." page.
- **Flavour steer:** history favours monuments, old streets and sites of events; food favours markets,
  distilleries, historic inns and food halls; kids favours hands-on attractions, animals, big views
  and good stories; nature favours parks, waterfronts, hills and gardens; architecture favours named
  buildings and their architects.
- **Always include the anchor itself as stop 1** if it is a place rather than an abstract topic.

If Wikipedia yields fewer than N good candidates, widen the radius **once** (1200 to 2500, or 2500 to
4000) and search again. If still short, top up from your own knowledge of the town, using coordinates
you are confident about, and mark those stops `"source": "knowledge"`.

Use the **`thumbnail.source`** URL exactly as the API returns it (from `pithumbsize=800`; the host is usually `thumb.wikimedia.org` and the width may snap to 960), never `original.source` and never a hand-built thumb URL, which returns 400.
Originals are often 3000 px and close to 1 MB each, which is the wrong thing to send to a phone on
patchy Highland signal. Strip the `?utm_source=...` tracking query string off any
`upload.wikimedia.org` image URL before using it. A stop with no Wikipedia image gets `"image": null` and the page draws a styled placeholder.
**Never invent an image URL.**

## 4. Order the stops

Start at the anchor. Nearest-neighbour from there: repeatedly walk to the closest stop not yet used.
End near the start where that falls out naturally.

Distances are straight-line haversine between consecutive stops:

```
a = sin(dLat/2)^2 + cos(lat1) * cos(lat2) * sin(dLon/2)^2
d = 2 * 6371 * asin(sqrt(a))      # km
```

- `totalKm` = sum of consecutive legs, rounded to 2 decimals.
- `estMinutes` = round(totalKm * 15 + stopCount * 8).

## 5. Write the narration

For each stop write 170 to 240 words in a warm, knowledgeable guide voice. Professional but fun.
British spelling throughout. In this order:

1. A one-line hook that makes the visitor look up.
2. Two or three concrete facts, grounded in that stop's Wikipedia extract: dates, names, numbers,
   why it matters. Do not state a fact the extract does not support unless you are certain of it.
3. One "look for this" physical detail they can actually see while standing there.
4. One light aside or piece of local colour.
5. A closing line that leads into the next stop by name.

The narration is read aloud by text-to-speech, so: **plain prose only.** No bullet points, no
markdown, no headings, no parentheses full of dates, no ampersands. Write numbers and dates the way
you would say them.

Also write:

- `teaser` -- one sentence per stop, the line that appears under the title.
- `intro` -- two or three sentences for the whole tour, setting the scene and saying roughly how long
  the walk takes.
- `tip` -- optional, and only when the extract actually tells you something useful: opening hours,
  a steep climb, a lot of steps, whether it costs money. Otherwise `null`. Do not guess opening times.

## 6. Build the page

Read `template/tour.html`. Make exactly two replacements:

1. The single line `/*__TOUR_DATA__*/` becomes `const TOUR = <json>;`
2. `__TOUR_TITLE__` in the `<title>` becomes the tour title.

JSON shape:

```json
{
  "slug": "edinburgh-castle-7-0925",
  "title": "Edinburgh Castle and the Castlehill Old Town",
  "location": "Edinburgh, Scotland",
  "intro": "Two or three sentences.",
  "flavour": "general",
  "generated": "2026-09-14",
  "anchor": { "lat": 55.94861, "lon": -3.20083 },
  "totalKm": 0.61,
  "estMinutes": 65,
  "stops": [
    {
      "n": 1,
      "title": "Edinburgh Castle",
      "teaser": "One sentence.",
      "narration": "170 to 240 words of plain prose.",
      "tip": null,
      "lat": 55.94861,
      "lon": -3.20083,
      "image": "https://upload.wikimedia.org/wikipedia/commons/...jpg",
      "imageCredit": "Wikimedia Commons",
      "wiki": "https://en.wikipedia.org/wiki/Edinburgh_Castle",
      "source": "wikipedia"
    }
  ]
}
```

`slug` = kebab-case location, then stop count, then MMDD. For example `edinburgh-castle-7-0925`,
`kirkwall-5-0926`. Write the result to `tours/<slug>.html`.

Write the JSON with a real JSON serialiser (`python3 -c` with `json.dumps`, or `node`), never by
hand-pasting quotes into the template. That is what stops a stray apostrophe breaking the page.

## 7. Update index.html

Insert one `<li>` at the top of `<ul id="tours">` (newest first), immediately after the
`<!-- newest first -->` comment:

```html
<li><a href="tours/edinburgh-castle-7-0925.html">Edinburgh Castle and the Castlehill Old Town</a>
<span class="m">Edinburgh, Scotland &middot; 7 stops &middot; 14 September 2026</span></li>
```

It stays a plain static list. No JavaScript on the index beyond the browser bar that is already there; never remove it.

## 8. Commit and push

```bash
git add -A && git commit -m "Tour: <title>" && git push
```

Then tell Ian:

- the tour URL `https://ronangraphics.github.io/scotland-tours/tours/<slug>.html`
- the index URL `https://ronangraphics.github.io/scotland-tours/`
- that GitHub Pages takes about a minute to pick up the change

Mention that if the link opens inside the Claude app, the bar at the top of the page has Open in
Chrome / Open in Edge buttons; audio only works in a full browser.

## 9. Done when

All of these, checked, not assumed:

- [ ] `tours/<slug>.html` exists.
- [ ] The `TOUR` JSON parses. Extract the block between `const TOUR = ` and the final `;` and run it
      through `node -e` or `python3 -c "import json; json.loads(...)"`. A page whose JSON is broken
      renders blank.
- [ ] Every stop has a numeric `lat`, a numeric `lon`, and a non-empty `narration`.
- [ ] `__TOUR_TITLE__` and `/*__TOUR_DATA__*/` no longer appear in the generated file.
- [ ] `index.html` contains a link to the new file.
- [ ] `git push` succeeded.

## Troubleshooting

**Wikipedia or Nominatim fetch fails (sandbox network blocked, DNS refused, timeout).**
Say so plainly: "I could not reach Wikipedia from here, so this tour is built from my own knowledge."
Then build the tour anyway from what you know about the place. Every such stop gets
`"source": "knowledge"` and `"image": null`. The page handles null images with a placeholder, so it
still looks right. Do not fail the request, and do not invent image URLs to fill the gaps.

**Geosearch returns almost nothing** (small village, remote glen). Widen the radius once, then fill
the rest from knowledge. A five-stop tour of a small place beats a padded ten-stop one.

**The page renders blank in the browser.** The JSON did not parse. Check for an unescaped newline or
a smart quote in a narration. Re-serialise with `json.dumps` rather than patching by hand.

**No route line on the map.** OSRM's demo server is rate-limited and sometimes slow. The page falls
back to dashed straight lines after six seconds by design. Nothing to fix.

**Speech does nothing on iPhone.** Voices load asynchronously on iOS and speech needs a user gesture.
The template already handles both. If it is still silent, the phone is on silent mode with the
ringer switch, which also mutes speech synthesis in Safari.
