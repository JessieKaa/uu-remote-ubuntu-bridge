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

## Headless Mac: Separate GUI and Terminal Health

The 7050 iMac's operator later confirmed several successful UU GUI connections
with the monitor unplugged. UU therefore works headlessly on this host without
a dummy plug or virtual display. This supersedes the earlier inference that a
missing active display caused the connection problem.

The observations remain useful when kept separate:

- the device remained online after its monitor was unplugged;
- UU GUI connections opened repeatedly and were usable without the monitor;
- one UU `term` attempt progressed through setup and then timed out waiting for
  the remote open response;
- SSH and macOS Screen Sharing were not listening at the time of that terminal
  test.

The terminal-agent timeout does not imply that the UU GUI, framebuffer, or
input path is broken. Diagnose and verify the device heartbeat, GUI video, GUI
input, and terminal agent independently.

Do not reconnect a monitor, install BetterDisplay, or add a dummy plug solely
to repair UU when its headless GUI already works. Remote Login and Screen
Sharing are optional independent recovery paths. A virtual display is also
optional and is appropriate only when a real resolution, framebuffer, or
capture problem has been reproduced.

If a virtual screen is actually required, BetterDisplay supports one on macOS
13.2 or later and can be installed from its official Homebrew cask:

```bash
brew install --cask betterdisplay
open -a BetterDisplay
```

Create only the named virtual screen needed for the reproduced display issue,
then verify that macOS and UU both capture it. BetterDisplay's current CLI
supports `create -type=VirtualScreen`, `virtualScreenName`, `resolutionList`,
`virtualScreenHiDPI`, and `connected`. Query the installed version's help
before scripting those parameters.

## Failure Interpretation

| Observation | Meaning | Next action |
| --- | --- | --- |
| `device list` says offline | UU host heartbeat is absent | Check power, network, login, and host service |
| Online, terminal open timeout | Host service is reachable but the terminal-agent session did not open | Test GUI separately; inspect agent/version state or use optional SSH |
| Terminal works, GUI stalls | Agent is healthy; capture/display path is not | Inspect permissions and display state for the reproduced GUI failure |
| GUI works, terminal unsupported | Platform/version lacks the terminal agent | Use GUI once to install SSH |
| CLI exits `2` | Local controller IPC is unavailable | Verify `uu-remote-bridge.service` and UU server |
| CLI exits `5` | Vendor operation timed out | Do not retry destructive actions blindly |

## Security Boundary

`uu-agent` neither stores verification codes nor bypasses controlled-host
authentication. UU account state remains in the dedicated Wine prefix. Device
IDs, screenshots, terminal transcripts, and account output are operator data
and must stay out of source control.
