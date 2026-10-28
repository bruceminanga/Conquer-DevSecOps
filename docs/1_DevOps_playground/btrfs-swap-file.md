# 🚀 Creating a Swap File on BTRFS? One Crucial Step You Can't Afford to Miss 💡

---

## 🔥 The Problem

My 8GB Fedora machine kept getting hit by **Out of Memory (OOM) kills** while running heavy Minikube workloads.

Fedora ships with `zram` enabled by default. It's incredibly fast because it operates directly in RAM, but it has limits under sustained memory pressure. To stabilize the system, I added a traditional **disk-based swap file** as an overflow safety net.

On BTRFS, though, there's one critical requirement:

> ⚠️ **You MUST disable Copy-on-Write (CoW) on the swap file BEFORE writing any data to it.**

---

## 🛠️ The Correct Process

### 1️⃣ Create an empty swap file

```bash
sudo touch /swapfile
```

### 2️⃣ Restrict permissions

Only root should have access.

```bash
sudo chmod 600 /swapfile
```

### 3️⃣ Disable Copy-on-Write (CoW)

**This is the important BTRFS-specific step.**

```bash
sudo chattr +C /swapfile
```

### 4️⃣ Allocate the swap space

8GB example:

```bash
sudo dd if=/dev/zero of=/swapfile bs=1G count=8 status=progress
```

### 5️⃣ Format and enable the swap

```bash
sudo mkswap /swapfile
sudo swapon /swapfile
```

### 6️⃣ Make it persistent (optional)

```bash
echo '/swapfile none swap defaults,pri=-1 0 0' | sudo tee -a /etc/fstab
```

---

## ✅ Verify It Worked

Check that the CoW flag was applied:

```bash
lsattr /swapfile
```

You should see a capital `C`:

```text
---------------C------
```

---

> 💡 **Pro-tip:** Run `swapon --show` to confirm both swap devices are active and see their priorities.

---

## ⚖️ The Result: A Two-Tier Swap System

| Tier | Type            | Priority | Role                                   |
| ---- | --------------- | -------- | -------------------------------------- |
| ⚡ 1  | `zram`          | 100      | Fast everyday swapping in RAM          |
| 💽 2  | BTRFS swap file | -1       | Quietly catches heavy overflow on disk |

**Outcome:**

- ✅ Memory pressure dropped instantly
- ✅ OOM kills stopped
- ✅ Minikube workloads became far more stable

