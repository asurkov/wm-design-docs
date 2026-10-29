# The extension API surface — working notes

Companion to [migration-overview](./migration-overview.md).

This document describes `chrome.monetization`, the API the Web Monetization extension uses to hand its
work over to Chrome, one step at a time. It also covers the calls behind Chrome's own
`chrome://monetization` page: connecting and disconnecting a wallet, the session history, and clearing
data. The page itself is described in TBD migration-user-control.

The part of the API that extensions can use is specified in the [Web Monetization Extensions API][spec],
a WICG draft.

Chrome turns the API on in two stages. First Chrome tracks the payment sessions and tells the extension,
which still makes the payments: that part ships with the observe feature. Then Chrome holds the wallet and
pays: that part ships with the payments feature ([migration-gating][gating-rungs]). Each
section says which of the two it ships with and who may call it ([Access models](#access-models)), and
ends with the questions still open about it. Everything is tested with `ExtensionApiTest` browser tests,
because web platform tests can't reach extension APIs.

---

# Prerequisites

These have to be in place before any of the API reaches users.

- **The permission.** An extension lists `"monetization"` in its manifest to use `chrome.monetization`.
  The permission becomes available on the stable channel with observe's first opt-in
  ([migration-gating: User opt-in](./migration-gating.md#user-opt-in)).
  The permission alone turns nothing on: each method and event also waits for its feature.
- **No install warning.** When the extension's update adds `"monetization"`, Chrome checks whether the
  new permission comes with an install warning. If it does, Chrome turns the extension off until each
  user approves it again. Many users won't, and the hand-over stalls
  ([migration-overview: Risks](./migration-overview.md#risks)).
  So `monetization` has no warning, and the CL that defines it says so by listing it among the
  permissions that need no prompt (`skip` in `PermissionsTest.PermissionMessages`).
- **Tied to the link.** `chrome.monetization` exists only while the `Monetization` feature, which turns
  on the `<link>`, is on. The first CL adds that check to every part of the API
  ([migration-gating: Notes](./migration-gating.md#notes)).

**Open questions**

- Whether Chrome's launch review accepts a permission with no warning. If it asks for one, we ask for it
  to be covered by the "all sites" warning the extension's users have already accepted, as `tabs` and
  `webNavigation` are.

---

# Observing sessions

*This is Step 2 — observe in the [overview][step-2].* Chrome tells the extension when a session starts,
stops, pauses or resumes, and fires the page's events itself. The extension no longer needs to look into
pages, so it drops its content script and its access to all sites. It still makes the payments
([migration-native-vs-extension: Web Extension changes needed per step][nve-steps]).

## Sessions

These tell the extension which pages are being monetized, now that it no longer watches them. They are
in the spec ([Events][spec-events], [`Session`][spec-types]), ship with observe, and any extension with
the `monetization` permission can use them.

- `onStarted(session)` — a session started. It fires once the session is set up: the site's policy
  allows it, the user has granted permission, and the wallet address has loaded. Chrome needs no wallet
  of its own for this.
- `onStopped(session)` — the session ended: the tab closed, the page navigated away, or the link stopped.
- `onPaused(session)` / `onResumed(session)` — *proposed.* Something the session depends on went away or
  came back, such as the page being hidden or shown, or its window losing or regaining focus. The
  session keeps its `sessionId`, and `session.active` says whether it is running.
- `getSessions({tabId})` — *proposed.* Returns the sessions running right now, for the extension's
  background worker, which loses what it collected from events whenever Chrome stops it.
- `Session` — `tabId`, `frameId`, `sessionId` and `walletAddress`, one per link.

A site the user blocked has no session.

**Test:** each link gives one `onStarted` and then one `onStopped`; closing the tab, navigating away and
removing the link each give `onStopped`; hiding and showing the tab give `onPaused` and `onResumed`; a
session that starts in a background tab starts paused.

**Open questions**

- What exactly a session is, and the type of its `sessionId` ([#655][i655]).
- Adding `getSessions` ([#655][i655]).
- Pause and resume, and `session.active` ([#762][i762]).
- Whether the permission becomes one of the session's conditions ([#741][i741]). Then a blocked site
  has a paused session instead of none.

## Payment reports

The extension still makes the payments at this step, but without its content script it can't fire
`MonetizationEvent` on the page. Instead it tells Chrome about each payment, and Chrome fires the event.
Together with the link and sessions, this is the first state we can ship, the one the link's
Intent-to-Ship covers ([migration-overview: The first shippable state][first-shippable];
[#1484](https://github.com/interledger/web-monetization-extension/issues/1484)). It ships with observe,
and only the Web Monetization extension can call it.

- `reportPayment(sessionId, amountSent, incomingPayment)` — fires `MonetizationEvent` with these values
  on the session's link ([migration-monetization-link: MonetizationEvent][link-event]). It fails if the
  session is unknown or in another frame, if the caller isn't the Web Monetization extension, or if
  Chrome is already making the payments itself.

Only that one extension may call it, because no other extension should be able to tell a page that money
moved. It is removed in the payments feature's cleanup CL; until then, switching payments off hands
paying back to the extension, which needs it again.

**Test:** a report for a running session fires the event on its link; each failure above returns an
error.

**Open questions**

- Private or public. We allow only the Web Monetization extension; Rick Byers'
  [provider-extension sketch][rick-sketch] makes it public for any extension that acts as a monetization
  provider.
- Kept out of the spec, as a private method in the Chromium proposal ([#723][i723]).

## Observe's proposal

The first API proposal covers everything under Sessions, including the proposed `onPaused`, `onResumed`
and `getSessions`, plus `reportPayment` as a private method only the Web Monetization extension can call.
Its launch review also settles the permission's warning.

---

# Native payments

*This is Step 3 — payments in the [overview][step-3].* Chrome holds the user's wallet and makes the
payments. The extension keeps its controls and UI and drives them through the API
([migration-native-vs-extension: Web Extension changes needed per step][nve-steps]).

## Payment events

- `onPayment(session)` — Chrome made a payment for this session; `getWallet` returns the new balance.
  It's in the spec ([`onPayment`][spec-events]), ships with payments, and any extension with the
  `monetization` permission can listen to it.

Because `onPayment` exists only while payments is on, the extension checks for it to learn whether
Chrome makes the payments
([#1487](https://github.com/interledger/web-monetization-extension/issues/1487);
[migration-native-vs-extension: Prerequisite for every hand-over][nve-prereq]).

**Test:** each payment Chrome makes fires `onPayment` for its session, and `getWallet` in the handler
already returns the new balance; with payments off, `onPayment` is absent.

## Wallet

All of this ships with payments. How Chrome holds the wallet and its keys is in
TBD migration-payment-engine: Keys and Open Payments ownership.

- `getWallet()` → `Wallet?` — the connected wallet and its balance, or `null` if there is none. It's in
  the spec ([`getWallet()`][spec-interfaces]); extensions with the permission and the
  `chrome://monetization` page can call it.
- `connectWallet({address, budget, renewMonthly})` → public key — starts connecting a wallet and returns
  the key to register with it.
- `finalizeWalletConnection()` → `Wallet` — finishes the connection once the user approves it at the
  wallet; fails with `tab_closed` if the user closes the consent tab
  (TBD migration-payment-engine: Wallet connection).
- `disconnectWallet()` — disconnects the wallet.

**Test:** `getWallet` returns `null` before a connection and after `disconnectWallet`, and the wallet in
between; closing the consent tab fails `finalizeWalletConnection` with `tab_closed`.

**Open questions**

- Whether extensions may call the three connection methods, or only the `chrome://monetization` page
  (TBD migration-user-control: The open decision;
  [#1485](https://github.com/interledger/web-monetization-extension/issues/1485)). Settled before the
  payments proposal.

## Controls

These keep the extension's controls working once Chrome pays: Workshop I kept the controls in the
extension, going through the API ([Workshop I notes](https://docs.google.com/document/d/10ZCMIYPgnwujes7Wb-v2yB9RLekvBYwmRULzl9NG0iE/edit)). They ship with
payments, and extensions with the permission can call them.

- rate — sets how much Chrome pays over time.
- per-site rate — sets a different rate for one site.
- `pay` — makes a one-time payment to a site.
- `enable` / `disable` — turns paying on or off everywhere.
- `updateBalance` — adds funds to the wallet's grant.

Anything that starts spending, or spends more, needs a user gesture (`ExtensionFunction::user_gesture()`).
A click in the popup counts; `disable` needs none.

**Test:** after a rate or per-site rate change, the payments that follow match it; `pay` makes one
payment; `disable` stops payments and `enable` restarts them; `updateBalance` raises the balance
`getWallet` returns; a call that starts or raises spending fails without a user gesture, and `disable`
succeeds without one.

**Open questions**

- Which controls, and their names ([#723][i723]).
- What the rate method sets: an hourly rate, as the extension has, or a monthly budget, as Chrome's
  engine paces (TBD migration-payment-engine: Rate and amount contract).
- Per-site rate as data on the site's permission
  ([#740](https://github.com/WICG/webmonetization/pull/740)) or as a method here.
- Who may call `updateBalance`: it follows the decision on the wallet's connection methods.

## Site permission

This lets the popup show and change a site's permission: the per-site version of `enable` / `disable`.
The permission itself is in TBD migration-permission. What belongs here is the
extension's access to it: a new `monetization` entry in `chrome.contentSettings`, Chromium's existing
extension API for per-site settings (`content_settings.json`), backed by
`ContentSettingsType::MONETIZATION` and reviewed by the extension API owners. It ships with payments, behind a `feature_flag` on
that entry as `clipboard` does, and extensions with the `contentSettings` permission can use it.

- `get` — whether the site is allowed, blocked or set to ask.
- `set` / `clear` — allows or blocks one site, as extensions already can for `camera`.

**Test:** `get` matches Site Settings; a site blocked through `set` gets no session.

**Open questions**

- Whether the popup needs to change a site's permission at all, or links to Site Settings instead. The
  API needs the `contentSettings` permission, which has an install warning: added to the manifest, it
  turns the extension off until each user approves it again; as an optional permission, the user is
  asked when the popup first uses it.

## Payments' proposal

The second proposal covers `onPayment`, `getWallet` and the controls, plus `contentSettings.monetization`
if the popup needs it and the wallet connection methods if extensions may call them. The final set of controls has to be settled
before payments ships: once something reaches the stable channel without a gate, it can't be changed.

---

# The WebUI page

*This is Step 4 — WebUI in the [overview][step-4].* These are only for Chrome's own
`chrome://monetization` page, and ship with payments.

- `querySessions(searchTerm)` — returns the history of past sessions.
- `clearData()` — clears the stored monetization data.

**Test:** an extension can't call either, even with the `monetization` permission.

**Open questions**

- Moving the page onto a Mojo page handler, which Android needs. That takes these methods and the
  wallet connection methods out of `chrome.monetization`
  (TBD migration-user-control).

---

# Access models

Who may call a method is set in `_api_features.json`. There are three options:

| Model | Used for | Notes |
|---|---|---|
| A. An allowlist of extension IDs | `reportPayment` | Chrome does this only for its own and partners' extensions; `new_api_proposal.md` prefers public APIs |
| B. Public, for any extension with the `monetization` permission | sessions, payment events, the controls (with a user gesture) | depends on the WebExtensions Community Group accepting a third-party extension that controls payments |
| C. Only the `chrome://monetization` page | the wallet connection methods, unless extensions may call them | no third party to trust, but the extension then can't offer those controls (TBD migration-user-control) |

---

# Approval

New extension APIs follow `extensions/docs/new_api_proposal.md`. As outside proposers, Igalia and
Interledger open an issue in the WebExtensions Community Group, find a sponsoring browser through
`extension-api-reviews@chromium.org`, write a formal proposal the browser vendors agree on, and then go
through Chrome's launch reviews, which take four to six weeks. Code lands only after the proposal is
approved ([migration-gating: Off](./migration-gating.md#off)).

[spec]: https://webmonetization.org/specification/extensions-api/
[spec-events]: https://webmonetization.org/specification/extensions-api/#events
[spec-types]: https://webmonetization.org/specification/extensions-api/#types
[spec-interfaces]: https://webmonetization.org/specification/extensions-api/#interfaces
[gating-rungs]: ./migration-gating.md#which-rungs-a-feature-takes
[link-event]: ./migration-monetization-link.md#monetizationevent
[nve-steps]: ./migration-native-vs-extension.md#web-extension-changes-needed-per-step
[nve-prereq]: ./migration-native-vs-extension.md#prerequisite-for-every-hand-over
[first-shippable]: ./migration-overview.md#the-first-shippable-state
[rick-sketch]: https://docs.google.com/document/d/1oFYCe4nHlJyktxIv44jPC5mgqQLVzs1CeK05sWSUM7k/edit
[i655]: https://github.com/WICG/webmonetization/issues/655
[i723]: https://github.com/WICG/webmonetization/issues/723
[i741]: https://github.com/WICG/webmonetization/pull/741
[i762]: https://github.com/WICG/webmonetization/issues/762
[step-2]: ./migration-overview.md#step-2--observe
[step-3]: ./migration-overview.md#step-3--payments
[step-4]: ./migration-overview.md#step-4--webui
