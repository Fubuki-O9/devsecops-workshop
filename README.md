# DevSecOps CI/CD Pipeline

A hands-on **DevSecOps** project: a containerized Node.js service shipped through a
fully automated CI/CD pipeline that **builds, security-scans, publishes, and deploys**
to Kubernetes on every push to `main`. The security scan is a hard gate — a container
image with known **CRITICAL/HIGH** vulnerabilities never reaches the registry or the cluster.

![Node.js](https://img.shields.io/badge/Node.js-18-339933?logo=node.js&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-containerized-2496ED?logo=docker&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-image%20scan-1904DA?logo=aquasecurity&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-deploy-326CE5?logo=kubernetes&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?logo=githubactions&logoColor=white)

---

## What this demonstrates

- **CI/CD automation** with GitHub Actions on a self-hosted runner.
- **Security-shifted-left**: Trivy scans the built image and **fails the pipeline** on
  fixable CRITICAL/HIGH CVEs before anything is published or deployed.
- **Semantic versioning** derived automatically from commit messages
  (`feat:` → minor, `BREAKING CHANGE` → major).
- **Container publishing** to the GitHub Container Registry (GHCR).
- **Kubernetes deployment** with a versioned image, rollout wait, and hardened
  pod security context (non-root, read-only root filesystem, dropped capabilities).

---

## Pipeline flow

```
 push to main
      │
      ▼
┌──────────────┐   ┌───────────────┐   ┌──────────────────────┐   ┌──────────────┐
│ Semantic     │──▶│ Docker build  │──▶│ Trivy scan (gate)    │──▶│ Push to GHCR │
│ version      │   │ (app/)        │   │ CRITICAL/HIGH = fail │   │ :ver, :latest│
└──────────────┘   └───────────────┘   └──────────────────────┘   └──────┬───────┘
                                                                          │
                                                   ┌──────────────────────┘
                                                   ▼
                              ┌──────────────┐   ┌──────────────────────┐   ┌──────────────┐
                              │ Create GHCR  │──▶│ kubectl apply k8s/   │──▶│ Set image +  │
                              │ pull secret  │   │ (Deployment/Service) │   │ wait rollout │
                              └──────────────┘   └──────────────────────┘   └──────────────┘
```

If the Trivy step finds a fixable CRITICAL/HIGH vulnerability, the pipeline stops
there — no push, no deploy.

---

## Repository structure

```
.
├── app/                    # the application + its image
│   ├── index.js            # minimal Express service (listens on :3000)
│   ├── package.json
│   └── Dockerfile          # node:18-alpine, runs as non-root
├── k8s/                    # Kubernetes manifests
│   ├── deployment.yaml     # 2 replicas, probes, hardened securityContext
│   └── service.yaml        # ClusterIP service :80 -> :3000
└── .github/workflows/
    └── cicd.yaml           # the DevSecOps pipeline
```

---

## The application

A deliberately small service — the focus of this project is the **pipeline**, not the app.

```js
const express = require('express');
const app = express();
app.get('/', (req, res) => res.send('DevSecOps Workshop Working!'));
app.listen(3000, () => console.log('Running on port 3000'));
```

---

## Running it

### Run the app locally

```bash
cd app
npm install
node index.js
# open http://localhost:3000
```

### Build and run the container

```bash
docker build -t devsecops-app -f app/Dockerfile app/
docker run -p 3000:3000 devsecops-app
```

### Deploy to Kubernetes manually

```bash
kubectl apply -f k8s/
kubectl rollout status deployment/devsecops-app
kubectl get pods,svc -l app=devsecops-app
```

---

## Pipeline setup

The workflow runs on a **self-hosted runner** that has `docker` and `kubectl`
available and a kubeconfig pointing at your cluster.

**Required repository secret**

| Secret     | Purpose                                                              |
| ---------- | ------------------------------------------------------------------- |
| `GHCR_PAT` | A GitHub Personal Access Token with `write:packages` scope, used to push images to GHCR and create the in-cluster image-pull secret. |

**Trigger:** every push to `main` runs the full build → scan → push → deploy flow.

> Replace `fubuki-o9` in `cicd.yaml` and `k8s/deployment.yaml` with your own
> GHCR namespace if you fork this.

---

## Security notes

- **Image scanning** — Trivy runs on the freshly built image; `ignore-unfixed: true`
  means the build only fails on vulnerabilities that actually have a fix available,
  which keeps the gate meaningful rather than noisy.
- **Least privilege at runtime** — the Deployment runs the container as a non-root
  user with `readOnlyRootFilesystem`, `allowPrivilegeEscalation: false`, and all
  Linux capabilities dropped.
- **No secrets in the image** — registry credentials live in GitHub Actions secrets
  and an in-cluster `docker-registry` secret, never baked into the image.

---

## Credits

The pipeline design was built as a DevSecOps learning exercise, inspired by the
OpenShift/ACS DevSecOps workshop materials at
<https://devsecops-workshop.github.io/>.
