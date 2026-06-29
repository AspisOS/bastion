# bastion

The display manager and login greeter for **AspisOS**, a capability-based,
no-ambient-authority operating system built on the from-scratch
[Aegis](https://github.com/AspisOS/Aegis) kernel.

bastion is the first graphical thing a user sees: on a graphical boot it draws
the greeter, authenticates the user against the system credential store, and
launches their session into the [lumen](https://github.com/AspisOS/lumen)
compositor. It is a standalone component of the Lumen desktop, distributed as a
[herald](https://github.com/AspisOS/AspisOS) package.

## Role in the system

- Started by the `vigil` service manager **only on a graphical boot** (it ships
  its own `vigil` service definition, `mode=graphical`). On a text boot it is
  never started — and bastion itself exits immediately if the kernel cmdline
  does not resolve to graphical mode.
- Reads `/proc/cmdline` for a test-harness `bastion_autologin=USER` hook and an
  `/etc/aegis/autologin` file for passwordless session auto-start; otherwise it
  presents the interactive greeter.
- On successful authentication it elevates the session (binds uid/gid) and spawns
  the compositor for that user.

## Capabilities

bastion's cap policy (`pkg/etc/aegis/caps.d/bastion`) is:

```
service AUTH FB SETUID NET_SOCKET
```

- **AUTH** — verify user credentials.
- **SETUID** — drop into the authenticated user's identity for the session.
- **FB** — draw the greeter directly to the framebuffer before a compositor exists.
- **NET_SOCKET** — session/network bring-up needs.

Because its herald package id (`bastion`) is a distribution name and it installs
a `/bin` binary plus a cap policy and a vigil service, bastion is a `class=system`
package: first-party and signature-trusted, installed verbatim by herald.

## Building

bastion fetches a pinned [glyph](https://github.com/AspisOS/glyph) toolkit
artifact (the GUI libraries it links: glyph + libaudio + libauth) and builds
against it, then packs a signed herald package.

```sh
make MUSL_CC=/path/to/musl-gcc HERALD_KEY=/path/to/signing.key
```

- `GLYPH_VERSION` pins the toolkit release fetched by `tools/fetch-glyph.sh`.
- `MUSL_CC` is the musl cross-compiler (the only toolchain assumption — point it
  at an Aegis-native `cc` to build on-device in the future).
- `HERALD_KEY` signs the `.hpkg`.

Output: `bastion.hpkg` (a `class=system` herald package) + `bastion.hpkg.sig`.

## Package payload

```
/bin/bastion                         the display manager
/etc/aegis/caps.d/bastion            its capability policy
/etc/vigil/services/bastion/         the vigil service (mode=graphical)
```

## Repository layout

```
src/        bastion source
pkg/        install-tree skeleton shipped verbatim (caps.d + vigil service)
tools/      fetch-glyph.sh (toolkit fetch) + pack.sh (build the signed .hpkg)
Makefile    fetch toolkit -> build -> pack
VERSION         this component's version
GLYPH_VERSION   the pinned glyph toolkit version it builds against
```

## Dependencies

`depends=lumen` — bastion launches sessions into the compositor, so installing
it pulls [lumen](https://github.com/AspisOS/lumen) (which in turn provides the
desktop fonts).
