# Security

## Reporting a security problem

Please don't open a public issue or post about it anywhere public. Tell us
privately, in one of two ways:

- **[Report a vulnerability](https://github.com/klarinotte/klarinotte-server/security/advisories/new)**
  on this repository (the Security tab). Only you and the maintainers see it.
- The [contact page](https://www.klarinotte.com/contact) on klarinotte.com.

Include the version (Settings → General in the app shows it), what you did
and what happened, and the steps to reproduce it if you have them.

Klarinotte is made by one developer. We aim to reply within a week. Once a
fix is out, the release notes say what it fixed, and credit you unless you'd
rather not be named.

## What this covers

- The server image `ghcr.io/klarinotte/klarinotte-server` and the web app it
  serves.
- Problems in the Klarinotte apps or the hosted service at klarinotte.com are
  welcome through the same channels.

Not covered: the rest of your setup (the host, the reverse proxy, Tailscale or
Cloudflare themselves). Report those to their own projects; the
[guide](README.md) shows how Klarinotte uses them.

## Supported versions

Only the latest image gets security fixes. Upgrade with
`docker compose pull && docker compose up -d` (if you pinned a version in the
compose file, change it to the latest first); the guide's §8 says what an
upgrade does.

## How your notes are protected

Notes and attachments are encrypted on your devices, with a separate key per
notebook, so the server stores them only as ciphertext. It does see metadata,
such as account emails, timestamps, sizes and who shares with whom.
[Security & Privacy](https://www.klarinotte.com/docs/security-privacy.html)
explains what is encrypted, what the server can see, how the keys are
derived, and what this does not protect against.
