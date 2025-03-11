name: Build and Push Docker Images to ACR  

on:
  push:
    branches:
      - dev-domain 

env:
  DOMAIN_ACR_NAME: devdomainaks.azurecr.io 
  K8S_MANIFESTS_DIR: k8s

jobs:
  Build-and-Push:
    runs-on: self-hosted

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Install Docker Compose Manually
        run: |
          mkdir -p ~/.docker/cli-plugins/
          curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o ~/.docker/cli-plugins/docker-compose
          chmod +x ~/.docker/cli-plugins/docker-compose
          docker compose version

      - name: Clean up old Buildx containers and volumes
        run: |
          docker ps -a -q --filter "name=buildx_buildkit" | xargs -I {} docker rm -f {}
          docker volume ls -q --filter "name=buildx_buildkit" | xargs -I {} docker volume rm {}

      - name: Set Up Docker Buildx
        uses: docker/setup-buildx-action@v3
        
      - name: Log in to Azure Container Registry (ACR)
        run: |
          echo "${{ secrets.DOMAIN_ACR_PASSWORD }}" | docker login ${{ env.DOMAIN_ACR_NAME }} -u ${{ secrets.DOMAIN_ACR_USERNAME }} --password-stdin

      - name: Get Commit SHA
        id: get_commit_sha
        run: echo "COMMIT_SHA=$(git rev-parse --short=7 HEAD)" >> $GITHUB_ENV

      - name: Create .env File
        run: |
          cat <<EOF > .env
          DATABASE_URL=${{ secrets.DATABASE_URL }}
          VIRUSTOTAL_API_KEY=${{ secrets.VIRUSTOTAL_API_KEY }}
          VIEWDNS_API_KEY=${{ secrets.VIEWDNS_API_KEY }}
          POSTGRES_PASS=${{ secrets.POSTGRES_PASS }}
          DOMAINREPUTATION_ENV=${{ secrets.DOMAINREPUTATION_ENV }}
          IPINFO_TOKEN=${{ secrets.IPINFO_TOKEN }}
          DB_USER=${{ secrets.DB_USER }}
          DB_PASSWORD=${{ secrets.DB_PASSWORD }}
          DB_NAME=${{ secrets.DB_NAME }}
          DB_HOST=${{ secrets.DB_HOST }}
          SECRET=${{ secrets.SECRET }}
          ALGORITHM=${{ secrets.ALGORITHM }}
          API_KEY=${{ secrets.API_KEY }}
          EOF
        shell: bash

      - name: Build Docker images with BuildKit
        run: |
          DOCKER_BUILDKIT=1 docker compose build --build-arg BUILDKIT_INLINE_CACHE=1
          for IMAGE in $(docker compose config | awk '/image:/ {print $2}'); do
            IMAGE_NAME=${IMAGE/:latest/}
            docker tag $IMAGE_NAME:latest $IMAGE_NAME:${{ env.COMMIT_SHA }}
          done

      - name: Push Docker Images to ACR
        run: |
          for IMAGE in $(docker compose config | awk '/image:/ {print $2}'); do
            IMAGE_NAME=${IMAGE/:latest/}
            docker push $IMAGE_NAME:${{ env.COMMIT_SHA }}
          done

      - name: Logout from ACR
        run: docker logout ${{ env.ACR_NAME }}

      - name: Clean Up Everything After Build
        run: |
          docker system prune -af --volumes
          docker builder prune -af
          sudo rm -rf /var/lib/docker/*
          sudo rm -rf /var/lib/containerd/*
          sudo rm -rf ~/.docker
          sudo systemctl restart docker
        
  Deploy-to-AKS:
    needs: Build-and-Push
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
          az acr login --name $DOMAIN_ACR_NAME
       
      - name: Get Commit SHA
        id: get_commit_sha
        run: echo "COMMIT_SHA=$(git rev-parse --short=7 HEAD)" >> $GITHUB_ENV
       
      - name: Get AKS Credentials
        run: az aks get-credentials --resource-group ${{ secrets.RESOURCE_GROUP }} --name ${{ secrets.DOMAIN_CLUSTER_NAME }}

      - name: Update Kubernetes Manifests with New Image Tags
        run: |
          for FILE in ${K8S_MANIFESTS_DIR}/*.yaml; do
            sed -i "s|\(image: [^:]*\)|\1:${{ env.COMMIT_SHA }}|g" "$FILE"
          done
          echo "✅ The following Kubernetes manifests have been updated with COMMIT_SHA:"
          grep -H "image:" ${K8S_MANIFESTS_DIR}/*.yaml

          echo "🚀 Applying updated manifests to AKS..."
          kubectl apply -f ${K8S_MANIFESTS_DIR}
    
      - name: Verify Deployment
        run: kubectl get pods -n domain-subdomain-tools

  Deleting-old-images:
    needs: Build-and-Push
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository code
        uses: actions/checkout@v4

      - name: Login to Azure (Service Principal)
        run: |
          az login --service-principal -u ${{ secrets.AZURE_CLIENT_ID }} -p ${{ secrets.AZURE_CLIENT_SECRET }} --tenant ${{ secrets.AZURE_TENANT_ID }} 

      - name: Login to Azure Container Registry
        run: |
          echo "${{ secrets.DOMAIN_ACR_PASSWORD }}" | docker login ${{ env.DOMAIN_ACR_NAME }} -u ${{ secrets.DOMAIN_ACR_USERNAME }} --password-stdin

      - name: Purge old images from ACR            
        run: |
          az acr run --registry ${{ env.DOMAIN_ACR_NAME }} --cmd "acr purge --filter '*:.*' --ago 0d --keep 3 --untagged" /dev/null
