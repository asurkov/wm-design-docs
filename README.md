# Web Monetization in Chromium: design docs

Design documents for bringing [Web Monetization](https://webmonetization.org/specification/) into
Chromium natively, starting from the prototype
[CL 4930834](https://chromium-review.googlesource.com/c/chromium/src/+/4930834) and taking over from the
[Web Monetization extension](https://github.com/interledger/web-monetization-extension).

Start with the rollout plan; the others each cover one part of it.

| Document | What it covers |
|---|---|
| [Rollout plan](migration-overview.md) | The steps from the prototype to native Web Monetization in Chrome, and what each one needs approved |
| [The monetization link](migration-monetization-link.md) | `<link rel=monetization>`: its spec status and what Chromium lands for it |
| [The prototype and the extension, compared](migration-native-vs-extension.md) | Where the two implementations agree and differ, and what each hand-over needs |
| [Gating](migration-gating.md) | How each feature is switched on, from off to shipped |

Links marked **TBD** point to documents not published yet.

Feedback is welcome as issues or pull requests.
