---
title: "Settle a payment: thirteen fake coding tests and one git hook"
date: 2026-11-10
draft: true
slug: "settle-a-payment-git-hook"
description: "Thirteen fake take-home tests, one per language, ship with a post-checkout hook that fires on the exact git command the README asks for. The payload is an OtterCookie loader I read without running any of it."
tldr: "On 1 November someone uploaded 13 fake coding tests and 29 JavaScript files to MalwareBazaar. Every test ships with its `.git` folder and the same `post-checkout` hook, and the README asks the candidate to run `git checkout -b candidate/<your-name>`, which fires it. The JavaScript is an obfuscated loader carrying OtterCookie modules that talks to nine servers on ports 8085 to 8087. This post covers the lure, how I resolved the loader statically, what it carries and rules for both."
iocs: "/iocs/settle-a-payment-git-hook.csv"
---

## Summary

On 1 November 2026 someone uploaded 42 files to MalwareBazaar under the tags
ContagiousInterview and github: 13 zip archives and 29 JavaScript files.

The archives are one take-home test, "Settle a payment", written once per
language: C++, C#, Go, Haskell, Java, JavaScript, Kotlin, Move, Python, Ruby,
Rust, Solidity and Swift. Each one ships with its `.git` folder, and each
`.git` holds the same `post-checkout` hook. The README tells the candidate to
start with `git checkout -b candidate/<your-name>`. That command is what runs
the hook, which fetches a script from `parallelchain-labs[.]com` and pipes it
into `sh`, or into `cmd` under Git for Windows.

The 29 JavaScript files are one loader, 3.4 to 4.5 MB each, carrying OtterCookie
modules as split strings. I resolved all 29 without running any of their code.
They talk to nine servers, always on ports 8085, 8086 and 8087.

On 10 November nothing in signature-base caught the hook or the loader, and
neither did my own rules in crumble. Rules for both are at the end.

One caveat up front: I did not fetch the hook's URL, so I can't show that it
serves these exact files. They were uploaded together, and the stage domain and
all nine servers went to ThreatFox from the same reporter within the same
minute on 2 November.

## The tests

All 13 have a `SPEC.md`, a `README.md`, one small function that sums amounts
and a test file for it. Each repo has a single commit, `init`, made on
10 October 2026 between 01:49 and 01:55 UTC by `elxwed <elxwed23523[@]tem[.]me>`
with a +0530 timezone. No remote is configured. The Python one still has its
`__pycache__` from CPython 3.12, so someone ran those tests before zipping.

The archive is the point. `git clone` never copies hooks: they live in
`.git/hooks`, which belongs to each clone and is not part of the repository.
A test handed out as a zip with `.git` inside keeps its hook, executable bit
included (0775 here). Unzip, read the README, create your branch, and the hook
has already run.

The hook is about 45 lines of comments explaining what `post-checkout` hooks
are for, followed by a `case` on `uname -s`. Darwin gets
`curl -s` of `/settings/mac?flag=2` piped into `sh`, Linux gets `wget -qO-` of
`/settings/linux?flag=2` piped into `sh`, and anything matching
`MINGW*|MSYS*|CYGWIN*` gets `curl -s` of `/settings/windows?flag=2` piped into
`cmd`. Output goes to `/dev/null` and the hook exits 0, so the checkout looks
normal. The same file (`0f08f087…`) is in all 13 tests.

## Reading the loader without running it

The loader is javascript-obfuscator output with most options turned on: a
string array of 20,600 to 26,200 entries, two decoders (base64, and RC4 with a
key per call), five-argument wrapper functions whose index offsets sit in
small objects, and every string split into short pieces. Decrypting the array
on its own gives fragments like `});` and `}\n}\nc`. The strings only exist
once the calls are resolved and the pieces glued back together.

For the backdoor in the [last post](/2026/raw-tcp-backdoor-cleverstack/),
crumble ran the sample's own decoder in an isolated VM. This time I did not
want to run any of the sample's code, not even its decoder. So the resolver:

1. parses the file with Babel, as data only;
2. reimplements both decoders;
3. finds the array rotation by evaluating the start-up checksum for every
   possible rotation until it hits the target (rotation 148 and 501950 for
   `d395f5cf…`);
4. replaces every wrapper call it can resolve with the decoded string (34,705
   in that file) and folds the concatenations.

About two minutes per file. All 29 resolved.

Some names survive in clear because global renaming is off: the C2 settings
`u_s`, `l_s`, `s_s` and `u_k`, and the helpers `f_s_l`, `s_u_c`, `gsi` and
`Utils`. The rule further down keys on them.

## What it carries

The loader builds three modules as strings, `ldbScript`, `autoUploadScript` and
`socketScript`, and starts each one as
`node --max-old-space-size=4096 --no-warnings -` with the source written to
stdin. The child is detached with a hidden window, and the module's code never
touches the disk. A lock file in the temp directory,
`pid.<t>.1.lock`, `.2.lock` or `.3.lock`, holds the child's PID and start
time so a second run doesn't start it twice.

- **socketScript** connects to port 8087 with socket.io. The server sends
  `whour`, the implant answers `whoIm` with its key, campaign value and host.
  Command code 102 lists a directory, 107 uploads a file, 108 uploads a
  directory (up to 25 MB per file), and anything else runs as a shell command.
  `processControl` starts and stops the other modules.
- **autoUploadScript** walks the home directory for names on a long list:
  `.env`, SSH keys (`id_rsa`, `id_ed25519`), keystores, anything with "seed",
  "bank" or "wallet", config files, notes and images. It has a separate pass
  for `.env` files, skips files over 10 MB and uploads to port 8086.
- **ldbScript** goes after browsers: 40 extension IDs (wallets such as
  MetaMask, Phantom, Coinbase Wallet, TronLink and Binance Wallet), saved
  passwords from `Login Data` copied to the temp folder and read with sql.js,
  DPAPI decryption on Windows through throwaway PowerShell scripts
  (`decrypt-key-*.ps1`), and the macOS login keychain. Uploads go to port 8085.

There is also a clipboard watcher that uses PowerShell on Windows, and under
WSL as well.

I call this OtterCookie. My crumble rule for its collector matches the
resolved code once its 2 MB size limit is lifted. The same 8085/8086 upload ports
show up in the June OtterCookie samples I took apart for crumble, and
ThreatFox has had `144.172.94[.]202:8087` tagged OtterCookie since 9 June.

## Infrastructure

| C2 | Loader builds |
| --- | --- |
| `144.172.94[.]202` | 11 |
| `172.86.125[.]98` | 9 |
| `45.61.145[.]161` | 3 |
| `144.172.118[.]166` | 1 |
| `144.172.85[.]176` | 1 |
| `144.172.86[.]31` | 1 |
| `172.86.125[.]61` | 1 |
| `172.86.126[.]76` | 1 |
| `172.86.73[.]27` | 1 |

Every build points all three ports at one host. CMPC9 reported all nine to
ThreatFox on 2 November as unknown malware, in the same minute as the stage
domain. The exception is `144.172.94[.]202:8087`, reported in June.

`u_k` runs from 101 to 110, and `144.172.94[.]202` has at least one build for
each value. It looks like numbered slots. I don't know what they map to.

The early build of the TCP backdoor from the last post used `144.172.105[.]138`,
in the same /16 as four of these. That is a hosting choice, not attribution.

## Detection

Three YARA rules, in a pull request to signature-base:

- a generic one for shell scripts that choose a payload with `uname` and pipe
  a remote script into both `sh` and `cmd`;
- one for this campaign's hook;
- one for the loader, on the clear names above.

They match all 13 hooks and all 29 loaders. They match nothing in 50,362 files
from 72 npm packages, 15 real `curl | sh` installers (rustup, nvm, Homebrew,
Docker, bun, uv and others), git's sample hooks, or 10 benign files run
through javascript-obfuscator with aggressive presets.

To hunt on developer machines:

- any file in `.git/hooks` that isn't a `.sample`, in a repo that arrived as an
  archive;
- git starting `sh`, which starts `curl` or `wget` piped into `sh` or `cmd`;
- `node --max-old-space-size=4096 --no-warnings -`, a Node.js process reading
  its script from stdin, with another `node` as parent;
- `pid.*.1.lock`, `.2.lock` and `.3.lock` in the temp folder, and
  `decrypt-key-*.ps1` next to them on Windows.

## Indicators

| Indicator | Role |
| --- | --- |
| `parallelchain-labs[.]com` | stage host for the hook |
| `hxxps://parallelchain-labs[.]com/settings/{mac,linux,windows}?flag=2` | stage, piped into `sh` or `cmd` |
| nine hosts above, ports 8085, 8086, 8087 | OtterCookie upload and control |
| `0f08f08737dd5b6dfd76f396e361b7d795156888883fa89139a73b9934c64a24` | the `post-checkout` hook |
| `elxwed23523[@]tem[.]me` | git commit author in all 13 tests |

All 13 archives, the 29 loaders and the rest are in the CSV at the end.

## References

- MalwareBazaar, e.g. [the C++ test](https://bazaar.abuse.ch/sample/13e558bf5c80b16595161327d3104e11ae9fc8deab4b96917bb01278931891cf/) and [a loader](https://bazaar.abuse.ch/sample/d395f5cf1c1b049acaac87586787cdace39e8be51b9ffdedf90bad28088a43e5/).
- ThreatFox entries for the stage domain and the nine C2s (CMPC9, 2 November) and for `144.172.94[.]202:8087` (June, tagged OtterCookie).
- [crumble](https://github.com/n4uu/crumble), whose OtterCookie collector rule matches the resolved code.
- [A raw TCP backdoor behind the cleverstack git hook lure](/2026/raw-tcp-backdoor-cleverstack/), the same lure family earlier this year.
