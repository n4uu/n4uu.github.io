---
title: Contagious Interview tracker
build:
  list: never
---

Waves of the Contagious Interview campaign (also tracked as Famous Chollima, DeceptiveDevelopment, Void Dokkaebi) that I have either taken apart myself or seen reported with enough detail to place. Newest at the bottom. Values are defanged. Everything I analysed is also in the [CSV](/iocs/contagious-interview-tracker.csv).

"Analysed" means I had the samples and went through them statically. "Reported" means someone else published it and I have not checked it against samples yet.

Last updated 10 November 2026.

| First seen | Lure and delivery | Payload | Infrastructure | Status |
| --- | --- | --- | --- | --- |
| 2026-04-20 | not known | Node.js raw TCP backdoor, early build with an RC4 string array (`1bcf17ce…`) | `144.172.105[.]138:15152`, `cleverstack-ext30341.vercel[.]app` | Analysed, [post](/2026/raw-tcp-backdoor-cleverstack/#update-6-november-2026) |
| 2026-06 | not known | OtterCookie (socket.io RAT with upload modules) | `216.126.225[.]243:8085`, `:8086` | Analysed, [crumble validation](https://github.com/n4uu/crumble#validation-on-real-samples) |
| 2026-07-10 | cleverstack git hook lure; a Windows batch stage installs a portable Node.js and pulls the backdoor | Node.js raw TCP backdoor, newline-delimited JSON over plain TCP (`281d4d60…`, `6db4c3d2…`) | `5.230.122[.]63:15152`, `cleverstack-ext30341.vercel[.]app` | Analysed, [post](/2026/raw-tcp-backdoor-cleverstack/) |
| 2026-10-21 | EtherHiding: the loader reads its next stage from a smart contract; NetSupport client dropped later | loader, NetSupport Manager | see the PR | Reported by popkir, [SigmaHQ/sigma#6500](https://github.com/SigmaHQ/sigma/pull/6500) |
| 2026-11-01 | "Settle a payment": 13 fake coding tests, one per language, shipped as archives with `.git` included. A `post-checkout` hook fires on the `git checkout -b candidate/<your-name>` the README asks for | obfuscated loader carrying OtterCookie modules | `parallelchain-labs[.]com`; nine C2 hosts on ports 8085 to 8087, e.g. `144.172.94[.]202`, `172.86.125[.]98` | Analysed, [post](/2026/settle-a-payment-git-hook/) |
| 2026-11-07 | PolinRider / NullReceiver: EtherHiding JavaScript, some files posing as fonts | loader | `166.88.134[.]62:443` and others on ThreatFox | Reported by iljakrasnjanski, shmulc, Isk0o |
