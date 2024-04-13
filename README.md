## 0. Create or transfer a new GitHub repository in the example GitHub organization, then add CI/CD automation:

To set up a GitHub repository for a streamlit project in the example GitHub organization, you can either create a new one, or transfer an existing one:

___

### To create a new repository:

Navigate to the example organization repositories and click **New repository**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/b3729af2-83ee-44eb-8cb3-bdef00ff904c)

Input any unused name for the **Repository name \***:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/bf352585-cf5f-4fd4-8e98-7cb91a5d94da)

Make sure the repository visibility is set to **Private**, then click **Create repository**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/a9c9a526-26c6-4094-9a01-388d1678ab4c)

### To transfer ownership of an existing repository:

Navigate to **Settings**, scroll to the bottom of the page, then click **Danger Zone > Transfer ownership > Transfer**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/2aba309a-4aef-4e43-a307-f4182e2e9400)

For **Select one of my organizations** choose **example**, type the name of the repository, then click **I understand, transfer this repository.**

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/5d1dffb1-410b-459e-a218-fc3f2c1f2541)

___

The rest of this guide will cover how to deploy a streamlit.io project as an Azure web app.

### Add CI/CD automation:

To automatically build and deploy a web app on every push to the main branch, we will use a [GitHub Actions workflow](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions):

#### `./github/on_push_main.yml`:

```
# Docs for the Azure Web Apps Deploy action: https://github.com/Azure/webapps-deploy
# More GitHub Actions for Azure: https://github.com/Azure/actions

name: set Azure Vault secrets, build image and push to Azure Container Registry, deploy to Azure Web App

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
    id-token: write
    contents: read

jobs:
  build-and-push:
    runs-on: 'ubuntu-latest'

    steps:
    - name: Azure login
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - uses: actions/checkout@v3

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - run: az acr login --name ${{ vars.ACR_REGISTRY }}

    - name: Build and push container image to registry
      uses: docker/build-push-action@v5
      with:
        push: true
        tags: '${{ vars.ACR_REGISTRY }}/${{ vars.ACR_REPOSITORY }}:${{ github.sha }}'
        file: ./Dockerfile

  set-secrets:
    runs-on: 'ubuntu-latest'

    steps:
    - name: Azure login
      uses: azure/login@v2
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Set Key Vault Secrets
      run: |
        az keyvault secret set --vault-name "${AZURE_VAULT_NAME}" --name "APP-SECRETS" --value "${APP_SECRETS}"
      env:
        AZURE_VAULT_NAME: ${{ vars.AZURE_VAULT_NAME }}
        APP_SECRETS: ${{ secrets.APP_SECRETS }}

  deploy:
    runs-on: ubuntu-latest
    needs: [build-and-push, set-secrets]
    if: ${{ always() && !failure() && !cancelled() }}
    environment:
      name: 'production'
      url: ${{ steps.deploy-to-webapp.outputs.webapp-url }}

    steps:    
    - name: Deploy to Azure Web App
      id: deploy-to-webapp
      uses: azure/webapps-deploy@v3
      with:
        app-name: '${{ vars.AZURE_APP_NAME }}'
        slot-name: 'production'
        publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
        images: '${{ vars.ACR_REGISTRY }}/${{ vars.ACR_REPOSITORY }}:${{ github.sha }}'
```

The workflow above expects the outermost directory of the repository to contain a [Dockerfile](https://docs.docker.com/reference/dockerfile/):

#### `./Dockerfile`:

```
FROM mambaorg/micromamba:latest
USER root
RUN apt-get update && DEBIAN_FRONTEND=“noninteractive” apt-get install -y --no-install-recommends \
       nginx \
       ca-certificates \
       apache2-utils \
       certbot \
       python3-certbot-nginx \
       sudo \
       cifs-utils \
       cron \
       && \
    rm -rf /var/lib/apt/lists/*
RUN mkdir /opt/example
RUN chmod -R 777 /opt/example
RUN chown $MAMBA_USER:$MAMBA_USER /opt/example
USER $MAMBA_USER
WORKDIR /opt/example
COPY environment.yml environment.yml
COPY app/ app/
RUN micromamba install -y -n base -f environment.yml && \
   micromamba clean --all --yes
EXPOSE 8000
ENTRYPOINT ["/usr/local/bin/_entrypoint.sh","streamlit"]
CMD ["run","/opt/example/app/app.py","--server.port","8000","--theme.base","dark"]
```

And the Dockerfile above expects the outermost directory of the repository to contain a [conda environment.yml](https://conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html#create-env-file-manually):

#### ./`environment.yml`:

```
name: <app name goes here>
channels:
 - conda-forge
 - defaults
dependencies:
 - python=3.11
 - pip
 - pip:
   - -r app/requirements.txt
```

As well as an `./app/` directory with a pip `requirements.txt` and a python3 `app.py`. The workflow will use the Azure Container Registry to publish a docker image whenever a commit is pushed to the main branch, which will have the updated code along with all of its dependencies. The workflow will then deploy this image to an Azure Web Apps container, which we will set up next:

## 1. Create a user-assigned managed identity:

To authenticate with Azure from the workflow, we will create a managed identity with the [Azure portal](https://portal.azure.com/#home). Log in and navigate to **Home > Managed Identities** and click **+ Create**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/291ef3df-7d7f-4e9a-ab5e-afec4bd74311)

Choose the same **Subscription** and **Resource Group** that the Web App will belong to, then click **Review + create**, and then **Create**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/d6355d9a-6c24-410a-90de-1f5f4c302f4c)

Now navigate to **All resources > ExampleAppNameIdentity**, select **Settings | Federated credentials**, and click **+ Add Credential**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/0f4aa2ef-e107-40c5-a720-7dde1ddf8f05)

Find **Federated credential scenario** and select **"Github Actions deploying Azure resources"**. For **Organization** input **example**, input **Repository** as the name of the repository created in step 0, *ex.* example-project, for **Entity** select **Branch**, and for **Name** input **on_push_main**, then click **Add**. 

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/ba41ef96-db22-4d61-8971-70d837648441)

### Copy the managed identity credentials into GitHub Actions secrets:

Select **Overview** in the managed identity page and copy the **Subscription ID** field.

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/ebc0e17a-8712-4591-b8d2-736fd99eba58)

In another tab, open the GitHub repository from step 0, navigate to **Settings > Secrets and variables > Actions**, and select **New repository secret**.

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/879789de-3c31-460e-a492-3a7c02906479)

Set the **Name \*** input to `AZURE_SUBSCRIPTION_ID`, and in **Secret \*** paste the **Subscription ID** from the managed identity:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/5e37af29-7ea9-4383-a029-5430d9d8f518)

then click **Add secret**. Now back in the managed identity page of the Azure portal, navigate to **Properties**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/7e6ccb89-d6bb-4c03-b96d-a918df8b366f)

Copy the **Tenant Id** into another GitHub secret named `AZURE_TENANT_ID`, and the **Client Id** into another named `AZURE_CLIENT_ID`:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/4813147a-ddec-4aa5-a317-4d4807396870)

## 2. Create a key vault:

Back in the Azure portal, navigate to **Home > Key vaults** and click **+ Create**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/b1b4161e-08f5-4161-bf01-8abb2bce9f3a)

Choose the same **Subscription** and **Resource Group** that the Web App will belong to, and choose a **Key vault name** based on the name of the web app. The name needs to be globally unique, so I prefixed it with *ex-*. Then click **Review + create**, and then **Create**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/b2c26d36-8b18-462e-8a33-5506cfe233da)

Now the open the vault page, select **Access control (IAM)**, and click **+ Add > Add role assignment**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/c09abbb5-6dc6-4932-8930-0d1d79177be4)

Under **Job function roles**, search for *"Key Vault Secrets Officer"*, select it from the list, and click **Next**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/bca785d5-deae-4fcd-b0e4-33530905e164)

Under **Assign access to** select **Managed identity**, then click **+ Select members**. In the pane that opens on the right, under **Managed identity** choose **User-assigned managed identity**, select the managed identity created in step 1, click **Select**, then click **Next**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/a77a1ecc-f3ac-41d3-8e98-53d301d7a25d)

Now click **Review + assign**. In another tab, open the GitHub repository from step 0, navigate to **Settings > Secrets and variables > Actions**, above **Repository secrets** select **Variables**, and click **New repository variable**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/263a8cf2-0962-4f42-b557-e8dc63f700c2)

Set the **Name \*** input to **AZURE_VAULT_NAME**, and in **Value \*** paste the **Key vault name** that was used to create the key vault:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/782a91a6-232e-4f96-9dc8-d9d157e64eaa)

Then click **Add variable**. The workflow can now send GitHub Actions secrets to this key vault before each deployment is started.

## 3. Create or configure a container registry:

To create a Web App docker container, a container registry needs to be used. Many web apps can all share the same registry, so these instructions will configure an existing registry to be used with a new web app. In the Azure portal, navigate to **Home > All resources > examplemainllmcr**, and select **Access control (IAM)**. Then click **+ Add > Add role assignment**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/a1be19d4-61cb-4ca0-b8a5-25b486e306dd)

Under **Job function roles**, search for *"AcrPush"*, select it from the list, and click **Next**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/176e9a6f-4bea-4beb-ace5-6d59b89f8155)

For **Assign access to** choose **Managed identity**, then click **+ Select members**. In the pane that opens on the right, under **Managed identity** choose **User-assigned managed identity**, select the managed identity created in step 1, click **Select**, then click **Next**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/3b0c0c5d-3541-4081-8418-ac95ecea6bf1)

Now click **Review + assign**. Repeat these steps again to assign the *"AcrPull"* role to the same identity. Select **Overview** in the container registry page and copy the **Login server** name:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/85e31e27-bc9c-447e-825d-094f896576e5)

Now in another tab, open the GitHub repository from step 0, navigate to **Settings > Secrets and variables > Actions**, above **Repository secrets** select **Variables**, and click **New repository variable**. For **Name \*** input `ACR_REGISTRY`, and for **Value \*** paste the **Login server** name that you copied from the Azure portal, then click **Add variable**.

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/17246e96-4bcf-4a4a-b672-83d4c334a05d)

Add another variable named `ACR_REPOSITORY`, and choose its value based on the name of the GitHub repository from step 0. Its value can be any lowercase name, with hyphens but no other symbols. Add another variable named `AZURE_APP_NAME`, and choose its value based on the name of the repository as well. This one will have to be globally unique, so I prefixed it with *ex-*, but it is just a placeholder at this point and you may have to change it later. You should now have four variables defined:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/22db05d9-eaf4-403b-90ca-3bb6a2e5ed0b)

Navigate to **Actions > set Azure Vault secrets, (...)** and click **Run workflow > Run workflow**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/6b623735-1852-4208-afd2-2e81342dc6de)

The workflow run will fail when it gets to the deploy job, but it only needed to be run to populate the container repository.

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/e6971898-919c-4485-9a16-e4b9ad6fd894)

Wait for the run to fail before you move on to the last step.

## 4. Create a web app:

Open the Azure portal [Create Web App wizard](https://portal.azure.com/#create/Microsoft.WebSite). Choose the same **Subscription** and **Resource Group** that the resources created in the previous steps used. The input for **Name** needs to match the value of the `AZURE_APP_NAME` variable from step 2. If you need to change it, make sure to update that value in your GitHub repostiory settings. For **Publish** choose **Docker Container**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/229575c6-8039-40fd-acc0-54b1b0004a79)

Press **Next : Database >** and then **Next : Docker >**, or select the **Docker** tab. For **Registry**, choose the value of the `ACR_REGISTRY` variable from step 3, and for **Image**, choose the value of the `ACR_REPOSITORY` variable from step 3. For **Tag**, choose whatever option is made available, it should be the SHA of the head commit of the GitHub repostiory from step 0:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/17bc3533-be44-4103-9a21-3f721ab8d18d)

Click **Review + create**, and then **Create**. Navigate to **Home > All resources** and select the web app you just created, then select **Deployment Center**. For **Source** choose **GitHub Actions: (...)**, for **Organization** choose **example**, for **Repository** choose the GitHub repository from step 0, for **Branch** choose **main**, and for **Workflow Option** choose **Use available workflow: (...)**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/34423ef4-fd84-4fad-a725-353a7c767812)

Scroll to the bottom of the page, then for **Registry source** choose **Azure Container Registry**, for **Subscription ID** choose the subscription used in the previous steps, for **Authentication** choose **Managed Identity**, then for **Identity** choose the identity created in step 1 under **User assigned**, and for **Image** choose the value of the `ACR_REPOSITORY` variable from step 3, then click **Save**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/7bff3a75-a2d2-43ab-943c-019a93b9f362)

Now select **Identity**, and the page will open to the **System assigned** tab. Set its **Status** to **On**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/3a2b128c-aceb-4359-ac8c-2769811d1980)

Then select **User assigned** and click **+ Add**, choose the subscription used in the previous steps, select the managed identity the created in step 1, and click **Add**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/d8ab12ca-5b14-412c-82b5-20751a0b09dd)

Navigate to **Home > All resources** and select the key vault created in step 2. Select **Access control (IAM)**, then click **+ Add > Add role assignment**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/d8b71f34-09eb-4f7b-800f-c40bddccc758)

Under **Job function roles**, search for *"Key Vault Secrets User"*, select it from the list, and click **Next**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/19363d97-7edb-4179-8f2f-0f5ce268eebe)

For **Assign access to** choose **Managed identity**, then click **+ Select members**. In the pane that opens on the right, under **Managed identity** choose **System-assigned managed identity > App Service (?)**, select the identity of the web app you just created, click **Select**, then click **Next**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/00872420-7ec2-42b5-906f-e734ec874225)

Now click **Review + assign**, and then **Review + assign** again. Repeat these steps again to assign the *"Key Vault Reader"* role to the same identity. Now navigate to **Home > All resources** and select the web app you just created. Then select **Settings > Configuration**, and click **+ New application setting**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/e9db5c56-3a08-4e4a-9260-ada98c189860)

For **Name** input `APP_SECRETS`, and for **Value** input `@Microsoft.KeyVault(VaultName=example-vault;SecretName=APP-SECRETS)`, but replace the value of VaultName with the name of the key vault created in step 2. Then click **Okay**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/a31ddc0f-e738-42a5-b9f8-b8137db3a6a7)

Repeat these steps to add a setting with the name `WEBSITES_PORT` and the value `8000`, and a setting with the name `WEBSITES_CONTAINER_START_TIME_LIMIT` and the value `1600`. There should now be five settings in total, but if any are missing, set them according to these:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/6b9e5843-6501-4cd8-a767-34580ab91114)

Then click **Save**. Now select **Overview**, and click **Download publish profile**:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/a35aae04-dbe6-4063-8097-0d49979fb362)

And copy the contents of the downloaded file. Then open the GitHub repository, navigate to **Settings > Secrets and variables > Actions**, and click **New repository secret**. For **Name \*** input `AZURE_PUBLISH_PROFILE`, and for **Secret \*** paste the contents of the profile you just downloaded:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/e44d760b-503b-45f0-9d1f-c9f29297aa76)

Now, create a secret with the name `APP_SECRETS`, and set its value to the contents of the `secrets.toml` file found in the `.streamlit` directory of a streamlit project:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/5b4dfa20-004e-4268-b628-7a54db10a1a9)

Then edit the `./app/app.py` script to load these secrets from the environment instead of streamlit.io:

```python3
sp=st.file_util.get_project_streamlit_file_path("secrets.toml")
os.makedirs(os.path.dirname(sp), exist_ok=True)
with open(sp, "w") as sf:
    sf.write(os.getenv("APP_SECRETS"))
```

There should be five secrets in total:

![image](https://github.com/MayCXC/streamlit-to-azure-migration-mirror/assets/9441877/f600054a-6531-445d-a13e-2e8a7e7a837b)

Now navigate to **Actions > set Azure Vault secrets, (...)** and click **Run workflow > Run workflow**. The workflow run should succeed, and the web app will deploy when it does. This workflow needs to be run for changes to `APP_SECRETS` or the python scripts to be reflected by the web app.
