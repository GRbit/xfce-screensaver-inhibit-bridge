# xfce-screensaver-inhibit-bridge

A small user daemon for XFCE that makes `xfce4-power-manager` respect
"I am playing video" requests that applications send to
`org.freedesktop.ScreenSaver`.

## The problem

On XFCE, some media players keep playing while the desktop behaves as if
nobody is there: the backlight dims after a few minutes, and on battery the
laptop may go to sleep in the middle of a film.

Jellyfin Media Player (now Jellyfin Desktop) is the app this was written for,
but anything that only uses the `org.freedesktop.ScreenSaver` API is affected.

The player is not at fault. It does ask the desktop to stay awake. The
request just never reaches the component that dims the screen.

## Why this happens on XFCE

There are two separate D-Bus APIs for "do not go idle", and on XFCE they are
served by two unrelated processes:

| API | Meaning | Who serves it on XFCE |
|---|---|---|
| `org.freedesktop.ScreenSaver.Inhibit` | do not start the screensaver or lock the screen | the screen locker (`light-locker`) |
| `org.freedesktop.PowerManagement.Inhibit` | do not save power | `xfce4-power-manager` |

What each of them does with a request:

- `light-locker` owns the `org.freedesktop.ScreenSaver` name. On `Inhibit` it
  calls `XScreenSaverSuspend`, which pauses the X server's own screensaver
  timer, and that is all.
- `xfce4-power-manager` is the one that dims the backlight, drives DPMS and
  suspends the machine on inactivity. It skips all three while it holds an
  inhibit, but it only accepts inhibits through its own
  `org.freedesktop.PowerManagement.Inhibit` interface (or presentation mode).

So a player that calls only `ScreenSaver.Inhibit` gets a successful reply
from `light-locker`, and `xfce4-power-manager` never hears about it.

The integration exists in one direction only: `xfce4-power-manager` forwards
the inhibits it holds to `org.freedesktop.ScreenSaver`, but nothing forwards
them the other way.

Desktops where one component handles both sides do not have this gap. In KDE
Plasma, for example, the screen locker passes `ScreenSaver.Inhibit` on to the
power management daemon. Some applications work around the split themselves
by calling both APIs.

## What the daemon does

It watches the session bus for `Inhibit` and `UnInhibit` calls addressed to
`org.freedesktop.ScreenSaver` and holds one inhibit at `xfce4-power-manager`
for as long as at least one application holds a ScreenSaver inhibit.

```
  player --Inhibit--> org.freedesktop.ScreenSaver (light-locker)
                |
                | observed by the bridge (bus monitor)
                v
        screensaver-inhibit-bridge
                |
                +--Inhibit--> org.freedesktop.PowerManagement (xfce4-power-manager)
```

It cannot simply own the `org.freedesktop.ScreenSaver` name, because the
screen locker already does. Instead it connects as a bus monitor and tracks:

- each `Inhibit` call and the cookie returned for it,
- `UnInhibit` calls from the same sender,
- senders that leave the bus without releasing (crashed or killed apps),
- restarts of the screen locker and of `xfce4-power-manager`.

The upstream side has two states, `IDLE` and `HOLDING`. The bridge acquires
when the first application inhibits and releases when the last one is gone.
Calls made by `xfce4-power-manager` itself are ignored, otherwise its own
forwarding would create a feedback loop.

### Limitations

- Inhibits taken before the bridge started are invisible to it, because the
  screen locker has no way to list them. They are picked up the next time the
  application inhibits, for example after pause and resume.
- It only helps applications that call `org.freedesktop.ScreenSaver`.
  Applications that do not ask at all stay unprotected.
- There is no watchdog. systemd restarts the daemon if it crashes, not if it
  hangs.

## Requirements

- XFCE with `xfce4-power-manager` and a screen locker that owns
  `org.freedesktop.ScreenSaver`
- Python 3 with PyGObject (`python3-gi` on Debian and Ubuntu)
- a systemd user session, with the desktop session using the user bus at
  `/run/user/<uid>/bus` (the default on current Debian)

## Install

```sh
install -Dm755 screensaver-inhibit-bridge ~/.local/bin/screensaver-inhibit-bridge
install -Dm644 screensaver-inhibit-bridge.service ~/.config/systemd/user/screensaver-inhibit-bridge.service
systemctl --user daemon-reload
systemctl --user enable --now screensaver-inhibit-bridge
```

The service starts with your user session and is restarted on failure.

## Check that it works

Start playback (pause and resume if it was already playing), then:

```sh
busctl --user call org.freedesktop.PowerManagement \
    /org/freedesktop/PowerManagement/Inhibit \
    org.freedesktop.PowerManagement.Inhibit GetInhibitors
```

The reply should list `screensaver-inhibit-bridge`. The journal shows every
acquire and release:

```sh
journalctl --user -u screensaver-inhibit-bridge
```

For a line per observed call, run the daemon with `--debug`. The built-in
unit tests of the state machine run with:

```sh
screensaver-inhibit-bridge --self-test
```

## Uninstall

```sh
systemctl --user disable --now screensaver-inhibit-bridge
rm ~/.local/bin/screensaver-inhibit-bridge
rm ~/.config/systemd/user/screensaver-inhibit-bridge.service
systemctl --user daemon-reload
```

## Tested with

- Debian 13, XFCE 4.20 on X11
- xfce4-power-manager 4.20.0, light-locker 1.8.0
- dbus-daemon as the session bus (dbus-broker is untested)
- Jellyfin Media Player 1.12.0

Other screen lockers, such as xfce4-screensaver, are untested.

## Upstream status

The development branch of `xfce4-power-manager` gained a setting that
inhibits power management while a window is fullscreen. It is not part of
4.20. Once it reaches a release, this bridge may become unnecessary for
fullscreen playback. Windowed playback would still need it.

## License

MIT, see [LICENSE](LICENSE).
