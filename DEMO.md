# 🚀 Terraform CI/CD Demo Guide

This guide outlines how to set up and demonstrate a professional **Infrastructure as Code (IaC)** pipeline using Terraform, GitHub Actions, and AWS.

## 1. The Architecture & Story
We are deploying a **2-Tier Web Architecture** (Nginx on EC2) that is:
*   **Highly Available:** Auto Scaling Group across multiple Availability Zones.
*   **Secure:** Private subnets for apps, Public subnets for Load Balancers.
*   **Automated:** Managed entirely via GitHub Actions.

**The Workflow:**
1.  **`dev` branch:** Deploys to the Development Environment (Automatic).
2.  **`test` branch:** Deploys to the Test Environment (Automatic).
3.  **`main` branch:** Deploys to Production (Requires **Manual Approval**).

---

## 2. One-Time Setup (Prerequisites)

Before the demo, you must configure the "Control Plane".

### A. AWS Setup (State Management)
Run these commands locally to create the backend resources:
```bash
# 1. Create S3 Bucket for State
aws s3 mb s3://my-terraform-state-demo-123 --region us-east-1

# 2. Enable Versioning
aws s3api put-bucket-versioning --bucket my-terraform-state-demo-123 --versioning-configuration Status=Enabled
```
*Update `backend.tf` with this bucket name!*
*Note: We are using S3 Native Locking (`use_lockfile = true`), so no DynamoDB table is required.*

### B. GitHub Repository Setup
1.  **Secrets (CRITICAL):** Go to `Settings` -> `Secrets and variables` -> `Actions`.
    *   Click **"New repository secret"** (Do NOT use Environment secrets).
    *   Add `AWS_ACCESS_KEY_ID`: Your AWS Key.
    *   Add `AWS_SECRET_ACCESS_KEY`: Your AWS Secret.

2.  **Environments:** Go to `Settings` -> `Environments`.
    *   **Mandatory:** Create `prod`.
        *   Click "Configure".
        *   Check **"Required reviewers"**.
        *   Search for your GitHub username and add yourself.
        *   Click "Save protection rules".
    *   **Optional:** You can create `dev` and `test` for better visibility in the "Deployments" dashboard. If you skip this, GitHub will automatically create them (with no rules) when the pipeline runs.

---

## 3. The Demo Script (Showtime!)

### Scene 0: Understanding State Separation
"How do we manage 3 environments with 1 backend file?"

Explain **Terraform Workspaces**:
*   In our workflow, we run `terraform workspace select dev` (or test/prod).
*   This automatically changes the S3 path where the state is saved.
*   **Dev State:** `s3://.../env:/dev/terraform/state/main/terraform.tfstate`
*   **Prod State:** `s3://.../env:/prod/terraform/state/main/terraform.tfstate`
*   This keeps our environments completely isolated in the same bucket.

### Scene 1: The Feature Request
"We need to scale up our development environment to handle more load. Let's do that!"

1.  **Create a Branch:**
    ```bash
    git checkout -b feat/scale-up-dev
    ```
2.  **Make a Change:**
    *   Open `dev.tfvars`.
    *   Change `desired_capacity` from `1` to `2`.
3.  **Push:**
    ```bash
    git add .
    git commit -m "feat: scale up dev environment"
    git push origin feat/scale-up-dev
    ```

### Scene 2: The Pull Request (CI)
1.  Open a **Pull Request** from `feat/scale-up-dev` to `dev`.
2.  **Show the Checks:**
    *   Point out **TFLint** and **Trivy** running security scans.
    *   Wait for **Terraform Plan**.
3.  **Show the Bot Comment:**
    *   Open the PR comment to see the plan output.
    *   "See? We know exactly what will happen before we merge. It says `Plan: 1 to add, 0 to change, 0 to destroy`."

### Scene 3: Deployment to Dev (CD)
1.  **Merge the PR.**
2.  Go to the **Actions** tab.
3.  Watch the pipeline:
    *   `Terraform Plan` runs again (for the artifact).
    *   `Terraform Apply` runs automatically.
4.  **Verify:** Go to AWS Console -> EC2 -> Auto Scaling Groups. Show the instance count increasing.

### Scene 4: Promotion to Production (The Gate)
"Now that Dev is working, let's go to Production."

1.  **Create a PR** from `dev` (or your feature branch) to `main`.
2.  **Merge** to `main`.
3.  Go to **Actions** tab.
4.  **The Pause:**
    *   Show that the pipeline **stops** at the `Terraform Apply` step.
    *   "Production is protected. It waits for my approval."
5.  **The Approval:**
    *   Click "Review Deployments" -> "Approve and Deploy".
6.  **Success:** Watch the production infrastructure update safely.

---

## 4. Key Talking Points
*   **Immutable Infrastructure:** We don't SSH into servers; we update code.
*   **Security First:** Trivy scans every commit for vulnerabilities.
*   **Confidence:** The "Plan" artifact used in the PR is the *exact same binary* used in the "Apply", guaranteeing no surprises.
