---
author: Carlos Mendible
categories:
- dotnet
- azure
date: "2022-01-07T10:00:00Z"
description: 'Protect your Node.js or .NET API with Azure Active Directory)'
images: ["/assets/img/posts/aad.png"]
draft: true
tags: ["dotnet", "nodejs", "aad", "azure active directory"]
title: 'Protect your Node.js or .NET API with Azure Active Directory'
---

One question I often get from by my customers is to show then how to use Azure Active Directroy to protect their Node.js or .NET APIs. 

Every songle time I answer by redirecting them to this amazing post ([Proteger una API en Node.js con Azure Active Directory](https://www.returngis.net/2021/04/proteger-una-api-en-node-js-con-azure-active-directory/)), written in spanish, by my friend and peer Gisela Torres ([0gis0](https://twitter.com/0gis0)).

Sometimes they come back with more questions:

* How do we use Terrafrom to register the API and Client in Azure Active Directory?
* How can we validate the scope in the Node.js API when using JwtStrategy strategy?
* Can you provide a .NET application sample that performs the same validation and without boilerplate?

So  in this post what I'm going to do is answer those last 3 questions based on Gisela's [code](https://github.com/0GiS0/protected-nodejs-api-with-azure-ad) and alos create a PowerShell client to test your APIs:

## Terraform script to register the API and Client with Azure Active Directory

### Create main.tf with the following contents:

``` terraform
terraform {
  required_version = "> 0.14"
  required_providers {
    azuread = {
      version = ">= 2.6.0"
    }
    azurerm = {
      version = ">= 2.80.0"
    }
  }
}

provider "azurerm" {
  features {}
}

data "azurerm_client_config" "current" {}

// This is the AAD Application Registration for the API
resource "azuread_application" "api" {
  display_name    = "passport-test-api"
  identifier_uris = ["api://passport-test-api"]


  app_role {
    allowed_member_types = ["User"]
    description          = "ReadOnly roles have limited query access"
    display_name         = "ReadOnly"
    enabled              = true
    id                   = "497406e4-012a-4267-bf18-45a1cb148a01"
    value                = "User"
  }

  // Add access to User.Read.All (Microsoft Graph)
  required_resource_access {
    resource_app_id = "00000003-0000-0000-c000-000000000000" # Microsoft Graph

    resource_access {
      id   = "df021288-bdef-4463-88db-98f22de89214" # User.Read.All
      type = "Role"
    }
  }

  api {
    mapped_claims_enabled          = true
    requested_access_token_version = null

    // Add our sample client as a known Application
    known_client_applications = [
      azuread_application.client.application_id
    ]

    // This is the scope we'll validate in our APIs
    oauth2_permission_scope {
      admin_consent_description  = "Allow the application to access example on behalf of the signed-in user."
      admin_consent_display_name = "Access example"
      enabled                    = true
      id                         = "a7ef8bb6-5085-49a1-b803-517b5a439668"
      type                       = "User"
      value                      = "read"
    }
  }
}

// This is the AAD Application Registration for the client.
resource "azuread_application" "client" {
  display_name = "passport-client"

  // We'll be using a PowerShell client
  public_client {
    redirect_uris = [
      "http://localhost/",
    ]
  }
  
  api {
    known_client_applications      = []
    mapped_claims_enabled          = false
    requested_access_token_version = null
  }

  // You can also use Gisela's web client
  web {
    redirect_uris = [
      "http://localhost:8000/give/me/the/code"
    ]
  }
}

// Pre authorize our client
resource "azuread_application_pre_authorized" "pre_authorized" {
  application_object_id = azuread_application.api.object_id
  authorized_app_id     = azuread_application.client.application_id
  permission_ids        = ["a7ef8bb6-5085-49a1-b803-517b5a439668"]
}

// You'll need the following output values to configure your application na use the PowerShell client
output "tenant_id" {
  description = "TENANT_ID"
  value       = data.azurerm_client_config.current.tenant_id
}

output "api_client_id" {
  description = "API CLIENT_ID"
  value       = azuread_application.api.application_id
}

output "client_id" {
  description = "client CLIENT_ID"
  value       = azuread_application.client.application_id
}

output "powershell_command" {
  value     = "./client.ps1 ${data.azurerm_client_config.current.tenant_id} ${azuread_application.client.application_id}"
}
```

### Deploy the Application Registrations:

Run the following commands:

``` shell
terraform init
terraform apply
```

## Add scope validation to Gisela's passport / passport-jwt Node.js sample.

As mentioned by Gisela, the [JwtStrategy](http://www.passportjs.org/packages/passport-jwt/) expects configuration options (via a **jwtOptions** object) and also a callback function that can be used to validate the user, scope, etc... 

### Validating the scope:

I've modified the `verify` function in order to validate the scope against the value configured in the `SCOPE` environment variable. 

``` javascript
const verify = (jwt_payload, done) => {
    console.log(`Signature is valid for the JSON Web Token (JWT), let's check other things...`);
    console.log(jwt_payload);

    let tokenScope = `${jwt_payload.aud}/${jwt_payload.scp}`
    if (jwt_payload && jwt_payload.sub && process.env.SCOPE == tokenScope) {
        return done(null, jwt_payload);
    }

    return done(null, false);
};
```

> The full scope (`tokenScope`) is composed by `jwt_payload.aud` and `jwt_payload.scp`

### Full Node.js API:

Now that you learned how to validate the scope the ful Node.js API should look like this:

``` javascript
const express = require('express'),
    app = express();

require('dotenv').config();

//Modules to use passport
const passport = require('passport'),
    JwtStrategy = require('passport-jwt').Strategy,
    ExtractJwt = require('passport-jwt').ExtractJwt,
    jwks = require('jwks-rsa');

let jwtOptions = {
    jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
    // Dynamically provide a signing key based on the kid in the header and the signing keys provided by the JWKS endpoint.
    secretOrKeyProvider: jwks.passportJwtSecret({
        jwksUri: `https://login.microsoftonline.com/${process.env.TENANT_ID}/discovery/v2.0/keys`,
    }),
    algorithms: ['RS256'],
    audience: process.env.AUDIENCE,
    issuer: `https://sts.windows.net/${process.env.TENANT_ID}/`
};

const verify = (jwt_payload, done) => {
    console.log(`Signature is valid for the JSON Web Token (JWT), let's check other things...`);
    console.log(jwt_payload);

    let tokenScope = `${jwt_payload.aud}/${jwt_payload.scp}`
    if (jwt_payload && jwt_payload.sub && process.env.SCOPE == tokenScope) {
        return done(null, jwt_payload);
    }

    return done(null, false);
};

passport.use(new JwtStrategy(jwtOptions, verify));

app.get("/protected", passport.authorize('jwt', { session: false }), function (req, res) {
    res.json({ message: "This message is protected" });
});

app.listen(1000, () => {
    console.log(`API running on port 1000!`);
});
```

### Run the Node.js application:

Run the follwoing command:

``` shell
node main.ts
```

## Create a PowerShell script to test the protected APIs.

### Create a client.ps1 file with the following contents:

``` powershell
param(
    [Parameter(Mandatory=$true)]
    [string]
    $tenantId,
    [Parameter(Mandatory=$true)]
    [string]
    $clientId
)

if ((get-module MSAL.PS) -eq $null)
{
    echo "installing MSAL.PS"
    Install-Module -Name MSAL.PS -Scope CurrentUser -AcceptLicense -Force 
    # If you encounter this error:
    # WARNING: The specified module 'MSAL.PS' with PowerShellGetFormatVersion '2.0' is not supported by the current version of PowerShellGet. 
    # Get the latest version of the PowerShellGet module to install this module, 'MSAL.PS'
    # Install as Admin:
    # Install-PackageProvider NuGet -Force
    # Install-Module PowerShellGet -Force
}

$scope = "api://passport-test-api/read"
$redirectUri = "http://localhost"
$url = "http://localhost:1000/protected"
$token = Get-MsalToken -TenantId $tenantId -ClientId $clientId -Interactive -Scope $scope -RedirectUri $redirectUri

echo "Please Complete Azure AD Login"
echo ""
echo "ID Token:"
echo $($token.IDToken)
echo ""
echo "Bearer Token:"
echo $($token.AccessToken)
echo ""
echo "Protect API Call:"
curl -k -i -H "Authorization: Bearer $($token.AccessToken)" $url
```

> The PowerShell client application uses the **MSAL.PS** module to get an AAD token and then call the protected API  

### Test the Node.js API:

To get the command to test the API run:

``` shell
terraform output powershell_command
```

The returned value should look like this:

``` shell
./client.ps1 <tenant_id> <application_id>"
```

Now run the given command and login with your credentials. The console output should show the following information: 

* ID Token
* Bearer Token
* The `This message is protected` message returned by the API.

Congratulations you just called a protected Node.js API using a PowerShell client.

## Create a .NET API protected with AAD.

### Run the following commands to create the .NET API:

``` shell
mkdir dotnet-sample
cd dotnet-sample
dotnet new web 
dotnet add package Microsoft.Identity.Web -v 1.18.0
```

Also replace the `applicationUrl` in the `dotnet-sample` section of the **Properties/launchSettings.json** file with:

``` json
"applicationUrl": "http://localhost:1000",
```

> This will make the .NET application use the same port (1000) as the Node.js application. 

### Replace the contents of **Program.cs** with:

``` csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore;
using Microsoft.AspNetCore.Builder;
using Microsoft.Extensions.DependencyInjection;
using System.Text.Json;
using Microsoft.Identity.Web;
using Microsoft.Extensions.Configuration;
using System.IO;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using System.Collections.Generic;

var config = new ConfigurationBuilder()
                    .SetBasePath(Directory.GetCurrentDirectory())
                    .AddJsonFile("appsettings.json")
                    .AddEnvironmentVariables()
                    .Build();

WebHost.CreateDefaultBuilder().
ConfigureServices(s =>
{
    s.AddSingleton(new JsonSerializerOptions()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        PropertyNameCaseInsensitive = true,
    });

    s.AddMicrosoftIdentityWebApiAuthentication(config);

    s.Configure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, options =>
    {
        options.TokenValidationParameters.ValidAudiences = new List<string>() { config["Audience"] };
    });
}).
Configure(app =>
{
    app.UseRouting();
    app.UseAuthentication();
    app.UseAuthorization();

    app.UseEndpoints(e =>
    {
        e.MapGet("/protected",
            async c =>
            {
                var serializerOptions = e.ServiceProvider.GetRequiredService<JsonSerializerOptions>();
                var data = new { message = "This message is protected" };

                c.Response.ContentType = "application/json";
                await JsonSerializer.SerializeAsync(c.Response.Body, data, serializerOptions);
            })
            .RequireAuthorization()
            .RequireScope("read");
    });
}).Build().Run();
```

> .NET Top level programs are amazing!

> Check the code to understand where does the audience and scope validation occurs.

### Replace the contents of the **appsettings.json** file with:

``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*",
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "ClientId": "<web api client id>",
    "Domain": "<tenant domain (i.e contoso.onmicrosoft.com)>",
    "TenantId": "<tenant id>"
  },
  "Audience": "api://passport-test-api"
}
```

**Note**: set the proper values for `ClientId`, `Domain` and `TenantId` in the `AzureAd` section.

### Run the .NET application:

> Make sure you stop the Node.js application. 

Run the following command:

``` shell
dotnet run
```

### Test the .NET API:

To get the command to test the API run:

``` shell
terraform output powershell_command
```

The returned value should look like this:

``` shell
./client.ps1 <tenant_id> <application_id>"
```

Now run the given command and login with your credentials. The console output should show the following information: 

* ID Token
* Bearer Token
* The `This message is protected` message returned by the API.

Congratulations you just called a protected .NET API using a PowerShell client.

Hope it helps!!!

Please find the complete samples [here](https://github.com/cmendible/azure.samples/tree/main/aad_protect_web_api)

References:

* [Proteger una API en Node.js con Azure Active Directory](https://www.returngis.net/2021/04/proteger-una-api-en-node-js-con-azure-active-directory/)
