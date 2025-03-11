name: Build and Deploy to ACR

on:
  push:
    branches:
      - main  # or the branch you want to trigger the pipeline on

env:
  K8S_MANIFESTS_DIR: k8s-manifest

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Azure Container Registry (ACR)
        run: |
          echo "${{ secrets.ACR_PASSWORD }}" | docker login ${{ secrets.ACR_NAME }}.azurecr.io -u ${{ secrets.ACR_USERNAME }} --password-stdin

      - name: Get Commit SHA
        id: get_commit_sha
        run: echo "COMMIT_SHA=$(git rev-parse --short=7 HEAD)" >> $GITHUB_ENV

      - name: Build Docker image with BuildKit
        run: |
          DOCKER_BUILDKIT=1 docker build --build-arg BUILDKIT_INLINE_CACHE=1 -t ${{ secrets.ACR_NAME }}.azurecr.io/my-image:${{ env.COMMIT_SHA }} .

      - name: Push Docker Image to ACR
        run: |
          docker push ${{ secrets.ACR_NAME }}.azurecr.io/my-image:${{ env.COMMIT_SHA }}

      - name: Logout from ACR
        run: docker logout ${{ secrets.ACR_NAME }}.azurecr.io

  Deploy-to-AKS:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Log in to Azure
        run: |
          az login --service-principal \
            -u "${{ secrets.AZURE_CLIENT_ID }}" \
            -p "${{ secrets.AZURE_CLIENT_SECRET }}" \
            --tenant "${{ secrets.AZURE_TENANT_ID }}"

      - name: Set Azure Subscription
        run: az account set --subscription "${{ secrets.AZURE_SUBSCRIPTION_ID }}"

      - name: ACR Login
        run: |
          az acr login --name ${{ secrets.ACR_NAME }}

      - name: Get Commit SHA
        id: get_commit_sha
        run: echo "COMMIT_SHA=$(git rev-parse --short=7 HEAD)" >> $GITHUB_ENV

      - name: Get AKS Credentials
        run: az aks get-credentials --resource-group ${{ secrets.RESOURCE_GROUP }} --name ${{ secrets.AKS_CLUSTER_NAME }}

      - name: Update Kubernetes Manifests with New Image Tags
        run: |
          echo "Replacing image tag in Kubernetes manifests..."
          for FILE in ${K8S_MANIFESTS_DIR}/*.yaml; do
            sed -i "s|testrepo3.azurecr.io/my-image:[^ ]*|testrepo3.azurecr.io/my-image:${{ env.COMMIT_SHA }}|g" "$FILE"
          done
          echo "✅ The following Kubernetes manifests have been updated with COMMIT_SHA:"
          grep -H "image:" ${K8S_MANIFESTS_DIR}/*.yaml

          echo "🚀 Applying updated manifests to AKS..."
          kubectl apply -f ${K8S_MANIFESTS_DIR}

      - name: Verify Deployment
        run: kubectl get pods -n argo1
