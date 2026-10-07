# StudyShala DevOps Project: Complete Guide

Goal: learn DevOps by building a full production-style platform around a real app (StudyShala). Tools covered: Git, GitHub, GitHub Actions, shell scripting, Linux, Docker, AWS, Terraform, Kubernetes, Helm, Prometheus, Grafana.

This is a plain reading document. Read the whole thing once before you start. Then work phase by phase. There is no deadline. Do not move to the next phase until the checkpoint of the current phase passes.

---

## 0. How to use this document

1. Read Sections 1 to 5 first (the big picture).
2. Do the phases (Section 6 onward) in order. Each phase has: goal, concepts, steps, checkpoint, common errors, interview questions.
3. When you get stuck, use the "Asking for help" template in Section 5. It lets any AI tool or any person help you without knowing your project.
4. Keep a file called `LEARNING-LOG.md` in your repo. After each phase write: what I built, what broke, how I fixed it. This becomes your interview stories and your README content.
5. Some commands and versions will change over time. If a command fails, check the official docs of that tool. Do not copy blindly. Understand each line.

---

## 1. What you are building

You will take the existing StudyShala application (React frontend, Node/Express backend, MongoDB) and build a complete DevOps platform around it:

- Code is managed with a clean Git/GitHub workflow.
- Every pull request is automatically linted, tested, built and scanned.
- Every merge to `main` builds Docker images and pushes them to AWS ECR.
- All AWS infrastructure is created with Terraform (no clicking in the console, except for learning in Phase 6).
- The app runs on Kubernetes (first locally with kind, then on AWS EKS).
- Traffic comes in over HTTPS through a load balancer and a real domain name.
- Prometheus and Grafana show metrics. Logs are collected. Alerts exist.
- Everything is documented so anyone can clone the repo and deploy their own copy.

The final result in one sentence for interviews:

"I built a CI/CD and infrastructure-as-code platform for a real full-stack app: Terraform provisions the AWS network and EKS cluster, GitHub Actions builds and scans Docker images, Kubernetes runs the app with autoscaling and health probes, and Prometheus/Grafana monitor it."

---

## 2. Final architecture (target)

```
Developer
   |
   v
GitHub repo --(pull request)--> GitHub Actions CI: lint, test, build, scan
   |
   |--(merge to main)--> GitHub Actions CD: build image, push to ECR
   |
   v
AWS
 +-- VPC (public + private subnets across 2 availability zones)
 |     +-- Internet Gateway, NAT Gateway
 |     +-- Application Load Balancer (public subnets, HTTPS via ACM certificate)
 |     +-- EKS cluster (worker nodes in private subnets)
 |           +-- frontend Deployment (nginx serving React build)
 |           +-- backend Deployment (Node/Express)
 |           +-- Ingress (AWS Load Balancer Controller)
 |           +-- HorizontalPodAutoscaler
 |           +-- Prometheus + Grafana (monitoring namespace)
 +-- ECR (Docker image registry)
 +-- S3 (Terraform remote state)
 +-- Route53 (DNS) + ACM (TLS certificate)
 +-- IAM roles (GitHub OIDC, node roles, service account roles)
 +-- Secrets Manager or Kubernetes Secrets

External (not on AWS): MongoDB Atlas (database), Google OAuth, Google Drive API
```

Why MongoDB Atlas and not a database on AWS: it is free at small size, it is what the app already uses, and Amazon DocumentDB costs money and is not 100 percent MongoDB compatible. You will still run MongoDB as a container locally (Docker Compose) and optionally as a StatefulSet in local Kubernetes to learn persistent storage.

---

## 3. Skills map (what you learn where)

| Skill | Phase |
|---|---|
| Git workflow, branches, pull requests, branch protection | 1 |
| Linux basics, bash scripting | 2 |
| Making an app container-ready (12-factor) | 3 |
| Docker, multi-stage builds, Docker Compose, Nginx | 4 |
| GitHub Actions CI, image scanning, caching | 5 |
| AWS core services (IAM, VPC, EC2, S3, ECR) | 6 |
| Terraform, modules, remote state, environments | 7 |
| Kubernetes basics, Helm (local cluster) | 8 |
| EKS, ingress, DNS, HTTPS, CD pipeline | 9 |
| Monitoring, logging, alerting | 10 |
| Security, cost control, backups | 11 |
| Documentation, demo, interview preparation | 12 |
| Optional advanced: GitOps with ArgoCD, canary deploys | 13 |

---

## 4. Rules that protect you (read before starting)

### 4.1 Protect your live website
- StudyShala is live at studyshala.dev. Never experiment on it.
- Create a separate repo, for example `studyshala-devops`, by copying or forking your code. All experiments happen there.
- Use separate credentials for experiments: a new Google OAuth client, a new Drive credential, a new MongoDB Atlas database (a different database name or a different free cluster), a new JWT secret. Never put your production MongoDB URI or Drive refresh token in this project.

### 4.2 Secrets
- Never commit secrets: `.env` files, AWS keys, JWT secrets, refresh tokens.
- Add `.env` to `.gitignore` before the first commit.
- If you ever commit a secret by mistake, assume it is leaked. Rotate it (create a new one, disable the old one). Deleting the file in a later commit is not enough because Git history keeps it.
- Your current README has example `.env` values including a Render URL. Keep only placeholders in documentation.

### 4.3 AWS cost control (very important)
- On day one: create a Billing alarm / AWS Budget at 5 USD (and another at 20 USD) with email alerts.
- Use one AWS account and one region (for example `ap-south-1` Mumbai, or whichever is closest to you and supports what you need).
- The expensive things: EKS control plane (about 0.10 USD per hour, roughly 70+ USD per month if left on), NAT Gateway (hourly charge plus data charge), load balancers (hourly charge), EC2 instances, EBS volumes, public IPv4 addresses. Prices change, so check the AWS pricing pages.
- Habit: when you finish for the day, run `terraform destroy` for expensive stacks. Because everything is code, you can recreate it in 20 to 30 minutes. That is the whole point of Infrastructure as Code.
- Before ending any AWS session, check the console for leftovers: EC2, load balancers, NAT gateways, Elastic IPs, EBS volumes, EKS clusters. Also check the Billing dashboard next morning.
- Learn locally first (Docker, kind). Only move to AWS when the local version works.

### 4.4 AWS account safety
- Turn on MFA for the root account. Never use the root account for daily work.
- Create an IAM Identity Center user or an IAM user with admin permission for learning, with MFA. Use that.
- Never put AWS access keys in code or in GitHub. In CI you will use OIDC (no stored keys), explained in Phase 6 and 9.

---

## 5. Asking for help (template for any AI tool or person)

When stuck, paste this block and fill the blanks. It gives enough context for anyone to help.

```
I am learning DevOps by building a project.
App: StudyShala. React (Vite) frontend, Node.js/Express backend, MongoDB (Atlas), Google OAuth login, Google Drive API for files.
Current phase: <phase number and name>
Tools and versions: <docker --version, terraform version, kubectl version, node --version, OS>
What I am trying to do: <one sentence>
The exact command or file I ran: <paste>
The exact error (full text): <paste>
What I already tried: <list>
Please explain what the error means first, then give the fix step by step.
```

Rules for good debugging:
1. Read the error from the bottom up. The last lines usually say the real problem.
2. Reproduce with the smallest possible case.
3. Change one thing at a time.
4. Use the tool's own debug commands (listed in each phase).
5. Write the solution into `LEARNING-LOG.md`.

---

## PHASE 0: Preparation

### Goal
Set up accounts, tools and the repo so later phases do not stop on setup problems.

### Accounts to create
- GitHub (you have one).
- AWS account (needs a card). Set billing alarms immediately.
- MongoDB Atlas (free M0 cluster).
- Docker Hub account (optional, for pulling images without rate limit problems).
- A domain name (optional until Phase 9). You can use your existing studyshala.dev domain by creating a subdomain like `devops.studyshala.dev` and delegating it to Route53 later. Or buy a cheap domain. If you do not want a domain, you can still do everything except real HTTPS; you would use the load balancer's default DNS name over HTTP.

### Tools to install (on your laptop)
- Git
- A Linux environment. If you are on Windows, install WSL2 with Ubuntu. Do all shell and DevOps work inside WSL2. This matters because most DevOps tooling and all tutorials assume Linux.
- Docker Desktop (with WSL2 integration) or Docker Engine on Linux
- Node.js (LTS version) and npm
- AWS CLI v2
- Terraform (version 1.10 or newer is recommended)
- kubectl
- kind (Kubernetes in Docker)
- Helm
- jq (JSON tool), curl, make (optional)
- VS Code with extensions: Docker, Kubernetes, HashiCorp Terraform, YAML, GitHub Actions

Verify each tool:
```
git --version
docker --version
docker run hello-world
node --version
aws --version
terraform version
kubectl version --client
kind version
helm version
```

### Repo setup
1. Create a new GitHub repo `studyshala-devops` (public if you want to share it, but only after you confirm no secrets exist).
2. Copy the app code in. Keep the three folders: `studyshala-backend`, `studyshalaFrontend`, and optionally leave out `studyshala-mobile`.
3. Do not copy `node_modules`, `logs`, `dist`, `.expo`, `android` build output.
4. Create `.gitignore` at repo root containing at least:
```
node_modules/
.env
.env.*
!.env.example
logs/
dist/
build/
*.log
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
!*.tfvars.example
.DS_Store
```
5. Create `.env.example` files with placeholder values only.
6. Create `LEARNING-LOG.md` and a `README.md` skeleton.

### AWS first-day tasks
1. Root account MFA on.
2. Create admin IAM Identity Center user or IAM user (with MFA).
3. Set budget alerts.
4. Configure the CLI: `aws configure sso` (preferred) or `aws configure`.
5. Test: `aws sts get-caller-identity`.

### Checkpoint
- All tool versions print without error.
- `aws sts get-caller-identity` shows your account.
- The repo exists on GitHub with `.gitignore` and no secrets.
- Billing alarms are set.

---

## PHASE 1: Git and GitHub workflow

### Goal
Work the way real teams work: no direct commits to main, everything through pull requests.

### Concepts
- Commit, branch, merge, rebase, conflict.
- Pull request (PR), code review, branch protection.
- Conventional commit messages (`feat:`, `fix:`, `docs:`, `chore:`, `ci:`, `infra:`).
- Git tags and releases, semantic versioning (v1.0.0).
- `.gitignore`, and how to remove an accidentally committed secret.

### Steps
1. Branch model: `main` (always deployable) plus short-lived feature branches like `feature/add-health-endpoint`. Keep it simple (trunk-based). You do not need a long-lived `dev` branch.
2. On GitHub, enable branch protection for `main`: require a pull request before merging, require status checks to pass (you will add the checks in Phase 5), disallow force pushes.
3. Add `.github/pull_request_template.md` with a checklist: what changed, how tested, screenshots, risks.
4. Add `CODEOWNERS` (optional) and `CONTRIBUTING.md` (short).
5. Install pre-commit hooks:
   - Option A: Husky + lint-staged for JS files.
   - Option B: the Python `pre-commit` tool with hooks for trailing whitespace, YAML check, and secret detection (gitleaks).
6. Practice every day: create a branch, commit small, open a PR, merge, delete the branch.
7. Practice recovery commands on a throwaway repo: `git stash`, `git reset --soft/--hard`, `git revert`, `git cherry-pick`, `git rebase -i`, `git reflog`, resolving a merge conflict.

### Useful commands
```
git switch -c feature/xyz
git add -p
git commit -m "feat: add health endpoint"
git push -u origin feature/xyz
git log --oneline --graph --all
git diff
git restore <file>
git tag v0.1.0 && git push --tags
```

### Checkpoint
- Direct push to main is blocked.
- You merged at least 3 PRs.
- A pre-commit hook blocks a fake secret you try to commit.
- You can explain merge vs rebase and when you would use each.

### Common errors
- "Updates were rejected": your branch is behind. Run `git pull --rebase`.
- Committed `.env`: remove it with `git rm --cached .env`, add to `.gitignore`, rotate the secrets.

### Interview questions
- Difference between merge and rebase?
- How do you undo a pushed commit safely? (`git revert`)
- What is branch protection and why use it?
- Trunk-based development vs Git Flow?
- A secret was pushed to GitHub. What do you do?

---

## PHASE 2: Linux and shell scripting

### Goal
Be comfortable in a terminal and automate repeated tasks with bash.

### Concepts
- Files and permissions (`chmod`, `chown`), processes (`ps`, `top`, `kill`), networking (`curl`, `ss`, `ping`, `dig`), logs (`tail -f`, `grep`, `journalctl`), `systemd` basics, environment variables, pipes and redirects, exit codes, `cron`.
- Bash: variables, conditionals, loops, functions, arguments, `set -euo pipefail`, `trap`, quoting.

### Scripts to write (put them in `scripts/`)
1. `setup.sh`: checks that required tools exist (docker, node, aws, terraform, kubectl) and prints missing ones with install hints. Exits non-zero if something is missing.
2. `health-check.sh <url>`: curls a URL, expects HTTP 200, retries a number of times with sleep, exits 0 or 1. You will reuse this in CI and deploy scripts.
3. `backup-db.sh`: runs `mongodump` against a given URI (from environment variable), compresses with timestamp, optionally uploads to S3 using AWS CLI. Add a retention option (delete backups older than N days).
4. `deploy-local.sh`: builds images and runs Docker Compose, then waits for health-check to pass.
5. `cleanup-aws.sh`: lists leftover billable resources (EC2 instances, NAT gateways, load balancers, EBS volumes, Elastic IPs) with AWS CLI so you can see what to delete. Read-only by default.
6. `log-summary.sh`: takes a log file and prints counts of ERROR and WARN lines per hour (practice with `grep`, `awk`, `sort`, `uniq -c`).

Start every script with:
```
#!/usr/bin/env bash
set -euo pipefail
```
Meaning: stop on errors, error on unset variables, fail a pipeline if any part fails.

### Extra practice
- Write a script with a `usage()` function and option parsing using `getopts`.
- Lint scripts with `shellcheck`. Add shellcheck to CI later.
- Make scripts idempotent (safe to run twice).

### Checkpoint
- All 6 scripts exist, are executable (`chmod +x`), and pass `shellcheck`.
- You can explain what `set -euo pipefail` does and what exit codes mean.

### Common errors
- "Permission denied": missing `chmod +x`.
- Windows line endings (CRLF) breaking scripts: use WSL and set `git config core.autocrlf input`.
- Unquoted variables with spaces: always write `"$var"`.

### Interview questions
- How do you find which process uses port 5000?
- What does `2>&1` mean?
- How do you make a script fail fast?
- How do you schedule a task? (cron)
- Difference between `>` and `>>`?

---

## PHASE 3: Make the app container-ready (12-factor changes)

### Goal
Small code changes so the app behaves correctly in containers and Kubernetes. These are also strong interview topics ("12-factor app").

### Changes to make in the backend
1. **Health endpoint.** Add `GET /health` returning HTTP 200 and JSON like `{ "status": "ok" }`. Better: add two endpoints.
   - `/health/live` (liveness): the process is up, no dependency checks.
   - `/health/ready` (readiness): checks MongoDB connection state (Mongoose `connection.readyState === 1`). Return 503 if not ready.
2. **Logs to stdout.** Winston currently writes to `logs/combined.log` and `logs/error.log`. In containers, log to the console (stdout/stderr) so Docker and Kubernetes can collect logs. Add a Console transport. Use JSON format in production. Disable file transports when `NODE_ENV=production` or when an env var like `LOG_TO_FILE=false`.
3. **Config only from environment variables.** No hard-coded URLs. Frontend URL, CORS origin, callback URL, Mongo URI, JWT secret all come from env. Create a validated config module that fails fast with a clear message if a required variable is missing.
4. **Graceful shutdown.** Handle `SIGTERM`: stop accepting new connections, finish current requests, close Mongo connection, then exit. Kubernetes sends SIGTERM when stopping pods.
5. **Listen on `0.0.0.0` and `process.env.PORT`.**
6. **Trust proxy.** Behind a load balancer, set `app.set('trust proxy', 1)` so secure cookies/redirects and IP logging work. Check how your OAuth callback behaves behind HTTPS proxy.
7. **Metrics endpoint (used in Phase 10).** Install `prom-client`, expose `GET /metrics` with default metrics and an HTTP request duration histogram. Do not expose `/metrics` publicly through the ingress later; keep it internal.
8. **Tests.** Add a minimal automated test setup (Jest + Supertest): test `/health/live`, and one or two API tests that do not need Google services (mock them). CI needs something to run. A few reliable tests are enough.
9. **Scripts in package.json.** Make sure `npm start` runs `node server.js` (production) and `npm run dev` uses nodemon. Add `npm test` and `npm run lint`.

### Changes to make in the frontend
1. Find which env variable the code reads: Vite uses `import.meta.env.VITE_*`. Your README mentions `REACT_APP_API_URL`, which is the Create React App style. Check the code and make the README consistent.
2. Vite env variables are **baked in at build time**. The built JavaScript contains the API URL as plain text. So in Docker you pass the URL as a build argument, not as a runtime variable. Alternative (more flexible): serve the API under the same domain via a path (`/api`) and use a relative URL `/api`, so one image works in every environment. This is the recommended approach here: frontend at `https://devops.example.com/`, backend at `https://devops.example.com/api`, routed by the ingress/Nginx. It also avoids CORS problems.
3. Add ESLint config and `npm run lint`, `npm run build`.

### Checkpoint
- `curl localhost:5000/health/live` returns 200, `/health/ready` returns 200 when Mongo is connected and 503 when it is not.
- Logs appear in the terminal as JSON when `NODE_ENV=production`.
- Sending Ctrl+C or `kill -TERM` shows a graceful shutdown log line.
- `npm test` passes. `npm run lint` passes. Frontend `npm run build` passes.

### Interview questions
- What is a 12-factor app? Name 5 factors.
- Liveness vs readiness probes?
- Why log to stdout in containers?
- What does SIGTERM do and why handle it?

---

## PHASE 4: Docker

### Goal
Package the frontend, backend and database into images and run everything with one command.

### Concepts
- Image vs container, layers, build cache, registry.
- Multi-stage builds, small base images (alpine), non-root user.
- Volumes (persistent data), networks (containers talking by service name), ports.
- `.dockerignore`, environment variables, build arguments, health checks.
- Image tags: never rely on `latest` for deployment; tag with git commit SHA and version.

### 4.1 Backend Dockerfile (`studyshala-backend/Dockerfile`)
Example, adjust to your project:
```
# ---- build/deps stage ----
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

# ---- runtime stage ----
FROM node:20-alpine
ENV NODE_ENV=production
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN addgroup -S app && adduser -S app -G app && chown -R app:app /app
USER app
EXPOSE 5000
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD wget -qO- http://localhost:5000/health/live || exit 1
CMD ["node", "server.js"]
```
Notes:
- Use the Node version your app works with. Test it.
- `npm ci` needs `package-lock.json`.
- If the backend writes watermarked or temporary files (you have `watermarkService.js` and Multer uploads), make sure the target folder is writable by the non-root user. In Kubernetes use an `emptyDir` volume for temp files.

`studyshala-backend/.dockerignore`:
```
node_modules
logs
.env
.env.*
npm-debug.log
Dockerfile
.git
```

### 4.2 Frontend Dockerfile (`studyshalaFrontend/Dockerfile`)
```
# ---- build stage ----
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL=/api
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

# ---- runtime stage ----
FROM nginx:alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
HEALTHCHECK CMD wget -qO- http://localhost/ || exit 1
```
Change `VITE_API_URL` to the exact variable name the code uses.

`studyshalaFrontend/nginx.conf`:
```
server {
  listen 80;
  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ /index.html;   # single-page-app routing
  }

  location ~* \.(js|css|png|jpg|svg|woff2)$ {
    expires 7d;
    add_header Cache-Control "public";
  }
}
```
`.dockerignore` for the frontend: `node_modules`, `dist`, `.env*`, `.git`.

Note: the `try_files ... /index.html` line is required for React Router. Without it, refreshing on `/dashboard` returns 404.

### 4.3 Docker Compose (`docker-compose.yml` at repo root)
Services:
- `mongo`: image `mongo:7`, volume `mongo-data:/data/db`, healthcheck with `mongosh --eval "db.adminCommand('ping')"`.
- `backend`: build `./studyshala-backend`, env from `.env`, `depends_on` mongo with condition `service_healthy`, port 5000.
- `frontend`: build `./studyshalaFrontend`.
- `proxy` (Nginx or Traefik): routes `/` to frontend and `/api` to backend on port 8080. This mirrors production routing.
- Optional: `mongo-express` for viewing data in development only.

Use a root `.env` file for compose (not committed) and a committed `.env.example`.

Google OAuth locally: add `http://localhost:8080/api/auth/google/callback` (whatever your proxy URL is) as an authorized redirect URI in your **test** OAuth client.

### 4.4 Commands to practice
```
docker build -t studyshala-backend:dev ./studyshala-backend
docker run --rm -p 5000:5000 --env-file .env studyshala-backend:dev
docker compose up --build -d
docker compose ps
docker compose logs -f backend
docker compose exec backend sh
docker compose down            # keep volumes
docker compose down -v         # also delete volumes (deletes DB data)
docker image ls
docker system df
docker system prune
docker history <image>
docker inspect <container>
```

### 4.5 Improve and measure
- Compare image sizes with and without multi-stage / alpine. Write the numbers in your log.
- Reorder Dockerfile lines (copy package files before source) and observe build cache effect.
- Scan the image locally: `trivy image studyshala-backend:dev`.
- Run as non-root and confirm with `docker compose exec backend whoami`.

### Checkpoint
- `docker compose up` starts everything; you can log in and use the app on localhost.
- MongoDB data survives `docker compose down` and `up` (volume works).
- Backend image runs as non-root; image size is reasonable.
- You can explain every line of both Dockerfiles.

### Common errors
- Container exits immediately: read `docker compose logs <service>`.
- Backend cannot connect to Mongo: inside compose use host `mongo` (service name), not `localhost`.
- "Cannot find module": a dependency was only in devDependencies, or `COPY` order is wrong.
- Frontend calls wrong API URL: build-time variable not passed.
- Port already in use: change the host port.
- File permission errors for uploads with non-root user.

### Interview questions
- Image vs container? Layers and caching?
- CMD vs ENTRYPOINT? COPY vs ADD?
- Why multi-stage builds?
- How do containers talk to each other in Compose?
- How do you reduce image size and attack surface?
- Where should secrets go? (not inside the image)

---

## PHASE 5: Continuous Integration with GitHub Actions

### Goal
Every pull request is automatically checked. Every merge to main produces a trusted, scanned image.

### Concepts
- Workflow, job, step, runner, trigger (`on:`), matrix, cache, artifacts, secrets, environments.
- Reusable actions (`uses:`) and pinning versions.
- OIDC authentication to AWS (no long-lived keys) comes in Phase 6/9.

### Workflows to create (`.github/workflows/`)

**1. `ci-backend.yml`** (trigger: pull_request and push to main, only when backend files change using `paths:`)
Steps: checkout, setup-node with npm cache, `npm ci`, `npm run lint`, `npm test`.

**2. `ci-frontend.yml`**
Steps: checkout, setup-node, `npm ci`, `npm run lint`, `npm run build`. Upload `dist` as an artifact (optional).

**3. `docker-build.yml`** (pull_request: build only; main: build and later push)
Steps: checkout, setup buildx, build both images with cache (`cache-from/cache-to type=gha`), run Trivy scan and fail on HIGH/CRITICAL vulnerabilities (start with report-only so you are not blocked, then tighten).

**4. `shell-and-yaml-lint.yml`**
Steps: run `shellcheck scripts/*.sh`, `yamllint`, `hadolint` for Dockerfiles, later `terraform fmt -check` and `terraform validate`, `tflint`, `tfsec` or `checkov`.

**5. `secret-scan.yml`**
Run gitleaks on every PR.

### Example skeleton (backend CI)
```
name: ci-backend
on:
  pull_request:
    paths: ["studyshala-backend/**"]
  push:
    branches: [main]
    paths: ["studyshala-backend/**"]
jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: studyshala-backend
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
          cache-dependency-path: studyshala-backend/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm test
```
Check the latest major versions of actions on their GitHub pages. Pin to a major version at least; pinning to a commit SHA is the most secure.

### Make checks required
Go back to branch protection (Phase 1) and require these workflows to pass before merging.

### Add a test database job (optional, good learning)
Use GitHub Actions `services:` to start a MongoDB container for integration tests.

### Checkpoint
- A PR with a failing test shows a red check and cannot be merged.
- A PR with passing checks can be merged.
- Build time with caching is faster than the first run (compare).
- Trivy report is visible in the logs.

### Common errors
- "Process completed with exit code 1": open the failed step and read the last lines.
- `npm ci` fails: lock file out of sync with package.json, run `npm install` locally and commit the lock file.
- Workflow does not trigger: wrong `paths:` filter or YAML indentation. YAML uses spaces, never tabs.
- Secrets are empty in PRs from forks: expected for security.

### Interview questions
- Explain your pipeline from commit to deployment.
- Difference between CI and CD? And Continuous Delivery vs Deployment?
- How do you speed up pipelines? (cache, path filters, parallel jobs)
- How do you keep secrets safe in CI?
- What does image scanning protect against?

---

## PHASE 6: AWS fundamentals (learn by hand first)

### Goal
Understand the main AWS services by using them manually once, so Terraform in Phase 7 makes sense. Terraform automates what you understand; it does not replace understanding.

### Concepts to learn
- Region, Availability Zone.
- **IAM**: users, groups, roles, policies, least privilege, trust policy vs permission policy, instance profile.
- **VPC**: CIDR block, subnets (public vs private), route tables, Internet Gateway, NAT Gateway, security groups (stateful) vs network ACLs (stateless).
- **EC2**: instance types, AMI, key pairs, user data, EBS volumes, Elastic IP.
- **S3**: buckets, versioning, encryption, bucket policies, block public access.
- **ECR**: private image registry, lifecycle policies.
- **Route53** and **ACM**: DNS records, certificates with DNS validation.
- **ALB**: listeners, target groups, health checks.
- **CloudWatch**: logs, metrics, alarms.
- **Secrets Manager / SSM Parameter Store**.

### Hands-on exercise (do this manually in the console, then delete everything)
1. Create a VPC with 2 public and 2 private subnets in 2 AZs, an Internet Gateway, one NAT Gateway, and correct route tables. Draw the diagram on paper and compare with the console "Resource map".
2. Launch one small EC2 instance (free tier eligible type if available) in a public subnet with a security group allowing SSH only from your IP and HTTP from anywhere. Use SSH or Session Manager.
3. On the instance: install Docker, pull your backend image, run it. Hit it from your browser. This is your "first manual deploy". It teaches networking and security groups.
4. Create an ECR repository. Log in with `aws ecr get-login-password | docker login ...`, tag and push your images from your laptop. Pull them on EC2.
5. Create an S3 bucket, upload a file, make a pre-signed URL, enable versioning.
6. Create an IAM role for EC2 that can read from that S3 bucket only, attach it to the instance, and verify access with `aws s3 ls` on the instance (no keys on the box).
7. Create a CloudWatch alarm on EC2 CPU.
8. **Delete everything** (instance, NAT gateway, Elastic IPs, volumes, VPC). Verify in console and billing.

### GitHub OIDC to AWS (set up now, use in Phase 9)
Goal: GitHub Actions gets temporary AWS credentials without stored keys.
- Create an IAM OIDC identity provider for `token.actions.githubusercontent.com`.
- Create an IAM role with a trust policy limited to your repo and branch, for example condition `token.actions.githubusercontent.com:sub` equals `repo:YOUR_USER/studyshala-devops:ref:refs/heads/main`.
- Attach a narrow permission policy (for example: push to your specific ECR repository only).
- In the workflow use `aws-actions/configure-aws-credentials` with `role-to-assume` and set `permissions: id-token: write`.
Later you will create this role with Terraform.

### Checkpoint
- You can draw a VPC with public/private subnets from memory and explain how a private instance reaches the internet (via NAT).
- Images are in ECR and ran on EC2.
- You can explain the difference between a security group and a network ACL, and between an IAM user and an IAM role.
- All resources from the exercise are deleted.

### Common errors
- Cannot SSH: wrong security group source IP (your IP changed), wrong key permissions (`chmod 400 key.pem`), instance in a subnet with no route to the Internet Gateway.
- `no basic auth credentials` on docker push: ECR login expired (valid 12 hours), or wrong region in the command.
- `AccessDenied`: read the error, it names the action and resource missing from your policy.
- Surprise bill: forgotten NAT Gateway or Elastic IP.

### Interview questions
- Public vs private subnet?
- What is a NAT Gateway? Why not put everything in public subnets?
- IAM role vs user vs policy?
- How would you give an app on EC2 access to S3 securely?
- Security group vs NACL?
- How does GitHub Actions authenticate to AWS without access keys?

---

## PHASE 7: Terraform (Infrastructure as Code)

### Goal
Create the entire AWS foundation from code, repeatably, and destroy it when finished.

### Concepts
- Providers, resources, data sources, variables, outputs, locals.
- State: what it is, why it must be remote and locked, why it must never be committed.
- Modules: reusable building blocks.
- `plan` vs `apply`, drift, `import`, `taint`/replace.
- Workspaces vs separate directories for environments (this project uses separate directories or tfvars files, which are easier to understand).
- Dependency graph, `depends_on`, `count`, `for_each`, `lifecycle`.

### Folder layout
```
infra/
  bootstrap/            # creates the S3 state bucket (run once, local state)
  modules/
    vpc/
    ecr/
    eks/
    iam-github-oidc/
    dns/                # Route53 zone, ACM certificate
  envs/
    dev/
      main.tf
      variables.tf
      outputs.tf
      backend.tf
      terraform.tfvars.example
    prod/               # optional, copy of dev with different values
```

### Steps
1. **Bootstrap remote state.** In `infra/bootstrap`, with local state, create an S3 bucket with versioning, encryption and public access blocked. Then in `envs/dev/backend.tf` configure the S3 backend. With Terraform 1.10 or newer you can use S3 native locking (`use_lockfile = true`). Older setups used a DynamoDB table for locking. Check the Terraform S3 backend docs for current syntax.
2. **ECR module.** Repositories for frontend and backend, image scanning on push, lifecycle policy to keep the last N images, immutable tags (optional).
3. **VPC module.** Write it yourself first (VPC, subnets in 2 AZs, IGW, one NAT gateway for cost saving, route tables, tags needed by EKS such as `kubernetes.io/role/elb` and `kubernetes.io/role/internal-elb`). Later you may compare with the community module `terraform-aws-modules/vpc/aws`. Writing it yourself once is the learning; using the community module afterwards is what many companies do.
4. **GitHub OIDC module.** Identity provider plus role for CI, least privilege.
5. **DNS module.** Route53 hosted zone for your (sub)domain, ACM certificate with DNS validation. Domain delegation (NS records) is a manual one-time step at your registrar.
6. **EKS module** (done in Phase 9 after you know Kubernetes). Can use `terraform-aws-modules/eks/aws`. You will define node groups (small instance types, 2 nodes, private subnets), the OIDC provider for the cluster, and IAM roles for service accounts.
7. Always run in this order: `terraform fmt`, `terraform validate`, `terraform plan -out=tfplan`, read the plan carefully, `terraform apply tfplan`.
8. Add Terraform checks to CI (Phase 5 style): `fmt -check`, `validate`, `tflint`, `tfsec`/`checkov`. Run `terraform plan` on pull requests (read-only role). Apply only from main, or manually at the start (safer for learning).
9. Add tags to every resource: `Project`, `Environment`, `ManagedBy = terraform`. Tags help with cost tracking and cleanup.

### Commands
```
terraform init
terraform fmt -recursive
terraform validate
terraform plan -out=tfplan
terraform apply tfplan
terraform output
terraform state list
terraform destroy
terraform import <addr> <id>
```

### Rules
- Never edit infrastructure in the console after Terraform creates it (causes drift).
- Never commit `.tfstate` or `.tfvars` with secrets.
- Do not hardcode account IDs or secrets; use variables, data sources, and AWS Secrets Manager.
- Read every `plan` before `apply`. Look for the word "destroy" and "replace".

### Checkpoint
- `terraform apply` creates VPC, ECR and OIDC role from zero, and `terraform destroy` removes them cleanly.
- State is in S3, not in your repo.
- A second person (or you in a fresh clone) can run it using only the README.
- You can explain what state is and what happens if two people apply at once (locking).

### Common errors
- "Error acquiring the state lock": a previous run crashed. Use `terraform force-unlock <ID>` only when sure nobody else is running.
- "Provider produced inconsistent result": usually eventual consistency; run again.
- Dependency cycle: two resources reference each other; restructure (often security groups).
- Destroy fails on VPC: leftover resources (ENIs from load balancers created by Kubernetes). Delete Kubernetes load balancers (Ingress/Service type LoadBalancer) **before** destroying the cluster and VPC.
- Wrong region or profile: check `AWS_PROFILE` and `AWS_REGION`.

### Interview questions
- What is Terraform state? Why remote? Why lock?
- `plan` vs `apply`? What is drift?
- Module vs resource? When to write a module?
- How do you manage dev and prod environments?
- How do you handle secrets in Terraform?
- Terraform vs CloudFormation vs Ansible?
- What happens if someone changes a resource in the console?

---

## PHASE 8: Kubernetes locally (kind) and Helm

### Goal
Learn Kubernetes properly on your own machine, for free, before using EKS.

### Concepts
- Cluster, node, control plane.
- Pod, ReplicaSet, Deployment, Service (ClusterIP, NodePort, LoadBalancer), Ingress, Namespace.
- ConfigMap, Secret, environment variables, volumes, PersistentVolume and PersistentVolumeClaim, StatefulSet.
- Probes: liveness, readiness, startup.
- Resource requests and limits.
- Rolling updates and rollbacks.
- HorizontalPodAutoscaler (needs metrics-server).
- RBAC basics, ServiceAccount.
- Labels and selectors (the glue of everything).

### Steps
1. Create a cluster: `kind create cluster --name studyshala` (use a config file to map ports 80/443 for ingress).
2. Load local images: `kind load docker-image studyshala-backend:dev --name studyshala`.
3. Write plain YAML manifests in `k8s/` first, apply with `kubectl apply -f`. Learn the objects one by one:
   - `namespace.yaml`
   - `mongo-statefulset.yaml` plus `mongo-service.yaml` plus PVC (learning persistent storage). Locally you can use Atlas instead if simpler, but do the StatefulSet once.
   - `backend-deployment.yaml`: 2 replicas, env from ConfigMap and Secret, readiness probe `/health/ready`, liveness probe `/health/live`, resource requests/limits, `securityContext` (runAsNonRoot, readOnlyRootFilesystem where possible with an `emptyDir` for temp files).
   - `backend-service.yaml` (ClusterIP).
   - `frontend-deployment.yaml` and service.
   - `configmap.yaml` for non-secret settings.
   - `secret.yaml`: create from the command line with `kubectl create secret generic ...` instead of committing a secret file. Never commit real secrets. (Later options: External Secrets Operator or Sealed Secrets.)
   - `ingress.yaml`: install ingress-nginx in kind, route `/` to frontend and `/api` to backend.
   - `hpa.yaml`: scale backend between 2 and 5 pods at 70 percent CPU. Install metrics-server (kind needs an extra flag `--kubelet-insecure-tls`).
4. Experiments (very important for learning):
   - Delete a pod: watch it come back.
   - Scale: `kubectl scale deployment backend --replicas=4`.
   - Break the readiness probe: see traffic stop reaching the pod.
   - Push a bad image tag: see `ImagePullBackOff`, then `kubectl rollout undo`.
   - Set a too-low memory limit: see `OOMKilled`.
   - Generate load (for example with `hey` or `ab`) and watch the HPA scale.
5. **Helm.** Convert your manifests to a Helm chart in `helm/studyshala/` with `values.yaml` (image tags, replicas, resources, ingress host) and templates. Learn `helm template`, `helm lint`, `helm install`, `helm upgrade`, `helm rollback`, `helm uninstall`. Use separate values files: `values-dev.yaml`, `values-prod.yaml`.
6. Optional: try Kustomize overlays and compare with Helm.

### Debug commands (memorize these)
```
kubectl get pods -A
kubectl get all -n studyshala
kubectl describe pod <pod>          # read the Events section at the bottom
kubectl logs <pod> --previous       # logs of the crashed container
kubectl logs -f deploy/backend
kubectl exec -it <pod> -- sh
kubectl port-forward svc/backend 5000:5000
kubectl get events --sort-by=.lastTimestamp
kubectl top pods
kubectl rollout status deploy/backend
kubectl rollout history deploy/backend
kubectl rollout undo deploy/backend
kubectl explain deployment.spec
kubectl config get-contexts
```

### Checkpoint
- The whole StudyShala stack runs in kind and is reachable on localhost through the ingress.
- You have done each experiment above and can explain what happened.
- The Helm chart installs the whole app with one command and a values file.
- You can explain what happens, step by step, when you run `kubectl apply` on a Deployment.

### Common errors and what they mean
- `Pending`: not enough resources, or PVC unbound. Check `describe pod`.
- `ImagePullBackOff` / `ErrImagePull`: wrong image name/tag, or image not loaded into kind, or missing registry credentials.
- `CrashLoopBackOff`: app crashes on start. Use `logs --previous`.
- `CreateContainerConfigError`: missing ConfigMap/Secret key.
- Service has no endpoints: selector labels do not match pod labels. Check `kubectl get endpoints`.
- Readiness probe failing: wrong path/port, or Mongo not reachable.

### Interview questions
- Pod vs Deployment vs StatefulSet?
- How does a Service find pods? (label selectors)
- Liveness vs readiness?
- What are requests and limits? What happens when exceeded?
- How does a rolling update work? How do you roll back?
- How does the HPA decide to scale?
- ConfigMap vs Secret? Are Secrets encrypted? (base64 only, unless encryption at rest is enabled)
- A pod is in CrashLoopBackOff. Walk me through debugging.
- What is Helm and why use it?

---

## PHASE 9: AWS EKS, HTTPS, and continuous deployment

### Goal
Run the app on a real cluster on AWS, publicly reachable over HTTPS, deployed automatically by the pipeline.

### Steps

**1. EKS with Terraform**
- Add the `eks` module to `envs/dev` (control plane, managed node group in private subnets, 2 small nodes, cluster OIDC provider, access entries for your admin user).
- Pick a Kubernetes version currently supported by EKS (check AWS docs).
- After apply, connect: `aws eks update-kubeconfig --name <cluster> --region <region>`, then `kubectl get nodes`.

**2. Cluster add-ons (install with Helm or Terraform)**
- **AWS Load Balancer Controller**: creates an ALB from Ingress objects. Needs an IAM role for its service account (IRSA or EKS Pod Identity). This is a very common interview topic.
- **metrics-server** (for HPA).
- **EBS CSI driver** (only if you run MongoDB in-cluster with persistent volumes).
- **ExternalDNS** (optional): creates Route53 records automatically from Ingress hosts.
- Alternative for TLS: ACM certificate attached to the ALB through Ingress annotations (simplest) or cert-manager with Let's Encrypt.

**3. Deploy the app with Helm**
- Image URLs now point to ECR. Nodes pull from ECR through the node IAM role.
- Ingress class `alb`, annotations for scheme `internet-facing`, target type `ip`, certificate ARN, HTTP to HTTPS redirect, health check path.
- Route `/api` to backend, `/` to frontend.
- Secrets: create Kubernetes secrets from AWS Secrets Manager (External Secrets Operator or the Secrets Store CSI driver), or, simpler for the first version, create them with `kubectl` from your machine. Document which approach you chose.
- MongoDB Atlas: allow the NAT Gateway's Elastic IP in the Atlas network access list. This teaches why private nodes have a single outbound IP.

**4. DNS and TLS**
- Point `devops.yourdomain` at the ALB (Route53 alias record, or ExternalDNS).
- Update Google OAuth **test** client redirect URI to `https://devops.yourdomain/api/auth/google/callback`.
- Confirm login works over HTTPS.

**5. CD pipeline (`.github/workflows/deploy.yml`)**
Trigger: push to main (after CI passes), or manual `workflow_dispatch` with an environment input.
Steps:
1. Assume the AWS role via OIDC.
2. Log in to ECR.
3. Build and push images tagged with the git SHA (`${{ github.sha }}`), not only `latest`.
4. Update kubeconfig for the cluster.
5. `helm upgrade --install studyshala ./helm/studyshala --set backend.image.tag=$GITHUB_SHA ... --atomic --timeout 5m` (`--atomic` rolls back automatically if the release fails).
6. Run the smoke test script `scripts/health-check.sh https://devops.yourdomain/health/ready` (adjust to your route).
7. Use a GitHub **Environment** named `production` with required reviewer for manual approval (optional but impressive).

CD role permissions: this role needs to push to ECR and access the cluster. Keep it narrow. Map the role to Kubernetes permissions through EKS access entries.

**6. Operations practice**
- Do a deploy that fails and watch `--atomic` roll back.
- Roll back a release by hand with `helm rollback`.
- Drain a node (`kubectl drain`) and watch pods reschedule.
- Test a zero-downtime deploy: run a loop calling the site while deploying and count failures.

**7. Teardown (every time)**
Order matters to avoid stuck resources:
1. `helm uninstall` the app and anything that created AWS load balancers (wait until the ALB is gone from the console).
2. `terraform destroy` the EKS stack, then network.
3. Check for leftover load balancers, security groups, ENIs, NAT gateways, volumes.
4. Run `scripts/cleanup-aws.sh`.

### Checkpoint
- Pushing code to main results in the new version live at your HTTPS URL with no manual steps (or one approval click).
- A deliberately broken release rolls back automatically.
- You can explain every hop: browser, Route53, ALB, Ingress, Service, Pod.
- Full teardown leaves no billable leftovers.

### Common errors
- Ingress gets no address: Load Balancer Controller not running or missing IAM permissions. Check its pod logs.
- ALB target unhealthy: health check path or port wrong, or security group blocks node/pod traffic.
- Nodes `NotReady` or pods cannot pull images: node role missing ECR read permission, or no route to NAT/internet.
- `Unauthorized` from kubectl in CI: role not mapped in EKS access entries.
- OAuth `redirect_uri_mismatch`: the URI in Google console must match exactly, including `https`, domain and path.
- Login loops or cookies fail: missing `trust proxy`, or HTTP/HTTPS mismatch.
- Atlas connection timeout: NAT Elastic IP not in the Atlas allowlist.
- `terraform destroy` stuck on VPC: leftover load balancer ENIs. Delete the ALB first.

### Interview questions
- Walk me through what happens from `git push` to the new version serving traffic.
- How does Kubernetes get credentials to pull from ECR?
- IRSA / Pod Identity: what problem do they solve?
- How does the AWS Load Balancer Controller work?
- How do you do zero-downtime deployments and rollbacks?
- How do you manage secrets in Kubernetes on AWS?
- Why did you use private subnets for nodes?
- Why tag images with the commit SHA?

---

## PHASE 10: Monitoring, logging and alerting

### Goal
Know what the system is doing, and get notified when it breaks.

### Concepts
- Metrics, logs, traces (the three pillars). The four golden signals: latency, traffic, errors, saturation.
- Prometheus (pull-based metrics, PromQL), Grafana (dashboards), Alertmanager.
- SLI, SLO, error budget (basic idea).

### Steps
1. Install `kube-prometheus-stack` with Helm in namespace `monitoring` (includes Prometheus, Grafana, Alertmanager, node exporter, kube-state-metrics, many default dashboards).
2. Add a `ServiceMonitor` so Prometheus scrapes the backend `/metrics` endpoint from Phase 3.
3. Build a Grafana dashboard "StudyShala" with panels: request rate, error rate (5xx), p95 latency, pod CPU/memory, pod restarts, Mongo connection state.
4. Write alert rules (PrometheusRule): high 5xx rate for 5 minutes, pod crash looping, pod not ready, high memory usage, node disk pressure.
5. Send alerts to a destination: email, Slack webhook, or Telegram bot through Alertmanager. Trigger one on purpose (kill pods, add a fake error endpoint) and see the alert arrive.
6. Logs: choose one:
   - Simple: view logs with `kubectl logs` and send container logs to CloudWatch Logs (Fluent Bit / CloudWatch Observability add-on).
   - Better learning: install Loki + Promtail (or Grafana Alloy) and view logs in Grafana next to metrics.
7. Export the Grafana dashboard JSON and alert rules into the repo under `monitoring/` so they are reproducible.
8. External check: an uptime monitor (free tier of any uptime service) or a Route53 health check with a CloudWatch alarm.
9. Cost: Prometheus/Grafana run on your nodes, so you may need slightly bigger nodes. Persistent volumes cost a little. Disable persistence for learning, document the trade-off.

### Checkpoint
- Grafana shows live app metrics and cluster health.
- You triggered a real alert and received it.
- You can answer "the site is slow, how do you find out why?" using your dashboards.

### Common errors
- Prometheus target shows DOWN: wrong ServiceMonitor labels or port name, or `/metrics` blocked by a NetworkPolicy.
- No data in Grafana panel: wrong data source or PromQL label names; test the query in Prometheus UI first.
- Prometheus pods Pending: not enough node resources or PVC problems.

### Interview questions
- Metrics vs logs vs traces?
- What are the four golden signals?
- How does Prometheus collect metrics? (pull model)
- What is a good alert? (actionable, symptom based, not noisy)
- SLI vs SLO vs SLA?
- A site is slow; how do you investigate?

---

## PHASE 11: Security, reliability and cost

### Goal
Harden the platform and show production thinking.

### Checklist
**Security**
- Image scanning in CI (Trivy) with a fail threshold; dependency scanning with `npm audit` or Dependabot enabled on the repo.
- Terraform scanning (tfsec/Checkov) in CI.
- Secret scanning (gitleaks) and GitHub secret scanning enabled.
- Containers run as non-root, read-only root filesystem where possible, drop Linux capabilities, `allowPrivilegeEscalation: false`.
- Kubernetes NetworkPolicies: backend only reachable from the ingress/frontend, Mongo only from backend.
- Namespace separation; Pod Security Standards (restricted) labels on the app namespace.
- IAM: least privilege everywhere; no wildcard `*` permissions in your own roles; separate roles for CI plan (read-only) and apply.
- Security headers (Helmet) in Express; rate limiting on the login and code-entry endpoints.
- Encrypt S3 buckets, enable ECR scan on push, encrypt Kubernetes secrets at rest (EKS supports KMS envelope encryption).
- EKS API endpoint: restrict public access to your IP, or use private endpoint plus a bastion (optional advanced).

**Reliability**
- Run at least 2 replicas across 2 AZs (use topology spread constraints / anti-affinity).
- PodDisruptionBudget for the backend.
- Resource requests and limits on every pod.
- Backups: scheduled MongoDB backup (CronJob in Kubernetes using your `backup-db.sh` logic uploading to S3). Test a restore. A backup you never restored is not a backup.
- Document a runbook: "site down", "bad deploy", "database down", "certificate expired", each with steps.
- Try a failure drill: terminate a node in the console and watch recovery.

**Cost**
- Tag everything; use AWS Cost Explorer filtered by the `Project` tag.
- Use small instance types; consider Spot instances for the node group (understand the interruption risk).
- One NAT Gateway for dev (document the trade-off vs one per AZ).
- ECR lifecycle policy; S3 lifecycle rules for backups.
- Write down the monthly cost if left running 24/7 and the cost of running only during working sessions.

### Checkpoint
- Security scans run in CI and you have fixed or consciously accepted each finding.
- A NetworkPolicy blocks a direct connection to Mongo from the wrong pod (prove it with a test pod).
- A backup was restored successfully.
- Runbook file exists.

### Interview questions
- How do you secure a Kubernetes cluster? A CI/CD pipeline? A container image?
- What is least privilege? Show an example from your project.
- What is a PodDisruptionBudget? NetworkPolicy?
- RTO and RPO? How do you back up and restore your database?
- How would you reduce this platform's cloud bill?

---

## PHASE 12: Documentation, demo and interview preparation

### Goal
Turn the work into something others can use and employers can evaluate.

### Repository README must contain
1. One-paragraph description and the architecture diagram (draw with draw.io or Excalidraw or a Mermaid diagram in the README).
2. Features list (CI, CD, IaC, K8s, monitoring, security).
3. Prerequisites and tools.
4. Quick start: local with Docker Compose (3 commands).
5. Deploy to AWS: step-by-step, including costs and teardown.
6. Repo structure explanation.
7. Configuration reference: every environment variable, with description and example.
8. Screenshots: Grafana dashboard, GitHub Actions green pipeline, running app, `kubectl get pods`.
9. "Decisions and trade-offs" section (why EKS, why Atlas, why one NAT, why Helm).
10. "Problems I solved" section (from your LEARNING-LOG).
11. Troubleshooting and runbooks links.
12. License (MIT) and contributing guide.

### Other documents
- `docs/architecture.md`, `docs/runbook.md`, `docs/cost.md`, `docs/security.md`, `docs/decisions/` (short ADRs: one file per major decision, with context, decision, consequences).
- A 3 to 5 minute demo video or GIF: push a change, watch the pipeline, see it live, show monitoring.
- A blog post (LinkedIn or dev.to) telling the story.
- Tag a release `v1.0.0`.

### Make it reusable for others
- All environment-specific values in variables and example files (`terraform.tfvars.example`, `values-example.yaml`, `.env.example`).
- A `Makefile` with targets like `make up`, `make test`, `make plan`, `make deploy`, `make destroy`.
- Test the README on a fresh clone (or ask a friend) and fix every place it fails.

### Resume bullets (adapt to your real results)
- Built an end-to-end CI/CD and IaC platform for a production full-stack app: Terraform-provisioned AWS VPC, ECR and EKS; GitHub Actions pipelines with OIDC authentication to AWS and Trivy image scanning; Helm-based deployments with automatic rollback.
- Containerized a React/Node/MongoDB application with multi-stage Docker builds, reducing image size by X percent.
- Implemented observability with Prometheus, Grafana and Alertmanager (latency, error rate, pod health alerts).
- Hardened the platform with least-privilege IAM, Kubernetes NetworkPolicies, non-root containers and automated secret and vulnerability scanning.
- Reduced cloud cost by Y percent through right-sizing and scheduled teardown.
Only write numbers you actually measured.

### Interview preparation
- Be able to draw the architecture on a whiteboard and explain each arrow.
- Prepare 5 stories from your LEARNING-LOG in this format: situation, what broke, how I debugged, what I changed, what I learned.
- Know every file in your repo. If you copied something from an AI tool or a tutorial, make sure you understand it line by line; interviewers ask "why did you do it this way?"
- Practice answering the "Interview questions" listed in each phase out loud.
- Be honest about what is learning-scale (single environment, small traffic) versus production-scale. Say what you would add next (multi-account setup, service mesh, GitOps, load testing, disaster recovery in another region).

---

## PHASE 13 (optional advanced extensions)

Pick one or two after everything else works.
1. **GitOps with ArgoCD**: ArgoCD watches a Git repo of manifests and syncs the cluster. CI only updates the image tag in Git. Learn the pull-based deployment model.
2. **Progressive delivery**: canary or blue/green with Argo Rollouts.
3. **Cluster autoscaling**: Karpenter or Cluster Autoscaler.
4. **Secrets management**: External Secrets Operator with AWS Secrets Manager.
5. **Policy as code**: OPA Gatekeeper or Kyverno (for example, block images from unknown registries).
6. **Load testing**: k6 test in a pipeline stage; watch HPA and dashboards.
7. **Cost tooling**: Infracost in pull requests showing monthly cost change of Terraform changes.
8. **Multi-environment**: separate dev and prod stacks, promotion from dev to prod with approvals.
9. **Tracing**: OpenTelemetry plus Tempo or Jaeger.
10. **Ansible**: configure the EC2 version of the app with Ansible roles (adds a configuration-management tool to your resume).

---

## 14. Suggested order and how long it may take

Time depends on your pace. Rough effort (full focus days):

| Phase | Effort |
|---|---|
| 0 Preparation | 1 day |
| 1 Git/GitHub | 1 day |
| 2 Shell | 1 to 2 days |
| 3 App changes | 1 to 2 days |
| 4 Docker | 2 to 3 days |
| 5 CI | 2 days |
| 6 AWS basics | 2 to 3 days |
| 7 Terraform | 3 to 4 days |
| 8 Kubernetes + Helm | 4 to 6 days |
| 9 EKS + CD | 3 to 5 days |
| 10 Monitoring | 2 to 3 days |
| 11 Security/reliability | 2 to 3 days |
| 12 Docs/polish | 2 days |

Total: roughly 4 to 6 weeks at a relaxed pace. A shorter version (phases 0 to 5, then 8 locally, then 12) already gives a strong project without spending on AWS EKS.

### Minimum viable version (if you need to show results early)
Phases 0, 1, 3, 4, 5, then 7 (VPC + ECR only) and 8 (local Kubernetes). Add EKS later.

---

## 15. Final repository structure (target)

```
studyshala-devops/
├── README.md
├── LEARNING-LOG.md
├── Makefile
├── docker-compose.yml
├── .env.example
├── .gitignore
├── .github/
│   ├── workflows/
│   │   ├── ci-backend.yml
│   │   ├── ci-frontend.yml
│   │   ├── docker-build.yml
│   │   ├── lint.yml
│   │   ├── secret-scan.yml
│   │   ├── terraform-plan.yml
│   │   └── deploy.yml
│   └── pull_request_template.md
├── studyshala-backend/        (+ Dockerfile, .dockerignore, tests)
├── studyshalaFrontend/        (+ Dockerfile, nginx.conf, .dockerignore)
├── scripts/
│   ├── setup.sh
│   ├── health-check.sh
│   ├── backup-db.sh
│   ├── deploy-local.sh
│   ├── cleanup-aws.sh
│   └── log-summary.sh
├── infra/
│   ├── bootstrap/
│   ├── modules/ (vpc, ecr, eks, iam-github-oidc, dns)
│   └── envs/ (dev, prod)
├── k8s/                       (plain manifests used while learning)
├── helm/studyshala/           (Chart.yaml, values.yaml, templates/)
├── monitoring/                (Helm values, dashboards, alert rules)
└── docs/
    ├── architecture.md
    ├── runbook.md
    ├── security.md
    ├── cost.md
    └── decisions/
```

---

## 16. Glossary (quick reference)

- **CI**: Continuous Integration. Automatically build and test every change.
- **CD**: Continuous Delivery/Deployment. Automatically release tested changes.
- **IaC**: Infrastructure as Code. Infrastructure described in files and created by tools.
- **Idempotent**: running it many times gives the same result as running it once.
- **Immutable infrastructure**: replace servers/containers instead of changing them in place.
- **Image**: packaged filesystem and app. **Container**: a running image.
- **Registry**: storage for images (ECR, Docker Hub).
- **Pod**: smallest Kubernetes unit, one or more containers sharing network.
- **Ingress**: rules routing external HTTP(S) traffic to services.
- **Helm chart**: package of Kubernetes templates plus values.
- **OIDC**: OpenID Connect; lets GitHub prove its identity to AWS without stored keys.
- **IRSA / Pod Identity**: gives a Kubernetes service account an AWS IAM role.
- **NAT Gateway**: lets private subnets reach the internet without being reachable from it.
- **Drift**: real infrastructure differs from what Terraform state/code says.
- **SLI/SLO/SLA**: measurement / internal target / customer promise.
- **RTO/RPO**: how fast you must recover / how much data loss is acceptable.
- **12-factor app**: rules for apps that run well in cloud/containers (config in env, logs to stdout, stateless processes, and more).
- **GitOps**: Git is the source of truth; an agent in the cluster syncs to it.

---

## 17. Weekly self-check questions

After each phase, answer without looking:
1. What problem does this phase's tool solve?
2. What would happen if I did not use it?
3. What did I break on purpose, and what did I learn?
4. Can I redo this phase from an empty folder in under an hour?
5. Can I explain it to a beginner in 2 minutes?

If you answer "no" to the last two, repeat the phase's core exercise before moving on.

---

## 18. Where to read official docs when stuck

- Git: git-scm.com/doc
- GitHub Actions: docs.github.com/actions
- Docker: docs.docker.com
- AWS: docs.aws.amazon.com (and the AWS pricing pages)
- Terraform: developer.hashicorp.com/terraform and registry.terraform.io (provider docs)
- Kubernetes: kubernetes.io/docs
- Helm: helm.sh/docs
- Prometheus: prometheus.io/docs
- Grafana: grafana.com/docs
- EKS best practices guide: aws.github.io/aws-eks-best-practices

Start every troubleshooting session with the official docs page for the exact command or resource that failed, then ask an AI tool with the template from Section 5.

---

End of guide. Start with Phase 0, and write your first entry in LEARNING-LOG.md today.
