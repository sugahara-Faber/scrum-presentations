---
marp: true
theme: uncover
title: Consolidate Docker Environments
author: Masayuki Sugahara
paginate: true
---

<style>
section { font-size: 1.5rem; text-align: left; }
h1 { font-size: 2.2rem; }
h2 { font-size: 1.7rem; }
</style>

# Consolidate Docker Environments

PR [#20153](https://github.com/FaberTechnology/faber-extract/pull/20153) / `faber-extract`

Masayuki Sugahara

---

## The Problem

- **4** environment compose files × **3** environment Dockerfiles
- ~95% of each file was copy-pasted from its siblings
- Drift had already happened — `dockerd` healthcheck used `docker info` in dev-test but `docker ps` everywhere else
- A new environment meant duplicating two more files

**377 lines of redundancy across 7 files.**

---

## Before

```
faber-extract/
├── Dockerfile-dev
├── Dockerfile-dev-test
├── Dockerfile-prod
├── Dockerfile-stag
├── compose-dev-test.yml
├── compose-develop.yml
├── compose-production.yml
└── compose-staging.yml
```

Eight files. Two real axes of variation hiding inside.

---

## What actually differs between envs?

Only two things, really:

1. **Webpack build target** — `npm run build` vs `npm run build:staging`
2. **Per-env deploy args** — `DEPLOY_PATH`, `TOMCAT_USER`, `TOMCAT_PASS`, `REDIS_HOST`
   - …and these were *already* passed via `--build-arg` on the CLI

Everything else was identical copy-paste.

---

## Fix #1 — Parameterize the Dockerfile

```dockerfile
ARG BUILD_SCRIPT=build

RUN npm run ${BUILD_SCRIPT} && mvn package -s settings.xml -DskipTests=true
```

- One `Dockerfile` replaces `Dockerfile-dev`, `-dev-test`, `-stag`, `-prod`
- Defaults to `build` → production / cronjob / in-company workflows pass nothing
- Staging & dev-test pass `--build-arg BUILD_SCRIPT="build:staging"`

---

## Fix #2 — Single `compose.yml` with a `deploy` profile

```yaml
services:
  mieruca:
    profiles: [deploy]            # hidden from local `docker compose up`
    build:
      context: "."
      dockerfile: Dockerfile
      network: container:dockerd
      args:
        - MAVEN_IMAGE=faber-extract-maven:latest
        - TOMCAT_IMAGE=faber-extract-tomcat:latest
    image: mieruca-app
    depends_on:
      dockerd:
        condition: service_healthy
```

---

## Gotcha — `container_name: dockerd`

```yaml
  dockerd:
    image: docker:dind
    container_name: dockerd       # ← required
```

`build.network: container:dockerd` is resolved by **`docker build`**, which doesn't know about Compose service names.

Without a fixed `container_name`, the build network attaches to nothing.

---

## Workflow change

```diff
- docker compose -f compose-staging.yml up -d dockerd
- DOCKER_BUILDKIT=0 docker compose -f compose-staging.yml build mieruca \
+ docker compose -f compose.yml up -d dockerd
+ DOCKER_BUILDKIT=0 docker compose -f compose.yml build mieruca \
+ --build-arg BUILD_SCRIPT="build:staging" \
   --build-arg DEPLOY_PATH="faber-extract" \
   ...
```

5 deployment workflows updated. Production-style workflows just drop the file flag — `BUILD_SCRIPT` defaults to `build`.

---

## Results

|                  | Before    | After    |
| ---------------- | --------- | -------- |
| Compose files    | 4         | **1**    |
| Dockerfiles      | 4         | **1**    |
| Lines changed    | —         | **−377 / +39** |
| New env =        | 2 new files | 1 CLI flag |

One source of truth. No more silent drift between envs.

---

## Takeaways

- **Parameters > file duplication.** If files differ by one line, that line wants to be an arg.
- **Compose `profiles`** keep deploy-only services out of local `docker compose up`.
- **`build.network: container:<name>`** needs an explicit `container_name` — Compose service names aren't visible to `docker build`.
- Spotting the divergent healthcheck (`docker info` vs `docker ps`) was the cue that drift had already cost us.
