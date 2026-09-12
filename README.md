# 🖥️ Linux on my GitHub profile

A real shared Alpine Linux guest is running behind the terminal shown on [my GitHub profile](https://github.com/sxwik).

Everyone interacts with the **same guest state**.

## Use it

### 1. Open the console

[Open the SXWIK Linux Console →](https://github.com/sxwik/sxwik/issues/1)

### 2. Add a comment

Put **one Linux command** in the comment box and post it.

```bash
uname -a
```

GitHub Actions then boots the guest, runs the command, persists the disk, and regenerates the terminal snapshot used by the profile.

Try:

```bash
ls -la
pwd
whoami
cat /etc/os-release
echo hello
uname -a
```

### 3. Watch the profile

[Open the profile →](https://github.com/sxwik)

Refresh the profile after the workflow finishes to see the new terminal state.

## DOOM mode 💀

The same issue now has a cursed second mode: **Chocolate Doom running on a GitHub-hosted runner, with the resulting game screen committed back into the profile.**

Write `doom` at the start of a comment and everything after it becomes a stacked input program:

```text
doom

doom !w !w !w !d !d !space

doom !w*20 !d*5 !space !wait:1000 !a*10
```

### Keyboard

Most normal keyboard keys are available as `!` actions:

```text
!w !a !s !d
!up !down !left !right
!space !enter !esc !tab
!shift !ctrl !alt
!f1 ... !f12
```

You can also use explicit X11 key names with `!key:name`.

### Mouse

```text
!click
!left-click
!right-click
!middle-click
!left click
!right click
!move:640:360
!mouse:40:-10
```

### Timing and holds

```text
!wait:500
!hold:w:1500
!space*3
```

A whole input sequence runs in one workflow, so you can stack movement, firing, clicks, waits, and key presses without needing a separate comment for every action.

### Examples

Start Doom and immediately move/fire:

```text
doom !enter !enter !w*8 !space
```

Turn, shoot, strafe, then turn back:

```text
doom !right*8 !space !a*6 !left*8 !space
```

The screenshot on the profile is the **latest captured game frame**, not a live browser game. The game itself runs on the GitHub-hosted runner and disappears when that workflow ends.

## Run a command from GitHub Actions

You can also skip the issue and manually dispatch the workflow.

[Open **Actions → SXWIK Linux Console**](https://github.com/sxwik/sxwik/actions/workflows/linux-console.yml)

For Doom, use the **SXWIK DOOM** workflow instead.

## The machine

```text
GitHub Issue comment
        │
        ├──────────────────────┐
        ▼                      ▼
GitHub Actions             GitHub Actions
        │                      │
        ▼                      ▼
QEMU x86_64              Xvfb + Chocolate Doom
        │                      │
        ▼                      ▼
Alpine Linux             xdotool input program
        │                      │
        ▼                      ▼
terminal SVG/PNG         Doom screenshot
        │                      │
        └──────────┬───────────┘
                   ▼
             github.com/sxwik
```

The Linux guest is a small **Alpine Linux x86_64** system booted by QEMU with a serial console. There is no GUI and no network attached to the guest.

Doom is separate from that guest: it uses Chocolate Doom + the Freedoom IWAD on an Ubuntu GitHub-hosted runner, with Xvfb providing the virtual display and xdotool generating keyboard/mouse events.

## ⚠️ Cons / limitations

This is intentionally cursed. It works, but it is **not** a normal interactive terminal or live game.

### ~45 seconds per Linux command / longer for Doom

A GitHub-hosted runner has to start, install tools, restore state, boot the VM or game, capture the result, commit it, and finish the workflow. Doom also has to install Chocolate Doom, Freedoom, Xvfb, and xdotool.

### Not real-time

You cannot type interactively character-by-character into the Linux process or play Doom in real time. Commands and input programs are submitted as GitHub comments or workflow inputs.

### Commands can race

Linux and Doom share a concurrency lock so two updates do not try to commit the profile screen at once. A newer run can cancel an older run.

### The display is a snapshot

The profile displays the latest generated PNG. GitHub may cache images, so a refresh can be needed before the newest state appears.

### Shared by everyone

The Linux guest has one public shared state. Another visitor can modify the filesystem before your next command. Doom itself is a fresh game session for each run.

### Small guest

The Linux machine is intentionally tiny. It is not a full desktop distribution and is not intended for heavy workloads.

### No network in the Linux VM

The Linux guest has no network interface. `curl`, `wget`, package installation, DNS lookups, and other network-dependent commands will not work inside the guest.

### GitHub Actions dependency

Everything depends on GitHub Actions, Issues, repository write access, runner availability, and Actions artifacts. If Actions is delayed, disabled, or unavailable, the display cannot update.

### Snapshot persistence, not a live server

The Linux guest is not running continuously between commands. Its disk is restored into a fresh runner and then saved again as an artifact after the command completes.

### GitHub rate / platform limits

This project inherits GitHub's limits, runner availability, artifact retention, and abuse/rate-control behavior. It is an experiment built on GitHub rather than a conventional hosted VM.

### Not for secrets

Never put passwords, tokens, API keys, private files, or other sensitive data into the guest or public issue conversation.

## Links

- 🖥️ [Live profile](https://github.com/sxwik)
- 🎛️ [Public Linux Console — Issue #1](https://github.com/sxwik/sxwik/issues/1)
- ⚙️ [SXWIK Linux Console workflow](https://github.com/sxwik/sxwik/actions/workflows/linux-console.yml)
- 💀 [SXWIK DOOM workflow](https://github.com/sxwik/sxwik/actions/workflows/doom.yml)
- 🧠 [Machine source](https://github.com/sxwik/sxwik/tree/main/machine)
- 📟 [Linux workflow source](https://github.com/sxwik/sxwik/blob/main/.github/workflows/linux-console.yml)
- 🎮 [Doom input controller](https://github.com/sxwik/sxwik/blob/main/scripts/doom-input.py)
