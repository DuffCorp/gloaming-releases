# Working with Gloaming

Gloaming turns a finished astro image into a complete Instagram post: labels placed from a plate solve, a
fact-checked card and caption, sky charts, the night's conditions, Reels. It's meant to sit at the end of
whatever workflow you already have. AstroWizard, Siril, PixInsight, GraXpert or a script of your own can hand
Gloaming an image (and everything it knows about it) and get a post back.

There are four ways in. All of them run the same engine operations the app uses, so anything you can do in the
app, you can do from outside it.

| Way in | Best for | Needs Gloaming open? |
| --- | --- | --- |
| [Inbox folder](#1-the-inbox-folder) | "Send to Gloaming" from any app: just save a file | Yes |
| [Links](#2-gloaming-links) | A button in your app that opens Gloaming on an image | No (it starts) |
| [Local API](#3-the-local-api) | Two-way: create, edit, render, read results | Yes |
| [Command line](#4-the-command-line) | Batch jobs and scripts, no window at all | No |

Nothing here sends anything off your computer. The API listens on 127.0.0.1 only and needs a token.

---

## The sidecar file

Whichever way you use, tell Gloaming what you know in a small JSON file next to the image. Name it
`<image>.gloaming.json` (e.g. `M31.tif.gloaming.json`) or `<image name>.gloaming.json` (`M31.gloaming.json`).
Every field is optional; anything you leave out, Gloaming works out or leaves for the user to fill in.

```json
{
  "gloaming": 1,
  "app": "AstroWizard 1.4",

  "object": "M31",
  "title": "Andromeda",
  "subtitle": "M31 · Spiral galaxy",

  "integration": "3h 10m",
  "telescope": "Seestar S30 Pro",
  "camera": "ZWO ASI585MC",
  "frames": 190,
  "exposure_s": 60,
  "filters": ["L-eXtreme"],
  "palette": "HOO",
  "stretch": "GHS",

  "starless": "C:/Astro/M31/M31 starless.tif",
  "fits": "C:/Astro/M31/M31_solved.fits",
  "capture": "C:/Astro/M31/stacked.fits",

  "date": "2026-10-20",
  "site": "Galloway Forest Park",
  "private": false,

  "notes": "First light with the new filter."
}
```

| Field | What Gloaming does with it |
| --- | --- |
| `gloaming` | The format version. Use `1`. |
| `app` | Shown to the user ("New post from AstroWizard 1.4") and kept with the post. |
| `object` | Catalogue name (M31, NGC 7000, Sh2-129, "Heart Nebula"). Facts are looked up from SIMBAD and Wikipedia. If missing, Gloaming reads it from the file name. |
| `title`, `subtitle` | The card's title and subtitle (instead of the looked-up ones). |
| `integration`, `telescope`, `notes` | The card's total time, kit and closing note. |
| `starless` | A starless version: becomes the starless slide and the Reel's "stars fade in". |
| `fits` | A **plate-solved** FITS (with WCS) of the same frame. Labels are placed from it: exact positions, never guessed. |
| `capture` | A stacked FITS or camera JPEG: exposures, frames and dates are read from its header. |
| `date`, `site`, `private` | The night it was taken: weather, Moon and darkness for the night slide. `private: true` keeps the place vague. |
| `camera`, `frames`, `exposure_s`, `filters`, `palette`, `stretch`, anything else | Kept with the post (`post.source`) for the caption, the specs slide and "how it was made". |

Paths can be absolute or relative to the image.

---

## 1. The inbox folder

Save the image (and its sidecar) into the inbox, and a draft post opens in Gloaming. Write the sidecar first, or
within a second of the image, so it's there when the image is picked up.

* Default location: `~/Pictures/Gloaming Inbox` (Windows: `C:\Users\<you>\Pictures\Gloaming Inbox`).
* Its actual location is in `bridge.json` (below) as `inbox`.

This is the simplest "Send to Gloaming" there is: an export preset that saves to that folder.

## 2. gloaming:// links

```
gloaming://import?image=<url-encoded path>[&sidecar=<url-encoded path>]
gloaming://open?post=<url-encoded path to post.json>
```

Opening the link starts Gloaming if it isn't running. From a terminal: `start "" "gloaming://import?image=…"`
(Windows), `open "gloaming://…"` (macOS), `xdg-open "gloaming://…"` (Linux).

## 3. The local API

While Gloaming is open it serves `http://127.0.0.1:<port>/v1`. The port and a token are in `bridge.json`, in
Gloaming's data folder:

| | `bridge.json` |
| --- | --- |
| Windows | `%APPDATA%\Gloaming\bridge.json` |
| macOS | `~/Library/Application Support/Gloaming/bridge.json` |
| Linux | `~/.config/Gloaming/bridge.json` |

```json
{ "port": 47321, "token": "…", "inbox": "…", "api": "http://127.0.0.1:47321/v1" }
```

Every request needs `Authorization: Bearer <token>`. `GET /v1/info` says what's available; everything else is
`POST /v1/<operation>` with a JSON body of parameters. Replies are `{"ok": true, "result": …}` or
`{"ok": false, "error": "…"}`.

```bash
TOKEN=…; API=http://127.0.0.1:47321/v1
curl -H "Authorization: Bearer $TOKEN" $API/info

# An image and its sidecar become a post (and it opens in Gloaming)
curl -H "Authorization: Bearer $TOKEN" -d '{"image": "C:/Astro/M31/M31.tif"}' $API/import

# Draw the slides small (paths to JPEGs come back)
curl -H "Authorization: Bearer $TOKEN" -d '{"post": "C:/…/post.json", "width": 540}' $API/post.previews

# Export the finished post: slides, caption, alt text (and Story/Reel if switched on)
curl -H "Authorization: Bearer $TOKEN" -d '{"post": "C:/…/post.json"}' $API/post.export
```

```python
import json, os, urllib.request
cfg = json.load(open(os.path.expandvars(r"%APPDATA%\Gloaming\bridge.json")))
def gloaming(op, **params):
    req = urllib.request.Request(f"{cfg['api']}/{op}", json.dumps(params).encode(), {"Authorization": f"Bearer {cfg['token']}", "Content-Type": "application/json"})
    return json.load(urllib.request.urlopen(req))["result"]

post = gloaming("import", image=r"C:\Astro\M31\M31.tif", sidecar={"app": "My script", "object": "M31", "integration": "3h"})
gloaming("post.export", post=post["post"])
```

### Operations

| Operation | Parameters | Returns |
| --- | --- | --- |
| `info` | | Gloaming and engine versions, API version, data folder |
| `import` | `image`, `sidecar?` (path or object), `out?` | `{post, object, from}`, and the post opens in Gloaming |
| `post.new` | `hero`, `object?`, `out?` | `{post, object, slides}` |
| `post.load` / `post.save` | `post` / `post`, `data` | The post (plain JSON) / `{ok}` |
| `post.previews` | `post` or `data`, `width?` | `[{kind, key, hidden, image}]` |
| `post.export` | `post`, `out?` | `{folder, files, preview}` |
| `post.night` | `post`, `site?`, `date?`, `capture?`, `private?` | The night's details |
| `post.slides` | `post`, `on?: [...]`, `off?: [...]` (`specs sky compare night map`) | `{slides}` |
| `post.caption` | `post` | Fact-checked caption drafts (saved to the post) |
| `post.tags` / `post.alt` | `post` | Hashtags / alt text for each slide |
| `post.reel` | `post`, `seconds?` | `{reel, cover}` |
| `lookup` | `object`, `refresh?` | Facts from SIMBAD and Wikipedia |
| `plan.nights` | `days?`, `where?` | The next nights, scored |
| `plan.go` | `date` | Where to go that night |
| `styles` | | Post styles and the one in use |

A post is plain JSON (see `engine/render.py`, `new_post()`); you can read and change it directly, then save it
back with `post.save`.

The API is versioned (`api` in `info`). Fields may be added; nothing is removed or renamed without a new
version.

## 4. The command line

Every feature also runs from the command line with no window, for batch jobs. In the installed app it's
`gloaming` (`gloaming.exe` on Windows) in the app's `resources/engine` folder; from the repository it's
`python engine/cli.py`. Output is JSON.

```bash
gloaming new M31.tif -o M31/post.json --object M31
gloaming render M31/post.json
gloaming batch "Final images/NGC 7000"
```

Run `gloaming --help` for the full list (labels, captions, Reels, Stories, colour versions, then-and-now, the
planner and more).

---

## For AstroWizard

Suggested "Send to Gloaming" in AstroWizard:

1. Export the finished image as usual (TIFF or JPEG).
2. Write `<export>.gloaming.json` with what AstroWizard knows: `object`, `integration`, `frames`, `exposure_s`,
   `filters`, `palette`, `stretch`, the plate-solved `fits` if there is one, and `starless` if it made one.
3. Either save both into the inbox folder (no code needed), or open
   `gloaming://import?image=<export>` (starts Gloaming if needed), or call `POST /v1/import` to get the post
   path back.

Questions or ideas: open an issue on github.com/DuffCorp/gloaming-releases.
