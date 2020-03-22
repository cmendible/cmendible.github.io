---
author: Carlos Mendible
categories:
- dotnetcore
crosspost_to_medium: true
date: "2017-03-02T07:59:30Z"
description: Twilio with .NET Core
image: /wp-content/uploads/2017/03/twilio-csharp-blog-logo.png
tags: ["SMS", "Twilio"]
title: Twilio with .NET Core
url: /2017/03/02/twilio-with-net-core/
---
Last night after reading this tweet, I decided to try out **<a href="https://www.twilio.com/" target="_blank">Twilio</a> with .NET Core** 

<blockquote class="twitter-tweet" data-width="550">
  <p lang="en" dir="ltr">
    Your feedback led to the newly redesigned C# helper library. <br />See what's new and take it for a spin here: <a href="https://t.co/IuD4NZLBH9">https://t.co/IuD4NZLBH9</a> <a href="https://t.co/SzLB7udU1h">pic.twitter.com/SzLB7udU1h</a>
  </p>
  
  <p>
    &mdash; twilio (@twilio) <a href="https://twitter.com/twilio/status/836723837559537671">February 28, 2017</a>
  </p>
</blockquote>

Let's create the sample console app:

## 1. Create the application
---
Open a command prompt and run 
    
``` powershell
    md twilio.console
    cd twilio.console
    dotnet new
    dotnet restore
    code .
```

## 2. Replace the contents of project.json
---
Replace the contents on **project.json** file in order to include the references to: **Twilio**
    
``` json
{
  "version": "1.0.0-*",
  "buildOptions": {
    "debugType": "portable",
    "emitEntryPoint": true
  },
  "dependencies": {
    "Twilio": "5.0.2"
  },
  "frameworks": {
    "netcoreapp1.1": {
      "dependencies": {
        "Microsoft.NETCore.App": {
          "type": "platform",
          "version": "1.1.0"
        }
      },
      "imports": "dnxcore50"
    }
  }
}
```

## 3. Replace the contents of Program.cs
---
The **CreateClass** method is where the magic occurs. Each relevant line is explained to help you understand each step. 
    
``` csharp
using System;
using System.Threading.Tasks;
using Twilio;
using Twilio.Rest.Api.V2010.Account;
using Twilio.Types;

namespace ConsoleApplication
{
    public class Program
    {
        public static void Main(string[] args)
        {
            // Send the SMS
            var messageSid = Task.Run(() => 
            { 
                return SendSms(); 
            })
            .GetAwaiter()
            .GetResult();

            // Write the message Id to the console.
            Console.WriteLine(messageSid);
            Console.Read();
        }

        // Send SMS with async feature!
        public static async Task<string> SendSms()
        {
            // Your Account SID from twilio.com/console
            var accountSid = "[Account SID]";

            // Your Auth Token from twilio.com/console
            var authToken = "[Auth Token]";

            // Initialize Twilio Client
            TwilioClient.Init(accountSid, authToken);

            // Create & Send Message (New lib supports async await)
            var message = await MessageResource.CreateAsync(
                to: new PhoneNumber("[Target Phone Number]"),
                from: new PhoneNumber("[Your Twilio Phone Number]"),
                body: "Hello from dotnetcore");

            return message.Sid;
        }
    }
}
```

## 4. Run the application
---
Open a command prompt and run 
    
``` powershell
    dotnet restore
    dotnet run
```

Get a copy of the code here: <https://github.com/cmendible/dotnetcore.samples/tree/master/twilio.console>

Hope it helps!