 DevOps Assignment 05 & 06 — React App CI/CD Pipeline

A complete DevOps pipeline: a React application containerized with a multi-stage
Docker build, infrastructure provisioned with Terraform, configured with Ansible,
and deployed automatically through a Jenkins CI/CD pipeline with SonarQube
code-quality scanning.

---

 Repository Layout

 react-docker-app and pipeline-config` also exist as **separate,standalone GitHub
 repositories hussainsafdar/react-docker-app` and
 hussainsafdar/pipeline-config` since the Jenkins pipeline demonstrates a
multi-SCM checkout** by pulling both independently. Keep both copies in
sync when making changes to either.

---

 Assignment 05 — Ansible Roles & Docker Multi-Stage Builds

| Requirement | Status | Where |
|---|---|---|
| Node.js installed via Ansible role (local) |  | `ansible-nodejs/roles/nodejs`, run via `playbook.yml` |
| Multi-stage Docker build (Node + Nginx) |  | `react-docker-app/Dockerfile` |
| React app containerized |  | `react-docker-app/` |
| Image pushed to Docker Hub | | `mhussain8621/react-docker-app` |
| App accessible locally on a port | | `docker run -p 3000:80 ...` → `localhost:3000` |
| EC2 created via Terraform (minimal instance) |  | `terraform-ec2/main.tf` — `t3.micro` |
| Apache installed + port changed to 81 via Ansible |  | `ansible-nodejs/roles/apache_setup`, run via `playbook_ec2.yml` |

### How to run it

```bash
 1. Install Node.js locally via Ansible
cd ansible-nodejs
ansible-playbook -i inventory.ini playbook.yml

 2. Build and run the React app in Docker
cd ../react-docker-app
docker build -t react-docker-app .
docker run -d -p 3000:80 --name react-app react-docker-app
 → http://localhost:3000

 3. Push to Docker Hub
docker tag react-docker-app mhussain8621/react-docker-app:latest
docker push mhussain8621/react-docker-app:latest

 4. Provision EC2 with Terraform
cd ../terraform-ec2
terraform init
terraform apply
terraform output   # note the public IP

 5. Configure Apache on EC2 via Ansible (port 81)
cd ../ansible-nodejs
ansible-playbook -i inventory_ec2.ini playbook_ec2.yml
 → http://<EC2-Public-IP>:81
```

 Why a multi-stage Dockerfile?

A multi-stage build keeps the final image small and secure. The first stage
uses a full `node:18-alpine` image to install dependencies and compile the
React app (`npm run build`), but none of that — Node.js itself, `node_modules`,
dev dependencies, or the source code — is needed to actually *serve* the app.
The second stage starts from a clean, lightweight `nginx:alpine` image and
copies in only the compiled static output (`build/`). The result is a much
smaller final image, a reduced attack surface (no build tooling or source
code shipped to production), and faster, cheaper deployments.

---

 Assignment 06 — Jenkins CI/CD Pipeline & Code Quality

| Requirement | Status | Notes |
|---|---|---|
| Multi-SCM checkout | ✅ | Checks out `react-docker-app` and `pipeline-config` separately |
| Worker node/agent | ✅ | `agent { label 'worker' }`, running on `worker-node-1` |
| Git/SCM plugins installed | ✅ | |
| Code checked out and scanned with SonarQube | ✅ | `sonar-scanner` stage, `sonar.sources=src` |
| Docker build + deploy | ✅ | Multi-stage image built, pushed to Docker Hub |
| Deploy to remote location | ✅ | SSH deploy to EC2, running on port 8082 |
| Success/Failure notification | ⚠️ | `emailext` implemented in `post {}`; SMTP delivery currently fails from within the Jenkins container (see Known Limitations) |

 Pipeline flow

1. Checkout Multiple SCM** — pulls `react-docker-app` and `pipeline-config` into separate workspace subfolders.
2. SonarQube Analysis** — scans `react-docker-app/src` and uploads results to SonarQube.
3. Build Docker Image** — multi-stage build, tagged `jenkins-<BUILD_NUMBER>`.
4. Push to Docker Hub** — pushes the tagged image to `mhussain8621/react-docker-app`.
5. Deploy to Remote EC2** — SSHes into the EC2 instance, pulls the new image, and restarts the `react-app-live` container on port 8082.
6. Post actions** — logs success/failure to the console and attempts an email notification.

 How to trigger it

Open Jenkins (`http://localhost:8081`) → select the pipeline job → **Build Now**.
Console output shows each stage; the final app is reachable at:

---

 Known Limitations

- SonarQube JavaScript analysis** — the scanner reports
  `Error when running: 'node -v'. Is Node.js available during analysis?`
  because Node.js isn't installed inside the SonarScanner's execution
  environment. The scan still completes successfully overall, but JS-specific
  rules aren't fully applied. Fix: install Node.js on the Jenkins agent/worker
  image used for the SonarQube stage.
- Email notifications** — `emailext` is wired up in the `post {}` block with
  Gmail SMTP credentials configured in Jenkins, but delivery currently fails
  with `SMTP connection error` — the Jenkins container isn't able to reach
  `smtp.gmail.com:465` from inside its network. The pipeline still completes
  successfully (`Finished: SUCCESS`); only the email step fails.

---

 Environment

- Docker Hub image:** `mhussain8621/react-docker-app`
- EC2 public IP:** `98.90.194.84`
- EC2 instance type:** `t3.micro`
- Ports:** `81` (Apache/Nginx via Ansible), `8082` (Jenkins-deployed React app), `3000` (local Docker run)
