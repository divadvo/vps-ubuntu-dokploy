# Publishing & Running This Fork

This repo is `github.com/divadvo/vps-ubuntu-dokploy` — a fork of the upstream VPS hardening
scripts, adjusted for **Ubuntu 26.04 LTS ("Resolute Raccoon")** (it still runs on 24.04 LTS).

This document covers how to publish your changes to GitHub and how to run the scripts on a
server safely.

---

## 1. Publish your changes to GitHub first

`setup.sh`, when installed via the one-liner, downloads the other scripts
(`install-dokploy.sh`, `check.sh`, etc.) from **your repo's `main` branch at runtime**. So any
edit you make locally must be pushed to GitHub *before* it can be used by the curl install
path.

```bash
git add -A
git commit -m "Re-brand to divadvo/vps-ubuntu-dokploy + support Ubuntu 26.04"
git push origin main
```

Verify the raw URL that `setup.sh` uses is live (should print the script, not "404"):

```bash
curl -sSL https://raw.githubusercontent.com/divadvo/vps-ubuntu-dokploy/main/setup.sh | head
```

---

## 2. Run it on the server

### Recommended: git-clone on the server

This is the most robust and verifiable method. When the full set of scripts is present
locally, `setup.sh` uses them directly and **skips the runtime download entirely** — no
dependency on the raw GitHub URLs or `SHA256SUMS`, and you can even run un-pushed edits.

```bash
sudo -i
git clone https://github.com/divadvo/vps-ubuntu-dokploy.git
cd vps-ubuntu-dokploy
./setup.sh
```

### Alternative: curl one-liner

Convenient, but requires you to have pushed first (Step 1). `setup.sh` downloads the six
post-install scripts from your `main` branch and verifies them against `SHA256SUMS`.

```bash
sudo -i
curl -sSL https://raw.githubusercontent.com/divadvo/vps-ubuntu-dokploy/main/setup.sh -o setup.sh && chmod +x setup.sh && ./setup.sh
```

---

## 3. What to expect while it runs

- **It survives SSH drops.** `setup.sh` relaunches itself inside a `screen` session named
  `hardening`. If your connection drops mid-run, reconnect and resume with
  `sudo screen -r hardening`.
- **Phase 1 asks all questions up front** (SSH port, hostname, admin user, password, SSH
  key). Nothing is changed on the system until you've answered everything.
- **You must confirm the new SSH port.** After hardening, open a **new** SSH session on the
  new port and type `CONFIRM` when prompted. Only then is port 22 closed and password auth
  disabled.
- **Don't skip the confirmation.** If you never confirm, a scheduled `at` job auto-closes
  port 22 and disables password auth after **24 hours** as a safety fallback — so a botched
  run can't leave the box wide open, but it also can't lock you out before you've verified
  key login works.
- **A reboot happens after CONFIRM** so `ssh.socket` takes over the new port.
- **Then install Docker + Dokploy (optional):**
  ```bash
  cd ~/vps-hardening/
  sudo ./install-dokploy.sh
  ```
- **Audit anytime:** `sudo ~/vps-hardening/check.sh` prints a PASS/FAIL/WARN report.

---

## 4. If you edit the post-install scripts, regenerate checksums

`SHA256SUMS` covers the six scripts that the curl path downloads. If you change any of them,
regenerate it and push, or the curl install will print a checksum-mismatch warning:

```bash
sha256sum cleanup.sh check.sh purge.sh install-dokploy.sh allow-docker-port.sh remove-docker-port.sh > SHA256SUMS
git add SHA256SUMS && git commit -m "Update SHA256SUMS" && git push
```

(The git-clone method doesn't use `SHA256SUMS`, so this only matters for the curl path.)

---

## 5. Ubuntu 26.04 caveat — third-party apt repos

The scripts derive the release codename dynamically, so they adapt to 26.04 automatically —
**provided the upstream apt repos publish a `resolute` pool.** Two installs depend on this:

- **Docker** — `download.docker.com/linux/ubuntu` (in `install-dokploy.sh`)
- **Cloudflared** — `pkg.cloudflare.com/cloudflared` (optional, in `setup.sh`)

On a brand-new LTS these can lag. If `apt update` errors on either repo, that's why. Test the
full flow on a disposable 26.04 VPS before relying on it. If Docker's `resolute` pool isn't up
yet, the workaround is to point that one apt source at the previous LTS codename (`noble`)
until it publishes.
