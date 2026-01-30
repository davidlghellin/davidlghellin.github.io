---
layout: default
title: "Template Nix Sail: Reproducible Development"
date: 2026-01-30
categories: Blog
---

# 🚀 Template Nix Sail: A Complete Reproducible Development Environment

<p align="center">
  <img src="assets/img/nix_sail.png" alt="Nix + Sail Logo" width="320"/>
</p>

In a [previous post](0002-nix-sail.md) we explored how to integrate Sail with Nix using two separate flakes (server and client). Now we take it further: a **unified template** that simplifies the entire process and adds modern development tools.

The goal is to have an environment where with a single command you have **everything ready to work** with PySpark or PySail, without manual configurations or global dependencies.

---

## 🎯 What is template-nix-sail?

It's a development template that combines:

- **Nix** for environment reproducibility
- **Sail/PySail** as an alternative Spark engine (written in Rust, no Java needed)
- **PySpark** traditional as an alternative option
- **Modern tooling**: pytest, ruff, ptpython

Everything configured to work automatically when you enter the directory.

---

## 🐳 Why Nix Instead of Docker?

A common question: why not use Docker for this? Both solve the reproducibility problem, but with different approaches.

| Aspect | Nix | Docker |
|--------|-----|--------|
| **Execution** | Native on your system | Inside container |
| **I/O Performance** | No overhead | Bind mounts slow (especially on macOS) |
| **Editor/IDE** | Works directly | Need devcontainers or extra config |
| **Cache** | Per individual package | Per image layer |
| **Daemon** | Not needed | Requires Docker daemon running |
| **Shell/aliases** | Your normal config | Separate config inside container |
| **Disk space** | Only what's needed | Base images + layers |
| **File permissions** | No issues | UID/GID can cause problems |

**In summary**: Nix is lighter and more natural for local development. Your editor, terminal, and tools work without additional configuration. Docker is still excellent for deployment and when you need complete isolation.

---

## 🧩 Project Structure

```
template-nix-sail/
├── flake.nix           # Nix shell definitions
├── flake.lock          # Dependencies lockfile
├── .envrc              # direnv configuration
├── pyproject.toml      # Python project configuration
├── .env                # Environment variables
│
├── src/                # Source code
│   ├── main.py         # Interactive demo
│   ├── calculator.py   # Example functions
│   └── dataframes.py   # DataFrame operations
│
├── tests/              # Test suite
│   ├── conftest.py     # pytest fixtures
│   └── test_*.py       # Tests
│
└── resources/
    └── ciudades_espana.csv  # Sample dataset
```

---

## 🔧 Two Development Shells

The `flake.nix` defines two environments based on your needs:

| Shell | Command | Java | Use Case |
|-------|---------|------|----------|
| **default** | `nix develop` | Yes (JDK 17) | Traditional PySpark + PySail |
| **pysail** | `nix develop .#pysail` | No | PySail only (lighter) |

### When to use each one?

- **Default shell**: when you need full compatibility with traditional PySpark or want to test both backends.
- **Pysail shell**: for fast development without Java overhead. Ideal for prototypes and learning.

---

## ⚡ Getting Started

### 1. Clone the template

```bash
git clone https://github.com/davidlghellin/template-nix-sail.git
cd template-nix-sail
```

### 2. Activate the environment

```bash
nix develop
```

Or if you use **direnv** (recommended), the environment activates automatically when entering the directory.

### 3. Ready!

When activating the shell, Nix:
1. Sets up Python 3.12 and JDK 17 (if applicable)
2. Creates a `.venv-nix` virtualenv
3. Installs all dependencies automatically
4. Defines useful aliases

You'll see a sailboat animation (⛵) while dependencies are being installed.

---

## 🛠️ Available Aliases

Once inside the environment, you have these shortcuts:

| Alias | Command | Description |
|-------|---------|-------------|
| `t` | `pytest -v` | Run tests |
| `ts` | `SPARK_BACKEND=pysail pytest -v` | Tests with PySail |
| `tp` | `SPARK_BACKEND=pyspark pytest -v` | Tests with PySpark |
| `r` | `ruff check .` | Check linting |
| `rf` | `ruff check --fix . && ruff format .` | Fix and format |

---

## 🎮 Included Demo

The template includes a functional demo with a dataset of 100 Spanish cities:

```bash
python src/main.py
```

The demo shows:
- Reading CSV with Spark
- Top 10 most populated cities
- Population grouped by autonomous community
- Population density calculation

All using PySail by default (no Java needed).

---

## 🔀 Backend Selection

The `SPARK_BACKEND` variable controls which engine to use:

```bash
# PySail (default) - No Java, faster
SPARK_BACKEND=pysail python src/main.py

# Traditional PySpark - With Java/JVM
SPARK_BACKEND=pyspark python src/main.py
```

The code automatically detects if a Spark Connect server is available. If not found, it starts an internal one.

---

## 🧪 Integrated Testing

The project includes pytest tests that work with both backends:

```bash
# Tests with default backend (PySail)
pytest -v

# Unit tests only
pytest -m unit -v

# Tests forcing PySpark
SPARK_BACKEND=pyspark pytest -v
```

The fixtures in `conftest.py` automatically handle creation and cleanup of Spark sessions.

---

## 📝 Enhanced REPL with ptpython

The template includes **ptpython**, an enhanced Python REPL with:

- Fuzzy autocompletion (Tab)
- History search (Ctrl+R)
- Syntax highlighting
- Auto-suggestions

```bash
ptpython
```

```python
>>> from pyspark.sql import SparkSession
>>> spark = SparkSession.builder.remote("sc://localhost:50051").getOrCreate()
>>> spark.sql("SELECT 1 + 1").show()
```

---

## 🌟 Advantages of This Approach

Compared to the previous server/client separate setup:

1. **Single project**: Everything in one place, easier to maintain
2. **Two backends**: Flexibility to use PySail or PySpark as needed
3. **Auto-server**: No need to manually start a server
4. **Modern tooling**: pytest, ruff, ptpython already configured
5. **Functional demo**: Real example code, not just a "hello world"
6. **Direnv**: Automatic environment activation

---

## 🚧 Next Steps

This template is a starting point. Some ideas to expand it:

- Add more sample datasets
- Integrate notebooks with Jupyter
- Create more fixtures for testing
- Document common Spark patterns

---

## 📚 Resources

- [Template on GitHub](https://github.com/davidlghellin/template-nix-sail)
- [Sail Documentation](https://github.com/lakehq/sail)
- [Nix Flakes](https://nixos.wiki/wiki/Flakes)
- [Previous post: Nix + Sail](0002-nix-sail.md)

---

## 💭 Final Thoughts

With Nix and this template, setting up a Spark development environment reduces to a single command. No more "works on my machine", no more dependency conflicts.

It's the perfect combination for learning, experimenting, and prototyping with Spark in a reproducible way.

Hope you find it useful! If you have ideas or improvements, contributions are welcome. 🚀
