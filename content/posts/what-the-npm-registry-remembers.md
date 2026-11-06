---
title: "What the npm registry remembers"
date: 2026-11-06T16:15:00Z
draft: false
slug: "what-the-npm-registry-remembers"
description: "Removed npm releases leave a lot behind. Rebuilding the axios, easy-day-js and qix compromises from registry metadata alone, and turning that into a detector."
tldr: "npm keeps the publish time of removed versions, who published the live ones and how, and the security placeholders it puts in. With `curl` and `jq` that is enough to rebuild the axios, easy-day-js and qix compromises to the second, and it is what [graft](https://github.com/n4uu/graft) scores."
---

When npm takes down a malicious release, the tarball and its manifest go.
Most of the rest stays, and it is enough to rebuild an incident to the
second without touching a sample. This note covers what survives, how to
read it with `curl` and `jq`, and what it says about three compromises:
axios (March 2026), the `easy-day-js` dependency behind the Mastra
compromise (June 2026), and the phishing of the maintainer qix (September
2025). The last part is about turning it into a detector,
[graft](https://github.com/n4uu/graft), and what it took to stop that
detector flagging ordinary releases.

Every timestamp and field quoted here comes from registry.npmjs.org as of
2 November 2026. The commands are included so you can check them.

## The document behind a package

`https://registry.npmjs.org/<name>` returns a package's full metadata
document, which npm calls a packument. Three parts of it matter here:

- `versions`: one manifest per live version, which is the `package.json` as
  published plus fields the registry adds (`_npmUser`, `dist`, `gitHead`).
- `time`: a timestamp for every version ever published, plus `created` and
  `modified`.
- `dist-tags`: where `latest` and friends point.

`npm install` doesn't fetch this document. It asks for the abbreviated form
(`Accept: application/vnd.npm.install-v1+json`), which has `name`,
`modified`, `dist-tags` and `versions`, and no `time` at all. Anything built
on the install path never sees the part that matters most.

## Removed versions keep their timestamp

Unpublishing takes a version out of `versions` and leaves its entry in
`time`. The difference between the two is the list of versions that existed
and are gone:

```sh
curl -s https://registry.npmjs.org/axios | jq -r '
  .time as $t
  | ($t | keys) - (.versions | keys) - ["created", "modified"]
  | .[] | "\(.)  \($t[.])"'
```

```
0.30.4  2026-03-31T01:00:57.285Z
1.14.1  2026-03-31T00:21:58.168Z
```

Two removed versions in nearly twelve years of axios, both on the same night, 39
minutes apart, on two different release lines. Without reading anything
else, that is already the compromise.

When npm confirms malware in a package that exists only to carry it, it
goes further: the whole package is replaced with a `0.0.1-security`
placeholder, published by the `npm` account. Here is `plain-crypto-js`
today:

```sh
curl -s https://registry.npmjs.org/plain-crypto-js |
  jq '{time, versions: (.versions | keys), description}'
```

```json
{
  "time": {
    "created": "2026-03-31T04:26:32.907Z",
    "modified": "2026-03-31T04:26:35.731Z",
    "4.2.0": "2026-03-30T05:57:32.867Z",
    "4.2.1": "2026-03-30T23:59:12.278Z",
    "0.0.1-security": "2026-03-31T03:25:11.636Z",
    "0.0.1-security.0": "2026-03-31T04:26:33.055Z"
  },
  "versions": [
    "0.0.1-security.0"
  ],
  "description": "security holding package"
}
```

Look at `created`. It says 04:26 on 31 March, a day after both real
versions, because the document was rebuilt when the placeholder went in.
Ask today how old plain-crypto-js was when axios 1.14.1 came out and
`created` will tell you it didn't exist yet. The earliest version timestamp
is the one to trust.

## Who published, and how

Live versions say who published them. `_npmUser` is the account; a release
built by a CI workflow through trusted publishing (OIDC) shows up as
`GitHub Actions` with a `trustedPublisher` object, and `dist.attestations`
points to a provenance statement. The 1.x line of axios around the
incident:

```sh
curl -s https://registry.npmjs.org/axios | jq -r '
  .versions | to_entries[]
  | select(.key | test("^1\\.1[3-5]\\."))
  | [.key,
     (if .value._npmUser.trustedPublisher then "trusted publishing" else "account token" end),
     (if .value.dist.attestations then "provenance" else "-" end)]
  | @tsv'
```

```
1.13.0  account token       provenance
1.13.1  account token       provenance
1.13.2  account token       provenance
1.13.3  account token       -
1.13.4  trusted publishing  provenance
1.13.5  trusted publishing  provenance
1.13.6  trusted publishing  provenance
1.14.0  trusted publishing  provenance
1.15.0  trusted publishing  provenance
1.15.1  trusted publishing  provenance
1.15.2  trusted publishing  provenance
```

Two things in there are easy to get wrong. Provenance on its own doesn't
mean the release came from CI without a person's token: 1.13.0 to 1.13.2
have provenance and were published with an account token, which
`npm publish --provenance` from a CI job allows. Trusted publishing is the
stronger fact. And from 1.13.4 on 27 January, every 1.x release had gone
out that way, four in a row over two months. A 1.x release from an account
token, which is what public reporting says 1.14.1 was, broke that pattern.
The registry can't show it any more because the manifest is gone. On the
night, it could.

The 0.x line had no such pattern. Its 65 releases before the incident were
all published from an account token, so 0.30.4 looked like every 0.x
release before it. That is why graft compares a release with the previous
one on the same line: set against 1.14.0, 0.30.4 would have been flagged
for a publishing change the 0.x line never had.

The configuration ids tell the end of the story. 1.13.4 to 1.14.0 share one
`trustedPublisher.oidcConfigId`; 1.15.0 uses a new one. The 0.x line moved
to trusted publishing too with 0.31.0 on 12 April, under the same new
configuration.

## The night of 30 March

| UTC | Registry record | Gap |
| --- | --- | --- |
| 30 Mar 05:57:32 | `plain-crypto-js@4.2.0`, first version of a new package | |
| 30 Mar 23:59:12 | `plain-crypto-js@4.2.1` | |
| 31 Mar 00:21:58 | `axios@1.14.1`, since removed | 23 min after 4.2.1, 18 h after the package appeared |
| 31 Mar 01:00:57 | `axios@0.30.4`, since removed | 39 min after 1.14.1 |
| 31 Mar 03:25:11 | `plain-crypto-js@0.0.1-security`, published by `npm` | 3 h 03 min after 1.14.1 |
| 8 Apr 16:09:38 | `axios@1.15.0`, trusted publishing and provenance | |

The registry can't say what 1.14.1 contained. Public reporting (Microsoft,
Tenable) does: nothing changed but one new dependency,
`plain-crypto-js@^4.2.1`, whose postinstall ran `node setup.js`. Read
against the table, the dependency was 18 hours old when axios started
pulling it in, and the version carrying the payload was 23 minutes old.
What the registry doesn't keep is when the axios versions were removed:
`modified` moves with every change to the document, so it only gives the
last one.

## easy-day-js: a copy, not a theft

The Mastra compromise used a package that pretended to be dayjs. Comparing
the two documents:

```sh
curl -s https://registry.npmjs.org/dayjs > dayjs.json
curl -s https://registry.npmjs.org/easy-day-js > easy-day-js.json
jq -n --slurpfile a dayjs.json --slurpfile b easy-day-js.json '
  ([$a[0].versions[]._npmUser.name] | unique) as $pa
  | ([$b[0].versions[]._npmUser.name] | unique) as $pb
  | {same_description: ($a[0].description == $b[0].description),
     same_repository: ($a[0].repository.url == $b[0].repository.url),
     publishers_in_common: ($pa - ($pa - $pb) | length)}'
```

```json
{
  "same_description": true,
  "same_repository": true,
  "publishers_in_common": 0
}
```

The description is copied byte for byte, trailing space included. The
`repository` field points at github.com/iamkun/dayjs. The first version,
1.11.21 on 16 June, matched dayjs's own latest release, published on 26
May. Version 1.11.22 followed 18 hours later, at 01:01 UTC on 17 June, and
is gone; dayjs's real 1.11.22 came out on 16 August. Nothing was stolen to
publish easy-day-js. The evidence is the claim: a package that says it is
dayjs, published by an account that has never published dayjs.

## chalk: where metadata runs out

On 8 September 2025 a phished maintainer account published malicious
patch releases of chalk, debug, ansi-styles, supports-color and others,
between 13:12 and 13:20 UTC. `chalk@5.6.1` (13:13:05) is in `time` and not
in `versions`; 5.6.2 replaced it at 14:47:54, an hour and 35 minutes later.

Package by package there is nothing to see. Each release came from the
account that always published it, through the usual path, with no new
dependency. The only metadata signal is the burst: one account releasing
packages from several unrelated projects within minutes. After that the
code has to be read.

## From notes to a detector

graft reads the packument for every package in a lockfile, compares each
release with the previous one on its line, and scores what changed:
trusted publishing or provenance dropped, a young dependency with an
install script, a lookalike name, a removed version, a security
placeholder, a burst across unrelated projects. Its `-at` flag evaluates
against the registry as it stood at a given moment. At 01:05 UTC on 31
March, with no list of known-bad versions, it puts axios 1.14.1 at critical
and 0.30.4 at high; the
[graft README](https://github.com/n4uu/graft#would-it-have-caught-it-on-the-night)
has the full breakdown and how the deleted manifests were rebuilt for the
test.

Catching the incidents was the easy half. The harder half was staying quiet
on normal trees, so the weights were tuned live against two real lockfiles,
the Apache Superset frontend (3,146 packages) and a small Astro site (384),
until neither produced a single finding at medium or above. The false positives
taught more about npm than the incidents did:

- **Coordinated releases.** The first burst signal counted repositories and
  fired on sindresorhus shipping chalk, ansi-styles, ansi-regex and
  wrap-ansi within 23 minutes on 8 September 2025. Those were the clean
  releases replacing the qix ones. They all live under one owner; the qix
  burst spanned three. The signal now counts owners.
- **Projects splitting themselves up.** `sharp@0.34.5` added a day-old
  `@img/sharp-linux-riscv64`. Young, but from the same people and the same
  repository, which plain-crypto-js was not.
- **Forks.** `@zkochan/js-yaml` keeps js-yaml's repository field. A fork
  under the author's own scope is how forks are supposed to be named.
- **Old neighbours.** `preact` has been one edit away from `react` for a
  decade. A lookalike name only counts for packages under a year old or
  with few releases.
- **0.x minors.** Under 1.0 every minor is its own line, so the first
  release of a new 0.x minor has no predecessor on its line. Falling back
  to the same major fixed it.

## What the registry won't tell you

- **What a removed version contained.** The manifest and tarball are gone.
  If you want to answer that later, keep copies of the documents as you
  fetch them.
- **When something was removed.** Only the publish time survives.
- **A code-only compromise.** A release from the usual account, through
  the usual path, with no new dependency, looks like any other. The burst
  is the one hint, and only when there is a burst.
- **Anything about projects that never adopted trusted publishing.** For a
  package always published by hand, a stolen token produces a release that
  looks exactly like the last one.

So the registry is a good witness for supply-chain incidents that add
something, and a poor one for those that only change code. Check before
installing, not only after.

## Sources

- Microsoft, [Mitigating the axios npm supply chain compromise](https://www.microsoft.com/en-us/security/blog/2026/04/01/mitigating-the-axios-npm-supply-chain-compromise/)
- Tenable, [FAQ about the axios npm supply chain attack by UNC1069](https://www.tenable.com/blog/faq-about-the-axios-npm-supply-chain-attack-by-north-korea-nexus-threat-actor-unc1069)
- SecurityWeek, [North Korean hackers blamed for Mastra npm supply chain attack](https://www.securityweek.com/north-korean-hackers-blamed-for-mastra-npm-supply-chain-attack/)
- Check Point, [The great npm heist](https://blog.checkpoint.com/crypto/the-great-npm-heist-september-2025/)
- npm docs: [trusted publishing](https://docs.npmjs.com/trusted-publishers), [provenance](https://docs.npmjs.com/generating-provenance-statements), [unpublish policy](https://docs.npmjs.com/policies/unpublish)
