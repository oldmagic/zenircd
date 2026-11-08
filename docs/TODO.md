# ZenIRCd TODO

Complete prioritized task list derived from the verified master project instruction (2026-11-08).

Work strictly in phase order. Do **not** jump ahead to large features or architectural rewrites.

---

## Phase 0 — Secure Baseline (Immediate)

- [x] Obtain a clean, verified UnrealIRCd **6.2.7.1** (or later) source tree — GPG-verified tarball on top of upstream 6.2.7 (`7a3f9df5c`)
- [x] Record the exact version, tag, and Git commit hash — baseline `a3172c9b4` (upstream has no tags; see AUDIT.md A.1)
- [x] Verify the tree builds cleanly on the target platform(s) — Linux/GCC 15 and Debian trixie/GCC 14, `-Werror` + ASAN/UBSAN
- [x] Confirm the existing UnrealIRCd test suite passes — 388/388 on `b13e01456` (6.2.7.1 + upstream fix `f9a64a887`); Anope, Atheme and DB passes; TLS suite not yet run (AUDIT.md A.4)
- [x] Apply any available hot-patches if not already included in the tree — 6.2.7.1 tarball contains the full hot-patch fix set
- [x] Confirm the critical CVE-2026-114713 (and related 6.2.7.1 fixes) are present — mapped to code in AUDIT.md A.2
- [x] Document the exact baseline commit that ZenIRCd will start from — docs/AUDIT.md Part A

**Exit criteria:** A known-good, fully patched UnrealIRCd 6.2.7.1+ tree that builds and passes tests.

---

## Phase 1 — Foundation & Research (Technical Audit)

### 1.1 Repository & Architecture Inspection
- [ ] Map the overall directory structure and major subsystems
- [ ] Document the event-driven architecture
- [ ] Document client connection handling
- [ ] Document server-to-server linking and synchronization
- [ ] Document the IRC protocol parser and command handling
- [ ] Document user, channel, and membership management
- [ ] Document the module loading / unloading system
- [ ] Document the configuration parser and rehash mechanism
- [ ] Document TLS and certificate handling
- [ ] Document authentication systems
- [ ] Document operator permissions and commands
- [ ] Document anti-flood / anti-spam mechanisms
- [ ] Document hostname cloaking
- [ ] Document current IRCv3 implementation status
- [ ] Document logging and debugging facilities
- [ ] Document JSON-RPC functionality
- [ ] Document DNS and asynchronous networking
- [ ] Document the existing build and testing infrastructure

### 1.2 Research Competing IRCds
- [ ] UnrealIRCd (primary reference) — modular architecture, security, configuration, linking, TLS, auth, anti-flood, admin tools
- [ ] Solanum (https://github.com/solanum-ircd/solanum) — large-network architecture, S2S consistency, IRCv3, performance
- [ ] InspIRCd (https://github.com/inspircd/inspircd) — modular C++ design, module lifecycle & APIs, extensibility, configuration
- [ ] Bahamut (https://github.com/DALnet/bahamut) — network stability, linking, WebIRC, traditional oper functionality
- [ ] ircd-hybrid — lightweight architecture, efficient connection handling, resource use
- [ ] ngIRCd — portability, configuration validation, simplicity, reliability
- [ ] Ergo (https://github.com/ergochat/ergo) — **feature/UX reference only** (IRCv3, SASL, accounts, history, message tags, WebSocket). Do **not** import architecture assumptions (no S2S linking, integrated services, Go, MIT)
- [ ] Charybdis — historical reference only (archived; study only as predecessor of Solanum)
- [ ] IRCv3 specifications (https://ircv3.net/) and Modern IRC specification
- [ ] Relevant RFCs on datatracker.ietf.org

### 1.3 Feature & Architecture Comparison
- [ ] Produce a structured comparison table / notes for key areas (security, modules, linking, IRCv3, config, performance, admin tools)
- [ ] Categorize candidate improvements:
  - Keep unchanged
  - Improve existing UnrealIRCd implementation
  - Adapt idea from another IRCd
  - Implement as new module
  - Implement as core feature
  - Reject (complexity / security / maintainability risk)

### 1.4 Licensing & Attribution
- [ ] Confirm UnrealIRCd GPLv2-or-later status and any mixed historical notices
- [ ] Check license compatibility of any code that might later be considered from Solanum, InspIRCd, Bahamut, hybrid, ngIRCd, Ergo, etc.
- [ ] Document attribution requirements
- [ ] Establish a policy: prefer independent re-implementation of ideas; never copy without license + copyright + attribution review
- [ ] Plan how ZenIRCd will permanently retain UnrealIRCd (and other) copyright notices

### 1.5 Security Posture
- [ ] Review recent UnrealIRCd security advisories (especially 6.2.7 / 6.2.7.1 and CVE-2026-114713)
- [ ] Identify areas for potential hardening (TLS, SASL, password hashing, oper auth, throttling, anti-flood, DNSBL, cloaking, WebIRC trust, S2S auth, config validation, module loading, JSON-RPC ACLs, input validation, resource exhaustion)
- [ ] Note any insecure defaults that must never be introduced

### 1.6 ZEnet Compatibility Risks
- [ ] Document current ZEnet WebIRC architecture (Node.js middleware → WebIRC)
- [ ] Identify risks to existing clients, services, server links, channel/user modes, bans, oper workflows, TLS, and configuration
- [ ] Confirm UnrealIRCd `webirc` / `proxy` block behaviour that must be preserved
- [ ] List any behaviour that would require a migration strategy if changed

### 1.7 Deliverable
- [ ] Write a concise technical audit report + prioritized implementation roadmap
- [ ] Explicitly separate: verified facts / assumptions / proposed improvements

**Exit criteria:** Written audit + roadmap accepted. No large code changes yet.

---

## Phase 2 — Establish ZenIRCd Identity

Goal: smallest safe changes that give the project its own identity while preserving 100 % functional compatibility.

*Started 2026-11-08 at the maintainer's request, before formal acceptance of the Phase 1 audit (scope decided by the maintainer: branding + binary/paths + docs).*

- [x] Decide versioning scheme — `ZenIRCd-0.1.0`; upstream base recorded separately (`UPSTREAMVERSION`, e.g. UnrealIRCd 6.2.7.1)
- [x] Change daemon self-identification (version strings, `VERSION` reply, startup banners, logs, administrative output)
- [x] Update any internal version / package name constants that are safe to change — protocol/ABI/DB identifiers deliberately kept (AUDIT.md Part F)
- [x] Ensure clear permanent attribution to UnrealIRCd remains visible — banner, /INFO, /CREDITS, README
- [x] Update build system / packaging identity only where necessary and safe — `zenircd` binaries/script, `~/zenircd-server`, `zenircd.conf`; old names kept as symlinks/fallback
- [x] Confirm the tree still builds — CI build (ASAN+UBSAN, -Werror) in Docker
- [x] Confirm the existing test suite still passes — 388/388 via extras/build-tests/docker/run (with the identity test-expectation patch)
- [x] Document the exact identity changes made — AUDIT.md Part H, README.md
- [ ] Enable GitHub private vulnerability reporting (needed by SECURITY.md) — maintainer action

**Exit criteria:** A working tree that identifies as ZenIRCd, still fully compatible, builds and tests green.

---

## Phase 3 — Core Improvements (Security, Stability, Protocol, Reliability, Performance)

Only carefully selected, researched improvements. Prefer modules or small targeted changes over core rewrites.

### Security
- [ ] Review and harden TLS / certificate handling where justified
- [ ] Review SASL and authentication paths
- [ ] Review password hashing and oper authentication
- [ ] Review connection throttling and anti-flood systems
- [ ] Review WebIRC / proxy trust model (never trust client-supplied IPs)
- [ ] Review server-link authentication
- [ ] Review configuration validation and module loading security
- [ ] Review JSON-RPC access control
- [ ] Ensure no passwords, tokens, or private messages are logged
- [ ] Ensure daemon is never designed to run as root

### Stability & Protocol Correctness
- [ ] Fix or harden any identified protocol edge cases
- [ ] Ensure IRCv3 capabilities are only advertised when correctly implemented and tested
- [ ] Improve error handling / recovery where evidence shows benefit

### Network Reliability
- [ ] Review S2S linking, burst, netsplit, and recovery behaviour
- [ ] Compare with Solanum / Bahamut practices where relevant

### Performance
- [ ] Profile only after identifying real bottlenecks (large client counts, high message throughput, large channels, S2S sync, DNS, TLS, floods)
- [ ] Avoid premature optimization

### Module System
- [ ] Evaluate possible improvements to dependency handling, error reporting, API stability, lifecycle, and third-party module support
- [ ] Only implement changes that are clearly justified

**Exit criteria:** Stable, tested improvements with documented justification and no regressions.

---

## Phase 4 — Advanced Functionality

Evaluate after Phase 3 is stable. Potential areas (research first):

- [ ] Improved module management / runtime behaviour
- [ ] Additional correctly-implemented IRCv3 features
- [ ] Better operator tooling and network monitoring
- [ ] Enhanced WebIRC / proxy integration for ZEnet
- [ ] More comprehensive security controls
- [ ] Improved configuration validation and diagnostics
- [ ] Enhanced debugging / observability

For each candidate:
- [ ] Research existing implementations
- [ ] Compare with current UnrealIRCd behaviour
- [ ] Check license / attribution
- [ ] Prefer module or minimal core change
- [ ] Write tests before advertising new capabilities

---

## Phase 5 — ZEnet Production Readiness

- [ ] Test client connectivity (classic + modern IRCv3 clients)
- [ ] Test services compatibility (Anope / Atheme style)
- [ ] Test WebIRC middleware integration (authenticated, trusted proxy)
- [ ] Test server linking with existing ZEnet servers
- [ ] Test TLS configuration and certificate handling
- [ ] Test network recovery (netsplits, restarts, bursts)
- [ ] Perform security testing (malformed input, floods, auth failures, resource exhaustion)
- [ ] Perform basic performance / long-running stability checks
- [ ] Prepare a controlled migration strategy and rollback plan
- [ ] Document any remaining known differences from stock UnrealIRCd

**Exit criteria:** Production-ready release candidate suitable for controlled ZEnet deployment.

---

## Ongoing / Cross-Cutting Rules (Always Apply)

- [ ] Never force-push or rewrite Git history
- [ ] Never auto-commit or auto-push
- [ ] Prefer small, focused, reviewable changes
- [ ] Explain intent, affected components, and compatibility impact before implementing
- [ ] Build and run relevant tests after every change
- [ ] Never disable tests to make a build pass
- [ ] Never claim something works without having tested it
- [ ] Never introduce insecure defaults
- [ ] Never log secrets
- [ ] Preserve all existing copyright notices
- [ ] Do not create new Markdown documentation files unless explicitly requested (prefer updating existing docs when authorized)
- [ ] Do not modify deployment infrastructure or install system packages without approval

---

## Immediate Next Actions (Do These First)

1. Complete **Phase 0** (secure 6.2.7.1+ baseline).  
2. Perform the **Phase 1** technical audit and produce the roadmap.  
3. Only then begin the minimal **Phase 2** identity changes.

---

*This TODO is the working task list for ZenIRCd. Update checkboxes as work is completed.*