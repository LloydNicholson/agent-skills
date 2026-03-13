---
name: docker-debug
description: Diagnostic and troubleshooting skill for Docker containers and services.
---

# Docker Debug Skill

Automated diagnostic workflow for troubleshooting Docker containers and services.

## Overview

This skill provides systematic debugging for Docker environments by:
- Listing and inspecting containers and services
- Checking logs and error messages
- Diagnosing networking and volume issues
- Identifying resource constraints
- Suggesting remediation steps

## Invocation

```
/docker-debug [CONTAINER_NAME_or_SERVICE_NAME]
```

## Workflow

### Step 1 — Inventory
List running and stopped containers/services.

### Step 2 — Inspect
Examine container configuration, networking, and health status.

### Step 3 — Collect Logs
Retrieve container logs and recent error messages.

### Step 4 — Diagnose
Identify common issues:
- Out of memory
- Network connectivity
- Volume mount problems
- Port conflicts
- Service dependencies

### Step 5 — Recommend
Provide actionable solutions and next steps.

## Configuration

Customize diagnostic scope:
- Log tail length
- Network inspection depth
- Resource threshold alerts
- Service dependency chains

## Notes

This is a template skill. Extend with Docker Compose, Kubernetes, or container orchestration-specific diagnostics as needed.
