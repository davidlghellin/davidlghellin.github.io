---
layout: default
title: "Una config, N Raspberries: NixOS en mis rpi3"
date: 2026-04-12
categories: Blog
---

# Una config, N <img src="assets/img/rpi-logo.svg" alt="" width="28" style="vertical-align: -4px;"/> Raspberries: <img src="assets/img/nixos-logo.svg" alt="" width="28" style="vertical-align: -4px;"/> NixOS en mis rpi3

<p align="center">
  <img src="assets/img/nixos-logo.svg" alt="NixOS" width="140" style="vertical-align: middle;"/>
  &nbsp;&nbsp;<strong style="font-size: 2em; vertical-align: middle;">+</strong>&nbsp;&nbsp;
  <img src="assets/img/rpi-logo.svg" alt="Raspberry Pi" width="110" style="vertical-align: middle;"/>
</p>

Tengo varias Raspberry Pi 3 dando vueltas por casa. Todas hacen cosas parecidas (DNS, media, algún servicio tonto), pero cada vez que tenía que tocar una era un pequeño drama. Este post cuenta cómo he pasado de *"a ver qué configuré yo aquí hace meses"* a tener **una sola fuente de verdad en git** para todas mis pis.

El repo está aquí: [davidlghellin/nix-os/rpi3](https://github.com/davidlghellin/nix-os/tree/master/rpi3).

---

## 😩 El problema de siempre

Cualquiera que haya tenido una rpi conoce el ciclo:

1. Flasheas Raspbian.
2. `sudo apt install` de 15 cosas que recuerdas más o menos.
3. Editas configs a mano (`/etc/…`), ajustas permisos, abres puertos.
4. Funciona. Te olvidas.
5. **6 meses después**: se corrompe la SD, o quieres replicar el setup en otra pi, o simplemente quieres actualizar algo y no te acuerdas de qué tocaste.
6. Vuelta a empezar.

Y si tienes **varias rpi3 iguales**, peor todavía: cada una acaba ligeramente distinta. Configuras la primera con calma, la segunda con prisa, la tercera copiando mal lo de la segunda… *drift* garantizado.

Lo he hecho tantas veces que ya ni me quejaba. Asumía que era el peaje de tener raspberries.

---

## 🎯 Por qué NixOS encaja justo aquí

NixOS le da la vuelta al problema entero:

- **Todo está en un `flake.nix`** versionado en git. La config no vive "en la pi", vive en el repo. La pi es solo un sitio donde se aplica.
- **Misma config → pis idénticas, bit a bit**. Se acabó el drift.
- **Rollback nativo**: si un `nixos-rebuild` rompe algo, `nixos-rebuild switch --rollback` y vuelves al estado anterior. Sin backups, sin miedo.
- **Si se muere una SD**: flasheas la imagen, `git pull`, rebuild, y tienes exactamente la misma pi que antes. En 20 minutos.

Es decir: la pi pasa de ser una mascota a la que mimas a ser **ganado desechable**. Y eso cambia completamente cómo la tratas.

---

## 📦 Qué corre en mi pi (`myoboku`)

Para que no sea abstracto, este es el setup real de una de mis pis:

- **Hostname**: `myoboku`
- **Usuario**: `wizord` con acceso por SSH key
- **AdGuard Home** (puertos 80/53): bloqueo de anuncios y tracking a nivel DNS para toda la red de casa.
- **MiniDLNA** (puertos 8200/1900): servidor multimedia apuntando a `/mnt/media`, donde monto un disco externo.
- **SSH** con `trusted-users` configurado para poder hacer `nixos-rebuild` remoto desde mi portátil.

Nada exótico. Lo interesante no es *qué* corre, sino *cómo* está descrito: todo en archivos `.nix` que puedo leer, editar y versionar.

---

## 🛠️ Cross-compilar la imagen desde el portátil

Aquí viene el primer truco importante. Compilar NixOS **dentro** de una rpi3 es inviable: poca RAM, CPU lenta, tarda horas. La solución es cross-compilar la imagen desde tu máquina x86 y flashearla después.

En tu NixOS x86, activas emulación `aarch64`:

```nix
boot.binfmt.emulatedSystems = [ "aarch64-linux" ];
```

Y desde el flake del repo:

```bash
nix build .#sdImage
```

Eso te genera un `.img` listo para `dd` a la SD:

```bash
sudo dd if=result/sd-image/nixos-sd-image-*.img of=/dev/sdX bs=4M status=progress
```

Metes la SD en la rpi, arranca, y ya tienes NixOS con tu config base. El primer arranque tarda un poco porque expande el filesystem, pero a partir de ahí todo es `nixos-rebuild`.

Diferencia práctica: compilar una imagen completa de rpi3 en la propia pi → **horas**. Cross-compilar desde un portátil moderno → **minutos**.

---

## 🔗 Cómo el flake enlaza todo

Antes de entrar en la estructura multi-host, vale la pena entender cómo un `flake.nix` arma un sistema NixOS. Este es el mío para la pi, reducido:

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

Lo importante está en **cómo se fusionan los módulos**:

```
sd-image-aarch64.nix      (cómo construir la imagen de SD)
        +
raspberry-pi-3            (firmware, kernel, device tree)
        +
./configuration.nix       (MI config: usuarios, servicios)
        =
sistema final
```

Nix hace ese *merge* automáticamente. En la NixOS clásica, `configuration.nix` **es** el sistema. Con flakes, `configuration.nix` es **solo un módulo más** en una composición mayor. Eso es lo que habilita reutilizar módulos de la comunidad (como `nixos-hardware`) sin copiarlos a tu archivo.

Y `flake.lock` (generado automáticamente) guarda los commits exactos de `nixpkgs` y `nixos-hardware`. Eso es lo que da **reproducibilidad real**: hoy compila igual que dentro de tres años.

La última línea (`images.rpi3`) expone la imagen SD como salida del flake, por eso funciona `nix build .#images.rpi3` desde el portátil.

---

## 🧬 Una config, varias pis

Este es el núcleo del post y lo que justifica el título. La estructura que uso separa **qué se puede hacer** (módulos por rol) de **qué hace cada pi** (hosts):

```
rpi3/
├── flake.nix
├── modules/
│   ├── common.nix      # usuario, ssh, zsh, cosas compartidas
│   ├── adguard.nix     # rol: DNS + bloqueo
│   └── dlna.nix        # rol: media server
└── hosts/
    ├── myoboku.nix     # hostname + qué roles activa
    └── pi2.nix         # otra pi, otros roles
```

La idea clave: los módulos describen **capacidades**, los hosts solo las **componen**. Un host se parece a esto:

```nix
{
  imports = [
    ../modules/common.nix
    ../modules/adguard.nix
    ../modules/dlna.nix
  ];

  networking.hostName = "myoboku";
}
```

Si mañana compro una tercera rpi3 que solo va a ser DNS, el host nuevo son literalmente 5 líneas. Sin copiar config, sin olvidarme de nada, sin drift.

Esta es la diferencia entre *"tengo varias pis"* y *"tengo una flota"*.

---

## 🚀 Deploy remoto

Con la config ya escrita, aplicarla a una pi concreta es un comando desde el portátil:

```bash
nixos-rebuild switch \
  --flake .#myoboku \
  --target-host wizord@myoboku \
  --use-remote-sudo
```

Edito el flake en el portátil, commiteo, lanzo el rebuild, y la pi se actualiza sola. Si algo sale mal: `--rollback` y a otra cosa.

Para 2-3 pis esto sobra. Cuando tenga más, probablemente migre a [deploy-rs](https://github.com/serokell/deploy-rs) o [colmena](https://github.com/zhaofengli/colmena), que están pensados para desplegar a muchos hosts a la vez. Pero ese es otro post.

---

## 🔄 El flujo completo

```
editar flake.nix en el portátil
            ↓
      git commit
            ↓
nixos-rebuild --target-host pi
            ↓
  ¿algo roto? → --rollback
            ↓
 SD muerta → flash imagen + rebuild
            ↓
      misma pi que antes ✅
```

De un día perdido reinstalando, a minutos.

---

## 💭 Reflexión

Lo que más me ha cambiado NixOS en las raspberries no es técnico, es **mental**.

Antes tocaba la pi con miedo. Cada `apt upgrade` era una ruleta, cada cambio de config un "ojalá me acuerde de esto". Ahora la pi me da igual: si se rompe, la reflasheo; si quiero probar algo, lo hago sabiendo que `--rollback` existe; si compro otra, es un clon de la primera en 20 minutos.

La pi dejó de ser una mascota que mimar. La verdad vive en git, la pi solo la ejecuta.

Y eso, después de años reinstalando Raspbian a mano, se siente casi a trampa.

---

## 📚 Recursos

- [Repo nix-os/rpi3](https://github.com/davidlghellin/nix-os/tree/master/rpi3)
- [NixOS on ARM wiki](https://nixos.wiki/wiki/NixOS_on_ARM)
- [deploy-rs](https://github.com/serokell/deploy-rs) · [colmena](https://github.com/zhaofengli/colmena)
- Posts anteriores: [Nix + Sail](0002-nix-sail.md) | [Template Nix](0003-template-nix-sail.md) | [Maintainer en nixpkgs](0004-sail-nixpkgs-maintainer.md)
