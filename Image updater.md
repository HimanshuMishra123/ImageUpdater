Here's a step-by-step guide to deploy the Argo Image Updater as an operator in your Kubernetes production cluster with OIDC authentication for ECR and updates to Kubernetes manifests stored in Bitbucket repositories.

---

### **1. Prerequisites**
- Kubernetes cluster with admin access.
- OIDC provider set up in AWS and attached to the Kubernetes cluster.
- Amazon ECR repository with images.
- Bitbucket repository with Kubernetes manifests.
- `kubectl`, `eksctl`, and `helm` installed on your local machine.
- Argo CD already installed and managing your cluster resources.

---

### **2. Steps to Deploy Argo Image Updater**

#### **Step 1: Configure OIDC and IAM Roles for Service Accounts**
1. **Create an IAM policy for ECR access:**
   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Effect": "Allow",
               "Action": [
                   "ecr:GetAuthorizationToken",
                   "ecr:BatchGetImage",
                   "ecr:BatchCheckLayerAvailability",
                   "ecr:DescribeImages",
                   "ecr:GetDownloadUrlForLayer"
               ],
               "Resource": "*"
           }
       ]
   }
   ```

2. **Attach the IAM policy to a role:**
   - Use `eksctl` to create a service account with this role:
     ```bash
     eksctl create iamserviceaccount \
       --name argo-image-updater-sa \
       --namespace argo-image-updater \
       --cluster <your-cluster-name> \
       --attach-policy-arn arn:aws:iam::<account-id>:policy/ECRAccessPolicy \
       --approve \
       --override-existing-serviceaccounts
     ```

#### **Step 2: Install Argo Image Updater**
1. Add the Helm repository:
   ```bash
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update
   ```
2. Install the Argo Image Updater:
   ```bash
   helm install argo-image-updater argo/argocd-image-updater \
     --namespace argocd \
     --set serviceAccount.name=argo-image-updater-sa
     --set clusterName=<your-cluster-name> \
     --set serviceAccount.create=false \
   ```

#### **Step 3: Configure Bitbucket Access**
1. **Create a Bitbucket App Password:**
   - Generate an app password with `read/write` access to the repository.

2. **Create a Kubernetes Secret for Bitbucket:**
   ```bash
   kubectl create secret generic bitbucket-creds \
     --namespace argo-image-updater \
     --from-literal=username=<bitbucket-username> \
     --from-literal=password=<bitbucket-app-password>
   ```

#### **Step 4: Application YAML for Argo Image Updater**
Here’s an example `application.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-image-updater
  namespace: argo-image-updater
spec:
  project: default
  source:
    repoURL: https://<bitbucket-repo-url>.git
    targetRevision: HEAD
    path: path/to/manifests
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

#### **Step 5: Configure Argo Image Updater for ECR**
Create the configuration map for Argo Image Updater:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argo-image-updater-config
  namespace: argo-image-updater
data:
  repositories: |
    - name: ecr
      type: ecr
      registryUrl: <account-id>.dkr.ecr.<region>.amazonaws.com
  credentials: |
    - registry: <account-id>.dkr.ecr.<region>.amazonaws.com
      username: AWS
      passwordFrom:
        serviceAccount: argo-image-updater-sa
  bitbucket:
    username: <bitbucket-username>
    passwordSecret:
      name: bitbucket-creds
      key: password
```

#### **Step 6: Validate the Setup**
- Check logs to ensure Argo Image Updater connects to ECR and updates Bitbucket manifests:
  ```bash
  kubectl logs -f deployment/argo-image-updater -n argo-image-updater
  ```

- Verify that Kubernetes resources are updated upon changes to the image in ECR.


## Here's a breakdown of each step in simple terms:

### **1. Configure OIDC and IAM Roles for Service Accounts**
- **Purpose:** We need to give the Argo Image Updater permission to access Amazon ECR (Elastic Container Registry) to fetch Docker images.
  
1. **Create an IAM policy for ECR access:**
   - We create a policy that grants permission to access images from ECR.
  
2. **Attach the IAM policy to a role:**
   - We attach the above policy to a role that Argo Image Updater can use. This role will be tied to a Kubernetes service account (a type of identity for the application) that Argo Image Updater will use to interact with AWS.

---

### **2. Install Argo Image Updater**
- **Purpose:** We're installing the Argo Image Updater on your Kubernetes cluster. It will monitor ECR for new image versions and automatically update Kubernetes resources (like deployments) when a new image is found.

1. **Add the Helm repository:**
   - Helm is a package manager for Kubernetes. We're adding the Argo project’s Helm chart repository to install Argo Image Updater.
  
2. **Install Argo Image Updater:**
   - This installs Argo Image Updater in the Kubernetes cluster, using the service account that we created earlier to interact with AWS ECR.

---

### **3. Configure Bitbucket Access**
- **Purpose:** We need Argo Image Updater to push changes (new image versions) to a Bitbucket repository where Kubernetes manifests are stored.

1. **Create a Bitbucket App Password:**
   - Bitbucket needs a password (App Password) to let Argo Image Updater access the repository where the Kubernetes resources (manifests) are stored.

2. **Create a Kubernetes Secret for Bitbucket:**
   - Kubernetes secrets store sensitive information like passwords. We create a secret to store the Bitbucket username and App Password so that Argo Image Updater can access the Bitbucket repository securely.

---

### **4. Application YAML for Argo Image Updater**
- **Purpose:** This YAML file defines how Argo CD should sync and manage the image updater application.

- **What it does:**
  - The `repoURL` tells Argo CD where your Bitbucket repository is located.
  - The `path` specifies where the Kubernetes manifests are stored inside the repo.
  - The `destination` specifies where to deploy (i.e., the Kubernetes cluster).
  - `syncPolicy` ensures Argo CD keeps the application up to date automatically, even making changes when necessary.

---

### **5. Configure Argo Image Updater for ECR**
- **Purpose:** This config file tells the Argo Image Updater how to connect to ECR to check for new Docker image versions.

- **What it does:**
  - It defines the ECR registry URL and credentials for accessing ECR.
  - It links the Bitbucket credentials to ensure the updater can push changes to Bitbucket when it detects new images.

---

### **6. Validate the Setup**
- **Purpose:** We need to check if the setup is working correctly and if Argo Image Updater is updating the Kubernetes resources when new Docker images are pushed to ECR.

- **What it does:**
  - It checks the logs to make sure Argo Image Updater is doing its job, such as fetching new images from ECR and updating the Kubernetes deployments in Bitbucket.
