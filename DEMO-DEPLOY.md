# Starbucks clone – demo deployment runbook

## What changed vs. the original repo
| File | Change |
|---|---|
| `Dockerfile` | Multi-stage: builds the React app, serves it with nginx on port 3000 (~50 MB image instead of ~600 MB running the dev server). |
| `Dockerfile.dev` | The original `npm start` Dockerfile, kept for reference. |
| `nginx.conf` | SPA fallback so `/dashboard`, `/pay`, etc. work on refresh. |
| `.dockerignore` | Keeps `node_modules`/`build` out of the image context. |
| `Jenkinsfile` | The README pipeline, with your repo/image as variables and a redeploy-safe final stage. |

## Option A – 2-minute demo (any machine with Docker)
```bash
docker build -t starbucks .
docker run -d --name starbucks -p 3000:3000 starbucks
# open http://localhost:3000
```

## Option B – Full DevSecOps demo on AWS EC2
1. **Launch EC2**: Ubuntu 22.04/24.04, **t2.large** (Jenkins + SonarQube need ~8 GB RAM), 30 GB disk.
   Security group inbound: 22 (SSH), 8080 (Jenkins), 9000 (SonarQube), 3000 (app).
2. **SSH in** and run the install blocks in `README.md`: AWS CLI, Jenkins, Docker, Trivy, Docker Scout.
   Then also: `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins`
3. **SonarQube**: `docker run -d --name sonar -p 9000:9000 sonarqube:lts-community`
   → http://<EC2-IP>:9000 (admin/admin) → My Account → Security → generate a token.
4. **Jenkins** at http://<EC2-IP>:8080 (`sudo cat /var/lib/jenkins/secrets/initialAdminPassword`):
   - Plugins: Eclipse Temurin installer, SonarQube Scanner, NodeJS, OWASP Dependency-Check, Docker, Docker Pipeline, Email Extension.
   - Tools (Manage Jenkins → Tools): JDK `jdk17`, NodeJS `node16`, SonarQube Scanner `sonar-scanner`, Dependency-Check `DP-Check`, Docker `docker` — names must match the Jenkinsfile.
   - Credentials: `Sonar-token` (secret text), `docker` (Docker Hub username/password or access token).
   - System: SonarQube server named `sonar-server` → http://<EC2-IP>:9000. In SonarQube add a webhook → `http://<EC2-IP>:8080/sonarqube-webhook/` (needed for the Quality Gate stage).
5. **Push this code** to your own GitHub repo and edit `GIT_REPO` and `IMAGE` at the top of `Jenkinsfile`.
6. **Create a Pipeline job** → "Pipeline script from SCM" → your repo → Build Now.
7. Open **http://<EC2-IP>:3000** – the app is live.

## Demo tips
- Show the Stage View, the SonarQube dashboard, `trivy.txt`, and the Docker Scout CVE output — that's the "SecOps" story.
- The OWASP Dependency-Check stage downloads the NVD database on first run (can take 10–20+ min). Run the pipeline once *before* the demo, or add an NVD API key.
- **Tear down afterwards**: terminate the EC2 instance so it doesn't keep billing.
