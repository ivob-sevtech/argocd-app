# ArgoCD + GitHub Actions: A Working GitOps CI/CD Pipeline

A corrected, verified walkthrough for building a Docker image with GitHub Actions, pushing it to GHCR, and having ArgoCD deploy it to Kubernetes.

Adapted from [Mehmet Kanus's article](https://medium.com/@mehmetkanus17/argocd-github-actions-a-complete-gitops-ci-cd-workflow-for-kubernetes-applications-ed2f91d37641) (30 June 2025). The original's structure is sound but several steps don't work as published. Every command and manifest below was run end to end on Docker Desktop Kubernetes v1.34.1 (arm64) with ArgoCD v3.5.1.

**Reference implementation:** [`ivob-sevtech/argocd-app`](https://github.com/ivob-sevtech/argocd-app) (source + CI) and [`ivob-sevtech/argocd-deploy`](https://github.com/ivob-sevtech/argocd-deploy) (manifests). Substitute your own throughout.

---

## Changes from the original article

| # | Original | Problem | Fix |
|---|---|---|---|
| 1 | GHCR login via `secrets.CR_PAT` | A PAT needs `write:packages`; the article never says so. No `permissions:` block, so the job token is read-only regardless. | Drop the PAT. Use `GITHUB_TOKEN` with `permissions: packages: write`. |
| 2 | `git clone https://$PAT@github.com/...` | Supplies a *username* with no password. Git prompts for one and dies: `could not read Password`. | `https://x-access-token:$PAT@github.com/...` |
| 3 | Secret interpolated inline as `${{ secrets.GITOPS_PAT }}` | Splices the token into the command text, where it surfaces in logs and error messages. | Pass via `env:`, reference as `$GITOPS_PAT`. |
| 4 | Single-arch build | Produces `linux/amd64` only. Fatal on arm64 nodes (Apple silicon, Graviton, Pi): `no matching manifest for linux/arm64/v8`. | Add `setup-qemu-action` and `platforms: linux/amd64,linux/arm64`. |
| 5 | Step 4 creates an `imagePullSecret` | Only needed for private packages. With a public one it references a missing secret and emits a misleading `FailedToRetrieveImagePullSecret` warning. | Omit for public packages. |
| 6 | `deployment.yaml` pins `:latest` | Breaks GitOps — a new image under the same tag leaves the manifest unchanged, so ArgoCD sees no diff and never rolls the pod. | Pin the commit SHA. CI rewrites it each build. |
| 7 | No guard on an empty secret | A missing `GITOPS_PAT` fails deep inside git with an opaque message. | Fail fast with a clear error. |
| 8 | No no-op guard | Re-running on an unchanged SHA fails the job at `git commit` with "nothing to commit". | Exit cleanly when there's no diff. |
| 9 | `checkout@v3`, `login-action@v2`, `buildx@v2`, `build-push@v4` | Node 20 deprecation warnings on current runners. | Still functional; bump when convenient. |

---

## Architecture

```
you push code
      │
      ▼
┌─────────────────┐
│  argocd-app     │  Actions builds the image, pushes to ghcr.io
└────────┬────────┘
         │  rewrites the image tag in deployment.yaml, commits   ← the handoff
         ▼
┌─────────────────┐
│  argocd-deploy  │  desired cluster state
└────────┬────────┘
         │  ArgoCD polls, detects drift
         ▼
    your cluster    pod rolls to the new image
```

The two repos never talk directly. GitHub Actions can't reach your cluster; ArgoCD can't see your CI. They communicate through a git commit.

**Why two repos rather than one:** if CI committed the image tag back into the app repo, that push would retrigger the build, which would commit again — forever. Separate repos make the loop structurally impossible. You also get a clean deploy history (`git log` in the deploy repo is a record of every deploy, and `git revert` is a rollback).

## Prerequisites

- A running Kubernetes cluster
- Helm
- `kubectl` configured against the cluster
- Two GitHub repositories — one for source, one for manifests

**Repos can be public.** The original requires private repos and therefore an imagePullSecret. Public is simpler for learning; the private path is noted in Step 4.

## Step 1 — Install ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd
```

```yaml
# argocd-values.yaml
configs:
  cm:
    admin.enabled: true
    timeout.reconciliation: 15s
    timeout.hard.reconciliation: 0s
```

```bash
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd -f argocd-values.yaml
```

`timeout.reconciliation: 15s` overrides the 180s default, so deploys land in seconds rather than minutes. In production you'd instead add a GitHub webhook to `argocd-server` and sync on push.

Wait for all pods to be ready:

```bash
kubectl get pods -n argocd -w
```

## Step 2 — Access the dashboard

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open <https://localhost:8080> and accept the self-signed certificate. Log in as `admin`.

> Port-forwarding opens a tunnel from your machine through the API server to the pod. Nothing is exposed to your network. The pair is always `LOCAL:REMOTE`.

## Step 3 — The GitHub Actions workflow

`.github/workflows/build_and_deploy.yml` in the **app** repo:

```yaml
name: ArgoCD App

on:
  push:
    branches:
      - main

env:
  IMAGE_NAME: ghcr.io/${{ github.repository }}/app
  GITOPS_REPO: ivob-sevtech/argocd-deploy    # <-- your deploy repo

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout source code
        uses: actions/checkout@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          platforms: linux/amd64,linux/arm64
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Update GitOps deployment repo
        env:
          GITOPS_PAT: ${{ secrets.GITOPS_PAT }}
        run: |
          if [ -z "$GITOPS_PAT" ]; then
            echo "::error::GITOPS_PAT secret is empty. Add it under Settings > Secrets and variables > Actions > Secrets."
            exit 1
          fi

          git clone "https://x-access-token:${GITOPS_PAT}@github.com/${GITOPS_REPO}.git" gitops
          cd gitops

          # Write the current image tag
          sed -i "s|image: ghcr.io/.*/.*:.*|image: ${IMAGE_NAME}:${GITHUB_SHA}|g" deployment.yaml

          if git diff --quiet -- deployment.yaml; then
            echo "deployment.yaml already up to date; nothing to push."
            exit 0
          fi

          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"

          git add deployment.yaml
          git commit -m "Update image to ${GITHUB_SHA}"
          git push origin main
```

### 3a — GHCR authentication

No PAT required. `GITHUB_TOKEN` is minted per run and can push to packages owned by the same account, provided you grant the scope:

```yaml
permissions:
  contents: read
  packages: write
```

Omitting this block is why the original fails — the job token is read-only by default. The article's `CR_PAT` approach also needs `write:packages`, which it never mentions.

### 3b — Multi-architecture builds

`setup-qemu-action` registers binfmt handlers so the amd64 runner can execute arm64 binaries. For a `COPY`-on-`nginx` image nothing is compiled, so emulation costs seconds. Verify afterwards:

```bash
docker manifest inspect ghcr.io/OWNER/REPO/app:latest | jq '.manifests[].platform'
```

You want both `amd64` and `arm64`. The `unknown/unknown` entries are BuildKit provenance attestations — expected.

### 3c — The one secret you do need

Cross-repo pushes can't use `GITHUB_TOKEN`; it's scoped to the repo running the workflow. Create a classic PAT with the **`repo`** scope:

<https://github.com/settings/tokens/new>

**Copy the `ghp_…` value immediately — GitHub shows it exactly once.**

```bash
gh secret set GITOPS_PAT --repo OWNER/argocd-app
```

Verify it landed as a *secret*, not a variable:

```bash
gh secret list   --repo OWNER/argocd-app   # GITOPS_PAT should be here
gh variable list --repo OWNER/argocd-app   # and NOT here
```

The Variables and Secrets tabs look nearly identical, and a value on the Variables tab is only readable via `vars.NAME`. Referenced as `secrets.NAME` it expands to an empty string.

> Once working, replace this with a fine-grained PAT scoped to the deploy repo alone with Contents: Read and write. A `repo`-scoped classic PAT grants write access to every repository you own.

## Step 4 — The GitOps repo

`deployment.yaml` in the **deploy** repo:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: shopping
  name: shopping
  namespace: product
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shopping
  template:
    metadata:
      labels:
        app: shopping
    spec:
      containers:
      - image: ghcr.io/OWNER/argocd-app/app:PLACEHOLDER
        name: shopping
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: shopping
  name: shopping-svc
  namespace: product
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: shopping
```

**Mind the image path.** It has three segments, because the workflow appends `/app` to `github.repository`:

```
ghcr.io/OWNER/argocd-app/app        correct
ghcr.io/OWNER/argocd-app            wrong -> ErrImagePull: "denied denied"
```

GHCR returns `denied` for both private *and* nonexistent repositories, so a path typo looks like a permissions failure.

The `PLACEHOLDER` tag only matters for the first sync — CI overwrites the line on every build. Any hand-edit to it gets clobbered by the next push.

**No `imagePullSecrets` for a public package.** If your package is private, add:

```yaml
    spec:
      imagePullSecrets:
        - name: ghcr
```

and create the secret in the app namespace:

```bash
kubectl create secret docker-registry ghcr \
  --docker-server=ghcr.io \
  --docker-username=YOUR_GITHUB_USERNAME \
  --docker-password=A_PAT_WITH_read:packages \
  --namespace=product
```

Check visibility at `https://github.com/users/OWNER/packages/container/package/argocd-app%2Fapp`. Packages inherit the source repo's visibility on creation.

## Step 5 — Create the ArgoCD Application

Apply this with `kubectl`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: shopping
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/OWNER/argocd-deploy.git
    targetRevision: main
    path: .

  destination:
    server: https://kubernetes.default.svc
    namespace: product

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f application.yaml
```

Or via the UI — **+ NEW APP**, then:

| Field | Value |
|---|---|
| Application Name | `shopping` |
| Project Name | `default` |
| Sync Policy | **Automatic** + Prune Resources + Self Heal |
| Sync Options | ✓ Auto-Create Namespace |
| Repository URL | `https://github.com/OWNER/argocd-deploy.git` |
| Revision | `main` |
| Path | `.` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `product` |

**`repoURL` must be the deploy repo, never the app repo.** Pointing it at source code yields a green, empty Application — zero resources are trivially `Synced` and `Healthy`. The tell is a resource count of 0 and a synced revision authored by you rather than `GitHub Actions`.

**Namespace must match `metadata.namespace` in the manifests.** The manifest's own value wins when placing resources; the destination field only supplies a default for resources without one, and determines which namespace `CreateNamespace=true` creates. A mismatch means ArgoCD creates one namespace and applies into another.

Keep `application.yaml` *outside* the synced path, or ArgoCD ends up managing its own definition.

## Step 6 — Verify

```bash
kubectl get application shopping -n argocd
kubectl get pods,svc -n product
kubectl port-forward svc/shopping-svc -n product 8081:80
```

Then <http://localhost:8081>.

Push a visible change and watch the chain:

```bash
gh run watch --repo OWNER/argocd-app --exit-status          # 1. build (~60-90s with arm64)
gh api repos/OWNER/argocd-deploy/commits --jq '.[0].commit.message'   # 2. the handoff commit
kubectl get application shopping -n argocd -w              # 3. Synced -> OutOfSync -> Progressing -> Synced
kubectl get pods -n product -w                             # 4. new pod, old terminates
```

Roughly 90 seconds end to end.

Two non-bugs: the port-forward dies when the pod rolls (it's bound to that pod — restart it), and nginx sends caching headers with no `Cache-Control`, so the browser serves stale HTML. Hard-refresh, or trust `curl` over the browser.

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Error: Username and password required` | Secret empty or saved as a variable | Check `gh secret list` vs `gh variable list` |
| `denied: installation not allowed to Write organization package` | Missing `packages: write` | Add the `permissions:` block |
| `could not read Password for 'https://***@github.com'` | Token used as username, no password | Use `x-access-token:$TOKEN@` |
| `remote: Invalid username or token` | Not a valid GitHub PAT (expired, truncated, or from another service) | Regenerate; PAT values are shown only once |
| `ErrImagePull` / `denied denied` | Wrong image path, or private package with no pull secret | Check the `/app` segment; check visibility |
| `no matching manifest for linux/arm64/v8` | Single-arch image on an arm64 node | Add QEMU + `platforms:` |
| `FailedToRetrieveImagePullSecret` | `imagePullSecrets` references a missing secret | Remove it for public packages (pull still succeeds; warning is cosmetic) |
| App green but empty, `Synced 0` | `repoURL` points at the app repo | Point it at the deploy repo |
| Pod never rolls after a push | Manifest pins `:latest` — no textual diff, no new ReplicaSet | Pin the commit SHA |
| Manual `kubectl edit` reverts within seconds | `selfHeal: true` working as designed | Change git, not the cluster |
| Re-run uses old workflow code | Re-runs replay the original commit's workflow file | Push a new commit |

## Operating notes

**Rollback.** ArgoCD's **HISTORY AND ROLLBACK** rolls back to a *git revision*, not a ReplicaSet. The button is disabled while auto-sync is on — disable auto-sync, roll back, and the app sits `OutOfSync` (pinned, correctly). For a permanent rollback, `git revert` in the deploy repo instead.

**ReplicaSets accumulate**, one per image tag, scaled to zero. They cost nothing and make rollback a scale operation rather than a rebuild. Kubernetes keeps 10 (`revisionHistoryLimit`).

**Exposing the app.** On Docker Desktop, `type: LoadBalancer` binds to `localhost` and removes the need for a port-forward — far simpler than an ingress controller for local work. Make the change in git; `selfHeal` reverts anything else.

## Known gaps

- **Base image pulled anonymously from Docker Hub.** Shared runner IPs hit rate limits; expect intermittent `toomanyrequests`. Authenticate to Docker Hub or mirror the base image into GHCR.
- **`FROM nginx:latest` is a floating tag.** Two builds of the same commit can differ. Pin a digest for reproducibility.
- **Action versions** emit Node 20 deprecation warnings.
- **`repo`-scoped classic PAT** is broader than needed; a fine-grained PAT limited to the deploy repo is the tighter fit.
