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

## 🐍 Actualizar también pysail

Sail no es solo el CLI en Rust: también empaqueté **`pysail`**, los bindings de Python. Vive en otra parte del árbol de nixpkgs (`pkgs/development/python-modules/pysail`) y se expone bajo el _package set_ de Python, así que el atributo lleva prefijo: `python3Packages.pysail`.

El flujo es el mismo, cambiando el atributo:

```bash
nix-update python3Packages.pysail
```

`nix-update` actualiza aquí un poco más de lo habitual. `pysail` se construye con **maturin** (compila la parte Rust y la expone a Python vía PyO3), así que además del hash del source refresca el hash del _vendor_ de Cargo (`cargoDeps` / `fetchCargoVendor`). Un solo comando y quedan los dos sincronizados.

Y para compilar:

```bash
nix-build -A python3Packages.pysail
```

### Cómo comprobar que funciona

Aquí está la diferencia con el CLI: `pysail` no es un binario que ejecutas con `--version`, es un **módulo de Python**. Lo bueno es que la propia derivación ya se autocomprueba. En el `default.nix` hay un `pythonImportsCheck`:

```nix
pythonImportsCheck = [
  "pysail"
  "pysail._native"
];
```

Eso significa que si `nix-build` **termina sin error, el módulo importa de verdad** — incluida la extensión nativa `pysail._native`, que es la parte Rust compilada. Un build verde ya te dice que los bindings cargan. De hecho, al final del log verás la fase que lo confirma:

```
Running phase: pythonImportsCheckPhase
Check whether the following modules can be imported: pysail pysail._native
```

### Cómo testearlo

La derivación trae un test ligero de versión en `passthru.tests`, que puedes construir directamente:

```bash
nix-build -A python3Packages.pysail.tests.version
```

Que imprime la versión reportada por el paquete y confirma que coincide:

```
sail 0.6.6
```

Lo que **no** corre en el build es la suite completa: está desactivada a propósito (`doCheck = false`) porque necesita un servidor Spark Connect levantado y un montón de dependencias opcionales pesadas (`pyspark-client`, `duckdb`…). Para nixpkgs, con que el módulo importe y la versión sea correcta es suficiente; probar el motor entero es cosa del upstream, no del empaquetado.

```
Nueva versión de Sail publicada
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

## 🔍 Revisar una PR con nixpkgs-review

Ser maintainer no es solo subir versiones: también **revisas PRs** — las tuyas antes de abrirlas, y las de otros para ayudar a que se mergeen. La herramienta estándar para esto es [`nixpkgs-review`](https://github.com/Mic92/nixpkgs-review). Hace fetch del cambio a un **worktree aislado**, compila solo los paquetes afectados (o los baja de la caché binaria si ya están), y al terminar te abre una **shell** con ellos construidos para que los pruebes.

Yo normalmente **no la tengo instalada** en el sistema, así que la traigo al vuelo con `nix-shell -p nixpkgs-review` y la ejecuto dentro del mismo comando. Igual que con `nix-update`, no ensucias nada permanente. (Si prefieres, `nix run nixpkgs#nixpkgs-review -- <args>` hace lo mismo.)

Hay tres formas de usarla según qué estés revisando.

### 1. Una PR de GitHub (de otra persona)

Primero comprueba el **coste sin compilar nada**:

```bash
nix-shell -p nixpkgs-review --run "nixpkgs-review pr 539480 --dry-run"
```

Si en el `--dry-run` casi todo son descargas (`↓` desde la caché) y pocos builds locales, adelante. Si son muchos builds pesados, quizá lo dejes para otro momento. Cuando decidas revisarla de verdad:

```bash
nix-shell -p nixpkgs-review --run "nixpkgs-review pr 539480"
```

### 2. Tus cambios **sin commitear** (antes de abrir la PR)

Esto es lo que uso justo después del `nix-update`, para probar el bump antes de commitear nada. Compara tu árbol de trabajo (working tree) contra la base:

```bash
nix-shell -p nixpkgs-review --run "nixpkgs-review wip"
```

### 3. Un commit concreto o tu rama

Si ya has commiteado (o quieres revisar lo que hay encima de `master`):

```bash
nix-shell -p nixpkgs-review --run "nixpkgs-review rev HEAD"
```

### Validar en la shell

Al terminar de compilar, `nixpkgs-review` **te deja dentro de una shell** con todos los paquetes construidos en el `PATH`. Ahí es donde compruebas que funcionan de verdad. Un PR que sube `sail` y `pysail` a la vez trae ambos, y cada uno se valida distinto:

```bash
# sail: es un binario
sail --version                          # → sail 0.6.6

# pysail: es un módulo de Python
python3 -c "import pysail; print('ok')"
```

### La salida que se sube a la PR

Además de la shell, `nixpkgs-review` deja un **`report.md`** en `~/.cache/nixpkgs-review/pr-<N>/`. Esta es la salida **real** de revisar el PR [539480](https://github.com/NixOS/nixpkgs/pull/539480), que bumpea `sail` y `pysail` de `0.6.5` a `0.6.6` en un solo cambio:

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

De un solo `default.nix` de `pysail` salen **cuatro** entradas: nixpkgs lo instancia para **cada intérprete soportado** (3.13 y 3.14), y cada uno arrastra su `.dist` (el sdist/wheel). Más `sail`, el CLI en Rust → 5 en total.

### Cómo lo pego en el PR

Tres formas de sacar ese reporte para pegarlo:

```bash
# 1. copiar directo al portapapeles (macOS)
pbcopy < ~/.cache/nixpkgs-review/pr-539480/report.md

# 2. que nixpkgs-review lo imprima al terminar
nix-shell -p nixpkgs-review --run "nixpkgs-review pr 539480 --print-result"

# 3. que lo publique como comentario en el PR (requiere token de GitHub: gh auth login)
nix-shell -p nixpkgs-review --run "nixpkgs-review pr 539480 --post-result"
```

Dos detalles que aprendí a las malas:

- **Pégalo sin envolver en backticks**, para que GitHub renderice el `<details>` y los check verdes.
- **Di en qué plataforma lo probaste** (`aarch64-darwin` en mi Mac, por ejemplo). No afirmes plataformas que no tienes — de las otras se encargan ofborg y otros revisores.

Un mensaje de review típico:

```
Result of `nixpkgs-review pr 539480` run on aarch64-darwin — builds and tests pass. LGTM 👍
```

---

## 💭 Reflexión

Contribuir a nixpkgs parecía algo lejano cuando escribí aquel post. Pero al final, el paso más difícil es el primero. Una vez dentro, mantener un paquete es sencillo y gratificante.

Y siendo sincero: Sail me encanta. Viniendo del mundo Spark, donde conozco sus optimizaciones, sus internals y sus limitaciones, ver cómo Sail reimplementa todo eso en Rust con DataFusion es fascinante. No es solo usarlo, es poder entender cómo funciona por dentro y, con el tiempo, poder ayudar a implementar cosas que ya conozco del ecosistema Spark.

Como decía Marco Aurelio: _"Lo que se interpone en el camino se convierte en el camino."_ La dificultad del primer PR fue precisamente lo que me llevó a ser maintainer.

Si usas Nix y hay un paquete que te gustaría ver en nixpkgs... anímate a contribuir. El primer PR cuesta, pero los siguientes son solo unos pocos comandos. 🚀

---

## 📚 Recursos

- [Sail en nixpkgs](https://github.com/NixOS/nixpkgs/tree/master/pkgs/by-name/sa/sail)
- [pysail en nixpkgs](https://github.com/NixOS/nixpkgs/tree/master/pkgs/development/python-modules/pysail)
- [nix-update](https://github.com/Mic92/nix-update)
- [Guía de contribución a nixpkgs](https://github.com/NixOS/nixpkgs/blob/master/CONTRIBUTING.md)
- [Posts anteriores: Nix + Sail](0002-nix-sail.md) | [Template](0003-template-nix-sail.md)
