---
layout: default
title: "nixpkgs Maintainer"
date: 2026-03-03
categories: Blog
---

# 🎉 I'm a nixpkgs Maintainer

<p align="center">
  <img src="assets/img/nix_sail.png" alt="Nix + Sail Logo" width="320"/>
</p>

In the [second post](0002-nix-sail.md) of this blog I wrote:

> _"In the future, if I have time, I'd like to contribute to packaging pysail for Nix."_

Well, not only did I do it, but I'm now a **nixpkgs maintainer**, and the first package I maintain is Sail. 🚀

---

## 🏔️ The Hard Part: The First Merge

The most challenging part of contributing to nixpkgs is **the first PR**. The repository is huge, it has its own conventions, and the review process is demanding (and rightfully so).

For that first merge you need to:

- Understand the nixpkgs structure and how packaging works
- Write the correct derivation (in Sail's case, a Rust project with dependencies)
- Pass all CI checks
- Respond to the nixpkgs team's review
- Be patient 😄

But once your package is in and you're a maintainer... everything changes.

---

## ⚡ Version Update: 4 Commands

Once you're a maintainer, bumping a version is ridiculously simple. It's literally 4 commands. Why? Because nixpkgs already has tools built for this. You just need to enter a `nix-shell` with the `nix-update` tool and it does the heavy lifting for you.

### 1. Enter the shell with nix-update

```bash
nix-shell -p nix-update
```

We enter a temporary Nix shell that has the `nix-update` tool available. No need to install anything on your system — it downloads and runs on the fly.

### 2. Update the package

```bash
nix-update sail
```

This is the command that does the magic. `nix-update` automatically:
- Detects the latest version from the upstream repository
- Updates the source hash
- Updates the `cargoHash` (Rust dependencies)
- Modifies the `default.nix` file with the new version

### 3. Build and verify

```bash
nix-build -A sail
```

We build the package with the new version to make sure everything works.

### 4. Check the version

```bash
./result/bin/sail --version
```

If the version is correct, just open a PR and you're done.

---

## 🔄 The Full Flow

```
New Sail version released
        ↓
nix-shell -p nix-update
        ↓
nix-update sail
        ↓
nix-build -A sail
        ↓
./result/bin/sail --version
        ↓
git commit + PR → merge ✅
```

From a task that took days the first time, to something done in **minutes**.

---

## 🐍 Updating pysail too

Sail isn't just the Rust CLI: I also packaged **`pysail`**, the Python bindings. It lives in a different part of the nixpkgs tree (`pkgs/development/python-modules/pysail`) and is exposed under the Python _package set_, so the attribute is prefixed: `python3Packages.pysail`.

Same flow, just a different attribute:

```bash
nix-update python3Packages.pysail
```

`nix-update` does a bit more here. `pysail` is built with **maturin** (it compiles the Rust part and exposes it to Python via PyO3), so on top of the source hash it also refreshes the Cargo _vendor_ hash (`cargoDeps` / `fetchCargoVendor`). One command keeps both in sync.

And to build:

```bash
nix-build -A python3Packages.pysail
```

### How to verify it works

Here's where it differs from the CLI: `pysail` isn't a binary you run with `--version`, it's a **Python module**. The nice part is that the derivation checks itself. The `default.nix` has a `pythonImportsCheck`:

```nix
pythonImportsCheck = [
  "pysail"
  "pysail._native"
];
```

That means if `nix-build` **finishes without error, the module actually imports** — including the native `pysail._native` extension, which is the compiled Rust part. A green build already tells you the bindings load. In fact, near the end of the log you'll see the phase that confirms it:

```
Running phase: pythonImportsCheckPhase
Check whether the following modules can be imported: pysail pysail._native
```

### How to test it

The derivation ships a lightweight version test under `passthru.tests`, which you can build directly:

```bash
nix-build -A python3Packages.pysail.tests.version
```

Which prints the version reported by the package and confirms it matches:

```
sail 0.6.6
```

What does **not** run during the build is the full test suite: it's disabled on purpose (`doCheck = false`) because it needs a running Spark Connect server and a pile of heavyweight optional dependencies (`pyspark-client`, `duckdb`…). For nixpkgs, the module importing and the version being correct is enough; exercising the whole engine is upstream's job, not the packaging's.

```
New Sail version released
        ↓
nix-update python3Packages.pysail
        ↓
nix-build -A python3Packages.pysail   (pythonImportsCheck ✅)
        ↓
nix-build -A python3Packages.pysail.tests.version
        ↓
git commit + PR → merge ✅
```

---

## 🔍 Reviewing a PR with nixpkgs-review

Being a maintainer isn't just bumping versions: you also **review PRs** — your own before opening them, and other people's to help get them merged. The standard tool for this is [`nixpkgs-review`](https://github.com/Mic92/nixpkgs-review). It fetches the change into an **isolated worktree**, builds only the affected packages (or pulls them from the binary cache if they're already there), and when it's done it drops you into a **shell** with them built so you can try them out.

I usually **don't have it installed** on my system, so I pull it on the fly with `nix-shell -p nixpkgs-review` and run it inside the same command. Just like `nix-update`, nothing permanent gets installed. (If you prefer, `nix run nixpkgs#nixpkgs-review -- …` does the same thing.)

There are three ways to use it depending on what you're reviewing.

### 1. A GitHub PR (someone else's)

First check the **cost without compiling anything**:

```bash
nix-shell -p nixpkgs-review --run "nixpkgs-review pr 539480 --dry-run"
```

If the `--dry-run` is mostly downloads (`↓` from the cache) and few local builds, go ahead. If it's a lot of heavy builds, maybe leave it for another day. When you decide to actually review it:

```bash
nix-shell -p nixpkgs-review --run "nixpkgs-review pr 539480"
```

### 2. Your **uncommitted** changes (before opening the PR)

This is what I run right after `nix-update`, to test the bump before committing anything. It compares your working tree against the base:

```bash
nix-shell -p nixpkgs-review --run "nixpkgs-review wip"
```

### 3. A specific commit or your branch

If you've already committed (or want to review what's on top of `master`):

```bash
nix-shell -p nixpkgs-review --run "nixpkgs-review rev HEAD"
```

### Validating in the shell

Once the build finishes, `nixpkgs-review` **drops you into a shell** with every built package on your `PATH`. That's where you check they actually work. A PR that bumps `sail` and `pysail` at once brings both, and each is validated differently:

```bash
# sail: it's a binary
sail --version                          # → sail 0.6.6

# pysail: it's a Python module
python3 -c "import pysail; print('ok')"
```

### The output you upload to the PR

Besides the shell, `nixpkgs-review` leaves a **`report.md`** under `~/.cache/nixpkgs-review/pr-<N>/`. This is the **real** output of reviewing PR [539480](https://github.com/NixOS/nixpkgs/pull/539480), which bumps `sail` and `pysail` from `0.6.5` to `0.6.6` in a single change:

```markdown
## `nixpkgs-review` result

Command: `nixpkgs-review pr 539480`
Commit: `47340d6f…`

### `aarch64-darwin`
:white_check_mark: 5 packages built:
- python313Packages.pysail
- python313Packages.pysail.dist
- python314Packages.pysail
- python314Packages.pysail.dist
- sail
```

A single `pysail` `default.nix` produces **four** entries: nixpkgs instantiates it for **each supported interpreter** (3.13 and 3.14), and each one drags along its `.dist` (the sdist/wheel). Plus `sail`, the Rust CLI → 5 total.

### How I paste it into the PR

Three ways to pull that report out to paste it:

```bash
# 1. cat + copy straight to the clipboard (macOS)
cat ~/.cache/nixpkgs-review/pr-539480/report.md | pbcopy

# 2. have nixpkgs-review print it when it finishes
nix-shell -p nixpkgs-review --run "nixpkgs-review pr 539480 --print-result"

# 3. have it post as a PR comment (needs a GitHub token: gh auth login)
nix-shell -p nixpkgs-review --run "nixpkgs-review pr 539480 --post-result"
```

Two things I learned the hard way:

- **Paste it without wrapping in backticks**, so GitHub renders the `<details>` block and the green checks.
- **Say which platform you tested on** (`aarch64-darwin` on my Mac, for example). Don't claim platforms you don't have — ofborg and other reviewers cover the rest.

A typical review message:

```
Result of `nixpkgs-review pr 539480` run on aarch64-darwin — builds and tests pass. LGTM 👍
```

---

## 💭 Final Thoughts

Contributing to nixpkgs seemed out of reach when I wrote that post. But in the end, the hardest step is the first one. Once you're in, maintaining a package is simple and rewarding.

And to be honest: I love Sail. Coming from the Spark world, where I know its optimizations, its internals and its limitations, seeing how Sail reimplements all of that in Rust with DataFusion is fascinating. It's not just about using it — it's about understanding how it works under the hood and, over time, being able to help implement things I already know from the Spark ecosystem.

As Marcus Aurelius said: _"The impediment to action advances action. What stands in the way becomes the way."_ The difficulty of the first PR was precisely what led me to become a maintainer.

If you use Nix and there's a package you'd like to see in nixpkgs... go for it. The first PR is tough, but the following ones are just a few commands. 🚀

---

## 📚 Resources

- [Sail in nixpkgs](https://github.com/NixOS/nixpkgs/tree/master/pkgs/by-name/sa/sail)
- [pysail in nixpkgs](https://github.com/NixOS/nixpkgs/tree/master/pkgs/development/python-modules/pysail)
- [nix-update](https://github.com/Mic92/nix-update)
- [nixpkgs Contributing Guide](https://github.com/NixOS/nixpkgs/blob/master/CONTRIBUTING.md)
- [Previous posts: Nix + Sail](0002-nix-sail.md) | [Template](0003-template-nix-sail.md)
