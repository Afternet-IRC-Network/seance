# AfterNET Chat

AfterNET's web IRC client: [Seance](https://github.com/evilnet/seance) with AfterNET's branding, published to
**<https://afternet-irc-network.github.io/seance/>**.

Seance is a static, bouncer-less client — the browser speaks IRCv3 straight to the ircd over a
WebSocket, with no server in between. This repository holds **no client code**. It holds the
branding, and a workflow that fetches Seance, lays the branding over it, builds, and deploys.

```
overlay/client/config.json   the deploy's branding (this is the interesting file)
overlay/client/img/          icons, once they exist — anything here replaces Seance's
upstream.ref                 which Seance to build: a branch, tag or SHA
.github/workflows/pages.yml  build and publish
```

## How a build works

1. `upstream.ref` names a ref of `evilnet/seance` (`develop` today).
2. The workflow checks that out and copies `overlay/` over it, so `overlay/client/config.json`
   becomes `client/config.json` and any file under `overlay/client/img/` replaces the one Seance
   ships.
3. `yarn build` writes `public/`, which is deployed to Pages.

There is no fork and no merge. **Taking an upstream fix is a one-line change to `upstream.ref`**,
and there is never a conflict to resolve, because nothing here is a modified copy of anything
upstream. The cost of that is the other side of the same coin: AfterNET-specific *code* cannot
live here. Anything beyond configuration and assets is a pull request to
[evilnet/seance](https://github.com/evilnet/seance).

Every push to `main` rebuilds and republishes. So does publishing a release, and
**Actions → Pages → Run workflow**, which also takes a one-off ref if you want to try an
unmerged branch without committing to it.

The build fails on purpose if the branding did not reach `public/` — a copy that silently misses
would otherwise publish an unbranded client from a green build.

## Changing the branding

Edit `overlay/client/config.json` and push. The schema, every field and its default are documented
in [`docs/resources/branding.md`](https://github.com/evilnet/seance/blob/develop/docs/resources/branding.md)
upstream. What this deploy sets, and why:

| Setting                                              | Effect                                                                                                  |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `defaultNetwork` + `lockHost`, `allowCustomServer: false` | The client only ever connects to AfterNET. Server fields are not shown, and a link to any other host is refused. |
| `multiNetwork: false`                                | No "add another network" UI: this is AfterNET's client, not a general one.                              |
| `features.signIn: true`                              | The first screen is a sign-in panel — account and password, or a nick and "come in as a guest".        |
| `features.guestAccess: true`                         | Keep the guest option. Set `false` to require an account.                                               |
| `theme: "cobalt"`                                    | The blue theme ("Cobalt Frost", dark). `frost` is its light twin; users can still pick another in Settings. |
| `strings.*`                                          | The sign-in copy. Only the listed keys are overridable.                                                 |

An account and password typed into the panel are sent as **SASL PLAIN**, and the account becomes
the nick. "Stay signed in on this device" stores the password in that browser's `localStorage` and
connects automatically next time; without it, nothing is kept and the panel appears again. A
guest is never remembered.

If AfterNET's services refuse a login, the connection is **dropped** rather than continuing as an
unauthenticated stranger (`features.saslDisconnectOnFail`, on by default), the lobby says why, and
the entry stops connecting by itself so the next visit asks again.

## Before this can go live

- [x] **The WebSocket endpoint** is `wss://fractalrealities.afternet.org:9998/` (checked 2026-09-07:
      answers the WS upgrade with the `text.ircv3.net` subprotocol and advertises `sasl=PLAIN`,
      `draft/persistence`, `draft/webpush`, `draft/chathistory` and `draft/multiline`). Pages is
      HTTPS, so it has to stay `wss://`. Re-check any change with
      `node tools/irc-ws-probe.mjs wss://host:port/ somenick` from a Seance checkout.
- [x] **Icons.** All of them are AfterNET's, rendered from `overlay/client/img/logo-art.svg` — see Icons below.
- [x] **The upstream features.** #40 and #41 are merged; `upstream.ref` is `develop`.
      ~~ `theme: "cobalt"` and `features.signIn` need
      [evilnet/seance#40](https://github.com/evilnet/seance/pull/40) and
      [#41](https://github.com/evilnet/seance/pull/41) merged. Until then the build succeeds but
      falls back to the default theme and the ordinary connect form. (The workflow's branding check
      will fail on the missing `cobalt.css`, which is the point of it.)
- [x] **Uploads** go to Litterbox (`catbox-litterbox` preset), the same as upstream: anonymous,
      files expire after 72 hours. Point `uploads` at AfterNET's own endpoint to change that.

## Icons

Every icon is rendered from **`overlay/client/img/logo-art.png`**: the AfterNET mark cut out of the
original `AfterNETLogoBig.psd` (Photoshop's own flattened render, with its bevel, at 601px on
transparency — the background layer flood-filled away and the alpha eroded four pixels to drop the
drop shadow). The 100×100 GIF everyone has is that same artwork downscaled. To regenerate the set
from it:

```sh
M=logo-art.png
for s in 192 512; do convert $M -resize ${s}x${s} -background none -gravity center -extent ${s}x${s} PNG32:icon-$s.png; done
convert $M -resize 256x256 -background none -gravity center -extent 256x256 PNG32:logo-tile.png
for s in 192 512; do m=$((s*80/100)); convert -size ${s}x${s} xc:'#101720' \( $M -resize ${m}x${m} \) -gravity center -composite icon-maskable-$s.png; done
for s in 120 152 167 180; do m=$((s*88/100)); convert -size ${s}x${s} xc:'#101720' \( $M -resize ${m}x${m} \) -gravity center -composite apple-touch-icon-$s.png; done
convert $M \( -clone 0 -resize 16x16 \) \( -clone 0 -resize 32x32 \) \( -clone 0 -resize 48x48 \) -delete 0 ../favicon.ico
```

Maskable icons keep the art inside the central 80% on the theme's navy so Android can round or
circle-crop them; the Apple ones are opaque on the same navy, since iOS composites alpha on black.
Which file is used where is in the branding doc under "Files a rebranded deploy overwrites".

## Building it locally

```sh
git clone https://github.com/evilnet/seance.git
cd seance && git checkout "$(cat ../seance-afternet/upstream.ref)"
cp -R ../seance-afternet/overlay/. .
corepack yarn && corepack yarn build
python3 -m http.server -d public 8000
```

Point `defaultNetwork` at a local ircd while testing: `tools/nefarious-dev/run.sh` in the Seance
checkout runs one in Docker, on `ws://127.0.0.1:8067` (plain, fine for `http://localhost`).
