---
title: Indicators
build:
  list: never
---

Everything here was reported to [abuse.ch](https://abuse.ch/) (ThreatFox, URLhaus, MalwareBazaar) under this handle. Values are defanged; each post links a CSV.

## Contagious Interview raw TCP backdoor (2026-07)

| Indicator | Role | Platform |
| --- | --- | --- |
| `5.230.122[.]63:15152` | C2, plain TCP | ThreatFox |
| `hxxp://cleverstack-ext30341.vercel[.]app/api/handle` | follow-up payloads | ThreatFox, URLhaus |
| `hxxp://cleverstack-ext30341.vercel[.]app/api/g?st=...` | backdoor download | URLhaus |
| `hxxp://cleverstack-ext30341.vercel[.]app/api/package` | package.json staging | ThreatFox |
| [`281d4d60fc99cf7f54819defb1c7c14b9202d46890936c4c6b9ebe3ddc75dd2f`](https://bazaar.abuse.ch/sample/281d4d60fc99cf7f54819defb1c7c14b9202d46890936c4c6b9ebe3ddc75dd2f/) | backdoor | MalwareBazaar |
| [`6db4c3d2ab56227d36eff59d1dc9e13eadf9959996fab1e0b04591cc5e637d04`](https://bazaar.abuse.ch/sample/6db4c3d2ab56227d36eff59d1dc9e13eadf9959996fab1e0b04591cc5e637d04/) | backdoor | MalwareBazaar |
| [`f101fda6093f12758efae630bc265d6bad71479249685b7503c05e707424a276`](https://bazaar.abuse.ch/sample/f101fda6093f12758efae630bc265d6bad71479249685b7503c05e707424a276/) | batch dropper | MalwareBazaar |

## OtterCookie (2026-06)

| Indicator | Role | Platform |
| --- | --- | --- |
| `216.126.225[.]243:8085` | upload server | ThreatFox |
| `216.126.225[.]243:8086` | upload server | ThreatFox |
| [`049300aa5dd774d6c984779a0570f59610399c71864b5d5c2605906db46ddeb9`](https://bazaar.abuse.ch/sample/049300aa5dd774d6c984779a0570f59610399c71864b5d5c2605906db46ddeb9/) | main script | MalwareBazaar |
