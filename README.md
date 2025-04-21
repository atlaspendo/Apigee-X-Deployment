
# Apigee X Deployment via Azure DevOps & GKE Self-Hosted Agents

This document outlines the architecture and setup process for automating Apigee X API Proxy deployments using Azure DevOps pipelines running on self-hosted agents within Google Kubernetes Engine (GKE) clusters. Authentication to Google Cloud Platform (GCP) and Apigee X APIs is handled securely via Workload Identity Federation (WIF).

## Table of Contents

1.  [Architecture Overview](#architecture-overview)
2.  [Prerequisites](#prerequisites)
3.  [GKE Cluster & Agent Setup (Per GCP Project)](#gke-cluster--agent-setup-per-gcp-project)
4.  [Azure DevOps Setup](#azure-devops-setup)
5.  [Pipeline Structure (`azure-pipelines.yml`)](#pipeline-structure-azure-pipelinesyml)
6.  [Deployment Job Template (`templates/apigee-deploy-job.yml`)](#deployment-job-template-templatesapigee-deploy-jobyml)
7.  [Authentication Flow (Workload Identity Federation)](#authentication-flow-workload-identity-federation)
8.  [Configuration Variables](#configuration-variables)
9.  [How to Use](#how-to-use)
10. [Security Considerations](#security-considerations)
11. [Future Enhancements](#future-enhancements)

## Architecture Overview

This setup utilizes separate GCP projects for non-production and production environments to ensure isolation.

*   **GCP Non-Production Project:**
    *   Hosts an Apigee X instance for `dev`, `test`, and `uat` environments across all environment groups (`edd`, `wow`, `wpay`, `homerun`).
    *   Hosts a GKE cluster (`azdo-agents-cluster-nonprod`) dedicated to running Azure DevOps agents for non-production builds and deployments.
    *   Contains GCP Service Accounts (SAs) for each non-prod environment (e.g., `edd-dev-sa`, `wow-test-sa`, `wpay-uat-sa`).
    *   Configured with Workload Identity Federation (WIF) pool trusting the non-prod GKE cluster.
*   **GCP Production Project:**
    *   Hosts an Apigee X instance for `prod` environments across all environment groups.
    *   Hosts a separate GKE cluster (`azdo-agents-cluster-prod`) dedicated to running Azure DevOps agents *only* for production deployments.
    *   Contains GCP Service Accounts (SAs) for each prod environment (e.g., `edd-prod-sa`, `wow-prod-sa`).
    *   Configured with Workload Identity Federation (WIF) pool trusting the prod GKE cluster.
*   **Azure DevOps:**
    *   Hosts the Git repository containing the Apigee proxy source code (`/apiproxy` folder) and pipeline definitions (`azure-pipelines.yml`, templates).
    *   Manages two separate Self-Hosted Agent Pools:
        *   `gke-nonprod-agents`: Targets the agents running on the Non-Prod GKE cluster.
        *   `gke-prod-agents`: Targets the agents running on the Prod GKE cluster.
    *   Executes the CI/CD pipeline, orchestrating the build and deployment process.
    *   Utilizes Azure DevOps Environments for deployment approvals (Test, UAT, Prod).

**Authentication Flow:**
AzDO Job -> Runs on Agent Pod (GKE) -> Agent Pod runs as KSA (`azdo-agent-ksa`) -> `gcloud` uses KSA credentials via WIF -> `gcloud` impersonates Target GCP SA (e.g., `edd-dev-sa`) based on pipeline context -> Apigee API calls are made using the impersonated SA's credentials.

*(Refer to the architecture diagrams provided previously for a visual representation.)*

## Prerequisites

Before setting up the pipelines and agents, ensure the following are in place:

1.  **Azure DevOps:**
    *   An Azure DevOps Project.
    *   A Git repository within the project (`Azure Repos`).
2.  **GCP Projects:**
    *   Two separate GCP projects: one for Non-Production, one for Production. Note their Project IDs.
3.  **Apigee X:**
    *   Apigee X instances provisioned in both GCP projects.
    *   Required Apigee Environments (`dev`, `test`, `uat` in Non-Prod; `prod` in Prod) created.
    *   Required Apigee Environment Groups (`edd`, `wow`, `wpay`, `homerun`) created and associated with the correct environments in each instance.
4.  **GCP Service Accounts:**
    *   Dedicated GCP Service Accounts created in *each* GCP project for *each combination* of environment group and environment type they will deploy to.
    *   **Naming Convention:** Use a consistent pattern, e.g., `{envGroup}-{envType}-sa@{gcpProjectId}.iam.gserviceaccount.com` (e.g., `edd-dev-sa@nonprod-project.iam.gserviceaccount.com`, `wpay-prod-sa@prod-project.iam.gserviceaccount.com`).
5.  **GCP IAM Roles:**
    *   Assign necessary IAM roles to the GCP Service Accounts created above:
        *   `roles/apigee.environmentAdmin` (or more specific roles like `apigee.deployer`, `apigee.importer`) on the corresponding Apigee resources.
        *   `roles/iam.serviceAccountTokenCreator` on themselves (to allow impersonation).
6.  **GCP Workload Identity Federation Setup:**
    *   In GCP IAM for *each* project, configure WIF to trust identities from the *yet-to-be-created* GKE Kubernetes Service Account (`azdo-agent-ksa` in the `azdo-agents` namespace). This involves creating a WIF Pool linked to the GKE OIDC provider.
    *   Grant the Kubernetes Service Account identity the `roles/iam.workloadIdentityUser` role on *each* of the target GCP Service Accounts it needs to impersonate within that project. (See GKE Setup Step 4).

## GKE Cluster & Agent Setup (Per GCP Project)

Perform these steps in **both** the Non-Prod and Prod GCP projects. Replace placeholders like `your-nonprod-project-id` or `your-prod-project-id` accordingly.

1.  **Create GKE Cluster:**
    *   Ensure Workload Identity is enabled (`--workload-pool=PROJECT_ID.svc.id.goog`).
    *   Use Standard or Autopilot mode.
    *   Example command:
        ```bash
        GCP_PROJECT="your-gcp-project-id" # nonprod or prod
        CLUSTER_NAME="azdo-agents-cluster" # e.g., azdo-agents-cluster-nonprod
        REGION="your-region" # e.g., us-central1

        gcloud container clusters create $CLUSTER_NAME \
            --project=$GCP_PROJECT \
            --region=$REGION \
            --machine-type=e2-medium \
            --num-nodes=2 \
            --enable-autoscaling --min-nodes=1 --max-nodes=5 \
            --workload-pool="${GCP_PROJECT}.svc.id.goog" \
            --enable-shielded-nodes \
            --shielded-secure-boot
        ```
    *   Connect `kubectl` to the new cluster.

2.  **Create Kubernetes Namespace:**
    ```bash
    kubectl create namespace azdo-agents
    ```

3.  **Create Kubernetes Service Account (KSA):**
    ```bash
    kubectl create serviceaccount azdo-agent-ksa --namespace azdo-agents
    ```

4.  **Configure IAM Binding for WIF:**
    *   Allow the KSA (`azdo-agent-ksa`) to impersonate the target GCP SAs within this project. **Repeat this command for every target GCP SA the agents in this cluster need to access.**
    ```bash
    GCP_PROJECT="your-gcp-project-id" # nonprod or prod
    K8S_NAMESPACE="azdo-agents"
    K8S_SA_NAME="azdo-agent-ksa"

    # --- Repeat for each target SA ---
    TARGET_GCP_SA="{envGroup}-{envType}-sa@${GCP_PROJECT}.iam.gserviceaccount.com" # e.g., edd-dev-sa@...

    gcloud iam service-accounts add-iam-policy-binding $TARGET_GCP_SA \
        --project=$GCP_PROJECT \
        --role="roles/iam.workloadIdentityUser" \
        --member="serviceAccount:${GCP_PROJECT}.svc.id.goog[${K8S_NAMESPACE}/${K8S_SA_NAME}]"

    gcloud iam service-accounts add-iam-policy-binding $TARGET_GCP_SA \
        --project=$GCP_PROJECT \
        --role="roles/iam.serviceAccountTokenCreator" \
        --member="serviceAccount:${GCP_PROJECT}.svc.id.goog[${K8S_NAMESPACE}/${K8S_SA_NAME}]"
    # --- End Repeat ---
    ```

5.  **Build & Push Custom Agent Docker Image:**
    *   Create a `Dockerfile` (see previous detailed response for an example) including:
        *   Base OS (e.g., Ubuntu)
        *   `gcloud` CLI, `kubectl`
        *   `jq`, `curl`, `zip`, `git`
        *   Azure Pipelines Agent binaries
    *   Build the image and push it to a registry accessible by GKE (e.g., Google Artifact Registry):
        ```bash
        docker build -t your-registry/azdo-agent:latest .
        docker push your-registry/azdo-agent:latest
        ```

6.  **Deploy Agents using Helm Chart:**
    *   Add the Helm repo:
        ```bash
        helm repo add azure-pipelines-agent https://microsoft.github.io/azure-pipelines-agent-charts
        helm repo update
        ```
    *   Create a `values.yaml` file (e.g., `values-nonprod.yaml`, `values-prod.yaml`):
        ```yaml
        # values-nonprod.yaml (Example)
        adoUrl: "https://dev.azure.com/YOUR_ORG"
        adoPat: "YOUR_AZDO_PAT" # Ideally use --set-string or K8s secret reference
        adoPool: "gke-nonprod-agents" # Use 'gke-prod-agents' for prod values file

        image:
          repository: "your-registry/azdo-agent" # Your custom image
          tag: "latest"

        kubernetesServiceAccount:
          create: false # We created it manually above
          name: "azdo-agent-ksa"

        auth:
          type: pat
        ```
    *   Install the chart in the correct cluster, pointing to the correct values file:
        ```bash
        # Ensure kubectl context is set to the correct cluster (non-prod or prod)
        helm install azdo-agent azure-pipelines-agent/azure-pipelines-agent \
            --namespace azdo-agents \
            -f values-{nonprod/prod}.yaml \
            # Add --set-string adoPat=YOUR_PAT if not in values or using secret ref
        ```

## Azure DevOps Setup

1.  **Agent Pools:**
    *   In Azure DevOps Project Settings -> Agent Pools, create two pools:
        *   `gke-nonprod-agents`
        *   `gke-prod-agents`
2.  **Personal Access Token (PAT):**
    *   Generate an Azure DevOps PAT with scope `Agent Pools (Read, Manage)`.
    *   **Store this PAT securely.** Options:
        *   Azure Key Vault, linked via a Variable Group in AzDO Library.
        *   Kubernetes Secret (referenced during Helm install).
        *   Pass directly during Helm install (`--set-string adoPat=...`, less secure).
3.  **Pipeline Files:**
    *   Create `azure-pipelines.yml` in the root of your repository (content below).
    *   Create `templates/apigee-deploy-job.yml` in a `templates` folder (content below).
4.  **Variable Groups (Recommended):**
    *   Create Variable Groups in AzDO Library to store:
        *   Non-secret variables (`nonProdGcpProject`, `prodGcpProject`).
        *   Secrets linked from Azure Key Vault (e.g., the AzDO PAT).
5.  **Environments:**
    *   In AzDO Pipelines -> Environments, create environments for tracking and approvals, e.g.:
        *   `Apigee-Dev-{ProxyName}`
        *   `Apigee-Test-{ProxyName}`
        *   `Apigee-UAT-{ProxyName}`
        *   `Apigee-Prod-{ProxyName}`
    *   Configure approvals on the Test, UAT, and Prod environments.

## Pipeline Structure (`azure-pipelines.yml`)

This main pipeline file defines parameters, triggers, variables, and stages. It uses the deployment job template for the actual deployment logic.

*   **Parameters:** Takes `apiProxyName` and `environmentGroup` as input.
*   **Triggers:** Configured to run based on commits to `develop` (-> Dev), `release/*` (-> Test/UAT), and `main` (-> Prod). Also allows manual runs.
*   **Variables:** Defines GCP project IDs, agent pool names, and dynamically constructs the target GCP Service Account email based on the stage context.
*   **Stages:**
    *   `Build`: Runs on the `gke-nonprod-agents` pool. Zips the `./apiproxy` folder and publishes it as an artifact. Optional linting step.
    *   `Deploy_Dev`: Runs on `gke-nonprod-agents`. Uses the template to deploy to the `dev` environment in the non-prod project.
    *   `Deploy_Test`: Runs on `gke-nonprod-agents` (after Dev). Requires approval. Uses the template to deploy to `test`.
    *   `Deploy_UAT`: Runs on `gke-nonprod-agents` (after Test). Requires approval. Uses the template to deploy to `uat`.
    *   `Deploy_Prod`: Runs on **`gke-prod-agents`** (after UAT). Requires approval. Uses the template to deploy to the `prod` environment in the **prod** project.

*(See the previous detailed response for the full `azure-pipelines.yml` code.)*

## Deployment Job Template (`templates/apigee-deploy-job.yml`)

This template encapsulates the reusable deployment steps performed within each deployment stage.

*   **Parameters:** Receives context like GCP project, Apigee org/env, proxy name, target GCP SA, artifact name, etc., from the main pipeline.
*   **Deployment Job:** Uses an AzDO deployment job type, allowing targeting of AzDO Environments for approvals.
*   **Steps:**
    1.  Downloads the API proxy bundle artifact.
    2.  **Authenticates to GCP:** Uses `gcloud config set project ...` and crucially `gcloud config set auth/impersonate_service_account ...` to tell `gcloud` to act as the target GCP Service Account via Workload Identity. Verifies impersonation.
    3.  **Imports & Deploys:**
        *   Gets an OAuth token using `gcloud auth print-access-token` (which now belongs to the impersonated SA).
        *   Uses `curl` to call the Apigee Management API (`/apis?action=import`) to upload the bundle and create a new revision. Extracts the revision number using `jq`. Includes fallback logic to find the latest revision if extraction fails.
        *   Uses `curl` to call the Apigee Management API (`/deployments?override=true`) to trigger the deployment of the new revision to the target environment.

*(See the previous detailed response for the full `templates/apigee-deploy-job.yml` code.)*

## Authentication Flow (Workload Identity Federation)

1.  The Azure DevOps pipeline schedules a job on the appropriate agent pool (`gke-nonprod-agents` or `gke-prod-agents`).
2.  The job runs inside an agent pod on the corresponding GKE cluster. This pod is configured (via Helm chart) to run using the Kubernetes Service Account `azdo-agent-ksa` in the `azdo-agents` namespace.
3.  When the `gcloud` command runs inside the pod, it automatically detects the GKE environment and uses the KSA's projected token.
4.  The `gcloud config set auth/impersonate_service_account {TARGET_GCP_SA}` command instructs GCP IAM to use the KSA's identity (trusted via the WIF pool setup) to generate credentials for the specified `{TARGET_GCP_SA}`. This requires the KSA identity to have the `roles/iam.workloadIdentityUser` role on the target SA.
5.  Subsequent `gcloud auth print-access-token` calls retrieve an OAuth token for the impersonated `{TARGET_GCP_SA}`.
6.  `curl` commands use this token in the `Authorization: Bearer` header to authenticate API calls to Apigee as the target service account.

## Configuration Variables

Ensure the following placeholders and variables are correctly set in your `azure-pipelines.yml`, `templates/apigee-deploy-job.yml`, and Helm `values.yaml` files:

*   `YOUR_ORG` (Azure DevOps Organization URL)
*   `your-nonprod-project-id`
*   `your-prod-project-id`
*   `gke-nonprod-agents` / `gke-prod-agents` (Ensure these match AzDO pool names)
*   `your-registry/azdo-agent:latest` (Correct path to your custom agent image)
*   `YOUR_AZDO_PAT` (Provide securely during Helm install or via secrets)
*   Verify the GCP Service Account naming convention matches your created SAs: `$(parameters.environmentGroup)-$(apigeeEnv)-sa@$(gcpProject).iam.gserviceaccount.com`
*   Proxy source code location (`./apiproxy` assumed in build step)

## How to Use

1.  **Setup:** Complete all steps in the Prerequisites, GKE Setup, and Azure DevOps Setup sections.
2.  **Code:** Place your Apigee API Proxy source code in the `./apiproxy` directory (or adjust the path in the build stage) in your Azure Repo.
3.  **Commit & Push:** Commit and push your proxy code and the pipeline YAML files (`azure-pipelines.yml`, `templates/*`) to your Azure Repo.
    *   Push to `develop` to trigger a Dev deployment.
    *   Create/push to `release/*` branches to trigger Test/UAT deployments (after Dev succeeds).
    *   Merge to `main` to trigger a Prod deployment (after UAT succeeds).
4.  **Manual Trigger:** You can also manually run the pipeline, selecting the source branch and providing the required `apiProxyName` and `environmentGroup` parameters.
5.  **Approvals:** Approve deployments in the Azure DevOps Environments UI for Test, UAT, and Prod stages when prompted.
6.  **Monitor:** Observe the pipeline execution in Azure DevOps and check the Apigee UI for deployment status.

## Security Considerations

*   **PAT Security:** Use a PAT with the minimum required scope (`Agent Pools (Read, Manage)`). Store it securely (Azure Key Vault + Variable Group recommended). Use short expiry dates and rotate regularly.
*   **GCP Service Accounts:** Apply the principle of least privilege. Grant only the necessary Apigee roles (`apigee.environmentAdmin` or finer) for the specific environments they manage. Ensure only the correct KSA has `workloadIdentityUser` permission on them.
*   **GKE Security:** Consider Network Policies in GKE to restrict network access *from* the agent pods if needed. Use Shielded Nodes. Keep GKE updated.
*   **Agent Image:** Regularly scan the custom agent Docker image for vulnerabilities and keep base images and installed tools (`gcloud`, agent binaries) updated.
*   **Code Security:** Implement proxy code linting (`apigeelint`) and potentially static analysis within the pipeline.
*   **Approvals:** Enforce mandatory approvals for deployments to sensitive environments (Test, UAT, Prod).

## Future Enhancements

*   **Deployment Verification:** Add a step after deployment to poll the Apigee deployment status API or run basic smoke tests against the deployed proxy.
*   **Automated Testing:** Integrate API functional/integration tests (e.g., using Postman/Newman) into the pipeline after deployment to each environment.
*   **Secrets Management:** Enhance PAT handling using OIDC between AzDO and GCP if feasible, or more robust Vault integration.
*   **Apigee Revision Cleanup:** Add steps to list and optionally delete old, undeployed Apigee proxy revisions.
*   **Canary/Blue-Green:** Implement more advanced deployment strategies if required, although Apigee's deployment model primarily relies on revision swaps.
*   **Pipeline Optimization:** Further refine the pipeline using more advanced template features or conditional logic.
