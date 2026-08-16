# edtan.ca

A personal photo blog, built with [Hugo](https://gohugo.io/) and published to GitHub
Pages. Photos and posts live in this repository; there is no database and no external
service in the publishing path.

## Running it locally

### Prerequisites

Hugo 0.164 or newer.

```bash
# macOS
brew install hugo

# Arch / Manjaro
sudo pacman -S hugo

# Any platform with Go 1.21+
go install github.com/gohugoio/hugo@latest
```

Either edition works — WebP encoding is present in both the standard and extended
builds, and this site uses plain CSS rather than Sass. CI installs the extended
build for parity, so that adding Sass later does not silently break the deployment.

### Development server

```bash
hugo server -D
```

Serves on <http://localhost:1313> with drafts (`-D`) visible and live reload. The
first run encodes every image derivative and takes a few seconds; subsequent runs are
near-instant because derivatives are cached between builds and never re-encoded.

To check the site on a phone or tablet on the same network, bind to all interfaces
and set `baseURL` to the machine's LAN address — Hugo generates absolute URLs for
assets, so `localhost` in `baseURL` will leave the page unstyled on another device.

```bash
# find the LAN address
ip route get 1 | awk '{print $7; exit}'   # Linux
ipconfig getifaddr en0                    # macOS (Wi-Fi)

# serve with it, e.g. 192.168.1.20
hugo server -D --bind 0.0.0.0 --baseURL "http://192.168.1.20:1313"
```

### Production build

```bash
hugo --gc --minify
```

Output goes to `public/`. Neither `public/` nor `resources/` is committed.

To preview exactly what would be published:

```bash
hugo --gc --minify && python3 -m http.server -d public 8000
```

### Verifying a build

Published-site size matters — GitHub Pages enforces a hard 1 GB limit — so two checks
are worth running after adding photos:

```bash
du -sh public/                              # total published size

find public -name '*.webp' | grep -v _hu_   # should print nothing
```

The second confirms that only resized derivatives (`_hu_` in the filename) were
published and that full-resolution originals were excluded. If it prints filenames,
the `publishResources` cascade in `hugo.toml` has been broken and the published site
will grow roughly twice as fast as it should.

## Content

Posts are [Hugo leaf bundles](https://gohugo.io/content-management/page-bundles/):
a directory containing `index.md` plus its images, so a post and its photos move and
delete together.

```
content/
  photos/<slug>/index.md    + co-located .webp    # photo sets
  posts/<slug>/index.md     + co-located .webp    # writing
```

A photo set lists its images in front matter:

```yaml
---
title: "Harbourfront, August"
date: 2026-08-10
cover: boat.webp
location: "Toronto, ON"
images:
  - src: boat.webp
    alt: "A moored sailboat, rigging against an overcast sky"
    caption: "Late afternoon, the wind finally dropping off."
---
```

A written post uses `summary` and puts images inline:

```markdown
{{< figure src="inline.webp" alt="…" caption="…" >}}
```

Set `draft: true` to keep a post out of production builds; `hugo server -D` still
shows it.

## Images

Uploads are capped at 2560px and converted to WebP before they enter the repository.
At build time each image is resized to a ladder of widths — 480, 768, 1200, 1600,
2048 — and served via `srcset`, so a phone downloads roughly 40 KB where a desktop
downloads a few hundred.

All of that is handled by `layouts/_partials/image.html`, which is the only template
that turns an image reference into a URL. Both galleries and inline figures route
through it; adding a `<img>` tag directly to a template would bypass the ladder.

## Editing

Content can be edited by hand, or through [Sveltia CMS](https://sveltiacms.app/) at
`/admin` on the live site, which commits directly to this repository. The CMS is
authenticated against GitHub and is only usable by accounts with write access.

## Deployment

Pushing to `master` triggers `.github/workflows/deploy.yml`, which builds the site
and publishes it to GitHub Pages. There is no manual deploy step; the CMS commits to
`master`, which is what triggers a build.

The workflow caches Hugo's generated image derivatives between runs. This is not an
optimisation — without it, every deploy re-encodes the entire photo archive, so build
time would grow with the number of photos ever published rather than with the number
just added.

Two checks run before anything is published:

- the build fails if full-size originals were published alongside their derivatives
- the published size is printed, to track against the 1 GB Pages limit

## Layout

```
content/          posts and photo sets
layouts/          templates (Hugo v0.146+ template system)
  _partials/      image.html, head.html
  _shortcodes/    figure.html
assets/css/       stylesheet, fingerprinted at build
static/           files copied verbatim, including CNAME
hugo.toml         site configuration
```

## License

Site content — text and photographs — is © Ed Tan, all rights reserved.
The templates and configuration may be reused freely.
