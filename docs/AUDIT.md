# ZenIRCd — Phase 0 Baseline Record & Phase 1 Technical Audit

*Started 2026-11-08. Status: **DRAFT, awaiting acceptance** (Phase 1 exit criterion).*

Labels used throughout:
- **[FACT]** checked against source code, a signed artifact, or a primary upstream source on the date above
- **[ASSUMPTION]** believed true, not yet checked
- **[PROPOSAL]** a recommendation that needs approval; nothing here is implemented

---

## Part A — Phase 0: Secure Baseline

### A.1 Baseline identity [FACT]

| Item | Value |
|---|---|
| Upstream | https://github.com/unrealircd/unrealircd (branch `unreal60_dev`) |
| Upstream release commit | `7a3f9df5c` "** UnrealIRCd 6.2.7 **" (2026-09-11) |
| ZenIRCd baseline commit | `a3172c9b4` "Import UnrealIRCd 6.2.7.1 security release on top of 6.2.7" |
| Release artifact | `unrealircd-6.2.7.1.tar.gz` from https://www.unrealircd.org/downloads/ |
| SHA-256 | `4e372ba071f3211d34c5a3515af77fc2b275eb692e43e55c423e201b60958c3f` |
| Signature | Good. RSA-4096 key `36E6 F657 06E3 6B09 3728 0299 1010 01DA F48B B56D` ("UnrealIRCd releases and patches"), signed 2026-11-05 |
| Key provenance | Published in upstream `doc/KEYS` by commit `3aa26ef1f` (2024-11-18), a channel separate from the download site |
| Tree check | `diff -rq` of the baseline tree against the extracted tarball: **identical** |
| Binary self-report | `UnrealIRCd-6.2.7.1` |

**Why not a git tag:** upstream publishes no git tags, and 6.2.7.1 exists only as a tarball and hot-patch. Upstream commit `6ac590da1` says "6.2.7.1 will not exist in git". So the baseline is the upstream 6.2.7 release commit with the tarball's 21 changed files applied unmodified. Author attribution in that commit is kept as Bram Matthys.

### A.2 Security fixes present [FACT]

The official news post of 2026-11-07 says 6.2.7.1 fixes "9 security issues, including one CRITICAL (CVE-2026-114713)", which "allows normal users to execute commands with server/IRCOp authority". Fixes mapped to code:

| Fix | File | Change |
|---|---|---|
| **CVE-2026-114713** (critical) | `src/modules/labeled-response.c` | `labeled_response_mtag_is_ok()` rejects `label` values containing space, `;`, `\`, CR or LF |
| Decloak (reveal victim IP) | `src/modules/svsmode.c` | `unban_user()` no longer matches real host/IP of hidden users; extban match-and-remove oracle removed |
| Another IP leak | `src/modules/extended-monitor.c` | notifications only for `IsUser()` clients |
| Blacklist recheck hang | `src/modules/blacklist.c` | `list_for_each_entry_safe` (list mutated during iteration) |
| CHGIDENT crash | `src/modules/user.c`, `whowasdb.c` | handshake not "finished" while `user->server` is NULL; whowasdb requires `server` |
| "Server is full" counting | `src/modules/nick.c` | class client count decremented when the class changes |
| Services ban limit | `src/modules/mode.c`, `sjoin.c` | remote list-mode adds capped at `2 × MAXBANS` |
| TLS renegotiation flood | `src/modules/starttls.c` | `SSL_OP_NO_RENEGOTIATION` on STARTTLS sessions |

**Root cause of CVE-2026-114713 (from code reading):** `_parse_message_tags()` (`src/modules/message-tags.c`) *unescapes* tag values, so `\n` becomes a real LF. `labeled-response.c` then wrote the label straight into protocol lines with `snprintf("@label=%s …")` (lines 162, 169, 218), skipping `message_tag_escape()`. A label could therefore carry CRLF plus a forged command into the client stream and the S2S stream. The fix is input validation at the single entry point. It is correct, but it fixes this one instance and not the bug class (see C.1).

The docs' description of the CVE ("IRCv3 `label` tag server-to-server command injection") matches the code. Upstream's public wording only says "commands with server/IRCOp authority".

### A.3 Build [FACT]

- Host: Ubuntu, GCC 15.2.0, OpenSSL 3.5.5. Bundled argon2, c-ares, libsodium and jansson; system pcre2 runtime.
- Command: upstream CI script `extras/build-tests/nix/build`, which uses `-Werror`, ASAN+UBSAN, FORTIFY_SOURCE=3, full RELRO, CET, stack-clash protection, and `-DTESTSUITE`.
- Result: **success**. UnrealIRCd's own code produced zero warnings under `-Werror`. All 113 "warning" lines in the log come from the bundled third-party library builds (libtool/make notices, c-ares).
- Installed to `~/unrealircd`, which the test suite expects. No system packages were installed.

### A.4 Test suite

- Suite: https://github.com/unrealircd/unrealircd-tests, branch `unreal60`, commit `8043c40` (2026-09-14). It is Python and needs `asyncore`/`asynchat`, which were removed in Python 3.12, so it does not run on the host's Python 3.14.
- Isolation: the runner `pkill -9`s every process owned by the user whose name matches `ircd|services|anope|val…`. So it runs in a throwaway Docker container: `python:3.11-bookworm`, unprivileged user, Anope 2.0.7 and Atheme 7.2.10-r2 (the versions pinned by the suite).
- Toolchain note: Debian bookworm (GCC 12) cannot be used on this host. Its ASAN runtime hangs in an infinite `DEADLYSIGNAL` loop on Linux 7.0 (5 of 20 trivial ASAN runs hung; 0 of 20 on trixie/GCC 14.2). The harness uses `python:3.11-trixie`, links `/usr/bin/python3` to 3.11 (the test scripts hard-code that path), and caps every step with a timeout and a 2 GB file-size limit.
- Result (2026-11-08) **[FACT]**:

| Pass | Executed | Passed | Failed |
|---|---|---|---|
| Anope 2.0.7 | 187 | 186 | 1 |
| Atheme 7.2.10-r2 | 187 | 186 | 1 |
| DB write / read (plain) | 3 / 3 | 3 / 3 | 0 |
| DB write / read (encrypted) | 4 / 4 | 4 / 4 | 0 |

  No ASAN, UBSAN or LeakSanitizer reports in any pass. No tests were skipped or disabled.
- **The one failure, `tests/extensions/statusmsg`, is a known upstream bug present in 6.2.7 and 6.2.7.1. It is not a regression.**
  - The bug: `channel_member_modes_generate_equal_or_greater()` (`src/api-channelmode.c`) compares a mode letter where it should compare a rank. So `PRIVMSG +#chan` skips voiced users, and `~#chan`/`&#chan` messages also reach halfops and above. That second part is a minor confidentiality issue.
  - Upstream fixed it in `f9a64a887` (2026-09-14, bug #6491), after the 6.2.7 release; the 6.2.7.1 patchset did not include it. The test was added to `unrealircd-tests` the same day, on the `unreal60` branch, which tracks `unreal60_dev`.
  - **[IMPLEMENTED, approved 2026-11-08]** Cherry-picked as `b13e01456` (`git cherry-pick -x`, upstream authorship kept).

**Rerun on `b13e01456` [TESTED, 2026-11-08]:** unmodified upstream `extras/build-tests/nix/run-tests.bbwrapper`, exit code 0.

| Pass | Executed | Passed | Failed |
|---|---|---|---|
| Anope 2.0.7 | 187 | 187 | 0 |
| Atheme 7.2.10-r2 | 187 | 187 | 0 |
| DB write / read (plain) | 3 / 3 | 3 / 3 | 0 |
| DB write / read (encrypted) | 4 / 4 | 4 / 4 | 0 |

No sanitizer reports. The TLS test suite (`extras/tests/tls/tls-tests`, which needs testssl.sh) has **not** been run yet.

**ZenIRCd tested baseline: `b13e01456`** = UnrealIRCd 6.2.7.1 + upstream `f9a64a887`.

---

## Part B — Architecture Map (Phase 1.1)

Size: about 154k lines of C in `src/`. The core (`src/*.c`) provides the event loop, sockets, parser, config, TLS and the module API. Nearly all commands and features are modules in `src/modules/` (about 190 top-level modules plus `chanmodes/` (34), `usermodes/` (12), `extbans/` (18) and `rpc/` (14)).

| Subsystem | Primary location | Notes |
|---|---|---|
| Event loop / fd I/O | `src/dispatch.c`, `fdlist.c`, `socket.c` | epoll/kqueue/poll backends, single-threaded |
| Packet parse & dispatch | `src/parse.c` (`parse()` → `parse2()`), `api-command.c` | fake-lag accounting (`parse_addlag`), handshake flood ban |
| Message tags | `src/modules/message-tags.c`, `api-messagetag.c` | per-tag `is_ok` handlers; **unknown tags from servers are accepted** (`message_tag_ok`) |
| Send path | `src/send.c` | `sendto_one`, `sendbufto_one` (pre-formatted fast path) |
| Users / channels | `src/user.c`, `channel.c`, `hash.c` (SipHash) | membership in `channel.c`; modes in `src/modules/mode.c` |
| S2S linking | `src/modules/server.c`, `protoctl.c`, `sjoin.c`, `eos.c`, `netinfo.c` | PROTOCTL EAUTH/SID/MTAGS; SJOIN burst; `link-security.c` |
| Modules | `src/modules.c`, `modulemanager.c` | `MOD_TEST/INIT/LOAD/UNLOAD`; ABI tag `"unrealircd-6"` checked at `modules.c:153`; third-party module manager |
| Config / rehash | `src/conf.c` (12k LOC), `conf_preprocessor.c` | two-pass test-then-run; remote includes over https |
| TLS | `src/tls.c`, `openssl_hostname_validation.c`, `modules/tls_antidos.c`, `starttls.c`, `sts.c` | spkifp/certfp link auth |
| Auth / passwords | `src/auth.c`, `crypt_blowfish.c`; `modules/sasl.c`, `certfp.c` | argon2id recommended in example config |
| Opers | `modules/oper.c`, `src/operclass.c` | operclass permission trees |
| Security groups | `src/securitygroup.c` | basis for webirc/proxy masks, policies |
| Anti-flood / anti-abuse | `modules/chanmodes/floodprot.c`, `connthrottle.c`, `antirandom.c`, `antimixedutf8.c`, `blacklist.c`, `central-blocklist.c`, `reputation.c`, `targetfloodprot.c`, `tkl.c` | |
| Cloaking | `modules/cloak_sha256.c`, `cloak_md5.c`, `cloak_none.c` | cloak keys **must match network-wide** |
| WebIRC / proxies | `modules/webirc.c`, `webserver.c`, `websocket.c`, `src/conf.c` (proxy block) | types: WEBIRC, WEBIRC_PASS (legacy), FORWARDED, X-FORWARDED, CLOUDFLARE |
| IRCv3 | per-cap modules (see B.1) | |
| JSON-RPC | `src/api-rpc.c`, `modules/rpc/*` | off unless `rpc-user` blocks are configured |
| DNS | `src/dns.c` (c-ares) | async |
| Logging | `src/log.c` | JSON logging, webhook sink |
| Databases | `src/unrealdb.c`, `tkldb.c`, `channeldb.c`, `whowasdb.c`, history | file header magic `UnrealIRCd-DB-v1` / `-Crypted-v1` |
| Build / CI | `configure.ac`, `Config`, `extras/build-tests/nix/*`, `extras/tests/tls` | external test repo `unrealircd-tests` |

### B.1 IRCv3 status [FACT: capability names found in source]

Capabilities: `account-notify`, `account-tag`, `away-notify`, `batch`, `cap-notify`, `chghost`, `echo-message`, `extended-join`, `extended-monitor`, `invite-notify`, `labeled-response`, `message-tags`, `multi-prefix`, `sasl`, `server-time`, `setname`, `standard-replies`, `sts`, `tls`, `userhost-in-names`, `draft/chathistory`, `draft/multiline`, `draft/extended-isupport`, `draft/no-implicit-names`, plus vendor `unrealircd.org/*`.
Tags: `account`, `batch`, `bot`, `label`, `msgid`, `time`, `+typing`/`+draft/typing`, `+reply`/`+draft/reply`, `+draft/channel-context`, `draft/multiline-concat`.

[ASSUMPTION] Not implemented, because a grep found nothing: `draft/read-marker`, `draft/message-redaction`, `draft/account-registration`, `draft/metadata-2`, `draft/pre-away`. These are Phase 4 candidates only, and each needs research first.

### B.2 WebIRC trust model [FACT, `src/modules/webirc.c`]

- The `WEBIRC` command is accepted only before registration (`CMD_UNREGISTERED`).
- A proxy block must match the **connecting socket's IP** through a security-group mask, and the password must pass `Auth_Check()` (hashes and certfp supported). Otherwise the client is disconnected.
- A second `WEBIRC` on the same connection disconnects the client.
- The gateway-supplied **hostname is trusted as-is** when `valid_host()` passes. No forward-confirmed DNS check is done by the IRCd.
- The `secure` option is honoured only if the gateway→IRCd link itself is TLS. Otherwise the user loses `+z`.
- The legacy `PROXY_WEBIRC_PASS` type (`PASS WEBIRC_<ip>_<host>`) does **no password check**. Only the source-IP mask protects it.

---

## Part C — Security Posture (Phase 1.5)

### C.1 Findings
1. **[FACT] Bug class behind CVE-2026-114713 is still open in principle.** Tag values are stored unescaped, and any code that formats them raw into a line can reintroduce injection. Only `labeled-response.c` does this today (`grep '"@[a-z]+=%s'`), and its batch-id uses are internally generated. **[PROPOSAL] P1:** (a) route all tag output through `mtags_to_string()`; (b) add a last-line guard in the send path that drops any outgoing line with an embedded CR, LF or NUL before the terminator, and logs it without the payload; (c) add regression tests.
2. **[FACT] Unknown message tags from servers are accepted unchecked** (`message_tag_ok`). That is intended for S2S, but it means any linked server or services can inject arbitrary tags. **[PROPOSAL] P2:** length and charset sanity checks on S2S tag names.
3. **[FACT] WebIRC hostname is trusted from the gateway.** For ZEnet, **[PROPOSAL] P1 (config, not code):** the middleware sends the IP as hostname, or ZenIRCd gets an option to ignore the gateway hostname and resolve it itself.
4. **[FACT] Legacy `webirc { type old; }` (PASS-based) has no secret.** **[PROPOSAL] P1:** ZEnet config must never use it. Consider a config warning in ZenIRCd (Phase 3).
5. **[FACT] Defaults are sane.** `plaintext-policy`: user allow / oper deny / server deny. `outdated-tls-policy`: user warn / oper and server deny (`src/conf.c:1933`). JSON-RPC is off by default. Example oper uses argon2id. Upstream 6.2.8-git will warn on password-only outgoing links, becoming an error in Q1 2027. **[PROPOSAL]:** ZenIRCd should adopt that warning early.
6. **[FACT] Hardened build flags are already in place** (A.3). Keep them; do not weaken them.
7. **[ASSUMPTION] Not yet reviewed in depth:** SASL state machine, JSON-RPC ACL enforcement, `conf.c` remote-include fetching, websocket frame parser, `unrealdb` crypto. These are the Phase 3 security review queue.

### C.2 Insecure defaults that must never be introduced
Plaintext oper or server links; password-only link blocks without spkifp/certfp; JSON-RPC listening on non-local interfaces by default; WebIRC blocks without a source mask; trusting `X-Forwarded-For`/`Forwarded` from non-proxy IPs; logging of PASS/OPER/AUTHENTICATE/WEBIRC parameters or message bodies; running as root.

---

## Part D — Competing IRCds (Phase 1.2/1.3)

Repo metadata checked through the GitHub API on 2026-11-08 **[FACT]**. Characterisations are from prior knowledge of these projects **[ASSUMPTION — check against source before any adoption]**.

| Project | Lang | Licence (GitHub) | Latest | Active | Relevant strengths for ZenIRCd |
|---|---|---|---|---|---|
| UnrealIRCd | C | GPLv2(+) | 6.2.7.1 | yes | baseline |
| Solanum | C | GPL-2.0 | no GH releases | yes (2026-11-07) | large-network S2S (TS6), ban propagation, oper accountability (Libera.Chat) |
| InspIRCd | C++ | GPLv2 (LICENSE.txt) | v4.12.1 (2026-09-26) | yes | module lifecycle/API, rich config validation |
| Bahamut | C | NOASSERTION (check headers) | v2.2.4 (2024-10-04) | yes | long-running stability on DALnet, conservative linking |
| ircd-hybrid | C | GPL-2.0 | 8.2.47 | yes (2026-11-08) | lean memory/CPU, efficient connection handling |
| ngIRCd | C | GPL-2.0 | rel-28 (2026-07-12) | yes | portability, `--configtest` diagnostics, simplicity |
| Ergo | Go | MIT | v2.19.1 (2026-08-05) | yes | **feature/UX reference only**: IRCv3 breadth, history, account UX |
| Charybdis | C | GPL-2.0 | 3.5.5 (2017) | **archived** | historical only |

### D.1 Candidate classification [PROPOSAL]

| Area | Candidate | Category |
|---|---|---|
| Protocol output | central CR/LF/NUL guard in send path (C.1-1) | Improve existing (core, small) |
| Message tags | S2S tag-name sanity checks (C.1-2) | Improve existing |
| WebIRC | option to ignore gateway hostname; warn on `type old` | Improve existing (module) |
| Linking | adopt 6.2.8-git warning for password-only `link::outgoing` | Keep upstream behaviour, take early |
| Config | ngIRCd/InspIRCd-style richer `configtest` diagnostics | Adapt idea |
| Observability | Prometheus-style metrics through JSON-RPC (Ergo/InspIRCd idea) | New module (Phase 4) |
| IRCv3 | `draft/read-marker`, `draft/message-redaction`, `draft/metadata-2` | New module (Phase 4, spec-stability check first) |
| S2S protocol | switch to TS6 or another protocol | **Reject**: breaks ZEnet linking and services |
| Architecture | multithreading, or rewriting the event loop in another language | **Reject**: no evidence of need, high risk |
| Ergo-style integrated services | — | **Reject**: ZEnet uses Anope/Atheme |

---

## Part E — Licensing & Attribution (Phase 1.4)

**[FACT]**
- `LICENSE` is GPL version 2. About 274 files carry the classic ircd notice "version 1, or (at your option) any later version". 54 files say "GPLv2 or later" and 1 says "GPLv2 or newer". **2 files say only "License: GPLv2"** (`src/modules/central-api.c`, `central-blocklist.c`) and **1 says "GPLv1"**. Upstream `README.md` states "GPLv2 or later".
- Non-GPL pieces inside the tree: `crypt_blowfish.c` (Solar Designer, public domain / permissive fallback), ChaCha in `random.c` (OpenBSD, public domain), SipHash in `hash.c` (Aumasson/Bernstein, CC0/MIT-style). Bundled tarballs in `extras/`: argon2 (CC0/Apache-2.0), c-ares (MIT), libsodium (ISC), jansson (MIT), pcre2 (BSD), geoip-classic.
- Historical copyright holders (Oikarinen, Reed, Gruner, …) appear in the file headers.

**Implications**
- [PROPOSAL] ZenIRCd's README should say **"GPLv2 (with many files GPLv2-or-later / GPLv1-or-later)"** instead of a blanket "GPLv2-or-later", unless upstream confirms the two "GPLv2" files are or-later. Treating the combined work as **GPL-2.0** is the safe reading.
- Code from Solanum, hybrid, ngIRCd or InspIRCd: GPLv2 is compatible with a GPLv2 combined work. GPL-2.0-only code would remove the "or later" option for the whole work. Ergo (MIT) is compatible but is Go, so only ideas can come over. Bahamut licence is NOASSERTION, so check its headers before even studying code closely.
- **Policy:**
  1. Re-implement ideas independently by default.
  2. Copying any code needs a licence check, retained copyright headers, a `docs/` attribution entry and a commit message citing the source.
  3. Never remove or alter existing copyright headers. ZenIRCd additions go as *additional* lines.

---

## Part F — ZEnet Compatibility Risks (Phase 1.6)

**[FACT] Identity strings that are protocol or ABI and must NOT change in Phase 2 without a migration plan:**

| String | Where | Breaks if renamed |
|---|---|---|
| `UnrealIRCd-<buildid>` in `PROTOCTL EAUTH` | `src/modules/server.c:1115` | peers use it for the bad-ulines safety check (`server.c:1587`). Renaming disables that protection between ZenIRCd and UnrealIRCd servers; services/stats tooling may also parse it |
| `"unrealircd-6"` module ABI tag | `src/modules.c:153`, every `MOD_HEADER` | every third-party module stops loading |
| `UnrealIRCd-DB-v1` / `-Crypted-v1` headers | `src/unrealdb.c` | existing tkldb, channeldb and history databases become unreadable |
| `UnrealProtocol` number, PROTOCTL tokens | `server.c`, `protoctl.c` | linking and Anope/Atheme `unreal4` protocol modules |
| Cloak algorithm and keys | `cloak_sha256.c` | ban masks across the network stop matching |

**[PROPOSAL] Phase 2 rule:** change only user-visible banner strings (`VERSION` reply text, startup banner, `/INFO`, `BASE_VERSION` display) and keep every string above. For EAUTH, the safe form is something like `UnrealIRCd-6.2.7.1-ZenIRCd-0.1`, so prefix matching keeps working. This must be tested against Anope and Atheme.

**Other risks:** WebIRC behaviour (B.2) and the `+z` loss for a plaintext localhost middleware link. Mode and extban letters, operclass names and config syntax must stay identical so existing ZEnet configs keep loading.

**[MISSING INFORMATION] — needed from ZEnet before Phase 2/3:**
- the UnrealIRCd versions running on each ZEnet server
- the Anope or Atheme version and protocol module
- sanitized `unrealircd.conf` (especially `proxy`/`webirc`, `link`, `cloak-keys` policy and loaded modules)
- the Node.js middleware's WEBIRC line format, whether it connects over TLS, and the source IPs it uses
- the Nginx WebSocket setup (whether the IRCd's own websocket module is also in use)
- any third-party modules in use

---

## Part G — Prioritized Roadmap [PROPOSAL]

| P | Item | Phase | Size |
|---|---|---|---|
| P0 | Accept this audit; get the ZEnet information listed in Part F | 1 | — |
| P0 | Add the Docker test harness to `extras/` so tests are reproducible | 1→2 | small |
| P1 | Phase 2 identity: banner strings only, keep all protocol/ABI/DB strings (Part F); versioning scheme e.g. `ZenIRCd 0.1 (based on UnrealIRCd 6.2.7.1)` | 2 | small |
| P1 | CRLF/NUL send-path guard + regression tests for tag injection (C.1-1) | 3 | small |
| P1 | WebIRC: config warning for `type old`; option to ignore gateway hostname (C.1-3/4) | 3 | small |
| P1 | Upstream-tracking process: watch the unrealircd.org advisories and `unreal60_dev`, cherry-pick security fixes within 24h | ongoing | process |
| P2 | Security review queue (C.1-7) | 3 | medium |
| P2 | S2S tag-name sanity checks (C.1-2) | 3 | small |
| P2 | Licence wording clarification (Part E) | 2 | trivial |
| P3 | configtest diagnostics, metrics module, extra IRCv3 drafts | 4 | medium |

---

## Part H — Phase 2 Identity Changes (ZenIRCd 0.1.0) [IMPLEMENTED 2026-11-08]

Requested by the maintainer before formal acceptance of this audit. Scope chosen by
the maintainer: branding + binary/paths/config-file names + docs, with
compatibility for the old names.

### H.1 Changed

| Area | Change | Files |
|---|---|---|
| Version | `BASE_VERSION` "ZenIRCd"; `ZENIRCD_VERSION` from `AC_INIT` (0.1.0); `IRCDTOTALVERSION` = `ZenIRCd-0.1.0` now defined once; `UPSTREAMVERSION` = `UnrealIRCd-6.2.7.1` at the time; `UnrealIRCd-6.2.8-git` since the 2026-11-08 sync to upstream `89acd7d5e` | `include/common.h`, `include/version.h`, `configure.ac`, `include/windows/setup.h` |
| VERSION / 002 / 004 / SINFO software | `ZenIRCd-0.1.0` | (derived from `version`) |
| Banner, /INFO, /CREDITS header | ZenIRCd, with "based on UnrealIRCd 6.2.7.1" and credit to the UnrealIRCd developers | `src/ircd.c`, `src/serv.c`, `src/version.c.SH` |
| Startup logo | new ZenIRCd logo (see H.3) | `src/version.c.SH`, `src/hash.c` |
| Self-referencing messages | strings where the daemon refers to itself now say "ZenIRCd" (69 changed lines in `src/`, excluding the crash reporter and logo) | `src/*.c`, `src/modules/*.c` |
| Binaries / script | `bin/zenircd`, `bin/zenircdctl`, `./zenircd` (from `zenircd.in`); `bin/unrealircd`, `bin/unrealircdctl`, `./unrealircd` installed as symlinks | `Makefile.in`, `configure.ac`, `zenircd.in` |
| Paths | default install dir `~/zenircd-server`; `data/zenircd.pid`, `data/zenircd.ctl` | `Config`, `configure.ac` |
| Config file | `conf/zenircd.conf`; if absent, `conf/unrealircd.conf` is used with a notice | `include/config.h`, `src/ircd.c`, `zenircd.in`, `Config` |
| Command hints | `./unrealircd X` → `./zenircd X` in messages and example configs | `src/*.c`, `doc/conf/**` |
| systemd / AppArmor examples | `extras/startup/zenircd*.service`, `extras/security/apparmor/zenircd` (AppArmor matches the real binary path, so the profile names `bin/zenircd`) | `extras/` |
| CI / tests | paths and process names; test-expectation patch `0001-zenircd-identity.patch` (VERSION prefix, binary and pid names only) | `extras/build-tests/**` |

### H.2 Deliberately NOT changed (compatibility, attribution)

- `PROTOCTL EAUTH` software string `UnrealIRCd-<buildid>` (peer bad-ulines check, `server.c:1587`)
- module ABI tag `"unrealircd-6"`, `Mod_Version` scheme, the `VERSIONONLY` number (config `@if`, module manager)
- `UnrealIRCd-DB-*` file headers; log event IDs (`UNREALIRCD_START`, …); `unrealircd.org/*` tags and caps; JSON-RPC wording
- User-Agent strings for unrealircd.org infrastructure (module repository); the module manager still uses the UnrealIRCd module repository, because the ABI is the same
- 251 `"UnrealIRCd Team"` module-author credits, LICENSE/CREDITS text, every copyright header
- historical references ("upgrading from UnrealIRCd 4", "UnrealIRCd 3.2.x protocol")
- internal source file names (`unrealircdctl.c`, `unrealircd.h`), to keep upstream merges easy; ASAN log name `unrealircd_asan` (used by the crash reporter)
- Windows-specific code (untested; only the version define was added)

### H.3 Upstream anti-rebrand check

UnrealIRCd stores the string `UnrealIRCd` after the startup logo's NUL byte
(`src/version.c.SH`). `init_hash()` compares it with `BASE_VERSION`. On mismatch it sets
`loop.tainted`, and then `add_to_client_hash_table()` (`src/hash.c`) silently stops
adding clients to the nick hash, which breaks the server. The code comment says it is
there to stop people taking the code "without even giving us credit".

ZenIRCd keeps the check and satisfies it openly: the new logo carries the marker
`ZenIRCd`, and the offset is computed rather than hard-coded. Credit to UnrealIRCd stays
prominent in the banner, /INFO, /CREDITS and README.

**[TESTED 2026-11-08]** `extras/build-tests/docker/run` on the renamed tree: Anope 187/187,
Atheme 187/187, DB 14/14, no sanitizer reports. Servers were started as `bin/zenircd` and logged
"ZenIRCd started". If the check had fired, nick registration would have failed across the suite.

### H.4 Outbound features disabled (they would contact UnrealIRCd infrastructure)

- `./zenircd upgrade`, `hot-patch` and `cold-patch` now refuse with an explanation. They would
  download UnrealIRCd releases or patches from unrealircd.org and replace or patch the
  modified ZenIRCd code. The upgrade script is no longer installed, and an old copy is removed.
  Upstream security fixes reach ZenIRCd through this repository (TODO: upstream-tracking process, Part G).
- The crash reporter no longer offers to upload reports to `crash.unrealircd.org`.
  Reports are written locally, and the admin is told to review them for private data
  and report on GitHub (or privately, if security relevant).

### H.5 Migration notes for existing UnrealIRCd installs

- Scripts, cron jobs and systemd units that read `data/unrealircd.pid` must use `data/zenircd.pid`.
- `./unrealircd …` and `bin/unrealircd` keep working (symlinks). New installs default to `~/zenircd-server`.
- An existing `conf/unrealircd.conf` keeps working. Renaming it to `zenircd.conf` is recommended.
- Linked UnrealIRCd servers and services see `ZenIRCd-0.1.0` in VERSION/SINFO. Nothing in the
  link protocol changes. **[TO VERIFY in Phase 5]** against ZEnet's actual services and
  monitoring scripts, in case any of them parse the version string.
