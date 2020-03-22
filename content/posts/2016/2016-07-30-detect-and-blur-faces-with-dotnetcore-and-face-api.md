---
author: Carlos Mendible
categories:
- azure
- dotnet
date: "2016-07-30T16:07:18Z"
description: Detect and Blur Faces with .NET Core and Face API
image: /wp-content/uploads/2016/07/detectedfaces.jpg
tags: ["CognitiveServices", "FaceAPI", "ImageProcessorCoreProjectOxford"]
title: Detect and Blur Faces with .NET Core and Face API
url: /2016/07/30/detect-and-blur-faces-with-dotnetcore-and-face-api/
---
Today I'll show you how to create a small console application that will **Detect and Blur Faces with .NET Core and Face API**.

First be aware of the following prerequisites:

<table>
    <tr>
      <td>
        **OS**
      </td>
      <td>
        **Prerequisites**
      </td>
    </tr>
    <tr>
      <td>
        Windows
      </td>
      <td>
        Windows: You must have <a href="https://go.microsoft.com/fwlink/?LinkID=809122" target="_blank">.NET Core SDK for Windows</a> or both <a href="https://www.visualstudio.com/news/releasenotes/vs2015-update3-vs" target="_blank">Visual Studio 2015 Update 3*</a> and <a href="https://go.microsoft.com/fwlink/?LinkId=817245" target="_blank">.NET Core 1.0 for Visual Studio</a> installed.
      </td>
    </tr>
    <tr>
      <td>
        linux, mac or docker
      </td>
      <td>
        checkout <a href="https://www.microsoft.com/net/" target="_blank">.NET Core</a>
      </td>
    </tr>
</table>

You will also need an Azure Cognitive Services Face API account and the correct set of access keys. (Start here: <a href="https://www.microsoft.com/cognitive-services/en-us/sign-up" target="_blank">Subscribe in seconds</a> if you need a Cognitive Service Account and here for the <a href="https://www.microsoft.com/cognitive-services/en-us/documentation" target="_blank">Documentation</a>)

Now let's start:

## 1. Create a folder for your new project
---
Open a command promt an run 
    
``` powershell
mkdir projectoxford
```
## 2. Create the project
---

``` powershell
cd projectoxford
dotnet new
```

## 3. Create a settings file
---
Create an **appsettings.json** file to hold your Face API Key (remember to replace the values with those from your Cognitive Service account):

``` json
{
  "FaceAPIKey": "[Your key here]"
}
```

## 4. Modify the project file
---
Modify the **project.json** to add the **Microsoft.ProjectOxford.Face** dependency and also specify that the **appsettings.json** file must be copied to the output (**buildOptions** section) so it becomes available to the application once you build it.

We'll be needing <a href="https://github.com/JimBobSquarePants/ImageProcessor" target="_blank">ImageProcessorCore </a> to process the image (System.Drawing is not available in .NET Core) and also the extensions and tools to work with configuration files and user secrets.
    
``` json
{
  "userSecretsId": "cmendible3-dotnetcore.samples-projectOxford",
  "version": "1.0.0-*",
  "buildOptions": {
    "debugType": "portable",
    "emitEntryPoint": true,
    "copyToOutput": { 
      "include": "appsettings.json"
    }
  },
  "dependencies": {
    "Microsoft.Extensions.Configuration": "1.0.0",
    "Microsoft.Extensions.Configuration.Json": "1.0.0",
    "Microsoft.Extensions.Configuration.UserSecrets": "1.0.0",
    "Microsoft.ProjectOxford.Face": "1.1.0",
    "System.Runtime.Serialization.Primitives": "4.1.1",
    "ImageProcessorCore": "1.0.0-alpha-966"
  },
  "tools": {
    "Microsoft.Extensions.SecretManager.Tools": "1.0.0-*"
  },
  "frameworks": {
    "netcoreapp1.0": {
      "dependencies": {
        "Microsoft.NETCore.App": {
          "type": "platform",
          "version": "1.0.0"
        }
      },
      "imports": "dnxcore50"
    }
  }
}
```

## 6. Add ImageProcessorCore package source
---
ImageProcessorCore is in alpha stage and packages are available via MyGet so add **NuGet.config** file with the following content:

``` xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <add key="imageprocessor" value="https://www.myget.org/F/imageprocessor/api/v3/index.json" protocolVersion="3" />
  </packageSources>
</configuration>
```

## 7. Restore packages
---
You just modified the **project.json** file with new dependencies and added the NuGet.config file so please restore the packages with the following command:
    
``` powershell
dotnet restore
```

## 8. Modify Program.cs
---
Replace the contents of the **Program.cs** file with the following code 
    
``` csharp
namespace ConsoleApplication
{
    using System;
    using System.IO;
    using System.Linq;
    using System.Threading.Tasks;
    using ImageProcessorCore;
    using Microsoft.Extensions.Configuration;
    using Microsoft.ProjectOxford.Face;
    using Microsoft.ProjectOxford.Face.Contract;

    public class Program
    {
        /// <summary>
        /// Let's detect and blur some faces!
        /// </summary>
        /// 
        public static void Main(string[] args)
        {
            // The name of the source image.
            const string sourceImage = "faces.jpg";

            // The name of the destination image
            const string destinationImage = "detectedfaces.jpg";

            // Get the configuration
            var configuration = BuildConfiguration();

            // Detect the faces in the source file
            DetectFaces(sourceImage, configuration["FaceAPIKey"])
                .ContinueWith((task) =>
                {
                    // Save the result of the detection
                    var faceRects = task.Result;

                    Console.WriteLine($"Detected {faceRects.Length} faces");

                    // Blur the detected faces and save in another file
                    BlurFaces(faceRects, sourceImage, destinationImage);

                    Console.WriteLine($"Done!!!");
                });

            Console.ReadLine();
        }

        /// <summary>
        /// Build the confguration
        /// </summary>
        /// <returns>Returns the configuration</returns>
        private static IConfigurationRoot BuildConfiguration()
        {
            // Enable to app to read json setting files
            var builder = new ConfigurationBuilder()
                .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true);

#if DEBUG
            // We use user secrets in Debug mode so API keys are not uploaded to source control 
            builder.AddUserSecrets("cmendible3-dotnetcore.samples-projectOxford");
#endif

            return builder.Build();
        }

        /// <summary>
        /// Blur the detected faces from de source image.
        /// </summary>
        /// 
        /// 
        /// 
        private static void BlurFaces(FaceRectangle[] faceRects, string sourceImage, string destinationImage)
        {
            if (File.Exists(destinationImage))
            {
                File.Delete(destinationImage);
            }

            if (faceRects.Length > 0)
            {
                using (FileStream stream = File.OpenRead("faces.jpg"))
                using (FileStream output = File.OpenWrite(destinationImage))
                {
                    var image = new Image<Color, uint>(stream);

                    // Blur every detected face
                    foreach (var faceRect in faceRects)
                    {
                        var rectangle = new Rectangle(
                            faceRect.Left,
                            faceRect.Top,
                            faceRect.Width,
                            faceRect.Height);

                        image = image.BoxBlur(20, rectangle);
                    }

                    image.SaveAsJpeg(output);
                }
            }

        }

        /// <summary>
        /// Detect faces calling the Face API
        /// </summary>
        /// 
        /// 
        /// <returns>Detected faces rectangles</returns>
        private static async Task<FaceRectangle[]> DetectFaces(string imageFilePath, string apiKey)
        {
            var faceServiceClient = new FaceServiceClient(apiKey);

            try
            {
                using (Stream imageFileStream = File.OpenRead(imageFilePath))
                {
                    var faces = await faceServiceClient.DetectAsync(imageFileStream);
                    var faceRects = faces.Select(face => face.FaceRectangle);
                    return faceRects.ToArray();
                }
            }
            catch (Exception)
            {
                return new FaceRectangle[0];
            }
        }
    }
}
```
## 9. Build
---
Build and run the application with the following command 
    
``` powershell
dotnet run
```

## 10. Expected results
---
Command line should read 
    
``` powershell
Detected 26 faces
Done!!!
```

The new **detectedfaces.jpg** file should look like this:

<img src="/wp-content/uploads/2016/07/detectedfaces-300x169.jpg" alt="detectedfaces" width="300" height="169" class="alignleft size-medium wp-image-5431" srcset="/wp-content/uploads/2016/07/detectedfaces-300x169.jpg 300w, /wp-content/uploads/2016/07/detectedfaces-768x432.jpg 768w, /wp-content/uploads/2016/07/detectedfaces-1024x576.jpg 1024w, /wp-content/uploads/2016/07/detectedfaces-250x141.jpg 250w" sizes="(max-width: 300px) 100vw, 300px" />
       
You can get the code here: <a href="https://github.com/cmendible/dotnetcore.samples/tree/master/projectoxford">https://github.com/cmendible/dotnetcore.samples/tree/master/projectoxford</a>      
  
Hope it helps!