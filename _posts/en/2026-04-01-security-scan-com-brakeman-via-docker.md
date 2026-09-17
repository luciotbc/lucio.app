---
title: "Security scan on your Rails app with Brakeman via Docker"
date: 2026-04-01 14:00:00 -0300
updated: 2026-04-01 14:00:00 -0300
tags: [ruby, rails, security, docker, brakeman]
excerpt: "Running Brakeman in a container, without installing the gem in your Gemfile, for a quick vulnerability scan."
lang: en
ref: security-scan-com-brakeman-via-docker
---

[Brakeman](https://brakemanscanner.org/) is a static analyzer that catches classic Rails vulnerabilities: SQL injection, mass-assignment, XSS, open redirect. I prefer running it via Docker to avoid polluting the `Gemfile` with a scan dependency and to use the latest version without touching the project.

```bash
docker run -v ~/work/overgrad:/code presidentbeef/brakeman --color
```

Replace `~/work/overgrad` with your repo path — it mounts the code into the container and runs the scan.

## Using it in CI

The same image works as a step in any CI pipeline. In GitHub Actions, for example, you can run it as a container and fail the build if a high-severity warning appears.

```yaml
- name: Brakeman
  run: docker run --rm -v ${{ github.workspace }}:/code presidentbeef/brakeman -w2 --no-progress
```

`-w2` filters medium/high severity warnings (`-w3` shows only high).
