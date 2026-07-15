---
name: kserve-argo-workflows
description: Use when creating, reviewing, or debugging KServe 0.15 model serving projects that use Argo Workflows, including Kubernetes YAML, Python model server code, Docker images, BuildKit and Buildx Bake pipelines, Docker build speed and layer cache checks, uv run performance, package version checks, image digest/catalog checks, RBAC, storageUri/modelFormat fields, readiness, rollout updates, and inference smoke tests.
---

# KServe 0.15 With Argo Workflows

Use this skill for projects that serve models on KServe 0.15 and orchestrate deployment or validation through Argo Workflows.

## Version Rules

- Treat KServe as version `0.15` unless the user explicitly says otherwise.
- Prefer the KServe 0.15 archived docs when checking fields or examples: `https://kserve.github.io/archive/0.15/`.
- Use `apiVersion: serving.kserve.io/v1beta1` for `InferenceService`.
- Prefer the newer predictor schema:

```yaml
spec:
  predictor:
    model:
      modelFormat:
        name: sklearn
      storageUri: gs://...
```

- Avoid the old framework-specific predictor schema unless maintaining an existing manifest that already uses it.
- Do not deploy `InferenceService` objects into KServe/control-plane namespaces.

## Python And Package Checks

Do not review only YAML. For Python-based model servers or images, inspect package and runtime files before changing manifests:

- `pyproject.toml`
- `requirements.txt`
- `requirements/*.txt`
- `poetry.lock`
- `uv.lock`
- `Pipfile.lock`
- `setup.py` or `setup.cfg`
- `Dockerfile`
- inference server source files such as `model.py`, `app.py`, `predictor.py`, or `server.py`

Check these points:

- Python version in `Dockerfile`, `.python-version`, `pyproject.toml`, and CI files is consistent.
- `kserve` Python package version is pinned or intentionally ranged for KServe `0.15`.
- Model framework versions are explicit enough for reproducible builds: `torch`, `tensorflow`, `scikit-learn`, `xgboost`, `transformers`, `vllm`, `numpy`, `pandas`, and related runtime packages.
- CUDA, GPU runtime, `torch`, `vllm`, and base image tags agree when GPUs are used.
- The package set in the image matches the KServe runtime declared by `predictor.model.modelFormat.name` or custom runtime configuration.
- Dependency changes are reflected in lock files when the project uses lock files.
- Avoid unbounded upgrades. Prefer minimal pins/ranges that solve the compatibility issue.

When asked to "upgrade Python" or "bump Python", update all relevant places together:

- base image tag in `Dockerfile`
- `requires-python` or equivalent in `pyproject.toml`
- CI/test matrix
- local version files such as `.python-version`
- dependency pins that are incompatible with the new Python version

After changing Python or packages, validate with the project's existing commands when available:

```bash
python --version
python -m pip freeze
python -m pip check
python -m pytest
```

If the project uses Poetry, uv, or pip-tools, use that tool's native lock and sync commands instead of ad hoc pip installs.

## Feature Implementation Flow

When implementing a serving feature, keep the pipeline complete enough to prove the change works:

1. Update Python code and dependency declarations.
2. Update Dockerfile and `.dockerignore` if runtime files, packages, or build context changed.
3. Build and tag the serving image with an immutable tag such as Git SHA or release version; avoid relying on `latest` alone.
4. Push the image to the registry expected by the cluster.
5. Update the Argo Workflow or WorkflowTemplate so it deploys that exact image tag.
6. Apply or patch the KServe `InferenceService`.
7. Wait for Ready.
8. Run a smoke test against the expected inference endpoint.

Prefer a single Argo path for repeatable releases:

```text
build image -> push image -> apply InferenceService -> wait Ready -> smoke test
```

When the user asks for only skill changes, encode this workflow as guidance in `SKILL.md`; do not create separate workflow YAML unless explicitly requested.

## Argo BuildKit Image Pipeline

When the workflow builds images, prefer a DAG that separates preparation, quality gates, supply-chain metadata, release push, and catalog registration:

```text
validate-spec
  -> verify-parent-image
  -> render-dockerfile
  -> generate-bake-targets
  -> build-test-target
  -> unit-test + smoke-test + security-scan
  -> generate-sbom-provenance
  -> build-release-image-and-push
  -> resolve-image-digest
  -> cosign-sign-and-attest
  -> verify-signature
  -> update-image-catalog
  -> notify-build-result
```

Apply these rules:

- Build and test with BuildKit before release push; do not push a release image before unit, smoke, and security gates pass.
- If the organization uses a quarantine registry, push test images there first, then promote to the release repository after gates pass.
- Sign with cosign only after the image is pushed and the immutable registry digest is known.
- Sign and attest `repository@sha256:...`, not a mutable tag.
- Treat local BuildKit metadata digest and final registry manifest digest as different until verified.
- Use Argo `dependencies` or `depends` so quality gates block release push.
- Use `arguments.parameters`, `inputs.parameters`, and `outputs.parameters.valueFrom.path` consistently.
- Put source, wheelhouse, rendered Dockerfile, Bake file, and reports on a workflow PVC or artifact repository.
- Use Registry Cache for Docker layers; do not use PVC as a long-term BuildKit cache.

For Buildx Bake:

- Use Bake when multiple targets, platforms, Python versions, CPU/GPU variants, or service images must be managed together.
- Generate targets from the build spec and keep `docker-bake.hcl` hashable.
- Use registry cache references separate from final images, for example `harbor.example.com/build-cache/model:buildcache`.
- Prefer `cache-to=type=registry,ref=...,mode=max` so multi-stage intermediate layers can be reused.
- Limit concurrency for shared resources such as internal PyPI, BuildKit, and Harbor; more parallelism can increase failure rate.

For BuildKit execution in Kubernetes:

- Prefer remote BuildKit or rootless BuildKit instead of mounting `/var/run/docker.sock`.
- Keep Git clone, template render, wheel download, and BuildKit execution as separate concerns.
- If the BuildKit client image lacks Git or package tools, prepare files in an earlier task and pass them through PVC/artifacts.
- Use BuildKit secrets for credentials. Do not pass registry, Git, PyPI, or cosign secrets through Dockerfile `ARG` or `COPY`.

## Environment Fingerprint And Image Lineage

Before deciding that two builds used the same environment, compare more than the Dockerfile. Define an environment fingerprint from:

- platform and architecture
- parent image repository, tag, and digest
- Python, CUDA, framework, and runtime versions
- dependency lock hash
- Dockerfile template hash and rendered Dockerfile hash
- Buildx Bake definition hash
- build args and relevant environment variables
- BuildKit and Buildx versions
- source repository and commit
- build spec hash

If the fingerprint changes, call out which input changed. Do not promise identical final image digests from identical fingerprints when non-deterministic build steps remain.

Prefer image layers by responsibility:

```text
Trusted Base -> Runtime -> Framework -> Application Base -> Application Release
```

Use this split only when it reduces real rebuild time or improves shared governance. Avoid making a new base image for small project-only packages.

Track Runtime Lineage in image labels or catalog metadata:

- lineage id and version
- trusted base digest
- runtime digest
- framework digest
- application base digest
- final release digest
- source commit
- lock hash
- Dockerfile hash
- Bake definition hash
- SBOM digest
- provenance digest
- signature verification result
- workflow name and UID

Only allow approved Runtime Lineage for release builds when a catalog exists.

## Workflow Schema Repair Rules

When correcting user-provided Argo YAML or JSON, fix common malformed fields before reasoning about behavior:

```text
madata -> metadata
lables -> labels
mangedFields -> managedFields
input -> inputs
paramters / paraamters -> parameters
intiContainers -> initContainers
severAccountName -> serviceAccountName
```

Each template must contain one valid template body such as `container`, `script`, `steps`, `dag`, `resource`, or `suspend`. A template with only `name` is not executable.

When users ask for both YAML and JSON, ensure both formats describe the same `WorkflowTemplate`: same task names, templates, parameters, dependencies, volumes, secrets, and output paths.

## uv Run Performance Checks

Use this section when `uv run` feels slow in local development, Docker builds, CI, or Argo steps.

Facts to account for:

- `uv run` ensures the project environment is up-to-date before running the command.
- `uv run --no-sync ...` avoids syncing the virtual environment and implies `--frozen`.
- `uv run --frozen ...` runs without updating `uv.lock`.
- `uv run --locked ...` checks that the lockfile is up-to-date instead of silently re-locking.
- `uv cache dir` shows the cache directory; `UV_CACHE_DIR` or `--cache-dir` can pin it to a cacheable path.

Diagnose slow `uv run` by checking:

- Whether the command is repeatedly syncing `.venv`.
- Whether `uv.lock` is missing, stale, or being updated in each run.
- Whether Docker layers discard `.venv` or the uv cache between builds.
- Whether `UV_NO_CACHE`, `--no-cache`, or ephemeral cache directories are disabling reuse.
- Whether the command uses `--with` dependencies, inline script metadata, Git dependencies, or editable local packages that force extra resolution or environment work.
- Whether the project root is accidentally broad, causing uv to discover the wrong `pyproject.toml`, workspace, or `.venv`.

Recommended patterns:

- For development commands where sync is desired, keep plain `uv run`.
- For hot paths after a prior `uv sync --frozen`, prefer `uv run --no-sync <cmd>`.
- For CI where the lockfile must not change, prefer `uv run --locked <cmd>` or `uv run --frozen <cmd>` depending on whether lock freshness should be checked.
- In Docker, copy `pyproject.toml` and `uv.lock` before source files, run dependency sync in a cacheable layer, then use `uv run --no-sync` for later build/test commands that should not re-sync.
- In Argo steps, avoid installing or syncing dependencies in every short command container. Build dependencies into the image or add an explicit dependency-prep step with persistent cache only if the cluster supports it.

Example Docker shape with uv cache:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.11-slim

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN --mount=type=cache,target=/root/.cache/uv \
    uv sync --frozen --no-dev

COPY . .
CMD ["uv", "run", "--no-sync", "python", "-m", "model"]
```

Do not use `--no-sync` to hide dependency drift. First make the lockfile and environment correct, then skip sync only where repeat execution speed matters.

## Docker Build Speed And Layers

When reviewing or writing Dockerfiles for KServe Python serving images, check layer order and cache behavior. Slow builds are often caused by copying the full source tree before dependency installation.

Prefer this layer order:

1. Start from a pinned Python or runtime base image.
2. Install OS packages in one layer, clean package-manager caches in the same layer.
3. Copy only dependency declaration and lock files.
4. Install Python dependencies.
5. Copy application source and model server code last.
6. Set runtime command, user, and metadata.

For Python dependencies:

- Copy `requirements.txt`, `pyproject.toml`, `poetry.lock`, `uv.lock`, or generated lock files before `COPY . .`.
- Use BuildKit cache mounts for pip, uv, Poetry, or apt when the build environment supports it.
- Prefer wheels or prebuilt artifacts for heavy packages when builds repeatedly compile native extensions.
- Avoid reinstalling dependencies after every source-code change.
- Keep model artifacts out of the Docker image when `storageUri` or external model storage is intended.
- Use `.dockerignore` to exclude `.git`, local virtualenvs, notebooks, test outputs, caches, datasets, model binaries, and build artifacts unless the image explicitly needs them.
- Avoid `pip install` from unpinned branches or floating URLs in production serving images.

Example cache-friendly shape:

```dockerfile
# syntax=docker/dockerfile:1.7
FROM python:3.11-slim

WORKDIR /app

RUN --mount=type=cache,target=/var/cache/apt \
    apt-get update \
    && apt-get install -y --no-install-recommends build-essential \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    python -m pip install --upgrade pip \
    && python -m pip install -r requirements.txt

COPY . .

CMD ["python", "-m", "model"]
```

Do not force this exact Dockerfile. Adapt it to the project's package manager, base image, GPU runtime, and serving entrypoint.

## Argo Workflow Pattern

Use Argo `resource` templates to create, apply, patch, or delete KServe resources. Argo resource templates accept Kubernetes manifests including CRDs.

For KServe custom resources:

- Prefer `action: apply` for create-or-update deployment workflows.
- Use `action: patch` only for targeted changes, and set `mergeStrategy: merge` or `json`; do not rely on strategic merge for CRDs.
- Include `serviceAccountName` and RBAC for `inferenceservices.serving.kserve.io`.
- Add a separate readiness step with `kubectl wait` or `kubectl get` rather than relying only on Argo `successCondition` when the condition shape is uncertain.
- Add a smoke-test step after readiness when ingress details are known.

## Minimal Workflow Shape

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: deploy-kserve-
spec:
  serviceAccountName: kserve-deployer
  entrypoint: deploy
  arguments:
    parameters:
      - name: namespace
        value: kserve-test
      - name: model-name
        value: sklearn-iris
      - name: storage-uri
        value: gs://kfserving-examples/models/sklearn/1.0/model
  templates:
    - name: deploy
      steps:
        - - name: apply-isvc
            template: apply-inferenceservice
        - - name: wait-ready
            template: wait-inferenceservice

    - name: apply-inferenceservice
      resource:
        action: apply
        manifest: |
          apiVersion: serving.kserve.io/v1beta1
          kind: InferenceService
          metadata:
            name: "{{workflow.parameters.model-name}}"
            namespace: "{{workflow.parameters.namespace}}"
          spec:
            predictor:
              model:
                modelFormat:
                  name: sklearn
                storageUri: "{{workflow.parameters.storage-uri}}"

    - name: wait-inferenceservice
      container:
        image: bitnami/kubectl:latest
        command: [sh, -c]
        args:
          - |
            kubectl wait inferenceservice "{{workflow.parameters.model-name}}" \
              -n "{{workflow.parameters.namespace}}" \
              --for=condition=Ready \
              --timeout=10m
            kubectl get inferenceservice "{{workflow.parameters.model-name}}" \
              -n "{{workflow.parameters.namespace}}" \
              -o wide
```

## RBAC Shape

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kserve-deployer
  namespace: argo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kserve-deployer
  namespace: kserve-test
rules:
  - apiGroups: ["serving.kserve.io"]
    resources: ["inferenceservices"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kserve-deployer
  namespace: kserve-test
subjects:
  - kind: ServiceAccount
    name: kserve-deployer
    namespace: argo
roleRef:
  kind: Role
  name: kserve-deployer
  apiGroup: rbac.authorization.k8s.io
```

Adjust namespaces to match the cluster. If the workflow runs in the same namespace as the model, keep the ServiceAccount and RoleBinding namespace consistent.

## Review Checklist

When reviewing or generating manifests, check:

- `InferenceService` uses `serving.kserve.io/v1beta1`.
- `predictor.model.modelFormat.name` matches the runtime and model artifact.
- `storageUri` scheme is supported by the cluster and storage credentials exist.
- Namespace is not a control-plane namespace.
- Argo Workflow service account has RBAC in the target model namespace.
- CRD patches use `merge` or `json`, not strategic merge.
- Workflow has explicit readiness validation and a smoke-test path when possible.
- GPU resources, node selectors, tolerations, and runtime class are included only when the model/runtime needs them.
- Python version and dependency pins are checked against the serving image and KServe 0.15 target.
- Package lock files are updated when dependency declarations change.
- Dockerfile copies dependency files before source files so dependency layers are cacheable.
- `.dockerignore` excludes large or volatile files that break cache or bloat the serving image.
- BuildKit cache mounts or wheel caching are considered for slow dependency installs.
- `uv run` hot paths avoid unnecessary sync only after lockfile/environment correctness is established.
- CI and Docker builds use `uv.lock`, `--locked`, `--frozen`, or `--no-sync` intentionally instead of accidentally re-resolving each command.
- BuildKit pipelines keep release push after quality gates and cosign signing after registry digest resolution.
- Image catalog records final image digest, parent digest, source commit, lock hash, Dockerfile hash, Bake hash, SBOM/provenance digests, and signature verification.
- PVC/artifacts are used for workflow files and reports; Registry Cache is used for Docker layer reuse.

## Debugging Commands

Use these commands when diagnosing deployment failures:

```bash
python --version
python -m pip check
uv cache dir
uv run --locked python --version
docker buildx bake --print
kubectl get inferenceservice -n <namespace>
kubectl describe inferenceservice <name> -n <namespace>
kubectl get pods -n <namespace>
kubectl logs -n <namespace> <pod-name>
argo get <workflow-name> -n <argo-namespace>
argo logs <workflow-name> -n <argo-namespace>
```

For ingress smoke tests, first read the service hostname:

```bash
kubectl get inferenceservice <name> -n <namespace> -o jsonpath='{.status.url}'
```
