# The prototype and the extension, compared

Companion to [migration-overview](./migration-overview.md). The surface documents cover Chromium's
side one review body at a time. This one compares the two implementations and says what each difference
demands: where they agree there is nothing to do, where one side is missing something it gets built, and
where they diverge it gets settled — by a CL or by a conversation, noted with each. The surface
documents carry the code evidence.

The **prototype** is the unmerged Chromium implementation ([CL 4930834](https://chromium-review.googlesource.com/c/chromium/src/+/4930834)); where a point is
about what ships rather than what the CL does today, it says Chrome; "native" is the Chromium side in
either state.

It does everything in the browser: Blink handles the link, a browser-process Open Payments engine pays,
credentials live in an encrypted database, and `chrome://monetization` is the UI. The extension does the
same jobs from a MAIN-world polyfill, an isolated-world content script and an MV3 service worker, with
Ed25519 keys in `storage.local` and its own popup. Every difference below follows from that split.

---

# Where they agree

Feature detection — `relList.supports('monetization')`, which is how a page asks whether the browser
handles monetization links at all — and the `MonetizationEvent` a page receives on each payment. The
prototype exposes both, the extension polyfills both.

---

# Gaps

**Only the extension has it.** Each is built, or agreed away, before the hand-over that would retire it.

- **`MonetizationCurrencyAmount` on `amountSent`.** WebIDL forbids a dictionary as an attribute type, so
  the spec cannot switch to `PaymentCurrencyAmount`
  ([issue 501](https://github.com/interledger/web-monetization-extension/issues/501)). The prototype
  already exposes the interface (`monetization_currency_amount.idl`), Window-visible, no constructor. The
  polyfill landed it in [PR 1469](https://github.com/interledger/web-monetization-extension/pull/1469);
  the yielding version still has to reach users.
- **The `onmonetization` event handler**, which lets a page listen without `addEventListener` — as an
  inline `onmonetization="…"` attribute and as an IDL property. Build both; specified, unimplemented.
- **A `monetization` entry in `chrome.contentSettings`**, the conventional way an extension reads and
  sets a content setting, so the popup can show and change a site's permission. Either build it with
  payments, or agree the popup links to Site Settings instead
  ([migration-extension-api: Site permission](./migration-extension-api.md#site-permission)).
- **Automated key registration** — registering the browser's public key with the user's wallet without
  the user pasting it by hand. Either build it, or agree the extension keeps driving its own `keyAutoAdd`
  flow.
- **Structured error codes** from the browser's wallet flow, against the set the extension already models
  and shows the user. Build them.
- **Open Payments `quote.create`.** Once per session, before the first payment — not before
  every tick. The wallet is the only party that knows whether a debit is payable (fees and the
  path to the receiver). A quote asks that without moving or holding money; later outgoing
  payments send only a `debitAmount`, not the quote id. The number that comes back on a
  too-small quote is `minSendAmount` (`_findMinSendAmount` in `paymentSession.ts`), a floor in the sender's
  scale. The first payment is `max(rate-derived integer, minSendAmount)` (`start` in
  `paymentManager.ts`); later ticks pay in multiples of it (`getPayableAmount`), with no further quote.
  The quote itself is `createPaymentQuote` in `paymentSession.ts`. The prototype requests quote access on
  the grant (`OpenPaymentsFetcher::CreateOutgoingPaymentGrantBody`) but never creates one, so the first
  outgoing payment is the probe — and any failure other than a 403 with "Inactive Token" is
  `RevokeCredentials()`. Build it.
- **One-time pay.** The popup sends a user-chosen amount (`PAY_WEBSITE` → `payOneTime`). It spends an
  existing grant (prefers recurring), not a grant type. The prototype has no `pay` in `monetization.idl`
  and no WebUI form. Distinct from the streaming loop. Settled 2026-09-17: `pay` goes on the extension
  API, gesture-gated — the extension asks, the engine sends from the grant already approved; a WebUI
  form follows only if the page finds a place for one
  ([Workshop I notes](https://docs.google.com/document/d/10ZCMIYPgnwujes7Wb-v2yB9RLekvBYwmRULzl9NG0iE/edit);
  [migration-extension-api: Controls](./migration-extension-api.md#controls);
  TBD migration-payment-engine: Where the paying loop lives).
  Until `pay` exists, the extension disables the popup form once `if (chrome.monetization.onPayment)`
  is true.
- **One-time grant and top-up.** The extension stores `recurringGrant` and `oneTimeGrant`; out of
  funds, `addFunds` creates another grant on the same key — recurring replaces recurring, one-time
  can sit beside recurring. Not a third way to pay a site. The prototype is
  `connectWallet(..., renewMonthly)` only. Who drives top-up — WebUI or `addFunds` under the control
  open — is not settled ([migration-extension-api](./migration-extension-api.md)).
- **User-set rate after connect.** The extension edits `rateOfPay` in the popup; defaults come from
  fetched tables (`BUDGET_RECOMMENDATIONS_URL`, `EXCHANGE_RATES_URL`). The prototype sets budget at
  connect only (`monetization_connect_wallet_dialog.ts`: default `'10'`) and has no post-connect rate
  UI. Per-site overrides stay For discussion below.

**Only Chromium ships them.** They arrive new; nothing hands over for them.

- **Permissions-Policy `monetization`.** A site declares which frames may monetize. An extension
  could honour the token itself — parse the header and `allow` attributes, skip frames that are out
  — and has not: not prioritized, and not worth the work when native will do it.
- **CSP `monetization-src`.** Restricts where a wallet address may be fetched from. An extension
  cannot add a directive to Blink's table. The fetch is from the service worker, so page CSP never
  sees the request, and Chrome ignores a name it does not know.
- **Per-site permission** (`MONETIZATION` content setting). How a Chrome permission works —
  location, camera, this. The extension never had one: `enabled` is global, and a connected
  wallet pays every site. The prototype asks per origin (`MonetizationServiceImpl::CreateSession`).
  The extension-only control is a global stop — Pause-all below. Distinct from per-site rates. Detail:
  TBD migration-permission.

---

# Where they diverge

Both implement it, differently. Split by what would resolve each; the evidence is in the surface
documents.

**The prototype should follow the spec here.** The extension already does, so these are implementation
gaps rather than design choices. They land as ordinary CLs on the surfaces that own them.

- `load` and `error`. The prototype fires them only after a permission and valid credentials —
  leaking whether the visitor has a wallet. The extension fires them when the site's address
  resolves, wallet connected or not
  ([issue 648](https://github.com/interledger/web-monetization-extension/issues/648)). Whether
  `load` should mean "can pay" is the open spec thread
  ([#502](https://github.com/WICG/webmonetization/issues/502)); the plan keeps them
  visitor-invariant ([migration-monetization-link](./migration-monetization-link.md)).
- `MonetizationEvent.amountSent`. Always "how much the sender sent" in the **receiver's** currency.
  The extension reports Open Payments `receiveAmount` and is correct. The prototype is a bug: it
  takes the sender's debit number and labels it with the site's currency, with no conversion. Fine
  when both sides are USD; wrong when the user pays USD and the site is EUR.
- The prototype does not restart the session on attribute changes, and `disabled` leaves it running and
  paying; the extension restarts ([migration-monetization-link](./migration-monetization-link.md),
  Link lifecycle).

**For discussion.** Design choices, not bugs; each is settled in a session with the extension
developers, not by a CL. Rate, per-site rates, grants, cross-currency, visibility and pause-all have to be
settled before payments hands over; grant continuation and session identity do not. No review thread
will surface them, which is why they are the easiest items here to postpone indefinitely.

- **Rate.** The extension meters an hourly rate, the prototype a monthly budget target. Which promise
  should the user get? Multiplying hourly by hours in a 30-day month gives native a number that
  pays at the same per-second pace **while a session is open**. It is not a conversion of the
  promise: the user who set $0.60/hour meant spend-while-reading, not "empty $432 this month."
  Native only reaches that monthly total if a session runs the whole month. The extension
  spends the user's hourly rate (`rateOfPay` — set at connect, editable in the popup) for each
  hour a monetized page is open. Native takes a monthly number, divides by a 30-day month, and
  pays at that per-second rate while sessions exist. Users who already have `rateOfPay` still
  need a stored value for native at Step 3, or their spend changes when Chrome starts paying.
  Default connect values: what Chrome should suggest when a user connects a wallet, settled on what
  serves that user rather than on what either implementation already does. The extension fetches
  [budget-suggestions](https://github.com/interledger/web-monetization-budget-suggestions), the
  prototype hardcodes `'10'`. A formula, bundled data or no default at all are all open; depending on
  a third-party table needs its own justification, the same question as FX.
- **Interval clock.** When the next tick starts. The extension resets the timer when it kicks off
  the outgoing payment (`start` and `checkAndPayContinuously` in `paymentManager.ts`) — network time sits
  inside the period.
  The prototype reschedules only after the request finishes (`OnPaymentProcessed`,
  `monetization_service_impl.cc`) — period plus RTT. A slow wallet pays less often natively.
  Distinct from visibility pause: this is a live session, every tick.
- **Per-site rates.** Only the extension has them, keyed by hostname wildcard. Port them, or drop them
  and record that as a known regression?
- **Grants and top-up.** Who manages money once native holds the key: WebUI recurring-only, or
  `addFunds` / `updateBudget` so the extension chooses grant shape? Off the Step 3 minimum. User
  activation. Budget or currency change: the extension issues a new grant on the same key; native
  `disconnectWallet` + connect regenerates the key (`connectWallet` generates a fresh pair).
- **Cross-currency.** The prototype never compares the two assets. A mismatch can zero the receive
  amount and revoke the wallet. The extension's browser-side FX fetch (`EXCHANGE_RATES_URL`, plus a
  hardcoded MMAON rate) only seeds min-send and USD defaults; that is not a Chrome dependency.
  Detect and back off, or also port the FX seed?
- **Grant continuation.** The extension finishes by redirect, the prototype polls. Whether the wallets
  people use support polling is a fact to establish rather than a preference to settle.
- **Session identity.** `requestId` against `session_id`, both live during Step 2. What should the
  extension do when they disagree?
- **Pause-all.** The extension's global `enabled` / `TOGGLE_PAYMENTS` stops every site; native has no
  equivalent — only the per-origin content setting. WebUI, `chrome.monetization.disable()`, or hide
  the control and deep-link. Not the visibility pause below.
- **Visibility pause.** The extension pauses a hidden document's timer (`pause` in
  `paymentManager.ts`) and, on return or a same-page `reload()`, waits out the last interval
  (`preventOverpaying`, `paymentManager.ts`) — one payment per interval, however often the tab hides.
  The prototype has no visibility handling: nothing in its monetization code reads document visibility,
  and a session lives until its context is destroyed, so a hidden document keeps being paid. Whether
  Chrome pauses, and how it avoids paying again on every return, is open; if it stops and restarts, the
  next payment has to wait until the previous interval would have ended.

---

# Prerequisite for every hand-over

The extension needs per-capability detection shipped and in users' hands before Chrome enables anything.
Nothing above can hand over until it does.

The hand-overs are per capability — `load`/`error` go at Step 1, `MonetizationEvent` at Step 3 — but the
extension's detection is all-or-nothing, and split across worlds:

- `content/polyfill.ts` returns early, on its first statement, when `relList.supports('monetization')` is true, dropping the
  whole polyfill: the `MonetizationEvent` class, the `__wm_ext_monetization` bridge that dispatches it,
  and `onmonetization` on `HTMLElement`, `Window` and `Document` (in `content/polyfill.ts`).
- `load` and `error` dispatch from the isolated-world content script
  (`dispatchLoadEvent` in `content/services/monetizationLinkManager.ts`), which never reads `relList.supports`.

So the moment Chrome answers `relList.supports`, the extension stops firing payment events that Chrome
does not yet fire, and keeps double-dispatching `load` and `error`. Both failures from one flag.

That early return was written as double-install protection — its comment reads "already patched" — so
treating it as native detection is a reinterpretation rather than an extension of intent.

**Fix: one probe per capability that can switch on its own.** Capabilities on one feature switch
together, so one probe covers them; a capability on a feature of its own needs a probe of its own
([migration-gating: Feature detection in the extension](./migration-gating.md#feature-detection-in-the-extension)).
Splitting a feature adds a row here; that is the cost of the split.

The rule binds Chromium as much as the extension — anything put behind its own flag has to be detectable,
and where no probe exists the flag carries an ordering constraint instead
([migration-gating](./migration-gating.md)).

One row per probe, as the plan stands:

| The extension asks | Check | From |
|---|---|---|
| Does Chrome handle the link? | `relList.supports('monetization')` | Step 1 |
| Does Chrome define the handler? | `'onmonetization' in document` | Step 1, when that CL lands |
| Is Chrome tracking sessions? | `chrome.monetization.onStarted` exists | Step 2 |
| Is Chrome paying? | `chrome.monetization.onPayment` exists | Step 3 |

`relList.supports` reports one thing and stays true from Step 1 onward, so it answers the `load`/`error`
hand-over and no other. No extension-API signal exists at that step either, since `chrome.monetization`
arrives with observe, so that hand-over has to ride a web-exposed probe. It says Chrome handles the
link, not that Chrome fires those events correctly, which puts the obligation on our side: the flag
cannot be enabled until `load`/`error` fire unconditionally, or the extension stands down while Chrome
still gates them on a wallet and a granted permission
([migration-monetization-link](./migration-monetization-link.md)).

`onmonetization` ships under `Monetization`, so `relList.supports` answering true means Chrome defines
the handler too; the polyfill's guard — define it only if `'onmonetization' in document` is false — keeps
it from shadowing Chrome's accessor ([migration-monetization-link](./migration-monetization-link.md)).

`onPayment` is checked for existence, not waited on: the extension's loop can send before Chrome's first
event, and the owner is chosen once, at session start
([migration-extension-api: Payment events](./migration-extension-api.md#payment-events)).
It stays gated on the payments feature and absent at observe — shipping it alongside
`onStarted`/`onStopped` would make the check true while Chrome is still not paying.

Neither `chrome.monetization` check needs a link check beside it: Chrome keeps the API absent while
`Monetization` is off, so its presence means Chrome handles the link too
([migration-gating: The features](./migration-gating.md#the-features)).

Then wait for the release to reach users — the Web Store paces that, and it is the gate on enabling every step
([migration-overview: Risks](./migration-overview.md#risks)).

---

# Web Extension changes needed per step

What the yielding extension has to ship, in the order Chrome enables features. The checks are in the
section above. Wallet connect is Step 3, not observe.

**Step 1 — monetization link**

- Stop dispatching `load` / `error` when `relList.supports('monetization')` is true. Safe only once
  Chrome fires them unconditionally, which is a Chromium-side precondition on enabling the flag.
- Keep dispatching `MonetizationEvent`, but once `relList.supports('monetization')` is true, construct
  Chrome's interface instead of installing the polyfill's class — Chrome defines it here, constructible,
  and does not fire it yet; a polyfill class would shadow it.
- Define `onmonetization` only if `'onmonetization' in document` is false.

**Step 2 — observe**

- Consume `chrome.monetization.onStarted` / `onStopped`. Drop `documentObserver` and own
  session tracking from the DOM.
- Still pay. A session starting is not money moving. Report each payment through
  `chrome.monetization.reportPayment` and stop dispatching `MonetizationEvent` from the polyfill: Chrome
  fires it.
- Pause and resume a session's timer on `chrome.monetization.onPaused` / `onResumed`, in place of the
  content script's visibility and focus signals. Both are proposed
  ([#762](https://github.com/WICG/webmonetization/issues/762)).
- With nothing left page-side, drop the static content script and the all-sites host permissions — in
  the Chrome build, which `processManifestPlugin` already emits with a manifest of its own. A manifest
  cannot vary per user, and required hosts cannot be dropped at runtime (`permissions.remove` refuses
  them), so the hosts move to `optional_host_permissions`, the content script is
  registered through `scripting.registerContentScripts` only when `chrome.monetization.onStarted` is
  absent, and a user on a Chrome without observe grants the hosts from the popup — the wallet-connect
  flow a fresh install opens anyway. Existing installs see no prompt: Chrome bounds the stored grant by
  required ∪ optional on update (`PermissionsManager::GetBoundedExtensionDesiredPermissions`), so a host
  that turns optional stays active. New installs see no all-sites warning: warnings read active
  permissions, and optional ones start inactive. The optional declaration itself goes when
  `minimum_chrome_version` passes observe's shipping milestone. Key auto-add keeps the wallet-provider
  hosts it registers per provider.

**Step 3 — payments**

- Do not initialize Payment Manager (or any sessions for that matter) when
  `if (chrome.monetization.onPayment)` indicates that Chromium does it by itself.
- Stop reporting payments; Chrome pays and fires `MonetizationEvent` itself.
- Drive connect / disconnect through `connectWallet` / `finalizeWalletConnection` /
  `disconnectWallet`, or deep-link to `chrome://monetization` if those stay WebUI-only.
- Grant update and out-of-funds: `addFunds` / `updateBudget` if the control open keeps them,
  or link to Chrome UI.
- Refresh the popup balance from `onPayment` and `getWallet`.
- Send one-time pay through `pay`; until it exists, disable `PAY_WEBSITE` and show why.
- Disable custom / per-site rates and show why, unless native has them. That capability is
  For discussion above.
