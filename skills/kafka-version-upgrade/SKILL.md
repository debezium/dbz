---
name: kafka-version-upgrade
description: Use when the user wants to upgrade the Apache Kafka version in the Debezium project — covers pom.xml, antora.yml, and the Kafka container-image Dockerfile across the debezium and container-images repositories. Activated by phrases like "upgrade Kafka to X.Y.Z", "bump Kafka version", or when given a PR/issue link referencing a Kafka upgrade.
metadata:
  argument-hint: "<PR-or-issue-URL or 'kafka X.Y.Z to A.B.C'>"
---

# Kafka Version Upgrade

This skill guides the full Kafka version upgrade workflow across the `debezium` and `container-images` repositories.

## Step 0 — Parse the Input

The user supplies either:
- A GitHub **PR URL** (e.g. `https://github.com/debezium/debezium/pull/NNNN`) — fetch its title/body with `gh pr view <URL>` to extract old version, new version, and branch name.
- A GitHub **issue URL** (e.g. `https://github.com/debezium/debz/issues/NNNN`) — fetch with `gh issue view <URL>`.
- A plain text description like "upgrade Kafka from 4.3.0 to 4.3.1".

From the input extract:
- `OLD_VERSION` — e.g. `4.3.0`
- `NEW_VERSION` — e.g. `4.3.1`
- `ISSUE_NO` — the numeric issue/PR number (e.g. `2474`) — used as the branch name `dbz#<ISSUE_NO>` and commit prefix `debezium/dbz#<ISSUE_NO>`
- `DBZ_DIR` — the Debezium image directory inside `container-images/kafka/` to update (the Debezium release stream, e.g. `3.7`). Determine it by reading `container-images/kafka/` and finding the directory whose `Dockerfile` currently pins `OLD_VERSION`. Use `execute_command` with `grep -rl "KAFKA_VERSION=$OLD_VERSION" container-images/kafka/` to find it.

Set up these variables for all subsequent steps.

---

## Step 1 — Create `dbz#<ISSUE_NO>` branch in both repositories

Run these commands sequentially:

```bash
cd /home/jpechane/sources/debezium/debezium && git checkout -b "dbz#<ISSUE_NO>"
cd /home/jpechane/sources/debezium/container-images && git checkout -b "dbz#<ISSUE_NO>"
```

Confirm both branches were created (exit code 0).

---

## Step 2 — Download Kafka `NEW_VERSION` and collect companion information

### 2a. Download the tarball

```bash
curl -fSL -o /tmp/kafka-new.tgz \
  https://downloads.apache.org/kafka/<NEW_VERSION>/kafka_2.13-<NEW_VERSION>.tgz
```

If that fails (mirror not yet propagated) fall back to the archive URL:

```bash
curl -fSL -o /tmp/kafka-new.tgz \
  https://archive.apache.org/dist/kafka/<NEW_VERSION>/kafka_2.13-<NEW_VERSION>.tgz
```

### 2b. Capture SHA512 hash

```bash
sha512sum /tmp/kafka-new.tgz
```

Record the **uppercase** hex string only (no filename). This is `NEW_SHA512`.

To uppercase: pipe through `awk '{print toupper($1)}'`.

### 2c. Inspect companion library versions inside the tarball

```bash
tar -tzf /tmp/kafka-new.tgz \
  | grep -E "/(jackson|zstd|netty|slf4j)" \
  | grep '\.jar$' \
  | sort
```

Compare the jar filenames against the current property values in `debezium/pom.xml` (lines near `<!-- Kafka and its dependencies MUST reflect what the Kafka version uses -->`):

| Property | pom.xml key |
|---|---|
| Jackson core/databind/annotations | `version.jackson`, `version.jackson.databind`, `version.jackson.annotations` |
| SLF4J | `version.org.slf4j` |
| zstd-jni | `version.zstd-jni` |
| Netty | `version.netty` |

Note any version differences as `COMPANION_DELTAS`. For a patch Kafka release these are usually empty.

### 2d. Clean up

```bash
rm /tmp/kafka-new.tgz
```

---

## Step 3 — Update the `debezium` repository

### 3a. Edit `debezium/pom.xml`

Use `search_and_replace` or `apply_diff` to change:

```xml
<version.kafka>OLD_VERSION</version.kafka>
```
→
```xml
<version.kafka>NEW_VERSION</version.kafka>
```

If `COMPANION_DELTAS` is non-empty, apply each companion property change in the same edit.

### 3b. Edit `debezium/documentation/antora.yml`

Change:

```yaml
debezium-kafka-version: 'OLD_VERSION'
```
→
```yaml
debezium-kafka-version: 'NEW_VERSION'
```

### 3c. Commit

```bash
cd /home/jpechane/sources/debezium/debezium && \
  git add pom.xml documentation/antora.yml && \
  git commit -s -m "debezium/dbz#<ISSUE_NO> Upgrade Kafka to <NEW_VERSION>"
```

The `-s` flag adds a `Signed-off-by` trailer. The commit message prefix **must** be `debezium/dbz#<ISSUE_NO>`.

---

## Step 4 — Validate the `debezium` repository build

Run the full assembly build (skipping tests) to confirm Kafka `NEW_VERSION` artifacts resolve from Maven Central / Confluent repos:

```bash
cd /home/jpechane/sources/debezium/debezium && \
  ./mvnw -T 3 clean install -DskipTests -DskipITs -Passembly
```

This is a long-running command — set a generous timeout (≥ 30 minutes). Watch for dependency-resolution errors referencing Kafka artifacts. If the build fails because Maven Central does not yet carry `NEW_VERSION` artifacts, note the gap and proceed — the change is still correct, the artifacts just need time to propagate.

---

## Step 5 — Update the Kafka container-image Dockerfile

Identify the target file:

```
container-images/kafka/<DBZ_DIR>/Dockerfile
```

Apply two changes using `apply_diff` or `search_and_replace`:

1. `ARG KAFKA_VERSION=OLD_VERSION` → `ARG KAFKA_VERSION=NEW_VERSION`
2. `ARG SHA512HASH="OLD_HASH"` → `ARG SHA512HASH="NEW_SHA512"`

The hash must be the **uppercase** hex string captured in Step 2b, with no spaces and wrapped in double quotes.

### Commit

```bash
cd /home/jpechane/sources/debezium/container-images && \
  git add kafka/<DBZ_DIR>/Dockerfile && \
  git commit -s -m "debezium/dbz#<ISSUE_NO> Upgrade Kafka to <NEW_VERSION>"
```

---

## Step 6 — Build and validate the updated container image

### 6a. Build the base image (if not locally present)

```bash
cd /home/jpechane/sources/debezium/container-images && \
  DEBEZIUM_DOCKER_REGISTRY_PRIMARY_NAME=debezium docker build -t debezium/base base/
```

### 6b. Build the Kafka image

```bash
docker build \
  --build-arg DEBEZIUM_DOCKER_REGISTRY_PRIMARY_NAME=debezium \
  -t debezium/kafka:<DBZ_DIR>-test \
  /home/jpechane/sources/debezium/container-images/kafka/<DBZ_DIR>/
```

Confirm the build passes the `sha512sum -c` verification step. A hash mismatch means `NEW_SHA512` was wrong — re-run Step 2b carefully. A successful build validates the hash and the download URL.

---

## Step 7 — Summary

Report:
- `OLD_VERSION` → `NEW_VERSION` successfully applied
- Files changed:
  - `debezium/pom.xml` (`version.kafka` + any companion deltas)
  - `debezium/documentation/antora.yml` (`debezium-kafka-version`)
  - `container-images/kafka/<DBZ_DIR>/Dockerfile` (`KAFKA_VERSION` + `SHA512HASH`)
- Commits made in each repo (show `git log --oneline -1` for each)
- Maven build status (pass / failed / skipped with reason)
- Docker build status (pass / failed / skipped with reason)
- Any companion dependency version changes applied

---

## Notes

- Only the Debezium image directory matching `OLD_VERSION` is updated. Other `container-images/kafka/` directories (older Debezium streams) keep their existing versions.
- The `debezium-bom/pom.xml` uses `${version.kafka}` — no direct edits needed there.
- The `debezium.github.io` repository is **not** in scope for this skill; website metadata is managed separately when a full Debezium release is cut.
- Always use `origin` (personal fork) for pushes unless instructed otherwise.
- The commit message format **must** be: `debezium/dbz#<ISSUE_NO> <description>` with `-s` (sign-off).
