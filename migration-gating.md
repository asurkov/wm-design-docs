# Gating: how a feature gets enabled

The Web Monetization plan ([migration-overview](./migration-overview.md)) ports its steps from the
prototype ([CL 4930834][cl-4930834]), and everything in it lands behind a feature that is off. This
document describes how a feature then reaches users: how the plan's features depend on one another, how
each climbs the ladder from off to shipped, how the extension detects each hand-over, and what disabling
a feature does not reach or undo. The surface documents say what each feature's CLs must carry;
[Each feature's path](#each-features-path) links them.

---

# Terms

- **CL, landing** — a change list, Chromium's unit of code review; it lands when it merges.
- **surface, step** — the overview's terms. A surface is the work grouped by who approves it, one
  document each. A step is one job the extension does today that moves into Chrome behind a feature
  (below) ([migration-overview: Surfaces](./migration-overview.md#surfaces),
  [Steps](./migration-overview.md#steps)).
- **extension, native, extension API** — the Web Monetization extension implements the specification
  today. Native is the same functionality implemented in Chrome itself. The extension API is
  `chrome.monetization`, through which the extension talks to Chrome once Chrome takes a job over. A
  **hand-over** is the moment one job moves from the extension to Chrome.
- **payment session** — the spec's term for a period on a page. It begins once the wallet address of
  a `<link rel=monetization>` has been fetched and parsed — under
  [#741](https://github.com/WICG/webmonetization/pull/741), only with the user's permission — and ends
  when the link is removed or the user leaves the page.
- **Finch** — Google's server-side feature configuration. It can enable a feature for a percentage
  of users, which is a Finch rollout, or switch it off, which is a kill switch
  ([What disabling does not reach or undo](#what-disabling-does-not-reach-or-undo)).
- **feature** — one switchable part of the plan. Every feature has a `base::Feature`, Chromium's
  on/off switch, which `--enable-features`, Chrome's flags page and Finch can set. A feature is
  either a Blink feature or a browser feature (below).
- **Blink feature** — a feature declared in `runtime_enabled_features.json5`, the feature list of
  Blink, the renderer; Chromium calls it a Blink runtime feature. The entry gives two switches. The
  **Blink flag** is read by Blink code. The generated `base::Feature`, `blink::features::kX`, is read
  by browser code.
  Overriding the `base::Feature` also sets the Blink flag, never the reverse. `Monetization` is the
  plan's Blink feature; it gates Step 1, the monetization link, called the link for short
  ([Notes](#notes)).
- **browser feature** — a feature declared with `BASE_FEATURE` alone, with no
  `runtime_enabled_features.json5` entry, so its `base::Feature` is its only switch; browser and
  Blink code can both read it.
- **dependent feature** — one of the plan's browser features, observe (Step 2), payments (Steps 3
  and 4) or the CSP directive (no step): each acts on what `Monetization` produces
  ([The features](#the-features)).
- **web-exposed** — a change a web page can observe, such as a new API, element behaviour or event.
- **Intent-to-Prototype, Intent-to-Ship** — the blink-dev emails that gate landing and shipping a
  web-exposed change; the first is an FYI, the second needs three API owners' approval
  ([Which rungs a feature takes](#which-rungs-a-feature-takes)).
- **rung, the ladder** — the audiences a feature can be enabled for, in order: nobody (off),
  developers (developer opt-in), opt-in users (user opt-in), everyone (shipped). The plan takes no
  origin trial ([migration-overview: Monetization link](./migration-overview.md#monetization-link)).
- **`chrome://flags` entry** — a user-facing switch on Chrome's flags page; one entry can enable
  several features. The **experimental flag**,
  `chrome://flags/#enable-experimental-web-platform-features`, is Chromium's existing entry for
  features still in development ([Developer opt-in](#developer-opt-in)). The plan adds one entry of
  its own, the **opt-in entry**, for the Web Monetization features ([User opt-in](#user-opt-in)).
- **waterfall** — enabling on trunk and letting it reach stable through canary, dev and beta.
- **probe** — a check the extension makes to see whether Chrome is doing a job yet
  ([Feature detection](#feature-detection-in-the-extension)).

---

# The features

Users see Web Monetization as one feature. In Chromium it is several, each with its own switch and
its own approvals; [Each feature's path](#each-features-path) lists them.

Code lands in pieces, each behind a feature: a piece runs only while its feature is on. The
extension probes for each job, and once a probe answers yes it stops doing that job itself
([Feature detection](#feature-detection-in-the-extension)).

There are several features because the parts move on different schedules: the link needs spec text
in HTML before it ships, observe needs the extension API proposal, payments needs its spending
behaviour specified, and the CSP directive needs spec text in CSP and Fetch
([migration-overview: Steps](./migration-overview.md#steps),
[Risks](./migration-overview.md#risks)). With one feature for all of them, no part could ship before
the slowest part could. With separate features, the link can ship while
the CSP directive still waits for its Intent-to-Ship
(TBD migration-csp: Order). Separate features also let the extension hand
its jobs over one at a time.

A separate feature has costs: a declaration, a cleanup CL, and a line in the opt-in entry. A feature
that takes over a job from the extension also costs a probe in the extension
([Feature detection](#feature-detection-in-the-extension)). Wallet connection and
`chrome://monetization` involve no web page, so a feature of their own would need no Intent and would
not have to wait for the link's Intent-to-Ship, as observe and payments do
([Which rungs a feature takes](#which-rungs-a-feature-takes)). The first shippable state keeps wallet
and payment logic in the extension
([migration-overview: The first shippable state][first-shippable]), so such a feature would carry those
costs before anything used it, and the plan gates wallet connection and `chrome://monetization` by
payments instead. If that changes, this is the split to make.

Among the features, `Monetization` is the root. With it on, Chrome handles
`<link rel=monetization>`: it fetches the wallet address the `<link>` points at and opens a payment
session. Observe, payments and the CSP directive are the dependent features. Observe, for example,
acts on the session, reporting it to the extension; the CSP directive acts on the fetch, which a
site's `monetization-src` policy can block. Each can be switched on or off by itself, but with
`Monetization` off there is no session or fetch for it to act on.

Code can read more than one feature, and the plan uses that to tie observe and payments to
`Monetization` in code. Each member of `chrome.monetization` exists only while `Monetization`'s
`base::Feature` is on as well as the feature that gates it, observe or payments, and
`chrome://monetization` only while that `base::Feature` and payments are on ([Notes](#notes)). The
probes for observe and payments therefore cannot answer yes while Chrome leaves
`<link rel=monetization>` to the extension.

A dependent feature moves to a rung only after `Monetization` has reached it. This order is the
plan's choice, not a requirement of Chromium. If the link stalls, the plan can consider shipping
other features independently, under approvals of their own. The overview explains why the link
starts first ([migration-overview: Step 1][step-1], [The first shippable state][first-shippable]).

Everything under one feature is switched on together, so a piece is gated by the feature it has to
ship with, regardless of who reviews it. The Permissions API name is under `Monetization`:
Permissions owners review that CL, but the plan ships it with the link, under the link's
Intent-to-Ship, unless its spec text is late (below)
([migration-overview: Permission](./migration-overview.md#permission)). A piece that spans more than
one schedule is split: `MonetizationEvent` is defined under `Monetization` and fired under observe and
payments ([Notes](#notes)).

Pieces of one feature land one CL at a time, and the opt-in rungs wait only for the pieces a shared
probe covers ([Feature detection](#feature-detection-in-the-extension)). At developer and user opt-in
any other piece still in review is absent, and for extension users the extension keeps doing
that job until its probe shows Chrome doing it. `Monetization`, for example, can reach developer and
user opt-in before Chrome defines `onmonetization`. Shipping waits for every piece: it reaches
visitors who did not opt in, and for those without the extension nothing would cover a missing one.

A piece of `Monetization` that cannot ship with the rest moves to a Blink feature of its own, not a
dependent feature: a json5 entry and one check where the piece is gated. The candidates are the
Permissions API name, if its spec text is not written by the link's Intent-to-Ship, and
`onmonetization`, if API owners object at Intent-to-Ship to a handler that appears on every element,
document and window (TBD migration-permission: The process,
[migration-monetization-link: onmonetization][link-onmonetization]).

---

# The ladder

A feature is enabled for a wider audience one rung at a time, though not every feature has every
rung ([Which rungs a feature takes](#which-rungs-a-feature-takes)). The extension ships its probes
first, and a probe answers yes only in a Chrome that has the feature on
([Feature detection](#feature-detection-in-the-extension)). Before user opt-in a feature is on only
for someone who passes `--enable-features` or keeps the experimental flag on, in practice a
developer. User opt-in is therefore the first rung at which the hand-over reaches the extension's
users who turn the entry on.

| Rung | What turns it on | In code |
|---|---|---|
| Off | nothing by default; `--enable-features` on the command line | the json5 entry with no `status`; a browser feature with `FEATURE_DISABLED_BY_DEFAULT` ([Off](#off)) |
| Developer opt-in | the experimental flag | `status: "experimental"` on the json5 entry, plus a `content_switch_dependent_feature_overrides.cc` line if browser code checks the feature; no `about_flags.cc` entry of its own ([Developer opt-in](#developer-opt-in)) |
| User opt-in | the opt-in entry | the feature's name on the opt-in entry's list, in `about_flags.cc` ([User opt-in](#user-opt-in)) |
| Shipped | on everywhere; for a web-exposed feature, once an Intent-to-Ship is approved | `status: "stable"`, or `FEATURE_ENABLED_BY_DEFAULT` on a browser feature, by waterfall; the `base::Feature` stays as kill switch until a cleanup CL removes the feature's declaration and its checks ([Which rungs a feature takes](#which-rungs-a-feature-takes)) |

### Off

What a landing needs depends on what the code exposes. Web-exposed code needs an Intent-to-Prototype
first; the link's is already on blink-dev, linked from [chromestatus][chromestatus]. Beyond that a
landing waits on OWNERS review alone, since the code lands disabled: the CSP directive's CLs do not
wait for its Intent-to-Ship (TBD migration-csp: Order). An extension API is the
exception: Chromium approves an API before it is implemented, so the `chrome.monetization` code of
observe and payments waits on the API proposal even to land
([migration-extension-api: Approval](./migration-extension-api.md#approval)).

### Developer opt-in

This rung covers `Monetization` alone. Setting `status: "experimental"` on its json5 entry, with one
more line described below, makes it available to web developers who keep the experimental flag on.
They can try Chrome's link handling, `relList.supports`, `load` and `error`, before it ships. Chrome
fires no `MonetizationEvent` and pays nothing here, so full Web Monetization still
needs the extension, which keeps paying and firing the events. An extension user who keeps the flag
on gets that mix here, ahead of user opt-in: Chrome handles the link, and the extension does the
rest.

By itself the experimental flag turns on only half of `Monetization`. It sets the Blink flag and
leaves the `base::Feature` off, and the `base::Feature` gates `MonetizationService`, the
browser-side service that fetches the wallet address. With the flag alone `relList.supports` would
answer true while Chrome never fetches the wallet address or fires `load` and `error`. One line
fixes that. Chromium keeps a list of `base::Feature`s that a switch turns on as well, and the plan
adds `Monetization`'s under the experimental flag. The flag then turns on both halves, and the
developer does nothing extra ([Notes](#notes)).

The dependent features stay off the list, so none of them has this rung. Observe and payments are an
extension API, and the flag is meant for web platform features. Listing them would not be enough
either: until a CL grants the API's permission on `stable`, the release channel keeps the API away
([User opt-in](#user-opt-in)). The CSP directive is web-exposed and could be listed. The plan
considers it and has not decided: listing it would let a site with a strict policy try the link
(TBD migration-csp: Order). Until user opt-in, a developer turns a dependent
feature on with `--enable-features`, with `Monetization` on as well. For observe or payments, until
that CL lands, the developer also passes `--enable-experimental-extension-apis`.

### User opt-in

One `chrome://flags` entry, the opt-in entry, turns on every Web Monetization feature that has
reached this rung. It is added when the first feature reaches the rung, in a small CL that any
committer can approve. It takes the place of the prototype's entry, which enables one feature only
([Notes](#notes)). A feature stays on the entry's list from reaching user opt-in until it ships, and
the entry is deleted once the list is empty. A user who turned the entry on once therefore gets each
feature as it arrives, with a Chrome update. The user cannot pick among them. Only a developer can
choose which to turn on, with `--enable-features`.

Observe and payments have a second gate that the entry does not control: the release channel. Using
`chrome.monetization` requires an extension permission. `_permission_features.json`, the extension
system's list of who may hold which permission, grants it per release channel, and the channel is a
property of the build, not a setting. The prototype grants the permission on the `trunk` channel,
which only a developer's own build reports: no released Chrome does, Canary included. On any other
channel it grants it only with `--enable-experimental-extension-apis`
([migration-extension-api: Prerequisites](./migration-extension-api.md#prerequisites)).
A CL that grants it on `stable` without the switch has to land before either feature reaches this
rung. The entry alone then turns them on for users ([Notes](#notes)).

The entry has an expiry milestone ([flag_expiry.md: The Process][flag-expiry-process]). If it
lapses, Chrome hides the entry and drops each user's stored choice, so renewing it later does not
bring the choice back ([Notes](#notes)). Every opt-in user then falls back to the extension, if it
is installed, and has to turn the entry on again. The plan therefore renews the expiry while any
feature is on the entry's list.

Chromium's flags policy says that flags are not a supported configuration surface and that nothing
should depend on one ([flag_expiry.md][flag-expiry]). This rung goes against that: extension users
depend on the entry to get native Web Monetization. The plan accepts that because no other mechanism
fits. Finch cannot select users by installed extension. An origin trial selects origins, not users.
A setting would expose an unshipped web-exposed feature to every user. The choice has two costs: the
expiry above, and a kill switch that does not reach a user who enabled the entry
([What disabling does not reach or undo](#what-disabling-does-not-reach-or-undo)).

### Which rungs a feature takes

Two properties decide a feature's path: how it is declared, and whether it is web-exposed. The first
decides which rungs the feature has, the second how it ships.

Developer opt-in works through a json5 entry. A browser feature could also get it, through the list
of `base::Feature`s that the experimental flag turns on, but the plan puts only `Monetization`'s on
that list ([Developer opt-in](#developer-opt-in)). So `Monetization` has all four rungs, and each
dependent feature has three: off, user opt-in and shipped.

Whether the feature is web-exposed decides how the last rung, shipped, is reached. A web-exposed
feature needs an approved Intent-to-Ship and then ships by waterfall. Chromium discourages a Finch
rollout for it, because a page would then behave differently for different users
([flag_guarding_guidelines.md][flag-guarding]). A feature that is not web-exposed needs no Intent
and may ship gradually through a Finch rollout.

The two properties are independent, because web exposure is about what enabling changes, not how the
feature is declared. The CSP directive is the example. The plan declares it as a plain
`base::Feature` in the network namespace, which the network service and Blink both read, and gives
it no json5 entry. It therefore has three rungs. A json5 entry driven by that feature would give it
developer opt-in, and the plan has none so far ([Notes](#notes)). But enabling it changes how a page's
Content-Security-Policy header is parsed, which a page can observe. Its shipped rung therefore needs
an Intent and a waterfall, as `Monetization`'s does.

Observe and payments are browser features with three rungs too. All a page sees of them is
`MonetizationEvent` arriving from Chrome, for a payment Chrome made or one the extension reported
through `reportPayment`. That firing is approved under the link's Intent-to-Ship. As long as they
ship after the link, as the plan orders, they need no Intent of their own. They still ship by
waterfall, not by a Finch rollout. A rollout would make that firing differ at random between users
with the same setup, which the guideline warns against. The opt-in entry already gives extension
users a gradual start.

### Each feature's path

| Feature | Declared as | Rungs | Its gating, in its surface |
|---|---|---|---|
| `Monetization` (Step 1) | Blink feature | off, developer opt-in, user opt-in, shipped | [migration-monetization-link: Prerequisites](./migration-monetization-link.md#prerequisites) |
| observe (Step 2) | browser feature, read via `feature_flag` ([Notes](#notes)) | off, user opt-in, shipped | [migration-extension-api: Observing sessions](./migration-extension-api.md#observing-sessions); the site-settings gates read it: TBD migration-permission: The gating work it owes |
| payments (Steps 3 and 4) | browser feature, read via `feature_flag` ([Notes](#notes)) | off, user opt-in, shipped | TBD migration-payment-engine: The CLs, [migration-extension-api: Native payments](./migration-extension-api.md#native-payments), TBD migration-user-control: Where things stand |
| the CSP directive (no step) | browser feature | off, user opt-in, shipped | TBD migration-csp: Order |

---

# Feature detection in the extension

The extension probes for what Chrome does, job by job, not for features. Each piece that can arrive
on its own therefore needs a probe. Pieces that arrive together share one:
`relList.supports('monetization')` covers link handling. A piece that may land after its feature
reaches its first opt-in rung needs a probe of its own. The extension checks
`'onmonetization' in document` for that reason, and does not infer the handler from
`relList.supports`. Observe and payments each need their own, since each is enabled separately.
Those two probes need no link check beside them, since their API exists only while `Monetization` is
on ([The features](#the-features)). The CSP directive needs no probe, since nothing hands over for
it.

A shared probe carries a condition. Every job it covers has to work in Chrome before the feature
reaches its first opt-in rung, because the extension stops all of them at once. `load` and `error`
show this. The extension stops firing them when `relList.supports('monetization')` turns true, so
`Monetization` cannot reach developer opt-in until Chrome fires them in every case the extension
did.

Detection runs one way: the extension probes Chrome, and Chrome never checks for the extension
([migration-extension-api: Payment events](./migration-extension-api.md#payment-events)). A version of the extension without the
probes keeps doing every job beside Chrome. The version that has them is therefore published before
a feature is enabled ([migration-overview: Risks](./migration-overview.md#risks)).

The full list of probes, and what the extension changes at each step, is in
[migration-native-vs-extension](./migration-native-vs-extension.md).

---

# What disabling does not reach or undo

The Finch kill switch switches a feature off without a Chrome update. Only a Google owner of the
code can activate it. A command-line switch takes precedence over Finch (`base/feature_list.h`), and
`chrome://flags` applies its choices as command-line switches at startup. So the kill switch does
not reach a user who turned the feature on with a switch: the opt-in entry, `--enable-features` or,
for `Monetization`, the experimental flag ([Notes](#notes)). For a user who enabled the opt-in
entry, only a Chrome update switches the feature off, by taking it off the entry's list or by
deleting the entry.

The kill switch therefore matters only once a feature is on by default, which is when it ships. At
the opt-in rungs every user who has the feature turned it on with a switch, so the kill switch
reaches nobody there. Once a feature has shipped it has left the entry's list
([User opt-in](#user-opt-in)), and the kill switch reaches everyone who has not turned the feature
on with a switch.

Where disabling reaches a user, it stops the gated code and leaves that code's state behind: stored
credentials, and a live Open Payments grant. What to do with that state is the payment engine's
question
(TBD migration-payment-engine: What a disabled feature leaves behind).

---

# Notes

Any code reads a `base::Feature` with `base::FeatureList::IsEnabled`. Blink code reads a Blink
feature with `RuntimeEnabledFeatures::XEnabled(context)`, or with `[RuntimeEnabled=X]` in IDL, and
never with `IsEnabled`: a `base::Feature` sees the experimental flag only if it is on the list
described below ([Developer opt-in](#developer-opt-in), [RuntimeEnabledFeatures.md][rte-features]).

By default the Blink flag of a Blink feature takes the state of its `base::Feature` whenever that
`base::Feature` is enabled or overridden. That is how `--enable-features`, `chrome://flags` entries
and Finch reach Blink. `blink::features::kMonetization` is therefore `Monetization`'s kill switch and
the name the opt-in entry enables.

`content/public/common/content_switch_dependent_feature_overrides.cc` is the list of
`base::Feature`s that command-line switches override. Every entry under the experimental flag turns
one on ([Developer opt-in](#developer-opt-in)). The plan lists `blink::features::kMonetization`
under the experimental flag, as the experimental entries `JavaScriptImportText`,
`GlobalPrivacyControlTest` and `SpecCompliantXmlMimeTypes` already are. The line is needed because
the `MonetizationService` binder checks `kMonetization`, as defense in depth against a renderer that
asks for the service with the feature off. The list's overrides are registered after
`--enable-features` and before Finch's (`RegisterExtraFeatureOverrides` in
`components/variations/service/variations_field_trial_creator.cc`). The first override of a feature
wins (`base/feature_list.h`). So the kill switch does not reach `kMonetization` for a user with the
experimental flag on
([What disabling does not reach or undo](#what-disabling-does-not-reach-or-undo)).

Tests are not a rung: a `VirtualTestSuites` entry passes `--enable-features=Monetization`
([migration-monetization-link: relList.supports][link-rellist]), so web tests and WPT run against
the feature at any rung. `status: "experimental"` also turns it on in web tests, which run with
`--enable-experimental-web-platform-features`, so the line on the list above matters there too. The
exception is `--stable-release-mode`, which the `virtual/stable` suite runs with.

Observe and payments are declared with `BASE_FEATURE` in
`third_party/blink/public/common/features.h`, a header that every place checking them may include:
`extensions/common` for `kFeatureFlags[]` (next paragraph), `chrome/browser` and
`components/monetization`. They cannot live in `components/monetization`, which `extensions/common`
may not include. The CSP directive's feature is in `network::features` instead
(TBD migration-csp: `monetization-src`). A network feature can
drive a json5 entry: `content/child/runtime_features.cc` maps `network::features::kBrowsingTopics`
to `TopicsAPI`, among others. The plan declares no such entry for the directive.

The extension API reads a browser feature by name: `kFeatureFlags[]` in
`extensions/common/features/feature_flags.cc` maps a string name to the feature, and an
`_api_features.json` entry names it in `feature_flag`. The lookup comes before the entry's context
and permission checks (`simple_feature.cc`), and if the feature is disabled the API is absent. A
name missing from `kFeatureFlags[]` is a `CHECK` when the entry is evaluated, so the
`kFeatureFlags[]` line lands no later than the first entry that names it. The browser also checks
observe where it dispatches `onStarted` and `onStopped`, at one chokepoint
([migration-extension-api: Sessions](./migration-extension-api.md#sessions)).

A `feature_flag` names one feature, so the tie to `Monetization` is a second condition on the same
entries: `requires_delegated_availability_check`, with a handler that reads
`blink::features::kMonetization`, registered in `chrome/common/initialize_extensions_client.cc`.
Controlled Frame's handler reads its own Blink feature in the same way
(`chrome/common/controlled_frame/controlled_frame.cc`). The handler runs after the `feature_flag`
lookup (`simple_feature.cc`), so the API is present only while both features are on.
`chrome://monetization` is gated the same way: `MonetizationUIConfig::IsWebUIEnabled` reads the
payments feature and `blink::features::kMonetization`.

One check serves two features: `MonetizationServiceImpl::OnPaymentRecorded` reads observe for a
payment the extension reported and payments for one Chrome made, before `OnPayment` reaches the
renderer. `MonetizationSession::OnPayment`, on the renderer side, needs no check of its own: the
browser sends `OnPayment` only once that check has passed.

The opt-in entry uses `ENABLE_DISABLE_VALUE_TYPE_AND_VALUE` from
`components/webui/flags/feature_entry_macros.h`, passing `switches::kEnableFeatures` and the feature
names joined by commas, as `glic-actor` in `about_flags.cc` does. It replaces the prototype's
`FEATURE_VALUE_TYPE(blink::features::kMonetization)` entry, its `flag_descriptions.h` strings and
its `enums.xml` line. It needs no `enums.xml` line of its own, since its enable and disable switches
are already listed. Its `flag-metadata.json` entry names owners, who are the expiry contacts, and an
`expiry_milestone` ([User opt-in](#user-opt-in)). Once the milestone has passed,
`FlagsState::GetSanitizedEnabledFlags` in `components/webui/flags/flags_state.cc` takes the entry
off the stored list of enabled flags, which is how a user's choice is lost. `about_flags.cc`,
`flag_descriptions.h` and `flag-metadata.json` are `per-file *` in `chrome/browser/OWNERS`, which is
why any committer can approve the entry's CL.

[cl-4930834]: https://chromium-review.googlesource.com/c/chromium/src/+/4930834
[step-1]: ./migration-overview.md#step-1--monetization-link
[first-shippable]: ./migration-overview.md#the-first-shippable-state
[link-onmonetization]: ./migration-monetization-link.md#onmonetization
[chromestatus]: https://chromestatus.com/feature/4917148755689472
[flag-expiry]: https://chromium.googlesource.com/chromium/src/+/main/docs/flag_expiry.md#Do-Not-Depend-On-Flags
[flag-expiry-process]: https://chromium.googlesource.com/chromium/src/+/main/docs/flag_expiry.md#The-Process
[flag-guarding]: https://chromium.googlesource.com/chromium/src/+/main/docs/flag_guarding_guidelines.md#Prefer-waterfall-rollout-for-platform-changes
[rte-features]: https://chromium.googlesource.com/chromium/src/+/main/third_party/blink/renderer/platform/RuntimeEnabledFeatures.md#Blink-Features-and-Chromium-Features
[link-rellist]: ./migration-monetization-link.md#rellistsupportsmonetization
