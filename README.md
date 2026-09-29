# SudoCloud — CI/CD + GitOps + Argo CD Pipeline

A complete, production-style GitOps pipeline: GitHub Actions builds and pushes a Docker image, updates a separate GitOps repo, and Argo CD automatically syncs the change to Kubernetes. No `kubectl apply` in CI — Git is the single source of truth.

```
Developer → GitHub → GitHub Actions (self-hosted runner)
              │
              ├─► Docker Build/Push ─► Docker Hub
              └─► Update GitOps repo (deployment.yaml)
                        │
                        ▼
                     Argo CD ─► Kubernetes ─► SudoCloud Pods
```

## Stack

`GitHub` · `GitHub Actions` (self-hosted runner) · `Docker` / `Docker Hub` · `Kubernetes` · `Argo CD` · `GitOps`

## Repositories

| Repo | Contents |
|---|---|
| `YOUR-GITHUB-USERNAME/sudocloud` | App source, `Dockerfile`, `.github/workflows/main.yml` |
| `YOUR-GITHUB-USERNAME/sudocloud-gitops` (private) | `deployment.yaml`, `service.yaml` — desired Kubernetes state |

## Prerequisites

- GitHub account + two repos above
- Docker Hub account (`your-dockerhub-username/sudocloud`)
- A Kubernetes cluster + `kubectl` access
- Argo CD installed on the cluster
- A Linux host for the self-hosted GitHub Actions runner (`your-runner-host`, user `your-runner-user`)

---

## 1. Clone the GitOps repo

```bash
mkdir -p ~/argo_cd && cd ~/argo_cd
git clone git@github.com:YOUR-GITHUB-USERNAME/sudocloud-gitops.git
cd ~/argo_cd/sudocloud-gitops
```

## 2. Create the namespace

```bash
kubectl create namespace sudocloud-argocd
kubectl get namespace
```

## 3. Docker Hub access token

Docker Hub → Account Settings → Personal access tokens → create a token. Don't use your account password.

```bash
docker login   # username: your-dockerhub-username, password: <token>
docker info
```

## 4. Kubernetes image pull secret

```bash
kubectl create secret docker-registry dockerhub-secret \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=your-dockerhub-username \
  --docker-password='<DOCKER_HUB_ACCESS_TOKEN>' \
  -n sudocloud-argocd

kubectl get secrets -n sudocloud-argocd
```

## 5. Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

## 6. Build & push the initial image

```bash
docker build -t your-dockerhub-username/sudocloud:latest_v1 .
docker images
docker push your-dockerhub-username/sudocloud:latest_v1
```

## 7. `deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sudocloud
  namespace: sudocloud-argocd
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sudocloud
  template:
    metadata:
      labels:
        app: sudocloud
    spec:
      imagePullSecrets:
        - name: dockerhub-secret
      containers:
        - name: sudocloud
          image: your-dockerhub-username/sudocloud:latest_v1
          ports:
            - containerPort: 80   # change to your app's actual port
```

## 8. `service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sudocloud
  namespace: sudocloud-argocd
spec:
  type: NodePort
  selector:
    app: sudocloud
  ports:
    - port: 80
      targetPort: 80
```

## 9. Test the manifests manually

```bash
cd ~/argo_cd/sudocloud-gitops
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

kubectl get deployment -n sudocloud-argocd
kubectl get pods -n sudocloud-argocd
kubectl get svc -n sudocloud-argocd
```

## 10. Argo CD SSH key (read-only)

```bash
cd ~/argo_cd
ssh-keygen -t ed25519 -C "argocd-sudocloud-gitops" -f argocd-sudocloud
cat ~/argo_cd/argocd-sudocloud.pub
```

Add as a **Deploy key** on `sudocloud-gitops` → Settings → Deploy keys:
- Title: `Argo CD - SudoCloud`
- Allow write access: **unchecked**

## 11. Connect the GitOps repo to Argo CD

Argo CD → Settings → Repositories → Connect Repo (SSH)

| Field | Value |
|---|---|
| Name | `sudocloud-gitops` |
| Project | `default` |
| Repository URL | `git@github.com:YOUR-GITHUB-USERNAME/sudocloud-gitops.git` |
| SSH Private Key | contents of `argocd-sudocloud` |

## 12. Create the Argo CD Application

Applications → New App

| Field | Value |
|---|---|
| Application Name | `sudocloud` |
| Project | `default` |
| Repository URL | `git@github.com:YOUR-GITHUB-USERNAME/sudocloud-gitops.git` |
| Revision | `main` |
| Path | `.` |
| Cluster | `https://kubernetes.default.svc` |
| Namespace | `sudocloud-argocd` |
| Automated Sync | ON |
| Prune | ON |
| Self Heal | ON |

```bash
argocd app get sudocloud
```

## 13. Self-hosted GitHub Actions runner

`sudocloud` repo → Settings → Actions → Runners → New self-hosted runner (Linux/x64), then run the commands GitHub gives you.

```bash
mkdir -p ~/actions-runner && cd ~/actions-runner
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

Verify Docker access as the runner user:

```bash
docker ps
docker version
```

## 14. GitHub environment + secrets

`sudocloud` repo → Settings → Environments → New environment: **`Argo_app-docker`**

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | `your-dockerhub-username` |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `GITOPS_SSH_KEY` | GitHub Actions private SSH key (step 15) |

## 15. GitHub Actions SSH key (read + write)

```bash
cd ~/argo_cd
ssh-keygen -t ed25519 -C "github-actions-sudocloud-gitops"
cat ~/.ssh/id_ed25519.pub
```

Add as a **Deploy key** on `sudocloud-gitops`:
- Title: `GitHub Actions - SudoCloud`
- Allow write access: **checked**

Then add the private key as the `GITOPS_SSH_KEY` secret:

```bash
cat ~/.ssh/id_ed25519
```

### SSH key summary

| Key | Used by | Permission |
|---|---|---|
| `Argo CD - SudoCloud` | Argo CD | Read only |
| `GitHub Actions - SudoCloud` | GitHub Actions | Read + Write |

Never reuse the same key for both.

## 16. Workflow — `.github/workflows/main.yml`

```yaml
name: SudoCloud CI/CD

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: self-hosted
    environment: Argo_app-docker

    env:
      IMAGE_NAME: your-dockerhub-username/sudocloud
      GITOPS_REPO: YOUR-GITHUB-USERNAME/sudocloud-gitops
      GITOPS_BRANCH: main

    steps:

      - name: Checkout application repository
        uses: actions/checkout@v4

      - name: Set image tag
        run: |
          IMAGE_TAG="${GITHUB_SHA::7}"
          echo "IMAGE_TAG=$IMAGE_TAG" >> "$GITHUB_ENV"
          echo "Docker image: $IMAGE_NAME:$IMAGE_TAG"

      - name: Verify Docker Hub secrets
        run: |
          if [ -z "${{ secrets.DOCKERHUB_USERNAME }}" ]; then
            echo "ERROR: DOCKERHUB_USERNAME is empty"
            exit 1
          fi
          if [ -z "${{ secrets.DOCKERHUB_TOKEN }}" ]; then
            echo "ERROR: DOCKERHUB_TOKEN is empty"
            exit 1
          fi

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build Docker image
        run: |
          docker build \
            -t "$IMAGE_NAME:$IMAGE_TAG" \
            -t "$IMAGE_NAME:latest" \
            .

      - name: Push Docker image
        run: |
          docker push "$IMAGE_NAME:$IMAGE_TAG"
          docker push "$IMAGE_NAME:latest"

      - name: Checkout GitOps repository
        uses: actions/checkout@v4
        with:
          repository: ${{ env.GITOPS_REPO }}
          ref: ${{ env.GITOPS_BRANCH }}
          ssh-key: ${{ secrets.GITOPS_SSH_KEY }}
          path: gitops
          persist-credentials: false

      - name: Update Kubernetes deployment
        run: |
          cd gitops
          grep "image:" deployment.yaml
          sed -i \
            "s|image: .*|image: $IMAGE_NAME:$IMAGE_TAG|" \
            deployment.yaml
          grep "image:" deployment.yaml

      - name: Commit GitOps change
        run: |
          cd gitops
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add deployment.yaml
          if git diff --cached --quiet; then
            echo "No changes to commit."
            exit 0
          fi
          git commit -m "Update SudoCloud image to $IMAGE_TAG"

      - name: Push GitOps change
        run: |
          cd gitops
          git push origin "$GITOPS_BRANCH"
```

> This workflow never runs `kubectl apply`. It only updates the GitOps repo — Argo CD does the actual deployment.

Commit and push it:

```bash
git add .github/workflows/main.yml
git commit -m "Add SudoCloud CI/CD workflow"
git push origin main
```

---

## Verification commands

```bash
# All resources in the namespace
kubectl get all -n sudocloud-argocd

# Currently running image
kubectl get deployment sudocloud -n sudocloud-argocd \
  -o jsonpath='{.spec.template.spec.containers[0].image}'

# Pods / deployment / service
kubectl get pods -n sudocloud-argocd
kubectl get deployment -n sudocloud-argocd
kubectl get svc -n sudocloud-argocd

# Argo CD app status
argocd app get sudocloud

# GitOps repo state
cd ~/argo_cd/sudocloud-gitops
grep "image:" deployment.yaml
git log --oneline --decorate --graph
```

## Debugging

```bash
# Pod not running
kubectl describe pod <POD_NAME> -n sudocloud-argocd
kubectl logs <POD_NAME> -n sudocloud-argocd

# Deployment issues
kubectl describe deployment sudocloud -n sudocloud-argocd

# Check the secret exists (ImagePullBackOff)
kubectl get secret dockerhub-secret -n sudocloud-argocd

# Runner status
cd ~/actions-runner
sudo ./svc.sh status
sudo ./svc.sh restart
```

| Problem | Check |
|---|---|
| Docker login failed | `DOCKERHUB_USERNAME=your-dockerhub-username`, using access token, not password |
| Secret is empty | Secrets exist under the `Argo_app-docker` environment, not repo-level |
| Actions can't push GitOps | `GITOPS_SSH_KEY` matches the deploy key with **write** enabled |
| Argo CD can't access GitHub | Argo CD deploy key is **read-only**; URL is `git@github.com:YOUR-GITHUB-USERNAME/sudocloud-gitops.git` |
| `ImagePullBackOff` | `dockerhub-secret` exists in the namespace; `imagePullSecrets` set in the Deployment |
| Workflow YAML error | File is plain YAML — no ```` ```yaml ```` fences pasted into the actual file |
| Workflow doesn't start | Push is going to the `main` branch |

## Security

Never commit SSH private keys, Docker Hub tokens, GitHub secrets, or passwords to any repository, image, or README. Use GitHub **Environment Secrets** only.

---

## License

MIT (or your preferred license)
