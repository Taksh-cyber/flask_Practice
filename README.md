# Flask Practice App — CI/CD Pipelines

This repository contains a simple Flask web application with two parallel CI/CD
implementations:

1. **Jenkins pipeline** — `Jenkinsfile` (root of repo)
2. **GitHub Actions pipeline** — `.github/workflows/ci-cd.yml`

Both pipelines perform the same logical flow — **install dependencies → run
tests → build → deploy** — using the tooling native to each platform.

---

## 1. Jenkins CI/CD Pipeline

### Prerequisites

- A Jenkins server (VM, Docker container, or cloud-hosted e.g. an EC2 instance
  or Jenkins-as-a-service offering).
- Plugins installed on Jenkins:
  - `Git plugin`
  - `Pipeline` (installed by default on modern Jenkins)
  - `Email Extension Plugin` (or use the built-in `Mailer` plugin used by this
    Jenkinsfile's `mail` step)
  - `GitHub plugin` (for webhook-triggered builds)
- Python 3.x and `python3-venv` installed on the Jenkins agent that will run
  the build (`sudo apt-get install python3 python3-venv python3-pip` on
  Debian/Ubuntu agents).
- `rsync` installed on the agent (used by the Deploy stage).
- The Jenkins service account has passwordless `sudo` rights for the specific
  commands used in the Deploy stage (copying to `/opt/staging`, restarting the
  process), or the Deploy stage is adapted to your actual staging target.

### Setup steps

1. **Fork** this repository: `https://github.com/mohanDevOps-arch/flask_Practice.git`
2. **Clone your fork** onto (or make it reachable from) the Jenkins server:
   ```bash
   git clone https://github.com/<your-username>/flask_Practice.git
   ```
3. **Create the pipeline job** in Jenkins:
   - New Item → Pipeline (or "Multibranch Pipeline" if you want automatic
     per-branch jobs).
   - Under "Pipeline" → "Definition", choose **Pipeline script from SCM**.
   - SCM: Git. Repository URL: your fork's URL. Branch: `*/main`.
   - Script Path: `Jenkinsfile` (default, already correct since it's in the
     repo root).
4. **Configure the email recipient**:
   - Either edit the `EMAIL_RECIPIENTS` value in the `environment {}` block of
     the `Jenkinsfile`, or set it as a Jenkins job/global environment variable
     of the same name (preferred, avoids hardcoding in source control).
   - Configure outbound SMTP under *Manage Jenkins → System → E-mail
     Notification* (or *Extended E-mail Notification* if using that plugin).
5. **Configure the GitHub webhook trigger**:
   - On GitHub: repo → Settings → Webhooks → Add webhook →
     Payload URL: `http://<your-jenkins-host>/github-webhook/`, content type
     `application/json`, event: "Just the push event".
   - On Jenkins: in the job configuration, under "Build Triggers", enable
     **"GitHub hook trigger for GITScm polling"**.
   - As a fallback (e.g. if Jenkins isn't publicly reachable for a webhook),
     the `Jenkinsfile` also polls SCM every 5 minutes (`pollSCM('H/5 * * * *')`).

### Pipeline stages

| Stage | What it does |
|---|---|
| **Checkout** | Pulls the latest commit and records the short SHA for notifications. |
| **Build** | Creates an isolated `venv` and installs dependencies from `requirements.txt` (falls back to `flask`/`pytest` if the file is missing). |
| **Test** | Runs `pytest` with JUnit XML + coverage output; results are published to the Jenkins build via the `junit` step. |
| **Deploy to Staging** | Only runs if tests passed **and** the branch is `main`. Syncs the app to `/opt/staging/flask-practice-app` and (re)starts the Flask process on port `8000`. |

### Notifications

The `post { success {...} failure {...} }` block in the `Jenkinsfile` emails
`EMAIL_RECIPIENTS` on every build with the job name, build number, commit SHA,
and a link to the console output.

### Screenshots to capture for submission

- Jenkins job overview page showing a green (successful) build.
- The "Stage View" showing Checkout → Build → Test → Deploy as separate green
  boxes.
- Console output showing `pytest` results.
- The email notification received on success (and optionally on a
  deliberately-failed build).

---

## 2. GitHub Actions CI/CD Pipeline

### Prerequisites

- A GitHub repository with `main` and `staging` branches.
- Nothing else needs to be installed locally — GitHub Actions runners are
  fully managed (`ubuntu-latest`).

### Setup steps

1. Ensure the workflow file exists at `.github/workflows/ci-cd.yml` (already
   included in this repo).
2. Create the `staging` branch if it doesn't exist:
   ```bash
   git checkout -b staging
   git push -u origin staging
   ```
3. **Configure repository secrets** (Settings → Secrets and variables →
   Actions → New repository secret):

   | Secret | Used for |
   |---|---|
   | `STAGING_HOST` | Hostname/IP of the staging server |
   | `STAGING_USER` | SSH user for the staging server |
   | `STAGING_SSH_KEY` | Private key for SSH access to the staging server |
   | `PROD_HOST` | Hostname/IP of the production server |
   | `PROD_USER` | SSH user for the production server |
   | `PROD_SSH_KEY` | Private key for SSH access to the production server |

4. (Optional but recommended) Create GitHub **Environments** named `staging`
   and `production` (Settings → Environments) so you can add required
   reviewers / protection rules before a production deploy runs.

### Workflow jobs

| Job | Trigger | What it does |
|---|---|---|
| `install-and-test` | Every push/PR to `main` or `staging` | Installs dependencies, runs `pytest`, uploads test results as a build artifact. |
| `build` | After `install-and-test` succeeds | Packages the app into a zip artifact tagged with the commit SHA. |
| `deploy-staging` | Push to `staging` (after build succeeds) | Copies the build artifact to the staging host over SSH/SCP and restarts the service. |
| `deploy-production` | A GitHub Release is published (after build succeeds) | Copies the build artifact to the production host over SSH/SCP and restarts the service. |

### How the triggers work

- Pushing to `main` or `staging`, or opening a PR against either, runs
  `install-and-test` and `build`.
- Pushing to `staging` additionally runs `deploy-staging`.
- Publishing a **Release** (with a tag, e.g. `v1.0.0`) runs `deploy-production`
  — production deploys are never triggered by a plain push, only by a tagged
  release, so you always know exactly what's live.

### Screenshots to capture for submission

- The **Actions** tab showing a full green run with all four jobs (or three,
  if that run wasn't a release) completed.
- The expanded log for `install-and-test` showing the pytest output.
- The expanded log for `build` showing the packaged artifact.
- The expanded log for `deploy-staging` (from a push to `staging`) and, if
  possible, `deploy-production` (from a published release).

---

## Repository layout

```
.
├── Jenkinsfile                     # Jenkins pipeline definition
├── .github/
│   └── workflows/
│       └── ci-cd.yml                # GitHub Actions pipeline definition
├── app.py                            # Flask application entrypoint (from base repo)
├── requirements.txt                  # Python dependencies (from base repo)
├── tests/                            # pytest test suite (from base repo)
└── README.md                         # this file
```

## Notes / assumptions

- **This app requires MongoDB.** `test_app.py` connects to `mongodb://localhost:27017/test_student_db`,
  and `app.py` requires a `MONGO_URI` in its environment (via a `.env` file) to start at all.
  - **Jenkins:** if Jenkins runs in a container, run it with `--network host` (or otherwise
    ensure the Jenkins agent can reach `localhost:27017`) and run a `mongo:6` container
    published on that same host port. See setup steps above.
  - **GitHub Actions:** the workflow spins up a `mongodb` service container automatically
    for the `install-and-test` job — no extra setup needed there.
- Both pipelines assume the entrypoint is `python3 app.py` (this app hardcodes
  `host="0.0.0.0", port=5000` inside its `__main__` block rather than reading CLI flags —
  don't expect `--port` overrides to do anything unless `app.py` itself is changed to read
  `os.environ` for these values).
- The Deploy steps in both pipelines target a generic Linux host reachable via
  SSH/sudo. Swap these for your actual target (Docker registry + container
  host, PaaS like Heroku/Render, Kubernetes, etc.) as needed — the
  install/test/build stages stay the same regardless of deploy target.
