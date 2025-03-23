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
  - Rocky Linux with `Ansible` and `Git` installed
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

### Clone repo and install required Ansible collections
```bash
git clone git@github.com:<your-github-username>/scanops.git
cd scanops
ansible-galaxy install -r requirements.yaml
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

<details close>
  <summary> <h4>Image Results</h4> </summary>
    
![scanops](https://i.imgur.com/fk2TFSC.png)

  - **These secrets are accessed by the GitHub Actions workflow to authenticate with AWS and send alerts to Slack**
<br><br>
</details>

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
    
![scanops](https://i.imgur.com/NDC24VN.png) 
![scanops](https://i.imgur.com/hVgJwLa.png) 
![scanops](https://i.imgur.com/rFx1Ecg.png) 

  - **AWS CLI Configuration**:
    - AWS credentials are set up using a shared credentials file, and the region is configured as us-east-1
    - The IAM user is verified via sts get-caller-identity, confirming its UserId, Account, and ARN
  - **Output confirms the three ECR repos are created**:
    - `docker-dev`
    - `docker-prod`
    - `docker-quarantine`
  - **Verifies successful provisioning of S3 storage**
<br><br>

![scanops](https://i.imgur.com/tmSbOyS.png) 
![scanops](https://i.imgur.com/Wv76N7z.png) 

  - **Confirmed in AWS web console**:
    - All three repositories created
    - S3 storage provisioned
<br><br>
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
    
![scanops](https://i.imgur.com/I8LUEGX.png) 

  - **Output of sample Dockerfile creation from `setup_docker_apps.yaml` playbook**
  - **Confirmed four Docker apps**:
    - Alpine
    - Nginx
    - Python
    - Node.js
<br><br>
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
    
![scanops](https://i.imgur.com/AWvYfDb.png) 
  - **Matrix strategy kicks off 4 parallel build-and-push jobs**:
    - alpine
    - nginx
    - python
    - nodejs
  - **Scan-move matrix is queued then completes build jobs with no errors**
<br><br>

![scanops](https://i.imgur.com/DKCPWxa.png) 
  - **SBOMs for all four sample apps were successfully generated and uploaded to the `scanops-s3` bucket**
<br><br>

![scanops](https://i.imgur.com/fCFsuBF.png)
  - **Trivy scan results were also successfully uploaded to the `scanops-s3` bucket** 
<br><br>

![scanops](https://i.imgur.com/6FNeXAA.png)
  - **Trivy scan for the `nginx` image returned 0 vulnerabilities, marking it as clean**
<br><br>

![scanops](https://i.imgur.com/rwejnA2.png)
  - **Scan for the `nodejs` image detected 374 total vulnerabilities, including 22 CRITICAL and 352 HIGH severity issues**
<br><br>

![scanops](https://i.imgur.com/Sd6V77g.png)
  - **Trivy scan for the `alpine` image found 11 vulnerabilities, including 1 CRITICAL and 10 HIGH severity**
<br><br>

![scanops](https://i.imgur.com/SqWLHIS.png)
  - **The `python` image scan revealed 129 total vulnerabilities, with 1 marked CRITICAL and 128 categorized as HIGH severity**
<br><br>

![scanops](https://i.imgur.com/sVAqNVv.png)
  - **The `nginx` image passed the Trivy scan and was successfully promoted to the `docker-prod` ECR repository**
<br><br>

![scanops](https://i.imgur.com/nysAYkU.png)
  - **The `python`, `nodejs`, and `alpine` images were pushed to the `docker-quarantine` ECR repository for isolation**
<br><br>

![scanops](https://i.imgur.com/hemgDJf.png)
  - **Slack alert triggered by ScanOps, reporting vulnerability summaries for `alpine`, `nodejs`, and `python` images**
<br><br>
</details>

## Conclusion

ScanOps automates end-to-end container security as part of a modern **(CI/CD)** workflow. Leveraging GitHub Actions, Trivy, and AWS services it ensures only vulnerability-free images are promoted to production, while others are quarantined with detailed SBOMs and scan results stored in S3 and alerts sent via Slack.

> Note: Run `cleanup.yaml` playbook to delete all AWS resources
