---
layout: default
title: "Sail Maintainer in nixpkgs"
date: 2026-03-03
categories: Blog
---

# 🎉 I'm a Sail Maintainer in nixpkgs

<p align="center">
  <img src="assets/img/nix_sail.png" alt="Nix + Sail Logo" width="320"/>
</p>

In the [second post](0002-nix-sail.md) of this blog I wrote:

> _"In the future, if I have time, I'd like to contribute to packaging pysail for Nix."_

Well, not only did I do it, but I'm now an **official Sail maintainer in nixpkgs**. 🚀

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

## 💭 Final Thoughts

Contributing to nixpkgs seemed out of reach when I wrote that post. But in the end, the hardest step is the first one. Once you're in, maintaining a package is simple and rewarding.

And to be honest: I love Sail. Coming from the Spark world, where I know its optimizations, its internals and its limitations, seeing how Sail reimplements all of that in Rust with DataFusion is fascinating. It's not just about using it — it's about understanding how it works under the hood and, over time, being able to help implement things I already know from the Spark ecosystem.

As Marcus Aurelius said: _"The impediment to action advances action. What stands in the way becomes the way."_ The difficulty of the first PR was precisely what led me to become a maintainer.

If you use Nix and there's a package you'd like to see in nixpkgs... go for it. The first PR is tough, but the following ones are just a few commands. 🚀

---

## 📚 Resources

- [Sail in nixpkgs](https://github.com/NixOS/nixpkgs/tree/master/pkgs/by-name/sa/sail)
- [nix-update](https://github.com/Mic92/nix-update)
- [nixpkgs Contributing Guide](https://github.com/NixOS/nixpkgs/blob/master/CONTRIBUTING.md)
- [Previous posts: Nix + Sail](0002-nix-sail.md) | [Template](0003-template-nix-sail.md)
