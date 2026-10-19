# Nimbus

**Nimbus** is an offline-first notebook: keep thoughts, sketches, and photos
together. It is built as a demo PWA for [PWABuilder](https://www.pwabuilder.com)
using the [pwa-starter](https://github.com/pwa-builder/pwa-starter) stack:
Lit, Web Awesome, TypeScript, Vite, and Workbox.

**Try it:** [pwa-builder.github.io/nimbus](https://pwa-builder.github.io/nimbus/).

## The notebook

The home screen is your notebook, not a launcher for separate demo apps.
Create, search, pin, edit, share, and export notes. **Add sketch** opens a
pressure-sensitive drawing pad; **Add photo** opens the camera, image picker,
and worker-powered filters. Both attach images directly to the active note.
Each attachment can be downloaded or removed independently.

The editor has **Write** and **Preview** tabs. Write edits the original Markdown;
Preview renders headings, emphasis, lists, task lists, links, quotes, fenced code,
and tables locally, including offline. Switching tabs never changes the saved
source. Preview HTML is sanitized with DOMPurify, links open in a separate tab,
and task checkboxes are read-only. Remote Markdown images still require a
network connection; attached sketches/photos remain available offline.

**Note tools** integrates useful web capabilities into the note:

| Action | Web capability |
| --- | --- |
| Dictate / read aloud | Speech recognition and synthesis |
| Insert contact / add location | Contact Picker and Geolocation |
| Keep screen awake | Screen Wake Lock |
| Notify me now | Service-worker notifications linking back to a specific note |
| Protect local notes | Storage persistence and quota estimates |
| Share / copy text | Web Share (including images), Clipboard |
| Pin note | Pinned notes sort first; supported installed apps show a pinned-note count badge |
| Focus | Distraction-free editor layout |

Unavailable device features are disabled; errors and permission denials are
shown in the UI. Dictation may require a network and use the browser's online
speech service. Notifications are immediate, not scheduled reminders.
Unrelated hardware demos and the mock Background Sync request have been removed:
Nimbus has no cloud-sync backend.

## Data and backups

Notes and attachment `Blob`s are stored together in IndexedDB. Existing Nimbus
text notes need no migration. Previously saved photos remain available under
**Add photo > Previously saved photos**. Removing an attachment does not delete
the original legacy-gallery photo.

Saving is automatic, and writes/deletes are ordered. Failed writes are reported
with a retry action; do not close Nimbus before retrying or exporting a backup.
Browser-local storage is not a backup and may be cleared by the user or browser.

**Export** creates a `.nimbus.json` backup containing text, metadata, and embedded
image bytes. It uses the File System Access save picker when supported, with a
download fallback. **Import** accepts these backups as new notes, plus `.txt`
and `.md` files. File sharing falls back to a complete backup if the browser
cannot share images; text-only sharing falls back to the clipboard.

The manifest includes note, sketch-note, and photo-note shortcuts, text Share
Target, file handling, and a `web+nimbus` protocol handler. Navigation uses
query strings: `/?view=about`, `/?note=ID`, and `/?tool=sketch` (or `photo`,
`tools`). Shortcuts use `/?new=1`, `/?new=sketch`, and `/?new=photo`.
Old path-based bookmarks are recognized if a service worker or host serves the
app shell, but are not valid first-visit links on GitHub Pages.

## Development

```sh
npm ci
npm run dev
npm test
npm run build
npm run preview
```

Use Node.js 22 or newer. The tests use Node's test runner and `fake-indexeddb`
to cover legacy records, image persistence, transaction failures, deletion,
and backup round trips. The build type-checks and creates a static `dist`.
`npm run test:pages` builds root and `/nimbus/` variants and runs Playwright
against plain static servers with no SPA fallback. Install its browser once
with `npx playwright install chromium`. The suite covers cold deep links,
refresh, back/forward, browsers with and without the Navigation API, scoped
service workers, and offline assets. `tests/notebook.browser.js` also
runs in an isolated context and covers in-note media, camera cleanup,
permission errors, dictation, failed-save recovery, offline reload, mobile
layout, and note-switch/delete races. Camera and speech are mocked, so the
regression suite does not request access to real hardware.

For offline verification, use the production preview: visit once online, wait
for the service worker to control the page, then reload with the browser offline.
Notes, attachments, sketching, and photo filters work without a network.
Device-dependent APIs still require browser support and permission.

## Offline assets

Web Awesome styles and the used Font Awesome Free icons are bundled/self-hosted.
`scripts/copy-icons.mjs` copies the selected icon set into
`public/assets/fa/{solid,brands}` during builds. Add new icon names to that script.
The service worker precaches the app, lazy-loaded media editors, workers, and
assets. HTTPS or localhost is required for PWA features.

## GitHub Pages

This standalone repository publishes
**https://pwa-builder.github.io/nimbus/** through
[`.github/workflows/pages.yml`](.github/workflows/pages.yml). It does not
deploy or change the PWABuilder documentation site or its custom domain.

The workflow runs on pull requests, pushes to `main`, and manual dispatch:

1. Set up Node.js 22 and install the locked dependencies with `npm ci`.
2. Run `npm test` and `npm run test:pages`, including the root and `/nimbus/`
   production builds and the Playwright notebook/Markdown regressions.
3. Build the deployment with `npm run build -- --base=/nimbus/`.
4. On `main` only, upload **`dist/`** with `actions/upload-pages-artifact`
   and deploy with `actions/deploy-pages` to the `github-pages` environment.

Repository **Settings > Pages > Build and deployment > Source** must be
**GitHub Actions**. Pull requests validate the app but cannot publish it.
Deployment waits for the build and tests; concurrent deployments are serialized.
No custom domain or DNS configuration is required.

Build for the URL where the app will be served (the base must start and end
with `/`):

```sh
# Custom domain or an organization/user site, at the origin root:
npm run build

# This repository's GitHub Pages project site:
npm run build -- --base=/nimbus/
```

Publish the **contents of `dist/`**, not the source folder. The repository path
is case-sensitive. A root-hosted deployment uses the root build instead.

All new links request the deployed index document. For example, About becomes
`/nimbus/?view=about`, so direct visits and refreshes return HTTP 200 without
rewrites, generated route pages, or a `404.html` workaround. The Navigation API
enhances same-document navigation where supported; other browsers follow normal
links and use their native history. Unsaved-note protections remain active.

Vite's base controls bundled assets, self-hosted icons, HTML links, and
service-worker registration/scope. The static manifest uses relative URLs for
its start URL, scope, shortcuts, and handlers. It intentionally omits `id` so
identity defaults to the resolved start URL: an explicit relative `id` would
resolve against the origin and collide across project sites. This keeps the
existing root-hosted identity while giving each project path its own identity.
Service-worker notification URLs
resolve against its registration scope, not the origin root.

Do not move the app to another base/domain after users install it without a
migration plan: manifest identity and service-worker scope depend on that URL.
Notes are origin-local, so moving domains requires export/import. Other apps
on the same Pages origin share origin-level storage and permissions; a dedicated
custom domain gives Nimbus its own origin.

## Source provenance

Nimbus was extracted from
[`pwa-builder/PWABuilder`](https://github.com/pwa-builder/PWABuilder), directory
`apps/pwa-demo`, at commit
[`b252ac860688868ddc7b163fed3979371398eea9`](https://github.com/pwa-builder/PWABuilder/tree/b252ac860688868ddc7b163fed3979371398eea9/apps/pwa-demo).
The initial standalone import preserves the tracked application, tests,
dependency lockfile, icons, and screenshots. Repository naming, deployment
configuration, test hosting paths, and this documentation were adapted for
the standalone repository; the notebook implementation was not rewritten.
The original root MIT license and Microsoft attribution are retained in
[`LICENSE.txt`](LICENSE.txt).

## License

MIT - see [`LICENSE.txt`](LICENSE.txt). Font Awesome Free icons retain their
embedded upstream notices (Copyright Fonticons, Inc.) and are licensed under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/); see
[Font Awesome's license](https://fontawesome.com/license/free) for details.
