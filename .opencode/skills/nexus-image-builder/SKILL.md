---
name: nexus-image-builder
description: Use when creating, reviewing, or debugging an AI/ML container image builder that uses Nexus PyPI wheel-only dependencies, BuildKit, Harbor, Runtime Contract, runtime version matrices, Oracle client version checks, Runtime Lineage, declarative builder specs, dependency caching, Docker image size optimization, digest-based reproducibility, image build reports, and training/serving/batch image separation.
---

# Nexus AI/ML Image Builder

Use this skill for AI/ML image builder work where the goal is not just a faster Docker build, but a standard, auditable image-building system.

## Core Position

Treat Nexus as a wheel-only package source. Put standardization, cache reuse, security, reproducibility, and reporting in the Builder.

Do not assume download speed is the main bottleneck. First check image structure, dependency/application layer separation, BuildKit cache, Runtime Contract, and report/digest policy.

## Required Inputs

Before generating Dockerfiles, Argo workflows, or builder code, require a declarative builder spec such as `image-builder.spec.json`.

Validate at least:

- source repository, branch or commit, repository name
- image category: `training`, `serving`, `batch`, or shared runtime/base
- runtime matrix or single runtime variant id when multiple Python/framework versions are supported
- runtime contract id and version
- runtime image reference and digest
- Python ABI and version
- CUDA/framework/runtime versions when applicable
- MLflow version when MLflow is used; treat the user-provided spec version as authoritative
- Oracle client or Instant Client version when Oracle connectivity is used
- dependency lock file and lock hash
- Nexus PyPI repository and wheel-only policy
- target platform and architecture
- BuildKit target: `dependencies`, `test`, or `release`
- Harbor image repository, immutable tag, and cache repository
- security policy: non-root, digest pinning, latest tag ban, SBOM/provenance/signature requirements
- report output path and notification destination when used

Fail early when any required digest, lock hash, runtime contract, runtime variant id, or image category is missing.

## 9 Requirements

Implement and review against these requirements:

1. Runtime Contract: common runtime environments must be fixed by digest.
2. Same Environment Criteria: compare ABI, framework, Python, CUDA, platform, lock hash, and builder spec, not image name alone.
3. Runtime Lineage: manage images by lineage, role, project, and parent digest chain.
4. Training/Serving Separation: keep training, serving, and batch images separate; distinguish strict and optimized runtime patterns.
5. Declarative Builder Spec: centralize layer, target, validation, and report rules.
6. Multi-stage Target: separate dependency, test, and release targets; push only release targets after gates pass.
7. Model Packaging Policy: externalize models only when the platform supports it; when PVC/NAS is unavailable, allow model-in-image with strict digest, size, cache, and release controls.
8. Security and Reproducibility: require digest, lock hash, non-root user, latest tag ban, report, and policy checks.
9. Wheel Repository: use Nexus wheel-only download and dependency cache reuse.

## Image Categories

Choose the image category before writing Dockerfile stages:

- `training`: includes training libraries, notebooks only if required, distributed training tools, and heavier debug dependencies.
- `serving`: minimal runtime for online inference; optimized startup, size, health check, and non-root runtime.
- `batch`: offline inference or scheduled jobs; may include data connectors and batch orchestration tools.
- `runtime-base`: organization-approved language/runtime base image.
- `framework`: shared PyTorch/TensorFlow/MLflow/FastAPI framework image.
- `application-base`: project or service-family shared dependencies without service-specific source.
- `release`: final deployable application image.

Do not combine training, serving, and batch requirements into one image unless the user explicitly chooses that tradeoff.

## Runtime Contract

Use Runtime Contract as the approved execution environment boundary.

Include:

- contract id and version
- parent image repository and digest
- operating system and architecture
- Python ABI and exact Python version
- CUDA, cuDNN, NCCL, PyTorch/TensorFlow versions when used
- MLflow version when the image logs, serves, loads, or registers MLflow models
- Oracle client/Instant Client version, driver package, and required runtime libraries when Oracle is used
- common runtime packages
- expected user id and filesystem permissions
- allowed package sources
- supported image categories
- health check and entrypoint conventions

The builder must reject or flag builds that mix incompatible Python ABI, CUDA, framework, or parent image digest.

## Runtime Matrix

When a project supports multiple Python, Oracle client, CUDA, framework, or MLflow versions, model them as explicit runtime variants. Do not build one image that tries to support every combination.

Use one matrix row per supported combination:

```json
{
  "variant_id": "py310-cpu-mlflow214-serving",
  "image_category": "serving",
  "python": "3.10.14",
  "python_abi": "cp310",
  "platform": "linux/amd64",
  "cuda": null,
  "framework": {"fastapi": "0.115.0", "mlflow": "2.14.3"},
  "oracle": {"client": "21.13", "driver": "oracledb==2.4.1"},
  "runtime_contract": "serving-py310-cpu@sha256:...",
  "lock_file": "locks/serving-py310.lock",
  "nexus_wheelhouse_key": "serving/cp310/linux-amd64/<lock_hash>"
}
```

For each matrix row:

- build and report a separate image digest
- use a separate lock hash, wheelhouse key, and BuildKit cache scope
- validate Nexus wheels against that Python ABI and platform
- validate Oracle client libraries and Python Oracle driver compatibility when Oracle is declared
- compare environments by `variant_id`, ABI, platform, framework versions, lock hash, and parent digest
- allow only declared combinations; reject accidental cross-product expansion

For Argo, expand the matrix into parallel build items only after spec validation. Limit concurrency per Nexus, BuildKit, and Harbor separately so a large matrix does not overload shared infrastructure.

## MLflow Version Policy

Do not pin MLflow to the latest version by default. Use the version supplied by the user in the builder spec or lock file.

Require MLflow version checks only when MLflow is part of the image contract:

- `training`: include MLflow when experiment tracking, model registration, or artifact logging runs inside the image.
- `serving`: include MLflow only when the serving runtime loads MLflow models or uses MLflow APIs at runtime.
- `batch`: include MLflow only when batch jobs read/write MLflow models, registry metadata, or tracking data.
- `framework`: allow MLflow as a shared framework package only if downstream images agree on the same version contract.

Validate:

- the user-provided MLflow version is exact, such as `mlflow==x.y.z`, not a floating range
- the lock file resolves the same MLflow version as the builder spec
- Nexus contains MLflow and all transitive wheels for the target Python version and platform
- the wheelhouse install result matches the requested MLflow version
- the Runtime Contract, Runtime Lineage, and Build Report record the requested and installed MLflow versions

Fail or require explicit approval when the lock file upgrades/downgrades MLflow, when Nexus is missing wheels, or when a serving image includes MLflow only for training-time convenience.

## Oracle Version Policy

Do not infer or auto-upgrade Oracle versions. Use the Oracle client or Instant Client version supplied by the user in the builder spec.

Require Oracle checks only when the image connects to Oracle:

- record Oracle client/Instant Client version separately from Python package driver versions
- pin Python driver packages such as `oracledb` or `cx_Oracle` in the lock file
- verify required shared libraries are present in the final image
- validate `LD_LIBRARY_PATH` or equivalent runtime path only when the driver requires it
- record Oracle client version, driver version, and connectivity mode in Runtime Contract, Runtime Lineage, and Build Report

Fail or require approval when the Oracle client version is missing, when Python driver and client mode are incompatible, or when the final image accidentally drops required shared libraries during size optimization.

## Runtime Lineage

Track images through this chain:

```text
Trusted Base -> Runtime -> Framework -> Application Base -> Application Release
```

Record at least:

- lineage id and version
- runtime variant id when matrix builds are used
- image role/category
- project or service owner
- parent image digest
- runtime/framework/application-base digest chain
- source commit
- lock hash
- builder spec hash
- Dockerfile hash
- BuildKit/Bake hash
- final image digest
- SBOM/provenance/signature status

Use Runtime Lineage as the approval unit when deciding whether a build may release.

## Dockerfile And BuildKit Rules

Use one multi-stage Dockerfile built by BuildKit. Do not split Docker image layers into many Argo tasks.

Prefer these stages:

```text
base -> dependencies -> test -> release
```

Keep cache-sensitive order:

```text
Runtime Image -> Lock File -> Wheelhouse -> Dependency Install -> Application Source -> Runtime Configuration
```

Rules:

- Copy lock files before source files.
- Copy wheelhouse before dependency install.
- Keep tests and build tools out of the final release image.
- Decide model packaging before writing the final stage; do not assume PVC/NAS exists.
- Use `USER` with a non-root UID in final images.
- Ban `latest` as the only deployable tag.
- Prefer digest-pinned `FROM` images.
- Use `.dockerignore` to exclude `.git`, virtualenvs, caches, notebooks, datasets, models, test outputs, and local artifacts.

## Image Size Optimization

Optimize release image size without breaking reproducibility, cache reuse, or startup reliability.

Use this priority order:

1. Separate `training`, `serving`, and `batch` images so serving images do not include training/debug tools.
2. Use multi-stage builds so compilers, build tools, wheelhouse, tests, and scan tools stay out of release images.
3. Use slim/runtime base images for release; avoid CUDA `devel` images in serving releases.
4. Keep dependency, application, and model layers separate so model changes do not reinstall Python packages.
5. Remove package manager caches in the same layer that creates them.
6. Exclude large or volatile files through `.dockerignore`.
7. Measure final image size and largest layers before calling the build optimized.

Release images must not contain:

- `tests`, notebooks, training-only scripts, or local fixtures
- compilers, headers, `build-essential`, Git, curl/wget unless needed at runtime
- wheelhouse, pip/uv cache, apt cache, BuildKit metadata
- `.git`, `.venv`, `__pycache__`, `.pytest_cache`, `.mypy_cache`, `.ruff_cache`
- datasets, temporary outputs, profiling dumps, and unused model variants
- credentials, config with secrets, or registry/package tokens

For Python:

- Prefer wheel-only installs from Nexus.
- Use `--no-cache-dir` for pip when not using a BuildKit cache mount.
- Prefer installing into a deterministic prefix such as `/opt/python-dependencies`.
- Remove `.pyc` and test data only when the runtime does not need them.

For model-in-image:

- Put model files in a late layer or `release-with-model` target.
- Keep `release-without-model` buildable for test/scan comparison.
- Report model size separately from application/runtime size.
- Flag rollout risk when model size dominates the image.

Do not chase tiny images by removing files needed for observability, health checks, CA certificates, timezone/locale support, or runtime diagnostics required by operations.

## Model Packaging Without PVC/NAS

If PVC/NAS is unavailable, prefer one of these options:

1. Object storage or model registry pull at startup, if network and credentials are reliable.
2. OCI artifact/model layer in Harbor, referenced by digest, if the platform supports artifact retrieval.
3. Model-in-image, when startup reliability matters more than image size or rollout speed.

When using model-in-image:

- Copy model artifacts after dependency install and application source decisions, so dependency cache remains reusable.
- Store model files under a stable path such as `/models/<model-name>`.
- Record model name, version, format, size, checksum, and source in the build report.
- Add model checksum or model artifact digest to the image label and Runtime Lineage catalog.
- Keep model files out of base, runtime, framework, and application-base images; include them only in release images or a dedicated model-release target.
- Avoid rebuilding dependency layers when only the model changes.
- Scan and sign the final image after the model is included.
- Call out rollout cost when model size makes image pull slow.

Recommended target split when models are packaged into images:

```text
dependencies -> test -> release-without-model -> release-with-model
```

Use `release-with-model` only for deployable images that intentionally include model artifacts.

## Nexus Wheelhouse Flow

In closed-network builds, separate package preparation from image building:

```text
validate spec -> download wheels from Nexus -> verify lock/hash -> BuildKit offline install
```

Use Nexus only in the dependency preparation step. BuildKit should install offline from wheelhouse:

```bash
python -m pip install --no-index --find-links=/build/wheelhouse \
  --only-binary=:all: --prefix=/opt/python-dependencies \
  -r /build/requirements.lock
```

Check:

- all required wheels exist in Nexus
- source distributions are not used in release paths unless explicitly approved
- wheel count and total bytes are reported
- Nexus download time is measured
- dependency cache hit/miss is explained
- Nexus, BuildKit, and Harbor concurrency limits are independent

For uv projects, avoid repeated `uv run` sync in short tasks. Prepare dependencies once with `uv sync --frozen` or an equivalent wheel preparation step, then use `uv run --no-sync` only after correctness is established.

## Builder Pipeline

Prefer this flow:

```text
validate-build-spec
  -> expand-runtime-matrix
  -> verify-runtime-contract
  -> resolve-parent-digest
  -> prepare-wheelhouse-from-nexus
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

Quality gates must pass before release push, unless the organization uses a quarantine registry. If quarantine is used, promote only after gates pass.

Sign only immutable references such as `repository@sha256:...`.

## Build Report

Every successful or failed build should produce a machine-readable report.

Include:

- workflow name, workflow UID, status, timestamp
- builder spec hash
- source repository and commit
- runtime variant id and matrix row hash when matrix builds are used
- image category and runtime contract
- runtime lineage id/version
- parent image digest and final image digest
- Python ABI, Python version, CUDA/framework versions
- requested and installed MLflow versions when MLflow is used
- requested and installed Oracle client and Python Oracle driver versions when Oracle is used
- lock file name and lock hash
- Nexus repository
- wheel count, total bytes, download seconds, average throughput
- dependency cache hit/miss
- model packaging mode, model name/version/checksum/size when present
- final image size, release-without-model size when available, and largest layer contributors
- BuildKit cache hit/miss when available
- build seconds and Harbor push seconds
- SBOM/provenance/signature status
- non-root/latest/digest policy result
- failure step and reason

Use the report to explain whether the bottleneck is Nexus download, dependency install, BuildKit build, Harbor push, or runtime rollout.

## Review Checklist

When reviewing or generating an image builder, check:

- A declarative builder spec exists and is validated before rendering.
- Runtime Contract is explicit and digest-based.
- Multiple Python/framework/runtime versions are represented as explicit matrix variants, not hidden conditionals inside one image.
- Same-environment comparison uses ABI/framework/Python/platform/lock/spec, not image name alone.
- Training, serving, and batch images are separated.
- Dockerfile stages are `base`, `dependencies`, `test`, and `release` or an equivalent clear structure.
- Dependency layers are isolated from application source layers.
- Nexus wheelhouse is prepared once and BuildKit installs offline.
- Release image excludes tests, compilers, wheelhouse, caches, and credentials.
- MLflow is included only when needed, pinned to the user-provided version, resolved from Nexus wheels, and recorded in the report.
- Oracle client and Python Oracle driver versions are pinned, validated in the final image, and recorded when Oracle is used.
- Release image size is measured; large layers and model contribution are reported.
- Models are either externalized through supported infrastructure or intentionally packaged with digest/checksum/report metadata.
- Runtime Lineage and Image Catalog record parent and final digests.
- BuildKit registry cache is separate from final Harbor image repositories.
- Only release images are pushed to deployable repositories after gates pass.
- Digest, lock hash, non-root user, latest tag ban, report, SBOM/provenance/signature rules are enforced.

## Failure Triage

Classify failures before changing code:

- Nexus slow: check wheel-only availability, download concurrency, wheelhouse reuse, and response timing.
- uv slow: check repeated sync, stale lock, cache path, workspace discovery, and whether `--no-sync` is safe.
- BuildKit slow: check dependency layer invalidation, registry cache import/export, builder CPU/memory, and `.dockerignore`.
- Harbor push slow: check image size, push concurrency, release/cache repository split, and network throughput.
- Image too large: check image category separation, CUDA devel base usage, build tools in release, wheelhouse/cache leakage, `.dockerignore`, model layer size, and unused dependencies.
- Model image rollout slow: check model size, image pull time, node cache reuse, rollout strategy, and whether model-in-image is still the right tradeoff.
- Reproducibility failure: compare runtime contract, parent digest, lock hash, Dockerfile hash, builder spec hash, source commit, and build args.
- Runtime mismatch: compare Python ABI, CUDA/framework versions, platform, and Runtime Lineage.
- Matrix mismatch: compare variant id, matrix row hash, lock file, parent digest, cache scope, and intended image category.
- MLflow mismatch: compare builder spec, lock file, Nexus wheel version, installed package metadata, and runtime category need.
- Oracle mismatch: compare builder spec, Oracle client libraries, Python driver version, runtime library path, final image contents, and target database compatibility.
- Security failure: check non-root user, latest tag, digest pinning, SBOM/provenance/signature, and credential handling.
