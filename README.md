# E-Commerce Microservices: GitOps & CI/CD Deployment

This repository is the master project containing the deployment architecture for Google's [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) microservices application. 

It demonstrates a complete, cost-optimized Cloud Native lifecycle: from infrastructure provisioning (Terraform) to continuous integration (GitHub Actions), pure GitOps deployment (Kluctl + K3s), and full-stack observability.

## Repository Structure (Submodules)

* [**`cloud-microservices-demo`**](https://github.com/6ill/cloud-microservices-demo): Forked from Google's original repo. Contains application source code and the custom CI pipeline.
* [**`microservices-boutique-kubernetes`**](https://github.com/6ill/microservices-boutique-kubernetes.git): Contains Infrastructure as Code (Terraform) and GitOps deployment states.

## Tech Stack
* **Cloud & Compute:** AWS (EC2 Spot Instances, VPC), S3 (Terraform remote state)
* **Orchestration:** K3s (Lightweight Kubernetes)
* **GitOps & Config Management:** Kluctl, Kustomize, Kluctl GitOps Controller (Source Controller)
* **CI/CD:** GitHub Actions, GitHub Container Registry (GHCR)
* **Local Environment:** Devbox

## Observability Stack

The project implements a unified observability pipeline to provide deep visibility into the microservices ecosystem without the overhead of heavy enterprise suites:

* **Unified Collection:** Uses **Grafana Alloy** as a single, lightweight agent to collect and forward metrics, logs, and traces, reducing resource consumption on the K3s node.
* **Infrastructure & App Metrics:** **Prometheus** handles time-series data, utilizing **ServiceMonitors** for automated discovery of microservices and cluster components.
* **Log Aggregation:** **Grafana Loki** centralizes logs across all 11 services, enabling high-performance log querying with label-based indexing.
* **Distributed Tracing:** **Grafana Tempo** provides request-level visibility. By instrumenting critical services (Frontend/Checkout) via **Kustomize patches** and **OpenTelemetry**, the stack correlates logs and traces to accelerate Root Cause Analysis (RCA).
* **Visualization:** A centralized **Grafana** instance provides unified dashboards, merging data from all three pillars to monitor the "RED" (Rate, Errors, Duration) signals.

---

## Prerequisites (Requirements)

This project uses **[Devbox](https://www.jetpack.io/devbox/)** to guarantee a consistent, isolated local development environment. 

1. Install **Devbox** on your local machine.
2. A GitHub Personal Access Token (PAT) with `repo` and `packages:write` scopes. This must be configured as a secret (`GITOPS_BRIDGE_PAT`) in the Application Repository to allow GitHub Actions to push configurations to the Infrastructure Repository.
3. AWS Credentials configured locally with permissions to provision VPCs, EC2, and S3.

*Note: If you do not wish to use Devbox, you must manually install Terraform (v1.5+), AWS CLI v2, `kubectl`, and `kluctl`.*

---

## Key Features & Implementation Highlights

### 1. Application Repository (What was added to the Fork)
Instead of modifying the core application code, I designed a custom CI/CD pipeline (`.github/workflows/ci-all-services.yaml`) optimized for a monorepo structure.

* **Matrix Strategy with Path Filtering:** The pipeline uses a bash script (`git diff`) to check which specific microservice directory was modified. It only triggers the Docker build for that specific service, saving massive amounts of compute time compared to rebuilding all 11 services.
* **GitOps Bridge (CD Handoff):** Once the image is pushed to GHCR, the CI uses `kustomize edit set image` to update the image tag (using the git commit SHA) in the Infrastructure repository.
* **Concurrency & Race Condition Handling:** Implemented a Jitter retry pattern (`git pull --rebase` with random sleep) to handle situations where multiple matrix jobs try to push new tags to the infrastructure repo simultaneously.

### 2. Infrastructure Repository
This repository controls the actual state of the AWS environment and the Kubernetes cluster.

* **Terraform Provisioning:** Fully automated provisioning of AWS VPC, Security Groups, and an EC2 Spot Instance (`t3.large`) with K3s bootstrapped via Cloud-init (`user_data`). Terraform state is stored securely in S3.
* **Resource Tuning via Kustomize:** Added Kustomize patches to strictly limit CPU/Memory requests and limits (especially for heavy Java/.NET services). This allows the entire 11-microservice architecture to run smoothly on a single, cost-effective EC2 node.
* **Pure GitOps (Pull Model):** The cluster uses Kluctl Deployment and Kluctl GitOps Controller. The cluster actively reaches out to GitHub to pull configurations, meaning the K3s API port (`6443`) remains tightly locked behind AWS Security Groups, restricted only to my personal IP.
* **Unified Observability Pipeline:** Deployed a modern observability stack using Grafana Loki, Tempo, and Prometheus. By utilizing Grafana Alloy as a unified collector, the system correlates metrics, logs, and distributed traces while maintaining a low resource footprint. 
* **Helm Vendoring & Cold-Start Optimization:** Helm charts are strictly vendored into the Git repository to enforce pure GitOps principles. Additionally, Operator Admission Webhooks were disabled to prevent race conditions during automated cold-start cluster bootstrapping.

---

## The CI/CD Workflow

1.  **Code:** Developer pushes changes to the application repository (e.g., modifying `cartservice`).
2.  **Build:** GitHub Actions dynamically detects the change, builds the new Docker image, and pushes it to GHCR tagged with the Git commit SHA.
3.  **Update State:** GitHub Actions clones the Infrastructure repo, updates `kustomization.yaml` with the new image tag, and pushes the commit.
4.  **Sync:** The Kluctl Controller running inside the K3s cluster detects the new commit on GitHub.
5.  **Deploy:** Kluctl pulls the updated manifests, applies Kustomize patches, and rolls out the new Pod automatically.

---

## Bootstrap

To reproduce this environment from scratch:

1.  **Start the Isolated Environment:**
    ```bash
    # This automatically downloads exact versions of terraform, kubectl, and kluctl
    devbox shell
    ```
2.  **Provision Infrastructure:**
    ```bash
    cd microservices-boutique-kubernetes/infrastructures
    terraform init
    terraform apply
    ```
3.  **Connect to Cluster:** Update your local `kubeconfig` with the new EC2 Public IP generated by Terraform.
4.  **Bootstrap GitOps:**
    ```bash
    # Install Kluctl controller into the K3s cluster
    kluctl controller install

    # Apply the root GitRepository and KluctlDeployment sync definition
    kubectl apply -f deployment/bootstrap/gitops-sync.yaml
    ```
From this point forward, the cluster is fully autonomous and managed via Git commits.

---

## Accessing the Observability Dashboard

The Grafana dashboard is deployed as a `ClusterIP` service to prevent public internet exposure. To access it:

1. Open a secure port-forward tunnel from your local machine to the cluster:
   ```bash
   kubectl port-forward svc/prometheus-grafana 3000:80 -n monitoring

2. Open your browser and navigate to http://localhost:3000.

3. Log in with the configured credentials (default: admin/admin) to view the live Kubernetes and microservices metrics generated by VictoriaMetrics.
