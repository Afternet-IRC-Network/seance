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
- [x] **Icons, the small ones.** The sidebar logo and both favicons are AfterNET's, made from the
      100×100 GIF, which is enough for those sizes. The installed-app icons (`icon-192/512`, the
      maskable and Apple ones) are still Seance's — see Icons below.
- [x] **The upstream features.** #40 and #41 are merged; `upstream.ref` is `develop`.
      ~~ `theme: "cobalt"` and `features.signIn` need
      [evilnet/seance#40](https://github.com/evilnet/seance/pull/40) and
      [#41](https://github.com/evilnet/seance/pull/41) merged. Until then the build succeeds but
      falls back to the default theme and the ordinary connect form. (The workflow's branding check
      will fail on the missing `cobalt.css`, which is the point of it.)
- [ ] **Uploads.** `uploads` is not set, so file sharing is off. Adding it points the paperclip at
      an uploader AfterNET runs (or at a third-party one); see the branding doc's Uploads section
      for the contract, which is mostly about CORS.

## Icons

Drop replacements under `overlay/client/img/` (and `overlay/client/favicon.ico`) and they take the
place of Seance's. The full list, and what each one is for, is in the branding doc under
"Files a rebranded deploy overwrites". The sizes that matter:

| File                                     | Used for                                              |
| ---------------------------------------- | ------------------------------------------------------- |
| `favicon.ico`, `img/favicon-alerted.ico` | Browser tab, normal and the unread-highlight variant  |
| `img/icon-192.png`, `img/icon-512.png`   | Installed app, and the notification icon              |
| `img/icon-maskable-*.png`                | Android/ChromeOS, artwork inside the central 80%      |
| `img/apple-touch-icon-*.png`             | iOS home screen — must be opaque, iOS ignores alpha   |
| `img/logo-tile.png`                      | The sidebar logo and the loading splash               |

The AfterNET mark we have is a **100×100 GIF from 2008**, which is enough for a favicon and
nothing larger: a 512px icon from it would be a five-fold upscale, and the install prompt shows
that icon at full size. The mark is simple geometry — a blue ring and a wordmark — so the right
fix is to redraw it once as SVG and render every size from that.

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
