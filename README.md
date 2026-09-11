# Azure Static Web App CI/CD with Azure DevOps

This project demonstrates how to deploy a static HTML website from a **GitHub repository** to **Azure Static Web Apps** using an **Azure DevOps multi-stage YAML pipeline**.

The GitHub repository initially contains only:

```text
index.html
```

The Azure DevOps pipeline consists of two stages:

1. **Validate Build** – Validates that `index.html` exists.
2. **Deploy** – Deploys the website to Azure Static Web Apps.

The pipeline automatically runs whenever changes are pushed to the `main` branch.

---

# Architecture

```text
                  GitHub Repository
                         │
                         │ Push to main
                         ▼
                ┌──────────────────┐
                │   Azure DevOps   │
                │     Pipeline     │
                └────────┬─────────┘
                         │
                         ▼
                ┌──────────────────┐
                │ Stage 1: Validate│
                │                  │
                │ Check index.html │
                └────────┬─────────┘
                         │
                      Success
                         │
                         ▼
                ┌──────────────────┐
                │  Stage 2: Deploy │
                │                  │
                │ Azure Static Web │
                │      App         │
                └────────┬─────────┘
                         │
                         ▼
                    Live Website
```

---

# 1. Prerequisites

Before starting, you need:

* GitHub account
* Azure account
* Azure DevOps account
* Azure DevOps project
* GitHub repository
* Azure Static Web App

For this project, the GitHub repository initially contains only:

```text
my-static-website/
│
└── index.html
```

---

# 2. Create the Static Website in GitHub

Create a GitHub repository.

For example:

```text
my-static-website
```

Create an `index.html` file in the repository.

Example:

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Static Website</title>
</head>

<body>

    <h1>Welcome to My Static Website</h1>

    <p>
        This website is deployed using Azure DevOps
        and Azure Static Web Apps.
    </p>

</body>

</html>
```

At this point, the repository should contain:

```text
my-static-website/
│
└── index.html
```

---

# 3. Create Azure Static Web App

Go to the **Azure Portal**.

Navigate to:

```text
Azure Portal
      │
      ▼
Create a resource
      │
      ▼
Search: Static Web App
      │
      ▼
Create
```

Configure the required settings:

* Subscription
* Resource Group
* Static Web App name
* Region
* Hosting plan

For the deployment source, select:

```text
Other
```

We select **Other** because Azure DevOps will handle the deployment.

The deployment flow will be:

```text
GitHub
   │
   ▼
Azure DevOps
   │
   ▼
Azure Static Web App
```

Complete the Static Web App creation.

---

# 4. Get the Static Web App Deployment Token

After creating the Static Web App, open it in the Azure Portal.

Navigate to:

```text
Azure Portal
      │
      ▼
Static Web Apps
      │
      ▼
Your Static Web App
```

Find:

```text
Manage deployment token
```

Copy the deployment token.

This token will be used by Azure DevOps to authenticate with the Static Web App.

> **Important:** Do not add the deployment token directly to your YAML file. It will be stored securely in an Azure DevOps Variable Group.

---

# 5. Create the Azure DevOps Pipeline

Go to your Azure DevOps project.

Navigate to:

```text
Azure DevOps
      │
      ▼
Pipelines
      │
      ▼
New Pipeline
```

Azure DevOps will ask:

```text
Where is your code?
```

Select:

```text
GitHub
```

---

# 6. Connect GitHub to Azure DevOps

Azure DevOps will ask you to authorize GitHub.

Select:

```text
Authorize
```

You may be redirected to GitHub.

Sign in to GitHub and authorize Azure DevOps.

After authorization, Azure DevOps will be able to access your GitHub repositories.

```text
GitHub Account
      │
      │ Authorization
      ▼
Azure DevOps
      │
      ▼
GitHub Repositories
```

---

# 7. Select the GitHub Repository

Azure DevOps will display the repositories available through your GitHub connection.

Select:

```text
my-static-website
```

The repository contains:

```text
my-static-website/
│
└── index.html
```

---

# 8. Select Starter Pipeline

After selecting the repository, Azure DevOps will ask how you want to configure the pipeline.

Select:

```text
Starter pipeline
```

The Azure DevOps YAML editor will open.

Remove the default YAML content.

We will add our own two-stage pipeline.

---

# 9. Create the Azure DevOps Variable Group

The Azure Static Web App deployment token should be stored securely.

Go to:

```text
Azure DevOps
      │
      ▼
Pipelines
      │
      ▼
Library
      │
      ▼
Variable groups
```

Select:

```text
+ Variable group
```

Create the Variable Group:

```text
New variable group 11-Sep
```

Add the following variable:

| Variable                          | Value                           | Secret |
| --------------------------------- | ------------------------------- | ------ |
| `AZURE_STATIC_WEB_APPS_API_TOKEN` | Static Web App deployment token | Yes    |

Mark the variable as **secret**.

The Variable Group should look like:

```text
New variable group 11-Sep
│
└── AZURE_STATIC_WEB_APPS_API_TOKEN
       │
       └── Secret: Yes
```

---

# 10. Configure the Pipeline

The Variable Group is referenced in the pipeline using:

```yaml
variables:
- group: 'New variable group 11-Sep'
```

The deployment token can then be accessed using:

```text
$(AZURE_STATIC_WEB_APPS_API_TOKEN)
```

The actual token is never written into the YAML file.

---

# 11. Two-Stage Pipeline

The pipeline contains two stages:

```text
Pipeline
   │
   ├── Stage 1: Validate
   │
   └── Stage 2: Deploy
```

The complete YAML is:

```yaml
trigger:
- main

pool:
  vmImage: ubuntu-latest

variables:
- group: 'New variable group 11-Sep'

stages:

# ============================================================
# STAGE 1: VALIDATE
# ============================================================
- stage: Validate
  displayName: 'Validate Build'

  jobs:
  - job: ValidateWebsite
    displayName: 'Validate Static Website'

    steps:

    - checkout: self

    - script: |
        echo "Checking website files..."
        echo "Repository: $(Build.SourcesDirectory)"

        if [ ! -f "$(Build.SourcesDirectory)/index.html" ]; then
          echo "ERROR: index.html was not found."
          exit 1
        fi

        echo "index.html found successfully."
        echo "Website validation completed successfully."

      displayName: 'Validate Website'


# ============================================================
# STAGE 2: DEPLOY
# ============================================================
- stage: Deploy
  displayName: 'Deploy to Azure Static Web App'

  dependsOn: Validate
  condition: succeeded()

  jobs:
  - job: DeployWebsite
    displayName: 'Deploy Website'

    steps:

    - checkout: self

    - task: AzureStaticWebApp@0
      displayName: 'Deploy to Azure Static Web App'
      inputs:
        app_location: '/'
        output_location: ''
        skip_app_build: true
        skip_api_build: true
        azure_static_web_apps_api_token: '$(AZURE_STATIC_WEB_APPS_API_TOKEN)'
```

---

# 12. Stage 1 — Validate Build

The first stage checks whether `index.html` exists in the repository.

```text
GitHub Repository
       ↓
Checkout Code
       ↓
Check index.html
       ↓
Validation Successful
```

If `index.html` is not found, the stage fails and deployment does not continue.

---

# 13. Stage 2 — Deploy

The second stage runs only after the validation stage succeeds.

```text
Validate
   ↓
Deploy
   ↓
Azure Static Web App
```

The `AzureStaticWebApp@0` task deploys the website directly from the repository.

Because `index.html` is located in the repository root, the pipeline uses:

```yaml
app_location: '/'
```

---

# 14. Save and Run the Pipeline

After entering the YAML, select:

```text
Save and run
```

Azure DevOps will commit the YAML file to the repository.

The repository will now contain:

```text
my-static-website/
│
├── index.html
└── azure-pipelines.yml
```

---

# 15. Pipeline Execution

The pipeline will execute:

```text
Pipeline
│
├── Validate Build
│
└── Deploy to Azure Static Web App
```

A successful run will look like:

```text
✓ Validate Build
        │
        ▼
✓ Deploy to Azure Static Web App
        │
        ▼
✓ Website Deployed
```

---

# 16. Verify the Deployment

Go to the Azure Portal.

Navigate to:

```text
Azure Portal
      │
      ▼
Static Web Apps
      │
      ▼
Your Static Web App
```

Azure provides a URL similar to:

```text
https://your-app-name.azurestaticapps.net
```

Open the URL in your browser.

Your `index.html` page should now be displayed.

---

# 17. Automatic Deployment

The pipeline contains:

```yaml
trigger:
- main
```

Therefore, whenever you push changes to the `main` branch, Azure DevOps automatically starts the pipeline.

```text
Developer
    │
    ▼
Update index.html
    │
    ▼
git push origin main
    │
    ▼
GitHub
    │
    ▼
Azure DevOps
    │
    ├── Validate
    │
    └── Deploy
           │
           ▼
    Azure Static Web App
           │
           ▼
      Updated Website
```

---

# 18. Final Repository Structure

Initially:

```text
my-static-website/
│
└── index.html
```

After creating the pipeline:

```text
my-static-website/
│
├── index.html
└── azure-pipelines.yml
```

---

Every push to the `main` branch can automatically validate and deploy the latest version of the static website.
