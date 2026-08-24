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

## Installation

Tested on macOS 26.5.2 (MacBookPro18,2, M1 Max) with kanata 1.12.0.

### 1. kanata

```sh
brew install kanata
kanata --version                                  # expect 1.12.0
kanata --check --cfg kanata.kbd                   # expect "config file is valid"
```

`--check` only parses the config; it touches nothing and needs no privileges.

### 2. The virtual HID driver

kanata needs a virtual keyboard device to write its output through. **The version
is coupled to the kanata version** — kanata 1.12.0 requires
Karabiner-DriverKit-VirtualHIDDevice **v6.2.0**. v8.0.0 is for kanata 1.13.0+, and
mismatching fails in a confusing way. A future `brew upgrade kanata` past 1.13
means upgrading the driver in lockstep.

This is *only* the virtual device. Karabiner-Elements is not installed, not needed,
and has no rules here — the driver on its own is inert and changes nothing about
your keyboard.

```sh
curl -LO https://github.com/pqrs-org/Karabiner-DriverKit-VirtualHIDDevice/releases/download/v6.2.0/Karabiner-DriverKit-VirtualHIDDevice-6.2.0.pkg

# verify before installing
pkgutil --check-signature Karabiner-DriverKit-VirtualHIDDevice-6.2.0.pkg
#   Status:        signed by a developer certificate issued by Apple for distribution
#   Notarization:  trusted by the Apple notary service
#   Certificate:   Developer ID Installer: Fumihiko Takayama (G43BCU2T37)

sudo installer -pkg Karabiner-DriverKit-VirtualHIDDevice-6.2.0.pkg -target /
```

### 3. Activate and approve

```sh
sudo /Applications/.Karabiner-VirtualHIDDevice-Manager.app/Contents/MacOS/Karabiner-VirtualHIDDevice-Manager forceActivate
```

Then approve it by hand: **System Settings → General → Login Items & Extensions →
Driver Extensions**. (Older macOS puts this under Privacy & Security.)

Confirm it registered:

```sh
systemextensionsctl list | grep -i karabiner        # expect [activated enabled]
```

### 4. Run it — foreground first

```sh
sudo kanata --cfg ~/Repositories/pelto/mbp-dvorak-kanata/kanata.kbd
```

Root is required: the driver exposes its IPC under
`/Library/Application Support/org.pqrs/tmp/rootonly/`.

If it complains about Input Monitoring, grant it to
`/opt/homebrew/opt/kanata/bin/kanata` in System Settings → Privacy & Security →
Input Monitoring and re-run.

**Stay in the foreground until the config is proven.** Do not set up `launchd` or
`sudo brew services start kanata` yet — a bad config that starts at boot is a much
worse problem than one you can Ctrl-C.

### 5. Verify

Key names below are the legends printed on the keys (physical positions).

| Test | Expect |
|---|---|
| Tap Caps Lock | Enter |
| Right Opt + `J` `K` `L` / `I` | ← ↓ → / ↑ |
| Right Opt + `W` `E` | Home, PgUp |
| Right Opt + `C` `V` | Copy, Paste |
| Space (hold) + `J`+`L`, then `G` | Chrome DevTools (Cmd+Opt+I) |
| Left Opt + `Q` | `{` |
| Right Opt + `Q` | Ins — *not* `{` |

The fifth row is the real end-to-end test: home row mods giving Cmd+Opt on the
right hand while the left hits the letter. The last two confirm special characters
are now left-Option only, achieved by kanata consuming right Option rather than by
editing the keylayout.

## Rollback and recovery

### Nothing here persists

While kanata is run in the foreground there is no `launchd` job, no login item and
no `brew services` unit. It dies with Ctrl-C, with its terminal window, and on
reboot. Svorak A5 is never modified, so the moment kanata stops the keyboard is
back to its normal behaviour. There is no state to unwind.

kanata has **no menu bar icon** — it is a bare CLI process. That is a real
ergonomic loss versus Karabiner-Elements, so know the outs before you start.

### If the keyboard misbehaves

Keyboard: **Ctrl-C**. Under Svorak that is Ctrl + physical `I`; both are untouched
in the base layer.

Mouse only — the trackpad is never affected, since kanata grabs the keyboard device
only:

1. **Close the terminal window** — red X, then "Terminate". Fastest.
2. **Activity Monitor** → search `kanata` → select → X in the toolbar → Force Quit.
3. **Apple menu → Restart.**

If you need to type and cannot: System Settings → Accessibility → Keyboard →
**Accessibility Keyboard** gives a clickable on-screen keyboard.

### Reverting a config change

```sh
git checkout -- kanata.kbd     # discard local edits
kanata --check --cfg kanata.kbd
```

Always `--check` before running. A config that fails to parse will not start, which
is a safe failure; a config that parses but is wrong is the one to be careful with.

### Removing the driver

The pkg ships its own uninstall scripts. The Manager binary exposes exactly three
subcommands — `activate`, `forceActivate`, `deactivate` — so this is the supported
path, not a workaround.

```sh
# 1. deactivate the system extension (run as the console user)
/Applications/.Karabiner-VirtualHIDDevice-Manager.app/Contents/MacOS/Karabiner-VirtualHIDDevice-Manager deactivate

# 2. remove the files (as root)
sudo '/Library/Application Support/org.pqrs/Karabiner-DriverKit-VirtualHIDDevice/scripts/uninstall/remove_files.sh'
```

`remove_files.sh` ends with a chain of `rmdir` calls on
`/Library/Application Support/org.pqrs/`. This machine has stale files there from a
previously removed Karabiner-Elements, so those `rmdir`s will fail harmlessly.

### Removing kanata

```sh
brew uninstall kanata
rm -rf ~/.config/kanata
```

## Not yet ported

- **Double-tap Shift → Caps Lock** (`TD_LSFT` / `TD_RSFT`). kanata's plain
  `tap-dance` would add latency to every Shift press; `tap-dance-eager` is the
  right primitive but needs testing before Shift — a key used constantly — is
  put behind it.
- **Function row F1–F12.** Interaction with the MacBook's own fn/media handling
  needs testing on hardware.
- **`NUMPAD` layer**, `TD_LOCK`, `SE_ACUT`, SOCD/snap-tap, RGB. Either
  hardware-specific or not wanted on the laptop.
