# GLPI New Machine Setup

**Instructions for GitHub Copilot Agent:** Execute each step below in order using the terminal. Confirm each step succeeds before proceeding to the next. If a step fails, stop and report the error.

---

## Prerequisites

Docker Desktop must already be installed and running. Verify:

```bash
docker info
```

Install Homebrew if not present:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Install required tools:

```bash
brew install git make mkcert node@22
brew link node@22
```

Verify versions:

```bash
git --version
gmake --version
mkcert --version
node --version   # must be 22.x
```

---

## Step 1 — Clone the Fork

Clone the repo into whatever directory you prefer, then `cd` into it:

```bash
git clone https://github.com/shanel-svi/glpi.git
cd glpi
```

Now run `pwd` and note the absolute path — this is your repo root. Store it for use in later steps:

```bash
pwd
export REPO_ROOT=$(pwd)
echo "Repo root: $REPO_ROOT"
```

> **Copilot:** After running the above, confirm the value of `$REPO_ROOT` before continuing.

---

## Step 2 — Configure Git Remotes

Set `origin` to the fork and add `upstream` for the original GLPI project:

```bash
git remote set-url origin https://github.com/shanel-svi/glpi.git
git remote add upstream https://github.com/glpi-project/glpi.git
git remote -v
```

Expected output:
```
origin    https://github.com/shanel-svi/glpi.git (fetch)
origin    https://github.com/shanel-svi/glpi.git (push)
upstream  https://github.com/glpi-project/glpi.git (fetch)
upstream  https://github.com/glpi-project/glpi.git (push)
```

Set the local branch to track `origin`:

```bash
git branch --set-upstream-to=origin/11.0/bugfixes 11.0/bugfixes
git checkout -b local/infra-setup
git branch -vv
```

---

## Step 3 — Loopback Alias (macOS-specific)

All Docker port bindings use `127.0.0.2` instead of `127.0.0.1`. This requires a persistent loopback alias on `lo0`.

Copy the launchd plist from the repo and load it:

```bash
sudo cp "$REPO_ROOT/com.glpi.loopback-alias.plist" /Library/LaunchDaemons/com.glpi.loopback-alias.plist
sudo chown root:wheel /Library/LaunchDaemons/com.glpi.loopback-alias.plist
sudo launchctl load /Library/LaunchDaemons/com.glpi.loopback-alias.plist
```

Verify the alias is active:

```bash
ifconfig lo0 | grep "127.0.0.2"
```

Expected output: `inet 127.0.0.2 netmask 0xff000000`

---

## Step 4 — Local DNS (macOS-specific)

Add `itsm.shanel.com` to `/etc/hosts` so the hostname resolves to the loopback alias:

```bash
grep -q "itsm.shanel.com" /etc/hosts || echo "127.0.0.2 itsm.shanel.com" | sudo tee -a /etc/hosts
```

Verify:

```bash
ping -c 1 itsm.shanel.com
```

Expected: replies from `127.0.0.2`.

---

## Step 5 — TLS Certificate (macOS-specific)

The HTTPS vhost requires a certificate trusted by the macOS Keychain. Install the mkcert CA and generate the cert:

```bash
mkcert -install
mkdir -p "$REPO_ROOT/.docker/certs"
cd "$REPO_ROOT/.docker/certs"
mkcert itsm.shanel.com
```

Rename the generated files to match the filenames expected by the Apache SSL vhost:

```bash
mv itsm.shanel.com.pem itsm.shanel.com.pem
mv itsm.shanel.com-key.pem itsm.shanel.com-key.pem
ls -la
```

Expected files:
- `itsm.shanel.com.pem`
- `itsm.shanel.com-key.pem`

---

## Step 6 — Create Required Local Directories

```bash
mkdir -p "$REPO_ROOT/logs/php"
```

---

## Step 7 — Build and Install GLPI

This builds the custom ARM64 Docker image (`php:8.4-apache`), starts all containers, installs Composer and NPM dependencies, and initialises the database. It will take several minutes.

```bash
cd "$REPO_ROOT"
gmake install
```

If `gmake` is not found, use:

```bash
cd "$REPO_ROOT"
make install
```

---

## Step 8 — VS Code Extensions

Install the PHP Debug extension for Xdebug support. Run in the VS Code integrated terminal:

```bash
code --install-extension xdebug.php-debug
code --install-extension bmewburn.vscode-intelephense-client
```

The Xdebug launch configuration is already in `.vscode/launch.json` — it listens on port `9003` and maps `/var/www/glpi` → the local repo root.

---

## Step 9 — Verify the Installation

Check all containers are running:

```bash
docker compose ps
```

Expected: `glpi-app`, `glpi-db`, `glpi-mailpit`, `glpi-dbgate` all with status `Up`.

Open the following URLs in a browser and confirm they load:

| URL | Expected |
|---|---|
| `https://itsm.shanel.com` | GLPI login page, green padlock |
| `http://itsm.shanel.com:8080` | GLPI login page (testing env) |
| `http://itsm.shanel.com:8025` | Mailpit inbox |
| `http://itsm.shanel.com:9000` | DbGate database browser |

---

## Daily Workflow

Once installed, the standard commands are (always run from `$REPO_ROOT`):

```bash
gmake up      # start containers
gmake down    # stop containers (data preserved)
gmake bash    # shell into the app container
gmake cc      # clear GLPI cache
gmake build   # rebuild Docker image after config changes
```

Refer to `.github/copilot-instructions.md` in the repo for full coding conventions, git workflow, and architecture notes.
