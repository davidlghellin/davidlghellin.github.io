---
layout: default
title: "Maintainer en nixpkgs"
date: 2026-03-03
categories: Blog
---

# 🎉 Soy Maintainer en nixpkgs

<p align="center">
  <img src="assets/img/nix_sail.png" alt="Nix + Sail Logo" width="320"/>
</p>

En el [segundo post](0002-nix-sail.md) de este blog escribí:

> _"En el futuro, si tengo tiempo, me gustaría contribuir al empaquetado de pysail para Nix."_

Pues bien, no solo lo he hecho, sino que ahora soy **maintainer en nixpkgs**, y el primer paquete que mantengo es Sail. 🚀

---

## 🏔️ Lo Difícil: El Primer Merge

La parte más complicada de contribuir a nixpkgs es **el primer PR**. El repositorio es enorme, tiene sus convenciones, y el proceso de revisión es exigente (y con razón).

Para ese primer merge hay que:

- Entender la estructura de nixpkgs y cómo se empaqueta
- Escribir la derivación correcta (en el caso de Sail, un proyecto Rust con dependencias)
- Pasar todos los checks del CI
- Responder a la revisión del equipo de nixpkgs
- Tener paciencia 😄

Pero una vez que tu paquete está dentro y eres maintainer... todo cambia.

---

## ⚡ Actualizar Versión: 4 Comandos

Una vez que eres maintainer, subir de versión es ridículamente sencillo. Literalmente son 4 comandos. ¿Por qué? Porque nixpkgs ya tiene herramientas preparadas para esto. Solo necesitas entrar en un `nix-shell` con la herramienta `nix-update` y ella se encarga del trabajo pesado.

### 1. Entrar en el shell con nix-update

```bash
nix-shell -p nix-update
```

Entramos en un shell temporal de Nix que tiene la herramienta `nix-update` disponible. No hace falta instalar nada en el sistema, se descarga y se usa.

### 2. Actualizar el paquete

```bash
nix-update sail
```

Este es el comando que hace la magia. `nix-update` automáticamente:
- Detecta la última versión en el repositorio upstream
- Actualiza el hash del source
- Actualiza el `cargoHash` (dependencias de Rust)
- Modifica el archivo `default.nix` con la nueva versión

### 3. Compilar y verificar

```bash
nix-build -A sail
```

Compilamos el paquete con la nueva versión para asegurarnos de que todo funciona.

### 4. Comprobar la versión

```bash
./result/bin/sail --version
```

Si la versión es la correcta, solo queda hacer el PR y listo.

---

## 🔄 El Flujo Completo

```
Nueva versión de Sail publicada
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

De una tarea que la primera vez llevó días, a algo que se hace en **minutos**.

---

## 💭 Reflexión

Contribuir a nixpkgs parecía algo lejano cuando escribí aquel post. Pero al final, el paso más difícil es el primero. Una vez dentro, mantener un paquete es sencillo y gratificante.

Y siendo sincero: Sail me encanta. Viniendo del mundo Spark, donde conozco sus optimizaciones, sus internals y sus limitaciones, ver cómo Sail reimplementa todo eso en Rust con DataFusion es fascinante. No es solo usarlo, es poder entender cómo funciona por dentro y, con el tiempo, poder ayudar a implementar cosas que ya conozco del ecosistema Spark.

Como decía Marco Aurelio: _"Lo que se interpone en el camino se convierte en el camino."_ La dificultad del primer PR fue precisamente lo que me llevó a ser maintainer.

Si usas Nix y hay un paquete que te gustaría ver en nixpkgs... anímate a contribuir. El primer PR cuesta, pero los siguientes son solo unos pocos comandos. 🚀

---

## 📚 Recursos

- [Sail en nixpkgs](https://github.com/NixOS/nixpkgs/tree/master/pkgs/by-name/sa/sail)
- [nix-update](https://github.com/Mic92/nix-update)
- [Guía de contribución a nixpkgs](https://github.com/NixOS/nixpkgs/blob/master/CONTRIBUTING.md)
- [Posts anteriores: Nix + Sail](0002-nix-sail.md) | [Template](0003-template-nix-sail.md)
