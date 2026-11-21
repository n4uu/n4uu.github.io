---
title: "PolinRider after TRON: the builds nobody read and where the index went"
date: 2026-11-21
draft: false
slug: "polinrider-tron-to-nullreceiver"
description: "Part 2. The 22 RAT builds from May and June are JS-Confuser on top of javascript-obfuscator, and with their publish times they show three obfuscator settings in four days, two builds that could never run and a debugging session. On 7 October the TRON index was replaced by an Ethereum wallet whose transactions point to BSC stages by block and position, and the wallet that pays for the C2 bot's gas behaves like a sweep wallet."
tldr: "The May and June builds that crumble couldn't read are JS-Confuser run over the same javascript-obfuscator output; decoded and dated from the chain, they include two builds that crash on load and a 26-minute debugging session. On 7 October the loaders stopped reading TRON and started reading `0x596dcc9f…91bb` on Ethereum, whose transactions point to a BSC block and a position in it. It is still publishing, every 256 blocks. The gas for the NullReceiver C2 bot comes from `0xf97062cb…`, which collects whole-balance deposits from many addresses and sends them out through LI.FI. 51 stages the BSC writer published since 7 October are still not pointed to by anything I know."
iocs: "/iocs/polinrider-tron-to-nullreceiver.csv"
---

[Part 1](/2026/polinrider-rat-changelog-on-chain/) ended with two loose ends. From 19 May the RAT builds published on TRON switched obfuscator and I couldn't read them, and on 7 October the TRON side stopped, the last payload being the string `'error'`. Both have answers now, and the second one is better than a dead end: the archive didn't stop, it moved to Ethereum.

Part of this is already public, and it's worth saying what before adding to it:

- OpenSourceMalware named NullReceiver on 2 August: a loader that reads its C2 out of the recipient address of a zero-value Ethereum transaction. GBHackers, The Hacker News and Sonatype followed.
- [SafeDep](https://safedep.io/polinrider-ethereum-c2-github-repositories/) (30 September) decoded the variant that rounds the Ethereum block number down to a multiple of 256, looks for a sender containing a fixed string, and reads two `IP:port` pairs out of the recipient, with the wallet `0x33ff…9891`.
- [Greenflagged](https://blog.greenflagged.dev/posts/2026-11-03-adminide-stack-eth-dead-drop/) (3 and 4 November) took apart the `@adminide-stack` npm compromise. That write-up covers the current NullReceiver wallet `0x1eef…cef3` and its C2 history, the 1000-block lookup, the stage runner that types the code into a `node -i` REPL, and the RAT's persistence in npm and in the editors' startup files.
- SafeDep, [PolinRider switches to Ethereum C2 in 30+ repositories](https://safedep.io/polinrider-ethereum-c2-github-repositories/) (30 September 2026)
- Greenflagged, [@adminide-stack npm compromise: an Ethereum dead drop](https://blog.greenflagged.dev/posts/2026-11-03-adminide-stack-eth-dead-drop/) (3 November 2026)
- [OsamaCodes62/nullreceiver-analysis](https://github.com/OsamaCodes62/nullreceiver-analysis) and community IOC sets such as [meSingh/polinrider-scanner](https://github.com/meSingh/polinrider-scanner) list most of the wallets and C2 addresses below.

I couldn't read every vendor post in full from where I work, so if something here is in one of them, it's theirs first. What I could not find anywhere, and what this post adds:

- the 22 builds from May and June, decoded statically and dated to the second from the chain, including two that could never run;
- the variant published on 7 October, whose Ethereum pointers no longer hold an IP but a BSC block and position, which is where the TRON index went and how the Ethereum side connects to the BSC writer from part 1;
- the wallet that pays for the NullReceiver bot's gas, and what else goes through it;
- 51 stages the BSC writer published since 7 October that nothing I know points to;
- a [live page](/deaddrops/) that reads all of it from the chains every six hours, and the tool behind it.

## How it was analysed

The same rules as part 1. Nothing was run. Every request went to public chain APIs: TronGrid, Blockscout for Ethereum, public Ethereum and BSC RPC nodes, and MalwareBazaar for six loader samples tagged PolinRider on 7 November. No address in this post was contacted. The JavaScript was parsed and rewritten as a syntax tree; the only code evaluated was a small arithmetic interpreter of my own, explained below.

## JS-Confuser on top of the old obfuscator

The builds published from 19 May didn't replace javascript-obfuscator. The operator ran [JS-Confuser](https://github.com/MichaelXF/js-confuser) over the same javascript-obfuscator output. Under the JS-Confuser layer there is the usual string array, the getter with an offset (`0x8b` in the first build) and the rotation loop. crumble looked for a javascript-obfuscator string array and found none, because JS-Confuser had hidden its strings.

Three JS-Confuser settings went out in four days:

| Published (UTC) | Build | VERSION | JS-Confuser options |
| --- | --- | --- | --- |
| 19 May 16:13 | `10c473fa…` | | String Concealing, zero-width identifiers. Doesn't parse |
| 19 May 17:25 | `ca5d0730…` | | the same. Doesn't parse |
| 19 May 17:32 | `f5e5e576…` | 260519 | String Concealing |
| 20 May 15:05, 15:07 | `b3fa4550…`, `04c15374…` | 260520 | String Compression and Control Flow Flattening |
| 20 May 15:59 | `b77a2d17…` | 260521 | the same, plus Pack |
| 21 May 12:38 to 13:14 | five builds | 260522 to 260524 | Compression and flattening, no Pack |
| 21 May 13:20 | `6e04db01…` | 260525 | plus Pack |
| 22 May to 8 June | ten builds | 260526 to 260605 | Compression, flattening and Pack |

`VERSION` is what the RAT reports when it connects. It reads like a date and it starts as one, but it is a counter written as YYMMDD: by the afternoon of 21 May it had reached 260525, and after 260527 on 23 May the next build is 260601.

All three settings come off without running anything:

- **String Concealing** encodes every string with basE91 and a shuffled alphabet that is different in every function scope, behind accessor functions that cache what they decode. Each accessor resolves to the decoder in its own scope, so each call site gets decoded with the right alphabet.
- **String Compression** joins all strings into one LZ-String blob (`decompressFromUTF16`, split on `|`).
- **Pack** puts the whole program in a string passed to `Function()`. Parsing that string is enough.

What's left is javascript-obfuscator. Its rotation loop shuffles the array until a checksum matches: `-parseInt(get(0xe7))/1 * (parseInt(get(0xf2))/2) + …`. That expression is only arithmetic on `parseInt()` of array entries, so I evaluate it with a few lines of my own over each rotation of the array until it equals the target. For the first build that is rotation 101. After that the getter calls resolve to literals and the code reads like the March builds did. The decoder is a Babel script for now; it is going into [crumble](https://github.com/n4uu/crumble) next.

## Two builds that could never run

The first two JS-Confuser builds use the zero-width identifier generator, so a name like `of` is followed by U+200C. The operator's publishing tool treats the file as one character per byte, XORs it and writes the result as UTF-8. The three bytes of each U+200C come back as three separate characters, `â`, U+0080 and U+008C. The loader decodes the transaction exactly the same way, `Buffer.from(input, 'hex').toString('utf8')`, then XOR with `charCodeAt`, then `eval`, so what it evaluates has U+0080 in the middle of an identifier. That is a syntax error on the first line:

```
SyntaxError: Unexpected character '\x80' (1:12)
  function ofâ(){}var inâ=Object['defineProperty']…
```

The loader always reads the newest transaction on its TRON index (`limit=1`), so from 16:13 to 17:32 on 19 May every infected machine that resolved `TA48d…` got a build that dies on load, and told the operator through the RAT's error channel (`failed to run clientCode: …`). The third build, seven minutes after the second, uses ordinary random names and parses.

It happened again on 8 September, on the chain 2 index `TFMry…`. A valid loader went out at 12:34. The ones at 13:02, 13:09 and 13:22 don't parse: two pieces were concatenated without a semicolon, `…return 4045})()var _$_a54a=(functio…`. At 13:23:57 a valid one appeared, built on a different design (`/0/body` instead of `/$/boot`, with a new key). For 21 minutes chain 2 served a loader that couldn't start.

Checking for this is cheap: decrypt the transaction and run it through a parser, not an interpreter.

## The changelog, May to June

Dates are the first build that carries each change. The injection targets are in OsamaCodes62's analysis; the chain adds when.

| First published (UTC) | VERSION | Change |
| --- | --- | --- |
| 19 May 17:32 | 260519 | Injects its stager into `@vscode/deviceid/dist/index.js` of VS Code, Cursor and Antigravity, GitHub Desktop's `main.js` and Discord's `discord_desktop_core`. Exits on CI and build sandboxes. Errors are POSTed to `<C2>/verify-human/<_V>` |
| 20 May 15:05 | 260520 | Marker `/*C260511A*/`. The path `/0x/js?_V=`, the first of the `/0x/` paths the NullReceiver servers use later |
| 21 May 12:54 | 260523 | `>>> 101` printed to the console |
| 21 May 13:02 | 260524 | `>>> 103 > `, `>>> 110` and `Error executing: ` |
| 21 May 13:09 | 260524 | `>>> 101 > `, `>>> 1011` |
| 21 May 13:20 | 260525 | All the `>>>` lines gone |
| 22 May 13:17 | 260526 | `global.i`, which becomes the campaign tag later |
| 23 May 16:28 | 260527 | Marker `/*C260512A*/` |
| 2 Jun 16:01 | 260601 | Reports a `FIRST_VISIT_TIME`, with new strings for a file in the temp directory (`Temp`, `/tmp/`, `.tsp`). From the 16:51 build the file name starts with a dot |
| 3 Jun 12:41 | 260603 | Reports its own `SOCKET_URL` and `UPLOAD_URL` |
| 5 Jun 14:31 | 260604 | Injects into npm itself: `npm root -g`, then `node_modules/npm/lib/cli.js`, marker `/*RS260605*/`. Every `npm` command then starts the stager |
| 8 Jun 21:29 | 260605 | Last build on `TA48d…` |

The 21 May rows are a person debugging. Four builds in twenty minutes, each with numbered trace lines that change from one to the next, then a fifth six minutes later with all of them removed, and the next day's build moves on.

Two checks keep the RAT away from its own developers. If the hostname is `EV-CHQG3L42MMQ` or `EV-4A6OE6M0E2D`, or the environment has `jsbot` set, injection becomes a dry run that only reports what it would have changed. I read those as the developers' test machines. On Windows it lists processes, hashes every name that starts with a capital `O` with MD5, and exits on `9a47bb48b7b8ca41fc138fd3372e8cc0`. I haven't found which process that is; it isn't `O` followed by up to six letters and `.exe`. On Linux outside WSL it exits as user `github-runner` or `buildbot`, or as root on hosts named `build-*`, `sandbox-pool-*`, `buildkitsandbox`, `cloudchamber` or `(none)`. Those are CI runners, Docker BuildKit and Cloudflare Containers, the places dynamic scanners run packages.

## The loaders pick a C2 by campaign

The C2 isn't in these builds at all. The loader sets it in globals before the RAT runs: `_t_s` for the socket.io server, `_t_u` for uploads and errors. From 20 May the chain 1 loaders choose those by the campaign tag `_V`, which the lure sets:

| `_V` | 20 to 23 May | 18 June | 23 June |
| --- | --- | --- | --- |
| starts with `A` | `166.88.54[.]158` | `23.27.13[.]43` | `166.88.134[.]62` |
| a number | `198.105.127[.]210` | same | same |
| anything else | `23.27.202[.]27` | same | same |

The `A` line moved every few weeks while the other two stayed put. Chain 2 does the same for `/$/boot`, with its own XOR key, `ThZG+0jfXE6VAGOJ`. In September the chain 2 loaders stopped pretending to be Chrome 131 and started sending `Python-urllib/3.13` as their User-Agent.

The second value the loaders carry, `_t_2`, is the Aptos fallback for when TronGrid doesn't answer. Each TRON index has one, and they move almost in step: `0x533b…` for `TA48d…` has sent 63 transactions against the index's 62, `0x9d20…` for `TCqf6…` 29 against 29.

## 7 October: TRON out, Ethereum in

The last two loaders published on `TCqf6…`, at 15:37 and 16:09 on 7 October, the afternoon chain 2 got its `'error'` placeholders, don't read TRON any more. The lookup is the one SafeDep described in September, with three changes: the offset is 5 instead of 3, the sender is a new wallet, and the recipient no longer holds a C2 address. With the strings decoded:

1. Take the Ethereum chain head, round it down to a multiple of 256 and add 5.
2. Look at that block and the two after it, then the same three blocks 256, 512, 1024 and so on back, up to 2^12 × 256 blocks.
3. In those blocks, find a transaction whose sender contains `596dcc9fda875309ff85eba09cf5a97d`. The loader never spells out the whole address.
4. Read that transaction's recipient. Bytes 0 to 3 are a BSC block number, byte 4 the position of a transaction in that block.
5. Fetch that BSC transaction, split its input on `?.?`, XOR with `8l%Sljl>J>U61O,n` and `eval` it.

The address is `0x596dcc9fda875309ff85eba09cf5a97d8c7691bb`. It has sent 1,289 transactions, from 16:29 on 6 October to 15:58 today, and 1,085 of them landed in a block whose number is 5, 6 or 7 above a multiple of 256: the bot republishes the pointer about every 51 minutes so the loader always finds a fresh one. Unlike NullReceiver, the pointers aren't zero-value: the first three carried 100 wei and every one since 500. Decoding the current one:

```
to      0x07b2bf3d 6b 07d96c1c3107d96d59280783e08100
        BSC block 0x07b2bf3d = 129154877, transaction 0x6b = 107
        -> 0x86fa4b3c15cef29510c02485d01955e2fe3a2be0d1ff23577028cf272347bb21
           from 0x9bc1355344b54dedf3e44296916ed15653844509, nonce 338
```

The sender is the same BSC writer as in part 1. Five stages so far:

| Published on BSC (UTC) | Writer nonce | Pointed to (UTC) | Pointers | SHA-256 |
| --- | --- | --- | --- | --- |
| 6 Oct 15:21 | 286 | 6 Oct 16:29 | 1 | `9d9161c9…` |
| 6 Oct 16:55 | 289 | 6 Oct 17:21 to 21:36 | 6 | `7313380f…` |
| 6 Oct 21:44 | 292 | 6 Oct 22:28 to 20 Oct 14:24 | 384 | `c069f232…` |
| 20 Oct 13:32 | 325 | 20 Oct 15:15 to 22 Oct 15:10 | 57 | `b88bf934…` |
| 22 Oct 15:35 | 338 | 22 Oct 16:01 to now | 841 | `79023ba1…` |

The index went live on the afternoon of 6 October, a day before the first loader that reads it was published on TRON. The stages are small, 50 to 55 KB of JS-Confuser Pack, and their string tables are loader code and `global.ix`, not the RAT; one of them carries the pointer to the first stage. I haven't taken them apart further.

The ETH for the index came from `0x33ff3edaf55a8e03dcbc7cb40d498a49cd499891`, on 6 and 31 October: the wallet SafeDep documented, which until then had published `IP:port` pairs on the same 256-block schedule. The operator funded the new index from the old one.

## NullReceiver, timed

The loaders on MalwareBazaar from 7 November use NullReceiver, the same stage 1 Greenflagged took apart; two of them carry their source in base64 rather than obfuscated, so it can be read as written. `SENDER` is `0x1eef0849eda15d87f64e51c73c8442cf3c8bcef3`. Greenflagged's table of its C2 addresses runs to 3 November; the full history is on the [live page](/deaddrops/). Three things the chain adds to it:

- **Two wallets at once.** Its predecessor, `0xa322e5f3d311d3080e6f0121063e9adc2490ef1a`, the wallet OpenSourceMalware published, didn't stop when the second started. They ran side by side from 4 September to 19 October and shared servers: `166.88.134[.]75` on both in September, and `23.27.13[.]72` on both on 14 October, less than five hours apart.
- **The schedule is kept.** The loader only looks at the multiple of 1000 below the chain head, the one before it, and one block either side of each. Of the first wallet's 607 pointers, 515 landed there; of the second's 347, 314. The bot republishes the current C2 about every 3 hours 20 minutes, and since 12 November that is `95.135.181[.]84`.
- **Gaps are money.** The second wallet went quiet from 12 September to 14 October. It resumed five minutes after it was sent ETH, which leads to the next section.

The campaign tags in these samples' `Sec-V` header are `A10-*2_2434`, `A10-**93` and `A10-**1`.

## Who pays for the gas

A bot that publishes every 1000 blocks needs ETH, and the chain shows where it comes from.

| UTC | From | To | ETH |
| --- | --- | --- | --- |
| 25 Jul 06:35 | `0x4e5b2e1d…` | first NullReceiver wallet | 0.0176 |
| 13 Oct 23:59 | `0xf97062cb…` | first wallet | 0.02 |
| 14 Oct 01:26 | first wallet | `0xf97062cb…` | 0.0293 |
| 14 Oct 01:47 | `0x4e5b2e1d…` | second wallet | 0.0207 |
| 16 Oct 05:16 | `0xf97062cb…` | first wallet | 0.007 |
| 11 Nov 03:34 | `0xf97062cb…` | second wallet | 0.06 |

`0x4e5b2e1d…`, which Greenflagged already noted as the second wallet's funder, also paid for the first. It holds about 1,290 ETH, is flagged on Blockscout and shows up in public rich-address lists. It is most likely an exchange or swap service, so it says where the ETH was bought, not who bought it. The second wallet's very first ETH, on 4 September, came through a verified `MulticallHandler` contract, the shape a cross-chain bridge delivery takes.

`0xf97062cb1b6ffa18b88e9faea373ed288a87ecd0` is more interesting. It took the first wallet's leftover ETH on the night the second wallet restarted, and it has paid gas for both. Its other transactions follow one pattern. On 3 September between 02:01 and 02:29 it received ten deposits from nine addresses, one after another. Six of the seven I looked at sent their whole balance and were left with exactly zero, which takes computing the gas to the wei: most likely a script holding the keys. Two hours later 2.15 ETH left for LI.FI's `LiFiDiamond` contract (`0x1231deb6…`), a cross-chain swap aggregator. More transfers to the same contract follow on 16 and 26 September.

That is what a sweep wallet looks like: something holding private keys, emptying the accounts and moving the money to another chain. In this campaign the obvious keys are the wallets stolen from infected developers. An operator consolidating their own wallets would look the same from the chain, and I can't tell the two apart from here. Either way this is the wallet that keeps the C2 bot running, and I found it in no public reporting or IOC list.

## 51 stages without an index

The BSC writer's next nonce today is 351. The last stage a TRON index points to is nonce 297, on 7 October, and the Ethereum index accounts for two after that, 325 and 338. That leaves 51 transactions written since 7 October that nothing I know points to. A node can't list an account's transactions, so I can't see them without a BSC indexer. They could be test publications, stages for an index I haven't found, or something a C2 response names. If you can query a BSC indexer for the writer's nonces 298 to 350, I'd like to compare notes.

## Watching it

[`prowl deaddrop`](https://github.com/n4uu/prowl#blockchain-dead-drops) reads all four kinds of dead drop in this post and part 1: the NullReceiver wallets, the Ethereum index, the TRON indexes and the writer's nonce. It prints the C2 periods, how many pointers kept to schedule, the ETH that came in and went out, and how many stages are still unaccounted for. It never contacts a C2 or downloads a stage. The [live page](/deaddrops/) is that output, rebuilt every six hours, and the C2 periods are a [CSV](/iocs/polinrider-c2-live.csv).

For defenders, three things follow:

- **On the chain.** The two active wallets, `0x1eef0849…` and `0x596dcc9f…91bb`, say where the C2 is and which stage is live, within the hour. So does the next wallet, as soon as a sample names it.
- **On the host.** As Greenflagged notes, the stages run in a `node -i` started detached by another `node` process with its stdin piped, which is unusual on a developer's machine. So is anything writing to `@vscode/deviceid/dist/index.js` or to npm's `lib/cli.js`.
- **On the wire.** Plain HTTP to port 443 with a `Sec-V` request header, and `HEAD` responses carrying `X-Payload-B64`.

## Indicators

| Indicator | Role | Status |
| --- | --- | --- |
| `0x596dcc9fda875309ff85eba09cf5a97d8c7691bb` (Ethereum) | index for BSC stages since 6 October, block and position in the recipient | address seen in a tracker's test data and in infected repositories; the pointer format not found in public reporting |
| `0x1eef0849eda15d87f64e51c73c8442cf3c8bcef3` (Ethereum) | NullReceiver C2 wallet, active | documented by Greenflagged |
| `0xf97062cb1b6ffa18b88e9faea373ed288a87ecd0` (Ethereum) | pays gas for both NullReceiver wallets; sweeps full-balance deposits to LI.FI | not found in public reporting |
| `0x9bc1355344b54dedf3e44296916ed15653844509` (BSC) | writes every stage | known (Ransom-ISAC, SafeDep) |
| `8l%Sljl>J>U61O,n` | XOR key for the Ethereum-indexed stages | |
| `q4FZkxX{!h,Sr3=@`, `y-p_>d$0B&@^1aQk` | XOR keys for `/0x/cls` and `/0x/ls` | in community IOC sets |
| `95.135.181[.]84` | current NullReceiver C2 (since 12 November) | on ThreatFox |
| `166.88.54[.]158`, `23.27.13[.]43` | May and June C2 for campaign tags starting with `A` | in community IOC sets |

Every build hash with its publish time and VERSION, the five Ethereum-indexed stages, the loaders, the keys, the Aptos fallbacks and every C2 period are in the [CSV](/iocs/polinrider-tron-to-nullreceiver.csv).

## Sources

- Part 1: [A DPRK RAT's changelog, read off the blockchain](/2026/polinrider-rat-changelog-on-chain/)
- OpenSourceMalware, NullReceiver (2 August 2026), and the [PolinRider dossier](https://github.com/OpenSourceMalware/PolinRider)
- SafeDep, [PolinRider switches to Ethereum C2 in 30+ repositories](https://safedep.io/polinrider-ethereum-c2-github-repositories/) (30 September 2026)
- Greenflagged, [@adminide-stack npm compromise: an Ethereum dead drop](https://blog.greenflagged.dev/posts/2026-11-03-adminide-stack-eth-dead-drop/) (3 November 2026)
- [OsamaCodes62/nullreceiver-analysis](https://github.com/OsamaCodes62/nullreceiver-analysis)
- The Hacker News, [Trojanized npm packages employ NullReceiver tactic](https://thehackernews.com/2026/08/trojanized-npm-packages-decode-c2-ip.html)
- OX Security, [PolinRider returns](https://www.ox.security/blog/polinrider-returns-hijacked-npm-packages-use-ethereum-blockchain-to-conceal-malware-infrastructure/)
- [JS-Confuser](https://github.com/MichaelXF/js-confuser), the obfuscator
- Blockscout, TronGrid and the public BSC and Ethereum RPC nodes, for everything on chain
