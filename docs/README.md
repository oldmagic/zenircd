# ZenIRCd

**ZenIRCd** is a new, independently maintained IRC daemon developed primarily for the **ZEnet IRC Network**.

It is an evolutionary project based on **UnrealIRCd 6**. It is **not** a from-scratch rewrite and **not** a simple rename. The goal is a mature, secure, high-performance, feature-rich IRC daemon with its own identity, architecture decisions, development roadmap, and release cycle while retaining UnrealIRCd’s proven foundations.

## Project Priorities (in order)

1. Stability and reliability  
2. Security by design  
3. IRC protocol correctness and interoperability  
4. High performance and efficient resource utilization  
5. Modular architecture and extensibility  
6. Modern IRCv3 support (correctly implemented and tested)  
7. Excellent network administration capabilities  
8. Long-term maintainability  
9. Cross-platform compatibility  
10. Clear documentation and predictable configuration  

**Never sacrifice stability or protocol compatibility merely to introduce a new feature.**

## Current Baseline

- **Foundation:** UnrealIRCd 6 (6.2.x series)  
- **Required starting point (as of 2026-11-08):** UnrealIRCd **6.2.7.1** or later  
  - This release (and its hot-patch) addresses multiple security issues, including the critical **CVE-2026-114713** (IRCv3 `label` tag server-to-server command injection).  
- Official UnrealIRCd sources:  
  - https://github.com/unrealircd/unrealircd  
  - https://www.unrealircd.org/docs/  
- License foundation: **GPLv2-or-later** (same as UnrealIRCd)

## What This Project Is / Is Not

| Is | Is Not |
|----|--------|
| An independent project with its own identity and roadmap | A simple rebrand of UnrealIRCd |
| Built on mature UnrealIRCd 6 foundations | A from-scratch IRC server |
| Compatible with existing ZEnet infrastructure (services, WebIRC, linking, TLS) | A drop-in that silently breaks existing networks |
| Transparent about its technical origins | Claiming to be independently written from scratch |

## ZEnet Compatibility Requirements

ZenIRCd must be able to replace an UnrealIRCd server in the existing ZEnet network with minimal disruption.

Existing infrastructure includes:
- UnrealIRCd-based servers
- IRC services (Anope / Atheme style)
- TLS-enabled connections
- WebIRC clients + custom Node.js WebIRC middleware (JSON/WebSocket → WebIRC)
- Nginx reverse proxies
- Docker-based supporting services

**Critical WebIRC rules:**
- Browser clients talk to the middleware; the middleware talks to the IRCd via authenticated WebIRC.
- Never assume the browser connects directly to the IRCd.
- Never trust arbitrary client-supplied IP or hostname information.
- Maintain support for trusted proxy / WebIRC configurations.

## Development Approach

Work is done in strict phases. Large architectural changes are forbidden until benefits, risks, and migration strategy have been documented and accepted.

See **TODO.md** for the complete, prioritized task list.

## Attribution

ZenIRCd is based on the excellent work of the UnrealIRCd team and contributors.  
All applicable copyright notices must be preserved.  
This project does **not** claim to have been written from scratch.

## License

GPLv2-or-later (compatible with the UnrealIRCd foundation).  
Any code adapted from other projects must be checked for license compatibility, copyright ownership, and proper attribution before inclusion.

## Status

- **Phase 0** (secure baseline): complete. Baseline is UnrealIRCd 6.2.7.1 plus upstream fix `f9a64a887`.
- **Phase 1** (technical audit): draft in [AUDIT.md](AUDIT.md), awaiting acceptance.
- **Phase 2** (identity): ZenIRCd 0.1.0 naming applied at the maintainer's request (see AUDIT.md Part H).

See TODO.md for details.

---

*Last updated: 2026-11-08*