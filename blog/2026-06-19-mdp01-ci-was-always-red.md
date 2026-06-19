---
layout: post
title: "CI Was Always Red"
date: 2026-06-19
entry_type: note
subtype: diary
projects: [casehub-quarkmind]
tags: [ci, maven, github-packages, dependencies]
---

CI had been failing since June 12. Every push — Build and Test red, before reaching a single test. The project itself was fine; the CI failure had just been queued behind feature work for a week.

I brought Claude in to read the logs. The failure was dependency resolution: eight SNAPSHOT artifacts that live in `~/.m2` on my machine and nowhere that GitHub Actions can see.

The fix split into two distinct problems.

## The Ones That Were Published but Not Configured

`casehub-ledger-api`, `casehub-engine-api`, `casehub-platform`, `casehub-qhorus` — all published to casehubio GitHub Packages. The engine, ledger, qhorus, and platform repos have publish workflows that fire on every main commit. Quarkmind's `pom.xml` just had no `<repositories>` entry pointing anywhere. Comparing against devtown's pom.xml, the fix was a single block:

```xml
<repository>
  <id>github</id>
  <url>https://maven.pkg.github.com/casehubio/*</url>
  <snapshots><enabled>true</enabled></snapshots>
</repository>
```

The wildcard `*` resolves across all repositories in the org — one block covers every artifact any casehubio repo has ever published. This isn't documented anywhere I could find. I only found it by reading other projects' pom.xml files.

## The Ones That Had Never Been Published

`casehub-core` and `casehub-persistence-memory` come from the POC — a personal fork with no publish workflow. `scelight-mpq` and `scelight-s2protocol` are from a local Scelight fork; the `feature/standalone-modules` branch that extracted them as standalone Maven modules had never been pushed to any GitHub remote.

I considered setting up proper publish pipelines — new repos, new GitHub Actions workflows, new token permissions. Both libraries are stable, open-source, and have exactly one consumer. We inlined the source instead: 150 Java files and 122 `.dat` SC2 protocol data files from the scelight fork, 55 files from casehub-core. Remove the external dependencies, rebuild.

One surprise: Maven silently drops non-Java files from `src/main/java/`. The `.dat` files vanish from the JAR without any warning. A `jar tf` on the output is the only way to discover they're missing — nothing in the build fails, nothing complains. The fix is an explicit `<resources>` declaration:

```xml
<resource>
  <directory>src/main/java</directory>
  <excludes><exclude>**/*.java</exclude></excludes>
</resource>
```

CI went green on the first run after those two changes.

The inlining approach is worth generalising: when a library is stable, Apache 2.0, and has one consumer, standing up a publish pipeline adds more maintenance than it removes. And the wildcard URL is worth knowing for any Maven project where multiple repos in an org publish artifacts — one `<repositories>` entry instead of one per dependency.
