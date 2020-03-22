---
author: Carlos Mendible
categories:
- dotnet
date: "2016-09-11T09:15:32Z"
description: .NET Core and Microsoft Bot Framework
image: /wp-content/uploads/2016/09/bot-framework-default-8.png
tags: ["aspNetCore", "BotFramewok", "REST"]
title: .NET Core and Microsoft Bot Framework
url: /2016/09/11/dotnetcore-and-microsoft-bot-framework/
---
Today I'll show you how to create a simple Skype bot that will reply with the same text message you send to it. 

You'll be using **.Net Core and <a href="https://dev.botframework.com/" target="_blank">Microsoft Bot Framework</a>**. As you already know not every library has been ported to .Net Core so you'll have to use the **<a href="https://docs.botframework.com/en-us/restapi/connector/#navtitle" target="_blank">Microsoft Bot Connector API &#8211; v3.0</a>** to bring the bot to life.

Let me show you what it takes to create a simple bot:

## 1. Register the bot
---
Head to the <a href="https://dev.botframework.com/bots/new" target="_blank">Register bot</a> page and fill the required fields.
    
For the messaging endpoint use something like: **<em>https://{url of your bot}/api/messages</em>**

Also get your Microsoft App ID and password from the Microsoft Application registration portal, and save them cause you'll need them to fill the BotCredentials in our sample.
      
## 2. Create a new .NET Core Web the project
---

``` powershell
md bot
cd bot
dotnet new -t web
```
      
## 3. Add the following dependencies to your project.json file
---     

``` json
    "Microsoft.AspNet.WebApi.Client": "5.2.3",
    "System.Runtime.Serialization.Xml": "4.1.1",
    "Microsoft.Extensions.Caching.Memory": "1.0.0",
    "Microsoft.Extensions.Options.ConfigurationExtensions": "1.0.0",
    "Microsoft.Extensions.DependencyInjection": "1.0.0",
    "Microsoft.Extensions.DependencyInjection.Abstractions": "1.0.0"
```

And restore the packages 
          
``` powershell
dotnet restore
```    
      
## 4. Add the following section to your appsettings.json file
---     
        
Replace the values with those of your registered bot.   
          
``` json
"BotCredentials": {
    "ClientId": "{Replace with your ClientId}",
    "ClientSecret": "{Replace with your ClientSecret}"
  }
```
      
## 5. Replace the ConfigureServices in you Startup class
You'll be using options to inject configurations (BotCredentials) to the controller and in Memory Cache to save tokens to avoid authenticating the bot on every call.
                
``` csharp
    // This method gets called by the runtime. Use this method to add services to the container.
    public void ConfigureServices(IServiceCollection services)
    {
        // Adding options so we can inject configurations.
        services.AddOptions();

        // Register the bot credentials section
        services.Configure<BotCredentials>(Configuration.GetSection("BotCredentials"));

        // we'll be catching tokens, so enable MemoryCache.
        services.AddMemoryCache();

        services.AddMvc();
     }
```
      
## 6. Create a BotCredentials class
This class will hold the bot credentials
      
``` csharp
namespace WebApplication
{
    /// <summary>
    /// Bot Credentials
    /// </summary>
    public class BotCredentials
    {
        /// <summary>
        /// CLientId
        /// </summary>
        public string ClientId { get; set; }

        /// <summary>
        /// ClientSecret
        /// </summary>
        public string ClientSecret { get; set; }
    }
}
```  
      
## 7. Create TokenResponse class
      
``` csharp
namespace WebApplication
{
    /// <summary>
    /// Token Response
    /// </summary>
    public class TokenResponse
    {
        /// <summary>
        /// Property to hold the token
        /// </summary>
        public string access_token { get; set; }
    }
}
```

## 8. Create a MessagesController class
     
``` csharp
namespace WebApplication.Controllers
{
    using System;
    using System.Collections.Generic;
    using System.Dynamic;
    using System.Net.Http;
    using System.Net.Http.Headers;
    using System.Threading.Tasks;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Extensions.Caching.Memory;
    using Microsoft.Extensions.Options;

    /// <summary>
    /// This controller will receive the skype messages and handle them to the EchoBot service. 
    /// </summary>
    [Route("api/[controller]")]
    public class MessagesController : Controller
    {
        /// <summary>
        /// memoryCache
        /// </summary>
        IMemoryCache memoryCache;

        /// <summary>
        /// Bot Credentials
        /// </summary>
        BotCredentials botCredentials;

        /// <summary>
        /// Constructor
        /// </summary>
        /// 
        /// 
        public MessagesController(IMemoryCache memoryCache, IOptions<BotCredentials> botCredentials)
        {
            this.memoryCache = memoryCache;
            this.botCredentials = botCredentials.Value;
        }

        /// <summary>
        /// This method will be called every time the bot receives an activity. This is the messaging endpoint
        /// </summary>
        /// 
        /// <returns>201 Created</returns>
        [HttpPost]
        public virtual async Task<IActionResult> Post([FromBody] dynamic activity)
        {
            // Get the conversation id so the bot answers.
            var conversationId = activity.from.id.ToString();

            // Get a valid token 
            string token = await this.GetBotApiToken();

            // send the message back
            using (var client = new HttpClient())
            {
                // I'm using dynamic here to make the code simpler
                dynamic message = new ExpandoObject();
                message.type = "message/text";
                message.text = activity.text;

                // Set the toekn in the authorization header.
                client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);

                // Post the message
                await client.PostAsJsonAsync<ExpandoObject>(
                    $"https://api.skype.net/v3/conversations/{conversationId}/activities",
                    message as ExpandoObject);
            }

            return Created(Url.Content("~/"), string.Empty);
        }

        /// <summary>
        /// Gets and caches a valid token so the bot can send messages.
        /// </summary>
        /// <returns>The token</returns>
        private async Task<string> GetBotApiToken()
        {
            // Check to see if we already have a valid token
            string token = memoryCache.Get("token")?.ToString();
            if (string.IsNullOrEmpty(token))
            {
                // we need to get a token.
                using (var client = new HttpClient())
                {
                    // Create the encoded content needed to get a token
                    var parameters = new Dictionary<string, string>
                    {
                        {"client_id", this.botCredentials.ClientId },
                        {"client_secret", this.botCredentials.ClientSecret },
                        {"scope", "https://graph.microsoft.com/.default" },
                        {"grant_type", "client_credentials" }
                    };
                    var content = new FormUrlEncodedContent(parameters);

                    // Post
                    var response = await client.PostAsync("https://login.microsoftonline.com/common/oauth2/v2.0/token", content);

                    // Get the token response
                    var tokenResponse = await response.Content.ReadAsAsync<TokenResponse>();

                    token = tokenResponse.access_token;

                    // Cache the token for 15 minutes.
                    memoryCache.Set(
                        "token",
                        token,
                        new DateTimeOffset(DateTime.Now.AddMinutes(15)));
                }
            }

            return token;
        }
    }
}
```  
      
## 9. Deploy your bot and add it to your skype
---      

1. Deploy your bot (I deployed it to an Azure Web App)
2. Head to the **My Bots** section of the <a href="https://dev.botframework.com/" target="_blank">Microsoft Bot Framework</a> page and add the bot to your skype contacts.
      
## 10. Talk to your bot
---      
No you can talk to your bot!
          
<a href="/wp-content/uploads/2016/09/pancito.png"><img src="/wp-content/uploads/2016/09/pancito-300x195.png" alt="pancito" width="300" height="195" class="alignleft size-medium wp-image-5881" srcset="/wp-content/uploads/2016/09/pancito-300x195.png 300w, /wp-content/uploads/2016/09/pancito-768x500.png 768w, /wp-content/uploads/2016/09/pancito-1024x667.png 1024w, /wp-content/uploads/2016/09/pancito-250x163.png 250w, /wp-content/uploads/2016/09/pancito.png 1444w" sizes="(max-width: 300px) 100vw, 300px" /></a>
            
You can get the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/echobot">https://github.com/cmendible/dotnetcore.samples/tree/master/echobot</a>
              
Hope it helps!