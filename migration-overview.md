# Web Monetization: Chromium Rollout Plan

**Prototype** [CL 4930834](https://chromium-review.googlesource.com/c/chromium/src/+/4930834) · **Spec**
[webmonetization.org/specification](https://webmonetization.org/specification/) · **Tracking bug**
[crbug/1031476](https://crbug.com/1031476) · **Chrome Status**
[Web Monetization](https://chromestatus.com/feature/4917148755689472) · **Extension**
[interledger/web-monetization-extension](https://github.com/interledger/web-monetization-extension)

**Goal:** upstream the Web Monetization prototype and move the WM extension's users onto it with no
change in behaviour — sites keep getting paid, at the same rate, through every hand-over. The extension
itself changes plenty: it ends as UI over Chrome's API, implementing none of the behaviour itself.

The prototype implements the whole path in one CL: `<link rel=monetization>` handling, the events, an
Open Payments engine, `chrome://monetization`. Upstreaming splits that into **steps** behind features:
`Monetization` first, which everything else depends on, then a feature per job that must move on its own
clock ([migration-gating: The features](./migration-gating.md#the-features)), so any
capability can be enabled — or switched back off — without touching the others. Each step needs a
different approval — a Blink launch for the monetization link, internal review for the payment engine —
so they run in parallel rather than in sequence.

The extension yields before a feature is enabled, not after. It ships a version that detects each native
capability and stands down from that job, and we enable the flag only once that version has reached
users. What it has to build to do that, and where the two implementations diverge, is in
[migration-native-vs-extension](./migration-native-vs-extension.md).

---

# Surfaces

A **surface** is the work grouped by who approves it: a web-exposed change goes to Blink API owners, a
new extension API through the Extensions API proposal process. One capability can need work on several
surfaces at once. Each entry names what the surface covers and who signs it off.

## Monetization link

- **What:** `<link rel=monetization>` handling and its events, plus the Permissions-Policy
  `monetization` token — the site's own declaration of which frames may use monetization at all, checked
  before any user grant is consulted.
- **Approval:** Blink API owners, 3 LGTMs — the full Blink launch: Intent-to-Prototype, Experiment,
  Ship, plus TAG review, a spec, WPT, and cross-vendor signals, and the ChromeStatus Privacy and
  Security gates (Chrome Privacy, WP Security), reviewed at Experiment and again at Ship before the
  owners look. Irreversible once shipped, as anything Blink ships is.
- **Detail:** [migration-monetization-link](./migration-monetization-link.md).

## CSP directive

- **What:** the `monetization-src` directive — parsing and enforcement in the renderer and the network
  service. Split from the monetization link because it answers to different people: CSP has owners of its
  own, and its feature is declared in the network namespace, not Blink's.
- **Approval:** CSP owners for the implementation, Blink API owners for the Intent chain, on its own
  timeline rather than holding the monetization-link launch. Blocked on spec text in CSP, which W3C
  WebAppSec owns, and on a Fetch destination, which WHATWG owns.
- **Detail:** TBD migration-csp.

## Extension API

- **What:** `chrome.monetization` — session observation (`onStarted`/`onStopped`, `onPayment`), and
  methods for connecting and managing a wallet.
- **Approval:** the Extensions API proposal process — a WECG proposal with consensus from browser
  vendors, then a launch bug through Security, Privacy, Legal, UX, the Extensions team and DevRel.
  Cheaper than a Blink launch and retractable, with no spec, TAG or WPT, but it needs cross-vendor
  agreement and is more than an `OWNERS` review.
- **Detail:** TBD migration-extension-api.

## Permission

- **What:** the user's per-site grant — the `MONETIZATION` content setting, its prompt and its
  site-settings entry — plus `navigator.permissions.query({name: "monetization"})`, which is how a page
  reads that grant. Enforcing it at session start lands with observe. Same shape as location or
  camera. The extension's global on/off is Pause-all, not this surface.
- **Approval:** Permissions owners, with privacy and security, notifying `permissions-core@`. A
  permission is always a `PermissionType`/`ContentSettingsType` pair: the content setting and its UI are
  internal, while the queryable name is web-exposed and ships under the monetization link's Intent, gated
  on `Monetization`, as a name added alongside a new API does — provided the spec's Permissions API
  integration is written before that Intent-to-Ship ([Risks](#risks)).
- **Detail:** TBD migration-permission.

## Payment engine

- **What:** everything that moves money — keys, credentials, the Open Payments client, wallet
  connection, and the paying loop. The API that drives it is the extension API surface. Grant
  management (top-up, one-time grant vs recurring-only) is the same control open as `addFunds`
  (TBD migration-extension-api).
- **Approval:** internal Chrome code, but not an ordinary owners review — its Mojo interface is pinned
  to IPC security owners, so the usual chain cannot approve it alone.
- **Detail:** TBD migration-payment-engine.

## User control

- **What:** the control UI — `chrome://monetization`: wallet connection, budget and rate management,
  session history.
- **Approval:** Chrome product and UX, with security and privacy. Internal only; no external body.
- **Detail:** TBD migration-user-control.

---

# Reaching users

Every feature climbs the same ladder from off to on by default, rung by rung
([migration-gating: The ladder](./migration-gating.md#the-ladder)); what differs is what has to be true
before it can reach users, and since each moves on its own schedule the steps overlap — one can be at
opt-in while the next is still merging CLs. Two rungs carry the plan's claims. User opt-in is where
existing WM extension users move onto the native path, ahead of any wider rollout, through one
`chrome://flags` entry for the WM features at opt-in — not internal testing. Shipped is Intent-to-Ship
for anything web-exposed: nothing web-exposed reaches stable users ahead of its approval, and the
monetization link takes an origin trial first so sites can try it.

Enabling a flag is necessary, not sufficient. Who may *call* `chrome.monetization` is gated separately, by
channel in `_permission_features.json` and by context in `_api_features.json`, and nothing happens on any
path until someone connects a wallet.

What has to be true before a surface can reach users:

| Surface | Prerequisite |
|---|---|
| Monetization link | the extension's update rollout — the yielding version has to reach users first ([Risks](#risks)) |
| CSP directive | spec text in CSP (WebAppSec) and a Fetch destination (WHATWG) ([Risks](#risks)) |
| Extension API, payment engine | spending behaviour agreed before payments hands over |
| Permission | none; rides the observe step |
| User control | none, but control stays WebUI-only until the open decision settles |

---

# Steps

The milestones, in the order the capability stacks up: the browser processes the link, and sessions,
events and payments all start from there. Each step is a capability Chrome takes on; where the extension
does that job today, it stands down once it can see Chrome doing it, picking things up as they land
rather than matching the implementation.

A step can span surfaces: payments is engine work, the `chrome.monetization` methods that trigger it, and
firing `MonetizationEvent`. Each CL is still reviewed by the surface it touches.

What the extension must change at each step is in
[migration-native-vs-extension: Web Extension changes needed per step](./migration-native-vs-extension.md#web-extension-changes-needed-per-step).

## Step 1 — monetization link

Chrome serves the web-exposed API, and the extension stops shimming it: `relList.supports('monetization')` answers true, the `<link>`'s `href` resolves, `load` and `error` fire on that resolution, and the site's Permissions-Policy token is honoured.

It goes first and finishes last. This is the only step whose approval reaches outside Google, so its clock is the longest in the plan. The **Intent-to-Prototype** is filed; nothing after it has moved, and the entry records no TAG review, no signal from Firefox or WebKit, and no WPT. Little depends on it — Step 2 waits only on the link restart — so it runs alongside the others rather than blocking them.

It does not finish the surface. `MonetizationEvent` ships here as an interface that fires nothing, since Chrome cannot pay until Step 3 — and it has to ship constructible, because until then the extension goes on firing those events itself. The link's lifecycle is the one piece with no complete spec text to point at, which a reviewer may raise; [#724](https://github.com/WICG/webmonetization/pull/724) is in review for [#698](https://github.com/WICG/webmonetization/issues/698).

`onmonetization` belongs to this step and ships under `Monetization` and its Intent, as a CL of its own. Its spec IDL landed in [PR 697](https://github.com/WICG/webmonetization/pull/697); Chromium has neither handler form, so what remains is implementation.

## Step 2 — observe

Chrome owns the session lifecycle. `chrome.monetization` gets `onStarted` and `onStopped`, so the
extension stops watching the DOM for links and observes Chrome's sessions instead. The
`MONETIZATION` content setting is enforced at session start — the same per-origin prompt as
location or camera (TBD migration-permission). The extension's
global on/off is Pause-all, not this grant.

Adding the API is a proposal with sign-offs, not an owners review, and no external body — but the
`MonetizationEvent` firing its flip starts is declared in the monetization link's Intent-to-Ship, so
observe ships behind Step 1 and reaches extension users first through the opt-in entry
([The first shippable state](#the-first-shippable-state)). It waits on Step 1 for one more thing: native
has to restart a session when a link's attributes change, or the extension cannot safely stop watching.

The extension still pays here — a session starting is not money moving — and reports each payment
through `chrome.monetization.reportPayment`, so Chrome fires `MonetizationEvent` and the content script
can go ([The first shippable state](#the-first-shippable-state)).

## Step 3 — payments

Chrome takes over everything that touches money. Wallet management first, then the paying loop.

**Wallet management.** Keys, connection, and the Open Payments grant.

- **Wallet methods.** `chrome.monetization` gains `connectWallet`, `finalizeWalletConnection`,
  `disconnectWallet` — WebUI-only today. `getWallet`, already reachable from extensions, arrives
  with observe. Who may call them is the plan's open decision, binding here, not at Step 4: only
  `chrome://monetization`, or the extension API too, probably behind a user activation requirement.
- **Grants.** Top-up, one-time grant, and budget change ride that same decision
  (TBD migration-payment-engine: Grants).
- **Rate.** The extension meters an hourly rate; native meters a monthly budget target.
  Those are different promises, not a unit mismatch
  (TBD migration-payment-engine: Rate and amount contract).

**Payment streaming.** The paying loop and `onPayment`. The extension retires its engine and stops
reporting payments. Who decides how much and how often is still open: if the meter stays
extension-side, the API has to expose `pay()` — which one-time payments also need.

The extension stops paying on `if (chrome.monetization.onPayment)` — existence, not a firing.
`relList.supports` already answered Step 1. The event stays absent until streaming
([migration-native-vs-extension](./migration-native-vs-extension.md)).

Internal review; the Mojo interface is pinned to IPC security owners, and the wallet methods re-enter
the extension API proposal.

## Step 4 — WebUI

Chrome has its own control UI at `chrome://monetization` — the only one on Android, and the only one for
users without the extension.

The decision taken at Step 3 settles whether it is *also* the only place a user can connect a wallet or
change a budget or rate — the operations that authorize or change spending. If it is, the extension's UI
can show status but cannot offer those controls, and has to send the user here for them. Until that
settles the extension keeps its UI as it is, which makes this the one step where it may give up nothing
at all.

Internal review; no external body.

## No step — the CSP directive

Nothing hands over: an extension cannot put `monetization-src` in the browser's CSP table, and it
fetches wallet addresses from its own context, so page CSP never sees the request. Sites gain it
rather than change providers. It gates the same fetch whose outcome Step 1 reports as `load` or `error`, but it
waits on no step and holds none up. Spec PRs to Fetch and CSP, and disabled Chromium CLs, can start
now; enabling waits on WebAppSec and WHATWG
(TBD migration-csp).

---

# The first shippable state

Everything lands off, gated by code OWNERS alone — which is also the risk: a path we go far down and are
never approved to enable. The loss is bounded on the user's side and capped on ours. Users lose nothing:
the extension yields only after a feature is enabled, so a feature that never ships leaves the extension
doing that job, as today. What is lost is landed code — Chromium's guidance for a feature that is not
stable and no longer under active development is to drop its status and consider deleting it
([RuntimeEnabledFeatures.md](https://chromium.googlesource.com/chromium/src/+/main/third_party/blink/renderer/platform/RuntimeEnabledFeatures.md#Guidelines-for-Setting-Feature-Status))
— so what lands ahead of approval signals stops at one state.

That state is Step 1 and Step 2 plus one call, `chrome.monetization.reportPayment`: the extension tells
Chrome about a payment it made, and Chrome fires `MonetizationEvent` for it
([#723](https://github.com/WICG/webmonetization/issues/723) on the spec side,
[#1484](https://github.com/interledger/web-monetization-extension/issues/1484) on the extension's). The
call is transitional — allowlisted to the WM extension, not public, since any caller could otherwise
have Chrome tell a page that money moved — and goes in payments' cleanup CL. The allowlist is one
hashed-ID line in `_api_features.json`; the cost is the extension's: an unpacked build shares the store
ID only with the package's `key` in its manifest
(TBD migration-extension-api: reportPayment and transitional members).
After it the extension has no job on the page — Chrome handles the link, tracks sessions and fires the
events — so the content script and the all-sites host permission go, and with them the install warning
for access to every site. That is one manifest change, the same for every user: the Chrome build makes
the hosts optional and registers the content script only where the Step 2 probe fails. A new install
sees no warning, an existing grant survives the update, and a user on a Chrome without observe grants
the same access once, from the popup
([migration-native-vs-extension: Web Extension changes needed per step](./migration-native-vs-extension.md#web-extension-changes-needed-per-step)).
Key auto-add keeps injecting only on the wallet-provider hosts it already
registers. Payments and the control UI stay in the extension, which keeps Step 3's spending decision and
Step 4's control-plane decision off the path. Enabling Chrome's payments first would lose both gains: the
extension would still need its content script to find the links, and the spending decision would come
first instead of last.

Its gate is the monetization link's Intent-to-Ship, the one external approval on the path, since the
Blink half is web-exposed; the extension-API half is internal and retractable. Before that approval the
state reaches extension users through the opt-in entry
([migration-gating: The ladder](./migration-gating.md#the-ladder)). The signals that Intent needs — TAG
review, WPT, vendor positions — are what Step 1 lacks today, and they come before the payment engine
lands.

---

# Risks

Things with no owner or no lever. The technical gaps are in the surface documents.

**The launch can stall on a body we do not control.** Blink will not ship a web-exposed API on spec text
that has not landed, and some of this plan's text is not ours to land: the WM spec marks its whole
"Link type" section a monkey patch for HTML — which covers the link keyword, the steps that fire `load`
and `error`, and the link's lifecycle — and marks its CSP section as destined for the CSP and Fetch
specifications. We can write and submit; WHATWG and W3C WebAppSec decide, on their own schedule,
and no escalation inside Chromium moves them. That text is not merely unwritten: what `load` reports is
contested in [#344](https://github.com/WICG/webmonetization/issues/344), where the events fire for a
`Link:` header in [#343](https://github.com/WICG/webmonetization/issues/343) — both open since 2023 — and
the lifecycle in [#698](https://github.com/WICG/webmonetization/issues/698), with
[#724](https://github.com/WICG/webmonetization/pull/724) in review. The piece that is ours —
the Permissions API integration — carries no such exposure and is in review:
[#741](https://github.com/WICG/webmonetization/pull/741) defines the permission and
[#740](https://github.com/WICG/webmonetization/pull/740) adds a per-site rate to it, both for
[#713](https://github.com/WICG/webmonetization/issues/713). The
`onmonetization` IDL was the other, and [PR 697](https://github.com/WICG/webmonetization/pull/697) has
landed it.

**The extension's update rollout gates the enabling of every step, and we cannot speed it up.** Enable
before the yielding version has reached an install and that extension keeps doing the job native has just
taken over: with today's release a monetized page gets `load` and `error` twice and no `monetization`
event at all
([migration-native-vs-extension: Prerequisite for every hand-over](./migration-native-vs-extension.md#prerequisite-for-every-hand-over)).
The Web Store paces the rollout, nobody has set a threshold for how many installs is enough, and nothing
measures it — users who never update are a tail with no end. The rollout is silent only while the
yielding version adds no permission warning: if the `monetization` permission gets one, the update
switches the extension off until each user accepts
(TBD migration-extension-api: Approval).

# Out of scope

**Android — deferred.** Chrome for Android phones ships no extensions, so the migration half of this
document has nothing to attach to there: no dual path, no feature detection, no hand-over. The
upstreaming half applies to the engine, which has no desktop-only dependency, but not to user control:
`chrome://monetization` is built for desktop only and reaches the browser through the extension API's
WebUI context, which Android phones do not compile, and the permission prompt has no Android icon.
Moving the page onto a Mojo page handler is the one Android prerequisite the desktop plan takes on,
since it also removes the WebUI-only members from the extension API proposal
(TBD migration-user-control); the Android build of the page, its prompt
icon and Site Settings entry, an entry from Android Settings, key registration without an extension
and a visibility pause wait.

It is also the case with no extension in the picture at all, which makes it the clearest view of the end
state and of what a browser-only Web Monetization looks like. That is worth reading once the prototype is
upstreamed and the shape is visible, not designed around now.
