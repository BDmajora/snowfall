# Snowfall — YetiOS Login Manager

A minimal, DRM-native graphical login manager for YetiOS.

## Architecture

Snowfall is a framebuffer login manager that renders directly to
DRM/KMS (no X11, no Wayland compositor required). It authenticates
users via PAM, then execs the selected session as the authenticated
user. When no session files are installed, snowfall falls back to a
built-in default — on YetiOS, CrystallineLattice (`glacier`).

```
snowfall (DRM master)
  ├── drm.c          KMS/DRM framebuffer setup
  ├── input.c        libinput keyboard handling
  ├── renderer.c     Cairo-based UI drawing
  ├── auth.c         PAM authentication
  ├── session.c      Session discovery + launch
  ├── ui_state.c     UI state machine (field focus, navigation)
  └── main.c         Lifecycle: init → render loop → auth → exec
```

When snowfall starts, it grabs DRM master (which causes snowcone to
exit). After successful authentication it drops DRM master, switches
to the user's VT, and execs the compositor.

The compositor (`glacier`, sway, …) acquires the GPU and input devices
through **libseat**, which talks to the **seatd** daemon. snowfall does
not start a seat manager itself: `seatd` runs as its own OpenRC service
that snowfall `need`s, and snowfall sets `LIBSEAT_BACKEND=seatd` in the
session environment. See [Seat management](#seat-management).

## Dependencies

- libdrm
- libinput + libudev
- cairo
- libpam
- libxkbcommon (keymap handling)
- seatd (runtime — seat/GPU/input management for the launched compositor)

### Alpine / YetiOS

```sh
apk add libdrm-dev libinput-dev cairo-dev linux-pam-dev libxkbcommon-dev eudev-dev
```

## Build

```sh
make
sudo make install
```

## OpenRC

```sh
sudo rc-update add snowfall default
```

The service depends on `udev` (for libinput) and `seatd` (for the
compositor's seat), and starts after `snowcone` (the boot splash). When
snowfall grabs DRM master, snowcone detects the loss and exits.

## Seat management

The login manager renders directly to DRM/KMS, but the compositor it
hands off to (`glacier`, sway, …) needs a **seat manager** to be given
the GPU and input devices. snowfall uses **seatd** for this — not
`seatd-launch`, which is upstream's workaround for systems *without* a
running seatd.

Two things must be in place:

1. **The seatd service is enabled.** snowfall `need`s it, so installing
   the seatd package (which ships `/etc/init.d/seatd`) is enough for
   OpenRC to start it on demand. To run it independently:

   ```sh
   sudo rc-update add seatd boot
   ```

2. **Login users are in the `seat` group.** libseat connects to
   `/run/seatd.sock`, which is owned `root:seat`. snowfall applies the
   user's groups (`initgroups`) before launching the session, so simply:

   ```sh
   sudo addgroup <user> seat
   ```

snowfall sets `LIBSEAT_BACKEND=seatd` for the session so libseat targets
seatd directly instead of probing for a (nonexistent on YetiOS) logind.
A missing/stopped seatd shows up in `~/.snowfall-session.log` as:

```
[ERR ] libseat_open_seat: No such file or directory (is seatd running, user in 'seat'?)
```

## Session files

Snowfall reads `.desktop` files from `/usr/share/wayland-sessions/`.
Each file must have an `Exec=` line. Example:

```ini
[Desktop Entry]
Name=Sway
Exec=sway
Type=Application
```

If the directory is empty or absent, snowfall offers a built-in default
session that launches **CrystallineLattice** (`glacier`), YetiOS's
from-scratch DRM/KMS layer. Override it by dropping a `.desktop` file here,
or change `SF_DEFAULT_EXEC` in [`src/session.c`](src/session.c).

## License

MIT License - YetiOS Project
