# UU Controller CLI and Remote Agent

## Scope

UU Remote 4.34.0.8979 includes `uuyc-cli.exe` beside the official controller.
It talks to the already authenticated UU GUI/server through local IPC. The
bridge installs `uu-agent`, a small launcher that discovers the live private
X display, Xauthority file, Wine prefix, and controller executable from the
systemd service. It does not implement or emulate UU's network protocol.

The observed command surface is:

```text
version
user info|wallet
device list|connect|disconnect|status
cloudpc list|launch|shutdown|connect|disconnect
echo
term
```

This is an observed vendor interface, not a stability promise. Run
`uu-agent cli --help` after every upstream UU update and keep automation
bounded to the commands that the installed version reports.

## Local Wrapper

```bash
uu-agent version
uu-agent list
uu-agent status
uu-agent runtime
```

`runtime` prints paths and a display number, but no account or device
identifier. `list` can print private device names and IDs; do not paste it
into issues, CI logs, or a public repository.

The wrapper also provides private-display diagnostics:

```bash
uu-agent windows
uu-agent focus '网易UU远程'
capture="$(uu-agent snapshot)"
printf '%s\n' "$capture"
```

Captures default to `~/.local/state/uu-remote-agent/captures`, use mode `0600`,
and must stay outside Git.

## Mac Terminal Agent

Prefer the terminal agent for deterministic Mac inspection and builds:

```bash
uu-agent term 'Mac device name' --shell zsh --new-session
```

Non-interactive input is supported by the tested CLI:

```bash
printf '%s\n' \
  'sw_vers' \
  'xcodebuild -version' \
  'xcrun simctl list devices available' \
  'exit' |
  uu-agent term 'Mac device name' --shell zsh --new-session
```

Select an exact device name from the live list. Never hard-code a controller
device ID in a script or document. A terminal session has the authority of the
logged-in remote user; do not send passwords, signing secrets, recovery keys,
or destructive disk commands through reusable scripts.

The current CLI advertises `powershell`, `cmd`, `zsh`, and `bash`. Actual
support depends on the controlled platform and UU host version. In particular,
the tested Windows 7 host reports that its terminal agent is unsupported.

## GUI Fallback

Use GUI control only when the task inherently needs Xcode, Simulator, System
Settings, keychain approval, or another visual surface:

```bash
uu-agent connect 'Mac device name'
uu-agent windows
uu-agent snapshot
```

Controller success means only that a request reached the local UU process.
Confirm that a remote window appeared before sending input. Coordinates are
resolution- and version-dependent, so scripts must rediscover the current
window and inspect a fresh private screenshot. Never automate account
publication, a purchase, firmware flashing, credential entry, or a destructive
confirmation dialog.

## Headless Mac Failure Mode

UU can show a Mac as online while both GUI connection and `term` fail. The
account/device heartbeat proves that the host service reached UU; it does not
prove that macOS has an active framebuffer or that the terminal agent opened a
session.

Observed on the headless 7050 iMac:

- the device remained online after its monitor was unplugged;
- SSH and macOS Screen Sharing were not listening;
- the UU terminal progressed through connection setup and then timed out
  waiting for the remote open response;
- earlier GUI attempts reached a wallpaper or connection optimization state
  but did not become controllable.

Restore one independent access path before changing the host:

1. Temporarily reconnect a monitor or use an HDMI/DisplayPort dummy plug.
2. Enable macOS Screen Sharing or Remote Login for a named local user.
3. Install a persistent virtual display if software-only headless operation is
   required.

BetterDisplay supports virtual screens for headless Macs. On macOS 13.2 or
later, install it from its official Homebrew cask:

```bash
brew install --cask betterdisplay
open -a BetterDisplay
```

Create one named virtual screen, verify that macOS and UU both capture it, and
enable BetterDisplay at login before unplugging the monitor again. Its current
CLI supports `create -type=VirtualScreen`, `virtualScreenName`,
`resolutionList`, `virtualScreenHiDPI`, and `connected`. Query the installed
version's help before scripting those parameters.

The hardware dummy plug is the lowest-maintenance recovery path. A software
virtual screen is more flexible, but it depends on the GUI login session,
BetterDisplay startup, and macOS permissions.

## Failure Interpretation

| Observation | Meaning | Next action |
| --- | --- | --- |
| `device list` says offline | UU host heartbeat is absent | Check power, network, login, and host service |
| Online, terminal open timeout | Host service is reachable but agent session did not open | Restore a display or independent SSH/Screen Sharing path |
| Terminal works, GUI stalls | Agent is healthy; capture/display path is not | Inspect displays and install or reconnect a framebuffer |
| GUI works, terminal unsupported | Platform/version lacks the terminal agent | Use GUI once to install SSH |
| CLI exits `2` | Local controller IPC is unavailable | Verify `uu-remote-bridge.service` and UU server |
| CLI exits `5` | Vendor operation timed out | Do not retry destructive actions blindly |

## Security Boundary

`uu-agent` neither stores verification codes nor bypasses controlled-host
authentication. UU account state remains in the dedicated Wine prefix. Device
IDs, screenshots, terminal transcripts, and account output are operator data
and must stay out of source control.
