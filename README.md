# eks-project
Eks Project


## **Repository Structure**  
```bash
📂 eks-project  
├── README.md
├── fluxcd
│   ├── README.md
│   ├── app
│   │   ├── Dockerfile
│   │   ├── default
│   │   └── index.html
│   ├── flux-resources
│   │   ├── README.md
│   │   ├── gitrepository.yaml
│   │   └── kustomization.yaml
│   ├── flux-system
│   │   ├── gotk-components.yaml
│   │   ├── gotk-sync.yaml
│   │   └── kustomization.yaml
│   └── manifests
│       ├── app-deployment.yaml
│       └── app-service.yaml
└── infrastructure
    └── eks-cluster
        ├── README.md
        ├── app.py
        ├── cdk.json
```

---

## Step 1: Infrastructure Provisioning

1. **Navigate to the infrastructure directory:**
   ```bash
   cd infrastructure/eks-cluster
   ```

2. **Create a Python virtual environment:**
   ```bash
   python3 -m venv .venv
   ```

3. **Activate the virtual environment:**
   ```bash
   source .venv/bin/activate
   ```

4. **Install required dependencies:**
   ```bash
   pip install -r requirements.txt
   ```

5. **Synthesize the CDK application:**
   ```bash
   cdk synth
   ```

6. **Deploy the network and EKS stacks:**
   ```bash
   cdk deploy --all --require-approval never
   ```

This process will provision the basic EKS cluster infrastructure.


## Step 2: **Bootstrapping Flux**  

### **1. Install Flux CLI**  
```bash
# macOS/Linux
brew install fluxcd/tap/flux

# Verify
flux --version
```

### **2. Bootstrap Flux on Cluster**  
```bash
flux bootstrap github \
  --owner=RevanthChowdary1708 \
  --repository=eks-project \
  --branch=main \
  --path=fluxcd \
  --personal
```

This:  
- Creates `flux-system` namespace  
- Deploys Flux controllers (source, kustomize, helm)  
- Configures Git repository synchronization  

![FluxCD Setup](./assets/fluxcd-setup.png)

---

## Step 3: Configure repo structure

## Step4: **Example: GitOps Deployment**  
*Deploy app1 by committing Kubernetes manifests*  

### **Workflow**  

### **1. Configure Flux Objects**  
**a. GitRepository (fluxcd/repos/infra-repo/apps/app1/gitrepository.yaml)**  
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: sample-repo
  namespace: fluxcd-demo
spec:
  interval: 1m0s
  ref:
    branch: main
  url: https://github.com/RevanthChowdary1708/eks-project
```

**What it does**:  
- Monitors `main` branch in your repo  
- Checks for changes every minute  

**b. Kustomization (fluxcd/repos/infra-repo/apps/app1/kustomization.yaml)**  
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: sample-app
  namespace: fluxcd-demo
spec:
  interval: 15m
  path: "./fluxcd/manifests/"
  prune: true
  sourceRef:
    kind: GitRepository
    name: sample-repo
```

**What it does**:  
- Applies manifests from [fluxcd/manifests](fluxcd/manifests) directory  
- Auto-deletes removed resources (`prune: true`)  
- Syncs every 5 minutes  

## Step 5:
### **2. GitHub Actions Automation (.github/workflows/eks-flux-app-build.yaml)**  
```yaml
# .github/workflows/eks-flux-app-build.yaml
name: EKS FluxCD Project | App - Docker Build

on:
  workflow_dispatch:
    inputs:
      bump-type:
        description: 'Version bump type'
        required: true
        default: 'minor'
        type: choice
        options:
        - major
        - minor
        - patch
  push:
    branches:
      - main
    paths:
      - 'fluxcd/app'

jobs:
  build-push-update:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code (with all branches)
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
          ref: main
      
      - name: Prepare version
        id: prep
        run: |
          if [[ $GITHUB_REF == refs/tags/* ]]; then
            VERSION=${GITHUB_REF#refs/tags/}
            BUILD_IMAGE=true
          else
            VERSION=${GITHUB_SHA::8}
            BUILD_IMAGE=true
          fi
          echo "VERSION=$VERSION" >> $GITHUB_ENV
          echo "IMAGE=sairevanthchowdarych/eks-flux-app:$VERSION" >> $GITHUB_ENV
          echo "BUILD_IMAGE=$BUILD_IMAGE" >> $GITHUB_ENV


      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push Docker image (Simple App)
        uses: docker/build-push-action@v4
        with:
          context: fluxcd/app/
          file: fluxcd/app/Dockerfile
          push: true
          tags: |
            sairevanthchowdarych/eks-flux-app:latest
            sairevanthchowdarych/eks-flux-app:${{ env.VERSION }}
          platforms: linux/amd64,linux/arm64
          build-args: |
            VERSION=${{ env.VERSION }}
      
      - name: Update Kubernetes manifests
        run: |
          # sed -i "s|image: sairevanthchowdarych/eks-flux-app:.*|image: ${{ env.IMAGE }}|" fluxcd/manifests/app-deployment.yaml
          sed -i "s|image:.*|image: sairevanthchowdarych/eks-flux-app:${{ env.VERSION }}|" fluxcd/manifests/app-deployment.yaml
          git config user.name "FluxCDBot CI"
          git config user.email "fluxcdbot@users.noreply.github.com"

      - name: Commit and push changes
        uses: EndBug/add-and-commit@v7
        with:
          add: |
            - fluxcd/manifests/app-deployment.yaml
          message: "Update image to ${{ env.VERSION }}"
          signoff: true
          push: true
          branch: main
          author_name: "Fluxcdbot CI"
          author_email: "fluxcdbot@users.noreply.github.com"
          #token: ${{ secrets.PR_PAT_TOKEN }}
      
```

**Workflow**:  
1. Developer pushes code to `fluxcd/app`  
2. GitHub Actions:  
   - Builds and Pushes Docker image with commit SHA tag  
   - Updates [app-deployment.yaml](fluxcd/manifests/app-deployment.yaml)  
   - Commits changes to `main` branch  
3. Flux detects Git changes and deploys new image  
