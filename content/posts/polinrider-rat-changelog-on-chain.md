---
title: "A DPRK RAT's changelog, read off the blockchain"
date: 2026-11-17
draft: true
slug: "polinrider-rat-changelog-on-chain"
description: "PolinRider's loaders fetch their payloads from TRON and BNB Smart Chain, so every build the operator shipped is still on chain. Reading all 190 of them gives the DEV#POPPER RAT a dated changelog, a Python port nobody mentioned and the operator's own test runs."
tldr: "Seven TRON addresses pointed PolinRider loaders to 190 BSC transactions between June 2025 and October 2026, all sent by one wallet. 187 decrypt with the published keys. 95 are readable builds of the DEV#POPPER RAT, which grows from 7 commands to 15 over eight months before switching obfuscator. The archive also holds a Python port of the RAT published once in March 2026, three C2 addresses I could not find in public reporting, and the operator's tests, ending with a payload that is just the string `'error'`."
iocs: "/iocs/polinrider-rat-changelog-on-chain.csv"
---

## Summary

The loaders that PolinRider plants in GitHub repositories don't carry their
next stage. They read a TRON address's latest transaction, take a BNB Smart
Chain (BSC) transaction hash out of its memo, and decrypt that transaction's
input. The operator ships a new build by sending a new transaction, and
nothing is ever deleted. So the chain holds every build that was ever
served, in order, with timestamps.

I walked the full history of the seven TRON index addresses I could tie to
the campaign. They hold 200 transactions from 6 June 2025 to 7 October
2026. 190 of them point to BSC transactions, all sent by the same wallet,
`0x9bc1355344b54dedf3e44296916ed15653844509`, to the burn address. 187
decrypt with keys that are already public, into 176 distinct scripts.
95 of those are builds of the DEV#POPPER RAT that eSentire described in
March (89 distinct files), and 22 more are later builds in a different
obfuscator. Lining them up gives the RAT a changelog:

![Payloads published on chain by TRON index address, June 2025 to October 2026, and RAT commands per build rising from 7 to 15](/img/polinrider-on-chain-builds.svg)

This infrastructure is not new. Ransom-ISAC's XCTDH series and Crystal
Intelligence traced the BSC wallet, its funding and the RAT's C2 addresses
through November 2025, and eSentire, SafeDep, Socket, Greenflagged and Trend
Micro documented individual builds. What I could not find in public
reporting is:

- the build-by-build changelog: when each command and persistence marker
  first shipped, and the obfuscator switch on 19 May 2026;
- a Python port of the RAT published once, on 10 March 2026, from a TRON
  address used for nothing else;
- three server addresses: `154.91.0[.]196`, `154.91.0[.]15` and
  `89.19.222[.]25`. None was on ThreatFox when I checked on 17 November;
- a second funding wallet that created three of the index addresses within
  two minutes;
- the operator's test runs, which are also on chain.

## How it was analysed

Nothing was run, and nothing talked to the operator's servers. Every
request went to public blockchain APIs: TronGrid for TRON and a public BSC
RPC node. The decoding is the one Ransom-ISAC, SafeDep and StepSecurity
published. Take the TRON memo, hex-decode it and reverse it to get a BSC
transaction hash. Take that transaction's input, hex-decode it, split it
on `?.?` and XOR the middle part with the loader's key. Four keys from
earlier reports cover all but three transactions.

The decrypted scripts went through [crumble](https://github.com/n4uu/crumble),
which resolves javascript-obfuscator's string array statically and can
emulate a script against Node.js stubs that only record what it calls. I
used the emulation only on the small loaders, to see which address or
server each one reads next.

One check that the pipeline is right: BSC transaction
`0x804b000af7d7e4337ba5db28bb367da64a08391de09ffb07847ac897c5f82954`,
published on 27 January 2026, decrypts to a file with SHA-256
`207b5feb80c67f02023a07c7271b48c1d8b07ea5b59d76f98c1b0b357fbabfb1`.
That is the hash eSentire lists as "DEV#POPPER RAT (Obfuscated)", and
crumble's output has the same thirteen commands as their table.

## The archive

| TRON index | What it serves | Transactions | Active |
| --- | --- | --- | --- |
| `TMfKQEd7TJJa5xNZJZ2Lep838vrzrs7mAP` | chain 1 loader; the 2025 builds read `TLmj13…` | 27 | Jun 2025 – Jun 2026 |
| `TXfxHUet9pJVU1BgVkBAbrES4YUc1nGzcG` | chain 2 loader, which fetches `/$/boot` from a C2 | 9 | Jun 2025 – Jun 2026 |
| `TLmj13VL4p6NQ7jpxz8d9uYY6FUKCYatSe` | RAT builds, first stream | 58 | Jun 2025 – Jan 2026 |
| `TA48dct6rFW8BXsiLAtjFaVFoSuryMjD3v` | RAT builds, second stream | 62 | Nov 2025 – Jun 2026 |
| `TCqf6ZkaQD84vYsC2cuu1jRwB6JveTaRrF` | 2026 loaders, chain 1 | 29 | Nov 2025 – Oct 2026 |
| `TFMryB9m6d4kBMRjEVyFRbqKSV1cV2NcpH` | 2026 loaders, chain 2 | 14 | Nov 2025 – Oct 2026 |
| `TPDBqDKAYfRVvVYDNZVxY8CRfovkwDaSgW` | the Python RAT, once | 1 | 10 Mar 2026 |

Every BSC payload transaction, all 190, goes from `0x9bc1…4509` to
`0x000000000000000000000000000000000000dEaD`. That makes this the original
EtherHiding transaction shape with one more hop in front, and it means a
single sender address saw every build. Ransom-ISAC found that wallet's
first malicious transaction on 7 February 2025, and OpenSourceMalware
found the same wallet behind the hijacked `itsa-*` npm packages of May 2025.
Anyone watching that address would have seen each new RAT build as it was
published.

The Aptos fallback addresses in the loaders are real accounts. They have
sent 22, 10, 63 and 65 transactions, close to the counts of the TRON
addresses they back up. The public Aptos fullnode no longer serves that
history, so this post covers TRON and BSC only.

## The changelog

The RAT is javascript-obfuscator output with an RC4 string array, and
crumble reads 95 of the builds up to 18 March 2026. Command names are the
ones eSentire documented. The date is the first build that carries each
change:

| First published (UTC) | Change |
| --- | --- |
| 12 Jun 2025 | First build, 31 KB, 7 commands: `ss_info`, `ss_ip`, `ss_upf`, `ss_upd`, `ss_dir`, `ss_fcd`, `ss_stop`. C2 `136.0.9[.]8`, `166.88.4[.]2` |
| 17 Jun 2025 | Size doubles to 62 KB. `+ss_eval`. First persistence marker, `/*C250617A*/` |
| 19 Jun 2025 | `+ss_inz`, which writes the stager into other applications' JavaScript files |
| 20 Jun 2025 | C2 `23.27.202[.]27` added |
| 19 Aug 2025 | `+ss_inzx`, which removes it again |
| 1 Oct 2025 | Second marker, `/*C250618A*/` |
| 30 Oct 2025 | C2 `202.155.8[.]173` added |
| 14 Nov 2025 | Second stream starts on `TA48d…`, with its own key and only `198.105.127[.]210` |
| 19 Nov 2025 | `+ss_connect`, which switches C2 at runtime |
| 16 Dec 2025 | `+ss_eval64`, `+ss_cb` (clipboard). Third marker, `/*C250619A*/`, the next day |
| 30 Jan 2026 | Two builds carry a Python one-liner that runs whatever `hxxp://154.91.0[.]196:443/download/2` returns |
| 11–12 Feb 2026 | `+ss_exit`, `+ss_exit_f`. From here on no C2 address appears in clear text |
| 25 Feb 2026 | Fourth marker, `/*C250620A*/` |
| 18 Mar 2026 | Last build crumble can read: 75 KB, 15 commands |
| 19 May 2026 | The RAT stream switches obfuscator. 22 more builds up to 8 June |

The markers are already known: Trend Micro lists them and Ransom-ISAC's
YARA rule keys on `C250617A`. What the chain adds is dates. The first one, `C250617A`, shipped on
17 June 2025, which reads like a build date. The next three only count up,
and they arrived months apart. Each new marker is kept alongside the older
ones, presumably so that a new build recognises files that an old one
already injected.

From November 2025 to January 2026 two RAT streams ran side by side.
`TLmj13…` kept serving builds with the old C2 list, and from 1 December
it added `198.105.127[.]210`. `TA48d…` served only `198.105.127[.]210`.
The build eSentire analysed was the last one on the old stream.

From 19 May the RAT stream uses a different obfuscator: no
javascript-obfuscator string array, generator functions and its own string
decoding. Ransom-ISAC describes a new obfuscator in their September 2026
samples; I haven't checked whether it's the same one. crumble doesn't
handle it yet, so after 19 May I have sizes and dates but not contents.

## The operator at the keyboard

The first transactions on `TMfKQ…` are not payloads. On 6 June 2025 the
address received 100 TRX from the fee wallet Ransom-ISAC documented,
`TQdwohPCWqqfCUaCispyV1NaUZ1HgiJPUy`. It then sent three 1-sun transfers
with no memo, then one with the memo `Mxy custom memo text`, then the same
memo to the TRON zero address. Someone was finding out how memos work.

On 11 June it tried putting the code itself in a memo. Two transactions,
one minute apart, carry 1,024 and 8,192 bytes of a javascript-obfuscator
script encrypted with `cA]2!+37v,-szeU}`. Both are cut off mid-script.
The next day, 12 June, the first TRON memo holding a reversed BSC hash
appears, on `TLmj13…`, and that is the scheme the campaign has used since. The same key, unused in
between as far as the chain shows, fronted the second RAT stream from
November.

On 25 February 2026, at 18:05 and 18:07 UTC, the same RAT build was
published twice on `TA48d…`, first under `cA]2!+37v,-szeU}` and then
under `2[gWfGj;<:-93Z^C`. Everything after that uses the second key. That
was a busy evening: 12 RAT builds and 13 loaders in under four hours, one
of them with `global["_V"]="00"`.

Test markers left in published loaders: `_V='9-test'` on 1 October 2025,
`_V="00"` on 25 February, `_V='A-00'` on 23 May and `global.i='00-test'`
on 5 October 2026. That loader and its chain 2 twin, published in the
same minute, are the only ones that point to `89.19.222[.]25`, at
`:443/init` and `/0/body`.

The TRON side ends on 7 October 2026. That afternoon the chain 2 index,
`TFMry…`, got three payloads of 13 or 14 bytes in 13 minutes. The last one,
decrypted with that chain's own key, is the JavaScript literal `'error'`.
A loader still resolving chain 2 now evaluates a string and stops there.

## A Python port on a one-shot address

On 9 March 2026 the fee wallet `TQdwoh…` sent 100 TRX to a fresh address,
`TPDBqDKAYfRVvVYDNZVxY8CRfovkwDaSgW`. The next afternoon that address made
its only transaction, a pointer to a 10.5 KB payload that isn't
JavaScript. It's a Python RAT built on `asyncio`, with the same command
vocabulary: `ss_shell`, `ss_eval`, `ss_eval64`, `ss_cb`, `ss_upf`,
`ss_upd`, `ss_stop`, `ss_type`, `ss_info` and `ss_exit`. Its configuration
falls back to `SERVER_HOST=154.91.0[.]15`, `SERVER_PORT=443` and an upload
server at `hxxp://198.105.127[.]210`, where files go to `/u/f`. That is the
upload path OpenSourceMalware saw in the 2025 `itsa-*` chain.

`154.91.0[.]0/24` keeps coming back: `.103` was the 2025 bootstrap server,
`.196` serves the Python stage in two January 2026 RAT builds and `.15` is
the Python port's C2.

## The second funder

Ransom-ISAC traced the fees for the first index addresses to `TQdwoh…`. The
2026 addresses were paid from somewhere else.
`TGJ1YJJNkFYLZKU4ZGqJfS6dU2cvGac7pS` sent 50 TRX each to `TFMry…`,
`TCqf6…` and `TA48d…` at 15:02, 15:03 and 15:04 UTC on 13 November 2025.
That created all three addresses in one sitting, a day before `TA48d…`
published its first build. The same wallet topped up `TLmj13…` on 30
October, three minutes before that address published two builds, and sent
`TA48d…` another 100 TRX on 3 March 2026. `TLmj13…` was also funded by
`TQdwoh…` in June 2025, so both fee wallets serve the same set of
addresses.

## Why it matters

Blocking a C2 address does little here, because the chain decides where
the loader goes next. The chain is also the best telemetry anyone has on
this campaign. For defenders that means three things:

- **Watch the writer.** BSC transactions from `0x9bc1…4509` to the burn
  address have been the payload feed since February 2025. New ones are new
  builds.
- **Hunt the pattern, not the address.** A TRON memo that reverses to
  `0x` plus 64 hex characters is how every index in this post works, and it
  doesn't depend on any address:

  ```
  curl -s "https://api.trongrid.io/v1/accounts/<addr>/transactions?only_from=true&limit=200" |
    jq -r '.data[].raw_data.data // empty' |
    while read -r h; do echo "$h" | perl -ne 'chomp; print pack("H*", $_)' | rev; echo; done |
    grep -E '^0x[0-9a-f]{64}$'
  ```
- **Date the infection.** Builds changed C2 addresses over time, so the
  date a loader ran tells you which addresses to look for in proxy logs. The
  [CSV](/iocs/polinrider-rat-changelog-on-chain.csv) has every payload hash
  with its publish date, and each C2 address with the date it first appears.

## Indicators

| Indicator | Role | Status |
| --- | --- | --- |
| `0x9bc1355344b54dedf3e44296916ed15653844509` (BSC) | writes every payload | known (Ransom-ISAC, SafeDep) |
| `TGJ1YJJNkFYLZKU4ZGqJfS6dU2cvGac7pS` (TRON) | funded `TFMry…`, `TCqf6…`, `TA48d…`, `TLmj13…` | not found in public reporting |
| `TPDBqDKAYfRVvVYDNZVxY8CRfovkwDaSgW` (TRON) | one-shot index, Python RAT | not found in public reporting |
| `0x3414a658f13b652f24301e986f9e0079ef506992472c1d5224180340d8105837` (Aptos) | fallback for `TLmj13…` | not found in public reporting |
| `hxxp://154.91.0[.]196:443/download/2` | Python stage fetched by two Jan 2026 RAT builds | not found in public reporting |
| `154.91.0[.]15:443` | Python RAT C2 | not found in public reporting |
| `89.19.222[.]25:443/init`, `hxxp://89.19.222[.]25/0/body` | `00-test` loader, Oct 2026 | not found in public reporting |
| `hxxp://198.105.127[.]210/u/f` | Python RAT uploads | address known (Ransom-ISAC) |
| `4f7490cc2c0278f74097fb934d55dfaabfb6239e2fe27549895dc856b4d7a25b` | Python RAT | |

The seven TRON indexes are in the table above. Every payload hash with its
date, index and BSC transaction, plus the network indicators, is in the CSV
at the end of the post.

## What the samples did to my tooling

On the first pass crumble resolved 216 string lookups in a RAT whose
array has 1,467 entries. Two problems in the deobfuscator, both fixed:

1. **Wrappers that take their offsets from constant objects.** This preset
   routes every string lookup through small functions like
   `function aM(a, b) { return a0d(a - -a0d8.a, b); }`, called as
   `aM(a0db.a, 'zDx6')`, with `const a0db = {a: 479, d: 'Lr**'}`.
   crumble already folded constant objects whose values were strings, but
   gave up on any object with a number in it, so neither the wrapper nor
   the call ever looked constant. Numbers now fold too.
2. **Folding a string concatenation can orphan a parenthesis.** goja keeps
   no node for parentheses, so the span of `'a' + ('b' + 'c')` ends at the
   last literal, and replacing it left the `)` behind. The next parse failed,
   which silently ended the pass loop. The RC4 keys made this worse,
   because some of them contain parentheses (`'5)#)'`, `')J^Z'`). The fix
   widens the span until its parentheses balance, skipping quoted text.

With both, the same file resolves 1,491 calls with no failures, and every
command eSentire lists is in the output.

## References

- Ransom-ISAC, [Cross-Chain TxDataHiding Crypto Heist, parts 1 to 4](https://ransom-isac.org/blog/cross-chain-txdatahiding-crypto-heist-part-4/) and [XCTDH Adopts Hash Hiding](https://ransom-isac.org/blog/xctdh-adopts-hash-hiding/): the wallet, the funding trail and the C2 timeline up to November 2025.
- eSentire, [DEV#POPPER RAT and OmniStealer](https://www.esentire.com/blog/north-korean-apt-malware-analysis-dev-popper-rat-and-omnistealer-everyday-im-shufflin), and their [IOC list](https://github.com/eSentire/iocs/blob/main/DEV%23POPPER/DEV%23POPPER-DPRK-IoCs-02-20-2026.txt).
- SafeDep, [PolinRider switches to Ethereum C2](https://safedep.io/polinrider-ethereum-c2-github-repositories/) and [Joyfill](https://safedep.io/joyfill-npm-blockchain-c2-supply-chain/).
- StepSecurity, [Joyfill npm compromise](https://www.stepsecurity.io/blog/joyfill-npm-supply-chain-compromise).
- OpenSourceMalware, [PolinRider malware found in 2025 npm packages](https://opensourcemalware.com/blog/polinrider-malware-found-in-2025-npm-packages) and their [PolinRider dossier](https://github.com/OpenSourceMalware/PolinRider).
- Greenflagged, [@adminide-stack npm compromise](https://blog.greenflagged.dev/posts/2026-11-03-adminide-stack-eth-dead-drop/).
- Trend Micro, [Void Dokkaebi uses fake job interview lure](https://www.trendmicro.com/en_us/research/26/d/void-dokkaebi-uses-fake-job-interview-lure-to-spread-malware-via-code-repositories.html), for the `C2506xxA` markers.
