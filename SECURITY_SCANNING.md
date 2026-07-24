# Food Gorilla — Security Scan: Team Onboarding & Installation Guide

**Owner:** Ryan · **Pipeline stage:** `Security Scan` (Jenkinsfile) · **Tools:** Trivy + OWASP Dependency-Check

This guide gets every teammate onto the **exact same scanner versions and
configuration**, locally and on the Jenkins server, so we don't get
"works on my machine" discrepancies or merge conflicts over tooling. Read
sections 1–2 (5 minutes), then do sections 3–4 once on your own machine.
Section 5 is the one-time Jenkins server setup. Section 6 is the day-one
baseline run we all do **before** the pipeline starts hard-failing builds.

---

## 1. What the Security Scan stage does, and why

The stage runs on **every push and every nightly timer trigger**, right
after `Integration Testing`. It answers a question tests can't:

> The app *works* — but is it shipping with known, publicly documented
> vulnerabilities we inherited from base images and third-party libraries?

That's **defense in depth**: "it works" is not "it's safe." Nobody is
going to hand-check every dependency against a CVE database on every
release, so the pipeline does it:

| Tool | What it scans | What it catches |
|---|---|---|
| **Trivy** | The Docker images the **`Integration Testing` stage just built** (`foodgorilla_test-backend:latest`, `foodgorilla_test-frontend:latest`) | Vulnerable OS packages in the base image + vulnerable libraries actually installed inside the container |
| **OWASP Dependency-Check (DC)** | The **source dependency manifests** in the repo (`backend/requirements.txt`, `frontend/package-lock.json`) | Third-party libraries with published CVEs, straight from the NVD database |

**Why scan the locally built images rather than pulling from a registry?**
Because this project has no registry to pull from. There is no Docker Hub
push stage in the `Jenkinsfile`, and `docker-compose.yml` declares no
`image:` keys — every service is build-only, so Compose names the images
locally and nothing is ever published. A registry-based scan could never
go green here.

Scanning the local build works because `Integration Testing` runs
immediately before this stage on **every** branch and **every** trigger,
and its teardown is `down -v --remove-orphans` **without `--rmi`** — so
the images it built are still on the Docker host when the scan runs.
Compose names build-only images `<project>-<service>`, and the test
project is `foodgorilla_test`, which is where those two image names come
from.

**The trade-off, stated plainly:** we give up "re-scan the artifact that
is actually published" and get "scan exactly what this commit builds."
The nightly run still does its job — a CVE published *tomorrow* against a
base image we built *today* is caught, because Integration Testing
rebuilds both images from the latest commit before the nightly scan runs,
and the scan checks them against a freshly updated CVE database. If the
team later adds a real Docker Hub push stage, revisit this: scanning the
published artifact as well would be a genuine improvement.

**Two-phase rollout (important):**

- **Phase 1 — report-only (where we are now).** Scanners run and produce
  full reports, but never fail the build. This is deliberate: day one, the
  images will contain pre-existing findings nobody has triaged, and
  hard-failing every teammate's push over old debt would block all 6 of us.
- **Phase 2 — enforcement.** After the baseline triage (section 6), we flip
  two values in the Jenkinsfile (section 7) and HIGH/CRITICAL findings
  start failing builds for real.

---

## 2. Tool versions we standardise on

Everyone installs **the same versions** — scanner output differs between
versions, and we don't want one person's "clean" being another person's
"3 criticals" in a PR argument.

| Tool | Version | Why pinned |
|---|---|---|
| Trivy | **v0.72.0** | Current release at time of writing |
| OWASP Dependency-Check | **v12.1.0** | Current release at time of writing |
| Java (DC prerequisite) | 11 or newer | DC is a Java tool; the Jenkins container already has JDK 17 |

If we ever bump a version, we bump it **here and on the Jenkins server in
the same PR** — never drift silently.

---

## 3. Install the tools locally

### 3.1 Trivy

**macOS (Homebrew):**

```bash
brew install trivy
trivy --version   # confirm it reports 0.72.x
```

**Linux / WSL (official install script, pinned version):**

```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
  | sudo sh -s -- -b /usr/local/bin v0.72.0
trivy --version
```

**Windows (native):** `choco install trivy` or `scoop install trivy`, or
download `trivy_0.72.0_windows-64bit.zip` from
<https://github.com/aquasecurity/trivy/releases/tag/v0.72.0> and put
`trivy.exe` on your `PATH`.

**One-time warm-up** (downloads Trivy's vulnerability DB, ~60 MB, so your
first real scan isn't slow):

```bash
trivy image --download-db-only
```

### 3.2 OWASP Dependency-Check

**Prerequisite — Java 11+:**

```bash
java -version   # need 11 or newer; install a JDK (e.g. Temurin) if missing
```

**Install (macOS / Linux / WSL):**

```bash
curl -L -o /tmp/dc.zip \
  https://github.com/jeremylong/DependencyCheck/releases/download/v12.1.0/dependency-check-12.1.0-release.zip
sudo unzip -q /tmp/dc.zip -d /opt && rm /tmp/dc.zip
sudo ln -sf /opt/dependency-check/bin/dependency-check.sh /usr/local/bin/dependency-check.sh
dependency-check.sh --version   # confirm 12.1.0
```

(macOS users can alternatively `brew install dependency-check`, but check
the version matches the pin above.)

**Windows:** unzip the same release zip to e.g. `C:\tools\dependency-check`
and use `bin\dependency-check.bat` instead of `dependency-check.sh`.

---

## 4. NVD API key — everyone does this once

Dependency-Check downloads the entire NVD (National Vulnerability
Database) on first run and pulls updates after that. **Without an API key,
NVD rate-limits you to a crawl** — the first sync can take *hours* and
sometimes fails outright with 403s. With a key it's minutes.

1. Request a free key: <https://nvd.nist.gov/developers/request-an-api-key>
   (arrives by email almost immediately).
2. Put it in your shell profile (`~/.zshrc` / `~/.bashrc`):

   ```bash
   export NVD_API_KEY="your-key-here"
   ```

3. Do the one-time local database sync (10–20 min with a key):

   ```bash
   dependency-check.sh --updateonly --nvdApiKey "$NVD_API_KEY"
   ```

Treat the key like a low-sensitivity secret: don't commit it, don't paste
it in Discord screenshots. The pipeline is written so a missing key
**degrades to slow, not broken** — but get the key.

---

## 5. Install & configure on the Jenkins server (one person does this, once)

The pipeline runs with `agent any`, i.e. **directly inside the Jenkins
container** — so the tools must exist *inside that container*, not just on
the host. The compose file runs the container as root, so installs are
straightforward.

> ⚠️ **Do this BEFORE merging the `Security Scan` stage to main.** The
> stage preflight-checks for both tools and fails with a pointer to this
> section if either is missing. That failure is intentional and
> actionable — but let's just not hit it.

### 5.1 Set the pipeline environment variables

There is **only one** variable to set, and it's optional-but-recommended.
No Docker Hub credentials are needed: the stage scans images built
locally by the previous stage, so it never logs in or pulls from a
registry.

In Jenkins UI: **Manage Jenkins → System → Global properties → Environment
variables → Add**:

| Name | Value | Required? |
|---|---|---|
| `NVD_API_KEY` | An NVD API key (fine to use the setup person's — or request a dedicated one for the team) | Recommended — without it DC's NVD updates are painfully slow |

The stage reads this from the environment, and the key is deliberately
kept out of build logs (`set +x` around that command). If it's unset the
stage prints a warning and carries on — slow, not broken.

### 5.2 Install Trivy in the Jenkins container

```bash
# From the host that runs the compose stack:
docker exec -it $(docker ps -qf "name=jenkins") bash

# Now inside the Jenkins container:
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
  | sh -s -- -b /usr/local/bin v0.72.0
trivy --version
```

Trivy talks to the Docker daemon through the socket the container already
mounts (`/var/run/docker.sock`) — no extra configuration needed.

### 5.3 Install Dependency-Check in the Jenkins container

Still inside the container:

```bash
apt-get update && apt-get install -y --no-install-recommends unzip && rm -rf /var/lib/apt/lists/*
curl -L -o /tmp/dc.zip \
  https://github.com/jeremylong/DependencyCheck/releases/download/v12.1.0/dependency-check-12.1.0-release.zip
unzip -q /tmp/dc.zip -d /opt && rm /tmp/dc.zip
ln -sf /opt/dependency-check/bin/dependency-check.sh /usr/local/bin/dependency-check.sh
dependency-check.sh --version
```

(The container is `jenkins/jenkins:lts-jdk17`, so Java is already there.)

### 5.4 Pre-warm the vulnerability databases

The pipeline stores both databases on the `jenkins_home` **volume** so
they survive container rebuilds (the build workspace is wiped by
`deleteDir()` every run, so they can't live there). Warm them up now so
the first pipeline run doesn't sit "stuck" downloading for 20 minutes:

```bash
# Still inside the Jenkins container:
mkdir -p /var/jenkins_home/trivy-cache /var/jenkins_home/dependency-check-data
TRIVY_CACHE_DIR=/var/jenkins_home/trivy-cache trivy image --download-db-only
dependency-check.sh --updateonly \
  --data /var/jenkins_home/dependency-check-data \
  --nvdApiKey "<the-key-from-5.1>"
```

💡 Memory note: this box has no swap and has been OOM-killed before
(that's why the pipeline builds images one at a time). Run this pre-warm
while **no pipeline build is running**, not alongside one.

### 5.5 Caveat: container rebuilds lose the tool installs

`docker exec` installs live in the container's writable layer. The
**databases** survive rebuilds (they're on the volume) but the **binaries
don't** — if the `jenkins` service is ever rebuilt from `jenkins/Dockerfile`,
re-run 5.2–5.3. The durable fix is adding the installs to
`jenkins/Dockerfile` itself, roughly:

```dockerfile
# --- security scanners (Trivy + OWASP Dependency-Check) ---
RUN curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
      | sh -s -- -b /usr/local/bin v0.72.0 && \
    apt-get update && apt-get install -y --no-install-recommends unzip && \
    curl -L -o /tmp/dc.zip https://github.com/jeremylong/DependencyCheck/releases/download/v12.1.0/dependency-check-12.1.0-release.zip && \
    unzip -q /tmp/dc.zip -d /opt && rm /tmp/dc.zip && \
    ln -sf /opt/dependency-check/bin/dependency-check.sh /usr/local/bin/dependency-check.sh && \
    rm -rf /var/lib/apt/lists/*
```

**But:** `jenkins/Dockerfile` is one of the hard-won, don't-touch-casually
infra files — do **not** apply this without coordinating with whoever owns
the Jenkins setup first. Until then, 5.2–5.3 after any rebuild is the
procedure.

---

## 6. Day-one baseline: run report-only scans locally FIRST

Same reasoning as Ming Hao's linting rollout: **see what's already there
before we make it fail builds.** Otherwise the first enforced run
hard-fails on pre-existing vulnerabilities nobody has looked at, and
whoever pushed that morning gets blamed for two years of base-image debt.

Everyone (or at least a couple of us, with results shared) runs this
before we schedule the Phase 2 flip.

### 6.1 Trivy against the locally built images — report-only

First build the same two images the pipeline scans. `build` only builds —
it starts no containers, so there is nothing to tear down afterwards:

```bash
# From the repo root. Copy .env.example to .env first if you haven't
# (Compose reads POSTGRES_PASSWORD from it):
cp -n .env.example .env

# -f docker-compose.yml on its own skips docker-compose.override.yml,
# exactly like the pipeline does. Services are named explicitly.
docker compose -f docker-compose.yml -p foodgorilla_test build backend frontend

# Confirm the two image names the scan expects now exist:
docker images | grep foodgorilla_test
```

> ⚠️ Never run a bare `docker compose down` or `up` in this repo — `down`
> cannot be scoped to individual services and tears down the whole named
> project, and `jenkins` is a service in this same compose file. The
> `build` command above avoids the problem entirely by never starting
> anything.

Now scan. Report-only is simply **omitting `--exit-code 1`** (Trivy's
default exit code is 0 — findings are printed, exit status stays clean):

```bash
# REPORT-ONLY: no --exit-code flag anywhere.
trivy image --scanners vuln --severity HIGH,CRITICAL --no-progress \
  foodgorilla_test-backend:latest | tee trivy-backend-baseline.txt
trivy image --scanners vuln --severity HIGH,CRITICAL --no-progress \
  foodgorilla_test-frontend:latest | tee trivy-frontend-baseline.txt
```

This mirrors the pipeline exactly (same images, `--scanners vuln`,
`HIGH,CRITICAL`), just without failing on findings. Save the two baseline
files somewhere shared (e.g. the team drive) — that's our before picture.

### 6.2 Dependency-Check against the repo — report-only

From the **repo root** (reports go outside the repo so nothing gets
accidentally committed):

```bash
dependency-check.sh \
  --project foodgorilla \
  --scan backend \
  --scan frontend \
  --exclude "**/node_modules/**" \
  --enableExperimental \
  --format HTML \
  --out ~/foodgorilla-security-reports \
  --failOnCVSS 11 \
  --nvdApiKey "$NVD_API_KEY"

open ~/foodgorilla-security-reports/dependency-check-report.html   # xdg-open on Linux
```

`--failOnCVSS 11` is the DC equivalent of "report-only": CVSS maxes out at
10, so 11 can never trigger a failure. `--enableExperimental` is required —
without it DC skips Python entirely and `backend/requirements.txt` would
be silently un-scanned.

### 6.3 Triage the baseline together

For each HIGH/CRITICAL finding, decide one of:

| Decision | Typical action |
|---|---|
| **Fix now** | Bump the base image tag in `backend/Dockerfile` / `frontend/Dockerfile`; bump the package in `requirements.txt`; `npm audit fix` / bump in `package.json` and refresh `package-lock.json` |
| **Accept (for now), with a reason** | Add the CVE to `.trivyignore` / a DC suppression file (section 7.2) **with a comment saying why and when to revisit** |
| **False positive** | Same suppression mechanism, comment it as such |

The goal of the baseline isn't zero findings — it's that every remaining
finding is *known and deliberately accepted*, so enforcement only ever
flags **new** problems.

---

## 7. Phase 2: flipping the pipeline to enforcement

### 7.1 The two knobs

Both live in one place — the `environment` block of the `Security Scan`
stage in the Jenkinsfile:

```groovy
TRIVY_EXIT_CODE = '0'   // flip to '1'  → HIGH/CRITICAL image findings fail the build
DC_FAIL_CVSS    = '11'  // flip to '7'  → dependencies with CVSS ≥ 7.0 fail the build
```

Flip them in a small dedicated PR once the baseline is triaged, so the
change is visible and revertable on its own.

### 7.2 Accepting specific findings

- **Trivy:** create a `.trivyignore` file at the repo root — one CVE ID
  per line, comments allowed. Trivy picks it up automatically when run
  from the repo root (the Jenkins workspace *is* the repo root):

  ```
  # Accepted 2026-07: only exploitable via local attacker, revisit when
  # base image python:3.x-slim ships the fix
  CVE-2026-XXXXX
  ```

- **Dependency-Check:** create `dependency-check-suppression.xml` (DC's
  standard XML suppression format — the HTML report has a "suppress"
  button on each finding that generates the XML block for you), then add
  `--suppression dependency-check-suppression.xml` to the DC command in
  the Jenkinsfile stage.

Every suppression needs a comment with a reason and a revisit condition —
an ignore file without reasons is just muting the alarm.

### 7.3 `--ignore-unfixed` (worth discussing at the flip)

Some OS-package CVEs have **no fixed version released yet** — nothing we
can do but wait for the base image. Adding `--ignore-unfixed` to the Trivy
commands makes enforcement only fail on findings that are *actually
actionable* (a fix exists and we haven't applied it). Recommended at flip
time; without it, expect occasional "red build we literally cannot fix"
situations.

---

## 8. Reading scan results in Jenkins

Every run — pass or fail — archives the reports on the build page under
**Archived artifacts** (same place as the integration-test debug logs):

- `trivy-backend.txt` / `trivy-frontend.txt` — human-readable image scan tables
- `dependency-check-report/dependency-check-report.html` — full DC report (download and open in a browser)
- `dependency-check-report/dependency-check-report.json` — machine-readable, for any future tooling

The Trivy tables are also `cat`-ed straight into the console log, so a
quick skim of the build output shows image findings without downloading
anything.

---

## 9. Troubleshooting & gotchas

| Symptom | Cause / fix |
|---|---|
| Stage fails immediately: `trivy is not installed` / `dependency-check.sh is not installed` | Do sections 5.2–5.3; if the Jenkins container was recently rebuilt, this is expected — see 5.5 |
| First DC run takes forever / NVD `403` or `404` errors | Missing/typo'd NVD API key — section 4 (local) or 5.1 (Jenkins). Also just retry: NVD itself has flaky days |
| Stage fails: `expected image foodgorilla_test-backend:latest is not present` | The scan targets are built by `Integration Testing` right above, so check that stage's log first. If it passed, the likely cause is an image-naming change: Compose names build-only images `<project>-<service>`, so renaming the compose project or a service renames the images too. Run `docker images \| grep foodgorilla_test` on the Jenkins host to see the real names, and update the two names in the stage to match |
| Trivy reports `0 vulnerabilities` on both images and it looks too good | Check the DB actually updated — an air-gapped or rate-limited Trivy will happily scan against a stale/empty database. `trivy image --download-db-only` on the Jenkins host, then re-run |
| DC reports nothing for the backend | `--enableExperimental` missing — Python analyzers are experimental in DC and off by default (the pipeline command includes it; include it locally too) |
| Everything is slow / Jenkins container gets killed | This box is memory-tight with no swap. Don't run local heavy builds or the section 5.4 pre-warm while a pipeline build is in flight |
| Your push's build fails in `Integration Testing` before ever reaching the scan, with weird container-name conflicts | Known collision: two branches' test stages fight over the same `foodgorilla_test` compose project. Don't push at the same time as a teammate testing their branch — re-run when theirs finishes |
| Scanner versions drift between people | Re-check section 2; upgrades go through a PR that updates this doc + the Jenkins server together |

---

## Quick reference card

```bash
# Build the two images the pipeline scans (starts nothing):
docker compose -f docker-compose.yml -p foodgorilla_test build backend frontend

# Local report-only image scan (mirrors the pipeline):
trivy image --scanners vuln --severity HIGH,CRITICAL foodgorilla_test-backend:latest
trivy image --scanners vuln --severity HIGH,CRITICAL foodgorilla_test-frontend:latest

# Local report-only dependency scan (from repo root):
dependency-check.sh --project foodgorilla --scan backend --scan frontend \
  --exclude "**/node_modules/**" --enableExperimental \
  --format HTML --out ~/foodgorilla-security-reports \
  --failOnCVSS 11 --nvdApiKey "$NVD_API_KEY"

# Update local vulnerability databases:
trivy image --download-db-only
dependency-check.sh --updateonly --nvdApiKey "$NVD_API_KEY"
```

Questions → Ryan. Nightly re-scan scheduling → Alden (his timer trigger
runs this same stage nightly; that dependency is why this stage has no
`TimerTrigger` guard).
