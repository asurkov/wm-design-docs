# The monetization link surface — working notes

Sketch, not a finished document. Companion to
[migration-overview](./migration-overview.md).

This surface is the **monetization link**: `<link rel=monetization>` and what a page uses to declare
it, detect it, and observe it, plus the Permissions-Policy token that decides which frames may use it.

The pieces, each a section below. The feature is a prerequisite; the rest is Step 1 except where noted.

- the `Monetization` runtime feature — no `status`, off by default
- `relList.supports('monetization')` — feature detection
- `load` and `error` on the `<link>` — wallet-address resolution
- `MonetizationEvent` — the interface (defined in Step 1, fired in Step 3 — payments)
- `onmonetization` — the event handler
- the link lifecycle — when a payment session starts and stops
- the Permissions-Policy `monetization` token

Two nearby APIs are not this surface: `monetization-src` is the CSP directive that restricts
wallet-address fetches, and `navigator.permissions.query({name: "monetization"})` is how script
reads the user's grant. They have other owners (TBD migration-csp,
TBD migration-permission).

The prototype is [CL 4930834](https://chromium-review.googlesource.com/c/chromium/src/+/4930834). Take what we need
from it; change what we need.

Applies to everything below:

- All of it lands in Step 1 (monetization link), except firing `MonetizationEvent`, which is Step 3
  (payments). Both steps are in [migration-overview](./migration-overview.md#steps).
- Approval is Blink API owners, 3 LGTMs, under the `Monetization` Intent chain unless noted.
- The spec's "Link type" section — the keyword, the `load`/`error` triggers, the lifecycle — is a
  monkey patch for HTML, so that text ends up WHATWG's.

---

# Prerequisites

The `Monetization` runtime feature, off by default
([CL 8359628](https://chromium-review.googlesource.com/c/chromium/src/+/8359628)) — the first landing on
the ladder ([migration-gating: The ladder](./migration-gating.md#the-ladder)). A json5 entry with no
`base_feature_status`, and web-exposed — the link, `relList.supports`, the events, `onmonetization` — so
it climbs every rung, so far the only Blink feature in the plan, and so the only one that can take the
origin trial; its derived `blink::features::kMonetization` is the kill switch and the name the
`chrome://flags` entry enables. `renderer/platform` OWNERS review the entry, or any API owner by its
per-file rule; the `onmonetization` CL, `core/dom` OWNERS. Every piece below is gated on it, and nothing
else on Chromium has to land first. It ships only once all of them have landed; at opt-in a piece may lag
— the extension keeps polyfilling `onmonetization` until Chrome defines it
([migration-gating: The features](./migration-gating.md#the-features)). Its landings also carry the
`MonetizationService` binder gated on `kMonetization`, which the prototype binds ungated, with a
`content_switch_dependent_feature_overrides.cc` line so that `kMonetization` follows the experimental
flag ([migration-gating: Developer opt-in](./migration-gating.md#developer-opt-in)), and the
`monetization` permission name's check
(TBD migration-permission: The gating work it owes).

---

# `relList.supports('monetization')`

**Spec** — defined. "Checking if Web Monetization is supported" tells authors to call `supports()` with
the keyword. The keyword itself is defined in the monkey-patched section, and is not registered as a link
relation anywhere yet — [WICG/webmonetization#503](https://github.com/WICG/webmonetization/issues/503) is
open on that, blocked on a stable spec reference. Spec-side work; nothing in Chromium waits on it.

**Do** — add `monetization` to the link supported tokens under the runtime feature, in
`RelList::ValidateTokenValue` (`rel_list.cc`). Keep the check out of the token set beside it: that set is a
`DEFINE_STATIC_LOCAL`, fixed for the process, and the origin-trial rung makes the answer per execution
context. The prototype puts the check inside that set. Nothing else on this surface has to land first.
[CL 8359696](https://chromium-review.googlesource.com/c/chromium/src/+/8359696), with the tests below.

**Order** — this is what the extension stands down on, and one feature gates it and `load`/`error`
alike, so the flag cannot be enabled until the `load`/`error` rewrite below lands. The token is a few lines
and the events are the surface's largest piece; enabling in between stands the extension down while Chrome
still fires them only for visitors with a wallet and a granted permission, so visitors without one get
nothing from either implementation
([migration-native-vs-extension](./migration-native-vs-extension.md)).

**Test** — a WPT asserting `supports('monetization')` is true with the feature on, filed with this
surface's other link tests and run under a `VirtualTestSuites` entry passing
`--enable-features=Monetization`. `compression-dictionary`, another token in that set, has a WPT of the
same shape (`fetch/compression-dictionary/dictionary-fetch-with-link-element.tentative.https.html`).

The shared keyword table (`external/wpt/html/semantics/rellist-feature-detection.html`) is the eventual
home. It covers the link types HTML defines and every engine runs it, so `monetization` can move there
once the keyword is registered, and listing it there before then only fails the engines that do not have
it.

---

# `load` and `error`

**Spec** — defined, with triggers: fire `load` at the element on success, `error` otherwise; in the
monkey-patched section, so that text ends up WHATWG's.

Defined is not settled. [#344](https://github.com/WICG/webmonetization/issues/344) asks whether `load`
fires once the wallet address is merely well-formed or once the fetch and parse have finished — this
surface implements the second, and the first would move the event ahead of the resolution it reports.
[#343](https://github.com/WICG/webmonetization/issues/343) asks where the events fire for a link given as
an HTTP `Link:` header, where there is no element to fire at and `error` collides with `window.onerror`.
Both have been open since 2023.

These events carry nothing about the visitor. An event that varied with visitor state would make `error`
positive proof that a visitor is monetization-capable — point a `<link>` at a deliberately broken address
and the response sorts your audience. That is what we implement, and the spec has to require it: a
normative line in the Link type section that neither the permission nor a connected wallet affects
whether or when they fire, and the threat written up in Privacy Considerations. Both are Web Monetization
text. [#741](https://github.com/WICG/webmonetization/pull/741) writes part of it, in Privacy Considerations:
a MUST that `load` fires independently of the permission state, and the threat. `error`, the wallet and
the line in the Link type section are still missing.
[#502](https://github.com/WICG/webmonetization/issues/502) is that discussion — it asks for
deliberate detection of an active wallet authorization, which is the capability these events must not
give away by accident. The extension already fires `load` with no wallet connected; [issue
648](https://github.com/interledger/web-monetization-extension/issues/648) asked whether that was
wrong and closed into #502.

**Do** — fire them on the site's wallet address resolving or failing. That moves the fetch ahead of the
permission and the credential check, so it happens for every visitor on a monetized page — the cost of
the rule, and what `monetization-src` is there to bound. Biggest piece on this surface: the others mostly
port prototype code, this reorders the browser-process flow.

**Test** — the firing pair first, then the invariance that carries the privacy property:

- `load` when the site's address resolves, `error` when it does not. WPT, feature on.
- Neither changes with the permission granted or denied. WPT: `test_driver.set_permission` has the
  plumbing it needs, the name reaching `PermissionType::MONETIZATION` through
  `PermissionDescriptorToPermissionType`.
- Neither changes with a wallet connected or not. Browser test — a connected wallet has no web-facing
  control, only `chrome.monetization` and `chrome://monetization`.

**Note** — the prototype gates the fetch and both events behind Permissions-Policy, the permission prompt
and connected credentials (`MonetizationServiceImpl::CreateSession` for the policy and the prompt,
`CreateSessionWithStatus` for the credentials): a rewrite, not a port.
The extension gates on none of it, so it cannot stop dispatching until Chrome matches.

---

# `MonetizationEvent`

**Spec** — defined, with IDL. The current attributes are `amountSent`, `incomingPayment` and
`walletAddress`; `amount`, `assetCode`, `assetScale`, `receipt` and — since
[PR 520](https://github.com/WICG/webmonetization/pull/520) landed on 2026-09-15 — `paymentPointer` are
marked deprecated, and [#733](https://github.com/WICG/webmonetization/issues/733) is to remove them.
[PR 744](https://github.com/WICG/webmonetization/pull/744) specifies how the constructor turns the init
dictionary's amount into the attribute's interface.

**Do** — land `MonetizationEvent` and `MonetizationCurrencyAmount` in Step 1 under
`RuntimeEnabled=Monetization`. The event carries the current attributes and none of the deprecated
ones. `amountSent`'s type is the second interface (`monetization_currency_amount.idl`:
Window-visible, no constructor, `currency` and `value`). The constructor init is
`MonetizationEventInit`, whose `amountSent` is a `PaymentCurrencyAmount` dictionary — allowed on
dictionaries, not the same type as the attribute. The prototype already has both. Gate dispatch on
the payments feature so nothing fires until Step 3.

**Ship `walletAddress`, not `paymentPointer`.** The prototype still has `paymentPointer` on the event and
its init dictionary (`monetization_event.idl`, `monetization_event_init.idl`), and the name is web-visible
the moment the interface is enabled. Leaving deprecated attributes out makes the port rename it.

**Keep the constructor exposed.** The extension fires these events from its content script until Step 2,
when it reports payments to Chrome instead, and needs `new MonetizationEvent(...)` until then.
`monetization_event.idl` already declares it.

**Test** — the interface first, then the event the browser dispatches:

- `MonetizationEvent` exposed and its constructor reflecting the init dictionary, with the payments
  feature off. WPT, from Step 1.
- The dispatched event bubbles (the `MonetizationEvent` constructor passes `Bubbles::kYes`). Browser test against a test Open Payments
  server, from Step 3.
- `amountSent` in the receiver's asset — how much the sender sent, in the receiver's units, which is Open
  Payments `receiveAmount` — and `incomingPayment` identifying the payment reported. The prototype takes
  the sender's debit number and labels it with the site's currency, so this is a fix, not a port
  ([migration-native-vs-extension](./migration-native-vs-extension.md)). Same browser test; both values
  come from the payment engine (TBD migration-payment-engine).

---

# `onmonetization`

**Spec** — specified in [PR 697](https://github.com/WICG/webmonetization/pull/697): `onmonetization` on a
`partial interface mixin GlobalEventHandlers`, as both an event handler IDL attribute and a content
attribute, on all elements, `Document` and `Window`. Nothing outstanding spec-side. Extending that mixin
from another spec is routine — `global_event_handlers.idl` already carries Pointer Events and Selection
API the same way.

**Do** — both handler forms, under `Monetization` and its Intent, as a CL of their own: every handler in
`global_event_handlers.idl` is gated on the feature of its event (`OverscrollGestures`,
`CSSScrollSnapChangeEvent`), and `MonetizationEvent` is already under `Monetization`. The mixin's reach
is what API owners will ask about; bubbling element events get `GlobalEventHandlers` handlers
(`onscrollend`, `onscrollsnapchange`), and `monetization` bubbles. If the objection comes at
Intent-to-Ship, a `RuntimeEnabled` name of its own on the member splits it then, in one line. Splitting
it late costs the extension nothing: the polyfill already guards on `'onmonetization' in document` to
avoid shadowing Chrome's accessor, and that guard is the probe a feature of its own would need
([migration-native-vs-extension](./migration-native-vs-extension.md)).

- `[RuntimeEnabled=Monetization] attribute EventHandler onmonetization;` in `global_event_handlers.idl`
  → `el.onmonetization`, and a property on every element, document and window in Chrome
- a name in `html_attribute_names.json5` plus a row in the attribute-to-event table in `html_element.cc`
  → `<link onmonetization="…">`; the rows there carry no feature check and need none — the attribute
  binds a listener for an event that fires only under the feature

The event type itself needs nothing: `event_type_names.json5` registers it, so
`addEventListener('monetization', …)` already works.

**Test** — WPT for both forms, handler bound above the `<link>` so a link-scoped implementation fails.

---

# Link lifecycle

**Spec** — in review. [#724](https://github.com/WICG/webmonetization/pull/724) closes
[#698](https://github.com/WICG/webmonetization/issues/698). Its 2026-10-07 revision (`7b8be16`) takes up our
review of 2026-10-01: a link has one *associated payment session* or null; the type gets its own
fetch-and-process algorithm that stops that session before running HTML's default one, so every trigger —
an `href` that no longer parses included — stops first; a response is ignored once a newer fetch became
appropriate; the session stops when the element stops creating a "monetization" link or being
browsing-context connected (`rel` and `href` removal). `disabled` only makes the setup steps return false;
its conformance in HTML is [#755](https://github.com/WICG/webmonetization/issues/755). That session starts
when the wallet address JSON parses, immediately before `load` — under
[#741](https://github.com/WICG/webmonetization/pull/741), after `load` is queued and only if the permission
is granted. Left: the "cannot navigate" step was dropped, so only the flows draft ends the sessions of a
document that stops being fully active; a failed response still starts a session (the "Otherwise" binds to
the session-limit step; #741 fixes it); the flows draft still carries its own re-process rule, which Michael
will adjust separately.

**Do** — re-run fetch-and-process on `rel`, `href` and `disabled`, stopping the payment session and
starting a new one. Visible from Step 1 as `load`/`error` firing again on the new value; from Step 3 it
also stops payments, `disabled` behaving as removal so re-enabling starts fresh rather than paying a
backlog.

`HTMLLinkElement` holds one `LinkResource` in `link_` and re-enters fetch-and-process through `Process()`
— a `LinkStyle` for a stylesheet link, a `LinkMonetization` for this one, holding a `MonetizationSession`
(`LinkMonetization::monetization_session_`) that is not the spec's payment session. A CL at each point:

- **`rel` and `href`** reach `Process()`, so the fix is `MonetizationSession::StartIfNeeded`, which returns
  early when a session already exists. In `modules/monetization`, reviews
  with this surface.
- **`disabled`** never reaches `Process()`: `GetLinkStyle()` returns null for a monetization link, so
  `HTMLLinkElement`'s `disabled` handling assigns a fresh `LinkStyle` to `link_` and drops the `LinkMonetization`
  without `CloseSession`. Generic `HTMLLinkElement` code, so that file's owners review too.

Step 3 adds the no-accumulation rule above, in the payment engine. Step 2's hand-over CL — the extension
dropping `documentObserver` for `onStarted`/`onStopped` — waits on the restart
(TBD migration-extension-api).

**Test** — per attribute, that the old session stops and a new one starts, seen as `load`/`error`
re-firing. Unit coverage, from Step 1. A disabled link paying nothing, and re-enabling paying no backlog,
needs a browser test at Step 3.

**Note** — while `disabled` is set the extension keeps adding up what it would have paid, then sends the
whole amount once the attribute is removed. #698 records that as a bug on its side, so it is not the
reference for `disabled`.

---

# Permissions-Policy `monetization` token

**Spec** — defined. "Permissions Policy integration" declares a policy-controlled feature named
`monetization`, default allowlist `'self'`.

**Approval** — Blink as usual, plus the Permissions-Policy owners for the code, a different set again.

**Do** — one entry in
`services/network/public/cpp/permissions_policy/permissions_policy_features.json5`, carrying
`depends_on: ["Monetization"]` so the token appears and disappears with this surface's feature. Nothing
hands over; an extension cannot add Permissions-Policy tokens.

**Test** — expectation updates rather than new tests. With no `status`, the feature is off in every default
run, so the token appears in `webexposed/feature-policy-features-expected.txt` and
`wpt_internal/isolated-permissions-policy/permissions_policy.https.html` only under a virtual suite that
enables it, and `virtual/stable/webexposed/feature-policy-features-expected.txt` must not list it. Keep
them with this piece; they say nothing about `relList.supports`.

**Not the permission surface, despite the name.** Same shape as camera: `Permissions-Policy: camera=()`
blocks a frame whatever the user granted, and a denial blocks it whatever the policy allows.
`MonetizationServiceImpl::CreateSession` applies both — the policy check synchronous and read from the
document, the permission check through `PermissionController` and able to prompt
(TBD migration-permission).

---

# Parity testing

Per-CL tests sit with each piece above. Parity across the hand-over belongs to no CL, and nothing verifies
it on either side today: one page driven against both implementations — extension-only, native-only, and
both installed with detection active — asserting the same events in the same order with the same payloads.
The both-installed run is what catches double dispatch and the polyfill's all-or-nothing bail.

Little to inherit. The prototype's own tests are `monetization_event_test.cc`, which no `BUILD.gn`
references, so it never runs; a case in `privacy_page_index_test.ts`; and an expectation row in
`content_security_policy_test.cc`. The token's appearances in the feature-policy expectations and the
isolated-permissions-policy WPT are enumeration lists, not behaviour tests.
