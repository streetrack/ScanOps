![scanops](https://i.imgur.com/tcFhk5o.png)

## Project Overview

ScanOps is an automated container security pipeline that builds, scans, and promotes Docker images based on their vulnerability status. Leveraging GitHub Actions **(CI/CD)**, Trivy, AWS ECR, and S3, it ensures only safe containers are pushed to production while automatically generating SBOMs **(Software Bill of Materials)** and posting alerts for vulnerable builds.

## Architecture

```scss
GitHub Actions
   ├── Build & Push → AWS ECR (docker-dev)
   └── Trivy Scan
         ├── If clean → ECR (docker-prod)
         ├── If vulnerable → ECR (docker-quarantine)
         ├── Generate SBOM (CycloneDX)
         └── Upload Scan Results & SBOM → S3
             └── Slack Alert (if vulnerable)
```

- **ECR Repos**:
  - `docker-dev`: Initial staging
  - `docker-prod`: Production-ready images
  - `docker-quarantine`: Vulnerable images
- **Trivy Scanning**: Checks for CRITICAL and HIGH vulnerabilities
- **Slack Alerts**: Summarized scan results for visibility

## Versions

| Component | Version | Component | Version |
|-----------|---------|-----------|---------|
| Python	  | 3.11.10 | Ansible	  | Latest  |
| AWS CLI	  | Latest  | Trivy	    | 0.60.0  |
| Flask	    | 3.1.0   | Node.js	  | 20.14.0 |
| Nginx	    | 1.26.3  | Alpine    |	3.16.0  |

## Prerequisites

- **Rocky Linux VM**
  - Fresh installation of Rocky Linux
  - Allocate sufficient resources: **2 CPUs, 4GB RAM**
- **AWS Account**
  - An AWS account with provisioned access-key and secret-key
- **GitHub Account**
  - A GitHub account with an existing repository and **GitHub Actions enabled**
  - **SSH key configured and authorized**:
    - SSH key pair generated on the Rocky Linux VM
    - Public key added to your GitHub account under **SSH and GPG keys**
    - Allows passwordless `git pull` and `git push` access
- **Slack Incoming Webhook URL**
  - A Slack Incoming Webhook URL for sending alerts or messages

## Environment Setup

### Fork this repository
> 🔁 Click the “Fork” button on the top-right of the GitHub repo page to copy it to your own GitHub account. This ensures that GitHub Actions will run under your account and can access your secrets.

### Clone your forked repo
```bash
git clone git@github.com:<your-github-username>/scanops.git
cd scanops
```

### Configure Variables and Repo Secrets
**Update variable file for Ansible playbook runs**:
```bash
cp vars.yaml.example vars.yaml
vim vars.yaml
```
> - Update the values in `vars.yaml` with your actual AWS and Slack credentials
> - Keep the sensitive file local. Be sure it's listed in `.gitignore` to avoid commiting sensitive info.
<br>

**Configure GitHub Secrets for `CI/CD`**: 
| Secret Name           | Description                     |
|-----------------------|---------------------------------|
| AWS_ACCOUNT_ID        | Your AWS account ID             |
| AWS_ACCESS_KEY_ID     | IAM access key ID               |
| AWS_SECRET_ACCESS_KEY | IAM secret access key           |
| AWS_REGION            | AWS region (e.g., `us-east-1`)  |
| SLACK_WEBHOOK_URL     | Slack webhook for notifications |
> To add these:
> - Go to your repository on GitHub
> - Click Settings → Secrets and variables → Actions
> - Click "New repository secret" and enter the name and value

These secrets are accessed by the GitHub Actions workflow to authenticate with AWS and send alerts to Slack

### Run Ansible setup playbook
```bash
ansible-playbook infra.yaml -vv
```
This playbook will:
- Installs system packages and dependencies
- Installs AWS CLI if not present
- Configures AWS CLI with provided credentials
- Setup ECR repositories (`docker-dev`, `docker-prod`, `docker-quarantine`)
- Create S3 bucket for SBOMs and scan results

**Confirm Successful Execution:**
```bash
ansible --version
podman --version
aws configure list
aws sts get-caller-identity
aws ecr describe-repositories --no-cli-pager
aws s3 ls
```

<details close>
  <summary> <h4>Image Results</h4> </summary>
    
![CVEDataLake](https://i.imgur.com/idwIvVZ.png)
![CVEDataLake](https://i.imgur.com/fWI7OLO.png)

  - **List S3**: Bucket contains results under athena-results/, including .csv and .csv.metadata files
  - **List local directory**: Confirmed `~/CVEDataLake/query_results/` has multiple JSON query result files
  - **Examine JSON file**: Results confirm properly formatted structured JSON data
</details>

## Deployment

### Create Sample Docker Apps 
```bash
ansible-playbook setup_docker_apps.yaml -vv
```
This playbook generates:
- 4 sample apps:
  - Nginx
  - Alpine
  - Node.js
  - Python (Flask)
- Corresponding Dockerfiles and placeholder app files

**Confirm Successful Execution:**
```bash
cat docker/nginx/Dockerfile
cat docker/alpine/Dockerfile
cat docker/nodejs/Dockerfile
cat docker/python/Dockerfile
```

<details close>
  <summary> <h4>Image Results</h4> </summary>
    
![CVEDataLake](https://i.imgur.com/idwIvVZ.png)
![CVEDataLake](https://i.imgur.com/fWI7OLO.png)

  - **List S3**: Bucket contains results under athena-results/, including .csv and .csv.metadata files
  - **List local directory**: Confirmed `~/CVEDataLake/query_results/` has multiple JSON query result files
  - **Examine JSON file**: Results confirm properly formatted structured JSON data
</details>

### Deploy GitHub Actions Workflow
```bash
ansible-playbook setup_pipeline.yaml -vv
```
This workflow runs on **every push or pull request** affecting files in the `docker/` directory. It uses a **matrix strategy** to iterate over 4 Docker images: `alpine`, `nginx`, `python`, and `nodejs`.

**Job 1**: `build-and-push`
- **Purpose**:
  - Build each Docker image and push it to the `docker-dev` ECR repository
- **Main Tasks**:
  - Checkout the repo
  - Configure AWS credentials
  - Log in to ECR
  - Generate a timestamped image tag
  - Build the Docker image
  - Push the image to `docker-dev`
  - Save the image tag for the next job

**Job 2**: `scan-move`

- **Depends on**: `build-and-push`
- **Purpose**:
  - Scan the image with **Trivy**, generate an **SBOM** (Software Bill of Materials), and move the image based on scan results
- **Main Tasks**:
  - Download the image tag
  - Install and run Trivy scanner
  - Upload scan results and SBOM to S3
  - If vulnerabilities are found:
    - Move image to `docker-quarantine`
    - Send Slack alert
  - If clean:
    - Promote image to `docker-prod`

### Trigger Pipeline
```bash
git add -A
git commit -m "Trigger pipeline with sample apps"
git push
```
This kicks off:
- Image build & push to ECR (`docker-dev`)
- Trivy scan
- SBOM and scan results upload to S3
- Image promotion or quarantine based on scan
- Slack alert if `CRITICAL` or `HIGH` vulnerabilities found

<details close>
  <summary> <h4>Image Results</h4> </summary>
    
![CVEDataLake](https://i.imgur.com/idwIvVZ.png)
![CVEDataLake](https://i.imgur.com/fWI7OLO.png)

  - **List S3**: Bucket contains results under athena-results/, including .csv and .csv.metadata files
  - **List local directory**: Confirmed `~/CVEDataLake/query_results/` has multiple JSON query result files
  - **Examine JSON file**: Results confirm properly formatted structured JSON data
</details>

## Conclusion

ScanOps automates end-to-end container security as part of a modern **(CI/CD)** workflow. With GitHub Actions, AWS, and Trivy at its core, this project ensures secure container delivery, real-time alerting, and compliance reporting through SBOMs—all orchestrated with Ansible.

This project can easily be extended to:
- Include GitHub OIDC integration for secure AWS access
- Integrate with AWS Glue & Athena for queryable vulnerability analytics
- Expand Slack notifications with richer summaries or visuals

> Note: Run `cleanup.sh` to delete all resources
