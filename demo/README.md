# Session 17: DevSecOps CI/CD Demo

This Flask demo runs unit tests and security checks in GitHub Actions, builds and scans a Docker image, pushes it to Docker Hub, and verifies deployment to a temporary Kubernetes Kind cluster.

## Project Contents

- `app/`: Flask application, templates, and static assets.
- `tests/`: pytest unit tests.
- `Dockerfile`: application image build.
- `../.github/workflows/devsecops.yml`: CI/CD and security pipeline (stored at the Git repository root for GitHub Actions discovery).
- `k8s/`: Kubernetes Deployment and Service manifests.
- `requirements.txt`, `requirements-dev.txt`: runtime and test dependencies.

## Prerequisites

- Python 3.12 and pip.
- Docker Engine/Desktop, running.
- Git and a GitHub repository.
- Docker Hub account and access token for publishing images.
- For manual deployment: `kubectl` and a running Kubernetes cluster such as Minikube.
- Optional local scanners: Gitleaks, pip-audit, and Trivy. CodeQL runs in GitHub Actions.
- Optional GitHub CLI (`gh`) for configuring repository secrets and viewing runs.

## Run Commands From This Project

Open a terminal at the workspace's `session17` folder and enter the project directory:

```bash
cd session17_25sept/demo
```

If your terminal starts elsewhere, use the full path to this directory.

### 1. Install and Run the Application

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt -r requirements-dev.txt
python app/app.py
```

Open `http://localhost:5001`. In another terminal, check the health endpoint:

```bash
curl --fail http://localhost:5001/health
curl --fail http://localhost:5001/api/status
```

### 2. Run Unit Tests

```bash
python -m pytest --cov=app --cov-report=term-missing
```

The command exits non-zero if tests fail. The current suite contains eight tests.

### 3. Run Dependency Scanning (SCA)

Install pip-audit into the active virtual environment and scan the pinned runtime dependencies:

```bash
python -m pip install pip-audit
pip-audit -r requirements.txt
```

An audit finding causes a non-zero exit; resolve or explicitly assess findings before publishing.

### 4. Run Secret Scanning

Install Gitleaks using its official installation instructions, then scan the repository and its Git history:

```bash
gitleaks git . --redact --verbose
```

Do not commit real credentials. If a credential was committed, revoke or rotate it even after removing it from the latest file version.

### 5. Build and Run the Docker Image

```bash
docker build -t session17-python:local .
docker run --rm --name session17-python -p 5001:5001 session17-python:local
```

In another terminal, verify the container responds, then stop it with `Ctrl+C` in the run terminal:

```bash
curl --fail http://localhost:5001/health
```

### 6. Scan the Container Image

Install Trivy using the official Aqua Security installation instructions, then run the same blocking gate as CI:

```bash
trivy image --exit-code 1 --severity HIGH,CRITICAL session17-python:local
```

The command fails when HIGH or CRITICAL vulnerabilities are found. Review scan results and update the base image or dependencies before retrying.

## GitHub Actions CI/CD

The workflow runs for pull requests targeting `main` and pushes to `main`. Test and security jobs run as independent gates. Docker build and image scanning wait for those checks to pass. Publishing and Kubernetes deployment run only for a push to `main`.

```text
Code -> Unit tests + CodeQL SAST + pip-audit SCA + Gitleaks
     -> Docker build -> Trivy HIGH/CRITICAL gate
     -> Push to Docker Hub -> Deploy and smoke-test in Kind
```

### Configure Docker Hub Secrets

In GitHub, open **Settings -> Secrets and variables -> Actions -> New repository secret** and add:

| Secret | Value |
| --- | --- |
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | A Docker Hub access token with permission to push the `hey-cicd` repository |

`GITHUB_TOKEN` is provided automatically by GitHub Actions. The workflow does not use a `KUBECONFIG` secret: it creates a temporary Kind cluster inside the runner to validate the deployment. This cluster is deleted when the job ends; it is not a persistent production cluster.

Alternatively, with GitHub CLI installed and authenticated, set the secrets from the project terminal. The token is read without echo and is not placed directly in shell history:

```bash
gh auth login
read -rp "Docker Hub username: " DOCKERHUB_USERNAME
gh secret set DOCKERHUB_USERNAME --body "$DOCKERHUB_USERNAME"
read -rsp "Docker Hub access token: " DOCKERHUB_TOKEN
printf '\n'
printf '%s' "$DOCKERHUB_TOKEN" | gh secret set DOCKERHUB_TOKEN
unset DOCKERHUB_TOKEN
```

### Commit and Trigger the Workflow

Check the remote and current changes before staging. Add only the files intended for this task; avoid `git add .` if the working tree contains unrelated files.

```bash
git remote -v
git status --short
git add README.md .github/workflows/devsecops.yml ../.github/workflows/devsecops.yml
git commit -m "Complete DevSecOps pipeline documentation and gates"
git push origin main
```

For a pull request, create and push a feature branch, then open a PR targeting `main`:

```bash
git switch -c devsecops-demo
git push -u origin devsecops-demo
```

Open the repository's **Actions** tab to inspect each job. A push to `main` should show successful tests, SAST, SCA, secret scan, Docker build, image scan, image push, and Kind deployment/smoke test. The image tags pushed to Docker Hub are the commit SHA and `latest`.

With GitHub CLI, inspect recent runs and logs:

```bash
gh run list --workflow devsecops.yml
gh run view RUN_ID --log
gh run view RUN_ID --web
```

Replace `RUN_ID` with the ID shown by `gh run list`.

## Deploy to Your Own Kubernetes Cluster

The workflow's Kind cluster is temporary. These commands deploy to the cluster selected by your local `kubectl` context. First publish an image to Docker Hub or use an existing accessible image. For a local push, authenticate without putting the token in command history:

```bash
read -rp "Docker Hub username: " DOCKERHUB_USERNAME
read -rsp "Docker Hub access token: " DOCKERHUB_TOKEN
printf '\n'
printf '%s' "$DOCKERHUB_TOKEN" | docker login --username "$DOCKERHUB_USERNAME" --password-stdin
IMAGE="$DOCKERHUB_USERNAME/hey-cicd:$(git rev-parse --short HEAD)"
docker build -t "$IMAGE" .
docker push "$IMAGE"
unset DOCKERHUB_TOKEN
```

Apply the manifests, point the Deployment at the image just pushed, and wait for the rollout:

```bash
kubectl config current-context
kubectl apply -f k8s/deployment.yaml
kubectl set image deployment/session17-python "session17-python=$IMAGE"
kubectl apply -f k8s/service.yaml
kubectl rollout status deployment/session17-python --timeout=120s
kubectl get deployments,pods,services
```

Access the app through a local port-forward. Keep this command running and use another terminal for the curl checks:

```bash
kubectl port-forward service/session17-python 5001:80
```

```bash
curl --fail http://localhost:5001/health
curl --fail http://localhost:5001/api/status
```

For Minikube, the NodePort service can also be opened with:

```bash
minikube service session17-python
```

Useful troubleshooting commands:

```bash
kubectl describe deployment session17-python
kubectl get pods -l app=session17-python
kubectl logs deployment/session17-python
kubectl get events --sort-by=.metadata.creationTimestamp
```

Remove the manually deployed resources when you no longer need them:

```bash
kubectl delete -f k8s/service.yaml
kubectl delete -f k8s/deployment.yaml
```

## Evidence and Screenshots

Capture real results after the workflow has completed; do not use expected output as proof of a successful run.

1. In GitHub **Actions**, capture the completed run with all required jobs green.
2. Open the run and capture the test summary and security/image scan job results.
3. In Docker Hub, capture the `hey-cicd` repository showing the commit-SHA image tag.
4. For manual Kubernetes deployment, capture `kubectl get deployments,pods,services` and the successful `/health` response.
5. Keep screenshots in the submission's requested evidence location and avoid including access tokens or other secrets.

## API Endpoints

| Method | Route | Purpose |
| --- | --- | --- |
| `GET` | `/` | Dashboard |
| `GET` | `/health` | Health check |
| `GET` | `/api/status` | Application status, uptime, and Python version |
| `GET` | `/api/greet/<name>` | Greeting endpoint |
| `POST` | `/api/add` | Add two numbers |
| `POST` | `/api/calculate` | Calculator operations |
| `POST` | `/api/pipeline/run` | Simulated pipeline endpoint |

## Security Tools

| Control | Tool | Enforcement |
| --- | --- | --- |
| Unit tests | pytest, pytest-cov | Failed tests block image build |
| SAST | GitHub CodeQL | Analysis job must pass |
| SCA | pip-audit | Audit job must pass |
| Secret scanning | Gitleaks GitHub Action | Detected secrets fail the scan job |
| Image scanning | Trivy | HIGH/CRITICAL findings return exit code 1 |
| Registry | Docker Hub | Push occurs only after build and scan pass |
| Kubernetes | Kind in CI; local cluster manually | CI verifies rollout and HTTP endpoints |