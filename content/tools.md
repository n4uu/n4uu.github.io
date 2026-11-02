---
title: Tools
build:
  list: never
---

All three are written in Go, MIT licensed, built in CI from tagged commits, with signed build provenance on every release.

## [crumble](https://github.com/n4uu/crumble)

Config extractor and C2 traffic decoder for the Node.js implants of the Contagious Interview campaign. Undoes javascript-obfuscator statically, then emulates the sample in a sandbox where every Node.js API that reaches the outside world only records the call. Gets the C2, the socket.io or TCP protocol, the tasking and any staged code. Also rebuilds socket.io sessions from a pcap. Ships YARA, Sigma and Suricata rules with positive and negative tests. [Validated on ten real samples](https://github.com/n4uu/crumble#validation-on-real-samples), failures included.

## [prowl](https://github.com/n4uu/prowl)

Hunts the campaign's infrastructure from public telemetry. Seeds from ThreatFox, pivots through URLhaus, certificate transparency, DNS and ASN data, and puts a probability on every indicator with the evidence behind it. Keeps CDNs and shared hosting out of the output. Exports MISP and STIX 2.1. [Summary of a real run](https://github.com/n4uu/prowl#a-real-run).

## [graft](https://github.com/n4uu/graft)

Finds compromised releases in npm dependency trees: a trusted package republished with one new dependency grafted on, whose install script does the work. The recipe behind the axios and Mastra compromises of 2026.
