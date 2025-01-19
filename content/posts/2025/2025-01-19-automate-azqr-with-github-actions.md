---
author: Carlos Mendible
categories:
- azure
date: "2025-01-19"
description: 'Automate Azure Quick Review with GitHub Actions'
draft: false
tags: ["azqr", "github-actions", "assessment"]
title: 'Automate Azure Quick Review with GitHub Actions'
---

Today we will walk through a GitHub Actions workflow that automates the Azure Quick Review (azqr) scan process. This workflow is designed to run on a schedule, on push events to the main branch, and on pull requests to the main branch.

## Prerequisites

Before you start, make sure you have the following prerequisites in place:

- An Azure subscription with resources to scan.
- Azure credentials in the form of a Service Principal with the following values set as GitHub Secrets:
  - `AZURE_CLIENT_ID`
  - `AZURE_CLIENT_SECRET`
  - `AZURE_TENANT_ID`

Let's break down each part of the workflow.

## Workflow Triggers

The workflow is triggered by three events:

- `workflow_dispatch`: Allows manual triggering of the workflow.
- `schedule`: Runs the workflow every Friday at midnight using a cron expression (`0 0 * * 5`).
- `push` and `pull_request`: Triggers the workflow on push and pull request events to the `main` branch.

## Jobs and Steps

The workflow consists of a single job named `scan` that runs on the latest version of Ubuntu. Here are the steps involved:

1. **Checkout the Repository**

    ```yaml
    - name: Checkout repository
      uses: actions/checkout@v2
    ```

    This step checks out the repository to the GitHub Actions runner. 
    
    > You'll need this if you work with `azqr` configuration files or other resources in your repository.

2. **Install azqr**

    ```yaml
    - name: Install azqr
      run: |
         latestAzqr=$(curl -sL https://api.github.com/repos/Azure/azqr/releases/latest | jq -r ".tag_name" | cut -c1-) \
         && wget https://github.com/Azure/azqr/releases/download/$latestAzqr/azqr-ubuntu-latest-amd64 -O /usr/local/bin/azqr \
         && chmod +x /usr/local/bin/azqr
    ```

    This step installs the latest version of azqr by downloading it from the GitHub releases page and making it executable.

3. **Run azqr Scan**

    ```yaml
    - name: Run azqr scan
      env:
         AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
         AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
         AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      run: |
         timestamp=$(date '+%Y%m%d%H%M%S')
         echo "DATETIME=$timestamp" >> $GITHUB_ENV
         azqr scan -o "${{ github.workspace }}/azqr_action_plan_$timestamp"
    ```
    
    This step runs the azqr scan using the provided Azure credentials stored in GitHub Secrets. The output is saved with a timestamp to ensure uniqueness.

4. **Publish azqr Action Plan**

    ```yaml
    - name: Publish azqr action plan
      uses: actions/upload-artifact@v2
      with:
         name: azqr_result
         path: ${{ github.workspace }}/azqr_action_plan_${{ env.DATETIME }}.xlsx
    ```

    This step uploads the azqr action plan as an artifact, making it available for download from the workflow run summary.

By following this workflow, you can automate the process of running Azure Quick Reviews and ensure that the results are consistently generated and stored. This can help you maintain a regular assessment of your Azure resources and stay on top of any potential issues.

## Full Workflow File

Check the full workflow file below:

```yaml
# GitHub Actions workflow for azqr

name: azqr-pipeline

on:
  workflow_dispatch: # Trigger manually
  schedule: # Trigger every Friday at midnight
    - cron: "0 0 * * 5"
  push:
    branches:
      - main # Trigger on push to main
  pull_request:
    branches:
      - main # Trigger on pull request to main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      # Checkout the repository
      - name: Checkout repository
        uses: actions/checkout@v2

      # Install azqr
      - name: Install azqr
        run: |
          latestAzqr=$(curl -sL https://api.github.com/repos/Azure/azqr/releases/latest | jq -r ".tag_name" | cut -c1-) \
          && wget https://github.com/Azure/azqr/releases/download/$latestAzqr/azqr-ubuntu-latest-amd64 -O /usr/local/bin/azqr \
          && chmod +x /usr/local/bin/azqr

      # Run azqr scan
      - name: Run azqr scan
        env:
          AZURE_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          AZURE_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          AZURE_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
        run: |
          timestamp=$(date '+%Y%m%d%H%M%S')
          echo "DATETIME=$timestamp" >> $GITHUB_ENV
          azqr scan -o "${{ github.workspace }}/azqr_action_plan_$timestamp"

      # Publish azqr action plan
      - name: Publish azqr action plan
        uses: actions/upload-artifact@v2
        with:
          name: azqr_result
          path: ${{ github.workspace }}/azqr_action_plan_${{ env.DATETIME }}.xlsx
```

Hope it helps!

**References:**

* [azqr GitHub repo](https://github.com/azure/azqr)
* [azqr documentation](https://azure.github.io/azqr)
