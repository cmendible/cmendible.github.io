---
author: Carlos Mendible
categories:
- azure
- dotnet
date: "2016-08-28T18:45:28Z"
description: Create vCard QR Codes using Azure Functions
images: ["/wp-content/uploads/2016/08/qrcodedark.png"]
tags: ["azure functions", "serverless"]
title: Create vCard QR Codes using Azure Functions
---
Today I'll show you how to develop a Web API to **Create vCard QR Codes using <a href="https://azure.microsoft.com/en-us/services/functions/" target="_blank">Azure Functions</a>**.

But wait what are <a href="https://azure.microsoft.com/en-us/services/functions/" target="_blank">Azure Functions</a>?

As defined by Microsoft:

> <a href="https://azure.microsoft.com/en-us/services/functions/" target="_blank">Azure Functions</a> is a serverless event driven experience that extends the existing Azure App Service platform. These nano-services can scale based on demand and you pay only for the resources you consume.

And what does serverless means? I really like the definition given by Scot Hanselman in his post <a href="http://www.hanselman.com/blog/WhatIsServerlessComputingExploringAzureFunctions.aspx" target="_blank">What is Serverless Computing? Exploring Azure Functions</a>:

> **Serverless Computing is like this &#8211; Your code, a slider bar, and your credit card**. You just have your function out there and it will scale as long as you can pay for it. It's as close to"cloudy" as The Cloud can get.

Now let's start:

## 1. Create a Function App
---
Head to <a href="http://portal.azure.com" target="_blank">portal.azure.com</a> and hit the New button. Search for **Function App** and create one. You'll be asked for an app name, resource group, app service plan and storage account where the code will live.
    
<a href="/wp-content/uploads/2016/08/functionapp-1.png"><img src="/wp-content/uploads/2016/08/functionapp-1-186x300.png" alt="functionapp" width="186" height="300" class="alignleft size-medium wp-image-5601" srcset="/wp-content/uploads/2016/08/functionapp-1-186x300.png 186w, /wp-content/uploads/2016/08/functionapp-1-634x1024.png 634w, /wp-content/uploads/2016/08/functionapp-1-250x404.png 250w, /wp-content/uploads/2016/08/functionapp-1.png 639w" sizes="(max-width: 186px) 100vw, 186px" /></a>
      
## 2. Create the function
---      
Create a new Function, selecting the empty C# template and give it a name: (i.e QRCoder)
          
<a href="/wp-content/uploads/2016/08/qrcoder.png"><img src="/wp-content/uploads/2016/08/qrcoder.png" alt="qrcoder" class="alignleft size-medium wp-image-5621" srcset="/wp-content/uploads/2016/08/qrcoder.png 1900w, /wp-content/uploads/2016/08/qrcoder-300x205.png 300w, /wp-content/uploads/2016/08/qrcoder-768x525.png 768w, /wp-content/uploads/2016/08/qrcoder-1024x701.png 1024w, /wp-content/uploads/2016/08/qrcoder-250x171.png 250w" sizes="(max-width: 1900px) 100vw, 1900px" /></a>
         
## 3. Add the code
---
Replace the contents of the **Code (run.csx)** section with the following code and save it:   
          
``` csharp
#r "System.Drawing"
#r "QRCoder.dll"

using System.Drawing;
using System.Drawing.Imaging;
using System.IO;
using System.Net;
using System.Net.Http;
using System.Net.Http.Headers;
using QRCoder;

public static async Task<HttpResponseMessage> Run(HttpRequestMessage req, TraceWriter log)
{
    // Read the json request
    var qrRequest = await req.Content.ReadAsAsync<SimpleVCardRequest>();

    // Create the vCard string
    var vCard = "BEGIN:VCARD\n";
    vCard += $"FN:{qrRequest.Name}\n";
    vCard += $"TEL;WORK;VOICE:{qrRequest.Phone}\n";
    vCard += "END:VCARD";

    // Generate de QRCode
    QRCodeGenerator qrGenerator = new QRCodeGenerator();
    QRCodeData qrCodeData = qrGenerator.CreateQrCode(vCard, QRCodeGenerator.ECCLevel.Q);
    QRCode qrCode = new QRCode(qrCodeData);

    // Save the QRCode as a jpeg image and send it in the response.
    using (Bitmap qrCodeImage = qrCode.GetGraphic(20))
    using (MemoryStream ms = new MemoryStream())
    {
        qrCodeImage.Save(ms, ImageFormat.Jpeg);

        var response = new HttpResponseMessage()
        {
            Content = new ByteArrayContent(ms.ToArray()),
            StatusCode = HttpStatusCode.OK,
        };
        response.Content.Headers.ContentType = new MediaTypeHeaderValue("image/jpeg");
        return response;
    }    
}

// Request class to hold the Name and Phone number used to generated the vCard QR Code
public class SimpleVCardRequest
{
    public string Name { get; set; }
    public string Phone { get; set; }
}
```  
      
## 4. Fixing the missing references
      
If you read the log you'll notice the compiler can't resolve the references to **<a href="https://github.com/codebude/QRCoder" target="_blank">QRCoder</a>** and that's because we are using a 3rd party library to generate the QR codes.
      
We need to upload the assemblies to the **Functions App**. Connect to your App via SFTP or your favorite method.
      
You'll need to create a **bin** folder under the folder with your function's name (i.e. QRCoder) and upload the **QRCoder.dll** and **UnityEngine.dll** files to it.
      
<a href="/wp-content/uploads/2016/08/sftpqrcoder.png"><img src="/wp-content/uploads/2016/08/sftpqrcoder.png" alt="sftpqrcoder" class="aligcenter size-medium wp-image-5641" /></a>
      
## 6. Create an HTML endpoint for the function
---    
Head to the Integrate tab and add a new HTML trigger and save it with the default values.
      
<a href="/wp-content/uploads/2016/08/trigger.png"><img src="/wp-content/uploads/2016/08/trigger.png" alt="trigger" class="aligcenter size-medium wp-image-5651" /></a>

## 7. Test the Web API
---    
Head to the Develop tab and copy the Function Url:
      
<a href="/wp-content/uploads/2016/08/functionUrl.png"><img src="/wp-content/uploads/2016/08/functionUrl.png" alt="functionUrl" class="aligncenter" size-medium wp-image-5662" srcset="/wp-content/uploads/2016/08/functionUrl.png 1800w, /wp-content/uploads/2016/08/functionUrl-300x28.png 300w, /wp-content/uploads/2016/08/functionUrl-768x70.png 768w, /wp-content/uploads/2016/08/functionUrl-1024x94.png 1024w, /wp-content/uploads/2016/08/functionUrl-250x23.png 250w" sizes="(max-width: 1800px) 100vw, 1800px" /></a>
      
Now use <a href="https://www.getpostman.com" target="_blank">postman </a>to send payload to the Web API and get a QR Code
      
<a href="/wp-content/uploads/2016/08/postman.png"><img src="/wp-content/uploads/2016/08/postman.png" alt="postman" class="aligcenter wp-image-5671" srcset="/wp-content/uploads/2016/08/postman.png 2354w, /wp-content/uploads/2016/08/postman-300x120.png 300w, /wp-content/uploads/2016/08/postman-768x307.png 768w, /wp-content/uploads/2016/08/postman-1024x409.png 1024w, /wp-content/uploads/2016/08/postman-250x100.png 250w" sizes="(max-width: 2354px) 100vw, 2354px" /></a>
      
The response should show a working QR code like the following:
      
<a href="/wp-content/uploads/2016/08/qrcode.png"><img src="/wp-content/uploads/2016/08/qrcode-295x300.png" alt="qrcode" width="295" height="300" class="aligcenter size-medium wp-image-5681" srcset="/wp-content/uploads/2016/08/qrcode-295x300.png 295w, /wp-content/uploads/2016/08/qrcode-768x782.png 768w, /wp-content/uploads/2016/08/qrcode-1006x1024.png 1006w, /wp-content/uploads/2016/08/qrcode-250x254.png 250w, /wp-content/uploads/2016/08/qrcode.png 1123w" sizes="(max-width: 295px) 100vw, 295px" /></a>
            
You can grab the code for the function here: <a href="https://gist.github.com/cmendible/4b6627bcf288b94af9be4d25f6e66d5f">https://gist.github.com/cmendible/4b6627bcf288b94af9be4d25f6e66d5f</a>
        
            
Hope it helps!