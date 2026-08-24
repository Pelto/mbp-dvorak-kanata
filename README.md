# mbp-dvorak-kanata

[kanata](https://github.com/jtroo/kanata) config that ports the niche behaviours of
[`q11-dvorak-firmware`](../q11-dvorak-firmware) onto the MacBook Pro's built-in
keyboard, for when the Q11 isn't on the desk.

## What this is *not*

It does not remap letters. The macOS input source stays **Svorak A.5**, and the
`SvorakA5.bundle` keylayout is left completely untouched. A `.keylayout` is a pure
`(keycode, modifiers, dead-key state) -> string` lookup — it can't express
tap-hold, layers, or timing, which is the entire reason kanata is here.

Notably, **Svorak A5's Option layer already is the QMK `MAC_SPECIAL` layer** —
24 of 26 symbol positions are byte-identical, and the two that differ
(`^` and `~`) are dead keys on both sides. So the special-character layer needs
no configuration at all.

## Layout

| Key | Behaviour | Q11 equivalent |
|---|---|---|
| **Left Option** | untouched — real Option, plus Svorak A5 special chars | `LOpt = KC_LOPT` |
| **Right Option** | hold → `fn` layer | `ROpt = MO(MAC_FN)` |
| **Caps Lock** | Enter | `KC_ENT` on caps position |
| **Space** | tap → space, hold → `mod` layer | `LT(MAC_MOD_L/R, KC_SPC)` |

### `fn` layer (hold right Option)

```
        Q      W      E              U      I      O
        Ins    Home   PgUp           Bspc   Up     Del

        A      S      D      F       J      K      L
        Del    End    PgDn   SelAll  Left   Down   Right

        Z      X      C      V   B   N          M
        Undo   Redo   Copy   Pst Cut WordLeft  WordRight
```

### `mod` layer (hold Space)

```
        A      S      D      F       J      K      L      Ö
        Shift  Opt    Ctrl   Cmd     Cmd    Ctrl   Opt    Shift
```

The Q11 splits these across two space bars (`MAC_MOD_L` / `MAC_MOD_R`); the laptop
has one, so both hands live on a single layer. That's strictly more capable — you
can hold Cmd+Opt with the right hand and hit the letter with the left, which is
what makes shortcuts like Cmd+Opt+I comfortable.

## Two things worth knowing

**Cmd shortcuts are translated.** The Q11 runs against a Swedish QWERTY OS layout,
so `LGUI(KC_C)` genuinely is Cmd+c. Here the OS layout is Svorak A5, so the config
presses the physical key that Svorak A5 maps to that letter. Resolved from the
layout's command keyMap: `a→a`, `z→/`, `c→i`, `v→.`, `x→b`.

**Right Option is no longer an Option key.** Cmd+Opt shortcuts (Chrome DevTools
etc.) must use the *left* Option. This matches the Q11, where `ROpt` is already
the FN key. Left Option is deliberately absent from `defsrc` so it passes straight
through to macOS.

## Device scoping

`macos-dev-names-include` restricts kanata to `Apple Internal Keyboard / Trackpad`.
Any device not named there is ignored entirely — the Q11 and the MX Keys never pass
through kanata, so this can be left running permanently with nothing to toggle.

Device config is **not** live-reloadable; adding a keyboard needs a kanata restart.

For per-device behaviour later (e.g. a different config for the MX Keys), kanata
supports `definputdevices` plus `(switch ((device-history <id> 1)) ...)` in a single
process — verified working on 1.12.0, macOS-only. Use `(name "...")`; the
`(hash "...")` matcher documented on `main` is not accepted by 1.12.0.

## Setup

```sh
brew install kanata                        # 1.12.0
kanata --check --cfg kanata.kbd            # validate
sudo kanata --cfg kanata.kbd               # foreground test — Ctrl-C to bail
```

kanata **1.12.0 requires Karabiner-DriverKit-VirtualHIDDevice v6.2.0**
(v8.0.0 is for kanata 1.13.0+), from the
[pqrs-org releases](https://github.com/pqrs-org/Karabiner-DriverKit-VirtualHIDDevice/releases):

```sh
sudo /Applications/.Karabiner-VirtualHIDDevice-Manager.app/Contents/MacOS/\
Karabiner-VirtualHIDDevice-Manager forceActivate
```

then approve the driver extension in System Settings → Privacy & Security.
This is only the virtual keyboard device kanata writes through — Karabiner-Elements
itself is not installed and is not needed.

Run in the foreground until the config feels right; only then set up launchd.

## Not yet ported

- **Double-tap Shift → Caps Lock** (`TD_LSFT` / `TD_RSFT`). kanata's plain
  `tap-dance` would add latency to every Shift press; `tap-dance-eager` is the
  right primitive but needs testing before Shift — a key used constantly — is
  put behind it.
- **Function row F1–F12.** Interaction with the MacBook's own fn/media handling
  needs testing on hardware.
- **`NUMPAD` layer**, `TD_LOCK`, `SE_ACUT`, SOCD/snap-tap, RGB. Either
  hardware-specific or not wanted on the laptop.
