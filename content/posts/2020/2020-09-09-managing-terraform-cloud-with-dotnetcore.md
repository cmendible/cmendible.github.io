---
author: Carlos Mendible
categories:
- dotnet
crosspost_to_medium: true
date: "2020-09-09T10:00:00Z"
description: 'Managing Terraform Cloud with .NET Core'
images: ["/assets/img/posts/terraform_dotnetcore.png"]
published: true
tags: ["terraform"]
title: 'Managing Terraform Cloud with .NET Core'
---

Today I'm going to show you how to mange [Terraform Cloud](https://app.terraform.io/) with .NET Core using the [Tfe.NetClient](https://github.com/everis-technology/Tfe.NetClient) library.

The idea is to create a simple console application that will:

* Add GitHub as a [VCS Provider](https://www.terraform.io/docs/cloud/vcs/index.html).
* Create a [Workspace](https://www.terraform.io/docs/cloud/workspaces/index.html) conected to a GitHub repo where your Terraform files live. 
* Create a [variable](https://www.terraform.io/docs/cloud/workspaces/variables.html) in the workspace.
* Create a Run (Plan) based on the Terraform files 
* Apply the Run.

> [Tfe.NetClient](https://github.com/everis-technology/Tfe.NetClient) is still in alpha and not every Terraform Cloud API or feature is present. Please feel free to submit any issues, bugs or pull requests.

## Prerequisites

* A [Terraform Cloud](https://app.terraform.io/) Account (You can create a [Free Organization](https://www.terraform.io/docs/cloud/paid.html#free-organizations))
* A [Terraform Cloud](https://app.terraform.io/) Organization
* A [Terraform Cloud](https://app.terraform.io/) [Organization Token](https://www.terraform.io/docs/cloud/users-teams-organizations/api-tokens.html#organization-api-tokens)
* A [Terraform Cloud](https://app.terraform.io/) [Team Token](https://www.terraform.io/docs/cloud/users-teams-organizations/api-tokens.html#team-api-tokens)
* A [GitHub Personal Access Token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) with the following permissions:
    * repo: repo:status Access commit status
    * repo: repo_deployment Access deployment status
    * repo: public_repo Access public repositories
    * repo: repo:invite Access repository invitations
    * repo: security_events Read and write security events
    * workflow
* A GitHub Repo with the Terraform script you want to run.

## 1. Create a folder for your new project
---
Open a command prompt an run:

``` shell
mkdir TerraformCloud
```

## 2. Create the project
---

``` shell
cd TerraformCloud
dotnet new console
```

## 3. Add a reference to Tfe.NetClient
---

``` shell
dotnet add package Tfe.NetClient -v 0.1.0
dotnet restore
```

## 4. Replace the contents of Program.cs with the following code
---

```csharp
namespace TerraformCloud
{
    using System;
    using System.Net.Http;
    using System.Threading.Tasks;
    using Tfe.NetClient;
    using Tfe.NetClient.OAuthClients;
    using Tfe.NetClient.Runs;
    using Tfe.NetClient.Workspaces;
    using Tfe.NetClient.WorkspaceVariables;

    class Program
    {
        static async Task Main(string[] args)
        {
            // The values of this variables are hardcoded here just for simplicity and should be retrieved from configuration.
            var organizationName = "<organizationName>";
            var organizationToken = "<organizationToken>";
            var teamToken = "<teamToken>";
            var gitHubToken = "<GitHub Personal Access Token>";
            var gitHubRepo = "<github user or organization name>/<repo name>"; // i.e. cmendible/terraform-hello-world

            // Create an HttpClient
            var httpClient = new HttpClient();

            // Create the Configiration used by the TFE client.
            // For management tasks you'll need to connect to Terraform Cloud using an Organization Token.
            var config = new TfeConfig(organizationToken, httpClient);

            // Create the TFE client.
            var client = new TfeClient(config);

            // Connect Terraform Cloud and GitHub adding GitHub as a VCS Provider.
            var oauthClientsRequest = new OAuthClientsRequest();
            oauthClientsRequest.Data.Attributes.ServiceProvider = "github";
            oauthClientsRequest.Data.Attributes.HttpUrl = new Uri("https://github.com");
            oauthClientsRequest.Data.Attributes.ApiUrl = new Uri("https://api.github.com");
            oauthClientsRequest.Data.Attributes.OAuthTokenString = gitHubToken; // Use the GitHub Personal Access Token
            var oauthResult = await client.OAuthClient.CreateAsync(organizationName, oauthClientsRequest);

            // Get the OAuthToken.
            var oauthTokenId = oauthResult.Data.Relationships.OAuthTokens.Data[0].Id;

            // Create a Workspace connected to a GitHub repo.
            var workspacesRequest = new WorkspacesRequest();
            workspacesRequest.Data.Attributes.Name = "my-workspace";
            workspacesRequest.Data.Attributes.VcsRepo = new RequestVcsRepo();
            workspacesRequest.Data.Attributes.VcsRepo.Identifier = gitHubRepo; // Use the GitHub Repo
            workspacesRequest.Data.Attributes.VcsRepo.OauthTokenId = oauthTokenId;
            workspacesRequest.Data.Attributes.VcsRepo.Branch = "";
            workspacesRequest.Data.Attributes.VcsRepo.DefaultBranch = true;
            var workspaceResult = await client.Workspace.CreateAsync(organizationName, workspacesRequest);

            // Get the Workspace Id so we can add variales or request a plan or apply.
            var workspaceId = workspaceResult.Data.Id;

            // Create a variable in the workspace.
            // You can make the values invible setting the Sensitive attribute to true.
            // If you want to se an environement variable change the Category attribute to "env".
            // You'll have to create as any variables your script needs.
            var workspaceVariablesRequest = new WorkspaceVariablesRequest();
            workspaceVariablesRequest.Data.Attributes.Key = "variable_1";
            workspaceVariablesRequest.Data.Attributes.Value = "variable_1_value";
            workspaceVariablesRequest.Data.Attributes.Description = "variable_1 description";
            workspaceVariablesRequest.Data.Attributes.Category = "terraform";
            workspaceVariablesRequest.Data.Attributes.Hcl = false;
            workspaceVariablesRequest.Data.Attributes.Sensitive = false;
            var variableResult = await client.WorkspaceVariable.CreateAsync(workspaceId, workspaceVariablesRequest);

            // Get the workspace by name.
            var workspace = client.Workspace.ShowAsync(organizationName, "my-workspace");

            // To create Runs and Apply thme you need to use a Team Token.
            // So create a new TfeClient.
            var runsClient = new TfeClient(new TfeConfig(teamToken, new HttpClient()));

            // Create the Run.
            // This is th equivalent to running: terraform plan. 
            var runsRequest = new RunsRequest();
            runsRequest.Data.Attributes.IsDestroy = false;
            runsRequest.Data.Attributes.Message = "Triggered by .NET Core";
            runsRequest.Data.Relationships.Workspace.Data.Type = "workspaces";
            runsRequest.Data.Relationships.Workspace.Data.Id = workspace.Result.Data.Id;
            var runsResult = await runsClient.Run.CreateAsync(runsRequest);

            // Get the Run Id. You'll need it to check teh state of the run and Apply it if possible.
            var runId = runsResult.Data.Id;

            var ready = false;
            while (!ready)
            {
                // Wait for the Run to be planned .
                await Task.Delay(5000);
                var run = await client.Run.ShowAsync(runId);
                ready = run.Data.Attributes.Status == "planned";

                // Throw an exception if the Run status is: errored.
                if (run.Data.Attributes.Status == "errored") {
                    throw new Exception("Plan failed...");
                }
            }

            // If the Run is planned then Apply your configuration.
            if (ready)
            {
                await runsClient.Run.ApplyAsync(runId, null);
            }
        }
    }
}
```

> [Tfe.NetClient](https://github.com/everis-technology/Tfe.NetClient) is still in early stages of development and the resulting code is very verbose and prone to errors. We will address this in a future relases introducing the use of enums and perhaps a fluent API. 

## 5. Run the program
---

Run the following command:

``` shell
dotnet run
```

## 6. Check the results
---

Log In to [Terraform Cloud](https://app.terraform.io/) and check the status of the new workspace.

Hope it helps! and please find the complete code [here](https://github.com/cmendible/dotnetcore.samples/tree/main/terraform.cloud)