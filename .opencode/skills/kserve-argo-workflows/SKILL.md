---
name: kserve-argo-workflows
description: Use when creating, reviewing, or debugging KServe 0.15 model serving projects that use Argo Workflows, including Kubernetes YAML, Python model server code, Docker images, Docker build speed and layer cache checks, package version checks, resource templates, RBAC, storageUri/modelFormat fields, readiness, rollout updates, and inference smoke tests.
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

## Debugging Commands

Use these commands when diagnosing deployment failures:

```bash
python --version
python -m pip check
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
