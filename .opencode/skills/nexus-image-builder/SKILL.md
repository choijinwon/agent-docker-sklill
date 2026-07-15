---
name: nexus-image-builder
description: Use when creating, reviewing, or debugging a PyTorch-focused AI/ML container image builder for OpenCode in a closed network, with Nexus wheel-only dependencies, BuildKit, Harbor, Argo Workflows, KServe serving images, Runtime Contract, runtime version matrices, Python/PyTorch/MLflow/Oracle checks, project indexing, bucket artifact performance, Docker size optimization, Build Reports, and failure triage.
---

# Nexus AI/ML Image Builder

Use this skill to design or review an auditable AI/ML image builder. Optimize the build system, not only Docker commands.

## Core Rules

- Treat Nexus as the only Python package source in closed-network builds.
- Use BuildKit with registry cache; keep cache images separate from release images.
- Build from a declarative spec, not hidden CI variables.
- Use Runtime Contract and Runtime Lineage for approval and reproducibility.
- Separate `training`, `serving`, `batch`, `runtime-base`, `framework`, `application-base`, and `release` images.
- Do not assume PVC/NAS exists. Support object storage, OCI model artifacts, or model-in-image with strict reporting.
- In OpenCode, index build inputs and changed files; do not scan the whole repository each run.
- Exclude `.opencode` from application build indexing unless the task explicitly updates OpenCode skills or config.

## Required Spec Checks

Before generating Dockerfiles, Argo workflows, or builder code, require a spec such as `image-builder.spec.json` or `image-builder.spec.yaml`.

Validate:

- source repository, branch or commit, project owner
- network mode: `closed-network`, `proxy-only`, or `internet-allowed`
- image category and runtime variant id
- Runtime Contract id, version, parent image reference, and digest
- Python version and ABI
- framework versions: PyTorch, torchvision, torchaudio, CUDA/CPU runtime, FastAPI, MLflow when used
- Oracle client/Instant Client and Python driver when Oracle is used
- lock file path and lock hash
- Nexus repository, wheel-only policy, and wheelhouse manifest
- Harbor release repository and BuildKit cache repository
- model/artifact source, checksum, and packaging mode
- security policy: non-root, digest pinning, latest ban, SBOM/provenance/signature
- report output path and notification target when used

Fail early when any required digest, lock hash, runtime variant, Runtime Contract, or image category is missing.

## Annotated Spec Skeleton

Use this commented YAML as the authoring template. If the builder requires JSON, strip comments when rendering.

```yaml
project:
  name: my-model-service          # Stable project/service id
  owner: ml-platform              # Team responsible for release approval
  source_ref: abc1234             # Git commit, not a floating branch

network:
  mode: closed-network            # closed-network | proxy-only | internet-allowed
  mirrors:                        # All endpoints must be internal in closed-network mode
    nexus_pypi: https://nexus.example/repository/pypi-wheels
    harbor: harbor.example/ml
    base_images: harbor.example/base
    os_packages: https://mirror.example/os
    model_bucket: s3://ml-artifacts/models

index:
  path: .build/index.json         # Generated build input index
  exclude: [".opencode", ".git", ".venv", "datasets", "models/raw"]

runtime_matrix:
  - variant_id: py310-cu121-torch24-mlflow214-oracle21-serving
    image_category: serving
    platform: linux/amd64
    python: "3.10.14"
    python_abi: cp310
    runtime_contract: serving-py310-cu121@sha256:...
    parent_image: harbor.example/base/pytorch-runtime@sha256:...
    framework:
      torch: "2.4.0"              # User-provided PyTorch version is authoritative
      torchvision: "0.19.0"       # Must match torch/CUDA compatibility policy
      torchaudio: "2.4.0"
      cuda: "12.1"                # null for CPU-only variants
      fastapi: "0.115.0"
      mlflow: "2.14.3"            # User-provided version is authoritative
    oracle:
      client: "21.13"             # Oracle client/Instant Client version
      driver: "oracledb==2.4.1"   # Python package must match the lock file
    lock_file: locks/serving-py310.lock
    lock_hash: sha256:...
    wheelhouse_key: serving/cp310/linux-amd64/<lock_hash>

artifacts:
  model_packaging: image          # external | oci-artifact | image
  manifest: s3://ml-artifacts/models/model-manifest.json
  checksum: sha256:...            # Cache and release decisions use checksum

build:
  target: release-with-model      # dependencies | test | release | release-with-model
  cache_repo: harbor.example/cache/my-service
  release_repo: harbor.example/ml/my-service
  tag: my-service-abc1234         # Do not use latest as deployable tag

report:
  path: reports/build-report.json
```

## Runtime Contract And Matrix

Use one matrix row per supported combination of Python, ABI, platform, PyTorch, CUDA/CPU runtime, MLflow, Oracle, and image category.

For each row:

- produce a separate image digest and Build Report entry
- use a separate lock hash, wheelhouse key, and BuildKit cache scope
- validate Nexus wheels against Python ABI and platform
- validate PyTorch, torchvision, torchaudio, CUDA runtime, and parent image compatibility
- validate Oracle client libraries and Python driver compatibility when declared
- compare environments by variant id, ABI, platform, framework versions, lock hash, parent digest, and spec hash
- reject undeclared cross-product combinations

For Argo, expand the matrix only after spec validation. Limit Nexus, BuildKit, bucket, and Harbor concurrency independently.

## Version Policies

Python:

- Require exact Python version and ABI, such as `3.10.14` and `cp310`.
- Do not reuse dependency layers across different ABI values.
- Record requested and installed Python versions in Runtime Contract and Build Report.

PyTorch:

- Treat `torch`, `torchvision`, `torchaudio`, Python ABI, platform, and CUDA/CPU runtime as one compatibility set.
- Use the user-provided versions from the spec or lock file; do not auto-upgrade to latest.
- Keep CPU and GPU variants separate; do not ship CUDA libraries in CPU-only serving images.
- Prefer CUDA runtime bases for serving; keep CUDA devel/toolkit images for training/build stages only.
- Verify PyTorch wheels are mirrored in Nexus for the target ABI/platform/runtime before BuildKit starts.
- Run a smoke check that imports torch and verifies `torch.__version__`, CUDA availability expectation, and model load path.

MLflow:

- Do not pin MLflow to latest by default.
- Use the user-provided spec or lock version as authoritative.
- Include MLflow only when training, serving, or batch runtime actually uses MLflow APIs or model format.
- Fail when spec, lock file, Nexus wheel, and installed package metadata disagree.

Oracle:

- Do not infer or auto-upgrade Oracle client versions.
- Record Oracle client/Instant Client separately from Python drivers such as `oracledb` or `cx_Oracle`.
- Verify required shared libraries and runtime library paths in the final image.
- Fail when client mode, Python driver, or target database compatibility is unclear.

## Nexus, Wheelhouse, And uv

Closed-network flow:

```text
validate spec -> prepare wheelhouse from Nexus -> verify lock/hash -> BuildKit offline install
```

Rules:

- Use wheel-only dependencies from Nexus.
- Do not allow public PyPI fallback.
- Report wheel count, total bytes, download seconds, and cache hit/miss.
- Install offline inside BuildKit from the prepared wheelhouse.
- For uv, run dependency preparation once with `uv sync --frozen` or equivalent.
- Use `uv run --no-sync` for short commands after correctness is established.
- Use a persistent uv cache keyed by Python ABI, platform, lock hash, and runtime variant.
- Set filesystem link mode, such as `UV_LINK_MODE=copy`, when links are slow or unsupported.

## Dockerfile, BuildKit, And Size

Use one multi-stage Dockerfile:

```text
base -> dependencies -> test -> release -> release-with-model
```

Keep cache-sensitive order:

```text
Runtime Image -> Lock File -> Wheelhouse -> Dependency Install -> Source -> Runtime Config -> Model
```

Rules:

- Copy lock files before application source.
- Keep wheelhouse, compilers, headers, tests, notebooks, and scan tools out of release images.
- Prefer digest-pinned `FROM` images and non-root final users.
- Ban `latest` as the only deployable tag.
- Use `.dockerignore` for `.git`, `.opencode`, virtualenvs, caches, notebooks, datasets, raw models, test outputs, and local artifacts.
- Use slim/runtime bases for serving; avoid CUDA `devel` images in serving releases.
- Report final image size, largest layers, and model contribution.
- Do not remove CA certificates, timezone/locale data, health checks, or required diagnostics.

## Model And Bucket Artifacts

If PVC/NAS is unavailable, choose one:

- object storage or model registry pull at startup when network and credentials are reliable
- OCI artifact/model layer in Harbor when supported
- model-in-image when startup reliability matters more than image size or rollout speed

Bucket performance rules:

- Avoid recursive prefix listing during every build.
- Read a small manifest first, then use `HEAD` or metadata checks before large downloads.
- Cache by object URI, version/id, size, checksum, and runtime variant.
- Download only changed objects.
- Use bounded multipart/parallel download; cap concurrency so the backend is not saturated.
- Stage shared artifacts once per Argo workflow and reuse them across matrix rows.
- Index model metadata and checksums, not raw model binaries.

For model-in-image:

- Copy model files after dependency install and application source decisions.
- Store under a stable path such as `/models/<model-name>`.
- Record model name, version, format, size, checksum, and source.
- Keep model files out of base, runtime, framework, and application-base images.
- Scan, sign, and report the final image after the model is included.

## OpenCode Closed-Network Performance

Preflight before build:

- verify internal endpoints: Git, Nexus, Harbor, base images, OS mirror, bucket/model store, SBOM/signing
- fail fast on public PyPI, Docker Hub, GitHub, apt/yum upstream, or external model URLs
- check DNS, TLS, proxy, and credentials once, then reuse results across matrix rows
- resolve parent digests before BuildKit starts
- verify wheelhouse completeness before dependency install
- import BuildKit registry cache before building

Project indexing:

- Index only build inputs: specs, Dockerfiles, Bake files, Argo templates, KServe manifests, dependency locks, Runtime Contract, Runtime Lineage, image catalog, `.dockerignore`, scripts, entrypoints, health checks, and package manager config.
- Exclude `.opencode`, `.git`, `.venv`, caches, test outputs, notebooks, datasets, raw model binaries, generated reports, wheelhouse output, and package manager caches.
- Include `.opencode` only for OpenCode skill/config updates.
- Record path, size, mtime, checksum, role, runtime variant, and cache key contribution.
- Rebuild the index only when Git diff, file watcher events, or checksum comparison shows relevant changes.

Use the index to skip dependency resolution, wheelhouse preparation, Dockerfile rendering, unchanged Argo matrix rows, and full-repository scans.

## Builder Pipeline

Prefer this flow:

```text
validate-build-spec
  -> load-or-refresh-project-index
  -> closed-network-preflight
  -> expand-runtime-matrix
  -> verify-runtime-contract
  -> resolve-parent-digest
  -> prepare-wheelhouse-from-nexus
  -> prepare-or-verify-bucket-artifacts
  -> render-dockerfile
  -> generate-bake-targets
  -> build-dependency-or-test-target
  -> run-unit-smoke-security-gates
  -> build-release-image
  -> push-release-image-to-harbor
  -> resolve-final-image-digest
  -> generate-sbom-provenance
  -> sign-and-attest
  -> update-runtime-lineage-catalog
  -> publish-build-report
```

Push only release targets after gates pass, unless a quarantine registry is explicitly used. Sign only immutable references such as `repository@sha256:...`.

## Build Report

Every build should produce JSON with:

- workflow name, UID, status, timestamp
- spec hash, source commit, network mode, preflight result
- project index hash, refresh seconds, changed input count
- runtime variant id, matrix row hash, image category, Runtime Contract
- parent digest, final digest, lineage id/version
- Python ABI/version, CUDA/framework versions, MLflow version, Oracle client/driver versions
- PyTorch, torchvision, torchaudio, and CPU/CUDA runtime expectation
- lock file, lock hash, Nexus repository, wheel count/bytes/download seconds
- uv sync seconds, uv run seconds, offline/index mode
- bucket source, checksum, bytes, object count, download seconds, cache hit/miss
- BuildKit cache import/export seconds, cache hit rate
- base image pull seconds, build seconds, Harbor push seconds, Argo queue/wait seconds
- model packaging mode, model metadata, final image size, largest layer contributors
- SBOM/provenance/signature status, non-root/latest/digest policy result
- failure step, reason, and triage category

Use the report to identify whether the bottleneck is indexing, blocked external access, Nexus, uv, bucket, BuildKit, Harbor, image size, rollout, or Argo scheduling.

## Review Checklist

Check:

- spec exists and is validated before rendering
- closed-network mode has internal mirrors, offline behavior, and preflight
- `.opencode` is excluded from application build indexing
- Runtime Contract is explicit and digest-based
- matrix variants represent multiple Python/framework/runtime combinations
- Nexus wheelhouse is prepared once and BuildKit installs offline
- uv does not repeatedly sync for short commands
- bucket artifacts use manifest/checksum caching instead of repeated prefix scans
- training, serving, and batch images are separated
- release image excludes tests, compilers, wheelhouse, caches, credentials, and raw unused artifacts
- MLflow, Oracle, and Python versions are pinned, validated, and reported
- PyTorch compatibility set is pinned, resolved from Nexus, smoke-tested, and reported
- BuildKit cache repos are separate from Harbor release repos
- SBOM/provenance/signature, non-root, latest ban, digest pinning, and report rules are enforced

## Failure Triage

- OpenCode indexing slow: check index scope, `.opencode`/`.git` ignores, large model/data files, checksum strategy, file watcher, and changed input count.
- Closed-network slow: check blocked external URLs, missing mirrors, DNS/proxy/TLS delays, full scans, cold caches, and Argo queue time.
- Nexus slow: check wheel-only availability, wheelhouse reuse, download concurrency, and response timing.
- uv slow: check repeated sync, stale lock, cache path, workspace discovery, offline index, link mode, and `--no-sync`.
- Bucket slow: check prefix listing, missing manifest, object count/size, checksum cache, multipart settings, concurrency, and workflow reuse.
- BuildKit slow: check dependency layer invalidation, cache import/export, builder CPU/memory, and `.dockerignore`.
- Harbor push slow: check image size, push concurrency, release/cache repo split, and network throughput.
- Image too large: check category separation, CUDA devel base, build tools, wheelhouse/cache leakage, `.dockerignore`, model layer size, and unused dependencies.
- Model rollout slow: check model size, image pull time, node cache reuse, rollout strategy, and model-in-image tradeoff.
- Runtime mismatch: compare ABI, Python, PyTorch, CUDA/CPU runtime, MLflow, Oracle, platform, Runtime Contract, and Runtime Lineage.
- Reproducibility failure: compare parent digest, lock hash, Dockerfile hash, spec hash, source commit, build args, and final digest.
- Security failure: check non-root user, latest tags, digest pinning, SBOM/provenance/signature, and credential handling.
