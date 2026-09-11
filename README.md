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

## Run a command from GitHub Actions

You can also skip the issue and manually dispatch the workflow.

[Open **Actions → SXWIK Linux Console**](https://github.com/sxwik/sxwik/actions/workflows/linux-console.yml)

Choose **Run workflow**, enter a command, and dispatch it.

## The machine

```text
GitHub Issue comment
        │
        ▼
GitHub Actions
        │
        ▼
QEMU x86_64
        │
        ▼
Alpine Linux
        │
        ▼
command execution
        │
        ├── persistent disk snapshot
        └── terminal SVG
                 │
                 ▼
          github.com/sxwik
```

The guest is a small **Alpine Linux x86_64** system booted by QEMU with a serial console. There is no GUI and no network attached to the guest.

## ⚠️ Cons / limitations

This is intentionally cursed. It works, but it is **not** a normal interactive terminal.

### ~45 seconds per command

A GitHub-hosted runner has to start, install the VM tools, restore the disk, boot QEMU, execute the command, render the terminal, commit the result, and upload the new disk snapshot.

In the current setup, a command can take roughly **tens of seconds, around ~45s at worst** depending on runner startup and network conditions.

### Not real-time

You cannot type interactively character-by-character into the Linux process. Commands are submitted as GitHub comments or workflow inputs.

### Commands can queue

The workflow uses a concurrency lock so two commands do not modify the shared machine simultaneously. If several people submit commands around the same time, later commands wait.

### The display is a snapshot

The profile does not contain a live terminal process. It displays an SVG generated from the latest VM console output. GitHub may also cache images, so a refresh can be needed before the newest state appears.

### Shared by everyone

There is one public guest state. Another visitor can modify the filesystem before your next command.

### Small guest

This is intentionally tiny. It is not a full desktop distribution and is not intended for heavy workloads.

### No network

The VM has no network interface. `curl`, `wget`, package installation, DNS lookups, and other network-dependent commands will not work inside the guest.

### GitHub Actions dependency

The machine depends on GitHub Actions, Issues, repository write access, and Actions artifacts. If Actions is delayed, disabled, or unavailable, the machine cannot update.

### Snapshot persistence, not a live server

The guest is not running continuously between commands. Its disk is restored into a fresh runner and then saved again as an artifact after the command completes.

### GitHub rate / platform limits

This project inherits GitHub's limits, runner availability, artifact retention, and abuse/rate-control behavior. It is an experiment built on GitHub rather than a conventional hosted VM.

### Not for secrets

Never put passwords, tokens, API keys, private files, or other sensitive data into the guest or public issue conversation.

## Links

- 🖥️ [Live profile](https://github.com/sxwik)
- 🎛️ [Public Linux Console — Issue #1](https://github.com/sxwik/sxwik/issues/1)
- ⚙️ [SXWIK Linux Console workflow](https://github.com/sxwik/sxwik/actions/workflows/linux-console.yml)
- 🧠 [Machine source](https://github.com/sxwik/sxwik/tree/main/machine)
- 📟 [Workflow source](https://github.com/sxwik/sxwik/blob/main/.github/workflows/linux-console.yml)
