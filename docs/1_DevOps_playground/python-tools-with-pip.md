# 🐍 Stop installing every Python tool with `pip`.

> **Not every Python package belongs in your system Python environment.**

---

## 🤔 The Problem

Installing CLI tools globally with `pip` pollutes your system Python, creates version conflicts, and can break packages your operating system depends on.

---

## 🧭 The Rule: Pick the Right Tool for the Job

| Tool   | Use it for                         | Where it installs                      |
| ------ | ---------------------------------- | -------------------------------------- |
| `dnf`  | Operating system packages          | System-wide                            |
| `pip`  | Project dependencies               | Inside a virtual environment           |
| `pipx` | Standalone Python CLI applications | Isolated environment, globally exposed |

---

## ⚙️ Why `pipx`?

`pipx` installs each CLI application into **its own isolated virtual environment** while exposing the executable globally.

- ✅ Prevents dependency pollution
- ✅ Avoids conflicts with OS-managed packages
- ✅ Keeps developer tooling isolated and reproducible
- ✅ Maintains a clean, predictable Python environment

---

## 💻 Example

Instead of installing `awscli-local` with `pip`:

```bash
# Install pipx
sudo dnf install pipx

# Install the CLI in its own isolated environment
pipx install awscli-local
```

---

## 📦 The Principle

- **Operating system packages** belong to the operating system.
- **Project dependencies** belong to virtual environments.
- **Developer tools** belong to `pipx`.

---

> 💡 Small architectural decisions like this compound over time, resulting in cleaner, more maintainable development environments.