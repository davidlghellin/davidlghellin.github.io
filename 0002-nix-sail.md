---
layout: default
title: "Nix + Sail"
date: 2025-10-08
categories: Blog
---

# 🧠 Experimentando con Sail y Nix: entornos reproducibles para Data y Rust

<p align="center">
  <img src="assets/img/nix_sail.png" alt="Nix + Sail Logo" width="320"/>
</p>

En este trabajo hemos estado integrando **Sail** —el motor de ejecución inspirado en Spark, desarrollado en Rust sobre DataFusion— con **Nix**, para facilitar la creación de entornos de desarrollo completamente reproducibles.

La idea es sencilla: poder _“jugar” con Sail en local_ sin tener que montar entornos virtuales, manejar dependencias a mano o lidiar con incompatibilidades de versiones.

---

## 🚀 ¿Por qué Nix?

Porque Nix nos permite definir el **entorno completo como código**.  
Con un solo comando (`nix develop`), tienes Rust, Sail, DataFusion, Protobuf y todas las dependencias necesarias configuradas de forma idéntica en cualquier máquina.

Esto significa:

- Sin `virtualenvs` ni `pip install`.
- Sin conflictos entre versiones de crates o librerías del sistema.
- Sin “en mi máquina sí funciona”.

Y lo mejor: puedes usarlo tanto para _desarrollar Sail_ como para _ejecutar tus propios planes de prueba o prototipos locales_ sin depender de entornos externos.

> 💡 **Nota:** Actualmente, **Sail** y **PySail** no se encuentran aún en los repositorios oficiales de Nix.  
> Por eso, el entorno instala manualmente las dependencias necesarias.  
> En el futuro, si tengo tiempo, me gustaría contribuir al empaquetado de `pysail` para Nix.  
> Si alguien tiene experiencia o interés en ayudar con esto, ¡sería genial colaborar! 😄

---

## 🧩 ¿Qué incluyen estos dos flakes?

El proyecto está dividido en dos entornos reproducibles:

- **Servidor (sail-server)** → levanta un _Spark Connect server_ con Sail.
- **Cliente (sail-client)** → un entorno mínimo con PySpark y un ejemplo de conexión.

Además, el cliente incluye un pequeño ejemplo en PySpark que se conecta al servidor y ejecuta una consulta.

En otras palabras, un entorno completo para **seguir jugando y explorando** 😄

---

## 🧭 Cómo probarlo

Puedes encontrar los flakes en mi repositorio:  
👉 [https://github.com/davidlghellin/nix-os/tree/master/flakes/sail](https://github.com/davidlghellin/nix-os/tree/master/flakes/sail)

### 🧱 Clonar el proyecto

```bash
git clone https://github.com/davidlghellin/nix-os.git
cd flakes/sail
```

Y tendremos un server y un cliente con un ejemplo básico.

```bash
cd sail-server
nix develop
# Levanta el servidor
launch-sail-server
# o modo debug:
debug-sail-server
```

```bash
cd sail-client
nix develop
# Test rápido
spark-connect-test
```

---

## 🌍 Reflexión final

Este experimento combina lo mejor de ambos mundos:
la reproducibilidad declarativa de Nix con la potencia de Sail y Spark Connect.

Gracias a Nix, podemos tener entornos completamente aislados, reproducibles y listos para usar con un solo comando, sin depender de configuraciones locales ni versiones globales.
