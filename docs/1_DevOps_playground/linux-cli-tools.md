# 📦 Need a CLI Tool That Isn't in Your Distro's Repos? Here's What I Do

---

## 🤔 The Problem

Most engineers default to one of three compromises:

1. **Heavy runtimes and compilers**, installed just to build one tool (hello, workstation bloat)
2. **Untracked binaries**, curled into random directories
3. **Long container commands**, typed out by hand every single time

---

## 🔧 The Situation

I hit this on Fedora when I needed `kube-score` to lint Kubernetes manifests. It isn't packaged in the upstream `dnf` repos.

Instead of polluting my host or wrestling with dependencies, I used a cleaner pattern: an **ephemeral container shim**.

---

## 💡 The Idea

Combine three things:

- **FHS**: `/usr/local/bin` is the standard home for locally installed executables
- **`$PATH` resolution**: your shell finds the script like any other command
- **Rootless Podman**: the tool runs in a throwaway container, with no root required

A tiny script turns a container image into a native-feeling CLI.

---

## 📄 The Shim

**`/usr/local/bin/kube-score`**

```bash
#!/usr/bin/env bash
exec podman run --rm -i \
  --userns=keep-id \
  -v "$PWD:$PWD:z" \
  -w "$PWD" \
  docker.io/zegl/kube-score:v1.18.0 "$@"
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/kube-score
```

### What each flag does

| Flag               | Purpose                                                                 |
| ------------------ | ----------------------------------------------------------------------- |
| `--rm`             | Deletes the container after it exits                                    |
| `-i`               | Keeps stdin open, so piping works                                       |
| `--userns=keep-id` | Maps your UID into the container, so file ownership stays correct       |
| `-v "$PWD:$PWD:z"` | Mounts your current directory at the same path, with an SELinux relabel |
| `-w "$PWD"`        | Sets the working directory, so relative paths just work                 |
| `"$@"`             | Forwards all your arguments to the tool                                 |

---

## ✅ The Result

```bash
kube-score score k8s/*.yaml
```

It feels **100% native**, as if it came from the official package manager. The version is pinned in the script, and nothing is installed on the host.

