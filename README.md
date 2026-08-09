# lms-custom

Custom multi-arch Docker image for [Frappe LMS](https://github.com/frappe/lms), built to fix a bug in the
official `ghcr.io/frappe/lms:stable` image where the `payments` app (a required dependency of `lms`,
declared in its `hooks.py`) is not bundled into the image at all — causing
`ModuleNotFoundError: No module named 'payments'` on `bench new-site --install-app lms`.

Related upstream issues:
- https://github.com/frappe/lms/issues/2350
- https://github.com/frappe/lms/issues/2467

## What this does

Builds via `frappe_docker`'s `images/layered/Containerfile`, targeting Frappe **v16**, with `apps.json`
correctly listing both:

```json
[
  { "url": "https://github.com/frappe/payments", "branch": "develop" },
  { "url": "https://github.com/frappe/lms", "branch": "develop" }
]
```

`payments` doesn't have a dedicated `version-16` branch yet (see
[frappe/payments#200](https://github.com/frappe/payments/issues/200)) — v16 compatibility landed on
its `develop` branch instead, so that's what's pinned here. Swap to `version-16` once that branch
exists upstream.

## Requirements

Frappe v16 requires **Python 3.14+** and **Node.js 24+** — already reflected in the build args below.

## Tags

- `main` branch → `amirul123/lms-custom:latest`
- other branches → `amirul123/lms-custom:<branch-name>`

Published as multi-arch (`linux/amd64` + `linux/arm64`) via native runners, manifest merged in a
separate job.

## Usage

Point your `docker-compose.yml` service images at `amirul123/lms-custom:latest` instead of
`ghcr.io/frappe/lms:stable`. `--install-app payments` in your `create-site` step will now succeed
since the app source is actually present in the image.

## Required repo secrets

- `DOCKERHUB_USERNAME`
- `DOCKERHUB_TOKEN`
