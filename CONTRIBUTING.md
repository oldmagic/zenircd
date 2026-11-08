# Contributing to ZenIRCd

Thanks for helping. ZenIRCd puts stability, security and protocol compatibility
before new features. Please read [docs/README.md](docs/README.md) and
[docs/TODO.md](docs/TODO.md) first.

## Ground rules

- Keep changes small, focused and reviewable. Explain the intent, the affected
  components and the compatibility impact in the pull request.
- Do not rewrite mature subsystems without a documented technical justification.
- Keep all existing copyright notices and attribution. Do not copy code from other
  projects without checking licence compatibility and adding proper attribution.
- Never introduce insecure defaults, and never log passwords, tokens or private messages.
- Do not rename protocol, ABI or file-format identifiers (`UnrealIRCd-` in
  `PROTOCTL EAUTH`, the `unrealircd-6` module ABI tag, `UnrealIRCd-DB` headers,
  `unrealircd.org/*` tags). ZenIRCd must stay link-compatible with UnrealIRCd 6.

## Testing

Run the full test suite before opening a pull request:

```sh
extras/build-tests/docker/run
```

Never disable or skip tests to make a change pass. If a change intentionally alters
behaviour that an upstream test checks, update the expectation in
`extras/build-tests/unrealircd-tests-patches/` and explain why in the patch header.

## Security issues

Report them privately, see [SECURITY.md](SECURITY.md).

## Upstream

Fixes that also apply to stock UnrealIRCd are best contributed upstream as well:
<https://www.unrealircd.org/docs/Contributing>.
