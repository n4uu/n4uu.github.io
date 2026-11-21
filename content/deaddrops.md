---
title: PolinRider dead drops
build:
  list: never
---

PolinRider's loaders don't carry their C2. They read it from public blockchains when they run, so the chains say where the operator's servers are right now, and every change the operator made is a dated transaction that can't be taken back. This page is that data, read straight from the chains each time the site builds.

It is produced by [`prowl deaddrop`](https://github.com/n4uu/prowl#blockchain-dead-drops), which only talks to Blockscout, TronGrid and a public BSC node. Nothing here comes from the operator's servers, no stage is downloaded, and no address on this page has been contacted. The background is in [the on-chain changelog post](/2026/polinrider-rat-changelog-on-chain/) and [part 2](/2026/polinrider-tron-to-nullreceiver/).

The C2 periods are also a CSV: [polinrider-c2-live.csv](/iocs/polinrider-c2-live.csv) (not defanged, one row per period, `current` marks the live one). The addresses are rented VPS IPs and get reassigned; use the dates.

{{< deaddrops >}}
