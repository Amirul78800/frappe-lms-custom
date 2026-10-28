# Frappe LMS Custom Image

A multi-arch, production-ready Docker image for [Frappe LMS](https://github.com/frappe/lms) targeting Frappe v16. Bundles the required `payments` dependency directly into the image layer to prevent runtime site installation crashes.

[![Build and Push](https://github.com/Amirul78800/frappe-lms-custom/actions/workflows/build.yml/badge.svg)](https://github.com/Amirul78800/frappe-lms-custom/actions)
[![Docker Hub](https://img.shields.io/badge/dockerhub-amirul123%2Flms--custom-blue?logo=docker&style=flat-square)](https://hub.docker.com/r/amirul123/lms-custom)
[![License](https://img.shields.io/github/license/Amirul78800/frappe-lms-custom?style=flat-square)](LICENSE)

---

## Why This Exists

In official builds (such as `ghcr.io/frappe/lms:stable`), the `payments` application—a mandatory dependency declared in LMS's `hooks.py`—is omitted from the container image. 

Executing `bench new-site --install-app lms` or subsequent setup steps fails immediately with:
```text
ModuleNotFoundError: No module named 'payments'
```

Installing apps via bench get-app inside a running container only writes to the ephemeral container layer, causing modifications to disappear upon recreation (docker compose down && up). This repository builds both payments and lms straight into the image layers using frappe_docker's layered Containerfile.

Related upstream issues:
- https://github.com/frappe/lms/issues/2350
- https://github.com/frappe/lms/issues/2467

## What this does

Builds via `frappe_docker`'s `images/layered/Containerfile`, targeting Frappe **v16**, with `apps.json`
correctly listing both:

```json
[
  { "url": "https://github.com/frappe/payments", "branch": "version-16" },
  { "url": "https://github.com/frappe/lms", "branch": "develop" }
]
```

## Bundled Applications (`apps.json`)


| Application | Branch / Target | Purpose |
| :--- | :--- | :--- |
| Frappe Framework | version-16 | Core web runtime and database layer |
| Frappe Payments | version-16 | Billing, payment gateway integrations, and prerequisites |
| Frappe LMS | develop | Course management, certifications, quizzes, and student portals |

> **Note on branches:** The `payments` app does not currently maintain an active `version-16` branch; Frappe v16 compatibility is merged into `develop`. This build tracks `develop` for `payments` and will transition to `version-16` once tagged upstream.

## Requirements

Frappe v16 requires **Python 3.14+** and **Node.js 24+** — already reflected in the build args below.

## Quick Start & Usage
Swap the backend image in your existing docker-compose.yml (or compose.yaml):
```bash
services:
  backend:
    image: amirul123/lms-custom:latest
    # ... rest of your compose configuration
```

## Site Creation
Create a new site with the proper database scope flag to prevent permission lockouts across container restarts:
```bash
docker compose exec backend bench new-site lms.yourdomain.local \
  --mariadb-root-password ${DB_ROOT_PASSWORD} \
  --admin-password ${ADMIN_PASSWORD} \
  --mariadb-user-host-login-scope=% \
  --install-app payments \
  --install-app lms \
  --no-mariadb-socket
```

## Image Tags & Multi-Arch Support
Builds are natively compiled for both `linux/amd64` and `linux/arm64` (Apple Silicon / AWS Graviton) via matrix runners, with manifests merged automatically:

- `amirul123/lms-custom:latest` — Tracks the latest stable commit on main.
- `amirul123/lms-custom:<branch>` — Tracks development branches (e.g., v1.1).
- `amirul123/lms-custom:<sha>` — Immutable commit SHA pins for deterministic deployments.

## Operations & Maintenance
Backups
```bash
docker compose exec backend bench --site ${SITE_NAME} backup --with-files
```

Site Restores
```bash
docker compose exec backend bench --site ${SITE_NAME} restore \
  /path/to/backup.sql.gz \
  --mariadb-root-password ${DB_ROOT_PASSWORD}
```

Schema Migration
When pulling updated images:
```bash
docker compose pull
docker compose up -d
docker compose exec backend bench --site ${SITE_NAME} migrate
docker compose exec backend bench --site ${SITE_NAME} clear-cache
```

## Local Build
To compile the image locally using Docker BuildKit:
```bash
git clone [https://github.com/frappe/frappe_docker](https://github.com/frappe/frappe_docker)
cd frappe_docker
docker build \
  --build-arg=FRAPPE_PATH=[https://github.com/frappe/frappe](https://github.com/frappe/frappe) \
  --build-arg=FRAPPE_BRANCH=version-16 \
  --build-arg=PYTHON_VERSION=3.14.0 \
  --build-arg=NODE_VERSION=24.12.0 \
  --secret=id=apps_json,src=/path/to/apps.json \
  --tag=lms-custom:test \
  --file=images/layered/Containerfile .
```

## License
Frappe Framework, Frappe Payments, and Frappe LMS are released under their respective upstream open-source licenses (GNU General Public License v3.0 / MIT). This repository provides containerization configuration and build tooling only. 
