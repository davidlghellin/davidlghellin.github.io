---
layout: default
title: "One config, N Raspberries: NixOS on my rpi3s"
date: 2026-04-12
categories: Blog
---

# One config, N <img src="assets/img/rpi-logo.svg" alt="" aria-hidden="true" width="28" style="vertical-align: -4px;"/> Raspberries: <img src="assets/img/nixos-logo.svg" alt="" aria-hidden="true" width="28" style="vertical-align: -4px;"/> NixOS on my rpi3s

<p align="center">
  <img src="assets/img/nixos-logo.svg" alt="NixOS" width="140" style="vertical-align: middle;"/>
  &nbsp;&nbsp;<strong style="font-size: 2em; vertical-align: middle;">+</strong>&nbsp;&nbsp;
  <img src="assets/img/rpi-logo.svg" alt="Raspberry Pi" width="110" style="vertical-align: middle;"/>
</p>

I have a handful of Raspberry Pi 3s scattered between home and another place. They all do similar things (DNS, media, some small service — stuff my Fritz!Box router could half-do, but I'd rather keep them separate and under my control), and every time I had to touch one of them it turned into a small drama. This post is about how I went from *"let me check what I configured here months ago"* to having **a single source of truth in git** for all my pis.

The repo lives here: [davidlghellin/nix-os/rpi3](https://github.com/davidlghellin/nix-os/tree/master/rpi3).

---

## 😩 The usual problem

Anyone who's owned a Raspberry Pi knows the cycle:

1. Flash Raspbian.
2. `sudo apt install` 15 things you sort of remember.
3. Edit configs by hand (`/etc/…`), fix permissions, open ports.
4. Works. You forget.
5. **6 months later**: the SD card dies, or you want to replicate the setup on another pi, or you just want to update something and have no idea what you touched. And even if you *do* remember, nothing guarantees the next `apt upgrade` leaves the system exactly the way it was.
6. Start over.

And if you have **several identical rpi3s**, it gets worse: each one ends up slightly different. You configure the first one carefully, the second in a rush, the third by badly copying the second… *drift* guaranteed. The "right thing" would be to have a script that sets everything up in one go, but either you're too lazy to write it, or you do write it and it becomes outdated within a month.

I've done this so many times I'd stopped complaining. Deep down I knew it could be solved better, but between laziness and lack of time, I just kept assuming it was the tax for owning raspberries.

---

## 🎯 Why NixOS fits here

Upfront: I'm not here to sell NixOS as *the* system. I've used Arch and Debian for years, I like both, and you don't replace something that works just for sport. But neither is really reproducible — that's the missing piece. And when managing several pis, **reproducibility is exactly the problem**. That's why NixOS fits here.

What flips the problem on its head:

- **Everything lives in a `flake.nix`** versioned in git. The config doesn't live "on the pi", it lives in the repo. The pi is just a place where it gets applied.
- **Same config → bit-for-bit identical pis**. No more drift.
- **Native rollback and fearless updates**: I update all at once whenever I want; if something breaks, `nixos-rebuild switch --rollback` and it's as if nothing happened. No backups, no drama.
- **If an SD card dies**: flash the image, `git pull`, rebuild, and you have exactly the same pi as before. In 20 minutes.

In other words: the pi goes from being a pet you care for to being **cattle**. And that changes how you treat it completely.

---

## 📦 What runs on my pi (`myoboku`)

To keep it concrete, this is the real setup of one of my pis:

- **Hostname**: `myoboku`
- **User**: `wizord` with SSH key access
- **AdGuard Home** (ports 80/53): ad and tracking blocking at DNS level for the whole home network.
- **MiniDLNA** (ports 8200/1900): media server pointing at `/mnt/media`, where I mount an external drive.
- **SSH** with `trusted-users` configured to allow remote `nixos-rebuild` from my laptop.

Nothing exotic. What matters isn't *what* runs, but *how* it's described: all in `.nix` files I can read, edit and version.

---

## 🛠️ Cross-compiling the image from the laptop

Here's the first important trick. Compiling NixOS **on** an rpi3 is not viable: little RAM, slow CPU, takes hours. The solution is to cross-compile the image from your x86 machine and flash it afterwards.

On your x86 NixOS, enable `aarch64` emulation:

```nix
boot.binfmt.emulatedSystems = [ "aarch64-linux" ];
```

And from the repo's flake:

```bash
nix build .#images.rpi3
```

That generates an `.img` ready to `dd` to the SD card:

```bash
sudo dd if=result/sd-image/nixos-sd-image-*.img of=/dev/sdX bs=4M status=progress
```

Put the SD in the pi, boot, and you've got NixOS with your base config. First boot takes a little longer because it expands the filesystem, but from then on it's all `nixos-rebuild`.

Practical difference: compiling a full rpi3 image on the pi itself → **hours**. Cross-compiling from a modern laptop → **minutes**.

---

## 🔗 How the flake glues everything together

Before getting into how the pis are laid out, it's worth understanding how a `flake.nix` assembles a NixOS system. This is mine for the pi, shortened:

```nix
{
  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-25.11";
    nixos-hardware.url = "github:NixOS/nixos-hardware";
  };

  outputs = { self, nixpkgs, nixos-hardware }: {
    nixosConfigurations.rpi3 = nixpkgs.lib.nixosSystem {
      system = "aarch64-linux";
      modules = [
        "${nixpkgs}/nixos/modules/installer/sd-card/sd-image-aarch64.nix"
        nixos-hardware.nixosModules.raspberry-pi-3
        ./configuration.nix
      ];
    };

    images.rpi3 = self.nixosConfigurations.rpi3.config.system.build.sdImage;
  };
}
```

The key thing is **how the modules get merged**:

```
sd-image-aarch64.nix      (how to build the SD image)
        +
raspberry-pi-3            (firmware, kernel, device tree)
        +
./configuration.nix       (MY config: users, services)
        =
final system
```

Nix performs that *merge* automatically. In classic NixOS, `configuration.nix` **is** the system. With flakes, `configuration.nix` is **just one more module** in a larger composition. That's what lets you reuse community modules (like `nixos-hardware`) without copying them into your own file.

And `flake.lock` (auto-generated) pins the exact commits of `nixpkgs` and `nixos-hardware`. That's what gives **real reproducibility**: it builds the same today as three years from now.

The last line (`images.rpi3`) exposes the SD image as a flake output — that's why `nix build .#images.rpi3` works from the laptop.

---

## 🧬 The same pi, in different networks

Here's the detail that makes my case easier: my pis **don't live on the same LAN**. One's at home, another one elsewhere — each on its own network. Which means there's no conflict between them: I can give them **exactly the same config** without renaming anything.

Same hostname (`myoboku`), same user (`wizord`), same static IP, same SSH keys, same services on the same ports. They're **literally clones**. Each one, on its own network, thinks it's the only one, and I don't have to remember *"this one was 192.168.1.10 and that one 192.168.2.10"*.

That's why the repo is so simple:

```
rpi3/
├── flake.nix
├── configuration.nix
└── flake.lock
```

One `configuration.nix` for all of them. When I reflash an SD or bring up a new pi, I don't touch the repo — `git pull`, rebuild, done. The only "state" that differs between pis is their network, and that's the router's problem, not NixOS's.

If one day I need a pi with a different role (say, one of them also running Home Assistant), that's when I'll need to split `configuration.nix` into modules and separate hosts. But as long as they all do the same thing on different networks, **identical clones is exactly what I want**.

---

## 🚀 Remote deploy

With the config written, applying it to a given pi is one command from the laptop:

```bash
nixos-rebuild switch \
  --flake .#rpi3 \
  --target-host wizord@myoboku \
  --sudo \
  --use-remote-sudo
```

I edit the flake on the laptop, commit, fire the rebuild, and the pi updates itself. If something goes wrong: `--rollback` and move on.

For 2-3 pis this is plenty. When I have more, I'll probably move to [deploy-rs](https://github.com/serokell/deploy-rs) or [colmena](https://github.com/zhaofengli/colmena), which are designed to roll out to many hosts at once. But that's another post.

---

## 🍎 Updating from the Mac

The laptop above was an x86 NixOS, where `nixos-rebuild` ships out of the box. But sometimes the machine in front of me is a **Mac**, which is neither NixOS nor able to build `aarch64-linux` binaries directly. Even so, with Nix installed, I can fire the deploy all the same.

The trick comes down to two details:

- **`nix run nixpkgs#nixos-rebuild`** pulls the tool on the fly without installing anything permanent (on macOS it doesn't exist as a system command).
- **`--build-host` pointing at the pi itself**: since the Mac can't compile for Linux, I let the pi build its own system. The Mac just orchestrates.

```bash
nix run nixpkgs#nixos-rebuild -- switch \
  --flake .#rpi3 \
  --target-host wizord@192.168.178.24 \
  --build-host wizord@192.168.178.24 \
  --sudo \
  --use-remote-sudo
```

If Nix doesn't respond (freshly installed or a new terminal), source the daemon profile first:

```bash
source /nix/var/nix/profiles/default/etc/profile.d/nix-daemon.sh
```

Because `--build-host` and `--target-host` are the same pi, the Mac compiles nothing heavy: it downloads the closure, the pi does the work, and the result is identical to deploying from the x86 NixOS. Same flake, same `flake.lock`, same final system — the machine I launch from doesn't matter.

---

## 🔄 The full flow

```
edit flake.nix on the laptop
            ↓
       git commit
            ↓
nixos-rebuild --target-host pi
            ↓
  something broken? → --rollback
            ↓
 dead SD → flash image + rebuild
            ↓
       same pi as before ✅
```

From a lost day reinstalling, to minutes.

---

## 💭 Reflection

What NixOS has changed the most on my raspberries isn't technical, it's **mental**.

I used to touch the pi with fear. Every `apt upgrade` was a coin flip, every config change a *"hope I remember this"*. Now the pi just doesn't matter to me: if it breaks, I reflash it; if I want to try something, I do it knowing `--rollback` is there; if I buy another one, it's a clone of the first in 20 minutes.

The pi stopped being a pet to care for. The truth lives in git, the pi just runs it.

And that, after years of reinstalling Raspbian by hand, feels almost like cheating.

---

## 📚 Previous posts

[Nix + Sail](0002-nix-sail.md) | [Nix template](0003-template-nix-sail-en.md) | [Maintainer at nixpkgs](0004-sail-nixpkgs-maintainer-en.md)
