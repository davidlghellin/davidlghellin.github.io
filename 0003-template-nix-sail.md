---
layout: default
title: "Template Nix Sail: Desarrollo Reproducible"
date: 2026-01-30
categories: Blog
---

# 🚀 Template Nix Sail: Un Entorno de Desarrollo Completo y Reproducible

<p align="center">
  <img src="assets/img/nix_sail.png" alt="Nix + Sail Logo" width="320"/>
</p>

En el [post anterior](0002-nix-sail.md) exploramos cómo integrar Sail con Nix usando dos flakes separados (servidor y cliente). Ahora damos un paso más: un **template unificado** que simplifica todo el proceso y añade herramientas modernas de desarrollo.

La idea es tener un entorno donde con un solo comando tengas **todo listo para trabajar** con PySpark o PySail, sin configuraciones manuales ni dependencias globales.

---

## 🎯 ¿Qué es template-nix-sail?

Es un template de desarrollo que combina:

- **Nix** para la reproducibilidad del entorno
- **Sail/PySail** como motor Spark alternativo (escrito en Rust, sin Java)
- **PySpark** tradicional como opción alternativa
- **Herramientas modernas**: pytest, ruff, ptpython

Todo configurado para que funcione automáticamente al entrar al directorio.

---

## 🐳 ¿Por qué Nix en lugar de Docker?

Una pregunta común: ¿por qué no usar Docker para esto? Ambos resuelven el problema de reproducibilidad, pero con enfoques diferentes.

| Aspecto | Nix | Docker |
|---------|-----|--------|
| **Ejecución** | Nativa en tu sistema | Dentro de contenedor |
| **Rendimiento I/O** | Sin overhead | Bind mounts lentos (especialmente en macOS) |
| **Editor/IDE** | Funciona directamente | Necesitas devcontainers o configuración extra |
| **Cache** | Por paquete individual | Por capa de imagen |
| **Daemon** | No necesita | Requiere Docker daemon corriendo |
| **Shell/aliases** | Tu configuración normal | Configuración separada dentro del contenedor |
| **Espacio en disco** | Solo lo necesario | Imágenes base + capas |
| **Permisos de archivos** | Sin problemas | UID/GID pueden dar problemas |

**En resumen**: Nix es más ligero y natural para desarrollo local. Tu editor, terminal y herramientas funcionan sin configuración adicional. Docker sigue siendo excelente para despliegue y cuando necesitas aislamiento completo.

---

## 🧩 Estructura del Proyecto

```
template-nix-sail/
├── flake.nix           # Definición de los shells de Nix
├── flake.lock          # Lockfile de dependencias
├── .envrc              # Configuración de direnv
├── pyproject.toml      # Configuración del proyecto Python
├── .env                # Variables de entorno
│
├── src/                # Código fuente
│   ├── main.py         # Demo interactiva
│   ├── calculator.py   # Funciones de ejemplo
│   └── dataframes.py   # Operaciones con DataFrames
│
├── tests/              # Suite de testing
│   ├── conftest.py     # Fixtures de pytest
│   └── test_*.py       # Tests
│
└── resources/
    └── ciudades_espana.csv  # Dataset de ejemplo
```

---

## 🔧 Dos Shells de Desarrollo

El `flake.nix` define dos entornos según tus necesidades:

| Shell | Comando | Java | Uso |
|-------|---------|------|-----|
| **default** | `nix develop` | Sí (JDK 17) | PySpark tradicional + PySail |
| **pysail** | `nix develop .#pysail` | No | Solo PySail (más ligero) |

### ¿Cuándo usar cada uno?

- **Shell default**: cuando necesitas compatibilidad total con PySpark tradicional o quieres probar ambos backends.
- **Shell pysail**: para desarrollo rápido sin la sobrecarga de Java. Ideal para prototipos y aprendizaje.

---

## ⚡ Cómo Empezar

### 1. Clonar el template

```bash
git clone https://github.com/davidlghellin/template-nix-sail.git
cd template-nix-sail
```

### 2. Activar el entorno

```bash
nix develop
```

O si usas **direnv** (recomendado), el entorno se activa automáticamente al entrar al directorio.

### 3. ¡Listo!

Al activar el shell, Nix:
1. Configura Python 3.12 y JDK 17 (si aplica)
2. Crea un virtualenv `.venv-nix`
3. Instala todas las dependencias automáticamente
4. Define aliases útiles

Verás una animación de velero (⛵) mientras se instalan las dependencias.

---

## 🛠️ Aliases Disponibles

Una vez dentro del entorno, tienes estos atajos:

| Alias | Comando | Descripción |
|-------|---------|-------------|
| `t` | `pytest -v` | Ejecutar tests |
| `ts` | `SPARK_BACKEND=pysail pytest -v` | Tests con PySail |
| `tp` | `SPARK_BACKEND=pyspark pytest -v` | Tests con PySpark |
| `r` | `ruff check .` | Verificar linting |
| `rf` | `ruff check --fix . && ruff format .` | Fix y formatear |

---

## 🎮 Demo Incluida

El template incluye una demo funcional con un dataset de 100 ciudades españolas:

```bash
python src/main.py
```

La demo muestra:
- Lectura del CSV con Spark
- Top 10 ciudades más pobladas
- Población agrupada por comunidad autónoma
- Cálculo de densidad de población

Todo usando PySail por defecto (sin necesidad de Java).

---

## 🔀 Selección de Backend

La variable `SPARK_BACKEND` controla qué motor usar:

```bash
# PySail (default) - Sin Java, más rápido
SPARK_BACKEND=pysail python src/main.py

# PySpark tradicional - Con Java/JVM
SPARK_BACKEND=pyspark python src/main.py
```

El código detecta automáticamente si hay un servidor Spark Connect disponible. Si no lo encuentra, inicia uno interno.

---

## 🧪 Testing Integrado

El proyecto incluye tests con pytest que funcionan con ambos backends:

```bash
# Tests con backend por defecto (PySail)
pytest -v

# Solo tests unitarios
pytest -m unit -v

# Tests forzando PySpark
SPARK_BACKEND=pyspark pytest -v
```

Los fixtures en `conftest.py` manejan automáticamente la creación y limpieza de sesiones Spark.

---

## 📝 REPL Mejorada con ptpython

El template incluye **ptpython**, una REPL de Python mejorada con:

- Autocompletado fuzzy (Tab)
- Búsqueda en historial (Ctrl+R)
- Syntax highlighting
- Auto-sugerencias

```bash
ptpython
```

```python
>>> from pyspark.sql import SparkSession
>>> spark = SparkSession.builder.remote("sc://localhost:50051").getOrCreate()
>>> spark.sql("SELECT 1 + 1").show()
```

---

## 🌟 Ventajas de Este Enfoque

Comparado con el setup anterior de servidor/cliente separados:

1. **Un solo proyecto**: Todo en un mismo lugar, más fácil de mantener
2. **Dos backends**: Flexibilidad para usar PySail o PySpark según necesidad
3. **Auto-servidor**: No necesitas levantar servidor manualmente
4. **Tooling moderno**: pytest, ruff, ptpython ya configurados
5. **Demo funcional**: Código de ejemplo real, no solo un "hello world"
6. **Direnv**: Activación automática del entorno

---

## 🚧 Próximos Pasos

Este template es un punto de partida. Algunas ideas para expandirlo:

- Añadir más datasets de ejemplo
- Integrar notebooks con Jupyter
- Crear más fixtures para testing
- Documentar patrones comunes de Spark

---

## 📚 Recursos

- [Template en GitHub](https://github.com/davidlghellin/template-nix-sail)
- [Documentación de Sail](https://github.com/lakehq/sail)
- [Nix Flakes](https://nixos.wiki/wiki/Flakes)
- [Post anterior: Nix + Sail](0002-nix-sail.md)

---

## 💭 Reflexión

Con Nix y este template, el setup de un entorno de desarrollo Spark se reduce a un comando. No más "funciona en mi máquina", no más conflictos de dependencias.

Es la combinación perfecta para aprender, experimentar y prototipar con Spark de forma reproducible.

¡Espero que te sea útil! Si tienes ideas o mejoras, las contribuciones son bienvenidas. 🚀
