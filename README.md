# 🖥️ Linux on a GitHub Profile

This is a deliberately cursed experiment: a **real Alpine Linux x86_64 filesystem** is booted inside **QEMU on GitHub Actions**, commands are executed inside that guest, the guest disk is persisted between runs, and the resulting console is rendered back onto my GitHub profile.

It also has a Doom mode: **Chocolate Doom runs inside the same Alpine guest**, not as a separate desktop process on the GitHub runner.

👉 [Open the live console — Issue #1](https://github.com/sxwik/sxwik/issues/1)

---

## What is actually running?

The important distinction is that there are **three different layers**:

```text
┌───────────────────────────────────────────────────────────┐
│ GitHub                                                     │
│                                                           │
│  Issue #1 / Actions / repository                          │
│                 │                                         │
│                 ▼                                         │
│       GitHub-hosted Ubuntu runner                         │
│                 │                                         │
│                 ▼                                         │
│            QEMU x86_64 TCG                                │
│                 │                                         │
│                 ▼                                         │
│          Alpine Linux x86_64                              │
│                 │                                         │
│          ┌──────┴────────┐                                 │
│          │               │                                 │
│       shell          Doom mode                            │
│                          │                                 │
│                  Xvfb + Openbox                           │
│                          │                                 │
│                  Chocolate Doom                           │
│                          │                                 │
│                       Freedoom                            │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

The Ubuntu machine is only the **host**. The interesting part is the Alpine guest running inside QEMU.

For normal commands, the workflow boots Alpine and executes a shell command in the guest.

For Doom commands, the workflow boots the same Alpine disk, starts a virtual X server **inside Alpine**, launches Chocolate Doom there, injects the requested input, captures the frame, closes Doom, and then finishes by syncing the same Alpine disk.

---

# 🚀 How to use it

## 1. Open the console

Go to:

**[SXWIK Linux Console — Issue #1](https://github.com/sxwik/sxwik/issues/1)**

The issue is effectively the public command interface.

## 2. Run a normal Linux command

Post a single command as a comment:

```bash
uname -a
```

Other useful tests:

```bash
pwd
whoami
ls -la
cat /etc/os-release
echo hello
uname -srmo
```

The comment is picked up by the Linux command router, GitHub Actions starts a fresh Ubuntu runner, and that runner boots the persistent Alpine disk with QEMU.

## 3. Wait for the workflow

The runner restores the latest saved filesystem image, executes the command, captures the console output, saves the new filesystem image as an Actions artifact, and updates the profile screen.

The profile is **not a live terminal**. It is a generated snapshot of the latest result.

---

# 💀 Doom mode

Doom is controlled from the same Issue #1 interface.

Start a Doom command with:

```text
doom
```

or:

```text
doom !enter !enter !w*20 !d*10 !space
```

Everything after `doom` is interpreted as an input program.

### Keyboard

```text
!w !a !s !d
!up !down !left !right
!space !enter !esc !tab
!shift !ctrl !alt
!f1 !f2 ... !f12
```

Explicit key names are also supported:

```text
!key:F1
!key:Return
```

### Repetition

```text
!w*20
!space*3
!right*8
```

### Holds

```text
!hold:w:1500
```

That means hold `w` for 1500 ms.

### Waits

```text
!wait:500
```

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

### Typing

```text
!type:hello
```

The complete sequence is executed in one workflow run.

Example:

```text
doom !enter !enter !w*8 !space !wait:500 !a*5
```

---

# 🧠 How the Linux machine persists

This is **not** a long-running VM.

GitHub-hosted runners are temporary. Every workflow run gets a fresh runner, so the VM is reconstructed on every run.

The trick is the disk image.

```text
Run N
  │
  ├─ download latest linux-state artifact
  │
  ├─ boot linux-state.img in QEMU
  │
  ├─ modify filesystem
  │
  ├─ shut down guest
  │
  └─ upload linux-state.img
              │
              ▼
          Run N+1
              │
              └─ restore same image
```

So the **machine is recreated**, but the **filesystem survives**.

That is why this feels like one shared Linux box even though there is no permanent server sitting somewhere.

---

# 💾 What is inside the disk?

The persistent image is an ext4 filesystem based on Alpine Linux.

On the first run, the workflow creates the disk and extracts an Alpine minirootfs into it.

After that, the same disk image is restored from the newest `sxwik-linux-state-*` Actions artifact.

The guest currently uses roughly **512 MiB of virtual disk space** and **512 MiB RAM**.

The disk can therefore remember things such as:

```text
/root files
/root/.doom-saves
installed Alpine packages
files created by commands
configuration changes
```

The exact contents are shared between users because everyone is interacting with the same persisted image.

---

# 🎮 How Doom works inside Alpine

Doom is not a second VM and it is not running directly on Ubuntu.

When the command begins with `doom`, the Alpine-side startup script does approximately this:

```text
boot Alpine
   │
   ├─ detect "doom" command
   │
   ├─ install Chocolate Doom + dependencies
   │      └─ only when they are not already present
   │
   ├─ start Xvfb on :99
   ├─ start Openbox
   ├─ start Chocolate Doom
   ├─ send XTEST keyboard/mouse events
   ├─ capture 1280×720 screenshot
   ├─ kill Chocolate Doom
   ├─ stop Xvfb/Openbox
   │
   └─ continue to the normal Alpine shutdown/sync path
```

The important part is the middle:

```text
Alpine
  └── Xvfb
       └── Openbox
            └── Chocolate Doom
```

So when Doom exits, it is simply another process being terminated inside the guest. The workflow then returns to the normal Alpine command-completion path and persists the disk.

---

# 🖼️ How the profile screen is generated

The profile image is not streamed from the VM.

For Linux commands:

```text
serial console output
        │
        ▼
/tmp/console.tail
        │
        ▼
SVG terminal renderer
        │
        ▼
linux/screen.svg
```

For Doom:

```text
Alpine X display
      │
      ▼
/root/doom.png
      │
      ▼
runner extracts image
      │
      ▼
linux/screen.svg
```

The repository profile then points at the generated `screen.svg` and uses a cache-busting query parameter so GitHub is encouraged to fetch the newest version.

That means the image you see on the profile is always a **snapshot**, never a live framebuffer.

---

# 🔐 Command routing

The public Issue #1 interface separates normal Linux commands and Doom commands.

Normal commands are routed to the Linux workflow.

Commands beginning with `doom` are routed to Doom handling.

That separation is intentional so a Doom command cannot accidentally be executed as a shell command.

The Linux workflow also validates the command as a single line and limits its length.

Doom input is similarly limited to a bounded number of actions so a single comment cannot create an absurdly long workflow run.

---

# 🛑 Stopping Doom

Doom runs only for the duration of its Action job.

To cancel an active Doom run, comment:

```text
doom stop
```

or:

```text
doom-stop
```

The stop workflow looks for an active `SXWIK DOOM` run and asks GitHub Actions to cancel it.

When a normal Doom run reaches its end, the guest-side script also explicitly kills the Doom process before syncing the Alpine filesystem.

So there is no hidden permanent Doom process left running between Actions jobs.

---

# ⚙️ Manual workflow dispatch

You can also run the workflows without using Issue comments.

### Linux

[Open SXWIK Linux Console workflow](https://github.com/sxwik/sxwik/actions/workflows/linux-console.yml)

Enter the command in the workflow input.

### Doom

[Open SXWIK DOOM workflow](https://github.com/sxwik/sxwik/actions/workflows/doom.yml)

The Doom workflow is retained as a manual entry point, while the public Issue interface is the easiest way to use the machine.

---

# 🏗️ The actual workflow architecture

Here is the end-to-end path for a normal Linux command:

```text
Issue comment
     │
     ▼
GitHub Actions router
     │
     ▼
Ubuntu GitHub-hosted runner
     │
     ├─ restore latest linux-state artifact
     │
     ├─ prepare Alpine disk
     │
     ├─ download Alpine kernel/initramfs
     │
     ├─ build minimal initramfs with VirtIO/ext4 modules
     │
     ▼
QEMU x86_64 (TCG)
     │
     ▼
Alpine Linux
     │
     ├─ mount /dev/vda
     ├─ execute /root/.sxwik-command
     ├─ sync
     └─ emit SXWIK_DONE
     │
     ▼
runner captures serial output
     │
     ├─ render screen.svg
     ├─ commit profile changes
     └─ upload updated linux-state.img
```

And for Doom:

```text
Issue comment: doom ...
          │
          ▼
    Doom routing
          │
          ▼
   restore same disk
          │
          ▼
      QEMU x86_64
          │
          ▼
      Alpine Linux
          │
          ├── Xvfb
          │    └── Openbox
          │         └── Chocolate Doom
          │
          ├── inject inputs
          ├── capture doom.png
          ├── close Doom
          ├── sync filesystem
          └── SXWIK_DONE
          │
          ▼
   extract screenshot
          │
          ▼
      screen.svg
          │
          ▼
     GitHub profile
```

---

# 📦 Files that make it work

The main repository contains the implementation pieces.

```text
.github/workflows/linux-console.yml
    Main Alpine/QEMU execution pipeline.

.github/workflows/doom.yml
    Doom entry point and manual Doom workflow.

.github/workflows/doom-stop.yml
    Cancels an active Doom workflow.

scripts/doom-input.py
    Parses the compact Doom input language and sends XTEST events.

machine/
    Machine/bootstrap related files.

linux/state artifact
    Persistent filesystem image restored between runs.
```

The exact implementation can change over time, but the architectural idea stays the same: **GitHub Actions provides the disposable host, QEMU provides the guest hardware, Alpine provides the persistent userspace, and GitHub artifacts provide the illusion of a continuing machine.**

---

# ⚠️ Limitations

### It is not real-time

There is no permanent interactive SSH session or browser terminal. Every command is a workflow execution.

### The VM is not always running

Between Actions jobs there is no live QEMU process. The disk survives; the machine does not.

### Doom is not a live web game

The profile only receives the final captured frame from a run.

### It is shared

The same filesystem image is shared across users. Someone else can modify it before your next command.

### It is intentionally small

This is an experiment, not a replacement for a normal VPS.

### Network behavior is limited

The guest environment is designed around the workflow's controlled boot path and should not be treated as a normal internet-connected server.

### Actions can be slow

Every run involves a GitHub-hosted runner, QEMU boot, image restoration, execution, rendering, Git operations, and artifact persistence.

### GitHub controls the infrastructure

Runner availability, Actions limits, artifact retention, repository permissions, rate limits, and caching can all affect the machine.

### Do not store secrets

Never put passwords, API tokens, private keys, personal documents, or other sensitive information into the guest or public Issue conversation.

---

# 🧪 Why this exists

Because apparently putting a persistent Linux machine behind a GitHub profile README was not cursed enough.

The fun part is that the pieces are individually ordinary:

```text
GitHub Issues
GitHub Actions
QEMU
Alpine Linux
ext4
VirtIO
Xvfb
Openbox
Chocolate Doom
```

The cursed part is combining them into one public interface and pretending the profile picture is a computer.

---

# 🔗 Links

- 🖥️ [Live profile](https://github.com/sxwik)
- 🎛️ [Public Linux Console — Issue #1](https://github.com/sxwik/sxwik/issues/1)
- ⚙️ [Linux workflow](https://github.com/sxwik/sxwik/blob/main/.github/workflows/linux-console.yml)
- 💀 [Doom workflow](https://github.com/sxwik/sxwik/blob/main/.github/workflows/doom.yml)
- 🛑 [Doom stop workflow](https://github.com/sxwik/sxwik/blob/main/.github/workflows/doom-stop.yml)
- 🎮 [Doom input controller](https://github.com/sxwik/sxwik/blob/main/scripts/doom-input.py)
- 🧠 [Machine source](https://github.com/sxwik/sxwik/tree/main/machine)

---

> **TL;DR:** Comment on Issue #1 → GitHub Actions boots a fresh QEMU VM → the same persistent Alpine disk is restored → your command runs inside Alpine → Doom can launch inside that same Alpine guest → the result is captured → the disk is saved again → the profile display gets updated.
