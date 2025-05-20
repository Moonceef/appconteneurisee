git # Interactive Visit Counter App

A containerized Node.js application that counts visits and provides an interactive counter interface.

## Features

- Visit counting with persistent storage
- Interactive UI with increment/decrement buttons
- Docker containerization
- Ready for Azure deployment

## Local Development

1. Clone this repository
2. Install dependencies: `npm install`
3. Start the app: `npm start`
4. Visit http://localhost:3000

## Docker Development

Build and run locally with Docker:

```bash
# Build the container
docker build -t visit-counter .

# Run the container
docker run -p 3000:3000 -d visit-counter
```

## Docker Compose

Start with docker-compose:

```bash
docker-compose up -d
```

## Azure Deployment

Follow these steps to deploy your Docker app to Azure using the Azure Portal web UI:

### Step 1: Create an Azure Container Registry (ACR)

1. Sign in to the [Azure Portal](https://portal.azure.com)
2. Click on **Create a resource** (top-left corner)
3. Search for and select **Container Registry**
4. Click **Create**
5. Complete the form with the following:
   - **Subscription**: Select your subscription
   - **Resource Group**: Create new or select existing
   - **Registry name**: Enter a unique name (e.g., `mydockerciregistry`)
   - **Location**: Select a location close to you
   - **SKU**: Standard
6. Click **Review + create**, then **Create**

### Step 2: Configure Access to Your Container Registry

1. Once your Container Registry is created, navigate to it in the Azure Portal
2. From the left menu, go to **Access keys**
3. Enable **Admin user**
4. Note down the **Login server**, **Username**, and **Password** - you'll need these later

### Step 3: Create an Azure App Service Plan

1. In the Azure Portal, click **Create a resource**
2. Search for and select **App Service Plan**
3. Click **Create**
4. Complete the form with the following:
   - **Subscription**: Select your subscription
   - **Resource Group**: Same as used for Container Registry
   - **Name**: Choose a name (e.g., `myAppServicePlan`)
   - **Operating System**: Linux
   - **Region**: Same as Container Registry
   - **Pricing Tier**: Click on "Change size" and select at least B1 (Basic)
5. Click **Review + create**, then **Create**

### Step 4: Create a Web App for Containers

1. In the Azure Portal, click **Create a resource**
2. Search for and select **Web App**
3. Click **Create**
4. Complete the basic settings:
   - **Subscription**: Select your subscription
   - **Resource Group**: Same as used previously
   - **Name**: `dockerci-job` (or a unique name of your choice)
   - **Publish**: Docker Container
   - **Operating System**: Linux
   - **Region**: Same as previously selected
   - **App Service Plan**: Select the plan created in Step 3
5. Click **Next: Docker**
6. In the Docker tab:
   - **Options**: Single Container
   - **Image Source**: Azure Container Registry
   - **Registry**: Select your ACR created in Step 1
   - **Image**: dockerci-job (or your image name)
   - **Tag**: latest
7. Click **Review + create**, then **Create**

### Step 5: Build and Push Docker Image to Azure Container Registry

1. In the Azure Portal, navigate to your Container Registry
2. From the left menu, click on **Quick start**
3. Follow the instructions to:
   - Log in to your registry: `docker login [registry-name].azurecr.io`
   - Build your image: `docker build -t [registry-name].azurecr.io/dockerci-job:latest .`
   - Push your image: `docker push [registry-name].azurecr.io/dockerci-job:latest`

Alternatively, you can set up a GitHub Actions workflow (as in this repository):

1. In your GitHub repository, go to **Settings**
2. Click on **Secrets and variables** > **Actions**
3. Add the following secrets:
   - `REGISTRY_LOGIN_SERVER`: Your ACR login server (e.g., `mydockerciregistry.azurecr.io`)
   - `REGISTRY_USERNAME`: Your ACR username
   - `REGISTRY_PASSWORD`: Your ACR password
   - `AZURE_WEBAPP_PUBLISH_PROFILE`: Content of your Web App's publish profile (download from Azure Portal > Your Web App > Overview > Get publish profile)

### Step 6: Configure Volume Mount for Persistent Storage (Optional)

1. In your Azure Web App, go to **Settings** > **Configuration**
2. Add a new application setting:
   - **Name**: `WEBSITES_ENABLE_APP_SERVICE_STORAGE`
   - **Value**: `true`

3. Then, navigate to **Settings** > **Path mappings**
4. Add a new Azure Files mount:
   - **Name**: `counter-data`
   - **Configuration Option**: Basic configuration
   - **Storage Account**: Create new or select existing
   - **Storage Container**: Create new or select existing
   - **Storage Type**: Azure Files
   - **Mount Path**: `/app/counter.json`

### Step 7: Check Deployment Status and Logs

1. Navigate to your Web App in the Azure Portal
2. Go to **Deployment Center** to view deployment status
3. Check **Log stream** for real-time application logs
4. Visit your app at `https://[your-app-name].azurewebsites.net`

## GitHub Actions Integration with Azure

This repository includes a GitHub Actions workflow file (`.github/workflows/azure.yml`) configured to automatically build and deploy your Docker application to Azure when you push to the main branch.

### Understanding the GitHub Actions Workflow

Our workflow:
1. Builds your Docker image
2. Pushes it to Azure Container Registry
3. Deploys it to Azure Web App for Containers

```yaml
name: Deploy Docker app to Azure
on:
  push:
    branches:
      - main
jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Azure Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ secrets.REGISTRY_LOGIN_SERVER }}
        username: ${{ secrets.REGISTRY_USERNAME }}
        password: ${{ secrets.REGISTRY_PASSWORD }}

    - name: Build and push container image to registry
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ secrets.REGISTRY_LOGIN_SERVER }}/dockerci-job:${{ github.sha }}

    - name: Deploy to Azure Web App for Containers
      uses: azure/webapps-deploy@v3
      with:
        app-name: 'dockerci-job'
        publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
        images: ${{ secrets.REGISTRY_LOGIN_SERVER }}/dockerci-job:${{ github.sha }}
```

### Setup Required GitHub Secrets

For this workflow to run successfully, you must configure these secrets in your GitHub repository:

1. Go to your GitHub repository > Settings > Secrets and variables > Actions
2. Add the following secrets:
   - `REGISTRY_LOGIN_SERVER`: Your ACR login server (e.g., `mydockerciregistry.azurecr.io`)
   - `REGISTRY_USERNAME`: Your ACR username (from Access keys in your Container Registry)
   - `REGISTRY_PASSWORD`: Your ACR password (from Access keys in your Container Registry)
   - `AZURE_WEBAPP_PUBLISH_PROFILE`: The contents of your publish profile file (download from Web App > Overview > Get publish profile)

With these secrets in place, every push to your main branch will automatically deploy your application to Azure.
