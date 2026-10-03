# OrbitOps — Cloud Operations & DevOps Automation Lab

OrbitOps is a responsive Astro website paired with a local, demonstrable CI/CD lab. It uses Jenkins to orchestrate builds, Docker to package and run the website, Ansible to deploy the image, and Portainer to inspect containers.

> **Local lab security note:** This setup mounts the host Docker socket into Jenkins and runs Jenkins as root so it can build and deploy containers on your own development machine. Docker socket access is effectively administrator-level access to the host. Use this configuration only on a trusted personal/college machine, never expose Jenkins or Portainer directly to the public internet, and do not reuse these settings for production.

## Architecture

1. Edit the site in VS Code and push commits to a Git repository.
2. Jenkins checks out the repository and runs `npm ci` and `npm run build`.
3. Jenkins builds a versioned Docker image.
4. Ansible uses the Docker API to replace the local application container.
5. Jenkins checks the deployed site at `http://host.docker.internal:8080`.
6. Portainer displays the containers, logs and resource usage.

## Prerequisites (Windows)

- Git for Windows
- Node.js 22 LTS (for local development)
- VS Code
- Docker Desktop with the WSL 2 backend enabled
- A GitHub account/repository for the Jenkins-from-SCM workflow

## A. Run the website by itself

In the VS Code terminal, from the repository root:

```bash
npm ci
npm run dev
```

Open <http://localhost:4321>. Stop with `Ctrl+C`.

## B. Start the DevOps lab

Open a terminal in the repository root (the folder containing `compose.yaml`). Start Docker Desktop and wait until its engine is running.

```bash
docker compose up -d --build
docker compose ps
```

This starts:

- Jenkins: <http://localhost:8081>
- Portainer: <http://localhost:9000>

On first Jenkins startup, retrieve the initial administrator password:

```bash
docker exec orbitops-jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Paste it into the Jenkins setup wizard, install the suggested plugins (including Pipeline, Git and Workspace Cleanup), and create an administrator account. Keep the password private.

On first Portainer startup, create its administrator account. Select the local Docker environment when prompted.

## C. Connect your repository to Jenkins

1. Push this project to a GitHub repository you own.
2. In Jenkins, choose **New Item → Pipeline** and give it a name such as `orbitops-ci-cd`.
3. Under Pipeline, choose **Pipeline script from SCM**, SCM **Git**, and enter your repository URL and branch (`*/main`, or your actual default branch).
4. Save and select **Build Now**. For a private repository, configure credentials in Jenkins rather than embedding a token in the URL.
5. The first run checks out the repository, installs dependencies, builds the site, and creates `orbitops-web:build-N` plus `orbitops-web:latest` images.
6. The `DEPLOY` parameter defaults to true for this local lab. After a successful run, open <http://localhost:8080>.

For automatic builds on every push, configure a GitHub webhook pointing to `http(s)://<reachable-jenkins-host>/github-webhook/` and enable the GitHub hook trigger in the Jenkins job. A localhost Jenkins instance is not reachable by GitHub; use polling or a secure tunnel only if you understand the security implications.

## D. Verify the deployment

```bash
docker ps
docker image ls orbitops-web
docker logs orbitops_web
```

Open <http://localhost:8080>. In Portainer, inspect `orbitops_web`, its health status, logs and resource usage. The Jenkins console output shows each pipeline stage and the final health-check result.

## E. Useful commands

```bash
# List lab services
docker compose ps

# Follow Jenkins logs
docker compose logs -f jenkins

# Stop Jenkins and Portainer (data volumes are retained)
docker compose down

# Stop and remove the lab AND its saved Jenkins/Portainer data (destructive)
docker compose down -v

# Stop the deployed website container
docker stop orbitops_web
```

## Troubleshooting

- **Port is already allocated:** change the left-side port in `compose.yaml` or set `JENKINS_HTTP_PORT` / `PORTAINER_HTTP_PORT` in a `.env` file. For the app port, update `APP_PORT` in the Jenkinsfile and use the same port when opening the site.
- **Docker engine unavailable:** start Docker Desktop and wait for “Engine running”, then retry `docker compose up -d --build`.
- **Jenkins cannot run Docker:** confirm the Docker socket mount is present and recreate the Jenkins container after changing Compose configuration.
- **Jenkins cannot find the pipeline file:** verify the configured repository and branch contain `Jenkinsfile` at the repository root.
- **Jenkins build fails on npm:** inspect the stage's Console Output; confirm the lockfile is committed and compatible with the Node version in the Dockerfile.
- **App health check fails:** inspect `docker logs orbitops_web` and verify port 8080 is free.

## Project structure

```text
src/                    Astro pages, components, layout and styles
public/                 Static assets
ansible/deploy.yml      Ansible container deployment playbook
ansible/inventory.ini   Local lab inventory
jenkins/Dockerfile      Jenkins image with Docker CLI and Ansible
compose.yaml            Jenkins + Portainer lab services
Dockerfile              Multi-stage website image (Node build, NGINX runtime)
Jenkinsfile             CI/CD pipeline definition
.env.example            Optional local port settings template
```

## Scope and production considerations

This repository provides a working local learning environment. The local Ansible inventory deploys to the Docker host via the mounted Docker socket; it does not simulate a separate remote server. For production, use a secured Jenkins agent, least-privilege credentials, a private image registry, TLS/reverse proxy, remote deployment inventory, backups, and a secrets manager. Do not expose the Docker socket or unauthenticated management interfaces.
