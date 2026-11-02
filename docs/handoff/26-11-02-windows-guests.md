# Handoff: Windows guests on hcs (2026-11-02)

A working note for whoever picks this branch up, human or agent. It is not
design: `DESIGN.md`, `docs/drivers/hcs.md`, and ADR 26-11-02-274 are. Delete
this file before the branch merges.

## Where it started

Running Microsoft's Windows Dev Config setup (`irm
https://aka.ms/devconfig/full/setup.ps1 | iex`) in a Windows 11 guest made
with `disco-vm` failed, and each fix exposed the next problem. The host is
Windows 11 **Home** (build 26300, `EditionID: Core`); the guest is Windows 11
Pro.

## What the branch does (5 commits on top of `main` at e3dca3a)

| commit | what |
|---|---|
| `bc3027c` fix(guest) | A process run as SYSTEM (`exec` without `--user`) gets SYSTEM's profile environment: `processEnv` made its block with no token, so `LOCALAPPDATA`, `APPDATA` and `USERPROFILE` were missing and PowerShellGet could not load. |
| `9353405` fix(guest) | `exec -t --user X` gets a console of X's own, from `disco-vm console-host` run as X. A pseudo console the SYSTEM agent made could be written but not read back by another account, which broke every PowerShell progress bar (`Invoke-WebRequest`). |
| `63420ad` feat(hcs) | `Processor.ExposeVirtualizationExtensions` on every VM: nested virtualization, so WSL 2 and Hyper-V run in a guest. Memory stays `Virtual`-backed. |
| `de738a5` docs(adr) | ADR 26-11-02-274, **Proposed**: the display decision below. |
| `4d9ea7c` feat(hcs)! | The display, below. |

### The display (ADR 26-11-02-274)

The VM worker's video console (vmwp serving RDP on a named pipe) never
answered on this Home host: it accepts the pipe and closes it without a byte,
on any VM, with old and new code. sandboxi proved it on Windows 11 Pro. It is
removed. A window is now:

1. **The guest's own Remote Desktop**, where the guest answers an RDP request
   on its `127.0.0.1:3389` (checked through the agent's `disco-vm pipe`).
   mstsc opens full screen at the host's resolution on a loopback port that
   relays to it through the agent, signed in by a **Domain Password**
   credential for `TERMSRV/127.0.0.1` named `COMPUTER\user`, written for the
   session and deleted after (one the user saved is left alone). Learned the
   hard way: mstsc ignores `password 51:b:` in a `.rdp` file and a *generic*
   credential; NLA does not need to be off; `smart sizing` defeats
   `dynamic resolution`; `PipeReady` (one `0x00` byte) leads every `pipe`
   stream and must be stripped.
2. **Otherwise, the agent's display stream** on `guest.DisplayPort` (7301):
   sandboxi's guestfb ported to Go without cgo. The agent (SYSTEM, session 0)
   starts `disco-vm display-host` in the console session as the logged-on
   user (elevated), reconnecting a session RDP left disconnected
   (`WTSConnectSessionW`, like `tscon`), and relays. GDI `BitBlt` capture,
   changed rows as raw BGRA, the cursor's shape, `SendInput`. The host window
   is plain user32/gdi32 (`pkg/machine/hcs/window_windows.go`).
   `DISCO_VM_HCS_VIEW=display` forces it on a guest that has RDP.

`disco-vm gui INSTANCE` opens a window on a running instance from any process
(`Engine.GUI` → `machine.Viewer` → local driver through the shim →
`local.MachineViewer`, which hcs implements). `start --gui` / `run --gui` open
it from the shim once the guest is up and again after a restart. vz refuses
`gui` as `ErrNotImplemented` (its window needs the shim's VM object).

Install changes: the image's account gets **its own name as its password**
(the agent's `LogonUser` tries the name, then blank, for older images), and
Remote Desktop is turned on except on `Core*` (Home) editions, firewall left
closed. `install --gui` now shows nothing on hcs.

## What is verified, and how

- In a real Win 11 Pro guest on the Home host, by hand: PowerShellGet loads as
  SYSTEM; `exec -t --user disco` reads the console back, draws a progress
  bar, downloads; `exec --user disco` signs in with the name as password;
  VBS runs in the guest and a WSL 2 Debian boots; `start --gui` and `gui` open
  the guest's RDP with no prompt at the right size; the guestfb window shows
  the desktop and takes mouse and keyboard (the user checked both windows).
- `go test ./...` passes; `go vet` on linux, darwin and windows and
  golangci-lint (linux, windows) are clean. Unit tests: `TestConsoleHost`,
  `TestChangedRows`, `TestDisplayRectsSplit`, `TestDocument*`.
- **Not verified:** a fresh `disco-vm install` with the new defaults
  (password, RDP on, `install --gui` message); guestfb on a real Home guest;
  any automated in-guest test of either window path; the darwin lint pass
  (needs a Mac).

## Known problems with this machine's tooling

- `go tool task ci` fails at `fmt`: the task runner's built-in `xargs` has no
  `-r`. Run its parts (`check:vet`, `check:go`, `test`) instead.
- `go tool task check:repo` fails on this Windows checkout: `CLAUDE.md` and
  `.claude/skills` are not symlinks. Not caused by this branch.
- No cgo, so `go test -race` cannot run here.

## Next steps

1. **Accept or amend ADR 26-11-02-274** (the decision gate), then update its
   status and the index in `docs/adr/README.md`.
2. **`install --gui` opens the window once there is a desktop**: run hcs's
   `watch` on the install's VM in `firstBoot`, using the display stream rather
   than RDP (an RDP sign-in would move the console session mid-Setup). It
   shows from OOBE's automatic logon through the Zero Day Patch reboot to the
   commit's shutdown. Nothing can show Setup itself now.
3. **Attach to an install started without `--gui`**: the install writes its
   VM's runtime ID beside its `.attached` breadcrumb; `disco-vm gui --install
   [IMAGE]` finds it (a seam change: the local driver lists installs in
   progress) and dials the guest over hvsocket directly. Amend the ADR.
4. **A clearer error when a guest restarts under `exec`**: a stream that ends
   before an exit code reads "wsarecv: An established connection was aborted";
   say the guest went away (it may have restarted).
5. **Verify the install path** with a fresh `disco-vm install` from an ISO,
   and guestfb on a Home guest; add the results to the proof table in
   `DESIGN.md`.
6. Deferred in the ADR: signing the `.rdp` file (mstsc's publisher warning),
   a window on a running vz instance, compressing the display stream.

## State left on the host and in the guest

- The `win11` instance (`8270f109b062`, image `win11base`) runs under a shim
  from `%TEMP%\disco-vm-gui\disco-vm.exe` (this branch).
  `%USERPROFILE%\go\bin\disco-vm.exe` is older; `go tool task install` after
  `disco-vm stop win11` replaces it.
- Inside `win11`, by hand and not in its image: the agent is this branch's
  build; `disco`'s password is `disco` (and the automatic logon's); the two
  inbound "Remote Desktop - User Mode" firewall rules are on for the local
  subnet only (close them if direct RDP to the guest's IP is not wanted);
  old agents `C:\disco-vm\disco-vm.old-*.exe` and an empty
  `C:\disco-vm\staging-empty` can be deleted.
- `Default.rdp` in the repository's folder is the user's, from mstsc; it is
  not committed.
- The sandboxi source the display was ported from is at
  `C:\Users\darre\src\sandboxi` (`crates/guestcap`, `crates/guestfb`,
  `crates/sandboxw-core/src/display.rs`).
