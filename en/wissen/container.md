---
title: "Containers and Docker: operational knowledge for self-hosting"
blatt: "container"
description: "Containers in everyday administration: images and tags, volumes and the persistence pitfalls, Compose as the standard, networks, update strategies, and health checks, with a focus on reliable long-term operation."
fakten:
  - { label: "Core idea", wert: "Application together with its dependencies as an isolated, reproducible package" }
  - { label: "De facto standard", wert: "Docker and Docker Compose", href: "https://docs.docker.com/" }
  - { label: "Open standard", wert: "OCI (Open Container Initiative)", href: "https://opencontainers.org/" }
  - { label: "Persistence", wert: "Volumes and bind mounts; containers themselves are disposable" }
  - { label: "Operational factors", wert: "Tag strategy, health checks, log handling, restart policies" }
  - { label: "Tools", wert: "docker, docker compose, ctop/lazydocker" }
werbung: ["newsletter"]
ctaThemen: ["rclone", "paperless-ngx"]
---

# Containers and Docker: operational knowledge for self-hosting

Paperless, Home Assistant, monitoring, sync tools: self-hosting today almost always means containers. The technology is convenient, but it shifts the operational questions instead of abolishing them. Knowing where that shift occurs makes it possible to run containerized services as reliably as traditional installations.

## The mental model: containers are disposable

An **image** is the immutable template, a **container** its running instance, replaceable at any time. Everything that is supposed to survive the next restart belongs in **volumes** or **bind mounts**. From this follows the most important operational rule: a container is not repaired but recreated; only the data is sacred. The classic persistence pitfall: a service writes to a path that is not mounted, unnoticed for months, until an update replaces the container and the data is gone. Every new service therefore comes with a control question: which paths have to be persistent, and are they really persistent?

With mounts of network and cloud storage, a second pitfall is added: the **startup order**. If the mount is not yet present when the container starts, the service happily writes into the empty local folder underneath. Dependencies and mount checks therefore belong in the startup logic, not in wishful thinking.

## Compose as the operational standard

**Docker Compose** describes a service declaratively: image, volumes, networks, environment variables, restart behavior. The YAML file is therefore documentation and recovery plan in one and belongs under version control, with secrets kept outside it. Two building blocks make the difference in long-term operation: **health checks**, so that “running” also means “working”, and a deliberate **restart policy** (`unless-stopped` as a sensible default).

## Updates: tags are a decision

`latest` is convenient and unpredictable: every recreation can bring a different version. For production services, established practice is to use **versioned tags**, read the release notes, update at a chosen moment, and back up the volumes before major version jumps. In setups with many services, update detection is automated while the timing of the rollout stays under manual control, exactly as in patch management for traditional systems.

## Quick diagnostics

```bash
docker compose ps                # status and health of all services
docker compose logs -f --tail=100 service
docker inspect service | jq '.[0].Mounts'   # is what should be mounted really mounted?
```

